# FastAPI in AWS Lambda - Complete Guide

## Overview

Run FastAPI applications serverlessly using AWS Lambda with API Gateway or ALB (Application Load Balancer).

**Benefits:**
- Pay only for requests (no idle costs)
- Auto-scaling
- No server management
- Cold start: 1-3 seconds (acceptable for most APIs)

---

## Architecture Options

### Option 1: API Gateway + Lambda (Recommended for REST APIs)
```
Client → API Gateway (HTTP API or REST API) → Lambda (FastAPI) → DynamoDB/RDS
```

**Pros:**
- Built-in throttling, caching, API keys
- WebSocket support (API Gateway WebSocket)
- Request/response transformations

**Cons:**
- 29-second timeout limit
- 6MB payload limit (REST API), 10MB (HTTP API)

### Option 2: ALB + Lambda (Recommended for internal APIs)
```
Client → Application Load Balancer → Lambda (FastAPI) → DynamoDB/RDS
```

**Pros:**
- Lower cost for high-volume internal APIs
- Same VPC as other resources
- Better for microservices

**Cons:**
- No built-in API keys or throttling
- Requires VPC setup

---

## Project Structure

```
my-fastapi-lambda/
├── app/
│   ├── __init__.py
│   ├── main.py              # FastAPI app
│   ├── routers/
│   │   ├── __init__.py
│   │   └── orders.py        # Route handlers
│   ├── services/
│   │   ├── __init__.py
│   │   └── order_service.py # Business logic
│   ├── models/
│   │   ├── __init__.py
│   │   └── order.py         # Pydantic schemas
│   └── core/
│       ├── __init__.py
│       ├── config.py        # Settings
│       └── database.py      # DB connection
├── lambda_handler.py        # Lambda entry point
├── requirements.txt
├── Dockerfile               # For Lambda container image
├── template.yaml            # SAM template
└── tests/
    └── test_orders.py
```

---

## Implementation

### 1. FastAPI Application (`app/main.py`)

```python
from fastapi import FastAPI
from fastapi.middleware.cors import CORSMiddleware
from mangum import Mangum
import logging

from app.routers import orders
from app.core.config import settings

# Configure logging for Lambda
logging.basicConfig(level=logging.INFO)
logger = logging.getLogger(__name__)

# Create FastAPI app
app = FastAPI(
    title=settings.PROJECT_NAME,
    version="1.0.0",
    docs_url="/docs" if settings.ENVIRONMENT != "production" else None,
    redoc_url="/redoc" if settings.ENVIRONMENT != "production" else None
)

# CORS middleware
app.add_middleware(
    CORSMiddleware,
    allow_origins=settings.ALLOWED_ORIGINS,
    allow_credentials=True,
    allow_methods=["*"],
    allow_headers=["*"],
)

# Include routers
app.include_router(orders.router, prefix="/api/v1/orders", tags=["orders"])

# Health check endpoint
@app.get("/health")
async def health_check():
    return {"status": "healthy", "environment": settings.ENVIRONMENT}

# Root endpoint
@app.get("/")
async def root():
    return {"message": "FastAPI on AWS Lambda", "version": "1.0.0"}
```

### 2. Lambda Handler (`lambda_handler.py`)

```python
from mangum import Mangum
from app.main import app

# Mangum adapter converts ASGI (FastAPI) to Lambda event format
handler = Mangum(app, lifespan="off")

# Lambda entry point
def lambda_handler(event, context):
    """
    AWS Lambda handler for FastAPI application.
    
    Args:
        event: API Gateway or ALB event
        context: Lambda context object
        
    Returns:
        API Gateway/ALB formatted response
    """
    return handler(event, context)
```

### 3. Configuration (`app/core/config.py`)

```python
from pydantic_settings import BaseSettings
from typing import List
import os

class Settings(BaseSettings):
    # Project
    PROJECT_NAME: str = "FastAPI Lambda"
    ENVIRONMENT: str = os.getenv("ENVIRONMENT", "dev")
    
    # CORS
    ALLOWED_ORIGINS: List[str] = ["*"]
    
    # Database (DynamoDB)
    DYNAMODB_TABLE: str = os.getenv("DYNAMODB_TABLE", "orders")
    AWS_REGION: str = os.getenv("AWS_REGION", "us-east-1")
    
    # Secrets (from AWS Secrets Manager)
    DB_SECRET_ARN: str = os.getenv("DB_SECRET_ARN", "")
    
    class Config:
        case_sensitive = True

settings = Settings()
```

