# Local Development & Testing Standards

Complete guide for running and testing applications locally before deployment.

---

## Prerequisites

### Required Tools
- **Python 3.12+** - Backend runtime
- **Node.js 20+** - Frontend tooling
- **Docker Desktop** - Container services
- **AWS CLI** - AWS service interaction
- **Git** - Version control

### Optional Tools
- **VS Code** - Recommended IDE
- **Postman/Insomnia** - API testing
- **DynamoDB Workbench** - DynamoDB GUI

---

## FastAPI Local Development

### Running FastAPI

```bash
# Standard uvicorn
uvicorn app.main:app --reload --host 0.0.0.0 --port 8000

# With environment variables
ENVIRONMENT=dev uvicorn app.main:app --reload

# With auto-reload on file changes
uvicorn app.main:app --reload --reload-dir app/

# With specific log level
uvicorn app.main:app --reload --log-level debug
```

### API Documentation

FastAPI provides automatic interactive documentation:

```text
http://localhost:8000/docs       # Swagger UI (interactive)
http://localhost:8000/redoc      # ReDoc UI (readable)
http://localhost:8000/openapi.json  # OpenAPI schema (JSON)
```

### Health Check

```bash
curl http://localhost:8000/health
# Response: {"status": "healthy", "environment": "dev"}
```

---

## Angular Local Development

### Running Angular

```bash
# Standard development server
ng serve

# Custom port
ng serve --port 4200

# Open browser automatically
ng serve --open

# Production build
ng build --configuration production

# Serve production build locally
ng serve --configuration production
```

### Application URLs

```text
http://localhost:4200          # Main application
http://localhost:4200/health   # Health check (if implemented)
```

---

## Angular + FastAPI Integration

### Option 1: Angular Proxy (Recommended)

**Why use proxy?**
- ✅ No CORS issues
- ✅ Same-origin requests
- ✅ Production-like setup
- ✅ Easy environment switching

**Create `proxy.conf.json` in Angular project root:**

```json
{
  "/api": {
    "target": "http://localhost:8000",
    "secure": false,
    "changeOrigin": true,
    "logLevel": "debug",
    "pathRewrite": {
      "^/api": ""
    }
  }
}
```

**Update `angular.json`:**

```json
{
  "projects": {
    "your-app": {
      "architect": {
        "serve": {
          "options": {
            "proxyConfig": "proxy.conf.json"
          }
        }
      }
    }
  }
}
```

**Angular services use relative paths:**

```typescript
// app/core/services/order.service.ts
import { HttpClient } from '@angular/common/http';
import { Injectable, inject } from '@angular/core';
import { Observable } from 'rxjs';

@Injectable({ providedIn: 'root' })
export class OrderService {
  private http = inject(HttpClient);
  
  // ✅ Use relative path - proxy forwards to FastAPI
  createOrder(order: CreateOrderRequest): Observable<OrderResponse> {
    return this.http.post<OrderResponse>('/api/orders', order);
  }
  
  getOrder(orderId: string): Observable<OrderResponse> {
    return this.http.get<OrderResponse>(`/api/orders/${orderId}`);
  }
  
  listOrders(userId: string): Observable<OrderResponse[]> {
    return this.http.get<OrderResponse[]>(`/api/users/${userId}/orders`);
  }
}
```

**Start both servers:**

```bash
# Terminal 1: FastAPI
cd backend
uvicorn app.main:app --reload

# Terminal 2: Angular (with proxy)
cd frontend
ng serve
```

### Option 2: CORS (Not Recommended for Local Dev)

Only enable CORS when required for external clients or cross-origin testing:

```python
# app/main.py
from fastapi.middleware.cors import CORSMiddleware

app.add_middleware(
    CORSMiddleware,
    allow_origins=["http://localhost:4200"],  # Angular dev server
    allow_credentials=True,
    allow_methods=["*"],
    allow_headers=["*"],
)
```

**Cons of CORS approach:**
- ❌ CORS preflight requests slow down development
- ❌ Different from production setup
- ❌ Potential CORS misconfiguration issues

---

## Local AWS Services

### DynamoDB Local

