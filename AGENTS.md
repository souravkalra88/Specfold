# SpecFold — Agent Map

Invoke by `@`-mentioning the path in Cursor chat.

---

## Flow: SpecFold v2 Incremental Workflow

### Phase 1: Requirements & Planning

1. **(Optional) Repo KB** — [`.cursor/agents/repo_kb.md`](.cursor/agents/repo_kb.md) builds **`specfold/<feature>/repo_kb.md`** from every **`@`-mentioned repo/folder** you need. Do this **before** or **alongside** drafting the spec when multiple trees or brownfield context matters.

2. **Draft Spec** — [`.cursor/agents/draft_spec.md`](.cursor/agents/draft_spec.md) creates **`specfold/<feature>/spec.md`** (and optional `plan.md`). You may `@`-mention `repo_kb.md` so constraints match reality.

3. **Slice Planning** — [`.cursor/agents/incremental_planner.md`](.cursor/agents/incremental_planner.md) breaks the feature into **5-10 vertical slices** (UI + API + Data + Tests per slice). Generates **`progress.md`** and initializes **`changes.md`**.

### Phase 2: Implementation (Per Slice)

4. **Implement ONE Slice** — attach **`@specfold/<feature>/spec.md`**, **`@specfold/<feature>/progress.md`** (to identify current slice), and, if it exists, **`@specfold/<feature>/repo_kb.md`**, then invoke an implement agent below.

5. **Update Tracking** — after implementing a slice, mark it complete in **`progress.md`**, add entry to **`changes.md`**.

6. **Deploy & Validate** — deploy slice to staging → production, then repeat for next slice.

---

## Available Agents

### Planning & Design Agents

| Agent | When to use | Output |
|-------|-------------|--------|
| [`.cursor/agents/repo_kb.md`](.cursor/agents/repo_kb.md) | Build **`specfold/<feature>/repo_kb.md`** — condensed **existing codebase** map from **`@`-mentioned** repos/paths (multi-repo / brownfield **before** code). | `repo_kb.md` |
| [`.cursor/agents/draft_spec.md`](.cursor/agents/draft_spec.md) | Turn an idea into **`specfold/<feature>/spec.md`** (spec generation only). Optional: `plan.md`, `acceptance.md`. | `spec.md`, `plan.md` |
| [`.cursor/agents/incremental_planner.md`](.cursor/agents/incremental_planner.md) | Break feature into **5-10 vertical slices** (UI + API + Data + Tests). Reads `spec.md` and `acceptance.md`. | `progress.md`, `changes.md` |

### Implementation Agents (Pick ONE per slice)

| Agent | When to use | Stack | Output |
|-------|-------------|-------|--------|
| [`.cursor/agents/ship_from_spec.md`](.cursor/agents/ship_from_spec.md) | Generic implement + tests from spec (any stack); reads **`repo_kb.md`** when present. Best for **single-language** or **simple** slices. | Any | Code + Tests |
| [`.cursor/agents/angular_from_spec.md`](.cursor/agents/angular_from_spec.md) | Angular UI, routes, forms, services, standalone components from spec. Use for **UI-only** slices. | Angular 19+ | Components + Tests |
| [`.cursor/agents/python_api_from_spec.md`](.cursor/agents/python_api_from_spec.md) | FastAPI routers, Pydantic schemas, thin handlers, OpenAPI spec from spec. Use for **API-only** slices (incl. APIs deployed on **ECS/Lambda**). | FastAPI | Routers + OpenAPI + Tests |
| [`.cursor/agents/python_aws_from_spec.md`](.cursor/agents/python_aws_from_spec.md) | **AWS Python**: Lambda functions, SQS/SNS/EventBridge consumers, boto3 services, DynamoDB/S3, Python CDK / SAM-adjacent code from spec. Use for **infrastructure** or **async processing** slices. | AWS SDK, CDK, SAM | Lambdas + IaC + Tests |
| [`.cursor/agents/python_persistence_from_spec.md`](.cursor/agents/python_persistence_from_spec.md) | SQLAlchemy models, queries, Alembic migrations from spec. Use for **data model** slices. | SQLAlchemy, Alembic | Models + Migrations + Tests |
| [`.cursor/agents/fullstack_from_spec.md`](.cursor/agents/fullstack_from_spec.md) | **One vertical slice**: Python API + Angular UI together. Use when slice spans **both** UI and API. | Angular + FastAPI | Full slice (UI + API + Tests) |

---

## Implementation Agent Selection Guide

### Slice Type → Agent Mapping

