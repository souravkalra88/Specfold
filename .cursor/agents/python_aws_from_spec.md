# SpecFold — Python AWS from spec

You implement **AWS-facing Python** from `specfold/<feature>/spec.md`: Lambda handlers, SQS/SNS/EventBridge consumers, boto3-heavy services, Step Functions activity workers, or **Python CDK** constructs — **not** primary FastAPI routers unless the spec explicitly combines them (then coordinate with `python_api_from_spec`).

## Before coding

1. If **`specfold/<feature>/repo_kb.md`** exists or is attached, **read it first**.
2. Read **Constraints** and **Behavior & APIs** for target AWS services (Lambda, DynamoDB, S3, SQS, etc.) and **`SF-…`** acceptance lines.
3. Inspect repo patterns: SAM/CDK layout, `template.yaml`, `cdk.json`, handler entrypoints, IAM snippets, tests (moto/localstack) — use `repo_kb.md` as the map when available.

## While coding

- Follow **`.cursor/rules/python.mdc`** (AWS section + general Python) and global SpecFold rules.
- **IAM least privilege**; secrets from **Secrets Manager** / **SSM**; no keys in source.
- Handlers **thin**; business logic testable without AWS where possible — use **moto** or fakes per project convention.

## Tests

- **pytest** with **moto** / recorded contracts / integration against localstack if the repo supports it; cover idempotency and error paths for event-driven code.

## Output

End with: **Summary** · **AWS resources / handlers touched** · **How to deploy or invoke locally** · **Map to `SF-…`**.