**Run DynamoDB Local with Docker:**

```bash
# Start DynamoDB Local
docker run -d -p 8001:8000 --name dynamodb-local amazon/dynamodb-local

# Stop
docker stop dynamodb-local

# Remove
docker rm dynamodb-local
```

**Or download and run with Java:**

```bash
# Download from AWS
wget https://s3.amazonaws.com/dynamodb-local/dynamodb_local_latest.tar.gz
tar -xzf dynamodb_local_latest.tar.gz

# Run
java -Djava.library.path=./DynamoDBLocal_lib -jar DynamoDBLocal.jar -sharedDb -port 8001
```

**Configure FastAPI for local DynamoDB:**

```python
# app/core/config.py
import os
from pydantic_settings import BaseSettings

class Settings(BaseSettings):
    ENVIRONMENT: str = "dev"
    AWS_REGION: str = "ap-south-1"
    
    # DynamoDB configuration
    DYNAMODB_ENDPOINT: str = os.getenv(
        "DYNAMODB_ENDPOINT",
        "http://localhost:8001" if ENVIRONMENT == "dev" else ""
    )
    DYNAMODB_TABLE: str = "orders-dev"
    
    class Config:
        case_sensitive = True

settings = Settings()

# app/core/database.py
import boto3

def get_dynamodb_resource():
    """Get DynamoDB resource (local or AWS)."""
    if settings.DYNAMODB_ENDPOINT:
        # Local DynamoDB
        return boto3.resource(
            'dynamodb',
            endpoint_url=settings.DYNAMODB_ENDPOINT,
            region_name=settings.AWS_REGION,
            aws_access_key_id='dummy',
            aws_secret_access_key='dummy'
        )
    else:
        # Production DynamoDB
        return boto3.resource('dynamodb', region_name=settings.AWS_REGION)
```

**Create local tables:**

```bash
# Using AWS CLI
aws dynamodb create-table \
  --table-name orders-dev \
  --attribute-definitions \
    AttributeName=PK,AttributeType=S \
    AttributeName=SK,AttributeType=S \
  --key-schema \
    AttributeName=PK,KeyType=HASH \
    AttributeName=SK,KeyType=RANGE \
  --billing-mode PAY_PER_REQUEST \
  --endpoint-url http://localhost:8001

# List tables
aws dynamodb list-tables --endpoint-url http://localhost:8001

# Scan table
aws dynamodb scan --table-name orders-dev --endpoint-url http://localhost:8001
```

**Python script to create tables:**

```python
# scripts/create_local_tables.py
import boto3

dynamodb = boto3.resource(
    'dynamodb',
    endpoint_url='http://localhost:8001',
    region_name='ap-south-1',
    aws_access_key_id='dummy',
    aws_secret_access_key='dummy'
)

# Create orders table
table = dynamodb.create_table(
    TableName='orders-dev',
    KeySchema=[
        {'AttributeName': 'PK', 'KeyType': 'HASH'},
        {'AttributeName': 'SK', 'KeyType': 'RANGE'}
    ],
    AttributeDefinitions=[
        {'AttributeName': 'PK', 'AttributeType': 'S'},
        {'AttributeName': 'SK', 'AttributeType': 'S'}
    ],
    BillingMode='PAY_PER_REQUEST'
)

print(f"Table created: {table.table_name}")
```

### LocalStack (Complete AWS Emulation)

**Why LocalStack?**
- ✅ Emulate S3, DynamoDB, SQS, SNS, Lambda, and more
- ✅ No AWS deployment needed for development
- ✅ Free tier supports core services
- ✅ Fast iteration

**Docker Compose setup:**

```yaml
# docker-compose.yml
version: '3.8'

services:
  localstack:
    image: localstack/localstack:latest
    container_name: localstack
    ports:
      - "4566:4566"  # LocalStack Gateway
      - "4571:4571"  # LocalStack Dashboard (Pro)
    environment:
      - SERVICES=s3,dynamodb,sqs,sns,secretsmanager,lambda
      - DEBUG=1
      - DATA_DIR=/tmp/localstack/data
      - LAMBDA_EXECUTOR=docker
      - DOCKER_HOST=unix:///var/run/docker.sock
    volumes:
      - "./localstack:/tmp/localstack"
      - "/var/run/docker.sock:/var/run/docker.sock"
    networks:
      - app-network

networks:
  app-network:
    driver: bridge
```

