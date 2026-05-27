# SpecFold — Full stack from spec

Use when the spec spans **Angular + Python** in one slice (e.g. one endpoint + one screen).

## Inputs

- `specfold/<feature>/spec.md` (required) and optional `plan.md` / `acceptance.md`.
- Optional: **`specfold/<feature>/repo_kb.md`** — read **first** when present (multi-repo / layout map).
- User may `@`-mention specific frontend and backend paths.

## Approach

1. Read **`repo_kb.md` first** if it exists or is attached; then read the spec and split work into **API contract** → **backend** → **frontend** (or the reverse if the UI already exists); keep each step small.
2. Apply **`.cursor/rules/python.mdc`** for `*.py` and **`.cursor/rules/angular.mdc`** for Angular files; follow global SpecFold scope and **`SF-…`** traceability.
3. **Align:** request/response types, error shapes, and auth headers with existing client/server conventions.

## Tests

- Backend: pytest for new/changed endpoints.
- Frontend: unit tests for components/services you change.

## Output

End with a **single** summary: backend files · frontend files · how to verify end-to-end · **`SF-…`** map.

If the change is large, tell the user to split into `@angular_from_spec.md` and `@python_api_from_spec.md` or `@python_aws_from_spec.md` (or persistence agent) in separate turns instead.
