# PulseIQ — Analytics & Reporting Platform (Backend MVP)

## Problem

Product teams need a multi-tenant analytics platform to ingest behavioral events, run aggregations, and serve dashboard-ready metrics without building auth, tenancy isolation, async pipelines, and rate limiting from scratch. PulseIQ backend MVP delivers production-oriented APIs for authentication, organization-scoped data, event ingestion, async processing, dashboards, widgets, saved queries, and analytics—using FastAPI, PostgreSQL, Redis, and Celery.

## In scope

### Module 1 — Authentication & multi-tenancy

- User signup, login, logout, token refresh; bcrypt/passlib password hashing; short-lived JWT access tokens (`user_id`, `organization_id`, `role`); securely stored refresh tokens.
- Organization created at signup; organization CRUD; membership management.
- Roles: `OWNER`, `ADMIN`, `ANALYST`, `VIEWER` with hierarchy `OWNER > ADMIN > ANALYST > VIEWER`; FastAPI dependency guards (e.g. `@require_roles(Role.OWNER, Role.ADMIN)`).
- Team invitations: invite by email, invitation token, accept flow, role assignment on accept.
- Multi-tenant isolation: `organization_id` on all business tables; repository-layer org filtering; no cross-tenant reads or writes.

### Module 2 — API key management

- Create, list, revoke, rotate organization API keys for ingestion.
- Store: `id`, `organization_id`, `name`, `hashed_key`, `created_at`, `expires_at`, `status`; raw key shown once on create.

### Module 3 — Data ingestion

- `POST /events` — single event.
- `POST /events/batch` — high-volume batch.
- `POST /events/upload-csv` — CSV upload, row validation, async processing.
- Pydantic validation (required fields, types, timestamps, payload size limits); structured validation errors for invalid records.
- Event storage (JSONB): `id`, `organization_id`, `event_name`, `timestamp`, `properties`, `source`, `created_at`; indexes on `organization_id`, `timestamp`, `event_name`.

### Module 4 — Async processing

- Celery + Redis broker for event normalization, batch processing, CSV processing.
- Retry with exponential backoff; dead-letter handling; task status tracking.

### Module 5 — Rate limiting

- Redis-backed organization-level limits; per API key and per organization (e.g. 100 req/min, 1000 events/min); `429 Too Many Requests` when exceeded.

### Module 6 — Dashboards

- Dashboard CRUD: `id`, `organization_id`, `name`, `description`, `created_by`, `created_at`, `updated_at`.
- Sharing modes: Private, Team Shared, Public Read Only; public sharing token for public dashboards.

### Module 7 — Widgets & analytics

- Widget CRUD on dashboards; types: `LINE_CHART`, `BAR_CHART`, `PIE_CHART`, `KPI_CARD`, `TABLE`; fields include `query_definition`, `position`, `size`, `time_range`.
- Saved queries: CRUD + execute; fields: `filters`, `aggregations`, `group_by`, `time_range`.
- Analytics engine: event count, unique users, aggregations, time-series grouping, property filters, date range; chart-ready responses.

### Module 8 — Dashboard auto refresh

- Configurable refresh intervals: 30 seconds, 1 minute, 5 minutes; APIs to refresh dashboard and return latest analytics.

### Non-functional (backend)

- Clean Architecture: Routers → Services → Repositories → Models.
- Security: JWT, RBAC, Pydantic validation, ORM-only DB access, secure password hashing, CORS.
- Error handling: custom exception hierarchy, global handlers, standardized JSON errors.
- Observability: structured logging, correlation IDs, `GET /health`, `GET /ready`.
- Database: PostgreSQL, Alembic migrations, soft delete (`deleted_at`), audit timestamps (`created_at`, `updated_at`, `deleted_at`) on entities.
- Testing: pytest, pytest-asyncio, httpx; unit, service, repository, API integration, async endpoint tests; **≥80% coverage** target.
- Deliverables: complete backend project, Docker Compose, Celery worker, Redis, OpenAPI docs, seed scripts.

## Out of scope

- Frontend UI (Angular, React, or other SPA).
- Real-time WebSocket push for dashboard updates (polling/refresh APIs only).
- Third-party BI connectors (Looker, Tableau, etc.).
- ML-based anomaly detection or predictive analytics.
- Multi-region active-active deployment.
- Billing, subscription, or usage-based invoicing.
- SSO/OAuth providers (Google, SAML) — email/password JWT only in MVP.
- Event schema registry or dynamic event-type versioning beyond JSONB `properties`.
- Kafka/Kinesis streaming ingestion (API + CSV + Celery only).

