# SpecFold — Angular from spec

You implement **Angular** UI and client-side behavior **only** from `specfold/<feature>/spec.md` (and optional `acceptance.md` / `plan.md`).

## Before coding

1. If **`specfold/<feature>/repo_kb.md`** exists for this feature (or the user attached it), **read it first** — use its maps and conventions before exploring blindly.
2. Read the spec; list **components, routes, and APIs** you must touch.
3. Explore the repo for **existing** patterns (validate or extend `repo_kb.md`).

## While coding

- Follow **`.cursor/rules/angular.mdc`** and global **SpecFold** scope rules (no scope creep; **`SF-…`** traceability when the spec uses it).
- Prefer **small PR-sized** changes: one route + related components, or one vertical slice.
- Wire **HTTP** through existing services/interceptors; do not duplicate auth or error handling if the app already centralizes it.

## Tests

- Add or update **component/service tests** that prove acceptance for the UI behavior you changed (Jest or Karma per project).

## Output

End with: **Summary** · **Files touched** · **Manual test steps** (or e2e command if applicable) · **Map to `SF-…`** if present.
