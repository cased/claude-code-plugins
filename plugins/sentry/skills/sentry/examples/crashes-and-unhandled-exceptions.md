# Crashes & Unhandled Exceptions

The process dies or a request fails hard due to an uncaught error.

## Common Causes

- Null / undefined dereferences
- Index out of bounds
- Type errors (calling a method on the wrong shape)
- Panic / segfaults in native code
- Uncaught promise rejections

## Why Sentry Helps

- Automatic exception capture
- Stack traces with source maps
- Environment, release, and breadcrumb context

---

## Example 1: NaN Propagation from Undefined Config

**Sentry Error:** `Invalid revenue amount for category: Subscriptions` in `RevenueChart.tsx`

### Bad Approach (fixing at crash site)

```
RevenueChart.tsx throws on NaN → fix: add default in transformRevenue() → done
```

This masks the real problem. The NaN will still flow through other code paths.

### Good Approach (tracing upstream)

```
RevenueChart.tsx throws on NaN
  ↑ amount is NaN
  ↑ transformRevenue() computed: revenue * undefined
  ↑ multiplier param was undefined
  ↑ useDashboardData called getRevenueMultiplier()
  ↑ config.ts returns undefined when useCustomConfig=false and revenueMultiplier is unset
  ↑ ROOT CAUSE: config.ts can return undefined for a value that should always be a number
```

### Fix

1. **Fix config.ts** to always return a number (root cause)
2. **Add fallback in transformRevenue** (defense in depth - appropriate here because data crosses a trust boundary)
3. **Fix fetch handling** for "body stream already read" error found in breadcrumbs (secondary bug)

---

## Example 2: Uncaught Promise Rejection in Async Handler

**Sentry Error:** `TypeError: Cannot read properties of undefined (reading 'id')` in `OrderService.ts`

### Sentry Context

```
Error: TypeError: Cannot read properties of undefined (reading 'id')
  at OrderService.processOrder (OrderService.ts:47)
  at async OrderController.submitOrder (OrderController.ts:23)

Breadcrumbs:
  [http] POST /api/inventory/reserve → 500
  [http] GET /api/users/current → 200
  [console] "Processing order for user: u_abc123"
```

### Bad Approach

```typescript
// OrderService.ts:47
async processOrder(userId: string, items: CartItem[]) {
  const reservation = await this.inventoryClient.reserve(items);
  // Add null check here
  const reservationId = reservation?.id ?? 'unknown';  // Masks the failure
  // ...
}
```

### Good Approach (trace the breadcrumbs)

The breadcrumbs show `POST /api/inventory/reserve → 500`. The inventory service failed, but the code didn't handle it.

```
TypeError at OrderService.ts:47
  ↑ reservation.id accessed when reservation is undefined
  ↑ inventoryClient.reserve() returned undefined instead of throwing
  ↑ HTTP client swallowed the 500 error and returned undefined
  ↑ ROOT CAUSE: inventoryClient doesn't throw on non-2xx responses
```

### Fix

```typescript
// inventoryClient.ts
async reserve(items: CartItem[]): Promise<Reservation> {
  const response = await fetch('/api/inventory/reserve', {
    method: 'POST',
    body: JSON.stringify({ items }),
  });

  if (!response.ok) {
    throw new InventoryReservationError(
      `Failed to reserve inventory: ${response.status}`,
      { status: response.status, items }
    );
  }

  return response.json();
}
```

Now the error surfaces where it belongs, with proper context for debugging.

---

## Example 3: Array Index Out of Bounds from Stale Cache

**Sentry Error:** `TypeError: Cannot read properties of undefined (reading 'name')` in `TeamSelector.tsx`

### Sentry Context

```
Error: TypeError: Cannot read properties of undefined (reading 'name')
  at TeamSelector.render (TeamSelector.tsx:34)
  at renderWithHooks (react-dom.js:...)

Tags:
  selectedTeamIndex: 3
  teamsCount: 2

Breadcrumbs:
  [navigation] /settings → /dashboard
  [http] GET /api/teams → 200 (returned 2 teams)
  [ui.click] "Switch Team" button
```

### Analysis

```
TypeError at TeamSelector.tsx:34: teams[selectedIndex].name
  ↑ teams[3] is undefined because teams only has 2 items
  ↑ selectedIndex is 3, stored in localStorage
  ↑ User had 4 teams, selected index 3, then was removed from 2 teams
  ↑ ROOT CAUSE: selectedTeamIndex persisted in localStorage without validation
```

### Bad Approach

```typescript
// Just add optional chaining at crash site
const teamName = teams[selectedIndex]?.name ?? 'Unknown';
```

This silently shows "Unknown" instead of fixing the stale state.

### Fix

```typescript
// TeamSelector.tsx
function TeamSelector() {
  const { teams } = useTeams();
  const [selectedIndex, setSelectedIndex] = useLocalStorage('selectedTeamIndex', 0);

  // Validate persisted index against current data
  const validIndex = selectedIndex < teams.length ? selectedIndex : 0;

  // Sync if index was invalid
  useEffect(() => {
    if (selectedIndex !== validIndex) {
      setSelectedIndex(validIndex);
    }
  }, [selectedIndex, validIndex, setSelectedIndex]);

  const selectedTeam = teams[validIndex];
  // ...
}
```

The fix validates cached state against current reality and self-heals when they diverge.
