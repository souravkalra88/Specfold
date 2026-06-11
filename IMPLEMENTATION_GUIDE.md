# SpecFlow v2: Implementation Guide

**Version:** 2.0  
**Date:** 2024-06-11  
**Stack:** Angular + Python FastAPI + AWS Serverless

---

## Quick Start

### 1. Read the Framework
Start here: [`SPECFLOW_V2.md`](SPECFLOW_V2.md) — Full framework overview

### 2. Understand Incremental Delivery
Read: [`.cursor/rules/specfold-v2.mdc`](.cursor/rules/specfold-v2.mdc) — Incremental change workflow

### 3. Learn the Agents
- [`.cursor/agents/incremental_planner.md`](.cursor/agents/incremental_planner.md) — Slice planning
- [`.cursor/agents/ship_from_spec.md`](.cursor/agents/ship_from_spec.md) — Generic implementation
- [`.cursor/agents/fullstack_from_spec.md`](.cursor/agents/fullstack_from_spec.md) — Fullstack vertical slices
- [`.cursor/agents/angular_from_spec.md`](.cursor/agents/angular_from_spec.md) — Angular UI
- [`.cursor/agents/python_api_from_spec.md`](.cursor/agents/python_api_from_spec.md) — FastAPI backend
- [`.cursor/agents/python_aws_from_spec.md`](.cursor/agents/python_aws_from_spec.md) — AWS infrastructure
- [`.cursor/agents/python_persistence_from_spec.md`](.cursor/agents/python_persistence_from_spec.md) — Database layer
- See [AGENTS.md](AGENTS.md) for full agent map and production-grade rules

### 4. Review Governance
- [Definition of Ready](.governance/definition-of-ready.md)
- [Definition of Done](.governance/definition-of-done.md)
- [Quality Gates](.governance/quality-gates.md)

---

## Feature Lifecycle (Step-by-Step)

### Phase 1: Requirements

**Goal:** Get spec approved

**Steps:**
1. Create feature folder: `specflow-v2/features/<feature-name>/`
2. Copy template: `templates/feature/requirements/spec.md` → `features/<feature-name>/requirements/spec.md`
3. Fill in spec (Problem, Goals, Scope, APIs, Acceptance)
4. Copy template: `templates/feature/requirements/acceptance.md` → `features/<feature-name>/requirements/acceptance.md`
5. Write Gherkin scenarios
6. Run Definition of Ready checklist
7. Get Product Owner approval
8. Get Tech Lead approval

**Output:** Approved `requirements/spec.md` + `requirements/acceptance.md`

**Time:** 1-2 days

---

### Phase 2: Design

**Goal:** Get architecture approved

**Steps:**
1. Invoke appropriate implementation agent:
   ```
   @.cursor/agents/fullstack_from_spec.md
   @features/<feature-name>/requirements/spec.md
   
   Generate design artifacts and implementation plan for this feature.
   ```

2. For complex features, consider creating design documents:
   - `design/architecture.md` — Component diagram, data flow
   - `design/api-spec.yaml` — OpenAPI 3.1
   - `design/data-model.md` — DynamoDB/RDS schema
   - `design/erd.mmd` — Entity relationships (if RDS)
   - `security/threat-model.md` — STRIDE analysis
   - `design/adr/` — Architecture Decision Records

3. Review design artifacts
4. Run Quality Gate 2
5. Get engineering team approval
6. Get security lead approval (if needed)

**Output:** Approved design artifacts

**Time:** 2-3 days

---

### Phase 3: Slice Planning

**Goal:** Break feature into shippable slices

**Steps:**
1. Invoke Incremental Planner:
   ```
   @.cursor/agents/incremental_planner.md
   @features/<feature-name>/requirements/spec.md
   @features/<feature-name>/requirements/acceptance.md
   
   Generate slice plan.
   ```

2. Agent generates:
   - `progress.md` — 5-10 slices with estimates
   - `changes.md` — Initialized for tracking

3. Review slice plan with team
4. Adjust slices based on feedback
5. Approve plan

**Output:** `progress.md` with slice plan

**Time:** 1 day

---

### Phase 4: Implementation (Per Slice)

**Goal:** Implement one vertical slice

**Steps per slice:**

1. **Read context:**
   ```
   @.cursor/agents/fullstack_from_spec.md
   @features/<feature-name>/requirements/spec.md
   @features/<feature-name>/progress.md
   @features/<feature-name>/design/
   
   Implement Slice [N]: [Name]
   ```

