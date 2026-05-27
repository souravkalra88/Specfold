# Step-by-step: README flow + one feature (demo)

Use this checklist in **any** app repo where you copied SpecFold (`.cursor/`, `specfold/`). This document mirrors **README.md** and records the **greet-demo** sample completed in this repo.

---

## Step 0 — Open the correct folder in Cursor

**File → Open Folder** → the **repository root** where `.cursor/rules/` and `specfold/` live.

If you unzipped twice, you may see `specfold-main/specfold-main/`; open the **inner** folder that contains `.cursor` and `README.md`.

---

## Step 1 — Describe the feature

In **plain language**, decide:

- What problem you are solving.
- What is in and out of scope.
- What “done” means (testable acceptance).
- A **feature folder name** in `kebab-case` (example used here: **`greet-demo`**).

---

## Step 2 — Draft the spec (agent)

1. Open **Cursor Agent** chat.
2. `@`-mention: **`.cursor/agents/draft_spec.md`**
3. Paste your description and the feature folder name.

Example (you can adapt):

```text
@.cursor/agents/draft_spec.md

Feature folder: greet-demo

I need a tiny Python helper greet(name) -> str that returns "Hello, {name}!" using a trimmed name; empty/whitespace-only after trim must raise ValueError. No HTTP, no i18n.

Write spec.md; also add plan.md with numbered steps.
```

4. **You** review and edit **`specfold/greet-demo/spec.md`** until **In scope / Out of scope / Acceptance** match what you want.

**Output in this repo:** `specfold/greet-demo/spec.md` and `specfold/greet-demo/plan.md`.

---

## Step 2a — Repo knowledge base (optional)

Skip for a single small repo. Use when the change spans **multiple packages/repos** or you need a **brownfield map**.

1. `@`-mention **`.cursor/agents/repo_kb.md`**
2. List the same feature folder and **`@`** every root/path to index.

**Output:** `specfold/<feature>/repo_kb.md`.

---

## Step 3 — Implement from the spec (agent)

1. **New** agent message.
2. Attach **`@specfold/greet-demo/spec.md`** (and **`@specfold/greet-demo/repo_kb.md`** if you created one).
3. Optionally attach **`@specfold/greet-demo/plan.md`**.

Pick **one** implement agent (from README table), for example:

```text
@.cursor/agents/ship_from_spec.md
Implement per @specfold/greet-demo/spec.md
Match Python src/ + pytest conventions in this repo.
```

For FastAPI-only work you would use **`python_api_from_spec.md`**; for DB models, **`python_persistence_from_spec.md`**, etc.

**Output in this repo:** `src/greet_demo/`, `tests/test_greet_demo.py`, `pyproject.toml` (so `pytest` can run).

---

## Step 4 — Review

- Compare the diff to **Acceptance** (`SF-001` …).
- If code drifted, tighten the spec and run the **same** implement agent again with a narrower prompt.

---

## Step 5 — Traceability (optional)

Reference **`SF-…`** in commit messages or PR text.

---

## Verify this demo

From the folder that contains **`pyproject.toml`**:

```powershell
python -m pip install -e ".[dev]"
python -m pytest
```

You should see **3 passed** tests mapping to **SF-001**, **SF-002**, **SF-003**.