## Constraints

| Area | Decision |
|------|----------|
| Language | Python 3.11+ |
| API | FastAPI; async/await throughout |
| ORM | SQLAlchemy 2.0 Async |
| Database | PostgreSQL 15+; JSONB for event `properties` |
| Cache / broker | Redis 7 |
| Tasks | Celery with Redis broker |
| Migrations | Alembic |
| Validation | Pydantic v2 |
| DI | FastAPI `Depends` |
| Architecture | Clean Architecture: `api/` → `services/` → `repositories/` → `models/` |
| Auth | JWT access (short-lived) + refresh token stored server-side (hashed); bcrypt via passlib |
| Tenancy | Every business table has `organization_id`; repositories enforce org scope on all queries |
| Logging | Structured JSON logs; correlation ID per request (header + log context) |
| Errors | `{ "error_code", "message", "details" }` response shape |
| Rate limits (defaults) | 100 requests/min per API key; 1000 events/min per organization (configurable via env) |
| Event payload | Max single-event JSON body 64 KB; batch max 500 events or 5 MB per request |
| CSV upload | Max file size 50 MB; async Celery processing |
| Coverage | pytest suite ≥80% line coverage on `services/`, `repositories/`, `api/` |
| Docker | Compose: API, PostgreSQL, Redis, Celery worker, Celery beat (optional for refresh) |

## Behavior & APIs

### Architecture layers

```text
api/routers          # HTTP, Depends, response models
services/            # business rules, orchestration
repositories/        # org-scoped SQLAlchemy queries
models/              # SQLAlchemy declarative models
schemas/             # Pydantic request/response
core/                # config, security, exceptions, logging, deps
tasks/               # Celery task definitions
```

### Auth & organizations

| Method | Path | Auth | Description |
|--------|------|------|-------------|
| POST | `/auth/signup` | Public | Create user + default organization (signup creates org, user as OWNER) |
| POST | `/auth/login` | Public | Returns access + refresh tokens |
| POST | `/auth/logout` | JWT | Revoke refresh token |
| POST | `/auth/refresh` | Refresh token | New access token |
| GET | `/auth/me` | JWT | Current user + org + role |
| POST | `/organizations` | JWT (OWNER) | Create org (if multi-org supported post-signup) |
| GET | `/organizations/{id}` | JWT + org member | Get org |
| PATCH | `/organizations/{id}` | JWT `@require_roles(OWNER, ADMIN)` | Update org |
| DELETE | `/organizations/{id}` | JWT `@require_roles(OWNER)` | Soft delete org |
| GET | `/organizations/{id}/members` | JWT + member | List members |
| PATCH | `/organizations/{id}/members/{user_id}` | JWT `@require_roles(OWNER, ADMIN)` | Update member role |
| DELETE | `/organizations/{id}/members/{user_id}` | JWT `@require_roles(OWNER, ADMIN)` | Remove member |
| POST | `/organizations/{id}/invitations` | JWT `@require_roles(OWNER, ADMIN)` | Invite by email + role |
| POST | `/invitations/{token}/accept` | Public or JWT | Accept invitation |

**Access token claims:** `sub` (user_id), `org_id`, `role`, `exp`, `iat`.

**Refresh token:** opaque token stored hashed in DB with `user_id`, `expires_at`, `revoked_at`.

### API keys

| Method | Path | Auth | Description |
|--------|------|------|-------------|
| POST | `/organizations/{org_id}/api-keys` | JWT `@require_roles(OWNER, ADMIN)` | Create key; response includes raw key once |
| GET | `/organizations/{org_id}/api-keys` | JWT `@require_roles(OWNER, ADMIN, ANALYST)` | List keys (no raw key) |
| POST | `/organizations/{org_id}/api-keys/{id}/revoke` | JWT `@require_roles(OWNER, ADMIN)` | Revoke |
| POST | `/organizations/{org_id}/api-keys/{id}/rotate` | JWT `@require_roles(OWNER, ADMIN)` | New key; old revoked |

Ingestion routes accept `Authorization: Bearer <api_key>` or `X-API-Key` header; key resolves to `organization_id`.

### Event ingestion

