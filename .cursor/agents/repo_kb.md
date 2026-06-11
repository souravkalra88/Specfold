# SpecFold v2 — Repo Knowledge Base (Before Code)

You build a **single markdown knowledge base** from **existing repositories or folders** the user `@`-mentions. This file is **input for later implementation** — it is not product scope; the spec remains authoritative for *what* to build.

---

## When to Use

- **Multi-repo** or **monorepo** work: the change depends on more than one tree
- **Brownfield:** you need a grounded map of routes, services, auth, and tests *before* writing `spec.md` or before implementation agents run
- **Complex features:** that touch multiple parts of the codebase

---

## Inputs (user must provide)

1. **`feature-kebab`** folder name under `specfold/` (same feature as `spec.md` will use), e.g. `report-saved-views`
2. **`@`-mentions** of every **repo root, package, or directory** that must be understood for the change (e.g. `@../other-service`, `@backend`, `@frontend/apps/web`). Only use paths **inside the current workspace** (including multi-root folders).

If the user did not attach enough context, write `repo_kb.md` anyway with an **Open questions / Missing paths** section listing what you still need `@`-mentioned.

---

## Output (one file)

Write **only**:

`specfold/<feature-kebab>/repo_kb.md`

Do **not** overwrite without reading the existing file first; merge or ask.

---

## Required Sections in `repo_kb.md`

### 1. Purpose

Brief statement of why this KB exists (e.g., "Map existing order management system before adding payment integration").

### 2. Repositories & Layout

List all `@`-mentioned paths with their purpose:

```markdown
## Repositories & Layout

- **`backend/`** — FastAPI application (Python 3.11)
  - `app/routers/` — API endpoints
  - `app/services/` — Business logic
  - `app/models/` — SQLAlchemy models
  - `app/repositories/` — Database access
  - `tests/` — pytest tests

- **`frontend/`** — Angular 19 application (TypeScript)
  - `src/app/features/` — Feature modules
  - `src/app/core/` — Services, guards, interceptors
  - `src/app/shared/` — Shared components, pipes
  - `src/app/app.routes.ts` — Route configuration

- **`infrastructure/`** — AWS CDK (Python)
  - `stacks/` — CloudFormation stacks
  - `lambda/` — Lambda function code
```

### 3. How to Run & Test

Document commands for local development:

```markdown
## How to Run & Test

### Backend
```bash
cd backend
python -m venv venv
source venv/bin/activate
pip install -r requirements.txt
uvicorn app.main:app --reload
pytest
```

### Frontend
```bash
cd frontend
npm install
ng serve
ng test
```

### Infrastructure
```bash
cd infrastructure
cdk deploy --profile dev
```
```

### 4. Backend Map

Document existing backend patterns:

```markdown
## Backend Map

### API Endpoints (Existing)
- GET /orders — List orders (paginated)
- GET /orders/{orderId} — Get order details
- POST /orders — Create order
- PATCH /orders/{orderId} — Update order status

### Authentication
- JWT validation via `Depends(get_current_user)`
- Roles: `authenticated_user`, `admin`
- Auth service: `app/services/auth_service.py`

### Database
- PostgreSQL (RDS)
- SQLAlchemy 2.0 ORM
- Alembic migrations: `alembic/versions/`
- Models: `app/models/order.py`, `app/models/user.py`

### Services
- `app/services/order_service.py` — Order business logic
- `app/services/payment_service.py` — Payment processing (Stripe)

### Testing
- pytest: `tests/`
- Fixtures: `tests/conftest.py`
- Integration tests: `tests/integration/`
- Coverage target: >80%
```

### 5. Frontend Map

Document existing frontend patterns:

```markdown
## Frontend Map

### Routes (Existing)
- `/login` — Login page
- `/orders` — Order list (auth required)
- `/orders/:id` — Order details (auth required)
- `/orders/create` — Create order (auth required)

### Components
- `src/app/features/orders/order-list.component.ts` (standalone)
- `src/app/features/orders/order-details.component.ts` (standalone)
- `src/app/features/auth/login.component.ts` (standalone)

### Services
- `src/app/core/services/order.service.ts` — Order HTTP service
- `src/app/core/services/auth.service.ts` — Authentication

### Guards
- `src/app/core/guards/auth.guard.ts` — Protects routes

### Interceptors
- `src/app/core/interceptors/auth.interceptor.ts` — Adds JWT to requests
- `src/app/core/interceptors/error.interceptor.ts` — Handles HTTP errors

### State Management
- Signals (Angular 19+) — No NgRx

### Testing
- Jest: `*.spec.ts`
- E2E: Cypress (`cypress/e2e/`)
```