### 4. Router Example (`app/routers/orders.py`)

```python
from fastapi import APIRouter, HTTPException, Depends
from typing import List
from uuid import UUID, uuid4

from app.models.order import CreateOrderRequest, OrderResponse
from app.services.order_service import OrderService

router = APIRouter()

def get_order_service() -> OrderService:
    """Dependency injection for OrderService."""
    return OrderService()

@router.post("", response_model=OrderResponse, status_code=201)
async def create_order(
    order: CreateOrderRequest,
    service: OrderService = Depends(get_order_service)
) -> OrderResponse:
    """Create a new order (SF-001)."""
    try:
        return await service.create_order(order)
    except ValueError as e:
        raise HTTPException(status_code=400, detail=str(e))
    except Exception as e:
        raise HTTPException(status_code=500, detail="Failed to create order")

@router.get("/{order_id}", response_model=OrderResponse)
async def get_order(
    order_id: UUID,
    service: OrderService = Depends(get_order_service)
) -> OrderResponse:
    """Get order by ID (SF-002)."""
    order = await service.get_order(order_id)
    if not order:
        raise HTTPException(status_code=404, detail="Order not found")
    return order

@router.get("", response_model=List[OrderResponse])
async def list_orders(
    user_id: UUID,
    page: int = 1,
    limit: int = 20,
    service: OrderService = Depends(get_order_service)
) -> List[OrderResponse]:
    """List orders for a user (SF-003)."""
    return await service.list_orders(user_id, page, limit)
```

### 5. Service Layer (`app/services/order_service.py`)

```python
import boto3
from decimal import Decimal
from uuid import UUID, uuid4
from datetime import datetime
from typing import List, Optional

from app.models.order import CreateOrderRequest, OrderResponse
from app.core.config import settings

class OrderService:
    def __init__(self):
        self.dynamodb = boto3.resource('dynamodb', region_name=settings.AWS_REGION)
        self.table = self.dynamodb.Table(settings.DYNAMODB_TABLE)
    
    async def create_order(self, order: CreateOrderRequest) -> OrderResponse:
        """Create order in DynamoDB."""
        order_id = uuid4()
        user_id = order.user_id
        
        item = {
            'PK': f'USER#{user_id}',
            'SK': f'ORDER#{order_id}',
            'order_id': str(order_id),
            'user_id': str(user_id),
            'items': [item.model_dump() for item in order.items],
            'total_amount': Decimal(str(order.total_amount)),
            'status': 'pending',
            'created_at': datetime.utcnow().isoformat()
        }
        
        self.table.put_item(Item=item)
        
        return OrderResponse(
            id=order_id,
            user_id=user_id,
            items=order.items,
            total_amount=order.total_amount,
            status='pending',
            created_at=datetime.utcnow()
        )
    
    async def get_order(self, order_id: UUID) -> Optional[OrderResponse]:
        """Get order from DynamoDB."""
        # In production, you'd need the user_id to construct PK
        # This is a simplified example
        response = self.table.scan(
            FilterExpression='order_id = :order_id',
            ExpressionAttributeValues={':order_id': str(order_id)}
        )
        
        items = response.get('Items', [])
        if not items:
            return None
        
        item = items[0]
        return OrderResponse(
            id=UUID(item['order_id']),
            user_id=UUID(item['user_id']),
            items=item['items'],
            total_amount=float(item['total_amount']),
            status=item['status'],
            created_at=datetime.fromisoformat(item['created_at'])
        )
    
    async def list_orders(
        self,
        user_id: UUID,
        page: int = 1,
        limit: int = 20
    ) -> List[OrderResponse]:
        """List orders for a user with pagination."""
        response = self.table.query(
            KeyConditionExpression='PK = :pk AND begins_with(SK, :sk)',
            ExpressionAttributeValues={
                ':pk': f'USER#{user_id}',
                ':sk': 'ORDER#'
            },
            Limit=limit
        )
        
        items = response.get('Items', [])
        return [
            OrderResponse(
                id=UUID(item['order_id']),
                user_id=UUID(item['user_id']),
                items=item['items'],
                total_amount=float(item['total_amount']),
                status=item['status'],
                created_at=datetime.fromisoformat(item['created_at'])
            )
            for item in items
        ]
```

### 6. Pydantic Models (`app/models/order.py`)

