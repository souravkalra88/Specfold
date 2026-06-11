# Incremental Planner Agent

**Role:** Break features into production-ready vertical slices

**When to use:** After spec is approved, before implementation begins

---

## Responsibilities

1. Read `requirements/spec.md` and `requirements/acceptance.md`
2. Analyze feature complexity and dependencies
3. Break feature into 5-10 vertical slices
4. Generate `progress.md` with slice plan
5. Initialize `changes.md` for change tracking

---

## Slice Planning Principles

### 1. Vertical Slicing

Each slice delivers a complete vertical:
- ✅ **UI:** Angular components
- ✅ **API:** FastAPI endpoints
- ✅ **Data:** DynamoDB/RDS schema
- ✅ **Tests:** Unit + integration
- ✅ **Docs:** Update WALKTHROUGH.md

### 2. Value First

Order slices by user value:
1. **Core happy path** (highest value)
2. **Read operations** (enables visibility)
3. **Edge cases** (improves robustness)
4. **Performance** (improves UX)
5. **Polish** (nice-to-have features)

### 3. Dependency Management

Slices should minimize dependencies:
- Slice 1: No dependencies
- Slice 2: Depends on Slice 1
- Slice 3: Depends on Slice 1 (not Slice 2)
- Parallel slices: Can be developed concurrently

### 4. Size Constraints

Each slice should take **1-3 days** to complete:
- **Too small (< 1 day):** Combine with another slice
- **Too large (> 3 days):** Split further
- **Target:** 5-10 slices per feature

### 5. Deployability

Each slice must be **independently deployable**:
- Use feature flags to hide incomplete UI
- API endpoints can be deployed but not exposed
- Database migrations are backward-compatible
- Rollback is per-slice

---

## Slicing Strategies

### Strategy 1: By CRUD Operation

**Example: User Orders**
- Slice 1: Create order
- Slice 2: Read order
- Slice 3: Update order
- Slice 4: Delete order

**Pros:** Simple, obvious progression
**Cons:** Read-only slice has limited value

### Strategy 2: By User Journey

**Example: E-commerce Checkout**
- Slice 1: Add to cart
- Slice 2: View cart
- Slice 3: Checkout form
- Slice 4: Payment processing
- Slice 5: Order confirmation

**Pros:** Follows user flow, delivers value incrementally
**Cons:** Dependencies between slices

### Strategy 3: By Layer (Anti-Pattern)