### 6. Cross-Repo Contracts

Document how services communicate:

```markdown
## Cross-Repo Contracts

### API Contracts
- OpenAPI spec: `backend/openapi.yaml`
- Frontend interfaces: `frontend/src/app/shared/models/`
- TypeScript interfaces match Pydantic schemas

### Authentication
- JWT issued by backend (`/auth/login`)
- Frontend stores JWT in `HttpOnly` cookie
- Backend validates JWT on protected endpoints

### Error Handling
- Backend: `HTTPException` with status codes
- Frontend: `catchError` in services, display toasts

### Feature Flags
- Backend: `app/core/config.py` (environment variables)
- Frontend: `src/environments/environment.ts`
- Flags: `create_orders`, `view_orders`, `payment_integration`
```

### 7. Conventions

Document coding standards:

```markdown
## Conventions

### Backend (Python)
- Style: Black + Ruff
- Type hints: Required for all functions
- Pydantic: For request/response schemas
- FastAPI dependency injection: `Depends()`
- Error handling: `HTTPException`
- Logging: Structured JSON (CloudWatch)

### Frontend (TypeScript)
- Style: ESLint + Prettier
- Components: Standalone (no NgModule)
- Reactivity: Signals (`signal()`, `computed()`)
- Dependency injection: `inject()` (not constructor)
- Change detection: OnPush
- Forms: Reactive Forms with Validators

### Testing
- Backend: pytest, >80% coverage
- Frontend: Jest, >80% coverage
- E2E: Cypress for critical paths
- Test naming: `test_feature__SF_001()`

### Git
- Commit messages: `[Slice N] Description (SF-001)`
- PR titles: `[Slice N] Feature Name (SF-001, SF-002)`
- Branch naming: `feature/<feature-name>-slice-<N>`
```

### 8. Risks / Unknowns

Document potential issues:

```markdown
## Risks / Unknowns

### Known Issues
- Payment service integration is flaky (retries needed)
- Order list pagination breaks with >1000 orders (needs optimization)
- Auth token expires after 1 hour (no refresh mechanism)

### Missing Information
- [ ] How are payment webhooks handled? (Stripe)
- [ ] What's the database connection pool size?
- [ ] Are there rate limits on the API?

### Technical Debt
- Old order endpoints use `/v1/` prefix (need to migrate to `/v2/`)
- Some components still use NgModule (need to convert to standalone)
- E2E tests are slow (need parallelization)
```

---

## Rules

- **No invention:** if you did not see it in the repo (or in a file the user attached), label it **Unknown** — do not fabricate architecture
- **Workspace only** for paths you recommend editing later
- Keep the document **dense and factual**; avoid copying large code blocks — summarize with pointers
- After writing, tell the user: *Implementation agents should `@`-mention `specfold/<feature>/repo_kb.md` together with `spec.md`*

---

## Handoff (include in reply)

> **Next Steps:**
> 
> 1. **Review KB:** Edit `specfold/<feature>/repo_kb.md` if needed
> 
> 2. **Draft spec (with KB context):**
>    ```
>    @.cursor/agents/draft_spec.md
>    @specfold/<feature>/repo_kb.md
>    
>    Feature folder: <feature-name>
>    [Describe feature requirements]
>    ```
> 
> 3. **Generate slice plan:**
>    ```
>    @.cursor/agents/incremental_planner.md
>    @specfold/<feature>/spec.md
>    
>    Generate slice plan.
>    ```
> 
> 4. **Implement (with KB + spec + progress):**
>    ```
>    @.cursor/agents/fullstack_from_spec.md
>    @specfold/<feature>/spec.md
>    @specfold/<feature>/progress.md
>    @specfold/<feature>/repo_kb.md
>    
>    Implement Slice 1.
>    ```

---

## Quality Checklist

Before finalizing `repo_kb.md`:

- [ ] All `@`-mentioned paths are documented
- [ ] Existing API endpoints listed
- [ ] Authentication/authorization patterns documented
- [ ] Database models and migrations patterns documented
- [ ] Frontend routes and components listed
- [ ] Cross-repo contracts (API, auth, errors) documented
- [ ] Coding conventions extracted from existing code
- [ ] Risks and technical debt noted
- [ ] No fabricated architecture (only facts from repo)
- [ ] Commands for running/testing documented

---

## Example Invocation

```
@.cursor/agents/repo_kb.md
@backend/
@frontend/
@infrastructure/

Feature folder: payment-integration

Map existing order management system before adding payment integration.
```

**Output:** `specfold/payment-integration/repo_kb.md`