| Slice Description | Agent to Use |
|-------------------|--------------|
| "Add create order form + POST /orders endpoint" | `fullstack_from_spec.md` (UI + API) |
| "Add order-list component + GET /orders endpoints" | `fullstack_from_spec.md` (UI + API) |
| "Add input validation to create order form" | `angular_from_spec.md` (UI-only) |
| "Add pagination to GET /orders endpoint" | `python_api_from_spec.md` (API-only) |
| "Add Lambda to process order events from SQS" | `python_aws_from_spec.md` (AWS-only) |
| "Add Order and OrderItem SQLAlchemy models" | `python_persistence_from_spec.md` (Data-only) |
| "Add email notifications via SES + S3 receipts" | `python_aws_from_spec.md` (AWS-only) |
| "Add authentication guard to order routes" | `angular_from_spec.md` (UI-only) |

### Production-Grade Rules (ALL Agents Must Follow)

1. **Scope Discipline**
   - Do NOT expand scope beyond `spec.md` / `acceptance.md`
   - If something is missing, **stop and ask** before implementing
   - List **questions** and **assumptions** before large changes
   - Reference **`SF-…`** IDs in code comments and commit messages

2. **Backward Compatibility**
   - **Prefer additive changes** (new endpoints, new fields, new components)
   - **Avoid breaking changes** (removing endpoints, renaming fields, changing response shapes)
   - Version APIs when breaking changes are unavoidable (`/v1/orders` → `/v2/orders`)
   - Deprecate old behavior with **30-day migration window**
   - Support both old and new during transition period

3. **Contract Alignment**
   - Keep **HTTP handlers, Pydantic schemas, and OpenAPI spec** aligned when APIs change
   - Update OpenAPI spec (`api-spec.yaml`) whenever endpoints change
   - Validate OpenAPI spec in CI (use `@redocly/cli lint`)
   - Keep TypeScript interfaces in sync with Pydantic schemas

4. **Testing Requirements**
   - **Unit tests:** >80% coverage per slice
   - **Integration tests:** Cover happy path + edge cases per slice
   - **E2E tests:** Cover acceptance criteria (`SF-…`) per slice
   - Test file naming: `test_*.py` (pytest), `*.spec.ts` (Angular)
   - Use pytest fixtures for database/API setup
   - Use Angular TestBed for component tests

5. **Security Baseline (OWASP)**
   - **Input validation:** Pydantic for FastAPI, Angular Reactive Forms for UI
   - **Output encoding:** Prevent XSS (Angular sanitizes by default, FastAPI returns JSON)
   - **Secrets:** No secrets in code; use AWS Secrets Manager / Parameter Store
   - **Authorization:** Server-side authz (verify JWT in FastAPI, not just Angular)
   - **Error handling:** Fail-safe errors without leaking internal details (use `HTTPException(status_code=400, detail="Invalid input")`)
   - **Dependencies:** Pin versions in `pyproject.toml` / `package.json`

6. **Incremental Deployment**
   - Each slice is **independently deployable**
   - Use **feature flags** to hide incomplete features:
     ```typescript
     // Angular
     if (featureFlags.viewOrders) { /* show UI */ }
     ```
     ```python
     # FastAPI
     if feature_flags["view_orders"]: # enable endpoint
     ```
   - Deploy to **staging** after every slice
   - Deploy to **production** after validation (1-2 times per week)
   - Use Lambda aliases/versions for gradual rollout

7. **Documentation Updates**
   - Update `progress.md` after every slice (mark complete, update status)
   - Update `changes.md` after every slice (add entry with deployment date)
   - Update `WALKTHROUGH.md` (if using SpecFlow v2) after production deploy
   - Update `api-inventory.md` when APIs change
   - Update `database-inventory.md` when schema changes

8. **Code Quality**
   - **Angular:** Standalone components, signals, inject(), OnPush change detection
   - **FastAPI:** Thin routers, fat services, dependency injection via `Depends()`
   - **AWS:** Use CDK L2 constructs, avoid raw CloudFormation
   - **Python:** Type hints everywhere, Ruff for linting, Black for formatting
   - **TypeScript:** Strict mode enabled, ESLint + Prettier

9. **Error Handling**
   - **FastAPI:** Use `HTTPException` for user errors, log server errors
   - **Angular:** Use `catchError` in services, display user-friendly messages in UI
   - **AWS Lambda:** Use structured logging (JSON), include correlation IDs
   - **Database:** Use transactions, handle deadlocks/conflicts

10. **Performance**
    - **API:** Add pagination for list endpoints (default page size: 20-50)
    - **Database:** Add indexes for query patterns (GSI for DynamoDB, indexes for RDS)
    - **UI:** Use lazy loading for routes, virtual scrolling for large lists
    - **AWS:** Set Lambda memory/timeout appropriately, use provisioned concurrency if needed

---

## Rules (Auto-Applied + Stack-Specific)