**Start LocalStack:**

```bash
# Start services
docker-compose up -d localstack

# Check logs
docker-compose logs -f localstack

# Stop services
docker-compose down
```

**Configure FastAPI for LocalStack:**

```python
# app/core/config.py
class Settings(BaseSettings):
    USE_LOCALSTACK: bool = os.getenv("USE_LOCALSTACK", "true").lower() == "true"
    LOCALSTACK_ENDPOINT: str = "http://localhost:4566"
    AWS_REGION: str = "ap-south-1"

settings = Settings()

# app/core/aws.py
import boto3
from app.core.config import settings

def get_boto3_client(service_name: str):
    """Get boto3 client (LocalStack or AWS)."""
    if settings.USE_LOCALSTACK:
        return boto3.client(
            service_name,
            endpoint_url=settings.LOCALSTACK_ENDPOINT,
            region_name=settings.AWS_REGION,
            aws_access_key_id='test',
            aws_secret_access_key='test'
        )
    else:
        return boto3.client(service_name, region_name=settings.AWS_REGION)

# Usage in services
dynamodb = get_boto3_client('dynamodb')
s3 = get_boto3_client('s3')
sqs = get_boto3_client('sqs')
sns = get_boto3_client('sns')
```

**Initialize LocalStack resources:**

```bash
# scripts/init-localstack.sh
#!/bin/bash

# Create S3 bucket
aws --endpoint-url=http://localhost:4566 s3 mb s3://risk-data-dev

# Create DynamoDB table
aws --endpoint-url=http://localhost:4566 dynamodb create-table \
  --table-name orders-dev \
  --attribute-definitions AttributeName=PK,AttributeType=S AttributeName=SK,AttributeType=S \
  --key-schema AttributeName=PK,KeyType=HASH AttributeName=SK,KeyType=RANGE \
  --billing-mode PAY_PER_REQUEST

# Create SQS queue
aws --endpoint-url=http://localhost:4566 sqs create-queue --queue-name order-events-dev

# Create SNS topic
aws --endpoint-url=http://localhost:4566 sns create-topic --name order-notifications-dev
```

---

## Lambda Local Testing

### SAM Local

**Install SAM CLI:**
```bash
# macOS
brew install aws-sam-cli

# Windows
choco install aws-sam-cli

# Linux
pip install aws-sam-cli
```

**Start API Gateway locally:**

```bash
# Start API
sam local start-api --port 3000

# With environment variables
sam local start-api --env-vars env.json

# Test endpoints
curl http://localhost:3000/health
curl -X POST http://localhost:3000/api/orders \
  -H "Content-Type: application/json" \
  -d '{"user_id": "123", "items": [{"product_id": "p1", "quantity": 2, "price": 10.00}], "total_amount": 20.00}'
```

**Invoke function directly:**

```bash
# Invoke with event
sam local invoke FastApiFunction -e events/create-order.json

# With debugger
sam local invoke -d 5858 FastApiFunction -e events/create-order.json
```

**Environment variables file:**

```json
// env.json
{
  "FastApiFunction": {
    "ENVIRONMENT": "dev",
    "DYNAMODB_ENDPOINT": "http://host.docker.internal:8001",
    "USE_LOCALSTACK": "true",
    "LOCALSTACK_ENDPOINT": "http://host.docker.internal:4566"
  }
}
```

### Serverless Offline

**Install plugin:**

```bash
npm install --save-dev serverless-offline
```

**Update `serverless.yml`:**

```yaml
plugins:
  - serverless-python-requirements
  - serverless-offline

custom:
  serverless-offline:
    httpPort: 3000
    lambdaPort: 3002
    websocketPort: 3001
```

**Start offline mode:**

```bash
# Start server
serverless offline start --stage dev

# With specific port
serverless offline start --httpPort 4000

# Test endpoints
curl http://localhost:3000/health
curl http://localhost:3000/api/orders
```

