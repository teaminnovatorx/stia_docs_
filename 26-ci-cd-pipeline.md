# CI/CD Pipeline — GitHub Actions, Balena Fleet & Terraform

> "Ship fast, ship safe, manage everything from one dashboard."
> — UDARA AI DevOps Playbook

---

## Table of Contents

1. [Pipeline Architecture Overview](#pipeline-architecture-overview)
2. [Branching Strategy](#branching-strategy)
3. [GitHub Actions: CI Pipeline](#github-actions-ci-pipeline)
4. [GitHub Actions: CD Cloud (AWS)](#github-actions-cd-cloud-aws)
5. [GitHub Actions: CD Edge (Balena Fleet)](#github-actions-cd-edge-balena-fleet)
6. [GitHub Actions: CD Frontend (Vercel)](#github-actions-cd-frontend-vercel)
7. [GitHub Actions: Nightly Pipeline](#github-actions-nightly-pipeline)
8. [Balena Fleet Management](#balena-fleet-management)
9. [Terraform: AWS Infrastructure](#terraform-aws-infrastructure)
10. [Vercel: Frontend Deployment](#vercel-frontend-deployment)
11. [Release Process](#release-process)
12. [Testing Pyramid](#testing-pyramid)
13. [Secret Management](#secret-management)
14. [Cost Estimates](#cost-estimates)

---

## Pipeline Architecture Overview

UDARA AI uses a **three-track CI/CD pipeline** that builds, tests, and deploys three distinct targets: the cloud backend on AWS, the edge fleet on Balena Cloud (Raspberry Pi 5 devices), and the frontend on Vercel.

```
                    GitHub Repository (main branch)
                               │
                    ┌──────────▼──────────┐
                    │   CI Pipeline       │
                    │  (on every PR)       │
                    │  lint, test, scan    │
                    └──────────┬──────────┘
                               │
                    ┌──────────▼──────────┐
                    │   CD Trigger        │
                    │  (on main merge)    │
                    └──┬───────┬───────┬──┘
                       │       │       │
          ┌────────────▼┐  ┌──▼────┐  ┌▼──────────┐
          │ TRACK 1     │  │TRACK 2│  │ TRACK 3   │
          │ Cloud AWS   │  │Edge   │  │ Frontend  │
          │ ECS Fargate │  │Balena │  │ Vercel    │
          └──────┬──────┘  │Fleet  │  └─────┬─────┘
                 │         └──┬───┘        │
          ┌──────▼──────┐     │      ┌────▼─────┐
          │  Docker →    │     │      │  npm →   │
          │  ECR → ECS   │     │      │  Vercel  │
          │  Smoke Test  │     │      │  Lighthouse│
          └─────────────┘     │      └──────────┘
                         ┌────▼────┐
                         │  Docker │
                         │  arm64  │
                         │  Balena │
                         │  Push   │
                         │  Rolling│
                         │  Update │
                         └─────────┘
```

### Three Tracks

| Track | Target | Trigger | Tools | Region |
|-------|--------|---------|-------|--------|
| **Track 1** | Cloud Backend API | Main merge | GitHub Actions → Docker → ECR → ECS | AWS af-south-1 |
| **Track 2** | Edge Fleet (RPi 5) | Main merge | GitHub Actions → docker buildx → Balena | Balena Cloud |
| **Track 3** | Web Dashboard | Main merge | GitHub Actions → npm build → Vercel | Vercel (global CDN) |

---

## Branching Strategy

```
main (production)
  │
  ├── develop (integration)
  │     │
  │     ├── feature/add-voice-asr
  │     ├── feature/bayesian-engine
  │     └── feature/ussd-flows
  │
  ├── hotfix/fix-sync-crash
  │
  └── release/v1.0.0
```

| Branch | Purpose | Protection | Deploy Target |
|--------|---------|------------|----------------|
| `main` | Production-ready code | Require 2 approvals, CI pass, no force push | Track 1+2+3 (all production) |
| `develop` | Integration branch | Require 1 approval, CI pass | Track 1 staging only |
| `feature/*` | Individual features | No protection | PR preview (Track 3 Vercel preview) |
| `hotfix/*` | Emergency fixes | Require 1 approval, CI pass | Direct to main + develop |
| `release/*` | Release candidates | Require 2 approvals, CI pass + E2E tests | Pre-prod staging |

### Git Workflow

```bash
# Start a feature
git checkout develop
git pull origin develop
git checkout -b feature/add-voice-asr

# Work, commit with conventional commits
git commit -m "feat(asr): add MMS-ASR pipeline for Swahili"
git commit -m "feat(asr): add language detection via fastText"
git commit -m "test(asr): add integration tests for voice pipeline"

# Push and create PR to develop
git push origin feature/add-voice-asr
# → GitHub Actions runs CI (lint, test, scan)
# → Vercel creates preview deployment
# → PR requires 1 approval

# After approval, merge to develop
# → Develop branch CI runs (full suite)
# → Deployed to staging environment

# Ready for release
git checkout develop
git pull
git checkout -b release/v0.5.0
# → Run full E2E tests, performance tests
# → If all green, merge to main
git checkout main
git merge release/v0.5.0
# → semantic-release generates version, CHANGELOG
# → Track 1+2+3 deploy to production
```

---

## GitHub Actions: CI Pipeline

Runs on every pull request. Provides fast feedback to developers.

```yaml
# .github/workflows/ci.yml
name: CI Pipeline

on:
  pull_request:
    branches: [main, develop]
  push:
    branches: [develop]

concurrency:
  group: ci-${{ github.ref }}
  cancel-in-progress: true

jobs:
  lint-and-typecheck:
    name: Lint & Type Check
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      
      - name: Set up Python
        uses: actions/setup-python@v5
        with:
          python-version: '3.11'
          cache: 'pip'
      
      - name: Install dependencies
        run: |
          pip install -r requirements.txt
          pip install -r requirements-dev.txt
      
      - name: Ruff lint (Python)
        run: ruff check backend/ ml/ --config pyproject.toml
      
      - name: Ruff format check
        run: ruff format --check backend/ ml/
      
      - name: MyPy type check
        run: mypy backend/ --ignore-missing-imports --no-strict-optional
      
      - name: ESLint (Frontend)
        uses: actions/setup-node@v4
        with:
          node-version: '20'
          cache: 'npm'
          cache-dependency-path: frontend/package-lock.json
      - run: |
          cd frontend
          npm ci
          npm run lint
          npx tsc --noEmit

  test:
    name: Unit & Integration Tests
    runs-on: ubuntu-latest
    needs: lint-and-typecheck
    services:
      postgres:
        image: pgvector/pgvector:pg16
        env:
          POSTGRES_USER: test
          POSTGRES_PASSWORD: test
          POSTGRES_DB: udara_test
        ports:
          - 5432:5432
        options: >-
          --health-cmd pg_isready
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5
      redis:
        image: redis:7-alpine
        ports:
          - 6379:6379
    steps:
      - uses: actions/checkout@v4
      
      - name: Set up Python
        uses: actions/setup-python@v5
        with:
          python-version: '3.11'
          cache: 'pip'
      
      - name: Install dependencies
        run: |
          pip install -r requirements.txt
          pip install -r requirements-dev.txt
      
      - name: Run pytest with coverage
        env:
          DATABASE_URL: postgresql://test:test@localhost:5432/udara_test
          REDIS_URL: redis://localhost:6379/0
          ENVIRONMENT: test
        run: |
          pytest backend/ ml/ \
            --cov=backend \
            --cov=ml \
            --cov-report=xml:coverage.xml \
            --cov-report=term-missing \
            --junitxml=test-results.xml \
            -v --tb=short
          
      - name: Upload coverage to Codecov
        uses: codecov/codecov-action@v4
        with:
          files: coverage.xml
          fail_ci_if_error: false
          token: ${{ secrets.CODECOV_TOKEN }}
      
      - name: Upload test results
        uses: actions/upload-artifact@v4
        if: always()
        with:
          name: test-results
          path: test-results.xml

  security-scan:
    name: Security Scan
    runs-on: ubuntu-latest
    needs: lint-and-typecheck
    steps:
      - uses: actions/checkout@v4
      
      - name: Set up Python
        uses: actions/setup-python@v5
        with:
          python-version: '3.11'
      
      - name: Install Bandit
        run: pip install bandit[toml]
      
      - name: Bandit security scan
        run: |
          bandit -r backend/ ml/ \
            -f json \
            -o bandit-report.json \
            -ll \
            --config pyproject.toml
      
      - name: Check Bandit results
        run: |
          python -c "
          import json
          with open('bandit-report.json') as f:
              data = json.load(f)
          high = [r for r in data['results'] if r['issue_severity'] == 'HIGH']
          if high:
              print(f'HIGH severity issues found: {len(high)}')
              for r in high:
                  print(f'  - {r[\"issue_text\"]} ({r[\"filename\"]}:{r[\"line_number\"]})')
              exit(1)
          print(f'No HIGH severity issues. {len(data[\"results\"])} total findings.')
          "
      
      - name: Trivy container scan
        uses: aquasecurity/trivy-action@master
        with:
          scan-type: 'fs'
          scan-ref: 'backend/'
          severity: 'HIGH,CRITICAL'
          exit-code: '1'

  docker-build-test:
    name: Docker Build (Test)
    runs-on: ubuntu-latest
    needs: [test, security-scan]
    steps:
      - uses: actions/checkout@v4
      
      - name: Set up Docker Buildx
        uses: docker/setup-buildx-action@v3
      
      - name: Build cloud API image
        run: |
          docker build \
            -f backend/Dockerfile \
            -t udara-cloud-api:test \
            --target production \
            .
      
      - name: Smoke test container
        run: |
          docker run -d --name udara-test \
            -e DATABASE_URL=sqlite:///test.db \
            -e ENVIRONMENT=test \
            -p 8000:8000 \
            udara-cloud-api:test
          sleep 5
          curl -f http://localhost:8000/health || exit 1
          docker stop udara-test
```

---

## GitHub Actions: CD Cloud (AWS)

Deploys the cloud backend to AWS ECS Fargate in af-south-1 (Cape Town).

```yaml
# .github/workflows/cd-cloud.yml
name: Deploy Cloud Backend

on:
  push:
    branches: [main]
    paths:
      - 'backend/**'
      - 'ml/**'
      - 'requirements.txt'
      - 'Dockerfile'

concurrency:
  group: cd-cloud
  cancel-in-progress: false

jobs:
  build-and-push:
    name: Build & Push to ECR
    runs-on: ubuntu-latest
    outputs:
      image-tag: ${{ steps.meta.outputs.tags }}
    steps:
      - uses: actions/checkout@v4
      
      - name: Configure AWS credentials
        uses: aws-actions/configure-aws-credentials@v4
        with:
          aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
          aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
          aws-region: af-south-1
      
      - name: Login to ECR
        uses: aws-actions/amazon-ecr-login@v2
      
      - name: Set up Docker Buildx
        uses: docker/setup-buildx-action@v3
      
      - name: Docker meta (tags)
        id: meta
        uses: docker/metadata-action@v5
        with:
          images: ${{ secrets.ECR_REGISTRY }}/udara-cloud-api
          tags: |
            type=sha,prefix=
            type=ref,event=branch
            type=semver,pattern={{version}}
      
      - name: Build and push
        uses: docker/build-push-action@v5
        with:
          context: .
          file: backend/Dockerfile
          target: production
          push: true
          tags: ${{ steps.meta.outputs.tags }}
          cache-from: type=gha
          cache-to: type=gha,mode=max
          build-args: |
            BUILD_DATE=${{ github.event.head_commit.timestamp }}
            VCS_REF=${{ github.sha }}

  deploy:
    name: Deploy to ECS
    runs-on: ubuntu-latest
    needs: build-and-push
    environment: production
    steps:
      - name: Configure AWS credentials
        uses: aws-actions/configure-aws-credentials@v4
        with:
          aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
          aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
          aws-region: af-south-1
      
      - name: Update ECS service
        run: |
          aws ecs update-service \
            --cluster udara-cluster \
            --service udara-api-service \
            --force-new-deployment \
            --region af-south-1
      
      - name: Wait for deployment
        run: |
          aws ecs wait services-stable \
            --cluster udara-cluster \
            --services udara-api-service \
            --region af-south-1
      
      - name: Smoke test
        run: |
          API_URL="https://api.udara.ai"
          # Health check
          curl -f $API_URL/health || exit 1
          # API version check
          curl -f $API_URL/api/v1/version || exit 1
          echo "Cloud deployment smoke test passed"
```

---

## GitHub Actions: CD Edge (Balena Fleet)

Builds multi-arch Docker images and deploys to the Balena fleet of RPi 5 devices.

```yaml
# .github/workflows/cd-edge.yml
name: Deploy Edge Fleet

on:
  push:
    branches: [main]
    paths:
      - 'backend/**'
      - 'ml/**'
      - 'edge/**'
      - 'requirements.txt'
      - 'Dockerfile.edge'

concurrency:
  group: cd-edge
  cancel-in-progress: false

jobs:
  build-multiarch:
    name: Build Multi-Arch Image
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      
      - name: Set up QEMU (for arm64 emulation)
        uses: docker/setup-qemu-action@v3
        with:
          platforms: linux/arm64
      
      - name: Set up Docker Buildx
        uses: docker/setup-buildx-action@v3
      
      - name: Login to Balena Registry
        run: |
          echo ${{ secrets.BALENA_API_KEY }} | balena login --token-stdin
      
      - name: Build for arm64
        run: |
          docker buildx build \
            --platform linux/arm64 \
            -f Dockerfile.edge \
            -t balena-registry.com/v2/udara-edge-fleet/edge-runtime:latest \
            --load \
            .
      
      - name: Push to Balena Registry
        run: |
          docker push \
            balena-registry.com/v2/udara-edge-fleet/edge-runtime:latest

  deploy-fleet:
    name: Deploy to Balena Fleet
    runs-on: ubuntu-latest
    needs: build-multiarch
    environment: edge-production
    steps:
      - uses: actions/checkout@v4
      
      - name: Login to Balena
        run: |
          echo ${{ secrets.BALENA_API_KEY }} | balena login --token-stdin
      
      - name: Set fleet target
        run: balena fleet udara-edge-fleet
      
      - name: Push to fleet
        run: |
          balena push udara-edge-fleet \
            --source . \
            --detached
      
      - name: Monitor rollout
        run: |
          echo "Waiting for fleet rollout..."
          sleep 300  # 5 min initial wait
          
          # Check fleet status
          STATUS=$(balena devices --json | jq -r '.[].status')
          echo "Device statuses: $STATUS"
          
          # Wait for all devices to be "idle" (updated)
          TIMEOUT=3600  # 1 hour max
          ELAPSED=300
          while [ $ELAPSED -lt $TIMEOUT ]; do
            NON_IDLE=$(balena devices --json | jq -r '.[] | select(.status != "idle") | .uuid' | wc -l)
            if [ "$NON_IDLE" -eq 0 ]; then
              echo "All devices updated successfully"
              break
            fi
            echo "Waiting for $NON_IDLE devices to update... (${ELAPSED}s elapsed)"
            sleep 60
            ELAPSED=$((ELAPSED + 60))
          done
          
          if [ $ELAPSED -ge $TIMEOUT ]; then
            echo "WARNING: Rollout timeout after 1 hour"
            balena devices --json | jq -r '.[] | "\(.uuid): \(.status)"'
          fi
```

---

## GitHub Actions: CD Frontend (Vercel)

```yaml
# .github/workflows/cd-frontend.yml
name: Deploy Frontend

on:
  push:
    branches: [main]
    paths:
      - 'frontend/**'
      - 'public/**'

jobs:
  deploy:
    name: Deploy to Vercel
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      
      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: '20'
          cache: 'npm'
          cache-dependency-path: frontend/package-lock.json
      
      - name: Install dependencies
        run: cd frontend && npm ci
      
      - name: Build
        run: cd frontend && npm run build
      
      - name: Deploy to Vercel Production
        uses: amondnet/vercel-action@v25
        with:
          vercel-token: ${{ secrets.VERCEL_TOKEN }}
          vercel-org-id: ${{ secrets.VERCEL_ORG_ID }}
          vercel-project-id: ${{ secrets.VERCEL_PROJECT_ID }}
          vercel-args: '--prod'
          working-directory: ./frontend

  lighthouse-ci:
    name: Lighthouse Audit
    runs-on: ubuntu-latest
    needs: deploy
    steps:
      - uses: actions/checkout@v4
      
      - name: Run Lighthouse CI
        uses: treosh/lighthouse-ci-action@v11
        with:
          urls: https://udara.ai
          budgetPath: frontend/lighthouse-budget.json
          uploadArtifacts: true
          quiet: true
```

### Lighthouse Budget

```json
{
  "ci": {
    "assertions": {
      "categories:performance": ["error", { "minScore": 0.90 }],
      "categories:accessibility": ["error", { "minScore": 0.95 }],
      "categories:best-practices": ["error", { "minScore": 0.90 }],
      "categories:seo": ["warn", { "minScore": 0.80 }],
      "first-contentful-paint": ["error", { "maxNumericValue": 2000 }],
      "largest-contentful-paint": ["error", { "maxNumericValue": 3000 }],
      "cumulative-layout-shift": ["error", { "maxNumericValue": 0.1 }]
    }
  }
}
```

---

## GitHub Actions: Nightly Pipeline

```yaml
# .github/workflows/nightly.yml
name: Nightly Tests

on:
  schedule:
    - cron: '0 2 * * *'  # 2 AM UTC daily
  workflow_dispatch:  # Manual trigger

jobs:
  full-test-suite:
    name: Full Test Suite
    runs-on: ubuntu-latest
    services:
      postgres:
        image: pgvector/pgvector:pg16
        env:
          POSTGRES_USER: test
          POSTGRES_PASSWORD: test
          POSTGRES_DB: udara_test
        ports:
          - 5432:5432
      redis:
        image: redis:7-alpine
        ports:
          - 6379:6379
      qdrant:
        image: qdrant/qdrant:latest
        ports:
          - 6333:6333
    steps:
      - uses: actions/checkout@v4
      
      - name: Set up Python
        uses: actions/setup-python@v5
        with:
          python-version: '3.11'
      
      - name: Install all dependencies
        run: |
          pip install -r requirements.txt
          pip install -r requirements-dev.txt
      
      - name: Run full pytest
        env:
          DATABASE_URL: postgresql://test:test@localhost:5432/udara_test
          REDIS_URL: redis://localhost:6379/0
          QDRANT_URL: http://localhost:6333
          ENVIRONMENT: test
        run: pytest backend/ ml/ -v --tb=long --timeout=120

  load-test:
    name: Load Test (Locust)
    runs-on: ubuntu-latest
    needs: full-test-suite
    steps:
      - uses: actions/checkout@v4
      
      - name: Set up Python
        uses: actions/setup-python@v5
        with:
          python-version: '3.11'
      
      - name: Install locust
        run: pip install locust
      
      - name: Run load test
        run: |
          locust -f tests/load/locustfile.py \
            --headless \
            --users 500 \
            --spawn-rate 10 \
            --run-time 5m \
            --host http://localhost:8000 \
            --format json \
            --output load-test-results.json
        env:
          LOCUST_HOST: https://api.udara.ai
      
      - name: Check load test thresholds
        run: |
          python3 -c "
          import json
          with open('load-test-results.json') as f:
              data = json.load(f)
          stats = data['stats']
          for s in stats:
              if s['name'] == 'Total':
                  p99 = float(s['ninety_nine_percentile'])
                  failures = float(s['num_failures'])
                  total = float(s['num_requests'])
                  fail_rate = failures / total if total > 0 else 0
                  if p99 > 5000:
                      print(f'FAIL: P99 latency {p99:.0f}ms > 5000ms')
                      exit(1)
                  if fail_rate > 0.01:
                      print(f'FAIL: Failure rate {fail_rate:.2%} > 1%')
                      exit(1)
                  print(f'PASS: P99={p99:.0f}ms, Failures={fail_rate:.2%}')
                  break
          "

  model-eval:
    name: Model Evaluation Benchmarks
    runs-on: ubuntu-latest
    needs: full-test-suite
    steps:
      - uses: actions/checkout@v4
      
      - name: Set up Python
        uses: actions/setup-python@v5
        with:
          python-version: '3.11'
      
      - name: Install dependencies
        run: pip install -r requirements.txt -r requirements-dev.txt
      
      - name: Run NER evaluation
        run: |
          python ml/tests/evaluate_ner.py \
            --model-path models/afrobert_ner \
            --test-data data/ner_test.jsonl \
            --output ner_eval_results.json
      
      - name: Run OCR evaluation
        run: |
          python ml/tests/evaluate_ocr.py \
            --test-images data/ocr_test/ \
            --output ocr_eval_results.json
      
      - name: Run Bayesian model calibration check
        run: |
          python ml/tests/evaluate_bayesian.py \
            --test-data data/resistance_test.jsonl \
            --output bayesian_eval_results.json
      
      - name: Upload evaluation results
        uses: actions/upload-artifact@v4
        with:
          name: model-eval-${{ github.run_number }}
          path: |
            ner_eval_results.json
            ocr_eval_results.json
            bayesian_eval_results.json
```

---

## Balena Fleet Management

### Fleet Configuration

| Setting | Value |
|---------|-------|
| **Fleet Name** | `udara-edge-fleet` |
| **Device Type** | Raspberry Pi 5 (arm64) |
| **Balena OS Version** | 2024.10 (Debian Bookworm based) |
| **Multi-container** | Yes (3 containers) |
| **VPN** | OpenVPN (Balena built-in) |
| **Logging** | Balena log stream + local journald |
| **Updates** | Automatic with rolling strategy |

### Container Architecture

```yaml
# docker-compose.yml (Balena)
version: "3.8"

services:
  edge-runtime:
    build:
      context: .
      dockerfile: Dockerfile.edge
      target: production
    restart: always
    ports:
      - "8000:8000"
    volumes:
      - udara-data:/app/data
    environment:
      - DEVICE_ID=${DEVICE_ID}
      - HEALTH_POST_ID=${HEALTH_POST_ID}
      - CLOUD_API_URL=${CLOUD_API_URL}
      - SYNC_INTERVAL=${SYNC_INTERVAL:-900}
      - LOG_LEVEL=${LOG_LEVEL:-info}
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:8000/health"]
      interval: 30s
      timeout: 10s
      retries: 3
      start_period: 60s

  model-server:
    build:
      context: .
      dockerfile: Dockerfile.models
    restart: always
    volumes:
      - model-cache:/app/models
    environment:
      - MODEL_BACKEND=llama.cpp
      - LLAMA_MODEL_PATH=/app/models/llama-3.2-3b-q4.gguf
      - NER_MODEL_PATH=/app/models/afrobert-ner-int8
    healthcheck:
      test: ["CMD", "python", "-c", "import requests; requests.get('http://localhost:8001/health')"]
      interval: 60s
      timeout: 10s
      retries: 3
      start_period: 120s

  monitoring:
    image: prom/node-exporter:latest
    restart: always
    ports:
      - "9100:9100"
    volumes:
      - /proc:/host/proc:ro
      - /sys:/host/sys:ro
      - /:/rootfs:ro

volumes:
  udara-data:
  model-cache:
```

### Rolling Update Strategy

```
Step 1: Push new release
    │
    ▼
Step 2: Canary (10% of fleet)
    │  - Randomly select 10% of devices
    │  - Deploy new containers
    │  - Observe for 30 minutes
    │  - Metrics: CPU, memory, API error rate, response time
    │
    ├── All metrics OK?
    │   ▼
Step 3: Progressive (50%)
    │  - Deploy to next 50% of devices
    │  - Observe for 30 minutes
    │
    ├── All metrics OK?
    │   ▼
Step 4: Full rollout (100%)
    │  - Deploy to remaining devices
    │  - Fleet fully updated
    │
    ├── Metrics NOT OK at any stage?
    │   ▼
    ▼
Auto-Rollback
    - Revert all updated devices to previous version
    - Alert engineering team
    - Log incident for post-mortem
```

### Auto-Rollback Triggers

| Metric | Threshold | Duration | Action |
|--------|-----------|----------|--------|
| CPU usage | > 90% | 5 minutes | Auto-rollback |
| Memory usage | > 85% | 10 minutes | Auto-rollback |
| API error rate | > 5% | 5 minutes | Auto-rollback |
| API P99 latency | > 10 seconds | 5 minutes | Auto-rollback |
| Container restarts | > 3 in 10 min | Any | Auto-rollback |

### Fleet Environment Variables

| Variable | Value | Description |
|----------|-------|-------------|
| `DEVICE_ID` | `RPi-NG-IBD-001` | Unique device identifier |
| `HEALTH_POST_ID` | `HP-NG-IBD-JERICHO` | Associated health post |
| `CLOUD_API_URL` | `https://api.udara.ai` | Cloud API endpoint |
| `SYNC_INTERVAL` | `900` | Sync every 15 minutes |
| `LOG_LEVEL` | `info` | Logging verbosity |
| `COUNTRY_CONFIG` | `ng` | Country-specific configuration |
| `DEFAULT_LANGUAGE` | `en` | Default UI language |

---

## Terraform: AWS Infrastructure

### `main.tf`

```hcl
provider "aws" {
  region = "af-south-1"
  
  default_tags {
    tags = {
      Project     = "udara-ai"
      Environment = terraform.workspace
      ManagedBy   = "terraform"
    }
  }
}

# ─── VPC ─────────────────────────────────────────────────────────

module "vpc" {
  source  = "terraform-aws-modules/vpc/aws"
  version = "5.5.0"
  
  name = "udara-vpc-${terraform.workspace}"
  cidr = "10.0.0.0/16"
  
  azs             = ["af-south-1a", "af-south-1b"]
  private_subnets = ["10.0.1.0/24", "10.0.2.0/24"]
  public_subnets  = ["10.0.101.0/24", "10.0.102.0/24"]
  
  enable_nat_gateway = true
  single_nat_gateway = true
  
  enable_dns_hostnames = true
  enable_dns_support   = true
}

# ─── ECS Fargate ─────────────────────────────────────────────────

resource "aws_ecs_cluster" "udara" {
  name = "udara-cluster-${terraform.workspace}"
  
  setting {
    name  = "containerInsights"
    value = "enabled"
  }
}

resource "aws_ecs_task_definition" "api" {
  family                   = "udara-api-${terraform.workspace}"
  network_mode             = "awsvpc"
  requires_compatibilities = ["FARGATE"]
  cpu                      = "512"
  memory                   = "1024"
  
  container_definitions = jsonencode([
    {
      name  = "udara-api"
      image = "${aws_ecr_repository.udara.repository_url}:latest"
      portMappings = [{ containerPort = 8000 }]
      environment = [
        { name = "ENVIRONMENT", value = terraform.workspace },
        { name = "DATABASE_URL", value = aws_rds_cluster.udara.endpoint },
        { name = "REDIS_URL", value = "redis://${aws_elasticache_cluster.udara.cache_nodes[0].address}:6379/0" },
        { name = "LOG_LEVEL", value = "info" }
      ]
      secrets = [
        { name = "SECRET_KEY", valueFrom = aws_secretsmanager_secret.api_secret.arn },
        { name = "DB_PASSWORD", valueFrom = aws_secretsmanager_secret.db_password.arn }
      ]
      logConfiguration = {
        logDriver = "awslogs"
        options = {
          "awslogs-group"         = aws_cloudwatch_log_group.udara.name
          "awslogs-region"        = "af-south-1"
          "awslogs-stream-prefix" = "udara-api"
        }
      }
      healthCheck = {
        command     = ["CMD-SHELL", "curl -f http://localhost:8000/health || exit 1"]
        interval    = 30
        timeout     = 10
        retries     = 3
        startPeriod = 60
      }
    }
  ])
}

resource "aws_ecs_service" "api" {
  name            = "udara-api-service-${terraform.workspace}"
  cluster         = aws_ecs_cluster.udara.id
  task_definition = aws_ecs_task_definition.api.arn
  desired_count   = terraform.workspace == "production" ? 2 : 1
  
  network_configuration {
    subnets         = module.vpc.private_subnets
    security_groups = [aws_security_group.ecs.id]
  }
  
  load_balancer {
    target_group_arn = aws_lb_target_group.api.arn
    container_name   = "udara-api"
    container_port   = 8000
  }
}

# ─── Application Load Balancer ────────────────────────────────────

resource "aws_lb" "api" {
  name               = "udara-alb-${terraform.workspace}"
  internal           = false
  load_balancer_type = "application"
  security_groups    = [aws_security_group.alb.id]
  subnets            = module.vpc.public_subnets
}

resource "aws_lb_target_group" "api" {
  name     = "udara-api-tg-${terraform.workspace}"
  port     = 8000
  protocol = "HTTP"
  vpc_id   = module.vpc.vpc_id
  
  health_check {
    path = "/health"
  }
}

resource "aws_lb_listener" "https" {
  load_balancer_arn = aws_lb.api.arn
  port              = 443
  protocol          = "HTTPS"
  ssl_policy        = "ELBSecurityPolicy-2016-08"
  certificate_arn   = aws_acm_certificate.udara.arn
  
  default_action {
    type             = "forward"
    target_group_arn = aws_lb_target_group.api.arn
  }
}

# ─── RDS PostgreSQL ────────────────────────────────────────────────

resource "aws_rds_cluster" "udara" {
  engine                = "aurora-postgresql"
  engine_version        = "15.4"
  database_name         = "udara"
  master_username       = "udara_admin"
  manage_master_password = true
  
  vpc_security_group_ids = [aws_security_group.rds.id]
  db_subnet_group_name   = aws_db_subnet_group.udara.name
  
  serverlessv2_scaling_configuration {
    min_capacity = 0.5
    max_capacity = 2
  }
}

# ─── ElastiCache Redis ───────────────────────────────────────────

resource "aws_elasticache_cluster" "udara" {
  cluster_id           = "udara-redis-${terraform.workspace}"
  engine               = "redis"
  engine_version       = "7.0"
  node_type            = "cache.t3.micro"
  num_cache_nodes     = 1
  parameter_group_name = "default.redis7"
  port                 = 6379
  
  subnet_group_name  = aws_elasticache_subnet_group.udara.name
  security_group_ids = [aws_security_group.redis.id]
}
```

### `variables.tf`

```hcl
variable "environment" {
  type    = string
  default = "development"
}

variable "allowed_cidr_blocks" {
  type    = list(string)
  default = ["0.0.0.0/0"]  # Restrict in production
}
```

---

## Vercel: Frontend Deployment

### `vercel.json`

```json
{
  "buildCommand": "npm run build",
  "outputDirectory": ".next",
  "framework": "nextjs",
  "regions": ["cpt1", "jnb1"],
  "headers": [
    {
      "source": "/api/(.*)",
      "headers": [
        { "key": "Cache-Control", "value": "no-store" },
        { "key": "X-Frame-Options", "value": "DENY" },
        { "key": "X-Content-Type-Options", "value": "nosniff" }
      ]
    },
    {
      "source": "/(.*)",
      "headers": [
        { "key": "X-Frame-Options", "value": "SAMEORIGIN" },
        { "key": "Referrer-Policy", "value": "strict-origin-when-cross-origin" }
      ]
    }
  ],
  "rewrites": [
    { "source": "/api/(.*)", "destination": "https://api.udara.ai/api/$1" }
  ]
}
```

---

## Release Process

```
Conventional Commit → semantic-release → CHANGELOG → Git Tag → Docker Tag → Deploy
```

| Commit Type | Version Bump | Example |
|-------------|-------------|---------|
| `feat:` | Minor (1.x.0) | `feat(asr): add MMS-ASR pipeline` |
| `fix:` | Patch (1.0.x) | `fix(sync): handle retry timeout` |
| `perf:` | Patch | `perf(ocr): reduce inference time 30%` |
| `BREAKING CHANGE:` | Major (2.0.0) | `feat!: redesign API v2` |
| `docs:`, `chore:`, `test:` | No bump | `docs: update README` |

---

## Testing Pyramid

```
                    ┌─────────┐
                    │   E2E   │  Playwright + BotFather  (slow, rare)
                    │  Tests  │
                 ┌──┴─────────┴──┐
                 │  Integration │  Docker Compose (medium, per PR)
                 │    Tests     │
              ┌──┴──────────────┴──┐
              │    Load Tests      │  Locust (weekly, 500 users)
              │                    │
           ┌──┴────────────────────┴──┐
           │   Model Eval Tests     │  F1, WER, Brier (nightly)
           │                        │
        ┌──┴────────────────────────┴──┐
        │     Unit Tests (pytest)       │  Fast, every commit, 80% cov
        └──────────────────────────────┘
```

---

## Secret Management

| Tool | What It Manages | Where |
|------|----------------|-------|
| **GitHub Secrets** | CI/CD tokens, ECR credentials, Vercel tokens | GitHub repo settings |
| **HashiCorp Vault** | Production API keys, DB passwords, encryption keys | AWS (self-hosted) |
| **Balena Fleet Vars** | Device IDs, sync intervals, feature flags | Balena Cloud dashboard |
| **`.env.example`** | Template for local development | Repo root (committed) |

---

## Cost Estimates

| Service | Tier | Monthly Cost (USD) |
|---------|------|---------------------|
| AWS ECS Fargate | 2 tasks × 0.5 vCPU × 1GB | $25 |
| AWS Aurora Serverless v2 | 0.5-2 ACU (PostgreSQL) | $40 |
| ElastiCache Redis | cache.t3.micro | $15 |
| Application Load Balancer | Standard | $20 |
| NAT Gateway | 1 unit (af-south-1) | $35 |
| S3 (sync archives) | ~10 GB | $0.25 |
| CloudWatch Logs | ~5 GB/month | $2 |
| Vercel (frontend) | Hobby → Pro | $0-20 |
| Balena Cloud | Fleet of 3-100 devices | $0-75 |
| GitHub Actions | Included (free tier) | $0 |
| Domain + SSL | udara.ai | $12 |
| **TOTAL (pilot)** | | **~$150-250/month** |
| **TOTAL (scale)** | | **~$300-500/month** |

---

> **Document Version**: v1.0 | **Last Updated**: 2026-05-27 | **Status**: Final
