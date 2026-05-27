# SpecFold Kit

Minimal **spec → code** setup for Cursor.

---

## Folder structure (this kit)

```text
specfold-kit/                      ← copy this whole folder to YOUR app repo root
├── README.md                      ← this file
├── AGENTS.md                      ← agent cheat sheet
├── .gitignore
├── .cursor/
│   ├── rules/
│   │   ├── specfold.mdc           ← always on: specs, scope, SF-…, security baseline
│   │   ├── angular.mdc          ← Angular / TS / HTML / SCSS (globs in file)
│   │   └── python.mdc           ← Python / FastAPI / ORM / Alembic / AWS
│   └── agents/
│       ├── repo_kb.md            ← build repo_kb.md from @-mentioned repos/folders
│       ├── draft_spec.md         ← generates spec.md from your idea
│       ├── ship_from_spec.md    ← generic code from spec
│       ├── angular_from_spec.md
│       ├── python_api_from_spec.md
│       ├── python_aws_from_spec.md
│       ├── python_persistence_from_spec.md
│       └── fullstack_from_spec.md
└── specfold/                      ← generated + hand-written specs live HERE
├── _template/
│   ├── spec.md               ← blank spec starter
│   └── repo_kb.md            ← blank KB starter (optional)
    └── <feature-name-kebab>/     ← one folder per feature (created by you or draft_spec)
        ├── spec.md               ← main contract (required for implement agents)
        ├── repo_kb.md            ← optional: existing-code map (multi-repo / brownfield)
        ├── plan.md               ← optional checklist
        └── acceptance.md         ← optional AC-only file
```

After you copy the kit into an **application** repository, your app’s own folders (e.g. `src/`, `backend/`) sit **next to** `specfold/` and `.cursor/` at the same project root.

**Multi-root Cursor:** add every repo you need as a workspace folder so `@`-mentions and `repo_kb.md` paths stay inside the workspace.

---

## Install (one time)

1. Copy the **`specfold-kit/`** directory into your application’s **repository root** (the folder you open in Cursor).
2. Rename the folder if you like (e.g. keep it as `specfold-kit` or merge only `.cursor/` + `specfold/` into an existing project).
3. In Cursor: **File → Open Folder** → select that **project root** so `.cursor/` is loaded.
4. Confirm `.cursor/rules/` and `.cursor/agents/` appear at the root of the opened workspace.

---

## Step-by-step: from idea to code

### Step 1 — Describe what you want

Open **Cursor Agent** chat. In plain language, describe the feature, constraints, and what “done” means. Decide a short **feature folder name** in `kebab-case` (e.g. `report-saved-views`). You will use the same name in paths below.

### Step 2 — Generate the spec (which agent + where it is written)

**Agent:** `@.cursor/agents/draft_spec.md`

**What it does:** creates (or updates) the spec file at:

`specfold/<feature-name-kebab>/spec.md`

Optionally, in the same chat you can ask it to also add `specfold/<feature-name-kebab>/plan.md` (implementation checklist).

**Example prompt:**

```text
@.cursor/agents/draft_spec.md

Feature folder: report-saved-views

I need saved filter presets on the reports page: save, list, apply, delete.
Stack: Angular front, FastAPI + MySQL back. Out of scope: sharing presets across tenants.

Write spec.md only unless I already have a plan — then add plan.md too.
```

**Your job after Step 2:** Open `specfold/report-saved-views/spec.md`, fix scope and acceptance until it matches what you want. Edit **`SF-001`** … IDs if you renumber acceptance.

### Step 2a — Repo knowledge base (optional, multi-repo or brownfield)

Use this when the change depends on **more than one folder/repo**, or you want a **single “map of reality”** read before drafting or implementing.

**Agent:** `@.cursor/agents/repo_kb.md`

**What it writes:** `specfold/<feature-name-kebab>/repo_kb.md` — layout, entrypoints, conventions, test commands, cross-repo links. **It does not replace `spec.md`:** it is **facts about existing code**; the spec still defines product scope and acceptance.

**What you do:** In agent chat, **`@`-mention every repo root or package** the feature touches (same `feature-name-kebab` you will use for `spec.md`).

**Example:**

```text
@.cursor/agents/repo_kb.md

Feature folder: report-saved-views

Index these paths for the KB (saved views spans API + UI):
@backend @frontend/projects/reports-app
```