| Method | Path | Auth | Description |
|--------|------|------|-------------|
| POST | `/events` | API key | Single event; enqueue normalization task |
| POST | `/events/batch` | API key | Array of events; enqueue batch task |
| POST | `/events/upload-csv` | API key | Multipart CSV; returns `task_id` |
| GET | `/events/tasks/{task_id}` | API key or JWT | Task status (pending, processing, completed, failed, dead_letter) |

**Single event body:**

```json
{
  "event_name": "page_view",
  "timestamp": "2026-01-01T12:00:00Z",
  "properties": {}
}
```

Optional: `user_id` or `distinct_id` in properties for unique-user analytics.

Validation failures return `422` with per-field errors; batch partial validation returns accepted count + error list.

### Dashboards

| Method | Path | Auth | Description |
|--------|------|------|-------------|
| POST | `/dashboards` | JWT `@require_roles(OWNER, ADMIN, ANALYST)` | Create |
| GET | `/dashboards` | JWT + org | List (respect sharing) |
| GET | `/dashboards/{id}` | JWT or public token | Get |
| PATCH | `/dashboards/{id}` | JWT + edit permission | Update |
| DELETE | `/dashboards/{id}` | JWT `@require_roles(OWNER, ADMIN, ANALYST)` | Soft delete |
| PATCH | `/dashboards/{id}/sharing` | JWT `@require_roles(OWNER, ADMIN, ANALYST)` | Set Private / Team Shared / Public Read Only |
| GET | `/public/dashboards/{share_token}` | Public | Read-only public dashboard |

### Widgets

| Method | Path | Auth | Description |
|--------|------|------|-------------|
| POST | `/dashboards/{id}/widgets` | JWT + edit | Create widget |
| GET | `/dashboards/{id}/widgets` | JWT or public read | List |
| PATCH | `/widgets/{id}` | JWT + edit | Update |
| DELETE | `/widgets/{id}` | JWT + edit | Soft delete |

**Widget types:** `LINE_CHART`, `BAR_CHART`, `PIE_CHART`, `KPI_CARD`, `TABLE`.

### Saved queries & analytics

| Method | Path | Auth | Description |
|--------|------|------|-------------|
| POST | `/queries` | JWT `@require_roles(OWNER, ADMIN, ANALYST)` | Create saved query |
| GET | `/queries` | JWT + org | List |
| GET | `/queries/{id}` | JWT + org | Get |
| PATCH | `/queries/{id}` | JWT `@require_roles(OWNER, ADMIN, ANALYST)` | Update |
| DELETE | `/queries/{id}` | JWT `@require_roles(OWNER, ADMIN, ANALYST)` | Soft delete |
| POST | `/queries/{id}/execute` | JWT + org | Run analytics; chart-ready JSON |
| POST | `/dashboards/{id}/refresh` | JWT or public read | Refresh all widgets; honor `refresh_interval` |

**Execute response (example):**

```json
{
  "query_id": "uuid",
  "time_range": { "start": "2026-01-01T00:00:00Z", "end": "2026-01-31T23:59:59Z" },
  "series": [{ "label": "page_view", "points": [{ "ts": "2026-01-01", "value": 120 }] }],
  "aggregates": { "event_count": 5000, "unique_users": 800 }
}
```

### Health & readiness

| Method | Path | Description |
|--------|------|-------------|
| GET | `/health` | Liveness (process up) |
| GET | `/ready` | DB + Redis connectivity |

### Error response shape

```json
{
  "error_code": "RESOURCE_NOT_FOUND",
  "message": "Dashboard not found",
  "details": {}
}
```

Common codes: `VALIDATION_ERROR`, `UNAUTHORIZED`, `FORBIDDEN`, `RESOURCE_NOT_FOUND`, `RATE_LIMIT_EXCEEDED`, `CONFLICT`, `INTERNAL_ERROR`.

### Celery tasks

| Task | Trigger | Behavior |
|------|---------|----------|
| `normalize_event` | Single ingest | Validate, normalize timestamp, persist event |
| `process_batch_events` | Batch ingest | Bulk insert with retry |
| `process_csv_upload` | CSV upload | Parse rows, validate, bulk insert; track progress |
| Dead letter | Max retries exceeded | Mark task failed; store error in task status table |

### Data model summary

All entities include `created_at`, `updated_at`, `deleted_at` (nullable soft delete).