---

## Testing Standards

### Unit Tests (pytest)

**Test structure:**

```
tests/
├── __init__.py
├── conftest.py           # Shared fixtures
├── test_services/
│   ├── __init__.py
│   └── test_order_service.py
├── test_repositories/
│   ├── __init__.py
│   └── test_order_repository.py
└── test_routers/
    ├── __init__.py
    └── test_orders.py
```

**Example unit test:**

```python
# tests/test_services/test_order_service.py
import pytest
from decimal import Decimal
from uuid import uuid4
from app.services.order_service import OrderService
from app.models.order import CreateOrderRequest, OrderItem

@pytest.mark.asyncio
async def test_create_order__SF_001():
    """Test order creation with valid data (SF-001)."""
    service = OrderService()
    order_data = CreateOrderRequest(
        user_id=uuid4(),
        items=[
            OrderItem(product_id="p1", quantity=2, price=Decimal("10.00"))
        ],
        total_amount=Decimal("20.00")
    )
    
    result = await service.create_order(order_data)
    
    assert result.id is not None
    assert result.user_id == order_data.user_id
    assert result.total_amount == Decimal("20.00")
    assert result.status == "pending"

@pytest.mark.asyncio
async def test_create_order_empty_items__SF_002():
    """Test order creation fails with empty items (SF-002)."""
    service = OrderService()
    order_data = CreateOrderRequest(
        user_id=uuid4(),
        items=[],
        total_amount=Decimal("0.00")
    )
    
    with pytest.raises(ValueError, match="at least one item"):
        await service.create_order(order_data)
```

### Integration Tests

```python
# tests/integration/test_orders_api.py
import pytest
from fastapi.testclient import TestClient
from app.main import app

client = TestClient(app)

@pytest.fixture(scope="module")
def setup_dynamodb():
    """Setup local DynamoDB for testing."""
    # Create tables
    yield
    # Cleanup

def test_create_order_integration__SF_001(setup_dynamodb):
    """Integration test for order creation (SF-001)."""
    order_data = {
        "user_id": "123e4567-e89b-12d3-a456-426614174000",
        "items": [
            {"product_id": "p1", "quantity": 2, "price": 10.00}
        ],
        "total_amount": 20.00
    }
    
    response = client.post("/api/orders", json=order_data)
    
    assert response.status_code == 201
    data = response.json()
    assert data["user_id"] == order_data["user_id"]
    assert data["total_amount"] == order_data["total_amount"]
    assert "id" in data

def test_get_order_not_found__SF_003():
    """Test 404 for non-existent order (SF-003)."""
    fake_id = "123e4567-e89b-12d3-a456-426614174999"
    
    response = client.get(f"/api/orders/{fake_id}")
    
    assert response.status_code == 404
```

### Run Tests

```bash
# All tests
pytest

# With coverage
pytest --cov=app --cov-report=html --cov-report=term

# Coverage report location: htmlcov/index.html

# Specific test file
pytest tests/test_services/test_order_service.py

# Specific test
pytest tests/test_services/test_order_service.py::test_create_order__SF_001

# With verbose output
pytest -v -s

# Fail fast (stop on first failure)
pytest -x

# Run tests matching pattern
pytest -k "order"

# Parallel execution (requires pytest-xdist)
pytest -n auto
```

### Angular Tests (Jasmine/Jest)

```bash
# Run all tests
ng test

# Run once (CI mode)
ng test --watch=false

# With coverage
ng test --code-coverage

# Coverage report: coverage/index.html

# Specific test file
ng test --include='**/order-form.component.spec.ts'
```

---

## Lambda Compatibility

Always maintain Lambda compatibility for production deployment:

```python
# lambda_handler.py
from mangum import Mangum
from app.main import app

# Mangum adapter converts ASGI to Lambda event format
handler = Mangum(
    app,
    lifespan="off",  # Disable lifespan for Lambda
    api_gateway_base_path="/"
)

# Optional: Support local testing
if __name__ == "__main__":
    import uvicorn
    uvicorn.run(app, host="0.0.0.0", port=8000)
```

**Test Lambda handler locally:**

