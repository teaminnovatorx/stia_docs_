# Deployment & DevOps

> UDARA AI Deployment Architecture — Edge, Cloud, and Dashboard layers with full CI/CD, infrastructure-as-code, and operational runbooks.

---

## Table of Contents

1. [Architecture Overview](#1-architecture-overview)
2. [CI/CD Pipeline — GitHub Actions](#2-cicd-pipeline--github-actions)
3. [Terraform — AWS Infrastructure](#3-terraform--aws-infrastructure)
4. [Balena Cloud — Edge Fleet](#4-balena-cloud--edge-fleet)
5. [Deployment Strategies](#5-deployment-strategies)
6. [Environment Management](#6-environment-management)
7. [Rollback Procedures](#7-rollback-procedures)
8. [Monitoring & Alerting](#8-monitoring--alerting)
9. [Disaster Recovery](#9-disaster-recovery)

---

## 1. Architecture Overview

UDARA AI operates across three deployment layers, each with its own deployment toolchain and strategy.

```
┌─────────────────────────────────────────────────────────────────┐
│                      UDARA DEPLOYMENT LAYERS                     │
├───────────────────┬───────────────────┬─────────────────────────┤
│   EDGE LAYER      │    CLOUD LAYER    │    DASHBOARD LAYER      │
│   Balena Cloud    │    AWS ECS        │    Vercel               │
├───────────────────┼───────────────────┼─────────────────────────┤
│ Agent-A (NER)     │ API Gateway       │ Next.js Frontend        │
│ Agent-B (Risk)    │ Orchestrator      │ MapLibre GL Maps        │
│ Agent-C (Therapy) │ PostgreSQL (RDS)  │ TanStack Charts         │
│ Whisper ASR       │ Qdrant (EC2)      │ shadcn/ui Components    │
│ Tesseract OCR     │ Redis (ElastiCache)│ Auth (Clerk)           │
│ WhisperSpeech TTS │ CloudFront CDN    │ Stripe Billing          │
│ SQLite (local)    │ S3 Storage        │                         │
│ Balena Supervisor │ CloudWatch        │                         │
└───────────────────┴───────────────────┴─────────────────────────┘
```

### Layer Communication Matrix

| From | To | Protocol | Port | Auth |
|------|-----|----------|------|------|
| Edge Device | Cloud API | HTTPS | 443 | mTLS + JWT |
| Edge Device | Cloud API (sync) | gRPC | 8443 | mTLS |
| Dashboard | Cloud API | HTTPS | 443 | JWT (Clerk) |
| Cloud API | Qdrant | gRPC | 6334 | Internal VPC |
| Cloud API | PostgreSQL | TCP | 5432 | IAM auth |
| Cloud API | Redis | TLS | 6379 | IAM auth |
| Dashboard | Vercel Edge | HTTPS | 443 | Vercel-managed |

---

## 2. CI/CD Pipeline — GitHub Actions

### 2.1 Pipeline Stages

The CI/CD pipeline is triggered on every push to `main` and `develop`, and on every pull request.

```
┌──────────┐    ┌──────────┐    ┌──────────┐    ┌──────────┐    ┌──────────┐
│  Ruff    │───▶│  Pytest  │───▶│  Docker  │───▶│   Push   │───▶│  Deploy  │
│  Lint    │    │  + Cov   │    │  Build   │    │  ECR/AKS │    │  Target  │
└──────────┘    └──────────┘    └──────────┘    └──────────┘    └──────────┘
   ~30s          ~2 min          ~4 min          ~2 min         ~3 min
```

### 2.2 Full GitHub Actions Workflow

```yaml
# .github/workflows/ci-cd.yml
name: UDARA CI/CD Pipeline

on:
  push:
    branches: [main, develop]
    paths:
      - 'services/**'
      - 'edge/**'
      - 'docker/**'
      - 'terraform/**'
  pull_request:
    branches: [main]
  schedule:
    - cron: '0 6 * * 1'  # Weekly security scan, Monday 6AM UTC

env:
  AWS_REGION: af-south-1
  ECR_REGISTRY: ${{ secrets.AWS_ACCOUNT_ID }}.dkr.ecr.af-south-1.amazonaws.com

jobs:
  # ──────────────────────────────────────────────
  # STAGE 1: Lint & Static Analysis
  # ──────────────────────────────────────────────
  lint:
    name: "Ruff Lint & Type Check"
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Set up Python 3.11
        uses: actions/setup-python@v5
        with:
          python-version: '3.11'
          cache: 'pip'

      - name: Install dependencies
        run: |
          pip install ruff mypy
          pip install -r services/requirements-dev.txt

      - name: Ruff lint
        run: |
          ruff check services/ edge/ \
            --config pyproject.toml \
            --output-format github

      - name: Ruff format check
        run: ruff format --check services/ edge/

      - name: MyPy type check
        run: |
          mypy services/ \
            --ignore-missing-imports \
            --no-strict-optional \
            --config-file pyproject.toml

  # ──────────────────────────────────────────────
  # STAGE 2: Test Suite
  # ──────────────────────────────────────────────
  test:
    name: "Pytest + Coverage"
    needs: lint
    runs-on: ubuntu-latest
    strategy:
      matrix:
        service: [agent-a, agent-b, agent-c, gateway, orchestrator]
    steps:
      - uses: actions/checkout@v4

      - name: Set up Python 3.11
        uses: actions/setup-python@v5
        with:
          python-version: '3.11'
          cache: 'pip'

      - name: Install service dependencies
        run: |
          pip install -r services/${{ matrix.service }}/requirements.txt
          pip install -r services/requirements-dev.txt

      - name: Run tests
        working-directory: services/${{ matrix.service }}
        env:
          DATABASE_URL: postgresql://test:test@localhost:5432/udara_test
          REDIS_URL: redis://localhost:6379/1
          QDRANT_URL: http://localhost:6333
        run: |
          pytest tests/ \
            -v \
            --cov=src \
            --cov-report=xml:coverage.xml \
            --cov-report=term-missing \
            --cov-fail-under=80 \
            --junitxml=test-results.xml

      - name: Upload coverage
        uses: actions/upload-artifact@v4
        with:
          name: coverage-${{ matrix.service }}
          path: services/${{ matrix.service }}/coverage.xml

  # ──────────────────────────────────────────────
  # STAGE 3: Docker Build (Multi-Arch)
  # ──────────────────────────────────────────────
  docker-build:
    name: "Docker Build (ARM64 + AMD64)"
    needs: test
    if: github.event_name == 'push'
    runs-on: ubuntu-latest
    strategy:
      matrix:
        image:
          - name: agent-a
            context: services/agent-a
            dockerfile: services/agent-a/Dockerfile
          - name: agent-b
            context: services/agent-b
            dockerfile: services/agent-b/Dockerfile
          - name: agent-c
            context: services/agent-c
            dockerfile: services/agent-c/Dockerfile
          - name: gateway
            context: services/gateway
            dockerfile: services/gateway/Dockerfile
          - name: orchestrator
            context: services/orchestrator
            dockerfile: services/orchestrator/Dockerfile
          - name: edge-bundle
            context: edge
            dockerfile: edge/Dockerfile
    steps:
      - uses: actions/checkout@v4

      - name: Configure AWS credentials
        uses: aws-actions/configure-aws-credentials@v4
        with:
          aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
          aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
          aws-region: ${{ env.AWS_REGION }}

      - name: Login to Amazon ECR
        id: ecr-login
        uses: aws-actions/amazon-ecr-login@v2

      - name: Set up Docker Buildx
        uses: docker/setup-buildx-action@v3

      - name: Generate image metadata
        id: meta
        uses: docker/metadata-action@v5
        with:
          images: ${{ env.ECR_REGISTRY }}/udara-${{ matrix.image.name }}
          tags: |
            type=sha,prefix=
            type=ref,event=branch
            type=semver,pattern={{version}}
            type=raw,value=latest,enable={{is_default_branch}}

      - name: Build and push multi-arch image
        uses: docker/build-push-action@v5
        with:
          context: ${{ matrix.image.context }}
          file: ${{ matrix.image.dockerfile }}
          platforms: linux/amd64,linux/arm64
          push: true
          tags: ${{ steps.meta.outputs.tags }}
          labels: ${{ steps.meta.outputs.labels }}
          cache-from: type=gha,scope=${{ matrix.image.name }}
          cache-to: type=gha,mode=max,scope=${{ matrix.image.name }}
          build-args: |
            BUILDKIT_INLINE_CACHE=1

  # ──────────────────────────────────────────────
  # STAGE 4: Deploy — Cloud (ECS)
  # ──────────────────────────────────────────────
  deploy-cloud:
    name: "Deploy to AWS ECS"
    needs: docker-build
    if: github.ref == 'refs/heads/main'
    runs-on: ubuntu-latest
    environment: production
    steps:
      - uses: actions/checkout@v4

      - name: Configure AWS credentials
        uses: aws-actions/configure-aws-credentials@v4
        with:
          aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
          aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
          aws-region: ${{ env.AWS_REGION }}

      - name: Render ECS task definitions
        run: |
          for svc in agent-a agent-b agent-c gateway orchestrator; do
            envsubst < terraform/services/${svc}-task-def.json.tpl \
              > terraform/services/${svc}-task-def.json
          done

      - name: Deploy ECS services (blue/green)
        run: |
          cd terraform
          terraform init -backend-config=backend.hcl
          terraform apply -auto-approve \
            -target=aws_ecs_service.agent_a \
            -target=aws_ecs_service.agent_b \
            -target=aws_ecs_service.agent_c \
            -target=aws_ecs_service.gateway \
            -target=aws_ecs_service.orchestrator

  # ──────────────────────────────────────────────
  # STAGE 5: Deploy — Edge (Balena)
  # ──────────────────────────────────────────────
  deploy-edge:
    name: "Deploy to Balena Fleet"
    needs: docker-build
    if: github.ref == 'refs/heads/main'
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Install Balena CLI
        run: |
          curl -fsSL https://github.com/balena-io/balena-cli/releases/download/v14.7.0/balena-cli-v14.7.0-linux-x64-standalone.zip \
            -o /tmp/balena.zip
          unzip -o /tmp/balena.zip -d /usr/local/bin

      - name: Balena login
        run: balena login --token ${{ secrets.BALENA_API_TOKEN }}

      - name: Push to Balena fleet
        run: |
          cd edge
          balena push udara-edge-fleet --nocache

  # ──────────────────────────────────────────────
  # STAGE 6: Deploy — Dashboard (Vercel)
  # ──────────────────────────────────────────────
  deploy-dashboard:
    name: "Deploy Dashboard to Vercel"
    needs: [test]
    if: github.ref == 'refs/heads/main'
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Deploy to Vercel (Production)
        uses: amondnet/vercel-action@v25
        with:
          vercel-token: ${{ secrets.VERCEL_TOKEN }}
          vercel-org-id: ${{ secrets.VERCEL_ORG_ID }}
          vercel-project-id: ${{ secrets.VERCEL_PROJECT_ID }}
          vercel-args: '--prod'
          working-directory: ./dashboard
```

### 2.3 Pipeline Timing Summary

| Stage | Duration (avg) | Runs On | Parallel |
|-------|---------------|---------|----------|
| Lint (Ruff + MyPy) | 30s | All events | No |
| Test (Pytest × 5) | 2–4 min | All events | Yes (5 services) |
| Docker Build × 6 | 3–6 min | Push only | Yes (6 images) |
| Deploy Cloud | 2–3 min | main push | No |
| Deploy Edge | 3–5 min | main push | No |
| Deploy Dashboard | 1–2 min | main push | No |

**Total pipeline time (main push): ~10–14 minutes**

---

## 3. Terraform — AWS Infrastructure

### 3.1 Project Structure

```
terraform/
├── backend.hcl
├── main.tf              # Provider, backend, locals
├── variables.tf         # All variable declarations
├── outputs.tf           # Outputs for other systems
├── versions.tf          # Provider version constraints
├── networking/
│   ├── vpc.tf           # VPC, subnets, IGW, NAT
│   ├── security-groups.tf
│   └── route-tables.tf
├── compute/
│   ├── ecs-cluster.tf   # Fargate cluster
│   ├── ecs-services.tf  # Service definitions
│   └── ecs-tasks.tf     # Task definitions
├── database/
│   ├── rds.tf           # PostgreSQL
│   └── elasticache.tf   # Redis
├── storage/
│   ├── s3.tf
│   └── cloudfront.tf
├── monitoring/
│   ├── cloudwatch.tf
│   └── alarms.tf
└── services/
    ├── agent-a-task-def.json.tpl
    ├── agent-b-task-def.json.tpl
    └── ...
```

### 3.2 Provider & Backend Configuration

```hcl
# terraform/versions.tf
terraform {
  required_version = ">= 1.7.0"

  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.40"
    }
    random = {
      source  = "hashicorp/random"
      version = "~> 3.6"
    }
  }

  backend "s3" {
    # Configured via backend.hcl at init
  }
}
```

```hcl
# terraform/backend.hcl
bucket         = "udara-terraform-state"
key            = "udara-infra/terraform.tfstate"
region         = "af-south-1"
encrypt        = true
dynamodb_table = "udara-terraform-locks"
```

### 3.3 VPC & Networking

```hcl
# terraform/networking/vpc.tf
resource "aws_vpc" "main" {
  cidr_block           = "10.0.0.0/16"
  enable_dns_hostnames = true
  enable_dns_support   = true

  tags = {
    Name        = "udara-main-vpc"
    Environment = var.environment
    Project     = "udara-ai"
  }
}

# Public subnets (for NAT Gateway, ALB)
resource "aws_subnet" "public" {
  count                   = length(var.availability_zones)
  vpc_id                  = aws_vpc.main.id
  cidr_block              = "10.0.${count.index + 1}.0/24"
  availability_zone       = var.availability_zones[count.index]
  map_public_ip_on_launch = true

  tags = {
    Name = "udara-public-${var.availability_zones[count.index]}"
    Tier = "Public"
  }
}

# Private subnets (for ECS Fargate tasks, RDS)
resource "aws_subnet" "private" {
  count             = length(var.availability_zones)
  vpc_id            = aws_vpc.main.id
  cidr_block        = "10.0.${count.index + 10}.0/24"
  availability_zone = var.availability_zones[count.index]

  tags = {
    Name = "udara-private-${var.availability_zones[count.index]}"
    Tier = "Private"
  }
}

# Internet Gateway
resource "aws_internet_gateway" "main" {
  vpc_id = aws_vpc.main.id
  tags   = { Name = "udara-igw" }
}

# NAT Gateway for private subnet egress
resource "aws_eip" "nat" {
  count  = var.high_availability ? length(var.availability_zones) : 1
  domain = "vpc"
}

resource "aws_nat_gateway" "main" {
  count         = length(aws_eip.nat)
  allocation_id = aws_eip.nat[count.index].id
  subnet_id     = aws_subnet.public[count.index].id

  depends_on = [aws_internet_gateway.main]

  tags = { Name = "udara-nat-${count.index}" }
}
```

### 3.4 Security Groups

```hcl
# terraform/networking/security-groups.tf

# ECS services security group
resource "aws_security_group" "ecs_services" {
  name        = "udara-ecs-services"
  description = "Security group for UDARA ECS services"
  vpc_id      = aws_vpc.main.id

  # Internal communication between agents
  ingress {
    from_port   = 8000
    to_port     = 8003
    protocol    = "tcp"
    self        = true
    description = "Inter-agent communication"
  }

  # Health checks from ALB
  ingress {
    from_port   = 8000
    to_port     = 8000
    protocol    = "tcp"
    security_groups = [aws_security_group.alb.id]
    description = "ALB health checks and traffic"
  }

  # Outbound internet
  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }
}

# RDS PostgreSQL security group
resource "aws_security_group" "rds" {
  name        = "udara-rds-postgres"
  description = "Security group for UDARA RDS PostgreSQL"
  vpc_id      = aws_vpc.main.id

  ingress {
    from_port       = 5432
    to_port         = 5432
    protocol        = "tcp"
    security_groups = [aws_security_group.ecs_services.id]
    description     = "PostgreSQL access from ECS"
  }
}

# Qdrant EC2 security group
resource "aws_security_group" "qdrant" {
  name        = "udara-qdrant"
  description = "Security group for Qdrant vector DB on EC2"
  vpc_id      = aws_vpc.main.id

  ingress {
    from_port   = 6333
    to_port     = 6334
    protocol    = "tcp"
    security_groups = [aws_security_group.ecs_services.id]
    description = "Qdrant HTTP and gRPC"
  }
}
```

### 3.5 ECS Fargate Cluster & Services

```hcl
# terraform/compute/ecs-cluster.tf
resource "aws_ecs_cluster" "main" {
  name = "udara-cluster"

  setting {
    name  = "containerInsights"
    value = "enabled"
  }

  tags = {
    Name = "udara-ecs-cluster"
  }
}

# Application Load Balancer
resource "aws_lb" "main" {
  name               = "udara-alb"
  internal           = false
  load_balancer_type = "application"
  security_groups    = [aws_security_group.alb.id]
  subnets            = aws_subnet.public[*].id

  tags = { Name = "udara-alb" }
}

resource "aws_lb_target_group" "gateway" {
  name     = "udara-gateway-tg"
  port     = 8000
  protocol = "HTTP"
  vpc_id   = aws_vpc.main.id
  target_type = "ip"

  health_check {
    path                = "/health"
    interval            = 30
    timeout             = 5
    healthy_threshold   = 3
    unhealthy_threshold = 2
  }
}

resource "aws_lb_listener" "https" {
  load_balancer_arn = aws_lb.main.arn
  port              = 443
  protocol          = "HTTPS"
  ssl_policy        = "ELBSecurityPolicy-2016-08"
  certificate_arn   = var.acm_certificate_arn

  default_action {
    type             = "forward"
    target_group_arn = aws_lb_target_group.gateway.arn
  }
}
```

```hcl
# terraform/compute/ecs-services.tf
# Gateway service (only one exposed via ALB)
resource "aws_ecs_service" "gateway" {
  name            = "udara-gateway"
  cluster         = aws_ecs_cluster.main.id
  task_definition = aws_ecs_task_definition.gateway.arn
  desired_count   = var.gateway_task_count
  launch_type     = "FARGATE"

  network_configuration {
    subnets          = aws_subnet.private[*].id
    security_groups  = [aws_security_group.ecs_services.id]
    assign_public_ip = false
  }

  load_balancer {
    target_group_arn = aws_lb_target_group.gateway.arn
    container_name   = "gateway"
    container_port   = 8000
  }

  deployment_circuit_breaker {
    enable   = true
    rollback = true
  }

  deployment_controller {
    type = "CODE_DEPLOY"
  }
}

# Agent services (internal only, no ALB)
resource "aws_ecs_service" "agent_a" {
  name            = "udara-agent-a"
  cluster         = aws_ecs_cluster.main.id
  task_definition = aws_ecs_task_definition.agent_a.arn
  desired_count   = var.agent_a_task_count
  launch_type     = "FARGATE"

  network_configuration {
    subnets          = aws_subnet.private[*].id
    security_groups  = [aws_security_group.ecs_services.id]
    assign_public_ip = false
  }

  deployment_circuit_breaker {
    enable   = true
    rollback = true
  }
}
```

### 3.6 RDS PostgreSQL

```hcl
# terraform/database/rds.tf
resource "aws_db_subnet_group" "main" {
  name       = "udara-db-subnet-group"
  subnet_ids = aws_subnet.private[*].id
}

resource "aws_db_instance" "postgres" {
  identifier     = "udara-postgres"
  engine         = "postgres"
  engine_version = "15.4"

  instance_class = var.db_instance_class  # db.t3.medium for dev, db.r6g.large for prod

  allocated_storage     = 100
  max_allocated_storage = 500
  storage_type          = "gp3"
  storage_encrypted     = true
  kms_key_id            = var.kms_key_arn

  db_name  = "udara"
  username = "udara_app"
  password = var.db_password

  vpc_security_group_ids = [aws_security_group.rds.id]
  db_subnet_group_name   = aws_db_subnet_group.main.name

  multi_az               = var.high_availability
  backup_retention_period = var.backup_retention_days
  backup_window          = "03:00-04:00"
  maintenance_window     = "Mon:04:00-Mon:05:00"

  deletion_protection = true

  performance_insights_enabled = var.high_availability

  tags = {
    Name        = "udara-rds-postgres"
    Environment = var.environment
  }
}
```

### 3.7 Qdrant EC2 Instance

```hcl
# terraform/compute/qdrant.tf
resource "aws_instance" "qdrant" {
  ami           = "ami-0abcdef1234567890"  # Ubuntu 22.04 ARM64
  instance_type = var.qdrant_instance_type  # r6g.large for prod
  subnet_id     = aws_subnet.private[0].id

  vpc_security_group_ids = [aws_security_group.qdrant.id]

  root_block_device {
    volume_type = "gp3"
    volume_size = 200
    encrypted   = true
    kms_key_id  = var.kms_key_arn
  }

  user_data = <<-EOT
    #!/bin/bash
    set -euo pipefail

    # Install Docker
    apt-get update && apt-get install -y docker.io
    systemctl start docker && systemctl enable docker

    # Install qdrant
    docker pull qdrant/qdrant:latest

    # Configure data persistence
    mkdir -p /data/qdrant/storage

    # Run Qdrant with config
    docker run -d \
      --name qdrant \
      --restart unless-stopped \
      -p 6333:6333 \
      -p 6334:6334 \
      -v /data/qdrant/storage:/qdrant/storage \
      -e QDRANT__SERVICE__GRPC_PORT=6334 \
      qdrant/qdrant:latest
  EOT

  tags = {
    Name = "udara-qdrant"
  }
}
```

### 3.8 CloudFront CDN

```hcl
# terraform/storage/cloudfront.tf
resource "aws_cloudfront_distribution" "api" {
  enabled             = true
  is_ipv6_enabled     = true
  comment             = "UDARA API CDN"
  price_class         = "PriceClass_100"  # Use only edge locations in NA, EU, and SA (cheapest with Africa)
  default_root_object = ""

  origin {
    domain_name = aws_lb.main.dns_name
    origin_id   = "udara-alb-origin"
    origin_protocol_policy = "https-only"

    custom_header {
      name  = "X-CloudFront-Secret"
      value = var.cloudfront_secret
    }
  }

  # Cache API GET responses (case data, digests)
  default_cache_behavior {
    allowed_methods  = ["GET", "HEAD", "OPTIONS"]
    cached_methods   = ["GET", "HEAD"]
    target_origin_id = "udara-alb-origin"

    forwarded_values {
      query_string = false
      cookies {
        forward = "none"
      }
      headers = ["Authorization"]
    }

    viewer_protocol_policy = "redirect-to-https"
    min_ttl                = 0
    default_ttl            = 300    # 5 minutes
    max_ttl                = 3600   # 1 hour
    compress               = true
  }

  restrictions {
    geo_restriction {
      restriction_type = "none"
    }
  }

  viewer_certificate {
    acm_certificate_arn      = var.cloudfront_cert_arn
    ssl_support_method       = "sni-only"
    minimum_protocol_version = "TLSv1.2_2021"
  }

  tags = { Name = "udara-cloudfront-api" }
}
```

### 3.9 CloudWatch Monitoring

```hcl
# terraform/monitoring/alarms.tf
resource "aws_cloudwatch_metric_alarm" "gateway_5xx" {
  alarm_name          = "udara-gateway-5xx-rate"
  comparison_operator = "GreaterThanThreshold"
  evaluation_periods  = 2
  metric_name         = "5XXError"
  namespace           = "AWS/ApplicationELB"
  period              = 60
  statistic           = "Sum"
  threshold           = 10
  alarm_description   = "Gateway 5XX error rate exceeds threshold"

  dimensions = {
    LoadBalancer = aws_lb.main.arn_suffix
    TargetGroup  = aws_lb_target_group.gateway.arn_suffix
  }

  alarm_actions = [aws_sns_topic.alerts.arn]
}

resource "aws_cloudwatch_metric_alarm" "ecs_cpu_high" {
  alarm_name          = "udara-ecs-cpu-high"
  comparison_operator = "GreaterThanThreshold"
  evaluation_periods  = 3
  metric_name         = "CPUUtilization"
  namespace           = "AWS/ECS"
  period              = 60
  statistic           = "Average"
  threshold           = 80

  dimensions = {
    ServiceName = aws_ecs_service.gateway.name
    ClusterName = aws_ecs_cluster.main.name
  }

  alarm_actions = [aws_sns_topic.alerts.arn]
}

resource "aws_cloudwatch_metric_alarm" "rds_storage" {
  alarm_name          = "udara-rds-storage-low"
  comparison_operator = "LessThanThreshold"
  evaluation_periods  = 2
  metric_name         = "FreeStorageSpace"
  namespace           = "AWS/RDS"
  period              = 300
  statistic           = "Average"
  threshold           = 5000000000  # 5 GB

  dimensions = {
    DBInstanceIdentifier = aws_db_instance.postgres.id
  }

  alarm_actions = [aws_sns_topic.alerts.arn]
}
```

### 3.10 Terraform Variables

```hcl
# terraform/variables.tf
variable "environment" {
  description = "Deployment environment"
  type        = string
  validation {
    condition     = contains(["development", "staging", "production"], var.environment)
    error_message = "Environment must be development, staging, or production."
  }
}

variable "availability_zones" {
  description = "AWS availability zones"
  type        = list(string)
  default     = ["af-south-1a", "af-south-1b"]
}

variable "high_availability" {
  description = "Enable multi-AZ deployment"
  type        = bool
  default     = false
}

variable "gateway_task_count" {
  description = "Number of gateway tasks"
  type        = number
  default     = 1
}

variable "agent_a_task_count" {
  description = "Number of agent-a tasks"
  type        = number
  default     = 1
}

variable "db_instance_class" {
  description = "RDS instance class"
  type        = string
  default     = "db.t3.medium"
}

variable "qdrant_instance_type" {
  description = "EC2 instance type for Qdrant"
  type        = string
  default     = "r6g.large"
}

variable "backup_retention_days" {
  description = "RDS backup retention"
  type        = number
  default     = 7
}
```

### 3.11 Cost Estimates by Environment

| Resource | Dev (monthly) | Staging (monthly) | Prod (monthly) |
|----------|--------------|-------------------|----------------|
| ECS Fargate (5 services) | $30 | $120 | $450 |
| RDS PostgreSQL | $15 | $60 | $300 |
| ElastiCache Redis | $12 | $35 | $150 |
| EC2 (Qdrant) | $15 | $80 | $350 |
| ALB | $0 | $22 | $22 |
| CloudFront | $0 | $5 | $50 |
| S3 Storage | $1 | $5 | $25 |
| NAT Gateway | $0 | $32 | $65 |
| CloudWatch | $0 | $10 | $50 |
| Data Transfer | $5 | $20 | $100 |
| **Total** | **$78** | **$389** | **$1,562** |

---

## 4. Balena Cloud — Edge Fleet

### 4.1 Fleet Configuration

```json
// balena/balena.json
{
  "name": "udara-edge-fleet",
  "type": "swarm",
  "tags": [
    { "key": "udara", "value": "true" }
  ],
  "configuration": {
    "RESIN_SUPERVISOR_POLL_INTERVAL": "600000",
    "RESIN_LOG_MANAGER_WAKE_CYCLE": "300000"
  }
}
```

### 4.2 Device Tagging Strategy

| Tag Key | Values | Purpose |
|---------|--------|---------|
| `state` | `lagos`, `abuja`, `kano`, `enugu`, ... | Geographic location |
| `site-type` | `phc`, `hospital`, `clinic`, `outreach` | Facility classification |
| `connectivity` | `always-on`, `intermittent`, `offline-first` | Network reliability tier |
| `priority` | `critical`, `standard` | Deployment priority |
| `hw-gen` | `rpi4-8gb`, `rpi5-8gb`, `jetson-nano`, `x86-64` | Hardware generation |
| `gpu` | `cuda`, `none` | GPU availability |
| `language-pack` | `en-sw`, `en-yo`, `en-ha`, `en-ig`, `full` | Preloaded language models |

### 4.3 Docker Compose for Edge

```yaml
# edge/docker-compose.yml
version: "3.8"

services:
  agent-a:
    image: ${ECR_REGISTRY}/udara-agent-a:latest
    privileged: true
    ports:
      - "8001:8001"
    volumes:
      - model-cache:/models
      - case-data:/data/cases
    environment:
      - AGENT_A_PORT=8001
      - MODEL_PATH=/models/xlm-roberta-ner
      - REDIS_URL=redis://redis:6379/0
      - LOG_LEVEL=INFO
      - EDGE_DEVICE_ID=${BALENA_DEVICE_UUID}
    restart: always
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:8001/health"]
      interval: 30s
      timeout: 5s
      retries: 3

  agent-b:
    image: ${ECR_REGISTRY}/udara-agent-b:latest
    ports:
      - "8002:8002"
    volumes:
      - model-cache:/models
      - case-data:/data/cases
    environment:
      - AGENT_B_PORT=8002
      - MODEL_PATH=/models/resistance-calculator
      - REDIS_URL=redis://redis:6379/0
      - QDRANT_URL=http://localhost:6333
      - LOG_LEVEL=INFO
    restart: always
    depends_on:
      - agent-a
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:8002/health"]
      interval: 30s
      timeout: 5s
      retries: 3

  agent-c:
    image: ${ECR_REGISTRY}/udara-agent-c:latest
    ports:
      - "8003:8003"
    volumes:
      - model-cache:/models
      - case-data:/data/cases
    environment:
      - AGENT_C_PORT=8003
      - MODEL_PATH=/models/therapy-recommender
      - REDIS_URL=redis://redis:6379/0
      - LOG_LEVEL=INFO
    restart: always
    depends_on:
      - agent-b
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:8003/health"]
      interval: 30s
      timeout: 5s
      retries: 3

  whisper-asr:
    image: ${ECR_REGISTRY}/udara-whisper-asr:latest
    ports:
      - "8004:8004"
    environment:
      - WHISPER_MODEL=medium
      - WHISPER_LANGUAGE=en
      - DEVICE=cpu
    volumes:
      - model-cache:/models
      - audio-data:/data/audio
    restart: always
    deploy:
      resources:
        limits:
          memory: 4G

  ocr-service:
    image: ${ECR_REGISTRY}/udara-ocr:latest
    ports:
      - "8005:8005"
    volumes:
      - image-data:/data/images
    restart: always

  sync-agent:
    image: ${ECR_REGISTRY}/udara-sync:latest
    environment:
      - CLOUD_API_URL=https://api.udara.health
      - SYNC_INTERVAL=300
      - OFFLINE_STORE=/data/offline
      - DEVICE_ID=${BALENA_DEVICE_UUID}
    volumes:
      - offline-store:/data/offline
      - case-data:/data/cases
    restart: always
    healthcheck:
      test: ["CMD", "python", "-c", "import requests; requests.get('http://localhost:8006/health')"]
      interval: 60s
      timeout: 10s
      retries: 3

  redis:
    image: redis:7-alpine
    ports:
      - "6379:6379"
    volumes:
      - redis-data:/data
    command: redis-server --maxmemory 256mb --maxmemory-policy allkeys-lru
    restart: always

volumes:
  model-cache:
  case-data:
  audio-data:
  image-data:
  offline-store:
  redis-data:
```

### 4.4 OTA Update & Rollback Commands

```bash
# Push new release to entire fleet
balena push udara-edge-fleet

# Push to specific tag group only
balena deploy udara-edge-fleet --tag state=lagos

# Rollback to previous release
balena fleet udara-edge-fleet restart --service agent-a --force
# Or pin to a specific release:
balena release pin <release-hash> --fleet udara-edge-fleet

# Monitor fleet-wide update progress
balena fleet udara-edge-fleet

# View device-level status
balena devices --fleet udara-edge-fleet --status running
balena devices --fleet udara-edge-fleet --status updating
balena devices --fleet udara-edge-fleet --status offline

# Force restart all devices in a state
balena ssh udara-edge-fleet --tag state=lagos --cmd "sudo systemctl restart agent-a"

# Enable/Disable auto-updates
balena config apply udara-edge-fleet --tag state=lagos \
  -e RESIN_SUPERVISOR_UPDATE_STRATEGY=manual
```

### 4.5 Device Health Dashboard (Balena)

```bash
# Get fleet-wide metrics
balena fleet udara-edge-fleet --json | python3 -c "
import json, sys
fleet = json.load(sys.stdin)
print(f'Total devices: {len(fleet)}')
print(f'Online: {sum(1 for d in fleet if d.get(\"is_online\"))}')
print(f'Current release: {fleet[0].get(\"is_on__commit\") if fleet else \"unknown\"}')
"

# Check device storage (important for model caching)
balena ssh <device-uuid> --cmd "df -h /data"
```

---

## 5. Deployment Strategies

### 5.1 Strategy Comparison

| Layer | Tool | Strategy | Downtime | Rollback Time |
|-------|------|----------|----------|---------------|
| Edge | Balena Cloud | Rolling update per device | 0 (per-device) | < 2 min |
| Cloud (API) | AWS ECS + CodeDeploy | Blue/Green | 0 | < 5 min |
| Cloud (DB) | AWS RDS | Native minor version upgrade | < 30s | Snapshot restore |
| Dashboard | Vercel | Instant rollback | 0 | < 1 min |
| Models | Balena + S3 | Model version pinning | 0 | < 2 min |

### 5.2 Blue/Green ECS Deployment Detail

```
┌──────────────┐                          ┌──────────────┐
│   BLUE (v1.2) │  ← Current production   │  GREEN (v1.3) │  ← New version
│               │                          │               │
│  agent-a:1.2  │     ┌──────────┐        │  agent-a:1.3  │
│  agent-b:1.2  │────▶│    ALB   │        │  agent-b:1.3  │
│  agent-c:1.2  │     │  Target  │        │  agent-c:1.3  │
│  gateway:1.2  │     │  Group   │        │  gateway:1.3  │
│               │     └────┬─────┘        │               │
└──────────────┘          │               └──────────────┘
                     100% traffic
                          │
              ┌───────────┼───────────┐
              │  Shift    │   Keep    │
              ▼           │           ▼
         ┌────────┐       │      ┌────────┐
         │  Test  │       │      │ Standby│
         │ Canary │       │      │  v1.2  │
         └────────┘       │      └────────┘
                          │
                   If health checks fail →
                   auto-rollback to v1.2
```

```bash
# Manual ECS blue/green via AWS CLI
# 1. Deploy new task definition
aws ecs update-service \
  --cluster udara-cluster \
  --service udara-gateway \
  --task-definition udara-gateway:23 \
  --deployment-configuration \
    deploymentCircuitBreaker={enable=true,rollback=true},\
    maximumPercent=200,minimumHealthyPercent=100

# 2. Monitor deployment
aws ecs describe-services \
  --cluster udara-cluster \
  --services udara-gateway \
  --query 'services[0].deployments[*].[status,runningCount,desiredCount,rolloutState]'

# 3. If manual rollback needed
aws ecs update-service \
  --cluster udara-cluster \
  --service udara-gateway \
  --task-definition udara-gateway:22  # Previous version
```

### 5.3 Dashboard Deployment (Vercel)

```bash
# Production deploy (auto via GitHub push to main)
# Manual deploy:
vercel --prod

# Rollback
vercel rollback

# Preview deployment (PR)
vercel --preview

# Environment-specific
vercel --env NEXT_PUBLIC_API_URL=https://staging-api.udara.health
```

---

## 6. Environment Management

### 6.1 Environment Matrix

| Variable | Development | Staging | Production |
|----------|------------|---------|------------|
| `ENVIRONMENT` | `development` | `staging` | `production` |
| `LOG_LEVEL` | `DEBUG` | `INFO` | `WARNING` |
| `DATABASE_URL` | `postgresql://localhost:5432/udara_dev` | `postgresql://staging-rds...` | `postgresql://prod-rds...` |
| `REDIS_URL` | `redis://localhost:6379/0` | `redis://staging-redis...` | `redis://prod-redis...` |
| `QDRANT_URL` | `http://localhost:6333` | `https://staging-qdrant...` | `https://prod-qdrant...` |
| `MODEL_QUANTIZATION` | `FP32` (full precision) | `INT8` | `INT4` (production) |
| `CACHE_TTL` | `0` (disabled) | `300` | `3600` |
| `RATE_LIMIT` | `1000/min` | `500/min` | `200/min` |
| `SYNC_INTERVAL` | `60s` | `300s` | `600s` |
| `HEALTH_CHECK_INTERVAL` | `10s` | `30s` | `30s` |

### 6.2 Environment File Templates

```bash
# environments/.env.development
ENVIRONMENT=development
LOG_LEVEL=DEBUG

# Database
DATABASE_URL=postgresql://udara:devpassword@localhost:5432/udara_dev
POSTGRES_MAX_CONNECTIONS=20

# Redis
REDIS_URL=redis://localhost:6379/0
REDIS_MAX_CONNECTIONS=10

# Qdrant
QDRANT_URL=http://localhost:6333
QDRANT_COLLECTION=udara_cases_dev

# API
GATEWAY_PORT=8000
AGENT_A_PORT=8001
AGENT_B_PORT=8002
AGENT_C_PORT=8003
CORS_ORIGINS=http://localhost:3000,http://localhost:5173

# Auth
JWT_SECRET=dev-jwt-secret-change-in-production
JWT_ALGORITHM=HS256
JWT_EXPIRY=86400

# Africa's Talking
AT_USERNAME=sandbox
AT_API_KEY=sandbox-key
AT_USSD_SHORTCODE=*384*12345#

# Telegram
TELEGRAM_BOT_TOKEN=dev-bot-token
TELEGRAM_TEST_MODE=true

# WhatsApp
WHATSAPP_PHONE_NUMBER_ID=dev-phone-id
WHATSAPP_TOKEN=dev-token
WHATSAPP_VERIFY_TOKEN=dev-verify-token

# External APIs
WHO_GLASS_API_URL=https://staging-api.who.int/glass
NCDC_API_URL=https://staging-api.ncdc.gov.ng

# Model Configuration
MODEL_QUANTIZATION=FP32
WHISPER_MODEL_SIZE=medium
NER_MODEL_PATH=./models/xlm-roberta-ner-finetuned

# Sync
SYNC_INTERVAL=60
OFFLINE_STORE_PATH=./data/offline

# Feature Flags
FF_BAYESIAN_RESISTANCE=true
FF_MULTILINGUAL=true
FF_GLASS_EXPORT=false
FF_ADVANCED_ANALYTICS=true
```

```bash
# environments/.env.production
ENVIRONMENT=production
LOG_LEVEL=WARNING

# Database
DATABASE_URL=postgresql://udara_app:${DB_PASSWORD}@udara-prod-rds.af-south-1.rds.amazonaws.com:5432/udara_prod
POSTGRES_MAX_CONNECTIONS=100
POSTGRES_SSL_MODE=require

# Redis
REDIS_URL=rediss://prod-redis.af-south-1.cache.amazonaws.com:6379/0
REDIS_MAX_CONNECTIONS=50
REDIS_SSL_MODE=require

# Qdrant
QDRANT_URL=https://udara-qdrant.internal:6333
QDRANT_COLLECTION=udara_cases
QDRANT_API_KEY=${QDRANT_API_KEY}

# API
GATEWAY_PORT=8000
AGENT_A_PORT=8001
AGENT_B_PORT=8002
AGENT_C_PORT=8003
CORS_ORIGINS=https://dashboard.udara.health,https://api.udara.health

# Auth
JWT_SECRET=${JWT_SECRET}
JWT_ALGORITHM=RS256
JWT_EXPIRY=3600

# Africa's Talking
AT_USERNAME=${AT_USERNAME}
AT_API_KEY=${AT_API_KEY}
AT_USSD_SHORTCODE=*384*UDARA#

# Telegram
TELEGRAM_BOT_TOKEN=${TELEGRAM_BOT_TOKEN}
TELEGRAM_TEST_MODE=false

# WhatsApp
WHATSAPP_PHONE_NUMBER_ID=${WHATSAPP_PHONE_NUMBER_ID}
WHATSAPP_TOKEN=${WHATSAPP_TOKEN}
WHATSAPP_VERIFY_TOKEN=${WHATSAPP_VERIFY_TOKEN}

# External APIs
WHO_GLASS_API_URL=https://api.who.int/glass
NCDC_API_URL=https://api.ncdc.gov.ng

# Model Configuration
MODEL_QUANTIZATION=INT4
WHISPER_MODEL_SIZE=medium
NER_MODEL_PATH=/models/xlm-roberta-ner-finetuned

# Sync
SYNC_INTERVAL=600
OFFLINE_STORE_PATH=/data/offline

# Feature Flags
FF_BAYESIAN_RESISTANCE=true
FF_MULTILINGUAL=true
FF_GLASS_EXPORT=true
FF_ADVANCED_ANALYTICS=true
```

### 6.3 Secret Management

```bash
# AWS Secrets Manager for production secrets
aws secretsmanager create-secret \
  --name udara/production/database \
  --secret-string '{"username":"udara_app","password":"$GENERATED_PASSWORD","host":"udara-prod-rds.af-south-1.rds.amazonaws.com"}'

# Rotate secrets (90-day policy)
aws secretsmanager rotate-secret \
  --secret-id udara/production/database \
  --rotation-lambda-arn arn:aws:lambda:af-south-1:ACCOUNT:function/udara-secret-rotator

# Grant ECS access
aws secretsmanager get-secret-value \
  --secret-id udara/production/database \
  --query SecretString --output text | jq -r '.password'
```

---

## 7. Rollback Procedures

### 7.1 Rollback Decision Matrix

| Signal | Severity | Auto-Rollback? | Manual Action |
|--------|----------|---------------|---------------|
| ECS deployment circuit breaker | Critical | Yes | Monitor alarm |
| Error rate > 5% for 5 min | High | Yes (after 5 min) | Check logs |
| Latency P99 > 10s | Medium | No | Scale up + investigate |
| Edge device offline > 1 hr | Medium | No | Reboot device |
| Model accuracy < 70% | High | Yes | Pin previous model |
| Database connection pool exhausted | Critical | No | Scale RDS |

### 7.2 Rollback Runbooks

#### Cloud API Rollback

```bash
#!/bin/bash
# scripts/rollback-cloud.sh
set -euo pipefail

SERVICE="${1:-gateway}"
PREVIOUS_VERSION="${2:-}"

if [ -z "$PREVIOUS_VERSION" ]; then
  echo "Fetching previous stable version..."
  PREVIOUS_VERSION=$(aws ecs list-task-definitions \
    --family-prefix "udara-${SERVICE}" \
    --sort DESC \
    --max-items 5 \
    --query 'taskDefinitionArns[1]' \
    --output text | xargs basename)
fi

echo "Rolling back $SERVICE to version $PREVIOUS_VERSION"

aws ecs update-service \
  --cluster udara-cluster \
  --service "udara-${SERVICE}" \
  --task-definition "udara-${SERVICE}:${PREVIOUS_VERSION}" \
  --force-new-deployment

echo "Waiting for rollback to complete..."
aws ecs wait services-stable \
  --cluster udara-cluster \
  --services "udara-${SERVICE}"

echo "Rollback complete. Current tasks:"
aws ecs describe-services \
  --cluster udara-cluster \
  --services "udara-${SERVICE}" \
  --query 'services[0].tasks[0].taskDefinitionArn'
```

#### Edge Fleet Rollback

```bash
#!/bin/bash
# scripts/rollback-edge.sh
set -euo pipefail

FLEET="udara-edge-fleet"
TARGET_RELEASE="${1:-}"

if [ -z "$TARGET_RELEASE" ]; then
  echo "Available releases:"
  balena releases $FLEET --json | python3 -c "
import json, sys
releases = json.load(sys.stdin)
for r in releases[:10]:
    print(f'{r[\"commit\"][:8]}  {r[\"createdAt\"]}  {r[\"status\"]}')
"
  echo "Usage: $0 <release-hash>"
  exit 1
fi

echo "Pinning fleet to release $TARGET_RELEASE..."
balena release pin $TARGET_RELEASE --fleet $FLEET

echo "Restarting all devices with pinned release..."
balena fleet $FLEET restart

echo "Rollback in progress. Monitor with: balena fleet $FLEET"
```

#### Dashboard Rollback

```bash
#!/bin/bash
# scripts/rollback-dashboard.sh
set -euo pipefail

echo "Current deployments:"
vercel ls --limit 5

echo ""
echo "Rolling back to previous deployment..."
vercel rollback

echo "Rollback complete."
```

#### Database Rollback

```bash
#!/bin/bash
# scripts/rollback-database.sh
set -euo pipefail

SNAPSHOT_ID="${1:-}"

if [ -z "$SNAPSHOT_ID" ]; then
  echo "Available RDS snapshots:"
  aws rds describe-db-snapshots \
    --db-instance-identifier udara-postgres \
    --query 'DBSnapshots[*].[DBSnapshotIdentifier,SnapshotCreateTime]' \
    --output table
  echo "Usage: $0 <snapshot-id>"
  exit 1
fi

echo "WARNING: This will replace the production database!"
echo "Snapshot: $SNAPSHOT_ID"
read -p "Are you sure? (type 'PRODUCTION ROLLBACK'): " confirm

if [ "$confirm" != "PRODUCTION ROLLBACK" ]; then
  echo "Cancelled."
  exit 1
fi

echo "Restoring database from snapshot..."
aws rds restore-db-instance-from-db-snapshot \
  --db-instance-identifier udara-postgres-restored \
  --db-snapshot-identifier "$SNAPSHOT_ID" \
  --db-instance-class db.r6g.large \
  --availability-zone af-south-1a \
  --no-multi-az

echo "Waiting for restore..."
aws rds wait db-instance-available \
  --db-instance-identifier udara-postgres-restored

echo "Restore complete. Manual switchover required:"
echo "1. Update DNS/security groups to point to udara-postgres-restored"
echo "2. Verify application connectivity"
echo "3. Rename instances: udara-postgres → udara-postgres-old, restored → udara-postgres"
```

### 7.3 Rollback Communication Template

```
Subject: [UDARA] Production Rollback — {service} — {timestamp}

Status: ROLLING BACK / ROLLED BACK / INVESTIGATING
Service: {agent-a | agent-b | agent-c | gateway | all}
Previous Version: {v1.2.3}
Rollback To: {v1.2.2}
Trigger: {circuit-breaker | error-rate | manual}
Impact: {description of user impact}
ETA to Resolution: {estimate}
Incident Lead: {name}
```

---

## 8. Monitoring & Alerting

### 8.1 Observability Stack

| Layer | Tool | What It Monitors |
|-------|------|-----------------|
| Edge | Balena Device Metrics | CPU, memory, disk, temperature, offline status |
| Cloud | AWS CloudWatch | ECS tasks, ALB metrics, RDS, Lambda |
| Application | Python structlog + Sentry | Error tracking, performance traces |
| Logs | Balena logs + CloudWatch Logs | Centralized log aggregation |
| Alerts | PagerDuty (prod), Slack (staging) | On-call routing |

### 8.2 Key Metrics & Thresholds

| Metric | Warning | Critical | Action |
|--------|---------|----------|--------|
| API Error Rate (5XX) | > 1% | > 5% | Auto-scale + alert |
| API Latency P99 | > 3s | > 10s | Investigate + scale |
| Edge Inference Time | > 3s | > 5s | Check model, reduce batch |
| Sync Success Rate | < 98% | < 95% | Check connectivity |
| RDS CPU | > 70% | > 90% | Scale up instance |
| RDS Free Storage | < 10 GB | < 5 GB | Expand storage |
| Redis Memory | > 80% | > 95% | Evict keys + scale |
| Edge Device Offline | > 30 min | > 2 hr | Alert + auto-reboot |
| Model Cache Hit Rate | < 80% | < 60% | Check cache config |
| Active Cases/Hour | > 50 | > 100 | Scale horizontally |

### 8.3 Health Check Endpoints

```bash
# Individual service health
curl -s https://api.udara.health/health | jq .
# Response:
# {
#   "status": "healthy",
#   "version": "1.3.0",
#   "uptime_seconds": 86400,
#   "services": {
#     "agent_a": { "status": "healthy", "model_loaded": true },
#     "agent_b": { "status": "healthy", "model_loaded": true },
#     "agent_c": { "status": "healthy", "model_loaded": true },
#     "database": { "status": "healthy", "latency_ms": 5 },
#     "redis": { "status": "healthy", "memory_used_mb": 128 },
#     "qdrant": { "status": "healthy", "vector_count": 15000 }
#   }
# }

# Deep health check (includes model warm-up status)
curl -s https://api.udara.health/health/deep | jq .

# Readiness check (for Kubernetes-style probes)
curl -s -o /dev/null -w "%{http_code}" https://api.udara.health/ready
# 200 = ready, 503 = not ready
```

---

## 9. Disaster Recovery

### 9.1 RPO/RTO Targets

| Layer | RPO (Recovery Point) | RTO (Recovery Time) |
|-------|---------------------|---------------------|
| Cloud Database | 5 minutes | 30 minutes |
| Edge Local Data | 0 (always current) | 0 (always available) |
| Cloud API | N/A (stateless) | 5 minutes |
| Dashboard | N/A (static) | 2 minutes |
| Vector DB (Qdrant) | 1 hour | 1 hour |

### 9.2 Backup Strategy

```bash
# Automated RDS backups (daily + point-in-time)
# Configured via Terraform - see rds.tf above
# Manual snapshot before major changes:
aws rds create-db-snapshot \
  --db-instance-identifier udara-postgres \
  --db-snapshot-identifier "udara-pre-deploy-$(date +%Y%m%d-%H%M%S)"

# Qdrant backup (weekly)
ssh udara-qdrant "docker exec qdrant curl -X POST 'http://localhost:6333/snapshots'"

# S3 versioning for case data exports
aws s3api put-bucket-versioning \
  --bucket udara-case-exports \
  --versioning-configuration Status=Enabled

# Edge SQLite backup (automated via sync agent)
# Local → Cloud sync acts as continuous backup
# Additional daily dump:
balena ssh <device-uuid> --cmd "sqlite3 /data/cases/cases.db '.dump'" \
  | gzip > "edge-backup-$(date +%Y%m%d).sql.gz"
```

### 9.3 DR Runbook Checklist

```markdown
## Disaster Recovery Checklist

### Phase 1: Assessment (T+0 to T+5 min)
- [ ] Identify the scope of the incident
- [ ] Determine which layer is affected (Edge / Cloud / Dashboard)
- [ ] Notify incident response team via PagerDuty
- [ ] Set Slack channel topic to incident status

### Phase 2: Containment (T+5 to T+15 min)
- [ ] If cloud API: trigger ECS circuit breaker
- [ ] If edge devices: devices continue offline operation
- [ ] If database: switch to read-only mode
- [ ] Communicate status to stakeholders

### Phase 3: Recovery (T+15 to T+30 min)
- [ ] Restore from latest healthy snapshot
- [ ] Deploy previous stable version
- [ ] Verify health checks pass
- [ ] Run smoke tests

### Phase 4: Validation (T+30 to T+45 min)
- [ ] Monitor error rates return to normal
- [ ] Verify data sync is flowing
- [ ] Check edge devices reconnecting
- [ ] Confirm dashboard data is accurate

### Phase 5: Post-Incident (T+1 hr to T+48 hr)
- [ ] Write incident post-mortem
- [ ] Update runbook with lessons learned
- [ ] Schedule follow-up actions
- [ ] Brief leadership on impact and resolution
```

---

## Appendix A: Useful Commands Cheatsheet

```bash
# Terraform
terraform plan -var-file=environments/production.tfvars
terraform apply -var-file=environments/production.tfvars -auto-approve
terraform destroy -var-file=environments/production.tfvars

# Balena
balena login --token $BALENA_TOKEN
balena push udara-edge-fleet
balena devices --fleet udara-edge-fleet
balena logs udara-edge-fleet --service agent-a --tail

# AWS
aws ecs list-tasks --cluster udara-cluster
aws ecs describe-task-definition --task-definition udara-gateway:23
aws logs tail /ecs/udara-gateway --since 1h --follow

# Docker (local development)
docker compose -f docker-compose.dev.yml up -d
docker compose -f docker-compose.dev.yml logs -f agent-a
docker compose -f docker-compose.dev.yml exec postgres psql -U udara -d udara_dev

# Vercel
vercel env ls
vercel env pull .env.production.local
vercel build && vercel deploy --prod
```

---

*Last updated: 2025-01-15 | Maintainer: DevOps Team*
