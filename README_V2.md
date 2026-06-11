# SpecFlow v2: Production-Grade AI-Driven SDLC Framework

**Stack:** Angular + Python FastAPI + AWS Serverless  
**Philosophy:** Spec-Driven + Incremental Delivery + Vertical Slicing

---

## 🚀 What is SpecFlow v2?

SpecFlow v2 is a complete redesign of your SDLC framework optimized for:

1. **AI-Driven Development** — Two specialized agents (Engineering + Documentation)
2. **Incremental Delivery** — Ship features in thin, production-ready slices
3. **Vertical Slicing** — Each slice: UI + API + Data + Tests + Docs
4. **AWS Serverless** — Lambda, API Gateway, DynamoDB, RDS, S3, Cognito patterns
5. **Quality by Design** — Governance gates enforce standards at every phase

---

## 📁 What's Included

### Core Framework
- **[SPECFLOW_V2.md](SPECFLOW_V2.md)** — Complete framework specification
- **[IMPLEMENTATION_GUIDE.md](IMPLEMENTATION_GUIDE.md)** — Step-by-step how-to guide

### Rules
- **[.cursor/rules/specfold-v2.mdc](.cursor/rules/specfold-v2.mdc)** — Incremental development workflow
- Original: [.cursor/rules/specfold.mdc](.cursor/rules/specfold.mdc) (legacy)

### Agents
- **[.agents/engineering-agent.md](.agents/engineering-agent.md)** — End-to-end implementation (Analysis → AWS deployment)
- **[.agents/documentation-agent.md](.agents/documentation-agent.md)** — Living documentation (WALKTHROUGH.md, ADRs, inventories)
- **[.cursor/agents/incremental_planner.md](.cursor/agents/incremental_planner.md)** — Break features into 5-10 shippable slices

### Governance
- **[.governance/definition-of-ready.md](.governance/definition-of-ready.md)** — Feature ready for development?
- **[.governance/definition-of-done.md](.governance/definition-of-done.md)** — Feature ready for production?
- **[.governance/quality-gates.md](.governance/quality-gates.md)** — 6 gates from requirements → production

### Templates
- **[templates/feature/requirements/spec.md](templates/feature/requirements/spec.md)** — Feature specification template
- **[templates/feature/requirements/acceptance.md](templates/feature/requirements/acceptance.md)** — Gherkin acceptance criteria template

---

## ⚡ Quick Start

### 1. Read the Docs (30 minutes)
1. **[SPECFLOW_V2.md](SPECFLOW_V2.md)** — Framework overview
2. **[.cursor/rules/specfold-v2.mdc](.cursor/rules/specfold-v2.mdc)** — Incremental workflow
3. **[IMPLEMENTATION_GUIDE.md](IMPLEMENTATION_GUIDE.md)** — Step-by-step guide

