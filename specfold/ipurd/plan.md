# iPURD — Implementation plan (Phase 0 + Phase 1)

Ordered checklist for MVP delivery. Each sprint is **2 weeks**. Traceability: **SF-…** IDs from `spec.md`.

**Feature folder:** `specfold/ipurd/`  
**Do not implement application code until spec is reviewed.**

---

## Prerequisites (before Sprint 1)

- [ ] Confirm Open questions in `spec.md` (PMS vendor, geocoding vendor, tenancy model, hazard data sources).
- [ ] Approve AWS account structure (dev/prod) and domain names.
- [ ] Seed role definitions with compliance (DFSA/GDPR) stakeholder sign-off.

---

## Sprint 1 — Foundation & local dev

**Goal:** Runnable monorepo, database schema, auth skeleton, local Docker stack.

### Deliverables

- [ ] Monorepo scaffold: `apps/web`, `apps/api`, `infra/docker`, `database/`
- [ ] Docker Compose: PostgreSQL 16 + PostGIS, Redis, API hot-reload
- [ ] Alembic initial migration: organizations, users, roles, user_roles, policies, locations
- [ ] FastAPI project shell: health check, OpenAPI, problem+json handler, CORS for local Angular
- [ ] Angular 20 app shell: Material theme, login route placeholder, auth interceptor stub
- [ ] PostGIS enabled; GIST on `locations.geom` (when geom present)

### Acceptance criteria

- [ ] `docker compose up` brings API + DB + Redis; `GET /health` returns 200
- [ ] `alembic upgrade head` succeeds on clean DB
- [ ] Angular serves and calls API health from `environment.ts`
- [ ] README in repo root documents local setup commands

**Maps to:** partial **SF-001** (local only), groundwork for **SF-002**

---

## Sprint 2 — Phase 0 security, audit, CI/CD (dev)

**Goal:** JWT + RBAC end-to-end; audit logging; GitHub Actions deploy to AWS dev.

### Deliverables

- [ ] JWT issue/refresh/logout; password hashing (bcrypt/argon2); `GET /auth/me`
- [ ] RBAC middleware + role seed migration (Admin, Underwriter, Risk Analyst, Read Only)
- [ ] `audit_logs` table + service hook on mutations
- [ ] User admin API (`/users`) — Admin only
- [ ] Terraform/CDK dev stack: VPC, RDS, Redis, Lambda, API Gateway, Secrets Manager placeholders
- [ ] GitHub Actions: lint, test, build, deploy to **dev** on merge to `develop`
- [ ] CloudWatch log groups + basic Lambda error alarm

### Acceptance criteria

- [ ] **SF-002** — Login works; Read Only cannot POST policies; Admin can manage users
- [ ] **SF-014** — Sample PATCH user writes `audit_logs` row with actor and diff
- [ ] **SF-012** — Merge to `develop` deploys API + runs migrations in dev
- [ ] Partial **SF-001** — Dev environment reachable via API Gateway URL

---

## Sprint 3 — Data management & integrations

**Goal:** Policy CRUD, CSV/Excel import, PMS and geocoding adapters.

### Deliverables

- [ ] Policy CRUD API + Pydantic validation (sum_insured &gt; 0, required address fields)
- [ ] `import_batches` + staging validation pipeline; S3 presigned upload flow
- [ ] CSV + Excel parsers (openpyxl/pandas); row-level error report JSON
- [ ] PMS adapter interface + mock/real client; `POST /policies/sync/pms`
- [ ] Geocoding adapter; async job or sync path to set `locations.geom`
- [ ] Angular: policy list/detail, import wizard (Data Management role guard)
- [ ] Seed script: demo org + 100 policies for integration tests

### Acceptance criteria

- [ ] **SF-003** — Invalid import rows reported; successful rows create policies + audit entry
- [ ] **SF-004** — PMS sync and geocode populate ≥100 policies with coordinates in dev
- [ ] Underwriter can create/update policy via UI and API
- [ ] Import UI shows batch status from `GET /policies/import/{batch_id}`

---

## Sprint 4 — Dashboard, search, performance base

**Goal:** Executive dashboard KPIs and sub-second search foundations.

### Deliverables