2. **Agent implements:**
   - `ui/` — Angular components
   - `api/` — FastAPI endpoints
   - `backend/` — Services, repositories
   - `infrastructure/` — CDK/Terraform
   - `tests/unit/` — Unit tests (>80% coverage)
   - `tests/integration/` — Integration tests

3. **Run tests:**
   ```bash
   # Angular
   cd ui && npm run test
   
   # FastAPI
   cd api && pytest --cov
   ```

4. **Update tracking:**
   - Mark slice complete in `progress.md`
   - Add entry to `changes.md`
   - Invoke Documentation Agent to update `docs/WALKTHROUGH.md`

5. **Code review:**
   - Create PR with slice scope: `[Slice N] <description>`
   - Run Quality Gate 3
   - Get peer approval

**Output:** Working, tested, reviewed code for one slice

**Time per slice:** 1-3 days

---

### Phase 5: Testing (Per Slice)

**Goal:** Validate slice before staging

**Steps:**
1. Write E2E tests:
   ```
   @.cursor/agents/ship_from_spec.md
   @features/<feature-name>/requirements/acceptance.md
   
   Write E2E tests for Slice [N] acceptance criteria.
   ```

2. Run tests:
   ```bash
   # E2E
   cd tests/e2e && npm run cypress
   
   # Load tests (if needed)
   cd tests/load && locust -f locustfile.py
   ```

3. Run security scan:
   ```bash
   # Python
   bandit -r backend/
   
   # Angular
   npm audit
   ```

4. Run Quality Gate 4
5. Get QA sign-off

**Output:** Validated slice

**Time:** 1 day

---

### Phase 6: Deployment (Per Slice)

**Goal:** Deploy slice to staging, then production

**Steps:**

1. **Deploy to staging:**
   ```bash
   # Via CI/CD
   git push origin main  # Triggers staging deploy
   
   # Or manual
   cdk deploy --profile staging
   ```

2. **Run smoke tests:**
   ```bash
   cd deployment
   ./smoke-tests.sh staging
   ```

3. **Validate in staging:**
   - Check CloudWatch logs
   - Check X-Ray traces
   - Manual QA

4. **Run Quality Gate 5**
5. Get SRE/DevOps approval

6. **Deploy to production:**
   ```bash
   git tag v1.0.0-slice-1
   git push origin v1.0.0-slice-1  # Triggers prod deploy with approval
   ```

7. **Run smoke tests in production:**
   ```bash
   ./smoke-tests.sh production
   ```

8. **Monitor for 24 hours:**
   - CloudWatch alarms
   - Error rates
   - Performance metrics

9. **Update Documentation Agent:**
   ```
   # Manually update docs/WALKTHROUGH.md after production deploy
   # Or use a custom documentation agent if you've created one
   
   @specfold/<feature-name>/progress.md
   @specfold/<feature-name>/changes.md
   
   Update documentation with Slice [N] deployment.
   ```

**Output:** Slice deployed to production

**Time:** 1 day

---

### Phase 7: Operations (Ongoing)

**Goal:** Monitor and maintain

**Steps:**
1. Monitor CloudWatch dashboards
2. Respond to alarms
3. Track customer feedback
4. Log bugs in `docs/WALKTHROUGH.md` (Known Issues)
5. Log technical debt in `docs/technical-debt.md`

**Ongoing**

---

## Incremental Workflow Example

### Feature: User Orders

**Total:** 5 slices, 12 days, 5 deployments

---

#### Week 1: Slices 1-2

**Monday-Wednesday (Slice 1):**
1. Implement create order endpoint + form
2. Write tests
3. Code review
4. Deploy to staging Tuesday EOD
5. Deploy to production Wednesday EOD
6. **Deployed:** Users can create orders (feature flag OFF)

**Thursday-Friday (Slice 2):**
1. Implement view orders endpoints + components
2. Write tests
3. Code review
4. Deploy to staging Friday EOD
5. **Staged:** View orders ready for testing

---

#### Week 2: Slices 2-4

**Monday (Slice 2 continued):**
1. QA validation in staging
2. Deploy to production Monday EOD
3. **Deployed:** Enable feature flag, users can create and view orders

**Tuesday-Wednesday (Slice 3):**
1. Add validation (FastAPI + Angular)
2. Write tests
3. Code review
4. Deploy to staging Wednesday EOD
5. **Staged:** Validation ready

**Thursday (Slice 3 continued):**
1. Deploy to production Thursday EOD
2. **Deployed:** Improved validation (no UI changes)