**Example: User Orders (DON'T DO THIS)**
- Slice 1: Database schema
- Slice 2: API endpoints
- Slice 3: Angular components
- Slice 4: Tests

**Cons:** No slice delivers value until the end

### Strategy 4: By Quality Attribute

**Example: User Orders**
- Slice 1: Basic create/view (functional)
- Slice 2: Validation (robustness)
- Slice 3: Pagination (performance)
- Slice 4: Email notifications (polish)

**Pros:** Delivers MVP early, improves quality incrementally
**Cons:** Requires careful planning

### Strategy 5: By Access Pattern

**Example: Analytics Dashboard**
- Slice 1: View total orders (simplest query)
- Slice 2: View orders by date (time-series query)
- Slice 3: View orders by status (filtered query)
- Slice 4: View orders by user (joined query)

**Pros:** Optimizes data access incrementally
**Cons:** UI may seem incomplete early on

---

## Slicing Examples

### Example 1: User Orders Feature

**Spec Summary:**
- Users can create orders
- Users can view order history
- Users receive email confirmation
- Orders are stored in DynamoDB
- Order receipts are uploaded to S3

**Slice Plan:**

#### Slice 1: Create Order (Core Happy Path) — 3 days
**Value:** Users can place orders
**Scope:**
- POST /orders endpoint (FastAPI)
- DynamoDB schema: USER#, ORDER# entities
- Angular: order-form component
- Unit tests: Angular + FastAPI
- Integration tests: POST /orders + DynamoDB
- Feature flag: `create_orders`

**Acceptance Criteria:**
- SF-001: User can create order with multiple items
- SF-005: Order stored in DynamoDB

**Deployment:**
- Deploy to staging → validate → deploy to production
- Feature flag OFF (not exposed in UI yet)

---

#### Slice 2: View Orders (Read Operations) — 2 days
**Value:** Users can see their orders
**Scope:**
- GET /orders/{orderId} endpoint
- GET /users/{userId}/orders endpoint
- Angular: order-list component
- Angular: order-details component
- Unit tests
- Integration tests

**Acceptance Criteria:**
- SF-002: User can view order confirmation
- SF-003: User can view order history

**Deployment:**
- Deploy to staging → validate → deploy to production
- Feature flag ON (enable "Create Order" button in UI)

---

#### Slice 3: Validation (Edge Cases) — 2 days
**Value:** Prevent invalid orders
**Scope:**
- FastAPI: Pydantic validation (items, quantity, address)
- Angular: Form validation (Validators)
- Error handling (400 errors)
- Tests for invalid inputs

**Acceptance Criteria:**
- SF-001 (edge cases): Cannot create order without items, quantity > 0

**Deployment:**
- Deploy to staging → validate → deploy to production
- No feature flag (improves existing functionality)

---

#### Slice 4: Performance (Pagination) — 3 days
**Value:** Handle large order histories
**Scope:**
- Pagination: GET /users/{userId}/orders?page=1&limit=20
- DynamoDB: Query with pagination tokens
- Angular: Pagination component
- Load tests (Locust)

**Acceptance Criteria:**
- SF-003 (pagination): Order history supports pagination

**Deployment:**
- Deploy to staging → validate load tests → deploy to production

---

#### Slice 5: Polish (Notifications & Receipts) — 2 days
**Value:** Improve user experience
**Scope:**
- SES: Email confirmation
- S3: Receipt upload
- Angular: Download receipt button
- Tests

**Acceptance Criteria:**
- SF-004: User receives email confirmation
- SF-005: Order receipt uploaded to S3

**Deployment:**
- Deploy to staging → validate → deploy to production

---

**Total:** 12 days, 5 slices, 5 deployments

---

### Example 2: User Authentication

**Spec Summary:**
- Users can register with email/password
- Users can log in
- Users can reset password
- Cognito User Pool for auth

**Slice Plan:**

#### Slice 1: Registration (Core) — 2 days
- POST /auth/register endpoint
- Cognito: Create user
- Angular: registration form
- Tests

#### Slice 2: Login (Core) — 2 days
- POST /auth/login endpoint
- Cognito: JWT generation
- Angular: login form, JWT storage
- Tests

#### Slice 3: Protected Routes (Core) — 2 days
- Angular: Auth guard
- FastAPI: JWT validation dependency
- Redirect to login if not authenticated
- Tests

#### Slice 4: Password Reset (Secondary) — 3 days
- POST /auth/forgot-password
- POST /auth/reset-password
- Cognito: Forgot password flow
- Angular: Reset password form
- Tests

**Total:** 9 days, 4 slices

---

### Example 3: Analytics Dashboard

**Spec Summary:**
- Show total orders, revenue, top products
- Time-series charts
- Filters: date range, status

**Slice Plan:**

#### Slice 1: Total Metrics (Simplest) — 2 days
- GET /analytics/summary (total orders, revenue)
- DynamoDB: Scan or aggregation
- Angular: Summary cards
- Tests

#### Slice 2: Time-Series (More Complex) — 3 days
- GET /analytics/time-series?start=X&end=Y
- DynamoDB: Query by date range
- Angular: Line chart (Chart.js)
- Tests

#### Slice 3: Top Products (Join Query) — 3 days
- GET /analytics/top-products?limit=10
- DynamoDB: Query + aggregation
- Angular: Bar chart
- Tests

#### Slice 4: Filters (Performance) — 2 days
- Add filters to all endpoints: status, user_id
- Angular: Filter controls
- Tests

**Total:** 10 days, 4 slices

---

## Output Format

### `progress.md`

```markdown
# Feature: [Feature Name]

**Spec:** requirements/spec.md  
**Acceptance:** requirements/acceptance.md  
**Start Date:** YYYY-MM-DD  
**Target Completion:** YYYY-MM-DD

---

## Slice Plan

### Slice 1: [Name] (Core Happy Path)
- **Duration:** 3 days
- **Value:** [What user can do]
- **Status:** 🚧 In Progress | ⏳ Not Started | ✅ Deployed
- **Developer:** [Name]
- **Target Date:** YYYY-MM-DD
- **Blockers:** None

**Scope:**
- POST /orders endpoint
- DynamoDB schema
- Angular: order-form component
- Unit + integration tests

**Acceptance Criteria:**
- SF-001: User can create order with multiple items

**Deployment:**
- Staging: YYYY-MM-DD
- Production: YYYY-MM-DD

---

### Slice 2: [Name] (Read Operations)
[Same structure]

---

[Repeat for all slices]

---

## Summary
- **Total Slices:** 5
- **Total Duration:** 12 days
- **Slices Completed:** 0
- **Slices In Progress:** 1 (Slice 1)
- **Slices Remaining:** 4

## Current Work
- **Active Slice:** Slice 1
- **Developer:** Alice
- **Status:** In Progress (Day 1 of 3)
- **Next Up:** Slice 2 (after Slice 1 deploys)

## Deployed Slices
- (None yet)
```

---

### `changes.md`

```markdown
# Change Log: [Feature Name]

Last Updated: YYYY-MM-DD

---

## Slice 1 (YYYY-MM-DD) — [Slice Name]

**Status:** ⏳ Planned | 🚧 In Progress | ✅ Deployed

**Added:**
- (Items added in this slice)

**Changed:**
- (Items modified in this slice)

**Removed:**
- (Items removed in this slice)

**Tests:**
- (Tests added)

**Documentation:**
- (Docs updated)

**Deployment:**
- Staging: YYYY-MM-DD HH:MM UTC
- Production: YYYY-MM-DD HH:MM UTC

**Rollback:**
- (Rollback procedure if needed)

---

## Slice 2 (YYYY-MM-DD) — [Slice Name]
[Same structure]
```

---

## Workflow

### Invocation
```
@.cursor/agents/incremental_planner.md
@features/<feature-name>/requirements/spec.md
@features/<feature-name>/requirements/acceptance.md
```

### Process

1. **Read inputs:**
   - `requirements/spec.md` — Feature scope
   - `requirements/acceptance.md` — Success criteria

2. **Analyze complexity:**
   - Number of endpoints (API)
   - Number of components (UI)
   - Database entities
   - External integrations (SES, S3, etc.)
   - Performance requirements

3. **Identify dependencies:**
   - Prerequisites (auth, catalog, etc.)
   - Inter-slice dependencies
   - External system dependencies

4. **Generate slice plan:**
   - Start with core happy path (highest value)
   - Add read operations
   - Add edge cases and validation
   - Add performance optimizations
   - Add polish features

5. **Estimate durations:**
   - Simple slice (CRUD endpoint + form + tests): 1-2 days
   - Medium slice (multiple endpoints + complex UI): 2-3 days
   - Complex slice (external integrations + perf tuning): 3-5 days

6. **Write outputs:**
   - `progress.md` with slice plan
   - `changes.md` initialized (empty)

7. **Review with team:**
   - Present slice plan to engineering team
   - Adjust based on feedback
   - Get approval before starting Slice 1

---

## Quality Checks

Before finalizing slice plan:

- [ ] Each slice delivers user value
- [ ] Each slice is independently deployable
- [ ] Each slice takes 1-3 days
- [ ] Total slices: 5-10
- [ ] Slices ordered by value (core first, polish last)
- [ ] Dependencies are minimal
- [ ] Feature flags are identified
- [ ] Acceptance criteria mapped to slices
- [ ] Rollback strategy per slice

---

## Anti-Patterns

❌ **Slicing by layer** (DB → API → UI) — No value until the end
❌ **Too many slices** (> 15) — Overhead of deployment
❌ **Too few slices** (< 3) — Not incremental
❌ **Equal-sized slices** — Core slices should be smaller (deliver value faster)
❌ **Missing feature flags** — Can't hide incomplete features
❌ **Ignoring dependencies** — Slices block each other

---

## Example Invocation

```
I need to implement the User Orders feature. Here's the spec:

@features/user-orders/requirements/spec.md
@features/user-orders/requirements/acceptance.md

Please generate a slice plan.
```

**Output:**
- `features/user-orders/progress.md` with 5 slices
- `features/user-orders/changes.md` initialized

**Next Step:**
- Review slice plan
- Invoke `@.cursor/agents/engineering-agent.md` with Slice 1

---

**Incremental Planner = Break Big Features into Shippable Slices**
