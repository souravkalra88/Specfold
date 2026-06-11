# Step-by-step: SpecFold v2 Incremental Workflow (Demo)

Use this checklist in **any** app repo where you copied SpecFold (`.cursor/`, `specfold/`). This document demonstrates the **SpecFold v2** incremental workflow using the **greet-demo** sample.

**Key Difference from v1:** Features are delivered in **thin, vertical slices** (UI + API + Data + Tests per slice), not all at once.

---

## Step 0 — Open the correct folder in Cursor

**File → Open Folder** → the **repository root** where `.cursor/rules/` and `specfold/` live.

If you unzipped twice, you may see `specfold-main/specfold-main/`; open the **inner** folder that contains `.cursor` and `README.md`.

---

## Phase 1: Requirements & Planning

### Step 1 — Describe the feature

In **plain language**, decide:

- What problem you are solving.
- What is in and out of scope.
- What "done" means (testable acceptance).
- A **feature folder name** in `kebab-case` (example used here: **`greet-demo`**).

---

### Step 2 — Draft the spec (agent)

1. Open **Cursor Agent** chat.
2. `@`-mention: **`.cursor/agents/draft_spec.md`**
3. Paste your description and the feature folder name.

Example (you can adapt):

```text
@.cursor/agents/draft_spec.md

Feature folder: greet-demo

I need a tiny Python helper greet(name) -> str that returns "Hello, {name}!" using a trimmed name; empty/whitespace-only after trim must raise ValueError. No HTTP, no i18n.

Write spec.md with acceptance criteria using SF-001, SF-002, SF-003 IDs.
```

4. **You** review and edit **`specfold/greet-demo/spec.md`** until **In scope / Out of scope / Acceptance** match what you want.

**Output in this repo:** `specfold/greet-demo/spec.md`

---

### Step 2a — Repo knowledge base (optional, for brownfield)

Skip for a single small repo. Use when the change spans **multiple packages/repos** or you need a **brownfield map**.

1. `@`-mention **`.cursor/agents/repo_kb.md`**
2. List the same feature folder and **`@`** every root/path to index.

**Output:** `specfold/<feature>/repo_kb.md`

---

### Step 3 — Generate slice plan (agent, NEW in v2)

Break the feature into **5-10 vertical slices** (each slice = UI + API + Data + Tests):

1. **New** agent message.
2. `@`-mention: **`.cursor/agents/incremental_planner.md`**
3. Attach **`@specfold/greet-demo/spec.md`**

Example:

```text
@.cursor/agents/incremental_planner.md
@specfold/greet-demo/spec.md

Generate slice plan for greet-demo feature.
```

**Output in this repo:** 
- `specfold/greet-demo/progress.md` (slice list with status)
- `specfold/greet-demo/changes.md` (initialized for tracking)

**Example `progress.md` for greet-demo:**

```markdown
# Feature: Greet Demo

## Slices
- [ ] Slice 1: Core greeting (happy path)
- [ ] Slice 2: Input validation (edge cases)
- [ ] Slice 3: Error handling (ValueError)

## Current Slice: Slice 1
- Status: Pending
- Target: 2024-06-12
```

---

## Phase 2: Implementation (Per Slice)

**IMPORTANT:** Implement **ONE slice at a time**. Deploy after each slice before moving to the next.

### Step 4 — Implement Slice 1 (agent)

1. **New** agent message.
2. Attach **`@specfold/greet-demo/spec.md`** (defines what to build)
3. Attach **`@specfold/greet-demo/progress.md`** (identifies current slice)
4. Optionally attach **`@specfold/greet-demo/repo_kb.md`** (if created in Step 2a)

Pick **one** implement agent (from [AGENTS.md](AGENTS.md) table):

```text
@.cursor/agents/ship_from_spec.md
@specfold/greet-demo/spec.md
@specfold/greet-demo/progress.md

Implement Slice 1: Core greeting (happy path)
Match Python src/ + pytest conventions in this repo.
```

**Agent selection guide:**
- **`ship_from_spec.md`** — Generic, any stack (Python-only in this demo)
- **`fullstack_from_spec.md`** — If slice spans Angular UI + FastAPI
- **`python_api_from_spec.md`** — If slice is FastAPI-only
- **`python_aws_from_spec.md`** — If slice is AWS Lambda/DynamoDB-only
- **`python_persistence_from_spec.md`** — If slice is SQLAlchemy models-only

**Output in this repo (Slice 1):** 
- `src/greet_demo/__init__.py`
- `src/greet_demo/greet.py` (core function)
- `tests/test_greet_demo.py` (tests for SF-001)

---

### Step 5 — Update tracking (manual or agent)

After Slice 1 is complete:

1. Mark Slice 1 complete in **`progress.md`**:
   ```markdown
   - [x] Slice 1: Core greeting (deployed 2024-06-12)
   ```