```python
# test_lambda_local.py
from lambda_handler import handler

# API Gateway event
event = {
    'httpMethod': 'GET',
    'path': '/health',
    'headers': {},
    'queryStringParameters': None,
    'body': None,
    'requestContext': {
        'requestId': 'test-request-id'
    }
}

context = type('Context', (), {
    'aws_request_id': 'test-request-id',
    'function_name': 'test-function'
})()

response = handler(event, context)
print(f"Status: {response['statusCode']}")
print(f"Body: {response['body']}")
```

---

## Environment Configuration

**`.env` file (NEVER commit):**

```bash
# Environment
ENVIRONMENT=dev

# AWS
AWS_REGION=ap-south-1
AWS_PROFILE=default

# DynamoDB
DYNAMODB_ENDPOINT=http://localhost:8001
DYNAMODB_TABLE=orders-dev

# LocalStack
USE_LOCALSTACK=true
LOCALSTACK_ENDPOINT=http://localhost:4566

# S3
S3_BUCKET=risk-data-dev

# Logging
LOG_LEVEL=DEBUG
```

**Load in FastAPI:**

```python
# app/core/config.py
from pydantic_settings import BaseSettings
from dotenv import load_dotenv
import os

load_dotenv()  # Load .env file

class Settings(BaseSettings):
    ENVIRONMENT: str = "dev"
    AWS_REGION: str = "ap-south-1"
    DYNAMODB_ENDPOINT: str = ""
    USE_LOCALSTACK: bool = False
    LOG_LEVEL: str = "INFO"
    
    class Config:
        case_sensitive = True
        env_file = ".env"

settings = Settings()
```

**`.gitignore` (ensure .env is ignored):**

```
.env
.env.local
.env.*.local
*.pyc
__pycache__/
.pytest_cache/
htmlcov/
.coverage
```

---

## Complete Development Workflow

### 1. Start Local Services

```bash
# Terminal 1: LocalStack (optional)
docker-compose up localstack

# Terminal 2: DynamoDB Local (if not using LocalStack)
docker run -d -p 8001:8000 amazon/dynamodb-local

# Terminal 3: FastAPI
cd backend
uvicorn app.main:app --reload

# Terminal 4: Angular
cd frontend
ng serve
```

### 2. Initialize Local Data

```bash
# Create tables
python scripts/create_local_tables.py

# Seed data (optional)
python scripts/seed_data.py
```

### 3. Verify Services

```bash
# FastAPI health
curl http://localhost:8000/health

# Angular app
open http://localhost:4200

# Swagger docs
open http://localhost:8000/docs
```

### 4. Make Code Changes

Both FastAPI and Angular have auto-reload enabled:
- FastAPI: uvicorn detects `.py` file changes
- Angular: webpack dev server detects `.ts`, `.html`, `.scss` changes

### 5. Test Changes

```bash
# Backend unit tests
pytest tests/test_services/

# Backend integration tests
pytest tests/integration/

# Frontend tests
ng test
```

### 6. Test Lambda Locally

```bash
# Using SAM
sam local start-api

# Using Serverless
serverless offline start
```

### 7. Deploy to Dev

```bash
# Serverless Framework
serverless deploy --stage dev

# SAM
sam deploy --parameter-overrides Environment=dev
```

---

## WALKTHROUGH.md Requirements

**`WALKTHROUGH.md` is the single source of truth for:**
- ✅ Onboarding new developers
- ✅ Local development setup
- ✅ Testing procedures
- ✅ Deployment instructions
- ✅ Operational procedures
- ✅ Troubleshooting guides

### When to Update WALKTHROUGH.md

1. **New APIs are added**
   - Document endpoint (method, path, request/response schemas)
   - Include curl examples
   - Add to API inventory section

2. **Local setup changes**
   - New dependencies or tools
   - Configuration file changes
   - Environment variables
   - Database schema changes

3. **Infrastructure changes**
   - New AWS resources
   - IAM permissions updates
   - Architecture modifications
   - New deployment steps

4. **Deployment process changes**
   - New environment configurations
   - Rollback procedures
   - Migration steps

### WALKTHROUGH.md Template

