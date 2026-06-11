# SpecFold v2 — Python API from Spec

You implement **HTTP APIs and application services** from `specfold/<feature>/spec.md` for the **current slice** identified in `progress.md`.

---

## Before Coding

1. If **`specfold/<feature>/repo_kb.md`** exists or is attached, **read it first**.
2. If **`specfold/<feature>/progress.md`** exists, **read it** to identify the **current slice**.
3. Read **Behavior & APIs** and **Acceptance** in the spec; note **`SF-…`** IDs for the current slice.
4. Find existing **routers**, **schemas**, and **dependency** patterns in the repo (use `repo_kb.md` as the primary map).

---

## While Coding

### Follow Rules
- **`.cursor/rules/python.mdc`** — FastAPI, Pydantic, type hints, dependency injection
- **`.cursor/rules/specfold-v2.mdc`** — Incremental slicing, backward compatibility, scope discipline
- Global SpecFold rules: scope, OpenAPI/Pydantic sync, no invented requirements

### FastAPI Best Practices
- **Thin routers:** Keep handlers small, delegate to services
  ```python
  @router.post("/orders", response_model=OrderResponse, status_code=201)
  async def create_order(
      order: CreateOrderRequest,
      service: OrderService = Depends(get_order_service),
      current_user: User = Depends(get_current_user)
  ) -> OrderResponse:
      return await service.create_order(order, current_user)
  ```
- **Fat services:** Business logic in service layer
- **Dependency injection:** Use `Depends()` for services, DB sessions, auth
- **Pydantic schemas:** Define request/response models
  ```python
  class CreateOrderRequest(BaseModel):
      items: list[OrderItem]
      total_amount: Decimal
      user_id: UUID
      
      @field_validator('items')
      def validate_items(cls, items):
          if not items:
              raise ValueError('Order must have at least one item')
          return items
  ```
- **OpenAPI sync:** Keep OpenAPI spec aligned with Pydantic schemas

### Backward Compatibility
- **Additive changes:** New endpoints, new optional fields
  ```python
  class OrderResponse(BaseModel):
      id: UUID
      total: Decimal  # Existing field
      total_amount: Decimal | None = None  # New field (optional)
  ```
- **API versioning:** Use `/v1/` and `/v2/` for breaking changes
  ```python
  @router.post("/v1/orders")  # Old version
  @router.post("/v2/orders")  # New version
  ```
- **Deprecation:** Mark old endpoints as deprecated in OpenAPI
  ```python
  @router.get("/v1/orders", deprecated=True)
  ```
- **Feature flags:** Control endpoint exposure
  ```python
  if feature_flags["view_orders"]:
      app.include_router(orders_router)
  ```

### Security (OWASP)
- **Input validation:** Pydantic models with validators
  ```python
  class CreateOrderRequest(BaseModel):
      items: list[OrderItem] = Field(..., min_length=1, max_length=100)
      total_amount: Decimal = Field(..., gt=0, le=1000000)
      email: EmailStr
  ```
- **Authorization:** Server-side JWT validation
  ```python
  async def get_current_user(token: str = Depends(oauth2_scheme)) -> User:
      # Validate JWT, check roles, return user
      pass
  ```
- **Secrets:** AWS Secrets Manager / SSM (never in code)
  ```python
  import boto3
  
  secrets_client = boto3.client('secretsmanager')
  db_password = secrets_client.get_secret_value(SecretId='db-password')
  ```
- **Error handling:** Don't leak internal details
  ```python
  try:
      result = await service.create_order(order)
  except ValidationError as e:
      raise HTTPException(status_code=400, detail="Invalid order data")
  except Exception as e:
      logger.error(f"Order creation failed: {e}")
      raise HTTPException(status_code=500, detail="Internal server error")
  ```

### AWS Integration (if deployed on ECS/Lambda)
- **IAM roles:** Attach roles to ECS tasks / Lambda functions (no keys in code)
- **Secrets:** Use AWS Secrets Manager or SSM Parameter Store
- **Observability:** CloudWatch Logs, X-Ray tracing
  ```python
  from aws_xray_sdk.core import xray_recorder
  
  @xray_recorder.capture('create_order')
  async def create_order(...):
      pass
  ```
- **Database:** Use RDS Proxy for connection pooling (ECS/Lambda)

