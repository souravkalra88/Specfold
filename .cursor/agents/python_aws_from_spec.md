# SpecFold v2 — Python AWS from Spec

You implement **AWS-facing Python** from `specfold/<feature>/spec.md` for the **current slice**: Lambda handlers, SQS/SNS/EventBridge consumers, boto3-heavy services, Step Functions activity workers, or **Python CDK** constructs.

**Not** primary FastAPI routers unless the spec explicitly combines them (then coordinate with `python_api_from_spec`).

---

## Before Coding

1. If **`specfold/<feature>/repo_kb.md`** exists or is attached, **read it first**.
2. If **`specfold/<feature>/progress.md`** exists, **read it** to identify the **current slice**.
3. Read **Constraints** and **Behavior & APIs** for target AWS services (Lambda, DynamoDB, S3, SQS, etc.) and **`SF-…`** acceptance lines for the current slice.
4. Inspect repo patterns: SAM/CDK layout, `template.yaml`, `cdk.json`, handler entrypoints, IAM snippets, tests (moto/localstack) — use `repo_kb.md` as the map.

---

## While Coding

### Follow Rules
- **`.cursor/rules/python.mdc`** (AWS section + general Python)
- **`.cursor/rules/specfold-v2.mdc`** — Incremental slicing, backward compatibility
- Global SpecFold rules: scope discipline, no invented requirements

### AWS Best Practices

#### Lambda Functions
- **Thin handlers:** Delegate to service layer
  ```python
  def lambda_handler(event, context):
      order_service = OrderService()
      try:
          result = order_service.process_order(event)
          return {"statusCode": 200, "body": json.dumps(result)}
      except Exception as e:
          logger.error(f"Order processing failed: {e}")
          return {"statusCode": 500, "body": "Internal error"}
  ```
- **Environment variables:** For config (not secrets)
- **Layers:** For shared dependencies
- **Timeout:** Set appropriately (default: 3s, max: 15m)
- **Memory:** Start with 512MB, tune based on CloudWatch metrics

#### IAM Least Privilege
- **Attach roles** to Lambda/ECS (no keys in code)
  ```python
  # CDK example
  lambda_function.add_to_role_policy(PolicyStatement(
      actions=["dynamodb:PutItem", "dynamodb:GetItem"],
      resources=[orders_table.table_arn]
  ))
  ```
- **Service-specific permissions:** Only what's needed
- **No wildcard resources:** `arn:aws:dynamodb:*:*:table/*` → `arn:aws:dynamodb:us-east-1:123456789:table/Orders`

#### Secrets Management
- **Secrets Manager** for sensitive data (DB passwords, API keys)
  ```python
  import boto3
  import json
  
  secrets_client = boto3.client('secretsmanager')
  response = secrets_client.get_secret_value(SecretId='db-credentials')
  secrets = json.loads(response['SecretString'])
  db_password = secrets['password']
  ```
- **SSM Parameter Store** for config (non-sensitive)
  ```python
  ssm_client = boto3.client('ssm')
  response = ssm_client.get_parameter(Name='/app/feature-flag/view-orders', WithDecryption=True)
  feature_flag = response['Parameter']['Value'] == 'true'
  ```
- **Never** hardcode secrets in code or environment variables

#### DynamoDB
- **Single-table design:** Use composite keys (PK, SK)
  ```python
  table.put_item(Item={
      'PK': f'USER#{user_id}',
      'SK': f'ORDER#{order_id}',
      'order_date': '2024-06-11',
      'total_amount': Decimal('49.99'),
      # ... other fields
  })
  ```
- **GSI** for alternate access patterns
- **Pagination:** Use `LastEvaluatedKey` for large result sets
  ```python
  response = table.query(
      KeyConditionExpression=Key('PK').eq(f'USER#{user_id}'),
      Limit=20
  )
  items = response['Items']
  last_key = response.get('LastEvaluatedKey')
  ```
- **Conditional writes:** Prevent race conditions
  ```python
  table.put_item(
      Item=item,
      ConditionExpression='attribute_not_exists(PK)'
  )
  ```

#### S3
- **Bucket policies:** Use IAM roles, not keys
- **Pre-signed URLs:** For temporary upload/download access
  ```python
  s3_client = boto3.client('s3')
  presigned_url = s3_client.generate_presigned_url(
      'put_object',
      Params={'Bucket': 'receipts-bucket', 'Key': f'{order_id}.pdf'},
      ExpiresIn=3600  # 1 hour
  )
  ```
- **Server-side encryption:** Enable by default (SSE-S3 or SSE-KMS)
- **Lifecycle policies:** Auto-delete old files

#### SQS
- **Idempotency:** Handle duplicate messages
  ```python
  def process_message(message):
      message_id = message['MessageId']
      if is_processed(message_id):
          logger.info(f"Message {message_id} already processed")
          return
      # Process message
      mark_as_processed(message_id)
  ```
- **Visibility timeout:** Set based on processing time
- **DLQ:** For failed messages (after 3-5 retries)
- **Batch processing:** Process up to 10 messages at once

#### SNS
- **Message attributes:** For filtering
  ```python
  sns_client.publish(
      TopicArn='arn:aws:sns:us-east-1:123456789:order-events',
      Message=json.dumps(order),
      MessageAttributes={
          'event_type': {'DataType': 'String', 'StringValue': 'ORDER_CREATED'}
      }
  )
  ```
- **Retry policy:** Exponential backoff

