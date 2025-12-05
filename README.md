# Hybrid Mobile App + ML Model AWS Deployment PoC

## Table of Contents

- [Overview](#overview)
- [Architecture](#architecture)
- [Technology Stack](#technology-stack)
- [Repository Structure](#repository-structure)
- [Components Overview](#components-overview)
- [System Flow](#system-flow)
- [Environments](#environments)
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
- ✅ Enterprise-grade ML Ops with SageMaker

---

## Technology Stack

| Component | Technology |
|-----------|-----------|
| **Container Orchestration** | AWS ECS (Fargate) |
| **CI/CD** | AWS CodePipeline + CodeBuild |
| **Container Registry** | Amazon ECR |
| **ML Model Management** | AWS SageMaker (Training, Registry, Endpoints) |
| **Load Balancing** | Application Load Balancer (ALB) |
| **Model Storage** | Amazon SageMaker Model Registry + S3 |
| **Model Deployment** | SageMaker Real-time Endpoints / Multi-Model Endpoints |
| **Database** | RDS/DynamoDB (optional) |
| **Monitoring** | CloudWatch Logs + Metrics + SageMaker Model Monitor |
| **Infrastructure as Code** | Terraform (recommended) |
| **Security** | IAM Roles, Secrets Manager, ACM, SageMaker Network Isolation |

---

## Architecture

### High-Level Diagram (Enhanced with SageMaker)

```
┌────────────────────────────────────────────────────────────────────┐
│                    MOBILE APP CLIENTS                              │
│              (React Native / Flutter / Ionic)                      │
└────────────────────────┬───────────────────────────────────────────┘
                         │
                   HTTPS (ACM SSL)
                         │
        ┌────────────────▼──────────────────┐
        │   Application Load Balancer       │
        │      (ALB - Public Subnets)       │
        └──┬──────────────────────┬─────────┘
           │                      │
    Path: /api/*          Path: /ml/*
           │                      │
    ┌──────▼──────┐        ┌──────▼─────────┐
    │   Backend   │        │  ML Service    │
    │   Service   │◄──────►│ (SageMaker     │
    │  (ECS Task) │        │ Endpoint)      │
    └──────┬──────┘        └──────┬─────────┘
           │                      │
           └──────┬───────────────┘
                  │
        ┌─────────▼──────────────────┐
        │  ECR + SageMaker Registry  │
        │  (Image & Model Artifacts) │
        └─────────┬──────────────────┘
                  │
    ┌─────────────┼──────────────┬──────────────┐
    │             │              │              │
┌───▼──┐   ┌─────▼──┐    ┌──────▼─┐   ┌───────▼────────┐
│ S3   │   │Secrets │    │Cloud   │   │ SageMaker      │
│Model │   │Manager │    │Watch   │   │ Model Monitor  │
│Store │   │        │    │Logs    │   │ (Data Drift)   │
└──────┘   └────────┘    └────────┘   └────────────────┘

┌────────────────────────────────────────────────────────────────────┐
│    SageMaker Training Pipeline                                     │
│  (Automated model retraining, evaluation, and versioning)          │
│                                                                    │
│ Data Prep → Training Job → Model Evaluation → Registry → Endpoint │
└────────────────────────────────────────────────────────────────────┘
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

**Key Endpoints:**
- `GET /health` - Health check endpoint
- `POST /api/predict` - Calls SageMaker endpoint for predictions

---

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

**Key Endpoints:**
- `GET /health` - Health check
- `POST /predict` - Model prediction endpoint

**Note:** This service can be replaced by SageMaker Real-time Endpoint in Phase 3

---

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

## Components Overview

### 1. Source Control

**GitHub / CodeCommit** - Version control for all repositories

Three main repositories:
- `mobile-app` - Hybrid mobile application (React Native/Flutter)
- `backend-api` - REST API service (Node.js/Python/Java)
- `ml-model-service` - ML prediction service (Python FastAPI/Flask)

---

### 2. CI/CD Pipeline

**AWS CodePipeline** - Orchestrates the entire deployment workflow

**AWS CodeBuild** - Builds Docker images, runs tests, pushes to ECR

Pipeline stages per service:
1. **Source** - Fetch code from GitHub on push
2. **Build** - Run tests, build Docker image, push to ECR
3. **Deploy (Dev)** - ECS rolling update to development
4. **Deploy (Prod)** - Manual approval + blue/green deployment

---

### 3. Container Registry & Hosting

**Amazon ECR** - Private Docker image repository

Two main repositories:
- `backend-api-repo` - Backend service images
- `ml-model-service-repo` - ML service images (pre-Phase 3)

**AWS ECS (Fargate)** - Serverless container orchestration

Two main services:
- `backend-service` - API handling
- `ml-model-service` - ML inference (migrates to SageMaker in Phase 3)

---

### 4. Networking

**AWS VPC** - Isolated network environment

Subnets:
- 2 Public Subnets (ALB)
- 2 Private Subnets (ECS tasks, databases)
- Multi-AZ deployment for high availability

**Application Load Balancer (ALB)** - Distributes traffic with path-based routing

Target groups:
- `/api/*` → Backend service
- `/ml/*` → ML service (pre-Phase 3) or SageMaker endpoint (Phase 3+)

---

### 5. Data & Storage

**Amazon S3** - Model artifact storage

Directory structure:
- `ml-models-bucket/` - Trained model files
- `training-data-bucket/` - Training datasets
- `monitoring-bucket/` - Model monitoring outputs

**SageMaker Model Registry** - Centralized model versioning and metadata

**RDS/DynamoDB** - Application database (optional for PoC)

**Secrets Manager** - Secure credential storage for API keys, DB passwords

---

### 6. ML Ops with SageMaker

**SageMaker Training Jobs** - Automated model training with hyperparameter tuning

**SageMaker Real-time Endpoints** - Low-latency inference for production predictions

**SageMaker Multi-Model Endpoints** - Host multiple model versions efficiently

**SageMaker Model Monitor** - Detect data/prediction drift in production

**SageMaker Pipelines** - Orchestrate end-to-end ML workflows

---

### 7. Monitoring & Observability

**CloudWatch Logs** - Centralized logging for all services

Log groups:
- `/ecs/backend-service` - Backend API logs
- `/ecs/ml-model-service` - ML service logs (pre-Phase 3)
- `/aws/codebuild/backend-api-build` - Build logs

**CloudWatch Metrics** - Performance monitoring and alarms

**CloudWatch Dashboard** - Real-time service health visualization

**SageMaker Model Monitor** - Track model quality metrics and data drift

---

## System Flow

### Normal Deployment Flow (Post-Setup)

```
1. Developer pushes code to main branch
                ↓
2. GitHub webhook triggers CodePipeline
                ↓
3. Source Stage: Fetch code from repository
                ↓
4. Build Stage: CodeBuild runs tests & builds Docker image
                ↓
5. Build Stage: Push image to ECR with version tag
                ↓
6. Deploy to Dev: ECS rolling update (automatic)
                ↓
7. Dev Validation: Team tests in development environment
                ↓
8. Deploy to Prod: Manual approval required
                ↓
9. Deploy to Prod: Blue/green deployment with ALB
                ↓
10. Monitoring: CloudWatch tracks health metrics
```

### ML Model Update Flow

**Pattern 1: Model in Docker Image (PoC)**
```
1. Data science team trains new model
                ↓
2. Commit model.pkl to ml-model-service repo
                ↓
3. Push to main triggers pipeline
                ↓
4. Build new Docker image with updated model
                ↓
5. Test and deploy to dev, then production
```

**Pattern 2: Model from S3 (Enterprise)**
```
1. Data science team trains model
                ↓
2. Upload model.pkl to S3
                ↓
3. Update model version in Parameter Store
                ↓
4. ECS tasks reload new model from S3
                ↓
5. No full redeploy needed
```

**Pattern 3: SageMaker Pipeline (Phase 3+)**
```
1. Automated training triggered by data update
                ↓
2. SageMaker Training Job runs with HPO
                ↓
3. Model evaluation metrics collected
                ↓
4. Model registered in SageMaker Registry
                ↓
5. Approval workflow for production
                ↓
6. Deploy to SageMaker Real-time Endpoint
                ↓
7. Model Monitor tracks data drift
```

---

## Environments

### Development (Dev)

- **Branch:** `develop`
- **ECS Configuration:** Minimal instances (1-2 tasks)
- **Deployment:** Automatic on code push
- **Approval:** Not required
- **Database:** Separate dev database
- **Monitoring:** Basic CloudWatch

### Staging (Stage)

- **Branch:** `stage` (optional)
- **ECS Configuration:** Medium instances (2-3 tasks)
- **Deployment:** Manual approval from dev
- **Data:** Production-like dataset
- **Monitoring:** Full monitoring enabled

### Production (Prod)

- **Branch:** `main`
- **ECS Configuration:** High availability (3+ tasks across AZs)
- **Deployment:** Blue/green with manual approval
- **Approval:** Required from security/tech lead
- **Database:** Multi-AZ RDS with backups
- **Monitoring:** Full monitoring + SageMaker Model Monitor
- **Alarms:** Critical alerts to on-call team

---

## Deployment Models

### Option A: ECS Fargate + Custom ML Service (Current PoC)

**Pros:**
- Full control over ML service code
- Can run custom preprocessing/post-processing logic
- Flexible scaling per service
- Cost-effective for small/medium workloads

**Cons:**
- Manual model versioning
- No built-in drift detection
- Requires custom monitoring

### Option B: SageMaker Endpoints (Phase 3+)

**Pros:**
- Managed ML endpoints (no infrastructure to manage)
- Built-in Model Registry
- Automatic drift detection
- A/B testing via Multi-Model Endpoints
- Better cost optimization with spot training

**Cons:**
- Less control over inference runtime
- Standard algorithms/frameworks only
- Higher learning curve

### Option C: Hybrid (Recommended Path)

**Phase 1-2:** ECS Fargate for both backend and ML service
- Quick to set up
- Full flexibility
- Easy iteration

**Phase 3+:** Migrate to SageMaker for ML service
- Leverage SageMaker's MLOps features
- Keep custom backend on ECS
- Better separation of concerns

---

## Model Versioning Strategy

### S3 Directory Structure

```
s3://ml-models-bucket/
├── v1.0/
│   ├── model.pkl
│   └── metadata.json
├── v1.1/
│   ├── model.pkl
│   └── metadata.json
├── v1.2/
│   ├── model.pkl
│   └── metadata.json (current in prod)
├── v1.3/
│   ├── model.pkl
│   └── metadata.json (in staging)
└── latest/
    └── model.pkl (symlink to v1.2)
```

### Metadata File Format

Each version includes:
- Training date
- Training dataset version
- Accuracy/performance metrics
- Hyperparameters used
- Author/data scientist
- Approval status

---

## Security Considerations

### Network Security

- VPC with private subnets for ECS tasks
- Security groups restrict traffic
- ALB in public subnets only
- No direct internet access to databases

### Data Security

- S3 encryption at rest (KMS)
- Model files access via IAM roles only
- Secrets Manager for sensitive credentials
- VPC endpoints for private S3 access

### Access Control

- IAM roles with least privilege
- CodePipeline service role limited to specific actions
- ECS task role can only access required S3 buckets
- SageMaker endpoint encryption

---

## Future Enhancements

### Phase 2: Production Hardening

- [ ] **Blue/Green Deployments** – Zero-downtime updates via CodeDeploy
- [ ] **API Gateway** – Rate limiting, authentication, API versioning
- [ ] **Database Setup** – RDS with Multi-AZ for production
- [ ] **VPC Peering** – Secure backend-to-backend communication
- [ ] **KMS Encryption** – Encrypt model files at rest in S3
- [ ] **AWS WAF** – Protect ALB from DDoS and common attacks

### Phase 3: Advanced ML Ops (SageMaker Focus)

- [ ] **SageMaker Model Registry** – Centralized model versioning ✨ **PRIORITY**
- [ ] **SageMaker Real-time Endpoints** – Replace ECS ML service ✨ **PRIORITY**
- [ ] **SageMaker Model Monitor** – Automated data/prediction drift detection ✨ **PRIORITY**
- [ ] **SageMaker Pipelines** – End-to-end ML workflow orchestration
- [ ] **A/B Testing** – Canary deployments for model versions
- [ ] **Automated Retraining** – Scheduled SageMaker Training Jobs
- [ ] **Feature Store** – MLflow / SageMaker Feature Store
- [ ] **Model Bias Detection** – SageMaker Clarify integration

### Phase 4: Mobile App Pipeline

- [ ] **GitHub Actions** – Build and test mobile app on commits
- [ ] **App Distribution** – Firebase App Distribution / App Center
- [ ] **Over-the-Air Updates** – Codepush or similar
- [ ] **Analytics** – Crash reporting, performance monitoring

### Phase 5: Advanced Observability

- [ ] **Distributed Tracing** – AWS X-Ray integration
- [ ] **Custom Metrics** – Application-level KPIs
- [ ] **APM Tools** – Datadog, New Relic integration
- [ ] **Log Aggregation** – CloudWatch Insights dashboard
- [ ] **Alert Escalation** – PagerDuty / Opsgenie integration

### Phase 6: Cost Optimization

- [ ] **Spot Instances** – 70% cost savings for non-critical workloads
- [ ] **Reserved Capacity** – 1-year/3-year commitment discounts
- [ ] **Auto-scaling Policies** – Dynamic scaling based on metrics
- [ ] **Compute Savings Plans** – Flexible EC2 pricing
- [ ] **Cost Explorer** – Track and optimize spending

---

## Quick Reference

### Key AWS Services Used

| Service | Purpose | Cost (Approx.) |
|---------|---------|----------------|
| **ECS Fargate** | Container orchestration | $0.015/hour/vCPU |
| **ALB** | Load balancing | $0.0225/hour |
| **ECR** | Image registry | $0.10 per GB stored |
| **S3** | Model storage | $0.023 per GB |
| **RDS** | Database | $0.17/hour (db.t3.micro) |
| **SageMaker Endpoint** | ML inference | $0.035/hour (ml.m5.large) |
| **CodePipeline** | CI/CD orchestration | $1.00 per active pipeline |
| **CodeBuild** | Build service | $0.005 per build minute |
| **CloudWatch** | Monitoring | $0.30 per custom metric |

**Estimated Monthly PoC Cost:** $200-400 depending on traffic and model size

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

## Version History

| Version | Date | Changes |
|---------|------|---------|
| 1.0 | 2024-12-05 | Initial PoC documentation |
| 2.0 | 2024-12-05 | Added comprehensive SageMaker integration |
| 2.1 | 2024-12-05 | Removed code examples, kept architecture |

---

**Last Updated:** December 5, 2024  
**Status:** ✅ Production Ready (PoC Phase)  
**Maintenance:** Reviewed quarterly  
**License:** Proprietary - Company Internal Use Only
