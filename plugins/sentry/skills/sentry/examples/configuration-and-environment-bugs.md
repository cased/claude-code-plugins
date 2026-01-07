# Configuration & Environment Bugs

Code behaves differently due to misconfiguration.

## Common Causes

- Missing env vars
- Incorrect feature flag values
- Secrets misconfigured
- Region-specific config drift

## Why Sentry Helps

- Errors isolated to specific environments
- Release vs environment mismatch
- Sudden failures after deploys without code changes

---

## Example 1: Missing Environment Variable in Production

**Sentry Error:** `Error: API_BASE_URL is not defined` in `apiClient.ts`

### Sentry Context

```
Error: Error: API_BASE_URL is not defined
  at createApiClient (apiClient.ts:8)
  at Module.<anonymous> (index.ts:12)

Tags:
  environment: production
  release: v2.4.0
  server: prod-us-east-1b

Breadcrumbs:
  [app.lifecycle] Application starting
  [console] "Initializing API client..."
```

### Analysis

```
Error at apiClient.ts:8
  ↑ process.env.API_BASE_URL is undefined
  ↑ Works in staging (env var set), fails in production
  ↑ New server spun up without complete env configuration
  ↑ ROOT CAUSE: No validation of required env vars at startup
```

### Bad Approach

```typescript
// Default to production URL
const API_BASE_URL = process.env.API_BASE_URL || 'https://api.example.com';
```

This silently uses potentially wrong defaults and masks configuration issues.

### Fix

```typescript
// config.ts - Validate all required config at startup
interface Config {
  apiBaseUrl: string;
  environment: 'development' | 'staging' | 'production';
  sentryDsn: string | null;
  featureFlags: {
    newCheckout: boolean;
  };
}

function getRequiredEnv(name: string): string {
  const value = process.env[name];
  if (!value) {
    throw new Error(
      `Missing required environment variable: ${name}. ` +
      `Ensure this is set in your deployment configuration.`
    );
  }
  return value;
}

function getOptionalEnv(name: string, defaultValue: string): string {
  return process.env[name] || defaultValue;
}

function loadConfig(): Config {
  // Validate all required vars upfront - fail fast
  const config: Config = {
    apiBaseUrl: getRequiredEnv('API_BASE_URL'),
    environment: getRequiredEnv('NODE_ENV') as Config['environment'],
    sentryDsn: process.env.SENTRY_DSN || null,
    featureFlags: {
      newCheckout: process.env.FF_NEW_CHECKOUT === 'true',
    },
  };

  // Additional validation
  if (!['development', 'staging', 'production'].includes(config.environment)) {
    throw new Error(`Invalid NODE_ENV: ${config.environment}`);
  }

  try {
    new URL(config.apiBaseUrl);
  } catch {
    throw new Error(`Invalid API_BASE_URL: ${config.apiBaseUrl}`);
  }

  console.log(`Config loaded for ${config.environment} environment`);
  return config;
}

// Load once at startup - crash immediately if misconfigured
export const config = loadConfig();

// apiClient.ts
import { config } from './config';

export const apiClient = axios.create({
  baseURL: config.apiBaseUrl,
});
```

The fix validates all configuration at startup, failing fast with clear error messages.

---

## Example 2: Feature Flag Type Mismatch

**Sentry Error:** `TypeError: Cannot read properties of undefined (reading 'enabled')` in `FeatureGate.tsx`

### Sentry Context

```
Error: TypeError: Cannot read properties of undefined (reading 'enabled')
  at FeatureGate.render (FeatureGate.tsx:18)

Tags:
  environment: production
  flagKey: new_pricing_page

Extra:
  flagValue: "true"  // String, not object!

Breadcrumbs:
  [http] GET /api/feature-flags → 200
  [console] "Flag new_pricing_page: true"
```

### Analysis

```
TypeError at FeatureGate.tsx:18
  ↑ Accessing flag.enabled but flag is the string "true"
  ↑ Code expects { enabled: boolean, variant?: string }
  ↑ Flag service returns simple boolean for some flags, object for others
  ↑ ROOT CAUSE: Inconsistent flag value shapes from flag service
```

### Bad Approach

```typescript
// Just handle both shapes inline
const isEnabled = typeof flag === 'object' ? flag.enabled : flag === 'true';
```

This spreads type handling throughout the codebase.

### Fix