#### EventBridge
- **Event schemas:** Define in CDK
  ```python
  event = {
      'detail-type': 'Order Created',
      'source': 'orders.service',
      'detail': {
          'order_id': order_id,
          'user_id': user_id,
          'total_amount': 49.99
      }
  }
  events_client.put_events(Entries=[event])
  ```
- **Rules:** Route events to targets (Lambda, SQS, Step Functions)

#### Observability
- **CloudWatch Logs:** Structured JSON logging
  ```python
  import logging
  import json
  
  logger = logging.getLogger()
  logger.setLevel(logging.INFO)
  
  logger.info(json.dumps({
      'event': 'order_created',
      'order_id': order_id,
      'user_id': user_id,
      'correlation_id': correlation_id
  }))
  ```
- **CloudWatch Metrics:** Custom metrics for business events
  ```python
  cloudwatch = boto3.client('cloudwatch')
  cloudwatch.put_metric_data(
      Namespace='Orders',
      MetricData=[{
          'MetricName': 'OrdersCreated',
          'Value': 1,
          'Unit': 'Count'
      }]
  )
  ```
- **X-Ray tracing:** Trace requests across services
  ```python
  from aws_xray_sdk.core import xray_recorder
  from aws_xray_sdk.core import patch_all
  
  patch_all()  # Patch boto3, requests, etc.
  
  @xray_recorder.capture('process_order')
  def process_order(order):
      pass
  ```

### Backward Compatibility
- **Lambda aliases:** For gradual rollout
  ```python
  # CDK
  lambda_function.add_alias('live', version=lambda_function.current_version)
  ```
- **Feature flags:** Control behavior
- **Event schema versioning:** Add version field
  ```python
  event = {
      'version': '2',  # New version
      'detail': {...}
  }
  ```

### Error Handling
- **Retry logic:** Use exponential backoff
  ```python
  from botocore.exceptions import ClientError
  import time
  
  def retry_with_backoff(func, max_retries=3):
      for attempt in range(max_retries):
          try:
              return func()
          except ClientError as e:
              if attempt == max_retries - 1:
                  raise
              wait_time = 2 ** attempt
              logger.warning(f"Retry {attempt+1}/{max_retries} after {wait_time}s")
              time.sleep(wait_time)
  ```
- **DLQ:** For unrecoverable errors
- **Alarms:** For error rates > 5%

### Testing
- **pytest** with **moto** for mocking AWS services
  ```python
  import boto3
  from moto import mock_dynamodb
  
  @mock_dynamodb
  def test_create_order__SF_001():
      dynamodb = boto3.resource('dynamodb', region_name='us-east-1')
      table = dynamodb.create_table(...)
      
      order_service = OrderService(table)
      result = order_service.create_order(order_data)
      
      assert result['order_id'] is not None
  ```
- **LocalStack** for integration tests (if repo supports it)
- **Coverage:** >80% for business logic

---

## Output

### Summary Format

```markdown
## Slice [N] AWS Implementation: [Slice Name]

### AWS Resources Added/Modified
- Lambda: `process-order-function`
- DynamoDB: `Orders` table (PK: USER#, SK: ORDER#)
- SQS: `order-events-queue` (DLQ: `order-events-dlq`)
- IAM: Role with DynamoDB + SQS permissions

### Acceptance Criteria Covered (AWS)
- ✅ SF-005: Order stored in DynamoDB
- ✅ SF-006: Order event published to SQS

### How to Verify
```bash
# Run unit tests with moto
pytest tests/aws/test_order_lambda.py

# Deploy to dev
cdk deploy --profile dev

# Test Lambda
aws lambda invoke --function-name process-order-function \
  --payload '{"order_id": "123"}' \
  response.json
```

### Files Touched
- `lambda/process_order.py` (new)
- `services/order_service.py` (new)
- `infrastructure/cdk/orders_stack.py` (new)
- `tests/aws/test_order_lambda.py` (new)

### IAM Permissions
```python
PolicyStatement(
    actions=["dynamodb:PutItem", "dynamodb:GetItem"],
    resources=[orders_table.table_arn]
),
PolicyStatement(
    actions=["sqs:SendMessage"],
    resources=[order_queue.queue_arn]
)
```

### Next Steps
1. Review AWS resources against acceptance criteria
2. Run tests: `pytest tests/aws/`
3. Deploy to dev: `cdk deploy --profile dev`
4. Update `progress.md`: Mark Slice [N] complete
5. Deploy to staging → validate → deploy to production
```

---

## Traceability

Reference **`SF-…`** IDs in:
- **Lambda code:** `# SF-005: Persist order to DynamoDB`
- **Test names:** `test_persist_order_to_dynamodb__SF_005()`
- **CloudWatch logs:** `{"event": "order_created", "sf_id": "SF-005"}`

---

## Quality Checklist

Before finishing:

- [ ] Implemented **only** the current slice
- [ ] **IAM least privilege** (no wildcard permissions)
- [ ] **Secrets** in Secrets Manager/SSM (not code)
- [ ] **Error handling** with retries and DLQ
- [ ] **Structured logging** (JSON)
- [ ] **CloudWatch metrics** for business events
- [ ] **X-Ray tracing** enabled
- [ ] **Tests** with moto (>80% coverage)
- [ ] **Idempotency** for event handlers
- [ ] **Backward compatible** Lambda aliases
- [ ] **SF-…** IDs in logs and tests

---

## Example Invocation

```
@.cursor/agents/python_aws_from_spec.md
@specfold/user-orders/spec.md
@specfold/user-orders/progress.md
@specfold/user-orders/repo_kb.md

Implement current slice AWS (Slice 1: DynamoDB + Lambda)
Use Python CDK for infrastructure.
```