**Friday (Slice 4):**
1. Start pagination implementation

---

#### Week 3: Slices 4-5

**Monday-Tuesday (Slice 4 continued):**
1. Complete pagination
2. Write tests
3. Code review
4. Deploy to staging Tuesday EOD
5. Deploy to production Wednesday EOD
6. **Deployed:** Pagination for large order histories

**Wednesday-Thursday (Slice 5):**
1. Email notifications (SES)
2. Receipt upload (S3)
3. Write tests
4. Code review
5. Deploy to staging Thursday EOD

**Friday (Slice 5 continued):**
1. Deploy to production Friday EOD
2. **Feature Complete:** All acceptance criteria met

---

**Result:** 12 working days, 5 production deployments, feature delivered incrementally

**Compare to waterfall:** 12 days, 1 deployment, delayed feedback, higher risk

---

## Agent Invocation Patterns

### Pattern 1: New Feature from Scratch

```
Step 1: Generate spec
@.cursor/agents/draft_spec.md
"I want to build a user authentication feature with Cognito"

Step 2: Generate slice plan
@.cursor/agents/incremental_planner.md
@features/user-auth/requirements/spec.md
"Generate slice plan"

Step 3: Implement Slice 1
@.cursor/agents/fullstack_from_spec.md
@features/user-auth/requirements/spec.md
@features/user-auth/progress.md
"Implement Slice 1"

Step 4: Update docs manually
# Update progress.md and changes.md after each slice
```

---

### Pattern 2: Brownfield Feature (Existing Codebase)

```
Step 1: Generate repo KB
@.cursor/agents/repo_kb.md
@existing-ui-folder/
@existing-api-folder/
@existing-backend-folder/
"Map existing codebase for user orders feature"

Step 2: Generate spec with constraints
@.cursor/agents/draft_spec.md
@features/user-orders/repo_kb.md
"Generate spec for user orders, considering existing code"

Step 3: Proceed as normal (design → slices → implement)
```

---

### Pattern 3: Incremental Change to Existing Feature

```
Step 1: Read progress
@features/user-orders/progress.md
@features/user-orders/changes.md
"What's the current state?"

Step 2: Implement next slice
@.cursor/agents/fullstack_from_spec.md
@features/user-orders/requirements/spec.md
@features/user-orders/progress.md
"Implement Slice 3"

Step 3: Update tracking
# Manually update progress.md to mark Slice 3 complete
# Add entry to changes.md with deployment date
```

---

## CI/CD Integration

### GitHub Actions Workflow

```yaml
# .github/workflows/feature-pr.yaml
name: Feature PR

on:
  pull_request:
    types: [opened, synchronize]
    paths:
      - 'features/**'

jobs:
  quality-gate-3:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      
      # Parse slice from PR title
      - name: Extract Slice Info
        id: slice
        run: |
          TITLE="${{ github.event.pull_request.title }}"
          SLICE=$(echo $TITLE | grep -oP '\[Slice \d+\]')
          echo "slice=$SLICE" >> $GITHUB_OUTPUT
      
      # Run tests
      - name: Unit Tests (Angular)
        run: |
          cd features/*/ui
          npm install
          npm run test:ci
      
      - name: Unit Tests (FastAPI)
        run: |
          cd features/*/api
          pip install -r requirements.txt
          pytest --cov=app --cov-fail-under=80
      
      # Validate OpenAPI
      - name: Validate OpenAPI Spec
        run: |
          npx @redocly/cli lint features/*/design/api-spec.yaml
      
      # Security scans
      - name: Security Scan (Python)
        run: bandit -r features/*/backend/
      
      - name: Security Scan (Angular)
        run: |
          cd features/*/ui
          npm audit --audit-level=high
      
      # Update progress tracking
      - name: Comment on PR
        uses: actions/github-script@v6
        with:
          script: |
            github.rest.issues.createComment({
              issue_number: context.issue.number,
              owner: context.repo.owner,
              repo: context.repo.repo,
              body: '✅ Quality Gate 3 passed for ${{ steps.slice.outputs.slice }}'
            })
```

---

## Best Practices

### 1. Keep Slices Small
- **Target:** 1-3 days per slice
- **Avoid:** > 500 lines changed per PR
- **Split:** If a slice grows beyond 3 days, split it further

### 2. Deploy Frequently
- **Staging:** After every slice (multiple times per week)
- **Production:** After validation (1-2 times per week)
- **Rollback:** Test rollback procedure in staging

