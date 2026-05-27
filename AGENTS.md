# SpecFold — agent map

Invoke by `@`-mentioning the path in Cursor chat.

## Flow: optional repo KB → draft spec → implement

1. **(Optional) Repo KB** — [`.cursor/agents/repo_kb.md`](.cursor/agents/repo_kb.md) builds **`specfold/<feature>/repo_kb.md`** from every **`@`-mentioned repo/folder** you need for the change. Do this **before** or **alongside** drafting the spec when multiple trees or brownfield context matters.
2. **Draft** — [`.cursor/agents/draft_spec.md`](.cursor/agents/draft_spec.md) creates **`specfold/<feature>/spec.md`** (and optional `plan.md`). You may `@`-mention `repo_kb.md` so constraints match reality.
3. **Implement** — attach **`@specfold/<feature>/spec.md`** and, if it exists, **`@specfold/<feature>/repo_kb.md`**, then invoke an implement agent below.

| Agent | When to use |
|-------|-------------|
| [`.cursor/agents/repo_kb.md`](.cursor/agents/repo_kb.md) | Build **`specfold/<feature>/repo_kb.md`** — condensed **existing codebase** map from **`@`-mentioned** repos/paths (multi-repo / brownfield **before** code). |
| [`.cursor/agents/draft_spec.md`](.cursor/agents/draft_spec.md) | Turn an idea into **`specfold/<feature>/spec.md`** (spec generation only). |
| [`.cursor/agents/ship_from_spec.md`](.cursor/agents/ship_from_spec.md) | Generic implement + tests from spec (any stack); reads **`repo_kb.md`** when present. |
| [`.cursor/agents/angular_from_spec.md`](.cursor/agents/angular_from_spec.md) | Angular UI, routes, forms, services from spec. |
| [`.cursor/agents/python_api_from_spec.md`](.cursor/agents/python_api_from_spec.md) | FastAPI routers, Pydantic, thin handlers, OpenAPI from spec (incl. APIs deployed on **ECS/Lambda**). |
| [`.cursor/agents/python_aws_from_spec.md`](.cursor/agents/python_aws_from_spec.md) | **AWS Python**: Lambda, SQS/SNS/EventBridge consumers, boto3 services, DynamoDB/S3, Python CDK / SAM-adjacent code from spec. |
| [`.cursor/agents/python_persistence_from_spec.md`](.cursor/agents/python_persistence_from_spec.md) | SQLAlchemy models, queries, Alembic migrations from spec. |
| [`.cursor/agents/fullstack_from_spec.md`](.cursor/agents/fullstack_from_spec.md) | One vertical slice: Python API + Angular together. |

## Rules (auto + stack)

| Rule | Scope |
|------|--------|
| [`.cursor/rules/specfold.mdc`](.cursor/rules/specfold.mdc) | Always on — specs, optional **`repo_kb.md`**, scope, `SF-…`, security baseline |
| [`.cursor/rules/angular.mdc`](.cursor/rules/angular.mdc) | Angular / TypeScript / templates |
| [`.cursor/rules/python.mdc`](.cursor/rules/python.mdc) | Python, FastAPI, ORM, Alembic, **AWS** (boto3, Lambda, messaging, IaC globs) |

See [README.md](README.md) for step-by-step usage and folder structure.