```python
from pydantic import BaseModel, Field
from typing import List
from uuid import UUID
from datetime import datetime
from decimal import Decimal

class OrderItem(BaseModel):
    product_id: str
    quantity: int = Field(..., gt=0)
    price: Decimal = Field(..., gt=0)

class CreateOrderRequest(BaseModel):
    user_id: UUID
    items: List[OrderItem] = Field(..., min_length=1)
    total_amount: Decimal = Field(..., gt=0)

class OrderResponse(BaseModel):
    id: UUID
    user_id: UUID
    items: List[OrderItem]
    total_amount: Decimal
    status: str
    created_at: datetime
    
    class Config:
        json_encoders = {
            UUID: str,
            Decimal: float
        }
```

### 7. Requirements (`requirements.txt`)

```txt
fastapi==0.109.0
mangum==0.17.0
pydantic==2.5.0
pydantic-settings==2.1.0
boto3==1.34.0
uvicorn==0.27.0
```

### 8. SAM Template (`template.yaml`)

```yaml
AWSTemplateFormatVersion: '2010-09-09'
Transform: AWS::Serverless-2016-10-31
Description: FastAPI Lambda Application

Globals:
  Function:
    Timeout: 30
    MemorySize: 512
    Runtime: python3.11
    Environment:
      Variables:
        ENVIRONMENT: !Ref Environment
        AWS_REGION: !Ref AWS::Region
        DYNAMODB_TABLE: !Ref OrdersTable

Parameters:
  Environment:
    Type: String
    Default: dev
    AllowedValues:
      - dev
      - staging
      - production

Resources:
  # Lambda Function
  FastAPIFunction:
    Type: AWS::Serverless::Function
    Properties:
      FunctionName: !Sub 'fastapi-lambda-${Environment}'
      CodeUri: .
      Handler: lambda_handler.lambda_handler
      Architectures:
        - x86_64
      Policies:
        - DynamoDBCrudPolicy:
            TableName: !Ref OrdersTable
      Events:
        ApiEvent:
          Type: HttpApi
          Properties:
            Path: /{proxy+}
            Method: ANY
            ApiId: !Ref HttpApi
  
  # HTTP API (API Gateway v2)
  HttpApi:
    Type: AWS::Serverless::HttpApi
    Properties:
      StageName: !Ref Environment
      CorsConfiguration:
        AllowOrigins:
          - '*'
        AllowMethods:
          - GET
          - POST
          - PUT
          - DELETE
          - OPTIONS
        AllowHeaders:
          - '*'
  
  # DynamoDB Table
  OrdersTable:
    Type: AWS::DynamoDB::Table
    Properties:
      TableName: !Sub 'orders-${Environment}'
      BillingMode: PAY_PER_REQUEST
      AttributeDefinitions:
        - AttributeName: PK
          AttributeType: S
        - AttributeName: SK
          AttributeType: S
      KeySchema:
        - AttributeName: PK
          KeyType: HASH
        - AttributeName: SK
          KeyType: RANGE
      StreamSpecification:
        StreamViewType: NEW_AND_OLD_IMAGES

Outputs:
  ApiUrl:
    Description: API Gateway endpoint URL
    Value: !Sub 'https://${HttpApi}.execute-api.${AWS::Region}.amazonaws.com/${Environment}'
  
  FunctionArn:
    Description: Lambda Function ARN
    Value: !GetAtt FastAPIFunction.Arn
  
  TableName:
    Description: DynamoDB Table Name
    Value: !Ref OrdersTable
```

---

## Deployment

### Option 1: SAM CLI (Recommended)

```bash
# Install SAM CLI
# https://docs.aws.amazon.com/serverless-application-model/latest/developerguide/install-sam-cli.html

# Build
sam build

# Deploy (first time)
sam deploy --guided

# Deploy (subsequent)
sam deploy

# Test locally
sam local start-api
curl http://localhost:3000/health
```

### Option 2: AWS CDK (Python)

Create `infrastructure/app.py`:

```python
from aws_cdk import (
    Stack,
    aws_lambda as lambda_,
    aws_apigatewayv2 as apigw,
    aws_apigatewayv2_integrations as integrations,
    aws_dynamodb as dynamodb,
    Duration,
    RemovalPolicy
)
from constructs import Construct

class FastAPILambdaStack(Stack):
    def __init__(self, scope: Construct, construct_id: str, **kwargs) -> None:
        super().__init__(scope, construct_id, **kwargs)
        
        # DynamoDB Table
        table = dynamodb.Table(
            self, "OrdersTable",
            partition_key=dynamodb.Attribute(
                name="PK",
                type=dynamodb.AttributeType.STRING
            ),
            sort_key=dynamodb.Attribute(
                name="SK",
                type=dynamodb.AttributeType.STRING
            ),
            billing_mode=dynamodb.BillingMode.PAY_PER_REQUEST,
            removal_policy=RemovalPolicy.DESTROY,  # Change for production
            stream=dynamodb.StreamViewType.NEW_AND_OLD_IMAGES
        )
        
        # Lambda Function
        fastapi_lambda = lambda_.Function(
            self, "FastAPIFunction",
            runtime=lambda_.Runtime.PYTHON_3_11,
            handler="lambda_handler.lambda_handler",
            code=lambda_.Code.from_asset("../"),
            timeout=Duration.seconds(30),
            memory_size=512,
            environment={
                "ENVIRONMENT": "dev",
                "DYNAMODB_TABLE": table.table_name
            }
        )
        
        # Grant DynamoDB permissions
        table.grant_read_write_data(fastapi_lambda)
        
        # HTTP API Gateway
        http_api = apigw.HttpApi(
            self, "FastAPIHttpApi",
            default_integration=integrations.HttpLambdaIntegration(
                "FastAPIIntegration",
                fastapi_lambda
            ),
            cors_preflight=apigw.CorsPreflightOptions(
                allow_origins=["*"],
                allow_methods=[apigw.CorsHttpMethod.ANY],
                allow_headers=["*"]
            )
        )
```

Deploy:
```bash
cd infrastructure
cdk deploy
```

### Option 3: Docker (Lambda Container Image)

Create `Dockerfile`:

```dockerfile
FROM public.ecr.aws/lambda/python:3.11

# Copy requirements
COPY requirements.txt ${LAMBDA_TASK_ROOT}

# Install dependencies
RUN pip install -r requirements.txt

# Copy application code
COPY app/ ${LAMBDA_TASK_ROOT}/app/
COPY lambda_handler.py ${LAMBDA_TASK_ROOT}

# Set the CMD to your handler
CMD ["lambda_handler.lambda_handler"]
```

Deploy:
```bash
# Build and push to ECR
aws ecr create-repository --repository-name fastapi-lambda
docker build -t fastapi-lambda .
docker tag fastapi-lambda:latest <account-id>.dkr.ecr.<region>.amazonaws.com/fastapi-lambda:latest
aws ecr get-login-password --region <region> | docker login --username AWS --password-stdin <account-id>.dkr.ecr.<region>.amazonaws.com
docker push <account-id>.dkr.ecr.<region>.amazonaws.com/fastapi-lambda:latest

# Create Lambda function with container image
aws lambda create-function \
  --function-name fastapi-lambda \
  --package-type Image \
  --code ImageUri=<account-id>.dkr.ecr.<region>.amazonaws.com/fastapi-lambda:latest \
  --role <lambda-execution-role-arn>
```

---

## Testing

### Local Testing with SAM

```bash
# Start API locally
sam local start-api

# Test endpoints
curl http://localhost:3000/health
curl -X POST http://localhost:3000/api/v1/orders \
  -H "Content-Type: application/json" \
  -d '{"user_id": "123e4567-e89b-12d3-a456-426614174000", "items": [{"product_id": "prod-1", "quantity": 2, "price": 24.99}], "total_amount": 49.98}'
```

### Unit Tests (`tests/test_orders.py`)

```python
import pytest
from fastapi.testclient import TestClient
from app.main import app

client = TestClient(app)

def test_health_check():
    response = client.get("/health")
    assert response.status_code == 200
    assert response.json()["status"] == "healthy"

def test_create_order__SF_001():
    order_data = {
        "user_id": "123e4567-e89b-12d3-a456-426614174000",
        "items": [
            {"product_id": "prod-1", "quantity": 2, "price": 24.99}
        ],
        "total_amount": 49.98
    }
    
    response = client.post("/api/v1/orders", json=order_data)
    assert response.status_code == 201
    data = response.json()
    assert data["user_id"] == order_data["user_id"]
    assert data["total_amount"] == order_data["total_amount"]

def test_get_order__SF_002():
    # First create an order
    order_data = {
        "user_id": "123e4567-e89b-12d3-a456-426614174000",
        "items": [{"product_id": "prod-1", "quantity": 1, "price": 10.00}],
        "total_amount": 10.00
    }
    create_response = client.post("/api/v1/orders", json=order_data)
    order_id = create_response.json()["id"]
    
    # Get the order
    response = client.get(f"/api/v1/orders/{order_id}")
    assert response.status_code == 200
    assert response.json()["id"] == order_id
```