- [ ] Materialized views: `mv_exposure_by_country`, `mv_exposure_by_city`, `mv_exposure_by_region`
- [ ] Refresh job (Lambda scheduled or post-import hook)
- [ ] `GET /dashboard/summary` with Redis cache (60s TTL)
- [ ] `GET /search/policies` with GIN/trigram indexes; pagination
- [ ] `risk_profiles` table + basic score computation (rule-based MVP, not CAT)
- [ ] Angular dashboard: KPI tiles, exposure charts, high-risk table
- [ ] Load test script (k6) baseline for 100k synthetic policies

### Acceptance criteria

- [ ] **SF-005** — Dashboard p95 &lt;3s on 100k dataset in dev (document test config)
- [ ] **SF-006** — Policy search p95 &lt;1s on 100k dataset
- [ ] **SF-007** — API returns risk profile fields; dashboard shows risk distribution
- [ ] MV refresh completes without blocking imports (concurrent refresh)

---

## Sprint 5 — Geo intelligence & hazard scan

**Goal:** Interactive map, layers, filters, neighbourhood hazard analysis.

### Deliverables

- [ ] `hazard_scans`, `hazard_features` migrations; optional `hazard_zones` reference layer loader
- [ ] Hazard scan service: `ST_DWithin` 500m — water, industrial, hazard zone features
- [ ] Flood/topographic fields on `risk_profiles` (elevation, flood_zone, terrain_risk) from adapter or DEM stub
- [ ] `POST /locations/{id}/hazard-scan`, `GET /hazard-scans/{id}`
- [ ] `GET /dashboard/exposure-map` GeoJSON by bbox
- [ ] Angular map module: Leaflet, OSM tiles, markers, layer toggle, country/city filters
- [ ] Map search: policy + address endpoints wired to UI
- [ ] `HazardScanPanel` + hazard summary on map select

### Acceptance criteria

- [ ] **SF-008** — Map renders markers; filters reduce visible set; search navigates to policy
- [ ] **SF-009** — Hazard scan returns features within 500m; profile shows summary, score, flood/terrain
- [ ] Risk Analyst can trigger scan; Read Only can view results only

---

## Sprint 6 — Risk profile UI, portfolio, reports, prod hardening

**Goal:** Complete Phase 1 UX, portfolio accumulation, reports, production go-live.

### Deliverables

- [ ] Angular `RiskProfilePage`: policy, location, risk, flood/terrain, hazard cards
- [ ] Portfolio APIs + UI: accumulation by country/city/region, filters, search
- [ ] Reports: `POST /reports`, async PDF/HTML generation to S3, presigned GET
- [ ] Prod IaC environment; GitHub Actions prod workflow with manual approval
- [ ] WAF optional; RDS Multi-AZ; backup retention; rollback runbook
- [ ] GDPR hooks: document/data export endpoint for user PII; retention Lambda spec
- [ ] E2E tests: login → dashboard → map → risk profile → portfolio
- [ ] Operational runbooks: deploy, rollback, key rotation, MV refresh failure

### Acceptance criteria

- [ ] **SF-007** — Risk profile page matches API contract for single policy view
- [ ] **SF-010** — Portfolio totals match MVs for filtered cohorts
- [ ] **SF-011** — Executive report downloadable via presigned URL
- [ ] **SF-001** — Prod environment provisioned and smoke-tested
- [ ] **SF-012** — `main` deploy with rollback tested (Lambda alias revert)
- [ ] **SF-013** — Retention/export documented and reviewed by compliance
- [ ] All **SF-001**–**SF-014** traced in test checklist or release notes

---

## Post-MVP backlog (not in plan scope)

- CAT modeling worker tier (ECS)
- RMS/AIR vendor adapters
- Claims analytics warehouse feed
- Pricing engine API
- AI risk scoring with model registry
- Multi-tenant isolation hardening at row level

---

## Suggested implementation agents (after spec review)

| Area | Agent |
|------|--------|
| Full vertical slice | `@.cursor/agents/fullstack_from_spec.md` |
| Angular only | `@.cursor/agents/angular_from_spec.md` |
| FastAPI + OpenAPI | `@.cursor/agents/python_api_from_spec.md` |
| SQLAlchemy + Alembic | `@.cursor/agents/python_persistence_from_spec.md` |
| Lambda, IaC, AWS | `@.cursor/agents/python_aws_from_spec.md` |

Attach `@specfold/ipurd/spec.md` and, when available, `@specfold/ipurd/repo_kb.md`.
