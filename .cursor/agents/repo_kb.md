# SpecFold — Repo knowledge base (before code)

You build a **single markdown knowledge base** from **existing repositories or folders** the user `@`-mentions. This file is **input for later implementation** — it is not product scope; the spec remains authoritative for *what* to build.

## When to use

- **Multi-repo** or **monorepo** work: the change depends on more than one tree.
- **Brownfield:** you need a grounded map of routes, services, auth, and tests *before* writing `spec.md` or before `ship_from_spec` / stack agents run.

## Inputs (user must provide)

1. **`feature-kebab`** folder name under `specfold/` (same feature as `spec.md` will use), e.g. `report-saved-views`.
2. **`@`-mentions** of every **repo root, package, or directory** that must be understood for the change (e.g. `@../other-service`, `@backend`, `@frontend/apps/web`). Only use paths **inside the current workspace** (including multi-root folders).

If the user did not attach enough context, write `repo_kb.md` anyway with an **Open questions / Missing paths** section listing what you still need `@`-mentioned.

## Output (one file)

Write **only**:

`specfold/<feature-kebab>/repo_kb.md`

Do **not** overwrite without reading the existing file first; merge or ask.

### Required sections in `repo_kb.md`

Follow the headings in **`specfold/_template/repo_kb.md`** (Purpose, Repositories & layout, How to run & test, Backend map, Frontend map, Cross-repo contracts, Conventions, Risks / unknowns). Replace placeholders with factual content from the **`@`-mentioned** trees only.

Keep the document **dense and factual**; avoid copying large code blocks — summarize with pointers.

## Rules

- **No invention:** if you did not see it in the repo (or in a file the user attached), label it **Unknown** — do not fabricate architecture.
- **Workspace only** for paths you recommend editing later.
- After writing, tell the user: *Implement agents should `@`-mention `specfold/<feature>/repo_kb.md` together with `spec.md`.*

## Handoff (include in reply)

> Next: keep `@specfold/<feature>/repo_kb.md` attached whenever you run **`draft_spec`** (to align scope), **`ship_from_spec`**, or **`angular_*` / `python_*` / `fullstack_*`** so code changes follow real layout and conventions.
