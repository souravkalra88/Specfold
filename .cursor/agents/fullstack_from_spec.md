# SpecFold v2 — Full Stack from Spec

Use when the spec spans **Angular + Python** in one slice (e.g. one endpoint + one screen) for the **current slice** identified in `progress.md`.

---

## Inputs

- `specfold/<feature>/spec.md` (required)
- `specfold/<feature>/progress.md` (required in v2) — identifies current slice
- Optional: `acceptance.md`, **`specfold/<feature>/repo_kb.md`** — read **first** when present (multi-repo / layout map)
- User may `@`-mention specific frontend and backend paths

---

## Approach

### 1. Read Context (in order)

1. **`repo_kb.md`** (if exists) — multi-repo layout, conventions
2. **`spec.md`** — feature requirements and acceptance criteria
3. **`progress.md`** (v2) — identify **current slice** to implement

### 2. Implement ONE Vertical Slice

Split work into **API contract** → **backend** → **frontend** (or the reverse if the UI already exists); keep each step small.

**Vertical slice components:**
- ✅ **API:** FastAPI endpoint + Pydantic schemas + OpenAPI
- ✅ **Data:** Database models + repositories (if needed)
- ✅ **UI:** Angular component + service + route
- ✅ **Tests:** Backend (pytest) + Frontend (Jest/Jasmine) + Integration

### 3. Apply Production-Grade Rules

Apply both:
- **`.cursor/rules/python.mdc`** for `*.py` files
- **`.cursor/rules/angular.mdc`** for Angular files
- **`.cursor/rules/specfold-v2.mdc`** — Incremental slicing, backward compatibility

### 4. Align Frontend + Backend

#### API Contract
- **OpenAPI spec:** Define contract first
- **Pydantic schemas:** Match OpenAPI
- **TypeScript interfaces:** Match Pydantic schemas
  ```python
  # Backend: Pydantic
  class CreateOrderRequest(BaseModel):
      items: list[OrderItem]
      total_amount: Decimal
  
  class OrderResponse(BaseModel):
      id: UUID
      user_id: UUID
      total_amount: Decimal
      status: str
      created_at: datetime
  ```
  
  ```typescript
  // Frontend: TypeScript
  export interface CreateOrderRequest {
    items: OrderItem[];
    totalAmount: number; // camelCase for frontend
  }
  
  export interface OrderResponse {
    id: string;
    userId: string;
    totalAmount: number;
    status: string;
    createdAt: string; // ISO 8601
  }
  ```

#### Error Shapes
- **Backend:** `HTTPException` with status codes
  ```python
  raise HTTPException(status_code=400, detail="Invalid order data")
  ```
- **Frontend:** Handle errors in service
  ```typescript
  createOrder(order: CreateOrderRequest): Observable<OrderResponse> {
    return this.http.post<OrderResponse>('/orders', order).pipe(
      catchError((error: HttpErrorResponse) => {
        console.error('Order creation failed:', error);
        return throwError(() => new Error('Failed to create order'));
      })
    );
  }
  ```

#### Auth Headers
- **Backend:** JWT validation via `Depends(get_current_user)`
- **Frontend:** HTTP interceptor adds `Authorization` header
  ```typescript
  @Injectable()
  export class AuthInterceptor implements HttpInterceptor {
    intercept(req: HttpRequest<any>, next: HttpHandler): Observable<HttpEvent<any>> {
      const token = this.authService.getToken();
      if (token) {
        req = req.clone({
          setHeaders: { Authorization: `Bearer ${token}` }
        });
      }
      return next.handle(req);
    }
  }
  ```

---

## Implementation Order

### Step 1: Backend (API + Data)

1. **Define Pydantic schemas** (request/response)
2. **Create router** (FastAPI endpoint)
3. **Implement service** (business logic)
4. **Create repository** (database access, if needed)
5. **Write tests** (pytest: unit + integration)

### Step 2: Frontend (UI)

1. **Define TypeScript interfaces** (match Pydantic schemas)
2. **Create service** (HTTP calls)
3. **Create component** (UI + form)
4. **Add route** (lazy loaded, auth guard)
5. **Write tests** (Jest/Jasmine: component + service)

### Step 3: Integration

1. **End-to-end test** (Cypress)
2. **Manual smoke test**
3. **Update tracking** (`progress.md`, `changes.md`)

---

## Tests

