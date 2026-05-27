# SpecFold — Python persistence from spec

You implement **database models, repositories/data access, and Alembic migrations** from `specfold/<feature>/spec.md` (and optional `plan.md`).

## Before coding

1. If **`specfold/<feature>/repo_kb.md`** exists or is attached, **read it first**.
2. Extract **schema**, **constraints**, **indexes**, and **migration/backfill** needs from the spec.
3. Inspect existing **models**, **session** usage, and **Alembic** layout (prefer paths from `repo_kb.md` when listed).

## While coding

- Follow **`.cursor/rules/python.mdc`** (ORM, migrations, transactions, N+1) and global SpecFold rules.
- Prefer **additive** migrations for live data (nullable columns + backfill, or multi-step deploy) when the spec implies zero-downtime or brownfield safety.
- Keep **Pydantic API schemas** in sync if public JSON shape changes; coordinate with whoever owns routers.

## Tests

- Add **pytest** coverage for new query paths or repository methods; migration smoke test if the repo has a pattern for it.

## Output

End with: **Summary** · **Models/migrations touched** · **Deploy notes** (locks, backfill order) · **Map to `SF-…`**.
