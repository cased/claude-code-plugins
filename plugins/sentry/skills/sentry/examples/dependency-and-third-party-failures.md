# Dependency & Third-Party Failures

Your code is correct, but something you rely on isn't.

## Common Causes

- Timeouts to payment providers
- SDK bugs after upgrades
- API contract changes
- DNS or TLS issues

## Why Sentry Helps

- External span failures
- Error spikes tied to dependency versions
- Geo-specific or provider-specific failures

---

## Example 1: Payment Provider Timeout

**Sentry Error:** `Error: Request timeout after 30000ms` in `PaymentService.ts`

### Sentry Context

```
Error: Error: Request timeout after 30000ms
  at StripeClient.createCharge (PaymentService.ts:45)
  at CheckoutService.processPayment (CheckoutService.ts:89)

Tags:
  provider: stripe
  operation: createCharge
  amount: 9999
  currency: USD

Breadcrumbs:
  [http] POST /api/checkout → pending
  [http] POST https://api.stripe.com/v1/charges → timeout
  [console] "Stripe request started at 1699900000000"
  [console] "Still waiting for Stripe response..."
```

### Analysis

```
Error at PaymentService.ts:45
  ↑ HTTP request to Stripe timed out after 30s
  ↑ No response received, unclear if charge was created
  ↑ Stripe may have processed the charge but response was lost
  ↑ ROOT CAUSE: No idempotency key, no retry logic, no timeout handling
```

### Bad Approach

```typescript
// Just increase the timeout
const response = await stripe.charges.create(chargeData, { timeout: 60000 });
```

This makes users wait longer and doesn't solve the underlying reliability issues.

### Fix

```typescript
// PaymentService.ts - Resilient third-party integration

import Stripe from 'stripe';
import { v4 as uuidv4 } from 'uuid';

interface ChargeResult {
  success: boolean;
  chargeId?: string;
  error?: string;
  requiresRetry: boolean;
}

class PaymentService {
  private stripe: Stripe;

  constructor() {
    this.stripe = new Stripe(process.env.STRIPE_SECRET_KEY!, {
      timeout: 10000,  // 10s timeout - fail fast
      maxNetworkRetries: 2,  // Stripe SDK handles retries for safe operations
    });
  }

  async createCharge(
    amount: number,
    currency: string,
    source: string,
    idempotencyKey: string  // Caller must provide
  ): Promise<ChargeResult> {
    const startTime = Date.now();

    try {
      const charge = await this.stripe.charges.create(
        { amount, currency, source },
        { idempotencyKey }  // Safe to retry with same key
      );

      return {
        success: true,
        chargeId: charge.id,
        requiresRetry: false,
      };
    } catch (error) {
      const elapsed = Date.now() - startTime;

      if (error instanceof Stripe.errors.StripeConnectionError) {
        // Network issue - might have succeeded
        Sentry.captureException(error, {
          level: 'error',
          tags: {
            provider: 'stripe',
            errorType: 'connection',
            elapsed: elapsed.toString(),
          },
          extra: { idempotencyKey, amount, currency },
        });

        // Try to retrieve the charge using idempotency key
        const existingCharge = await this.findChargeByIdempotencyKey(idempotencyKey);
        if (existingCharge) {
          return {
            success: true,
            chargeId: existingCharge.id,
            requiresRetry: false,
          };
        }

        return {
          success: false,
          error: 'Payment provider temporarily unavailable. Please try again.',
          requiresRetry: true,
        };
      }

      if (error instanceof Stripe.errors.StripeCardError) {
        // Card was declined - don't retry
        return {
          success: false,
          error: error.message,
          requiresRetry: false,
        };
      }

      // Unknown error
      Sentry.captureException(error, {
        tags: { provider: 'stripe', errorType: 'unknown' },
      });

      return {
        success: false,
        error: 'An unexpected error occurred. Please try again.',
        requiresRetry: true,
      };
    }
  }

  private async findChargeByIdempotencyKey(key: string): Promise<Stripe.Charge | null> {
    try {
      // List recent charges and find by metadata
      // Note: In production, you'd store the idempotency key -> charge mapping
      const charges = await this.stripe.charges.list({ limit: 10 });
      return charges.data.find(c => c.metadata?.idempotencyKey === key) || null;
    } catch {
      return null;
    }
  }
}

// CheckoutService.ts - Caller provides idempotency key
class CheckoutService {
  async processPayment(orderId: string, paymentDetails: PaymentDetails): Promise<void> {
    // Use orderId as idempotency key - same order always produces same charge
    const idempotencyKey = `order_${orderId}`;

    const result = await this.paymentService.createCharge(
      paymentDetails.amount,
      paymentDetails.currency,
      paymentDetails.source,
      idempotencyKey
    );

    if (!result.success) {
      if (result.requiresRetry) {
        // Queue for retry or prompt user
        await this.queuePaymentRetry(orderId, paymentDetails);
      }
      throw new PaymentError(result.error!);
    }

    await this.orderService.markAsPaid(orderId, result.chargeId!);
  }
}
```

