# SpecFold v2 — Angular from Spec

You implement **Angular** UI and client-side behavior **only** from `specfold/<feature>/spec.md` for the **current slice** identified in `progress.md`.

---

## Before Coding

1. If **`specfold/<feature>/repo_kb.md`** exists (or user attached it), **read it first** — use its maps and conventions.
2. If **`specfold/<feature>/progress.md`** exists, **read it** to identify the **current slice**.
3. Read the spec; list **components, routes, and APIs** for the **current slice only**.
4. Explore the repo for **existing** patterns (validate or extend `repo_kb.md`).

---

## While Coding

### Follow Rules
- **`.cursor/rules/angular.mdc`** — Angular 19+ standalone components, signals, OnPush
- **`.cursor/rules/specfold-v2.mdc`** — Incremental slicing, backward compatibility, scope discipline
- Global **SpecFold** scope rules: no scope creep, **`SF-…`** traceability

### Angular Best Practices
- **Standalone components** (no NgModule)
- **Signals** for reactive state (`signal()`, `computed()`, `effect()`)
- **inject()** for dependency injection (not constructor injection)
- **OnPush** change detection for performance
- **Reactive Forms** with Validators for input validation
- **Lazy loading** for routes
- **Virtual scrolling** for large lists (CDK VirtualScrollViewport)

### Backward Compatibility
- **Additive changes:** New routes, new components, new services
- **Feature flags:** Hide incomplete UI
  ```typescript
  export const FEATURE_FLAGS = {
    viewOrders: false, // Not ready yet
    createOrders: true
  };
  
  if (FEATURE_FLAGS.viewOrders) {
    // Show "View Orders" button
  }
  ```
- **Avoid breaking changes:** Removing routes, renaming components, changing public APIs

### Security
- **Input validation:** Reactive Forms with `Validators.required`, `Validators.email`, etc.
- **XSS prevention:** Angular sanitizes by default; avoid bypassing with `DomSanitizer`
- **Auth guard:** Use Angular `CanActivate` guard for protected routes
  ```typescript
  const routes: Routes = [
    { path: 'orders', component: OrderListComponent, canActivate: [AuthGuard] }
  ];
  ```
- **JWT handling:** Store JWT in `HttpOnly` cookie (not localStorage)
- **CSRF protection:** Angular `HttpClient` includes XSRF token by default

### Error Handling
- **HTTP errors:** Use `catchError` in services
  ```typescript
  return this.http.post('/orders', order).pipe(
    catchError((error: HttpErrorResponse) => {
      console.error('Order creation failed:', error);
      return throwError(() => new Error('Failed to create order'));
    })
  );
  ```
- **User-friendly messages:** Display errors in UI (toasts, inline messages)
- **Retry logic:** Use `retry(3)` for transient errors

### Performance
- **OnPush change detection** for all components
- **trackBy** for `*ngFor` loops
  ```typescript
  trackByOrderId(index: number, order: Order): string {
    return order.id;
  }
  ```
- **Virtual scrolling** for large lists (>100 items)
- **Lazy loading** for routes
- **Debounce** user input for search (500ms)
  ```typescript
  searchControl.valueChanges.pipe(debounceTime(500), distinctUntilChanged())
  ```

### Testing
- **Unit tests:** Jest or Jasmine with Angular TestBed
  ```typescript
  describe('OrderFormComponent', () => {
    it('should create order with valid inputs (SF-001)', () => {
      // Test logic
    });
  });
  ```
- **Coverage:** >80% for components and services in the slice
- **E2E tests:** Cypress for acceptance criteria
  ```typescript
  describe('Create Order (SF-001)', () => {
    it('allows user to create order with multiple items', () => {
      // Cypress test
    });
  });
  ```

---

## Integration with Backend

- Wire **HTTP** through existing services/interceptors
- Do **not** duplicate auth or error handling if the app already centralizes it
- **Align types:** Keep TypeScript interfaces in sync with Pydantic schemas
  ```typescript
  // Match backend Pydantic schema
  export interface CreateOrderRequest {
    items: OrderItem[];
    totalAmount: number; // Match backend field name
    userId: string;
  }
  ```
- **API contract:** Follow OpenAPI spec (if available)

---

## Tests

Add or update **component/service tests** that prove acceptance for the UI behavior in the **current slice** (Jest or Karma per project).

---

## Output

### Summary Format

```markdown
## Slice [N] UI Implementation: [Slice Name]

### Components Added/Modified
- `order-form.component.ts` (new, standalone)
- `order-form.component.html` (template)
- `order-form.component.scss` (styles)
- `order.service.ts` (HTTP service)

### Routes Added
- `/orders/create` → OrderFormComponent (lazy loaded, auth guard)

### Acceptance Criteria Covered (UI)
- ✅ SF-001: User can fill order form with items
- ✅ SF-002: Form validates inputs (required fields, min quantity)

### How to Verify
```bash
# Run unit tests
ng test --include='**/order-form.component.spec.ts'

# Run E2E tests
npx cypress run --spec 'cypress/e2e/create-order.cy.ts'

# Manual test
ng serve
# Navigate to http://localhost:4200/orders/create
```

### Files Touched
- `src/app/features/orders/order-form.component.ts` (new)
- `src/app/features/orders/order-form.component.html` (new)
- `src/app/features/orders/order-form.component.spec.ts` (new)
- `src/app/core/services/order.service.ts` (modified)
- `src/app/app.routes.ts` (added /orders/create route)

### Next Steps
1. Review UI against acceptance criteria
2. Run tests: `ng test`
3. Update `progress.md`: Mark Slice [N] complete
4. Deploy to staging → validate → deploy to production
```

---

## Traceability

Reference **`SF-…`** IDs in:
- **Component comments:** `// SF-001: Validate order items before submission`
- **Test descriptions:** `it('should validate order items (SF-001)', () => {...})`
- **Commit messages:** `[Slice 1] Add order form component (SF-001, SF-002)`

---

## Quality Checklist

Before finishing:

- [ ] Implemented **only** the current slice (not entire feature)
- [ ] **Standalone components** (no NgModule)
- [ ] **Signals** for reactive state
- [ ] **OnPush** change detection
- [ ] **Reactive Forms** with validation
- [ ] **Auth guard** for protected routes
- [ ] **Error handling** with user-friendly messages
- [ ] **Tests** written (>80% coverage for slice)
- [ ] **No secrets** in code (API keys, tokens)
- [ ] **Feature flags** for incomplete UI
- [ ] **SF-…** IDs referenced in tests/comments

---

## Example Invocation

```
@.cursor/agents/angular_from_spec.md
@specfold/user-orders/spec.md
@specfold/user-orders/progress.md
@specfold/user-orders/repo_kb.md

Implement current slice UI (Slice 1: Create Order Form)
Use Angular 19+ standalone components with signals.
```