```typescript
// featureFlags.ts - Normalize at the boundary

interface FeatureFlag {
  enabled: boolean;
  variant?: string;
  metadata?: Record<string, unknown>;
}

type RawFlagValue = boolean | string | { enabled: boolean; variant?: string };

function normalizeFlag(key: string, raw: RawFlagValue): FeatureFlag {
  // Boolean
  if (typeof raw === 'boolean') {
    return { enabled: raw };
  }

  // String (from env vars or misconfigured service)
  if (typeof raw === 'string') {
    const enabled = raw === 'true' || raw === '1' || raw === 'enabled';
    console.warn(`Feature flag "${key}" returned string value, normalizing to boolean`);
    return { enabled };
  }

  // Object shape
  if (typeof raw === 'object' && raw !== null) {
    return {
      enabled: Boolean(raw.enabled),
      variant: raw.variant,
    };
  }

  // Unexpected shape - log and default to disabled
  Sentry.captureMessage(`Unexpected feature flag shape for "${key}"`, {
    level: 'warning',
    extra: { key, rawValue: raw, rawType: typeof raw },
  });
  return { enabled: false };
}

class FeatureFlagService {
  private flags: Map<string, FeatureFlag> = new Map();

  async load(): Promise<void> {
    const response = await fetch('/api/feature-flags');
    const rawFlags: Record<string, RawFlagValue> = await response.json();

    for (const [key, value] of Object.entries(rawFlags)) {
      this.flags.set(key, normalizeFlag(key, value));
    }
  }

  isEnabled(key: string): boolean {
    const flag = this.flags.get(key);
    if (!flag) {
      console.warn(`Unknown feature flag: ${key}`);
      return false;
    }
    return flag.enabled;
  }

  getVariant(key: string): string | undefined {
    return this.flags.get(key)?.variant;
  }
}

export const featureFlags = new FeatureFlagService();

// Usage in components
function FeatureGate({ flag, children }: { flag: string; children: React.ReactNode }) {
  if (!featureFlags.isEnabled(flag)) {
    return null;
  }
  return <>{children}</>;
}
```

The fix normalizes all flag values to a consistent shape at load time.

---

## Example 3: Region-Specific Configuration Drift

**Sentry Error:** `Error: Connection timeout to database` in `DatabasePool.ts`

### Sentry Context

```
Error: Error: Connection timeout to database
  at DatabasePool.getConnection (DatabasePool.ts:34)

Tags:
  environment: production
  region: eu-west-1
  release: v3.1.0

Breadcrumbs:
  [app.lifecycle] Database pool initializing
  [console] "Connecting to: db-primary.us-east-1.rds.amazonaws.com"  // Wrong region!
```

### Analysis

```
Error at DatabasePool.ts:34
  ↑ Connection timeout after 30s
  ↑ EU servers connecting to US database (high latency + firewall rules)
  ↑ DATABASE_URL not set for EU region, fell back to hardcoded US default
  ↑ ROOT CAUSE: Region-specific config not deployed to new EU region
```

### Bad Approach

```typescript
// Add region detection and pick URL
const DATABASE_URL = process.env.DATABASE_URL
  || (isEuRegion() ? EU_DATABASE_URL : US_DATABASE_URL);
```

This adds complexity and still masks the real configuration problem.

### Fix

```typescript
// config.ts - Region-aware configuration with strict validation

interface DatabaseConfig {
  url: string;
  region: string;
  maxConnections: number;
  connectionTimeout: number;
}

function loadDatabaseConfig(): DatabaseConfig {
  const url = process.env.DATABASE_URL;
  const region = process.env.AWS_REGION || process.env.REGION;

  if (!url) {
    throw new Error(
      `DATABASE_URL is not set. ` +
      `Ensure the database configuration is deployed for region: ${region || 'unknown'}`
    );
  }

  if (!region) {
    throw new Error(
      'AWS_REGION or REGION environment variable must be set'
    );
  }

  // Validate that database URL matches expected region
  const dbRegion = extractRegionFromUrl(url);
  if (dbRegion && dbRegion !== region) {
    // This is a critical misconfiguration - fail loudly
    throw new Error(
      `Database region mismatch! ` +
      `App is running in ${region} but DATABASE_URL points to ${dbRegion}. ` +
      `This will cause latency issues and may violate data residency requirements.`
    );
  }

  return {
    url,
    region,
    maxConnections: parseInt(process.env.DB_MAX_CONNECTIONS || '10', 10),
    connectionTimeout: parseInt(process.env.DB_TIMEOUT || '5000', 10),
  };
}

function extractRegionFromUrl(url: string): string | null {
  // Match patterns like "db.us-east-1.rds.amazonaws.com"
  const match = url.match(/\.(us|eu|ap|sa|ca|me|af)-[a-z]+-\d+\./);
  return match ? match[0].slice(1, -1) : null;
}

// Validate at startup
export const dbConfig = loadDatabaseConfig();
console.log(`Database configured for region ${dbConfig.region}`);
```

The fix validates that configuration is correct for the current region and fails fast with actionable error messages.