| Entity | Key fields |
|--------|------------|
| `organizations` | `id`, `name`, `slug` |
| `users` | `id`, `email`, `password_hash`, `organization_id`, `role` |
| `refresh_tokens` | `id`, `user_id`, `token_hash`, `expires_at`, `revoked_at` |
| `invitations` | `id`, `organization_id`, `email`, `role`, `token_hash`, `expires_at`, `accepted_at` |
| `api_keys` | `id`, `organization_id`, `name`, `hashed_key`, `expires_at`, `status` |
| `events` | `id`, `organization_id`, `event_name`, `timestamp`, `properties` (JSONB), `source`, indexes |
| `dashboards` | `id`, `organization_id`, `name`, `description`, `created_by`, `sharing_mode`, `public_share_token`, `refresh_interval` |
| `widgets` | `id`, `dashboard_id`, `type`, `title`, `query_id`, `position`, `size`, `time_range` |
| `saved_queries` | `id`, `organization_id`, `name`, `filters`, `aggregations`, `group_by`, `time_range` |
| `ingestion_tasks` | `id`, `organization_id`, `type`, `status`, `celery_task_id`, `error`, `metadata` |

## Open questions

- **Unique users definition:** distinct `properties.user_id`, `properties.distinct_id`, or both with fallback?
- **Public dashboard scope:** public token exposes widget data only, or also saved query definitions?
- **Multi-org users:** one user per organization in MVP, or allow same email across orgs via invitations?
- **Refresh interval storage:** per-dashboard field only, or also user session preference?
- **API key expiration:** required on create or optional with null = no expiry?

## Acceptance

- **SF-001** — Docker Compose starts API, PostgreSQL, Redis, Celery worker; `GET /health` → 200; `GET /ready` → 200 when dependencies up; `alembic upgrade head` succeeds on clean DB.
- **SF-002** — Signup creates user + organization (user is OWNER); login returns JWT; `/auth/me` returns user, org, role; logout revokes refresh token; `/auth/refresh` issues new access token; passwords stored with bcrypt.
- **SF-003** — Organization CRUD and member list/update/remove work; all routes enforce org membership.
- **SF-004** — `@require_roles` blocks insufficient roles (e.g. VIEWER cannot create dashboard); hierarchy respected (OWNER can do ADMIN actions).
- **SF-005** — Invitation create/accept flow assigns role; expired or used tokens rejected.
- **SF-006** — Repository tests prove cross-tenant access impossible: org A token cannot read org B events, dashboards, or queries.
- **SF-007** — API key create returns raw key once; list shows metadata only; revoke prevents ingestion; rotate invalidates old key.
- **SF-008** — `POST /events` with valid API key accepts event and persists after async processing; invalid payload returns structured validation errors.
- **SF-009** — `POST /events/batch` accepts high-volume batch; partial errors reported; events stored with correct `organization_id`.
- **SF-010** — CSV upload returns task id; Celery processes file; invalid rows reported in task result; valid rows stored.
- **SF-011** — Events table has JSONB `properties` and indexes on `organization_id`, `timestamp`, `event_name`.
- **SF-012** — Celery tasks retry with exponential backoff; failed tasks after max retries land in dead-letter status; `GET /events/tasks/{id}` reflects status.
- **SF-013** — Exceeding org or API-key rate limit returns `429` with `RATE_LIMIT_EXCEEDED`; limits enforced via Redis.
- **SF-014** — Dashboard CRUD (create, list, get, update, soft delete) scoped to organization.
- **SF-015** — Sharing modes work: Private (creator/admin), Team Shared (org members), Public Read Only via share token without JWT.
- **SF-016** — Widget CRUD on dashboard; supported widget types validated.
- **SF-017** — Saved query CRUD; execute returns chart-ready series and aggregates for event count, unique users, filters, group_by, time range.
- **SF-018** — `POST /dashboards/{id}/refresh` returns latest analytics for all widgets; supports intervals 30s, 1m, 5m.
- **SF-019** — Global exception handlers return standardized error JSON; correlation ID present in logs and optional response header.
- **SF-020** — All entities have `created_at`, `updated_at`, `deleted_at`; soft delete excludes records from default queries.
- **SF-021** — Test suite passes with ≥80% coverage on services, repositories, and API layers (pytest-asyncio + httpx).
- **SF-022** — Seed script creates demo org, users per role, sample events, dashboard with widgets, and API key for local testing; OpenAPI docs available at `/docs`.