---

## Performance Optimization

### 1. Cold Start Reduction

```python
# Use provisioned concurrency for critical APIs
# In SAM template:
FastAPIFunction:
  Type: AWS::Serverless::Function
  Properties:
    ProvisionedConcurrencyConfig:
      ProvisionedConcurrentExecutions: 5
```

### 2. Connection Pooling (RDS)

```python
# Use RDS Proxy for connection pooling
# In lambda_handler.py
import functools

@functools.lru_cache()
def get_db_connection():
    """Cache database connection across invocations."""
    return create_connection()
```

### 3. Lambda Layers

Extract large dependencies to Lambda layers:

```bash
# Create layer
mkdir python
pip install -r requirements.txt -t python/
zip -r layer.zip python/

# Create layer version
aws lambda publish-layer-version \
  --layer-name fastapi-deps \
  --zip-file fileb://layer.zip \
  --compatible-runtimes python3.11
```

### 4. Environment Variable Caching

```python
import os
import functools

@functools.lru_cache()
def get_settings():
    """Cache settings across invocations."""
    return Settings()
```

---

## Monitoring

### CloudWatch Logs

```python
import logging
import json

logger = logging.getLogger(__name__)
logger.setLevel(logging.INFO)

# Structured logging
logger.info(json.dumps({
    'event': 'order_created',
    'order_id': str(order_id),
    'user_id': str(user_id),
    'request_id': context.aws_request_id
}))
```

### X-Ray Tracing

```python
from aws_xray_sdk.core import xray_recorder
from aws_xray_sdk.core import patch_all

# Patch boto3, requests, etc.
patch_all()

@xray_recorder.capture('create_order')
async def create_order(order: CreateOrderRequest) -> OrderResponse:
    pass
```

### CloudWatch Metrics

```python
import boto3

cloudwatch = boto3.client('cloudwatch')

cloudwatch.put_metric_data(
    Namespace='FastAPILambda',
    MetricData=[{
        'MetricName': 'OrdersCreated',
        'Value': 1,
        'Unit': 'Count'
    }]
)
```

---

## Best Practices

1. **Keep Lambda packages small** (< 50MB unzipped)
2. **Use Lambda layers** for large dependencies
3. **Cache connections** outside handler function
4. **Use async/await** for I/O operations
5. **Set appropriate timeout** (API Gateway max: 29s)
6. **Use provisioned concurrency** for critical APIs
7. **Enable X-Ray tracing** for debugging
8. **Use DynamoDB** over RDS for better cold start
9. **Implement health checks** at `/health`
10. **Version your APIs** (`/api/v1/`)

---

## Cost Optimization

**Lambda Pricing:**
- Free tier: 1M requests/month + 400,000 GB-seconds compute
- After: $0.20 per 1M requests + $0.0000166667 per GB-second

**Example: 1M requests/month, 512MB, 500ms avg:**
- Requests: $0.20
- Compute: 1M × 0.5s × 0.5GB × $0.0000166667 = $4.17
- **Total: ~$4.37/month**

Compare to EC2 t3.small (24/7): ~$15/month

---

## Troubleshooting

### Issue: 502 Bad Gateway
**Cause:** Lambda timeout or error  
**Fix:** Check CloudWatch Logs, increase timeout

### Issue: Cold start too slow
**Cause:** Large package size  
**Fix:** Use Lambda layers, remove unused dependencies

### Issue: DynamoDB throttling
**Cause:** Exceeded capacity  
**Fix:** Use on-demand billing or increase provisioned capacity

### Issue: CORS errors
**Cause:** Missing CORS headers  
**Fix:** Add CORS middleware in FastAPI

---

## Resources

- [Mangum Documentation](https://mangum.io/)
- [AWS SAM Documentation](https://docs.aws.amazon.com/serverless-application-model/)
- [FastAPI Documentation](https://fastapi.tiangolo.com/)
- [AWS Lambda Best Practices](https://docs.aws.amazon.com/lambda/latest/dg/best-practices.html)
