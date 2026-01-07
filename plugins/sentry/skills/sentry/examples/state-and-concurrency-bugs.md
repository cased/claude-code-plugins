# State & Concurrency Bugs

Bugs caused by timing, ordering, or shared state.

## Common Causes

- Race conditions
- Stale cache reads
- Lost updates
- Double execution of idempotent logic
- Incorrect locking

## Why Sentry Helps

- Rare, hard-to-reproduce error clusters
- Errors correlated with concurrency spikes
- Breadcrumb timelines showing ordering

---

## Example 1: Race Condition During Data Fetch

**Sentry Error:** `TypeError: Cannot read properties of null (reading 'map')` in `UserDashboard.tsx`

### Sentry Context

```
Error: TypeError: Cannot read properties of null (reading 'map')
  at UserDashboard.renderProjects (UserDashboard.tsx:45)
  at UserDashboard.render (UserDashboard.tsx:78)

Tags:
  route: /dashboard
  userId: u_789

Breadcrumbs:
  [navigation] /login → /dashboard
  [http] GET /api/user/profile → 200
  [http] GET /api/user/projects → 200
  [state] setUser({ id: 'u_789', ... })
  [state] setProjects([...])  // This arrived AFTER render
```

### Analysis

```
TypeError at UserDashboard.tsx:45: projects.map(...)
  ↑ projects is null during render
  ↑ Component rendered before setProjects completed
  ↑ useEffect triggers fetch, but render doesn't wait
  ↑ ROOT CAUSE: Component renders with initial null state before async data arrives
```

### Bad Approach

```typescript
// Just add optional chaining
{projects?.map(p => <ProjectCard key={p.id} project={p} />)}
```

This silently renders nothing, which may confuse users.

### Fix

```typescript
function UserDashboard() {
  const [projects, setProjects] = useState<Project[] | null>(null);
  const [loading, setLoading] = useState(true);
  const [error, setError] = useState<Error | null>(null);

  useEffect(() => {
    let cancelled = false;

    async function loadProjects() {
      try {
        const data = await fetchProjects();
        if (!cancelled) {
          setProjects(data);
        }
      } catch (e) {
        if (!cancelled) {
          setError(e as Error);
        }
      } finally {
        if (!cancelled) {
          setLoading(false);
        }
      }
    }

    loadProjects();
    return () => { cancelled = true; };
  }, []);

  if (loading) return <DashboardSkeleton />;
  if (error) return <ErrorState error={error} />;
  if (!projects?.length) return <EmptyState />;

  return (
    <div>
      {projects.map(p => <ProjectCard key={p.id} project={p} />)}
    </div>
  );
}
```

The fix explicitly handles all states (loading, error, empty, loaded) and prevents state updates after unmount.

---

## Example 2: Double Form Submission

**Sentry Error:** `Error: Duplicate order created` in `CheckoutService.ts`

### Sentry Context

```
Error: Error: Duplicate order created
  at CheckoutService.createOrder (CheckoutService.ts:67)
  at CheckoutController.submit (CheckoutController.ts:34)

Tags:
  orderId: ord_abc123
  userId: u_456

Breadcrumbs:
  [ui.click] "Place Order" button (timestamp: 1699900000000)
  [ui.click] "Place Order" button (timestamp: 1699900000150)  // 150ms later!
  [http] POST /api/orders → 201
  [http] POST /api/orders → 500 (duplicate key constraint)
```

### Analysis

```
Error at CheckoutService.ts:67
  ↑ Second order creation failed due to duplicate key
  ↑ Two POST requests sent 150ms apart
  ↑ User double-clicked the submit button
  ↑ ROOT CAUSE: No protection against duplicate submissions
```

### Bad Approach

```typescript
// Catch and ignore the duplicate error
try {
  await createOrder(orderData);
} catch (e) {
  if (e.message.includes('duplicate')) {
    // Ignore, probably double-click
    return;
  }
  throw e;
}
```

This masks real duplicate issues and doesn't prevent the wasted API call.

### Fix

```typescript
// Client-side: Disable button and use submission state
function CheckoutForm() {
  const [isSubmitting, setIsSubmitting] = useState(false);

  async function handleSubmit(e: FormEvent) {
    e.preventDefault();
    if (isSubmitting) return;

    setIsSubmitting(true);
    try {
      await submitOrder(formData);
      router.push('/order-confirmation');
    } catch (error) {
      setIsSubmitting(false);
      showError(error);
    }
  }

  return (
    <form onSubmit={handleSubmit}>
      {/* ... */}
      <button type="submit" disabled={isSubmitting}>
        {isSubmitting ? 'Processing...' : 'Place Order'}
      </button>
    </form>
  );
}

// Server-side: Idempotency key
async function createOrder(data: OrderData, idempotencyKey: string) {
  // Check if we've seen this key before
  const existing = await db.orders.findByIdempotencyKey(idempotencyKey);
  if (existing) {
    return existing; // Return the already-created order
  }

  return db.orders.create({
    ...data,
    idempotencyKey,
  });
}
```

Defense in depth: client prevents double-clicks, server handles duplicates gracefully with idempotency keys.

---

## Example 3: Stale Closure in Event Handler

**Sentry Error:** `Error: Cannot update item with stale version` in `DocumentEditor.tsx`

### Sentry Context

```
Error: Error: Cannot update item with stale version
  at DocumentEditor.saveDocument (DocumentEditor.tsx:89)

Tags:
  documentId: doc_123
  expectedVersion: 5
  actualVersion: 8

Breadcrumbs:
  [http] GET /api/documents/doc_123 → 200 (version: 5)
  [http] GET /api/documents/doc_123 → 200 (version: 6)  // Collaborator edit
  [http] GET /api/documents/doc_123 → 200 (version: 7)  // Another edit
  [http] GET /api/documents/doc_123 → 200 (version: 8)  // Another edit
  [ui.click] "Save" button
  [http] PUT /api/documents/doc_123 → 409 (version conflict)
```

### Analysis

```
Error at DocumentEditor.tsx:89
  ↑ PUT request sent with version 5, but document is at version 8
  ↑ Save handler captured version from initial load
  ↑ Real-time updates changed the document but handler still had old version
  ↑ ROOT CAUSE: Event handler closes over stale state
```

### Bad Approach

```typescript
// Force save anyway
async function saveDocument() {
  await api.updateDocument(docId, content, { force: true });
}
```

This overwrites collaborator changes.

### Fix

```typescript
function DocumentEditor({ documentId }: Props) {
  const [document, setDocument] = useState<Document | null>(null);
  const documentRef = useRef(document);

  // Keep ref in sync with state
  useEffect(() => {
    documentRef.current = document;
  }, [document]);

  // Subscribe to real-time updates
  useEffect(() => {
    const unsubscribe = subscribeToDocument(documentId, (updated) => {
      setDocument(updated);
    });
    return unsubscribe;
  }, [documentId]);

  // Use ref in handler to always get current value
  const saveDocument = useCallback(async () => {
    const currentDoc = documentRef.current;
    if (!currentDoc) return;

    try {
      const saved = await api.updateDocument(
        documentId,
        currentDoc.content,
        currentDoc.version
      );
      setDocument(saved);
    } catch (error) {
      if (error.code === 'VERSION_CONFLICT') {
        // Fetch latest and prompt user to merge
        const latest = await api.getDocument(documentId);
        showMergeDialog(currentDoc, latest);
      } else {
        throw error;
      }
    }
  }, [documentId]);

  // ...
}
```

The fix uses a ref to access current state in callbacks, plus handles version conflicts gracefully.
