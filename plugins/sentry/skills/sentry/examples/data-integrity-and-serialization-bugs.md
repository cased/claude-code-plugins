# Data Integrity & Serialization Bugs

Data is malformed, missing, or incorrectly transformed.

## Common Causes

- JSON schema mismatches
- Enum drift between services
- Corrupt payloads
- Incorrect encoding / decoding
- Version skew between producer & consumer

## Why Sentry Helps

- Exception metadata containing payloads
- Errors clustered by request shape
- Release-based regressions

---

## Example 1: Enum Drift Between Frontend and Backend

**Sentry Error:** `Error: Unknown status: PENDING_REVIEW` in `OrderStatusBadge.tsx`

### Sentry Context

```
Error: Error: Unknown status: PENDING_REVIEW
  at getStatusColor (OrderStatusBadge.tsx:12)
  at OrderStatusBadge (OrderStatusBadge.tsx:34)

Tags:
  orderStatus: PENDING_REVIEW
  apiVersion: 2.3.0
  frontendVersion: 2.1.0

Breadcrumbs:
  [http] GET /api/orders/ord_123 → 200
  [console] "Rendering order with status: PENDING_REVIEW"
```

### Analysis

```
Error at OrderStatusBadge.tsx:12
  ↑ Switch statement has no case for "PENDING_REVIEW"
  ↑ Backend returned a status the frontend doesn't know about
  ↑ Backend was updated to v2.3.0 with new status
  ↑ Frontend is still on v2.1.0
  ↑ ROOT CAUSE: No handling for unknown enum values from API
```

### Bad Approach

```typescript
// Add the new status to the switch
function getStatusColor(status: OrderStatus): string {
  switch (status) {
    case 'PENDING': return 'yellow';
    case 'PENDING_REVIEW': return 'orange';  // Just add it
    // ...
  }
}
```

This fixes today's problem but breaks again with the next new status.

### Fix

```typescript
type KnownOrderStatus = 'PENDING' | 'PROCESSING' | 'SHIPPED' | 'DELIVERED' | 'CANCELLED';

function getStatusColor(status: string): string {
  const statusColors: Record<KnownOrderStatus, string> = {
    PENDING: 'yellow',
    PROCESSING: 'blue',
    SHIPPED: 'purple',
    DELIVERED: 'green',
    CANCELLED: 'red',
  };

  if (status in statusColors) {
    return statusColors[status as KnownOrderStatus];
  }

  // Unknown status - use sensible default and log for visibility
  console.warn(`Unknown order status: ${status}`);
  Sentry.captureMessage(`Unknown order status: ${status}`, {
    level: 'warning',
    tags: { status },
  });

  return 'gray';  // Neutral color for unknown statuses
}

function getStatusLabel(status: string): string {
  const statusLabels: Record<KnownOrderStatus, string> = {
    PENDING: 'Pending',
    PROCESSING: 'Processing',
    SHIPPED: 'Shipped',
    DELIVERED: 'Delivered',
    CANCELLED: 'Cancelled',
  };

  // For unknown statuses, format the raw value nicely
  return statusLabels[status as KnownOrderStatus]
    ?? status.replace(/_/g, ' ').toLowerCase().replace(/^\w/, c => c.toUpperCase());
}
```

The fix gracefully handles unknown values while logging them for awareness.

---

## Example 2: JSON Schema Mismatch from API Version Change

**Sentry Error:** `TypeError: user.preferences.notifications is undefined` in `NotificationSettings.tsx`

### Sentry Context

```
Error: TypeError: user.preferences.notifications is undefined
  at NotificationSettings.render (NotificationSettings.tsx:23)

Tags:
  userId: u_older_account
  accountCreated: 2019-03-15

Breadcrumbs:
  [http] GET /api/users/me → 200
  [console] "User preferences: { theme: 'dark' }"  // No notifications key!
```

### Analysis

```
TypeError at NotificationSettings.tsx:23
  ↑ Accessing user.preferences.notifications.email
  ↑ notifications object doesn't exist on this user
  ↑ Older accounts created before 2020 don't have notifications in preferences
  ↑ API returns whatever is in the database without backfilling
  ↑ ROOT CAUSE: Frontend assumes schema that doesn't exist for all users
```

### Bad Approach

