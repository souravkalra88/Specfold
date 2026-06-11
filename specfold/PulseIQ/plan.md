# PulseIQ — Backend implementation plan

Ordered checklist for backend MVP delivery. Traceability: **SF-…** IDs from `spec.md`.

**Feature folder:** `specfold/pulseiq/`  
**Target codebase:** `pulseiq-api/` (or agreed monorepo path) at workspace root.  
**Do not implement until spec is reviewed.**

---

## Phase 0 — Prerequisites

- [ ] Resolve Open questions in `spec.md` (unique users field, public dashboard scope, multi-org users).
- [ ] Confirm default rate limits and payload size env vars.
- [ ] Choose project root path (`pulseiq-api/` recommended).

---

## Phase 1 — Foundation & local stack

**Goal:** Runnable FastAPI project, Clean Architecture skeleton, Docker, migrations, health endpoints.

### Deliverables

- [ ] Project scaffold: `api/`, `services/`, `repositories/`, `models/`, `schemas/`, `core/`, `tasks/`, `tests/`
- [ ] `core/config.py` — pydantic-settings (DB, Redis, JWT secrets, rate limits)
- [ ] Async SQLAlchemy engine + session factory; Alembic env wired to models
- [ ] Docker Compose: PostgreSQL, Redis, API (uvicorn), Celery worker
- [ ] Structured logging + correlation ID middleware
- [ ] Global exception handlers + error response schema
- [ ] `GET /health`, `GET /ready`; CORS configuration
- [ ] Initial migration: base mixins (`created_at`, `updated_at`, `deleted_at`)

### Acceptance

- [ ] **SF-001** — Compose up; health/ready pass; migrations run clean
- [ ] **SF-019** — Sample 404 returns `{ error_code, message, details }`; logs include correlation ID
- [ ] **SF-020** — Base mixin on first entity migration

---

## Phase 2 — Authentication & multi-tenancy

**Goal:** Signup/login/JWT, organizations, RBAC, invitations, tenant isolation.

### Deliverables

- [ ] Models: `organizations`, `users`, `refresh_tokens`, `invitations`
- [ ] Password hashing (passlib bcrypt); JWT access + refresh token service
- [ ] Auth router: signup, login, logout, refresh, me
- [ ] Organization router: CRUD, members, invitations, accept
- [ ] `require_roles` dependency + role hierarchy helper
- [ ] Base repository with mandatory `organization_id` filter
- [ ] Repository integration tests for cross-tenant denial

### Acceptance

- [ ] **SF-002** — Full auth flow with refresh rotation
- [ ] **SF-003** — Organization and membership APIs
- [ ] **SF-004** — Role guards on sample protected route
- [ ] **SF-005** — Invitation accept assigns role
- [ ] **SF-006** — Cross-tenant access tests fail as expected

---

## Phase 3 — API keys & rate limiting

**Goal:** Org API keys for ingestion; Redis rate limits.

### Deliverables

- [ ] Model: `api_keys` (hashed key, status, expires_at)
- [ ] API key service: create (return raw once), list, revoke, rotate
- [ ] API key auth dependency for ingestion routes
- [ ] Redis rate limiter: per API key (requests/min), per org (events/min)
- [ ] Rate limit middleware or dependency; `429` + `RATE_LIMIT_EXCEEDED`

### Acceptance

- [ ] **SF-007** — Key lifecycle; raw key not returned after create
- [ ] **SF-013** — Limits enforced; headers optional (`X-RateLimit-*`)

---

## Phase 4 — Event ingestion & storage

**Goal:** Single, batch, and CSV ingest with validation and persistence.

### Deliverables

- [ ] Model: `events` (JSONB `properties`, indexes)
- [ ] Model: `ingestion_tasks` for async job status
- [ ] Pydantic schemas: single event, batch, validation limits
- [ ] Routers: `POST /events`, `POST /events/batch`, `POST /events/upload-csv`, `GET /events/tasks/{id}`
- [ ] Event repository (org-scoped bulk insert)
- [ ] CSV parser + row-level validation report

### Acceptance

- [ ] **SF-008** — Single event ingest + validation errors
- [ ] **SF-009** — Batch ingest with partial error reporting
- [ ] **SF-010** — CSV upload enqueues task (processing in Phase 5)
- [ ] **SF-011** — Indexes verified in migration