**Then:** Optionally `@.cursor/agents/draft_spec.md` with `@specfold/report-saved-views/repo_kb.md` attached so **Constraints** match real stacks. When implementing, **always attach** `@specfold/<feature>/repo_kb.md` together with `@specfold/<feature>/spec.md` if `repo_kb.md` exists.

### Step 3 — Implement from the spec (which agent + how to chain)

In a **new** agent message (or after you’re happy with the spec), **`@`-mention one implementation agent** and **attach the spec file** so the model reads the same contract.

**Attach:** `@specfold/<feature-name-kebab>/spec.md`  
**Also attach (when it exists):** `@specfold/<feature-name-kebab>/repo_kb.md`  
(Optional: `@specfold/<feature-name-kebab>/plan.md`)

**Pick one agent:**

| You are building… | Agent to `@`-mention |
|--------------------|------------------------|
| Anything / unsure | `@.cursor/agents/ship_from_spec.md` |
| Angular UI only | `@.cursor/agents/angular_from_spec.md` |
| HTTP API + services (FastAPI, etc.) | `@.cursor/agents/python_api_from_spec.md` |
| **Lambda, SQS, DynamoDB, S3, CDK/SAM, boto3 workers** | `@.cursor/agents/python_aws_from_spec.md` |
| DB models + migrations | `@.cursor/agents/python_persistence_from_spec.md` |
| API + Angular in one slice | `@.cursor/agents/fullstack_from_spec.md` |

**Example (Python API after spec is written):**

```text
@.cursor/agents/python_api_from_spec.md
Implement per @specfold/report-saved-views/spec.md
@specfold/report-saved-views/repo_kb.md
Follow existing router and schema patterns in this repo.
```

**Example (Angular after spec is written):**

```text
@.cursor/agents/angular_from_spec.md
Implement per @specfold/report-saved-views/spec.md
```

**Example (Python on AWS — Lambda, SQS, boto3, SAM/CDK):**

```text
@.cursor/agents/python_aws_from_spec.md
Implement per @specfold/ingest-queue-worker/spec.md
Use existing SAM template and moto tests in this repo.
```

### Step 4 — Review and iterate

- Compare the diff to **In scope** and **Acceptance** (`SF-…` lines).
- If the code drifted, tighten the spec and run the **same** implement agent again with a narrower prompt (e.g. “only add the `GET /saved-views` endpoint and tests for SF-002”).

### Step 5 — Traceability (optional)

Reference **`SF-…`** in commit messages or the PR description so changes map back to the spec.

---

## Quick reference — spec output vs code

| Phase | Agent | Output location |
|-------|--------|-----------------|
| **Repo KB** (optional) | `repo_kb.md` | `specfold/<feature>/repo_kb.md` |
| Write spec | `draft_spec.md` | `specfold/<feature>/spec.md` (+ optional `plan.md`) |
| Write code | `ship_from_spec.md` or stack agents above | Your app source tree (e.g. `src/`, `app/`, `backend/`) — **not** inside `specfold/` unless you explicitly put code there |

---

## Rules (summary)

| File | When it applies |
|------|------------------|
| `.cursor/rules/specfold.mdc` | **Always** — specs, optional **`repo_kb.md`**, scope, `SF-…`, baseline security |
| `.cursor/rules/angular.mdc` | Angular / TypeScript / HTML / SCSS (see `globs` in file) |
| `.cursor/rules/python.mdc` | Python, FastAPI, ORM, Alembic, **AWS** (see `globs`: includes `template.yaml`, `cdk.json`, …) |

---

## Agents (summary)

| Agent | Use for |
|-------|---------|
| `repo_kb.md` | **Build** `specfold/<feature>/repo_kb.md` from **`@`-mentioned** repos/folders **before** code (optional) |
| `draft_spec.md` | **Generate** `specfold/<feature>/spec.md` from an idea |
| `ship_from_spec.md` | Generic implement + tests from spec |
| `angular_from_spec.md` | Angular UI and client behavior |
| `python_api_from_spec.md` | FastAPI / HTTP layer + services (incl. when API runs on **ECS/Lambda** — still use this for route code) |
| `python_aws_from_spec.md` | **AWS-first Python**: Lambda handlers, SQS/SNS/EventBridge, boto3 jobs, DynamoDB/S3, Python CDK stacks |
| `python_persistence_from_spec.md` | ORM models, queries, migrations |
| `fullstack_from_spec.md` | One slice: Python API + Angular |

Full table: **`AGENTS.md`**.
