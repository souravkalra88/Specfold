# SpecFold — Python API from spec

You implement **HTTP APIs and application services** from `specfold/<feature>/spec.md` (and optional `acceptance.md` / `plan.md`).

## Before coding

1. If **`specfold/<feature>/repo_kb.md`** exists or is attached, **read it first**.
2. Read **Behavior & APIs** and **Acceptance** in the spec; note **`SF-…`** IDs.
3. Find existing **routers**, **schemas**, and **dependency** patterns in the repo (use `repo_kb.md` as the primary map when available).

## While coding

- Follow **`.cursor/rules/python.mdc`** and global SpecFold rules (scope, OpenAPI/Pydantic sync, no invented requirements).
- Keep **handlers thin**; put rules in services/domain functions.
- When adding or changing endpoints, update **OpenAPI-visible** types and any **generated client** step if the project uses one.
- If the service runs on **AWS** (ECS, Lambda behind ALB/API Gateway, etc.), follow the **AWS** section in `python.mdc`: IAM roles, secrets (Secrets Manager / SSM), observability, and DB connection patterns for that runtime.

## Tests

- Add **pytest** tests: happy path + main error cases for new/changed endpoints (status codes, validation errors).

## Output

End with: **Summary** · **Endpoints touched** · **`pytest` command** · **Map to `SF-…`**.
