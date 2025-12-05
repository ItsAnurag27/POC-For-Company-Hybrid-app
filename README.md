# Hybrid Mobile App + ML Model AWS Deployment PoC

## Table of Contents

- [Overview](#overview)
- [Architecture](#architecture)
- [Prerequisites](#prerequisites)
- [Repository Structure](#repository-structure)
- [Quick Start](#quick-start)
- [Detailed Setup Guide](#detailed-setup-guide)
- [CI/CD Pipeline](#cicd-pipeline)
- [Deployment](#deployment)
- [Monitoring & Logging](#monitoring--logging)
- [Model Updates](#model-updates)
- [Troubleshooting](#troubleshooting)
- [Future Enhancements](#future-enhancements)
- [Team Contacts](#team-contacts)

---

## Overview

This Proof of Concept (PoC) demonstrates a production-ready AWS deployment pipeline for a hybrid mobile application with an integrated ML prediction model. The solution automates the entire process from code commit to production deployment with zero-downtime rollouts.

### Key Objectives

- ✅ Automated CI/CD pipeline for backend and ML services
- ✅ Containerized services running on AWS ECS Fargate
- ✅ Multi-environment support (Dev, Stage, Prod)
- ✅ Serverless deployment with high availability
- ✅ Secure model storage and version management
- ✅ Comprehensive monitoring and logging

### Technology Stack

| Component | Technology |
|-----------|-----------|
| **Container Orchestration** | AWS ECS (Fargate) |
| **CI/CD** | AWS CodePipeline + CodeBuild |
| **Container Registry** | Amazon ECR |
| **Load Balancing** | Application Load Balancer (ALB) |
| **Model Storage** | S3 + SSM Parameter Store |
| **Database** | RDS/DynamoDB (optional) |
| **Monitoring** | CloudWatch Logs + Metrics |
| **Infrastructure as Code** | Terraform (recommended) |
| **Security** | IAM Roles, Secrets Manager, ACM |

---

## Architecture

### High-Level Diagram

```
┌─────────────────────────────────────────────────────────────────┐
│                    MOBILE APP CLIENTS                           │
│              (React Native / Flutter / Ionic)                   │
└─────────────────────────┬───────────────────────────────────────┘
                          │
                    HTTPS (ACM SSL)
                          │
        ┌─────────────────▼─────────────────┐
        │   Application Load Balancer       │
        │      (ALB - Public Subnets)       │
        └──┬──────────────────────┬─────────┘
           │                      │
    Path: /api/*          Path: /ml/*
           │                      │
    ┌──────▼──────┐        ┌──────▼──────┐
    │   Backend   │        │  ML Model   │
    │   Service   │        │   Service   │
    │  (ECS Task) │        │  (ECS Task) │
    └──────┬──────┘        └──────┬──────┘
           │                      │
           └──────┬───────────────┘
                  │
        ┌─────────▼──────────┐
        │   Amazon ECR       │
        │  (Image Registry)  │
        └────────┬───────────┘
                 │
    ┌────────────┼────────────┐
    │            │            │
┌───▼──┐    ┌────▼──┐    ┌───▼──┐
│ S3   │    │Secrets│    │Cloud │
│Model │    │Manager│    │Watch │
└──────┘    └───────┘    └──────┘
```

### Component Details

#### 1. **Source Control**
- **GitHub / CodeCommit**: Version control for three repositories:
  - `mobile-app`: Hybrid mobile application (React Native/Flutter)
  - `backend-api`: REST API service (Node.js/Python/Java)
  - `ml-model-service`: ML prediction service (Python FastAPI/Flask)

#### 2. **CI/CD Pipeline**
- **AWS CodePipeline**: Orchestrates the entire deployment workflow
- **AWS CodeBuild**: Builds Docker images, runs tests, pushes to ECR
- **Triggers**: Automatically on code push to `main` (production) or `develop` (staging)

#### 3. **Container Registry & Hosting**
- **Amazon ECR**: Private Docker image repository
- **AWS ECS (Fargate)**: Serverless container orchestration
  - No server management required
  - Auto-scaling capability
  - Built-in IAM integration

#### 4. **Networking**
- **VPC**: Isolated network environment
  - 2 Public Subnets (ALB)
  - 2 Private Subnets (ECS tasks, databases)
  - Multi-AZ deployment for high availability
- **Application Load Balancer**: Distributes traffic with path-based routing

#### 5. **Data & Storage**
- **S3**: Model artifact storage (pkl, h5, ONNX formats)
- **RDS/DynamoDB**: Application database (optional for PoC)
- **Secrets Manager**: Secure credential storage

#### 6. **Monitoring & Observability**
- **CloudWatch Logs**: Centralized logging for all services
- **CloudWatch Metrics**: Performance monitoring and alarms
- **CloudWatch Dashboard**: Real-time service health visualization

---

## Prerequisites

### AWS Account & Permissions
- AWS account with billing enabled
- IAM user with at least these permissions:
  - ECS, ECR, CodePipeline, CodeBuild, CodeDeploy
  - IAM (for role creation)
  - VPC, ALB, CloudWatch
  - S3, Secrets Manager

### Local Development
- **Docker**: v20.10+ (for local testing)
- **AWS CLI**: v2.x (`aws --version`)
- **Git**: v2.30+
- **Python 3.9+** (for local ML model testing)
- **Node.js 16+** or **Java 11+** (for backend testing)

### AWS CLI Configuration

```bash
# Configure AWS credentials
aws configure

# Verify access
aws sts get-caller-identity
```

### Environment Variables Template

Create `.env` files for each environment:

```bash
# .env.dev
AWS_REGION=ap-south-1
ENVIRONMENT=dev
ECR_REGISTRY=<ACCOUNT_ID>.dkr.ecr.ap-south-1.amazonaws.com
ALB_DNS=<ALB-DEV-DNS>.ap-south-1.elb.amazonaws.com
LOG_LEVEL=DEBUG
MODEL_VERSION=v1.0

# .env.prod
AWS_REGION=ap-south-1
ENVIRONMENT=prod
ECR_REGISTRY=<ACCOUNT_ID>.dkr.ecr.ap-south-1.amazonaws.com
ALB_DNS=api.company.com  # Custom domain
LOG_LEVEL=INFO
MODEL_VERSION=v1.2
```

---

## Repository Structure

### 1. Backend API Repository

```
backend-api/
├── src/
│   ├── main.py (or app.js, etc.)
│   ├── routes/
│   │   ├── health.py
│   │   └── predict.py
│   ├── models/
│   │   └── prediction.py
│   └── config/
│       └── settings.py
├── tests/
│   ├── test_predict.py
│   └── test_health.py
├── Dockerfile
├── buildspec.yml
├── requirements.txt (Python) or package.json (Node.js)
└── README.md
```

**Example FastAPI Backend** (`src/main.py`):

```python
from fastapi import FastAPI
from fastapi.responses import JSONResponse
import httpx
import os

app = FastAPI(title="Backend API")

ML_SERVICE_URL = os.getenv("ML_SERVICE_URL", "http://ml-model-service:8000")

@app.get("/health")
async def health_check():
    return {"status": "healthy", "service": "backend-api"}

@app.post("/api/predict")
async def predict(data: dict):
    try:
        async with httpx.AsyncClient() as client:
            response = await client.post(
                f"{ML_SERVICE_URL}/predict",
                json=data,
                timeout=30.0
            )
            response.raise_for_status()
        return response.json()
    except Exception as e:
        return JSONResponse(
            status_code=500,
            content={"error": str(e)}
        )
```

### 2. ML Model Service Repository

```
ml-model-service/
├── app/
│   ├── main.py
│   ├── models.py (Pydantic schemas)
│   ├── ml_pipeline.py
│   └── utils.py
├── models/
│   └── model.pkl (or .h5)
├── tests/
│   └── test_predict.py
├── Dockerfile
├── buildspec.yml
├── requirements.txt
└── README.md
```

**Example FastAPI ML Service** (`app/main.py`):

```python
from fastapi import FastAPI, HTTPException
import pickle
import boto3
import os
from app.models import PredictionRequest, PredictionResponse

app = FastAPI(title="ML Model Service")

MODEL_PATH = "/app/models/model.pkl"
S3_BUCKET = os.getenv("S3_MODEL_BUCKET")
S3_KEY = os.getenv("S3_MODEL_KEY", "models/model.pkl")

@app.on_event("startup")
async def load_model():
    """Load model from S3 or local path on startup"""
    global model
    if S3_BUCKET:
        s3_client = boto3.client("s3")
        s3_client.download_file(S3_BUCKET, S3_KEY, MODEL_PATH)
    
    with open(MODEL_PATH, "rb") as f:
        model = pickle.load(f)

@app.get("/health")
def health_check():
    return {"status": "healthy", "service": "ml-model-service"}

@app.post("/predict", response_model=PredictionResponse)
async def predict(request: PredictionRequest):
    try:
        prediction = model.predict([request.features])
        confidence = float(model.predict_proba([request.features]).max())
        
        return PredictionResponse(
            prediction=prediction[0],
            confidence=confidence,
            model_version=os.getenv("MODEL_VERSION", "unknown")
        )
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))
```

### 3. Mobile App Repository (Optional PoC)

```
mobile-app/
├── src/
│   ├── services/
│   │   └── api.service.ts
│   ├── screens/
│   └── components/
├── config/
│   ├── dev.config.ts
│   ├── prod.config.ts
│   └── api.client.ts
├── package.json
└── .github/workflows/build.yml (separate CI/CD)
```

---

## Quick Start

### 1. Local Development & Testing

#### Backend API

```bash
cd backend-api

# Create virtual environment
python -m venv venv
source venv/bin/activate  # On Windows: venv\Scripts\activate

# Install dependencies
pip install -r requirements.txt

# Run locally
uvicorn src.main:app --reload --port 8001

# Run tests
pytest tests/ -v

# Build Docker image
docker build -t backend-api:latest .

# Test Docker container
docker run -p 8001:8000 -e ML_SERVICE_URL=http://host.docker.internal:8000 backend-api:latest
```

#### ML Model Service

```bash
cd ml-model-service

python -m venv venv
source venv/bin/activate

pip install -r requirements.txt

# Run locally
uvicorn app.main:app --reload --port 8000

# Run tests
pytest tests/ -v --cov=app

# Build Docker image
docker build -t ml-model-service:latest .

# Test Docker container
docker run -p 8000:8000 ml-model-service:latest
```

### 2. Quick Integration Test (Docker Compose)

Create `docker-compose.yml` for local testing:

```yaml
version: "3.8"

services:
  ml-service:
    build:
      context: ./ml-model-service
    ports:
      - "8000:8000"
    environment:
      - MODEL_VERSION=v1.0
      - LOG_LEVEL=DEBUG

  backend-api:
    build:
      context: ./backend-api
    ports:
      - "8001:8000"
    environment:
      - ML_SERVICE_URL=http://ml-service:8000
      - LOG_LEVEL=DEBUG
    depends_on:
      - ml-service

  # Optional: Reverse proxy for path-based routing
  nginx:
    image: nginx:latest
    ports:
      - "80:80"
    volumes:
      - ./nginx.conf:/etc/nginx/nginx.conf:ro
    depends_on:
      - backend-api
      - ml-service
```

Run locally:

```bash
docker-compose up -d

# Test endpoints
curl http://localhost/api/health
curl -X POST http://localhost/ml/predict -H "Content-Type: application/json" \
  -d '{"features": [1.0, 2.0, 3.0]}'
```

---

## Detailed Setup Guide

### Phase 1: AWS Infrastructure (Days 1-2)

#### Step 1.1: Create VPC & Networking

```bash
# Using AWS CLI (or use Terraform)
aws ec2 create-vpc --cidr-block 10.0.0.0/16 --region ap-south-1

# Create subnets (public & private)
aws ec2 create-subnet --vpc-id vpc-xxx --cidr-block 10.0.1.0/24 \
  --availability-zone ap-south-1a --region ap-south-1

# Repeat for additional subnets...
```

**Better approach: Use Terraform**

```hcl
# terraform/vpc.tf
resource "aws_vpc" "main" {
  cidr_block           = "10.0.0.0/16"
  enable_dns_hostnames = true
  enable_dns_support   = true

  tags = {
    Name = "prediction-app-vpc"
  }
}

resource "aws_subnet" "public" {
  count             = 2
  vpc_id            = aws_vpc.main.id
  cidr_block        = "10.0.${count.index + 1}.0/24"
  availability_zone = data.aws_availability_zones.available.names[count.index]

  tags = {
    Name = "public-subnet-${count.index + 1}"
  }
}

# Similar for private subnets...
```

#### Step 1.2: Create ECS Cluster

```bash
aws ecs create-cluster --cluster-name prediction-app-cluster \
  --region ap-south-1 \
  --tags key=Environment,value=poc key=Project,value=ML-Deployment
```

#### Step 1.3: Create ECR Repositories

```bash
aws ecr create-repository --repository-name backend-api-repo \
  --region ap-south-1 \
  --scan-on-push

aws ecr create-repository --repository-name ml-model-service-repo \
  --region ap-south-1 \
  --scan-on-push
```

#### Step 1.4: Create ALB

```bash
# Create target groups
aws elbv2 create-target-group --name backend-tg \
  --protocol HTTP --port 8000 --vpc-id vpc-xxx \
  --health-check-protocol HTTP \
  --health-check-path /health \
  --health-check-interval-seconds 30 \
  --health-check-timeout-seconds 5 \
  --healthy-threshold-count 2 \
  --unhealthy-threshold-count 2

# Create ALB
aws elbv2 create-load-balancer --name prediction-app-alb \
  --subnets subnet-xxx subnet-yyy \
  --security-groups sg-xxx \
  --scheme internet-facing \
  --type application
```

### Phase 2: Containerization (Days 2-3)

#### Dockerfile for Backend

```dockerfile
# backend-api/Dockerfile
FROM python:3.11-slim

WORKDIR /app

# Install system dependencies
RUN apt-get update && apt-get install -y --no-install-recommends \
    curl \
    && rm -rf /var/lib/apt/lists/*

# Copy and install Python dependencies
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

# Copy application code
COPY src/ ./src/

# Health check
HEALTHCHECK --interval=30s --timeout=10s --start-period=5s --retries=3 \
    CMD curl -f http://localhost:8000/health || exit 1

# Run application
CMD ["uvicorn", "src.main:app", "--host", "0.0.0.0", "--port", "8000"]
```

#### Dockerfile for ML Service

```dockerfile
# ml-model-service/Dockerfile
FROM python:3.11-slim

WORKDIR /app

RUN apt-get update && apt-get install -y --no-install-recommends \
    curl \
    && rm -rf /var/lib/apt/lists/*

COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

COPY app/ ./app/
COPY models/ ./models/

HEALTHCHECK --interval=30s --timeout=10s --start-period=10s --retries=3 \
    CMD curl -f http://localhost:8000/health || exit 1

CMD ["uvicorn", "app.main:app", "--host", "0.0.0.0", "--port", "8000"]
```

#### Test Images Locally

```bash
# Build
docker build -t backend-api:test -f backend-api/Dockerfile backend-api/
docker build -t ml-model-service:test -f ml-model-service/Dockerfile ml-model-service/

# Run and test
docker run --rm -p 8000:8000 ml-model-service:test &
docker run --rm -p 8001:8000 -e ML_SERVICE_URL=http://host.docker.internal:8000 backend-api:test &

# Verify
curl http://localhost:8000/health
curl http://localhost:8001/health
```

### Phase 3: CI/CD Pipeline Setup (Days 3-4)

#### Step 3.1: Create buildspec.yml for Backend

```yaml
# backend-api/buildspec.yml
version: 0.2

env:
  variables:
    IMAGE_REPO_NAME: "backend-api-repo"
    AWS_DEFAULT_REGION: "ap-south-1"
    DOCKERFILE_PATH: "Dockerfile"
  parameter-store:
    SLACK_WEBHOOK: "/devops/slack/webhook"

phases:
  pre_build:
    commands:
      - echo "[$(date)] Starting build for Backend API..."
      - ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
      - ECR_REGISTRY_URL=$ACCOUNT_ID.dkr.ecr.$AWS_DEFAULT_REGION.amazonaws.com
      - IMAGE_REPO_URL=$ECR_REGISTRY_URL/$IMAGE_REPO_NAME
      - IMAGE_TAG=$CODEBUILD_RESOLVED_SOURCE_VERSION
      - echo "Logging in to Amazon ECR..."
      - aws ecr get-login-password --region $AWS_DEFAULT_REGION | docker login --username AWS --password-stdin $ECR_REGISTRY_URL
      - echo "Build started on $(date)"

  build:
    commands:
      - echo "Building Docker image..."
      - docker build -t $IMAGE_REPO_URL:$IMAGE_TAG -f $DOCKERFILE_PATH .
      - docker tag $IMAGE_REPO_URL:$IMAGE_TAG $IMAGE_REPO_URL:latest
      - echo "Running tests..."
      - docker run --rm $IMAGE_REPO_URL:$IMAGE_TAG pytest tests/ -v

  post_build:
    commands:
      - echo "Pushing Docker image to ECR..."
      - docker push $IMAGE_REPO_URL:$IMAGE_TAG
      - docker push $IMAGE_REPO_URL:latest
      - printf '[{"name":"backend-container","imageUri":"%s"}]' $IMAGE_REPO_URL:$IMAGE_TAG > imagedefinitions.json
      - echo "Build completed on $(date)"

artifacts:
  files:
    - imagedefinitions.json
  name: BuildArtifact

cache:
  paths:
    - "/root/.cache/pip/**/*"
    - "/root/.docker/**/*"

reports:
  test-results:
    files:
      - "test-results.xml"
    file-format: "JUNITXML"
```

#### Step 3.2: Create buildspec.yml for ML Service

```yaml
# ml-model-service/buildspec.yml
version: 0.2

env:
  variables:
    IMAGE_REPO_NAME: "ml-model-service-repo"
    AWS_DEFAULT_REGION: "ap-south-1"

phases:
  pre_build:
    commands:
      - echo "[$(date)] Starting build for ML Model Service..."
      - ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
      - ECR_REGISTRY_URL=$ACCOUNT_ID.dkr.ecr.$AWS_DEFAULT_REGION.amazonaws.com
      - IMAGE_REPO_URL=$ECR_REGISTRY_URL/$IMAGE_REPO_NAME
      - IMAGE_TAG=$CODEBUILD_RESOLVED_SOURCE_VERSION
      - aws ecr get-login-password --region $AWS_DEFAULT_REGION | docker login --username AWS --password-stdin $ECR_REGISTRY_URL

  build:
    commands:
      - echo "Building Docker image..."
      - docker build -t $IMAGE_REPO_URL:$IMAGE_TAG .
      - docker tag $IMAGE_REPO_URL:$IMAGE_TAG $IMAGE_REPO_URL:latest
      - echo "Running model tests..."
      - docker run --rm $IMAGE_REPO_URL:$IMAGE_TAG pytest tests/ -v --cov=app

  post_build:
    commands:
      - echo "Pushing Docker image..."
      - docker push $IMAGE_REPO_URL:$IMAGE_TAG
      - docker push $IMAGE_REPO_URL:latest
      - printf '[{"name":"ml-service-container","imageUri":"%s"}]' $IMAGE_REPO_URL:$IMAGE_TAG > imagedefinitions.json

artifacts:
  files:
    - imagedefinitions.json
```

#### Step 3.3: Create CodeBuild Projects

```bash
# Create CodeBuild project for backend
aws codebuild create-project \
  --name backend-api-build \
  --source type=GITHUB,location=https://github.com/your-org/backend-api.git,gitCloneDepth=1 \
  --artifacts type=CODEPIPELINE \
  --environment type=LINUX_CONTAINER,image=aws/codebuild/standard:7.0,computeType=BUILD_GENERAL1_MEDIUM,environmentVariables='[{name=AWS_DEFAULT_REGION,value=ap-south-1,type=PLAINTEXT}]' \
  --service-role arn:aws:iam::ACCOUNT_ID:role/codebuild-role \
  --logs-config cloudWatchLogs={status=ENABLED,groupName=/aws/codebuild/backend-api}

# Create CodeBuild project for ML service
aws codebuild create-project \
  --name ml-model-service-build \
  --source type=GITHUB,location=https://github.com/your-org/ml-model-service.git \
  --artifacts type=CODEPIPELINE \
  --environment type=LINUX_CONTAINER,image=aws/codebuild/standard:7.0,computeType=BUILD_GENERAL1_MEDIUM \
  --service-role arn:aws:iam::ACCOUNT_ID:role/codebuild-role
```

#### Step 3.4: Create CodePipeline

```bash
# Create pipeline configuration
cat > pipeline-config.json <<EOF
{
  "pipeline": {
    "name": "ml-deployment-pipeline",
    "roleArn": "arn:aws:iam::ACCOUNT_ID:role/codepipeline-role",
    "artifactStore": {
      "type": "S3",
      "location": "ml-deployment-artifacts-bucket"
    },
    "stages": [
      {
        "name": "Source",
        "actions": [
          {
            "name": "SourceAction",
            "actionTypeId": {
              "category": "Source",
              "owner": "ThirdParty",
              "provider": "GitHub",
              "version": "1"
            },
            "configuration": {
              "Owner": "your-org",
              "Repo": "backend-api",
              "Branch": "main",
              "OAuthToken": "github-token"
            },
            "outputArtifacts": [{"name": "SourceOutput"}]
          }
        ]
      },
      {
        "name": "Build",
        "actions": [
          {
            "name": "BuildAction",
            "actionTypeId": {
              "category": "Build",
              "owner": "AWS",
              "provider": "CodeBuild",
              "version": "1"
            },
            "configuration": {
              "ProjectName": "backend-api-build"
            },
            "inputArtifacts": [{"name": "SourceOutput"}],
            "outputArtifacts": [{"name": "BuildOutput"}]
          }
        ]
      },
      {
        "name": "Deploy",
        "actions": [
          {
            "name": "DeployToECS",
            "actionTypeId": {
              "category": "Deploy",
              "owner": "AWS",
              "provider": "ECS",
              "version": "1"
            },
            "configuration": {
              "ClusterName": "prediction-app-cluster",
              "ServiceName": "backend-service",
              "FileName": "imagedefinitions.json"
            },
            "inputArtifacts": [{"name": "BuildOutput"}]
          }
        ]
      }
    ]
  }
}
EOF

aws codepipeline create-pipeline --cli-input-json file://pipeline-config.json
```

---

## CI/CD Pipeline

### Pipeline Architecture

```
GitHub/CodeCommit Push
        │
        ▼
   Source Stage
   (Fetch code)
        │
        ▼
   Build Stage
   ├─ Run tests
   ├─ Build Docker image
   ├─ Push to ECR
        │
        ▼
   Deploy Stage (Dev)
   ├─ Update ECS task definition
   ├─ Roll out new tasks
   ├─ Health checks
        │
        ▼
   Manual Approval (for Prod)
        │
        ▼
   Deploy Stage (Prod)
   └─ Blue/Green deployment
```

### Pipeline Configurations

#### Backend Pipeline

| Stage | Action | Trigger |
|-------|--------|---------|
| **Source** | GitHub webhook | Push to `main` |
| **Build** | CodeBuild (buildspec.yml) | Automatic |
| **Test** | Run pytest in container | Automatic |
| **Deploy (Dev)** | ECS rolling update | Automatic |
| **Deploy (Prod)** | ECS blue/green | Manual approval |

#### ML Service Pipeline

Same as backend but with separate CodeBuild project and ECS service.

### Environment-Specific Configurations

#### Development Pipeline

```bash
# Triggers on: develop or feature/* branches
# Deploys to: ECS Fargate (minimal tasks)
# Auto-deploy: Yes
# Manual approval: No
```

#### Production Pipeline

```bash
# Triggers on: main branch
# Deploys to: ECS Fargate (high availability)
# Auto-deploy: No
# Manual approval: Yes (security team)
```

---

## Deployment

### Manual Deployment (First Time Setup)

#### Step 1: Create ECS Task Definition

```json
{
  "family": "backend-service",
  "networkMode": "awsvpc",
  "requiresCompatibilities": ["FARGATE"],
  "cpu": "256",
  "memory": "512",
  "containerDefinitions": [
    {
      "name": "backend-container",
      "image": "<ACCOUNT_ID>.dkr.ecr.ap-south-1.amazonaws.com/backend-api-repo:latest",
      "portMappings": [
        {
          "containerPort": 8000,
          "protocol": "tcp"
        }
      ],
      "environment": [
        {
          "name": "ML_SERVICE_URL",
          "value": "http://ml-model-service:8000"
        },
        {
          "name": "ENVIRONMENT",
          "value": "dev"
        },
        {
          "name": "LOG_LEVEL",
          "value": "INFO"
        }
      ],
      "logConfiguration": {
        "logDriver": "awslogs",
        "options": {
          "awslogs-group": "/ecs/backend-service",
          "awslogs-region": "ap-south-1",
          "awslogs-stream-prefix": "ecs"
        }
      },
      "healthCheck": {
        "command": ["CMD-SHELL", "curl -f http://localhost:8000/health || exit 1"],
        "interval": 30,
        "timeout": 5,
        "retries": 2,
        "startPeriod": 10
      }
    }
  ],
  "executionRoleArn": "arn:aws:iam::<ACCOUNT_ID>:role/ecsTaskExecutionRole",
  "taskRoleArn": "arn:aws:iam::<ACCOUNT_ID>:role/ecsTaskRole"
}
```

```bash
# Register task definition
aws ecs register-task-definition --cli-input-json file://task-definition.json
```

#### Step 2: Create ECS Service

```bash
aws ecs create-service \
  --cluster prediction-app-cluster \
  --service-name backend-service \
  --task-definition backend-service:1 \
  --desired-count 2 \
  --launch-type FARGATE \
  --network-configuration "awsvpcConfiguration={subnets=[subnet-xxx,subnet-yyy],securityGroups=[sg-xxx],assignPublicIp=DISABLED}" \
  --load-balancers targetGroupArn=arn:aws:elasticloadbalancing:ap-south-1:ACCOUNT_ID:targetgroup/backend-tg/abc123,containerName=backend-container,containerPort=8000 \
  --deployment-configuration maximumPercent=200,minimumHealthyPercent=100
```

#### Step 3: Test Deployment

```bash
# Get ALB DNS
ALB_DNS=$(aws elbv2 describe-load-balancers \
  --names prediction-app-alb \
  --query 'LoadBalancers[0].DNSName' \
  --output text)

# Test health endpoint
curl http://$ALB_DNS/health

# Test predict endpoint
curl -X POST http://$ALB_DNS/api/predict \
  -H "Content-Type: application/json" \
  -d '{"features": [1.0, 2.0, 3.0]}'
```

### Automated Deployment (Via Pipeline)

Once CodePipeline is created, deployments are automatic:

```bash
# Push code to trigger pipeline
git push origin main

# Monitor pipeline progress
aws codepipeline get-pipeline-state --name ml-deployment-pipeline

# View CodeBuild logs
aws logs tail /aws/codebuild/backend-api-build --follow
```

### Rollback Strategy

```bash
# Rollback to previous task definition
aws ecs update-service \
  --cluster prediction-app-cluster \
  --service backend-service \
  --task-definition backend-service:2 \
  --force-new-deployment

# Verify rollback
aws ecs describe-services \
  --cluster prediction-app-cluster \
  --services backend-service \
  --query 'services[0].deployments'
```

---

## Monitoring & Logging

### CloudWatch Logs

#### View Logs

```bash
# Backend logs
aws logs tail /ecs/backend-service --follow

# ML service logs
aws logs tail /ecs/ml-model-service --follow

# CodeBuild logs
aws logs tail /aws/codebuild/backend-api-build --follow
```

#### Create Log Insights Queries

```bash
# Query: Error rate in last hour
fields @timestamp, @message, @logStream
| filter @message like /ERROR/
| stats count() as error_count by @logStream

# Query: Response time analysis
fields @duration
| stats pct(@duration, 50) as p50, pct(@duration, 95) as p95, pct(@duration, 99) as p99

# Query: Failed predictions
fields @timestamp, @message
| filter @message like /prediction failed/
| stats count() as failed_predictions
```

### CloudWatch Metrics & Alarms

```bash
# Create CPU utilization alarm
aws cloudwatch put-metric-alarm \
  --alarm-name backend-service-high-cpu \
  --alarm-description "Alert when CPU > 70%" \
  --metric-name CPUUtilization \
  --namespace AWS/ECS \
  --statistic Average \
  --period 300 \
  --threshold 70 \
  --comparison-operator GreaterThanThreshold \
  --dimensions Name=ServiceName,Value=backend-service Name=ClusterName,Value=prediction-app-cluster \
  --alarm-actions arn:aws:sns:ap-south-1:ACCOUNT_ID:alerts

# Create memory alarm
aws cloudwatch put-metric-alarm \
  --alarm-name backend-service-high-memory \
  --alarm-description "Alert when memory > 80%" \
  --metric-name MemoryUtilization \
  --namespace AWS/ECS \
  --statistic Average \
  --period 300 \
  --threshold 80 \
  --comparison-operator GreaterThanThreshold
```

### Dashboard Creation

```bash
# Create custom CloudWatch dashboard
cat > dashboard.json <<EOF
{
  "widgets": [
    {
      "type": "metric",
      "properties": {
        "metrics": [
          [ "AWS/ECS", "CPUUtilization", { "stat": "Average" } ],
          [ ".", "MemoryUtilization", { "stat": "Average" } ]
        ],
        "period": 300,
        "stat": "Average",
        "region": "ap-south-1",
        "title": "ECS Service Health"
      }
    },
    {
      "type": "log",
      "properties": {
        "query": "fields @duration | stats avg(@duration), max(@duration)",
        "region": "ap-south-1",
        "title": "API Response Times"
      }
    }
  ]
}
EOF

aws cloudwatch put-dashboard --dashboard-name ml-deployment-dashboard --dashboard-body file://dashboard.json
```

---

## Model Updates

### Pattern 1: Model in Docker Image (Recommended for PoC)

**Pros:**
- Simple deployment
- Version control friendly
- Easy rollback

**Cons:**
- Image size increases with each model
- Redeploy required for every model update

**Workflow:**

```bash
# 1. Data science team trains model
python train_model.py --data train.csv --output model.pkl

# 2. Commit model to repository
git add models/model.pkl
git commit -m "Update ML model to v1.2 (98.5% accuracy)"

# 3. Push to trigger pipeline
git push origin main

# 4. Pipeline automatically:
#    - Builds new Docker image (includes model.pkl)
#    - Runs tests
#    - Deploys to dev/staging
#    - Waits for manual approval for prod

# 5. After approval, prod deployment happens automatically
```

### Pattern 2: Model from S3 (Enterprise Pattern)

**Pros:**
- No image rebuild needed
- Flexible model management
- Faster deployments

**Cons:**
- Additional S3/IAM setup
- Need model versioning strategy

**Workflow:**

```bash
# 1. Train model and upload to S3
python train_model.py --data train.csv --output model.pkl
aws s3 cp model.pkl s3://ml-models-bucket/v1.2/model.pkl

# 2. Update model version in Parameter Store
aws ssm put-parameter \
  --name /ml-service/model-version \
  --value "v1.2" \
  --overwrite

# 3. (Optional) Restart ECS tasks to reload model
aws ecs update-service \
  --cluster prediction-app-cluster \
  --service ml-model-service \
  --force-new-deployment

# 4. ML service container:
#    - On startup: reads MODEL_VERSION from SSM
#    - Downloads corresponding model from S3
#    - Loads into memory
```

**ECS Task Definition (Pattern 2):**

```json
{
  "environment": [
    {
      "name": "S3_MODEL_BUCKET",
      "value": "ml-models-bucket"
    },
    {
      "name": "MODEL_VERSION",
      "value": "v1.2"
    }
  ]
}
```

**Python Code (Pattern 2):**

```python
import boto3
import os
import pickle

s3 = boto3.client('s3')
BUCKET = os.getenv('S3_MODEL_BUCKET')
VERSION = os.getenv('MODEL_VERSION', 'v1.0')
LOCAL_PATH = f'/tmp/model_{VERSION}.pkl'

# Download model
s3.download_file(BUCKET, f'{VERSION}/model.pkl', LOCAL_PATH)

with open(LOCAL_PATH, 'rb') as f:
    model = pickle.load(f)
```

### Model Versioning Best Practices

```bash
# S3 directory structure (Pattern 2)
s3://ml-models-bucket/
├── v1.0/
│   └── model.pkl
├── v1.1/
│   └── model.pkl
├── v1.2/
│   └── model.pkl (current in prod)
├── latest/
│   └── model.pkl (symlink to v1.2)
└── metadata.json
    {
      "current_version": "v1.2",
      "accuracy": 0.985,
      "trained_date": "2024-12-05",
      "training_data": "dataset-v3"
    }
```

---

## Troubleshooting

### Common Issues & Solutions

#### Issue 1: ECS Tasks Not Starting

```bash
# Check service status
aws ecs describe-services \
  --cluster prediction-app-cluster \
  --services backend-service

# Check task logs
aws ecs list-tasks --cluster prediction-app-cluster
aws ecs describe-tasks --cluster prediction-app-cluster --tasks <task-arn>

# View container logs
aws logs tail /ecs/backend-service --follow --since 1h
```

**Solutions:**
- Check IAM task role permissions
- Verify security group allows required ports
- Ensure ECR image exists and is accessible
- Check environment variables in task definition

#### Issue 2: Pipeline Build Fails

```bash
# View CodeBuild logs
aws codebuild batch-get-builds --ids <build-id>
aws logs tail /aws/codebuild/backend-api-build --follow
```

**Solutions:**
- Ensure buildspec.yml exists in repo root
- Verify Docker file syntax: `docker build --no-cache`
- Check IAM CodeBuild role has ECR push permissions
- Verify all test dependencies in requirements.txt

#### Issue 3: High Latency / Timeouts

```bash
# Check ALB target health
aws elbv2 describe-target-health --target-group-arn <tg-arn>

# Check ECS task performance
aws cloudwatch get-metric-statistics \
  --namespace AWS/ECS \
  --metric-name CPUUtilization \
  --start-time $(date -u -d '1 hour ago' +%Y-%m-%dT%H:%M:%S) \
  --end-time $(date -u +%Y-%m-%dT%H:%M:%S) \
  --period 300 \
  --statistics Average
```

**Solutions:**
- Increase ECS task CPU/memory
- Enable auto-scaling for ECS service
- Check ML model inference time
- Review CloudWatch Logs for errors

#### Issue 4: Model Service Can't Access S3

```bash
# Check IAM task role policy
aws iam get-role-policy --role-name ecsTaskRole --policy-name S3Access

# Test S3 access from container
docker run -it \
  -e AWS_ACCESS_KEY_ID=xxx \
  -e AWS_SECRET_ACCESS_KEY=xxx \
  ml-model-service:latest \
  bash -c "aws s3 ls s3://ml-models-bucket/"
```

**Solutions:**
- Add S3 read policy to ECS task role
- Verify S3 bucket name and model key
- Check if model file actually exists in S3

### Debug Commands Reference

```bash
# Get all running ECS tasks
aws ecs list-tasks --cluster prediction-app-cluster

# Describe specific task
aws ecs describe-tasks --cluster prediction-app-cluster --tasks <task-id>

# View CodePipeline execution history
aws codepipeline list-pipeline-executions --pipeline-name ml-deployment-pipeline

# Check recent deployments
aws ecs describe-services \
  --cluster prediction-app-cluster \
  --services backend-service \
  --query 'services[0].deployments'

# View ECR image tags
aws ecr list-images --repository-name backend-api-repo

# Get ALB DNS name
aws elbv2 describe-load-balancers --names prediction-app-alb \
  --query 'LoadBalancers[0].DNSName'
```

---

## Future Enhancements

### Phase 2: Production Hardening

- [ ] **Implement Blue/Green Deployments** – Zero-downtime updates via CodeDeploy
- [ ] **Add API Gateway** – Rate limiting, authentication, API versioning
- [ ] **Database Migration** – RDS setup with Multi-AZ for prod
- [ ] **VPC Peering** – Secure backend-to-backend communication
- [ ] **KMS Encryption** – Encrypt model files at rest in S3
- [ ] **AWS WAF** – Protect ALB from DDoS and common attacks

### Phase 3: Advanced ML Ops

- [ ] **Model Registry** – MLflow / SageMaker Model Registry
- [ ] **A/B Testing** – Canary deployments for new models
- [ ] **Model Monitoring** – Data drift detection, prediction drift
- [ ] **Automated Retraining** – Scheduled pipeline for model updates
- [ ] **Feature Store** – Centralized feature engineering (Feast, SageMaker)

### Phase 4: Mobile App Pipeline

- [ ] **GitHub Actions** – Build and test mobile app on commits
- [ ] **App Distribution** – Firebase App Distribution / App Center
- [ ] **Over-the-Air Updates** – Codepush or similar
- [ ] **Analytics** – Crash reporting, performance monitoring (Firebase)

### Phase 5: Advanced Observability

- [ ] **Distributed Tracing** – AWS X-Ray integration
- [ ] **Custom Metrics** – Application-level KPIs
- [ ] **APM Tools** – Datadog, New Relic integration
- [ ] **Log Aggregation** – ELK Stack or CloudWatch Insights
- [ ] **Alert Escalation** – PagerDuty / Opsgenie integration

### Phase 6: Cost Optimization

- [ ] **Spot Instances** – 70% cost savings for non-critical workloads
- [ ] **Reserved Capacity** – 1-year/3-year commitment discounts
- [ ] **Auto-scaling Policies** – Dynamic scaling based on metrics
- [ ] **Compute Savings Plans** – Flexible EC2 pricing
- [ ] **Cost Explorer Dashboards** – Track and optimize spending

---

## Team Contacts

### Key Stakeholders

| Role | Name | Email | Slack |
|------|------|-------|-------|
| DevOps Lead | - | - | @devops-lead |
| ML Engineer | - | - | @ml-engineer |
| Backend Lead | - | - | @backend-lead |
| Mobile Lead | - | - | @mobile-lead |
| AWS Account Owner | - | - | @aws-admin |

### Support & Escalation

| Issue | Slack Channel | On-Call |
|-------|---------------|---------|
| Pipeline Failures | #devops-incidents | @devops-oncall |
| Model Accuracy Issues | #ml-team | @ml-oncall |
| API Failures | #backend-support | @backend-oncall |
| Infrastructure | #aws-team | @infra-oncall |

---

## Quick Reference

### Useful Links & Documentation

- [AWS ECS Documentation](https://docs.aws.amazon.com/ecs/)
- [AWS CodePipeline User Guide](https://docs.aws.amazon.com/codepipeline/)
- [Docker Best Practices](https://docs.docker.com/develop/dev-best-practices/)
- [FastAPI Documentation](https://fastapi.tiangolo.com/)
- [Terraform AWS Provider](https://registry.terraform.io/providers/hashicorp/aws/latest)

### Environment Variables

```bash
# Common variables to set
export AWS_REGION=ap-south-1
export CLUSTER_NAME=prediction-app-cluster
export ECR_ACCOUNT_ID=<your-account-id>
export IMAGE_TAG=latest
export ALB_DNS=<your-alb-dns>
```

### Useful Commands

```bash
# Quick deploy
./scripts/deploy.sh backend prod

# View logs
./scripts/logs.sh backend

# Check status
./scripts/status.sh

# Rollback
./scripts/rollback.sh backend prod
```

---

## License

Proprietary - Company Internal Use Only

## Version History

| Version | Date | Author | Changes |
|---------|------|--------|---------|
| 1.0 | 2024-12-05 | DevOps Team | Initial PoC documentation |
| 1.1 | TBD | - | Production hardening additions |

---

**Last Updated:** December 5, 2024
**Status:** ✅ Production Ready (PoC Phase)
**Maintenance:** Reviewed quarterly