```markdown
# Project Walkthrough

## Table of Contents
1. [Prerequisites](#prerequisites)
2. [Local Setup](#local-setup)
3. [Development](#development)
4. [Testing](#testing)
5. [Deployment](#deployment)
6. [API Documentation](#api-documentation)
7. [Troubleshooting](#troubleshooting)

## Prerequisites

- Python 3.12+
- Node.js 20+
- Docker Desktop
- AWS CLI v2

## Local Setup

### Backend (FastAPI)

```bash
cd backend
python -m venv venv
source venv/bin/activate  # Windows: venv\Scripts\activate
pip install -r requirements.txt
```

### Frontend (Angular)

```bash
cd frontend
npm install
```

### Local AWS Services

```bash
docker-compose up -d localstack
python scripts/create_local_tables.py
```

## Development

### Start Services

```bash
# Backend
uvicorn app.main:app --reload

# Frontend
ng serve
```

### Access Points

- FastAPI: http://localhost:8000
- Swagger: http://localhost:8000/docs
- Angular: http://localhost:4200

## Testing

### Backend

```bash
pytest --cov=app
```

### Frontend

```bash
ng test
```

## Deployment

### Dev Environment

```bash
serverless deploy --stage dev
```

### Production Environment

```bash
serverless deploy --stage prod
```

## API Documentation

### Endpoints

#### POST /api/orders
Create a new order.

**Request:**
```json
{
  "user_id": "uuid",
  "items": [
    {"product_id": "string", "quantity": 1, "price": 10.00}
  ],
  "total_amount": 10.00
}
```

**Response: 201 Created**
```json
{
  "id": "uuid",
  "user_id": "uuid",
  "items": [...],
  "total_amount": 10.00,
  "status": "pending",
  "created_at": "2024-06-11T10:00:00Z"
}
```

## Troubleshooting

### Issue: Cannot connect to DynamoDB Local

**Solution:**
```bash
docker ps  # Check if container is running
docker logs dynamodb-local
docker restart dynamodb-local
```
```

---

## Debugging

### FastAPI Debugging

**VS Code `launch.json`:**

```json
{
  "version": "0.2.0",
  "configurations": [
    {
      "name": "FastAPI",
      "type": "python",
      "request": "launch",
      "module": "uvicorn",
      "args": [
        "app.main:app",
        "--reload",
        "--host", "0.0.0.0",
        "--port", "8000"
      ],
      "jinja": true,
      "justMyCode": false,
      "env": {
        "ENVIRONMENT": "dev",
        "DYNAMODB_ENDPOINT": "http://localhost:8001"
      }
    }
  ]
}
```

**Using debugpy:**

```python
# Add to app/main.py for debugging
import debugpy

if os.getenv("ENVIRONMENT") == "dev":
    debugpy.listen(("0.0.0.0", 5678))
    print("⏳ Waiting for debugger attach on port 5678...")
    debugpy.wait_for_client()
    print("✅ Debugger attached!")
```

### Angular Debugging

Use Chrome DevTools or VS Code debugger.

**VS Code `launch.json`:**

```json
{
  "version": "0.2.0",
  "configurations": [
    {
      "type": "chrome",
      "request": "launch",
      "name": "Angular",
      "url": "http://localhost:4200",
      "webRoot": "${workspaceFolder}/frontend/src"
    }
  ]
}
```

### Lambda Debugging

```bash
# SAM
sam local invoke -d 5858 -e events/event.json

# Attach debugger to port 5858 in VS Code
```

---

## Best Practices

1. ✅ Always use Angular proxy (never direct CORS)
2. ✅ Use DynamoDB Local or LocalStack (never AWS for local dev)
3. ✅ Test Lambda compatibility with SAM/Serverless offline
4. ✅ Keep `.env` in `.gitignore`
5. ✅ Update `WALKTHROUGH.md` with every change
6. ✅ Run tests before committing
7. ✅ Use `pytest --cov` to maintain >80% coverage
8. ✅ Use relative API paths in Angular services
9. ✅ Keep Lambda handler compatible with Mangum
10. ✅ Document all API changes in Swagger/OpenAPI