The fix uses idempotency keys, handles different error types appropriately, and provides clear retry guidance.

---

## Example 2: SDK Bug After Upgrade

**Sentry Error:** `TypeError: this.client.query is not a function` in `DatabaseService.ts`

### Sentry Context

```
Error: TypeError: this.client.query is not a function
  at DatabaseService.findUser (DatabaseService.ts:23)

Tags:
  release: v2.5.0
  nodeVersion: 18.17.0

Extra:
  pgVersion: "8.11.0"  // Note: was 7.x before this release

Breadcrumbs:
  [app.lifecycle] Application started
  [console] "Database connected"
  [http] GET /api/users/123 → 500
```

### Analysis

```
TypeError at DatabaseService.ts:23
  ↑ client.query doesn't exist
  ↑ pg package was upgraded from 7.x to 8.x
  ↑ v8 changed the Client API (breaking change)
  ↑ ROOT CAUSE: Major version upgrade with breaking API changes
```

### Bad Approach

```typescript
// Downgrade the package
// package.json: "pg": "^7.18.0"
```

This avoids the problem but misses security updates and new features.

### Fix

```typescript
// First, check the migration guide and fix the code

// Old pg@7 style:
// const { Client } = require('pg');
// const client = new Client();
// await client.connect();
// const result = await client.query('SELECT...');

// New pg@8 style - Pool is recommended over Client
import { Pool, PoolClient } from 'pg';

class DatabaseService {
  private pool: Pool;

  constructor() {
    this.pool = new Pool({
      connectionString: process.env.DATABASE_URL,
      max: 20,
      idleTimeoutMillis: 30000,
      connectionTimeoutMillis: 2000,
    });

    // Log pool errors
    this.pool.on('error', (err) => {
      Sentry.captureException(err, {
        tags: { component: 'database', event: 'pool_error' },
      });
      console.error('Unexpected database pool error:', err);
    });
  }

  async findUser(id: string): Promise<User | null> {
    const client = await this.pool.connect();
    try {
      const result = await client.query(
        'SELECT * FROM users WHERE id = $1',
        [id]
      );
      return result.rows[0] || null;
    } finally {
      client.release();  // Always release back to pool
    }
  }

  // Helper for transactions
  async withTransaction<T>(fn: (client: PoolClient) => Promise<T>): Promise<T> {
    const client = await this.pool.connect();
    try {
      await client.query('BEGIN');
      const result = await fn(client);
      await client.query('COMMIT');
      return result;
    } catch (error) {
      await client.query('ROLLBACK');
      throw error;
    } finally {
      client.release();
    }
  }
}

// Also add version checking in CI/deployment
// package.json scripts:
// "postinstall": "node scripts/check-dependencies.js"

// scripts/check-dependencies.js
const pkg = require('./package.json');
const semver = require('semver');

const KNOWN_BREAKING_UPGRADES = {
  'pg': { from: '7', to: '8', note: 'Client API changed, use Pool instead' },
  'react': { from: '17', to: '18', note: 'Concurrent features, createRoot API' },
};

// Check if any dependencies crossed a major version boundary
```

The fix updates the code to use the new API correctly and adds safeguards against future breaking upgrades.

---

## Example 3: Third-Party API Contract Change

**Sentry Error:** `TypeError: Cannot read properties of undefined (reading 'latitude')` in `GeocodingService.ts`

### Sentry Context