### Error Handling
- **User errors:** Use `HTTPException` with appropriate status codes
  ```python
  if not order.items:
      raise HTTPException(status_code=400, detail="Order must have items")
  ```
- **Server errors:** Log details, return generic message
  ```python
  except DatabaseError as e:
      logger.error(f"Database error: {e}", extra={"user_id": user.id})
      raise HTTPException(status_code=500, detail="Failed to create order")
  ```
- **Validation errors:** Pydantic handles automatically (422 response)

### Performance
- **Pagination:** Add to list endpoints
  ```python
  @router.get("/orders")
  async def list_orders(
      page: int = Query(1, ge=1),
      limit: int = Query(20, ge=1, le=100),
      service: OrderService = Depends(get_order_service)
  ):
      return await service.list_orders(page, limit)
  ```
- **Database:** Use indexes, avoid N+1 queries
- **Caching:** Redis for read-heavy endpoints
  ```python
  @cache(ttl=300)  # 5 minutes
  async def get_order_summary(order_id: UUID):
      pass
  ```
- **Async:** Use `async def` for I/O-bound operations

### Testing
- **pytest:** Unit tests for services, integration tests for endpoints
  ```python
  def test_create_order_with_valid_data__SF_001(client, auth_headers):
      response = client.post("/orders", json={
          "items": [{"product_id": "123", "quantity": 2}],
          "total_amount": 49.99
      }, headers=auth_headers)
      assert response.status_code == 201
      assert response.json()["id"] is not None
  ```
- **Coverage:** >80% for the slice
- **Mocking:** Use `pytest-mock` for external services

---

## Tests

Add **pytest** tests: happy path + main error cases for new/changed endpoints in the **current slice** (status codes, validation errors).

---

## Output

### Summary Format

```markdown
## Slice [N] API Implementation: [Slice Name]

### Endpoints Added/Modified
- POST /orders (create order)
- Schema: CreateOrderRequest, OrderResponse

### Services Added/Modified
- `order_service.py` (business logic)
- `order_repository.py` (database access)

### Acceptance Criteria Covered (API)
- ✅ SF-001: POST /orders creates order with items
- ✅ SF-005: Order stored in DynamoDB

### How to Verify
```bash
# Run unit tests
pytest tests/services/test_order_service.py

# Run integration tests
pytest tests/integration/test_orders_api.py

# Manual test
curl -X POST http://localhost:8000/orders \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"items": [{"product_id": "123", "quantity": 2}], "total_amount": 49.99}'
```

### Files Touched
- `app/routers/orders.py` (new)
- `app/services/order_service.py` (new)
- `app/repositories/order_repository.py` (new)
- `app/schemas/order.py` (new)
- `tests/integration/test_orders_api.py` (new)
- `tests/services/test_order_service.py` (new)

### OpenAPI Update
- Added POST /orders endpoint
- Added CreateOrderRequest schema
- Added OrderResponse schema

### Next Steps
1. Review API against acceptance criteria
2. Run tests: `pytest`
3. Update OpenAPI spec: `python -m app.main --generate-openapi`
4. Update `progress.md`: Mark Slice [N] complete
5. Deploy to staging → validate → deploy to production
```

---

## Traceability

Reference **`SF-…`** IDs in:
- **Code comments:** `# SF-001: Validate order items before persisting`
- **Test names:** `test_create_order_with_valid_items__SF_001()`
- **Commit messages:** `[Slice 1] Add create order endpoint (SF-001, SF-005)`

---

## Quality Checklist

Before finishing:

- [ ] Implemented **only** the current slice (not entire feature)
- [ ] **Type hints** on all functions
- [ ] **Pydantic schemas** for request/response
- [ ] **Input validation** with Pydantic validators
- [ ] **Authorization** with JWT validation
- [ ] **Error handling** with HTTPException
- [ ] **Tests** written (>80% coverage for slice)
- [ ] **No secrets** in code
- [ ] **OpenAPI** spec updated
- [ ] **Backward compatible** (or migration plan)
- [ ] **SF-…** IDs referenced in code/tests

---

## Example Invocation

```
@.cursor/agents/python_api_from_spec.md
@specfold/user-orders/spec.md
@specfold/user-orders/progress.md
@specfold/user-orders/repo_kb.md

Implement current slice API (Slice 1: Create Order Endpoint)
Use FastAPI with Pydantic schemas.
```