| Rule | Scope | When Applied |
|------|-------|--------------|
| [`.cursor/rules/specfold.mdc`](.cursor/rules/specfold.mdc) | Always on — specs, optional **`repo_kb.md`**, scope, `SF-…`, security baseline, legacy workflow | All files |
| [`.cursor/rules/specfold-v2.mdc`](.cursor/rules/specfold-v2.mdc) | **SpecFold v2** — incremental slicing, `progress.md`, `changes.md`, backward compatibility, deployment per slice | All files |
| [`.cursor/rules/angular.mdc`](.cursor/rules/angular.mdc) | Angular / TypeScript / templates — standalone components, signals, OnPush, testing | `*.ts`, `*.html`, `*.scss`, `angular.json` |
| [`.cursor/rules/python.mdc`](.cursor/rules/python.mdc) | Python, FastAPI, ORM, Alembic, **AWS** (boto3, Lambda, messaging, IaC globs) — type hints, Pydantic, dependency injection | `*.py`, `pyproject.toml`, `alembic/`, `cdk.json` |

---

## Example: Implementing "User Orders" Feature

### Step 1: Generate Spec

```text
@.cursor/agents/draft_spec.md

Feature folder: user-orders

I need a user orders feature where users can:
- Create orders (POST /orders)
- View their orders (GET /users/{userId}/orders)
- View order details (GET /orders/{orderId})

Acceptance:
- SF-001: Users can create orders with items, quantities, total
- SF-002: Users can view their order history
- SF-003: Users can view details of a single order
```

**Output:** `specfold/user-orders/spec.md`

---

### Step 2: Generate Slice Plan

```text
@.cursor/agents/incremental_planner.md
@specfold/user-orders/spec.md

Generate slice plan for user orders feature.
```

**Output:** `specfold/user-orders/progress.md` (5 slices), `specfold/user-orders/changes.md`

---

### Step 3: Implement Slice 1 (Create Order)

```text
@.cursor/agents/fullstack_from_spec.md
@specfold/user-orders/spec.md
@specfold/user-orders/progress.md

Implement Slice 1: Create order (POST /orders + form + tests)
```

**Output:** 
- `ui/order-form.component.ts` (Angular)
- `api/routers/orders.py` (FastAPI POST /orders)
- `backend/services/order_service.py`
- `tests/unit/test_order_form.spec.ts`
- `tests/integration/test_orders_api.py`

---

### Step 4: Update Tracking

Mark Slice 1 complete in `progress.md`, add entry to `changes.md`:

```markdown
## Slice 1 (2024-06-11) — Create Order
**Added:**
- POST /orders endpoint
- Angular: order-form component
- Unit + integration tests

**Deployed:**
- Staging: 2024-06-11 14:00 UTC
- Production: 2024-06-12 10:00 UTC
```

---

### Step 5: Deploy & Validate

```bash
# Deploy to staging
git push origin main

# Run smoke tests
cd deployment
./smoke-tests.sh staging

# Deploy to production (after validation)
git tag v1.0.0-slice-1
git push origin v1.0.0-slice-1
```

---

### Step 6: Implement Slice 2 (View Orders)

```text
@.cursor/agents/fullstack_from_spec.md
@specfold/user-orders/spec.md
@specfold/user-orders/progress.md

Implement Slice 2: View orders (GET endpoints + list + tests)
```

**Repeat Steps 3-5 for each slice.**

---

## Anti-Patterns to Avoid

| ❌ Don't | ✅ Do |
|---------|------|
| Implement entire feature at once | Break into 5-10 slices, implement one at a time |
| Slice by layer (DB → API → UI) | Slice vertically (DB + API + UI per slice) |
| Deploy multiple slices together | Deploy after each slice |
| Make breaking changes without migration plan | Version APIs, support old + new for 30 days |
| Skip tests | Write tests with every slice (>80% coverage) |
| Forget to update docs | Update `progress.md`, `changes.md`, `WALKTHROUGH.md` after every slice |
| Hardcode feature flags in code | Use config service or environment variables |
| Invent requirements not in spec | Stop and ask if spec is incomplete |

---

## Traceability

Reference **`SF-…`** IDs in:
- **Commit messages:** `[Slice 1] Add create order endpoint (SF-001)`
- **PR titles:** `[Slice 1] Create order form + API (SF-001)`
- **Code comments:** `# SF-001: Validate order items before saving`
- **Test names:** `test_create_order_with_valid_items__SF_001()`

---

## Resources

- [README.md](README.md) — Quick start guide
- [SPECFLOW_V2.md](SPECFLOW_V2.md) — Full v2 framework
- [IMPLEMENTATION_GUIDE.md](IMPLEMENTATION_GUIDE.md) — Detailed implementation workflow
- [SPECIFY_DEMO_WALKTHROUGH.md](SPECIFY_DEMO_WALKTHROUGH.md) — Step-by-step demo