- **Backend:** pytest for new/changed endpoints
- **Frontend:** unit tests for components/services you change
- **E2E:** Cypress for acceptance criteria

---

## Output

### Summary Format

```markdown
## Slice [N] Full Stack Implementation: [Slice Name]

### Backend Changes
**Endpoints:**
- POST /orders (create order)

**Files:**
- `app/routers/orders.py` (new)
- `app/services/order_service.py` (new)
- `app/schemas/order.py` (new)
- `tests/integration/test_orders_api.py` (new)

### Frontend Changes
**Components:**
- `order-form.component.ts` (new, standalone)

**Routes:**
- `/orders/create` → OrderFormComponent (auth guard)

**Files:**
- `src/app/features/orders/order-form.component.ts` (new)
- `src/app/features/orders/order-form.component.spec.ts` (new)
- `src/app/core/services/order.service.ts` (new)
- `src/app/app.routes.ts` (added route)

### Acceptance Criteria Covered
- ✅ SF-001: User can create order with items (UI + API)
- ✅ SF-002: Form validates inputs (UI)
- ✅ SF-005: Order stored in database (API)

### How to Verify

#### Backend Tests
```bash
pytest tests/integration/test_orders_api.py
pytest tests/services/test_order_service.py
```

#### Frontend Tests
```bash
ng test --include='**/order-form.component.spec.ts'
```

#### E2E Test
```bash
npx cypress run --spec 'cypress/e2e/create-order.cy.ts'
```

#### Manual Test
```bash
# Start backend
uvicorn app.main:app --reload

# Start frontend
ng serve

# Navigate to http://localhost:4200/orders/create
# Fill form and submit
```

### API Contract

**Request:**
```json
POST /orders
{
  "items": [
    {"productId": "123", "quantity": 2, "price": 24.99}
  ],
  "totalAmount": 49.98
}
```

**Response:**
```json
201 Created
{
  "id": "550e8400-e29b-41d4-a716-446655440000",
  "userId": "660e8400-e29b-41d4-a716-446655440000",
  "totalAmount": 49.98,
  "status": "pending",
  "createdAt": "2024-06-11T10:30:00Z"
}
```

### Files Touched
**Backend:** 5 files  
**Frontend:** 4 files  
**Total:** 9 files

### Next Steps
1. Review full stack changes against acceptance criteria
2. Run all tests (backend + frontend + E2E)
3. Update `progress.md`: Mark Slice [N] complete
4. Add entry to `changes.md` with deployment date
5. Deploy to staging → validate → deploy to production
6. Implement Slice [N+1] after validation
```

---

## When to Split

If the change is **large** (> 10 files or > 500 lines), tell the user to split into separate turns:

1. **Backend first:** `@python_api_from_spec.md` or `@python_aws_from_spec.md`
2. **Frontend second:** `@angular_from_spec.md`
3. **Or by slice:** Implement Slice 1 backend + frontend, then Slice 2, etc.

---

## Traceability

Reference **`SF-…`** IDs in:
- **Backend code:** `# SF-001: Validate order items before saving`
- **Frontend code:** `// SF-001: Display order form with items`
- **Test names:** `test_create_order_full_stack__SF_001()`
- **Commit messages:** `[Slice 1] Add create order (UI + API) (SF-001, SF-002, SF-005)`

---

## Quality Checklist

Before finishing:

- [ ] Implemented **only** the current slice (not entire feature)
- [ ] **API contract** defined (Pydantic schemas + OpenAPI)
- [ ] **TypeScript interfaces** match Pydantic schemas
- [ ] **Backend:** Type hints, input validation, error handling, tests
- [ ] **Frontend:** Standalone components, signals, OnPush, Reactive Forms, tests
- [ ] **Auth:** Server-side JWT validation + Angular auth guard
- [ ] **Error handling:** Backend HTTPException + Frontend catchError
- [ ] **Tests:** Backend (pytest) + Frontend (Jest) + E2E (Cypress)
- [ ] **No secrets** in code
- [ ] **Backward compatible** (or migration plan)
- [ ] **SF-…** IDs referenced in code/tests

---

## Example Invocation

```
@.cursor/agents/fullstack_from_spec.md
@specfold/user-orders/spec.md
@specfold/user-orders/progress.md
@specfold/user-orders/repo_kb.md

Implement current slice (Slice 1: Create Order - UI + API)
Use Angular 19+ standalone components + FastAPI.
```