### 3. Use Feature Flags
```typescript
// Angular
export const FEATURE_FLAGS = {
  createOrders: true,
  viewOrders: false,  // Not ready yet
  emailNotifications: false
};

if (FEATURE_FLAGS.viewOrders) {
  // Show "View Orders" button
}
```

```python
# FastAPI
FEATURE_FLAGS = {
    "create_orders": True,
    "view_orders": False,
    "email_notifications": False
}

if FEATURE_FLAGS["view_orders"]:
    # Enable GET /orders endpoints
```

### 4. Maintain Backward Compatibility
```
✅ Add new endpoint: POST /v2/orders
✅ Deprecate old endpoint: POST /v1/orders (support for 30 days)
✅ Add optional field: orderNotes
✅ Add new GSI: GSI2 for status queries

❌ Remove endpoint: DELETE /v1/orders
❌ Rename field: total → totalAmount
❌ Change response shape: items[] → orderItems[]
❌ Remove field: deliveryAddress
```

### 5. Update Docs Continuously
After every slice:
- Update `progress.md`
- Update `changes.md`
- Update `docs/WALKTHROUGH.md`
- Update `docs/api-inventory.md` (if API changes)
- Update `docs/database-inventory.md` (if schema changes)

### 6. Test Incrementally
- **Unit tests:** After every code change
- **Integration tests:** After every slice
- **E2E tests:** After every slice
- **Load tests:** Before production deploy
- **Security tests:** Before production deploy

### 7. Monitor Continuously
- **CloudWatch:** Logs, metrics, alarms
- **X-Ray:** Distributed tracing
- **Dashboards:** Per feature
- **Alerts:** Slack + PagerDuty

---

## Troubleshooting

### Problem: Slice is too large (> 5 days)

**Solution:** Split further
- Separate read from write operations
- Split UI into multiple components
- Split API into multiple endpoints
- Implement pagination separately

---

### Problem: Slice depends on another slice

**Solution:** Reorder slices
- Core happy path first (no dependencies)
- Add features that depend on core
- Parallel slices when possible

---

### Problem: Can't deploy slice independently

**Solution:** Add feature flag
- Hide incomplete UI behind feature flag
- Deploy API endpoints but don't expose routes
- Use Lambda aliases for gradual rollout

---

### Problem: Breaking change needed

**Solution:** Version API
- Create `/v2/endpoint` with new behavior
- Keep `/v1/endpoint` for 30 days
- Document migration path
- Support both during transition

---

### Problem: WALKTHROUGH.md out of date

**Solution:** Update manually or create a documentation automation
- Manually update docs/WALKTHROUGH.md after every slice
- CI/CD check: WALKTHROUGH.md must be updated
- Make it part of Definition of Done
- Consider creating a custom documentation agent if needed

---

## Common Pitfalls

❌ **Implementing entire feature at once**
→ ✅ Break into 5-10 slices

❌ **Slicing by layer (DB → API → UI)**
→ ✅ Slice vertically (DB + API + UI per slice)

❌ **Deploying multiple slices together**
→ ✅ Deploy after each slice

❌ **Breaking changes without migration plan**
→ ✅ Version APIs, support old + new

❌ **Skipping tests**
→ ✅ Write tests with every slice

❌ **Forgetting to update docs**
→ ✅ Update progress.md, changes.md, and docs manually after each slice

❌ **Not using feature flags**
→ ✅ Hide incomplete features

---

## Success Metrics

Track these metrics per feature:

| Metric | Target |
|--------|--------|
| Slice Size | 1-3 days |
| Deployment Frequency | 1-2 per week |
| Test Coverage | >80% |
| Time to Production (first slice) | < 1 week |
| Time to Production (feature complete) | < 3 weeks |
| Rollback Rate | < 5% |
| Production Incidents | 0 per slice |

---

## Next Steps

1. ✅ Read [`SPECFLOW_V2.md`](SPECFLOW_V2.md)
2. ✅ Read [`.cursor/rules/specfold-v2.mdc`](.cursor/rules/specfold-v2.mdc)
3. ✅ Review agents: [AGENTS.md](AGENTS.md) — Incremental Planner and implementation agents
4. ✅ Review governance files (if using advanced features)
5. ✅ Pick a feature to pilot
6. ✅ Follow the Feature Lifecycle (Phase 1-7)
7. ✅ Iterate and improve

---

**Ship value early and often. Deliver in slices, not waterfalls.**