```typescript
// Optional chaining everywhere
const emailEnabled = user?.preferences?.notifications?.email ?? true;
const pushEnabled = user?.preferences?.notifications?.push ?? true;
const smsEnabled = user?.preferences?.notifications?.sms ?? false;
```

Scattered defaults make it hard to understand what the actual defaults are.

### Fix

```typescript
// Define the expected shape with defaults
interface NotificationPreferences {
  email: boolean;
  push: boolean;
  sms: boolean;
}

interface UserPreferences {
  theme: 'light' | 'dark';
  notifications: NotificationPreferences;
}

const DEFAULT_NOTIFICATION_PREFERENCES: NotificationPreferences = {
  email: true,
  push: true,
  sms: false,
};

const DEFAULT_USER_PREFERENCES: UserPreferences = {
  theme: 'light',
  notifications: DEFAULT_NOTIFICATION_PREFERENCES,
};

// Normalize API response at the boundary
function normalizeUserPreferences(raw: unknown): UserPreferences {
  if (!raw || typeof raw !== 'object') {
    return DEFAULT_USER_PREFERENCES;
  }

  const prefs = raw as Partial<UserPreferences>;

  return {
    theme: prefs.theme ?? DEFAULT_USER_PREFERENCES.theme,
    notifications: {
      ...DEFAULT_NOTIFICATION_PREFERENCES,
      ...(prefs.notifications ?? {}),
    },
  };
}

// In your API layer
async function fetchCurrentUser(): Promise<User> {
  const response = await api.get('/users/me');
  return {
    ...response.data,
    preferences: normalizeUserPreferences(response.data.preferences),
  };
}
```

The fix normalizes data at the API boundary with explicit defaults.

---

## Example 3: Date Serialization Mismatch

**Sentry Error:** `RangeError: Invalid time value` in `EventCalendar.tsx`

### Sentry Context

```
Error: RangeError: Invalid time value
  at Date.toISOString (<anonymous>)
  at formatEventDate (EventCalendar.tsx:15)
  at EventCard (EventCalendar.tsx:45)

Tags:
  eventId: evt_789

Extra:
  rawDate: "2024-01-15"  // Missing time component
```

### Analysis

```
RangeError at EventCalendar.tsx:15
  ↑ new Date("2024-01-15").toISOString() works in most cases
  ↑ But Date parsing of date-only strings is timezone-dependent
  ↑ In some timezones, "2024-01-15" becomes "2024-01-14T23:00:00Z" or invalid
  ↑ ROOT CAUSE: Inconsistent date format from API + naive Date parsing
```

### Bad Approach

```typescript
// Wrap in try-catch
function formatEventDate(dateStr: string): string {
  try {
    return new Date(dateStr).toLocaleDateString();
  } catch {
    return 'Unknown date';
  }
}
```

This hides the data problem and shows users incorrect information.

### Fix

```typescript
import { parseISO, format, isValid } from 'date-fns';

// Strict parsing that doesn't guess
function parseEventDate(dateStr: string): Date | null {
  // Handle both formats: "2024-01-15" and "2024-01-15T10:00:00Z"
  const parsed = parseISO(dateStr);

  if (!isValid(parsed)) {
    Sentry.captureMessage('Invalid date from API', {
      level: 'warning',
      extra: { rawDate: dateStr },
    });
    return null;
  }

  return parsed;
}

function formatEventDate(dateStr: string): string {
  const date = parseEventDate(dateStr);
  if (!date) {
    return 'Date unavailable';
  }
  return format(date, 'MMMM d, yyyy');
}

// Even better: fix at API boundary
interface RawEvent {
  id: string;
  title: string;
  date: string;  // Could be various formats
}

interface Event {
  id: string;
  title: string;
  date: Date;  // Always a valid Date object
}

function normalizeEvent(raw: RawEvent): Event | null {
  const date = parseEventDate(raw.date);
  if (!date) {
    // Log and skip invalid events rather than crashing
    console.error(`Skipping event ${raw.id} with invalid date: ${raw.date}`);
    return null;
  }

  return {
    id: raw.id,
    title: raw.title,
    date,
  };
}

function normalizeEvents(rawEvents: RawEvent[]): Event[] {
  return rawEvents.map(normalizeEvent).filter((e): e is Event => e !== null);
}
```

The fix uses a proper date parsing library and validates at the API boundary.