2. Add entry to **`changes.md`**:
   ```markdown
   ## Slice 1 (2024-06-12) — Core Greeting
   **Added:**
   - `greet(name)` function
   - Unit tests for SF-001 (happy path)
   
   **Deployed:**
   - Production: 2024-06-12 10:00 UTC
   ```

3. Update **Current Slice** in `progress.md`:
   ```markdown
   ## Current Slice: Slice 2
   - Status: In Development
   ```

---

### Step 6 — Review & validate

- Compare the diff to **Acceptance** (`SF-001` from `spec.md`).
- Run tests:
  ```powershell
  python -m pip install -e ".[dev]"
  python -m pytest tests/test_greet_demo.py::test_greet_happy_path
  ```
- If code drifted, tighten the spec and run the **same** implement agent again.

---

### Step 7 — Deploy Slice 1 (optional for demo, required in production)

In production, you would:

```bash
# Deploy to staging
git commit -m "[Slice 1] Add core greeting function (SF-001)"
git push origin main

# Run smoke tests
pytest tests/test_greet_demo.py::test_greet_happy_path

# Deploy to production (after validation)
git tag v1.0.0-slice-1
git push origin v1.0.0-slice-1
```

**For this demo:** Skip deployment since it's a local library.

---

### Step 8 — Implement Slice 2 (repeat Steps 4-7)

```text
@.cursor/agents/ship_from_spec.md
@specfold/greet-demo/spec.md
@specfold/greet-demo/progress.md

Implement Slice 2: Input validation (edge cases)
Test trimming whitespace (SF-002).
```

**Output (Slice 2):** 
- Updated `src/greet_demo/greet.py` (add trimming)
- Updated `tests/test_greet_demo.py` (tests for SF-002)

---

### Step 9 — Implement Slice 3 (repeat Steps 4-7)

```text
@.cursor/agents/ship_from_spec.md
@specfold/greet-demo/spec.md
@specfold/greet-demo/progress.md

Implement Slice 3: Error handling (ValueError)
Test empty/whitespace-only input (SF-003).
```

**Output (Slice 3):** 
- Updated `src/greet_demo/greet.py` (add ValueError)
- Updated `tests/test_greet_demo.py` (tests for SF-003)

---

## Phase 3: Verification

### Step 10 — Verify all slices

From the folder that contains **`pyproject.toml`**:

```powershell
python -m pip install -e ".[dev]"
python -m pytest
```

You should see **3 passed** tests mapping to:
- **SF-001:** Core greeting (Slice 1)
- **SF-002:** Input trimming (Slice 2)
- **SF-003:** ValueError for empty input (Slice 3)

---

### Step 11 — Traceability (optional)

Reference **`SF-…`** in commit messages:

```bash
git commit -m "[Slice 1] Add core greeting function (SF-001)"
git commit -m "[Slice 2] Add input trimming (SF-002)"
git commit -m "[Slice 3] Add ValueError for empty input (SF-003)"
```

Or in PR titles:
- `[Slice 1] Core greeting (SF-001)`
- `[Slice 2] Input validation (SF-002)`
- `[Slice 3] Error handling (SF-003)`

---

## Summary: SpecFold v2 vs v1

| Aspect | v1 (Legacy) | v2 (Incremental) |
|--------|-------------|-------------------|
| **Planning** | Optional `plan.md` | Required `progress.md` (slice plan) |
| **Implementation** | Implement entire feature | Implement ONE slice at a time |
| **Deployment** | Deploy entire feature | Deploy after each slice |
| **Tracking** | None | `progress.md` + `changes.md` |
| **Risk** | High (large batch) | Low (small batches) |
| **Feedback** | Delayed | Continuous |
| **Time to first value** | End of feature | After Slice 1 |

---

## What's Next?

1. **Read:** [AGENTS.md](AGENTS.md) — Full agent map with production-grade rules
2. **Read:** [`.cursor/rules/specfold-v2.mdc`](.cursor/rules/specfold-v2.mdc) — Incremental change workflow
3. **Read:** [IMPLEMENTATION_GUIDE.md](IMPLEMENTATION_GUIDE.md) — Detailed implementation guide
4. **Try:** A real feature in your app using the incremental workflow

---

## Key Takeaways

✅ **Break features into 5-10 vertical slices** (UI + API + Data + Tests per slice)  
✅ **Implement ONE slice at a time** (use `progress.md` to track current slice)  
✅ **Deploy after each slice** (staging → production)  
✅ **Update tracking after each slice** (`progress.md`, `changes.md`)  
✅ **Maintain backward compatibility** (version APIs, use feature flags)  
✅ **Reference `SF-…` IDs** in code, commits, PRs for traceability  

**Ship value early and often. Deliver in slices, not waterfalls.**