### 2. Pilot a Feature (1 week)
Pick a small feature (e.g., "User Orders") and follow the [Feature Lifecycle](IMPLEMENTATION_GUIDE.md#feature-lifecycle-step-by-step):

1. **Requirements:** Write spec → Get approval (1 day)
2. **Design:** Generate architecture + API spec → Get approval (2 days)
3. **Slice Planning:** Break into 5 slices → Get approval (1 day)
4. **Implementation:** Implement Slice 1 → Deploy (2-3 days)
5. **Repeat:** Implement remaining slices (1-2 weeks)

### 3. Scale to Team (1 month)
- Train team on SpecFlow v2
- Migrate one feature per sprint
- Refine agents based on feedback
- Full adoption after 4-6 sprints

---

## 🎯 Key Concepts

### Vertical Slicing

Each slice delivers **complete functionality**:

```
❌ Horizontal (Layer by Layer) — No value until the end
┌──────────────┐
│   UI Layer   │  Week 1
├──────────────┤
│   API Layer  │  Week 2
├──────────────┤
│  Data Layer  │  Week 3
└──────────────┘

✅ Vertical (Feature by Feature) — Value after each slice
┌───┬───┬───┬───┬───┐
│ S │ S │ S │ S │ S │
│ l │ l │ l │ l │ l │
│ i │ i │ i │ i │ i │
│ c │ c │ c │ c │ c │
│ e │ e │ e │ e │ e │
│   │   │   │   │   │
│ 1 │ 2 │ 3 │ 4 │ 5 │
│   │   │   │   │   │
│ U │ U │ U │ U │ U │
│ I │ I │ I │ I │ I │
│   │   │   │   │   │
│ A │ A │ A │ A │ A │
│ P │ P │ P │ P │ P │
│ I │ I │ I │ I │ I │
│   │   │   │   │   │
│ D │ D │ D │ D │ D │
│ B │ B │ B │ B │ B │
└───┴───┴───┴───┴───┘
 3d  2d  2d  3d  2d
```

**Result:** 5 deployments, 5 PRs, value every 2-3 days

---

### Two-Agent Model

**Engineering Agent:**
- Analysis → Architecture → Data Modeling
- Angular → FastAPI → AWS
- Security → Testing

**Documentation Agent:**
- WALKTHROUGH.md (living system docs)
- ADRs (architecture decisions)
- API Inventory + Database Inventory
- Release Notes + Technical Debt

**Why two agents?**
- Clear separation of concerns
- Docs updated automatically after every slice
- No documentation drift

---

### Quality Gates

**6 Gates from Requirements → Production:**

1. **Requirements → Design:** Spec approved
2. **Design → Implementation:** Architecture approved
3. **Implementation → Testing:** Code reviewed
4. **Testing → Staging:** Tests pass
5. **Staging → Production:** Smoke tests pass
6. **Production:** Monitoring healthy

**Automated in CI/CD:**
- Lint, test, OpenAPI validation, security scans
- Block merge if gates fail

---

### Incremental Tracking

**`progress.md`** — Current slice status:
```markdown
## Slices
- [x] Slice 1: Create order (deployed)
- [x] Slice 2: View orders (deployed)
- [ ] Slice 3: Validation (in progress)
- [ ] Slice 4: Performance (planned)
- [ ] Slice 5: Polish (planned)
```

**`changes.md`** — What changed per slice:
```markdown
## Slice 2 (2024-06-12)
**Added:**
- GET /orders/{orderId}
- Angular: order-list component

**Deployed:**
- Production: 2024-06-13 09:00 UTC
```

---

## 📊 SpecFlow v2 vs v1

| Feature | v1 (Legacy) | v2 (New) |
|---------|-------------|----------|
| **Delivery Model** | Waterfall (entire feature) | Incremental (5-10 slices) |
| **Deployment Frequency** | Once per feature (2-4 weeks) | Per slice (1-2 per week) |
| **PR Size** | Large (> 1000 lines) | Small (< 500 lines) |
| **Feedback Loop** | End of feature | After each slice |
| **Rollback** | Entire feature | Per slice |
| **Documentation** | Manual | Auto-updated (Documentation Agent) |
| **Quality Gates** | Informal | 6 formal gates |
| **AWS Standards** | Ad-hoc | Documented in `standards/` |
| **Slice Planning** | No | Yes (`incremental_planner.md`) |
| **Progress Tracking** | No | Yes (`progress.md`, `changes.md`) |

---

## 🏗️ Folder Structure

```
specflow-v2/
├── features/                    # Vertical feature slices
│   ├── user-orders/
│   │   ├── requirements/
│   │   │   ├── spec.md          # Feature spec
│   │   │   └── acceptance.md    # Gherkin AC
│   │   ├── design/
│   │   │   ├── architecture.md
│   │   │   ├── api-spec.yaml    # OpenAPI
│   │   │   ├── data-model.md
│   │   │   └── adr/             # ADRs
│   │   ├── ui/                  # Angular
│   │   ├── api/                 # FastAPI
│   │   ├── backend/             # Services, repos
│   │   ├── infrastructure/      # CDK/Terraform
│   │   ├── security/
│   │   │   ├── iam-policies.json
│   │   │   └── threat-model.md
│   │   ├── tests/
│   │   ├── deployment/
│   │   ├── progress.md          # Slice status
│   │   └── changes.md           # Change log
│   └── ...
│
├── docs/
│   ├── WALKTHROUGH.md           # Living system docs
│   ├── api-inventory.md
│   ├── database-inventory.md
│   ├── adr/                     # Global ADRs
│   ├── runbooks/
│   └── technical-debt.md
│
├── standards/                   # AWS + stack standards
│   ├── fastapi.md
│   ├── lambda.md
│   ├── dynamodb.md
│   ├── rds.md
│   ├── api-gateway.md
│   ├── cognito.md
│   └── ...
│
├── templates/                   # Scaffolding
│   └── feature/
│
├── .agents/                     # AI agents
│   ├── engineering-agent.md
│   └── documentation-agent.md
│
└── .governance/                 # Quality gates
    ├── definition-of-ready.md
    ├── definition-of-done.md
    └── quality-gates.md
```

---

## 🔧 Technology Stack

### Frontend
- **Angular 21** (standalone components, signals, OnPush)
- **TypeScript** (strict mode)
- **Vitest + TestBed** (unit tests)
- **Cypress/Playwright** (E2E tests)

### Backend
- **Python 3.12+** (FastAPI)
- **Pydantic** (validation)
- **pytest** (tests)
- **SQLAlchemy + Alembic** (RDS)

### AWS
- **Lambda** (Python 3.12, Powertools)
- **API Gateway** (HTTP API)
- **DynamoDB** (single-table design)
- **RDS PostgreSQL/MySQL** (relational data)
- **S3** (static assets, receipts)
- **Cognito** (authentication)
- **CloudWatch** (logs, metrics, alarms)
- **X-Ray** (distributed tracing)
- **Secrets Manager** (secrets)
- **CDK** (infrastructure as code)

### CI/CD
- **GitHub Actions / GitLab CI**
- **Automated quality gates**
- **Staging → Production pipeline**

---

## 📖 Agent Usage Examples

### Example 1: New Feature from Scratch

```
Step 1: Generate spec
@.cursor/agents/draft_spec.md
"I want to build user authentication with Cognito"

Step 2: Generate design
@.agents/engineering-agent.md
@features/user-auth/requirements/spec.md
"Generate design artifacts"

Step 3: Generate slice plan
@.cursor/agents/incremental_planner.md
@features/user-auth/requirements/spec.md
"Generate slice plan"

Step 4: Implement Slice 1
@.agents/engineering-agent.md
@features/user-auth/requirements/spec.md
@features/user-auth/progress.md
"Implement Slice 1"

Step 5: Deploy Slice 1
# CI/CD pipeline
git commit -m "[Slice 1] Add user registration"
git push

Step 6: Update docs
@.agents/documentation-agent.md
@features/user-auth/progress.md
"Update WALKTHROUGH.md"
```

---

### Example 2: Brownfield (Existing Codebase)

```
Step 1: Map existing code
@.cursor/agents/repo_kb.md
@existing-ui/
@existing-api/
"Map existing codebase for orders feature"

Step 2: Generate spec with constraints
@.cursor/agents/draft_spec.md
@features/user-orders/repo_kb.md
"Generate spec considering existing code"

Step 3: Proceed normally
# Design → Slices → Implement
```

---

## 📈 Success Metrics

Track these per feature:

| Metric | Target |
|--------|--------|
| **Slice Size** | 1-3 days |
| **Deployment Frequency** | 1-2 per week |
| **Test Coverage** | >80% |
| **Time to First Production Deploy** | < 1 week |
| **Time to Feature Complete** | < 3 weeks |
| **Rollback Rate** | < 5% |
| **Production Incidents** | 0 per slice |

---

## 🚨 Common Pitfalls

| Pitfall | Solution |
|---------|----------|
| ❌ Implementing entire feature at once | ✅ Break into 5-10 slices |
| ❌ Slicing by layer (DB → API → UI) | ✅ Slice vertically per feature |
| ❌ Deploying multiple slices together | ✅ Deploy after each slice |
| ❌ Breaking changes without plan | ✅ Version APIs, support old + new |
| ❌ Skipping tests | ✅ Write tests with every slice |
| ❌ Forgetting to update docs | ✅ Invoke Documentation Agent |
| ❌ Not using feature flags | ✅ Hide incomplete features |

---

## 🎓 Learning Path

### Week 1: Learn the Framework
- Day 1: Read SPECFLOW_V2.md
- Day 2: Read specfold-v2.mdc
- Day 3: Review agents (Engineering, Documentation, Incremental Planner)
- Day 4: Review governance (DoR, DoD, Quality Gates)
- Day 5: Review IMPLEMENTATION_GUIDE.md

### Week 2: Pilot Feature
- Day 1-2: Write spec + acceptance criteria
- Day 3: Generate design
- Day 4: Generate slice plan
- Day 5: Implement Slice 1

### Week 3-4: Complete Pilot
- Complete all slices
- Deploy to production
- Document lessons learned

### Month 2-3: Team Rollout
- Train team
- Migrate one feature per sprint
- Refine agents and templates

### Month 4+: Full Adoption
- All new features use SpecFlow v2
- Deprecate legacy `specfold/`
- Continuous improvement

---

## 📞 Support

### Documentation
- **Framework:** [SPECFLOW_V2.md](SPECFLOW_V2.md)
- **Implementation:** [IMPLEMENTATION_GUIDE.md](IMPLEMENTATION_GUIDE.md)
- **Rules:** [.cursor/rules/specfold-v2.mdc](.cursor/rules/specfold-v2.mdc)

### Agents
- **Engineering:** [.agents/engineering-agent.md](.agents/engineering-agent.md)
- **Documentation:** [.agents/documentation-agent.md](.agents/documentation-agent.md)
- **Incremental Planner:** [.cursor/agents/incremental_planner.md](.cursor/agents/incremental_planner.md)

### Templates
- **Spec:** [templates/feature/requirements/spec.md](templates/feature/requirements/spec.md)
- **Acceptance:** [templates/feature/requirements/acceptance.md](templates/feature/requirements/acceptance.md)

---

## 🚢 Ship Value, Not Friction

**SpecFlow v2 = Spec-Driven + Incremental + AI-Powered**

- ✅ Write spec → Break into slices → Implement incrementally
- ✅ Deploy after each slice → Get feedback early
- ✅ Maintain backward compatibility → No breaking changes
- ✅ Automate quality gates → Catch issues early
- ✅ Update docs automatically → No drift
- ✅ Ship value every week → Delight users

---

**Ready to ship faster? Start with [IMPLEMENTATION_GUIDE.md](IMPLEMENTATION_GUIDE.md)**