---

## Phase 5 — Celery async processing

**Goal:** Background normalization, batch, CSV; retries and dead letter.

### Deliverables

- [ ] Celery app config (Redis broker/backend)
- [ ] Tasks: `normalize_event`, `process_batch_events`, `process_csv_upload`
- [ ] Retry policy: exponential backoff, max retries configurable
- [ ] Dead-letter status + error persistence on `ingestion_tasks`
- [ ] Wire ingest endpoints to enqueue tasks (API returns 202 + task id where appropriate)

### Acceptance

- [ ] **SF-010** — CSV fully processed async with status polling
- [ ] **SF-012** — Retry and dead-letter behavior covered by tests

---

## Phase 6 — Dashboards & sharing

**Goal:** Dashboard CRUD and sharing modes.

### Deliverables

- [ ] Model: `dashboards` (sharing_mode, public_share_token, refresh_interval)
- [ ] Dashboard service + repository (org-scoped, soft delete)
- [ ] Router: CRUD + sharing patch + public read by token
- [ ] Permission rules: Private / Team Shared / Public Read Only

### Acceptance

- [ ] **SF-014** — Dashboard CRUD
- [ ] **SF-015** — Sharing modes and public token access without JWT

---

## Phase 7 — Widgets, saved queries & analytics engine

**Goal:** Widget CRUD, saved queries, execute + chart-ready analytics.

### Deliverables

- [ ] Models: `widgets`, `saved_queries`
- [ ] Widget CRUD nested under dashboards
- [ ] Saved query CRUD + `POST /queries/{id}/execute`
- [ ] Analytics service: event count, unique users, aggregations, group_by, time-series, property filters
- [ ] SQL/query builder using org-scoped event repository (parameterized; no raw SQL from client)

### Acceptance

- [ ] **SF-016** — Widget types and CRUD
- [ ] **SF-017** — Query execute returns series + aggregates; filters and date range work

---

## Phase 8 — Dashboard refresh

**Goal:** Refresh interval and bulk widget refresh API.

### Deliverables

- [ ] `refresh_interval` enum: 30s, 1m, 5m on dashboard
- [ ] `POST /dashboards/{id}/refresh` — run linked queries, return combined widget results
- [ ] Optional Celery beat job for scheduled refresh (document if deferred)

### Acceptance

- [ ] **SF-018** — Refresh API returns latest analytics for all widgets

---

## Phase 9 — Testing, seed data & documentation

**Goal:** Coverage target, seed script, OpenAPI polish.

### Deliverables

- [ ] Unit tests: services (auth, analytics, rate limit)
- [ ] Repository tests: org isolation, soft delete
- [ ] API integration tests: httpx AsyncClient per module
- [ ] Coverage config (pytest-cov); CI gate ≥80% on target paths
- [ ] Seed script: demo org, 4 role users, API key, 1k events, 1 dashboard + widgets
- [ ] README: local setup, env vars, migrate, test, run worker

### Acceptance

- [ ] **SF-021** — Coverage ≥80%; all tests green
- [ ] **SF-022** — Seed runnable; `/docs` documents all public routes

---

## Suggested implementation order (single developer)

| Order | Phase | SF IDs |
|-------|-------|--------|
| 1 | Foundation | SF-001, SF-019, SF-020 |
| 2 | Auth & tenancy | SF-002 – SF-006 |
| 3 | API keys + rate limits | SF-007, SF-013 |
| 4 | Ingestion (sync API surface) | SF-008 – SF-011 |
| 5 | Celery | SF-010, SF-012 |
| 6 | Dashboards | SF-014, SF-015 |
| 7 | Widgets & analytics | SF-016, SF-017 |
| 8 | Refresh | SF-018 |
| 9 | Tests & seed | SF-021, SF-022 |

---

## Handoff to implement agent

After spec review, implement with:

`@.cursor/agents/python_api_from_spec.md` and `@.cursor/agents/python_persistence_from_spec.md`

Attach `@specfold/pulseiq/spec.md` (and `@specfold/pulseiq/repo_kb.md` if generated for brownfield repos).