```
Error: TypeError: Cannot read properties of undefined (reading 'latitude')
  at GeocodingService.parseResponse (GeocodingService.ts:34)

Tags:
  provider: mapbox
  endpoint: geocoding/v5

Extra:
  responseShape: { "features": [...], "type": "FeatureCollection" }
  expectedShape: { "results": [{ "latitude": ..., "longitude": ... }] }

Breadcrumbs:
  [http] GET https://api.mapbox.com/geocoding/v5/... → 200
  [console] "Geocoding response received"
```

### Analysis

```
TypeError at GeocodingService.ts:34
  ↑ Accessing response.results[0].latitude
  ↑ But response has .features[0].geometry.coordinates format
  ↑ API version changed from v4 to v5 (or similar)
  ↑ ROOT CAUSE: API response schema changed, code not updated
```

### Bad Approach

```typescript
// Just update the parsing to match new schema
const lat = response.features[0].geometry.coordinates[1];
```

This fixes it now but doesn't protect against future changes.

### Fix

```typescript
// GeocodingService.ts - Defensive third-party response handling

interface GeocodingResult {
  latitude: number;
  longitude: number;
  formattedAddress: string;
  confidence: number;
}

// Define expected response shapes
interface MapboxV5Response {
  type: 'FeatureCollection';
  features: Array<{
    geometry: {
      type: 'Point';
      coordinates: [number, number];  // [lng, lat]
    };
    properties: {
      full_address?: string;
      name?: string;
    };
    relevance: number;
  }>;
}

interface LegacyMapboxResponse {
  results: Array<{
    latitude: number;
    longitude: number;
    formatted_address: string;
    confidence: number;
  }>;
}

class GeocodingService {
  async geocode(address: string): Promise<GeocodingResult | null> {
    const response = await fetch(
      `https://api.mapbox.com/geocoding/v5/mapbox.places/${encodeURIComponent(address)}.json?access_token=${this.apiKey}`
    );

    if (!response.ok) {
      throw new GeocodingError(`Geocoding failed: ${response.status}`);
    }

    const data = await response.json();
    return this.parseResponse(data);
  }

  private parseResponse(data: unknown): GeocodingResult | null {
    // Try to detect and handle different response shapes
    if (this.isMapboxV5Response(data)) {
      return this.parseMapboxV5(data);
    }

    if (this.isLegacyResponse(data)) {
      // Log that we're seeing legacy format - might indicate config issue
      Sentry.captureMessage('Received legacy Mapbox response format', {
        level: 'warning',
        extra: { responseShape: Object.keys(data) },
      });
      return this.parseLegacyResponse(data);
    }

    // Unknown format - log full details for debugging
    Sentry.captureMessage('Unknown geocoding response format', {
      level: 'error',
      extra: {
        responseType: typeof data,
        responseKeys: data && typeof data === 'object' ? Object.keys(data) : null,
        sample: JSON.stringify(data).slice(0, 500),
      },
    });

    return null;
  }

  private isMapboxV5Response(data: unknown): data is MapboxV5Response {
    return (
      typeof data === 'object' &&
      data !== null &&
      'type' in data &&
      data.type === 'FeatureCollection' &&
      'features' in data &&
      Array.isArray(data.features)
    );
  }

  private isLegacyResponse(data: unknown): data is LegacyMapboxResponse {
    return (
      typeof data === 'object' &&
      data !== null &&
      'results' in data &&
      Array.isArray((data as any).results)
    );
  }

  private parseMapboxV5(data: MapboxV5Response): GeocodingResult | null {
    const feature = data.features[0];
    if (!feature) return null;

    const [lng, lat] = feature.geometry.coordinates;
    return {
      latitude: lat,
      longitude: lng,
      formattedAddress: feature.properties.full_address || feature.properties.name || '',
      confidence: feature.relevance,
    };
  }

  private parseLegacyResponse(data: LegacyMapboxResponse): GeocodingResult | null {
    const result = data.results[0];
    if (!result) return null;

    return {
      latitude: result.latitude,
      longitude: result.longitude,
      formattedAddress: result.formatted_address,
      confidence: result.confidence,
    };
  }
}
```

The fix uses type guards to detect response format, handles multiple schemas gracefully, and logs unknown formats for debugging.
