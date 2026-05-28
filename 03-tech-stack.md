# 03 — Tech Stack

> Complete technology inventory for every layer of the UDARA AI platform — from the
> Raspberry Pi's metal to the browser's pixels.

---

## Table of Contents

- [Edge Layer](#edge-layer)
  - [Hardware](#hardware)
  - [Operating System & Fleet Management](#operating-system--fleet-management)
  - [Application Runtime](#application-runtime)
  - [Database & Storage](#database--storage)
  - [AI/ML Runtime](#aiml-runtime)
  - [Communication & Bot Frameworks](#communication--bot-frameworks)
  - [Security & Privacy](#security--privacy-1)
- [Cloud Layer](#cloud-layer)
  - [Infrastructure](#infrastructure)
  - [Compute](#compute)
  - [Database & Storage](#database--storage-1)
  - [Caching & Messaging](#caching--messaging)
  - [Identity & Access Management](#identity--access-management)
  - [Secret Management](#secret-management)
  - [CDN & Content Delivery](#cdn--content-delivery)
  - [ML Training Infrastructure](#ml-training-infrastructure)
  - [Observability](#observability-1)
- [Frontend Layer](#frontend-layer)
  - [Core Framework](#core-framework)
  - [UI Component Library](#ui-component-library)
  - [Styling](#styling)
  - [Map Library](#map-library)
  - [Charting Library](#charting-library)
  - [Data Table](#data-table)
  - [Data Fetching & State](#data-fetching--state)
  - [Form Handling](#form-handling)
  - [Icons](#icons)
  - [Authentication](#authentication)
  - [PWA](#pwa)
- [Security Layer](#security-layer)
- [DevOps & CI/CD](#devops--cicd)
- [Shared Libraries](#shared-libraries)
- [Development Tools](#development-tools)

---

## Edge Layer

### Hardware

| Component | Specification | Model/Brand | Purpose |
|-----------|--------------|-------------|---------|
| **SBC** | Raspberry Pi 5 Model B | Broadcom BCM2712 | Primary compute |
| **CPU** | 4× Cortex-A76 @ 2.4GHz | ARM v8.2-A | AI inference + API |
| **RAM** | 8 GB LPDDR4X-4267 | — | Model loading + data |
| **Storage (boot)** | 128 GB microSD A2 | SanDisk High Endurance | OS + SQLite + ChromaDB |
| **Storage (data)** | 256 GB USB SSD | Samsung T7 | Models + USB sync |
| **Power** | 27W USB-C PD | Official RPi PSU | Reliable power |
| **UPS** | 18650 Li-ion | Waveshare Pi UPS Plus | Brownout protection |
| **Cellular** | USB LTE dongle | Huawei E3372 | Mobile connectivity |
| **Case** | Aluminium heatsink case | Argon ONE M.2 | Passive + active cooling |
| **Thermal** | 5V fan + heatsink | Argon ONE | Thermostatic @ 60°C |

### Operating System & Fleet Management

| Technology | Version | Purpose | Configuration |
|-----------|---------|---------|---------------|
| **Ubuntu Core** | 22.04 LTS | Immutable OS with snap packages, OTA updates | `snap refresh --channel=stable` |
| **Balena Cloud** | Latest | Fleet management, remote deployment, health monitoring | Fleet: `udara-edge-{region}` |
| **systemd** | 252+ | Service management, health checks | `udara-edge.service` (Type=notify) |
| **fake-hwclock** | System | Persist clock across reboots (Pi has no RTC) | Saves time to `/etc/fake-hwclock.data` on shutdown |
| **log2ram** | Latest | Keep logs in RAM, flush to disk hourly | Mount: `/var/log` as tmpfs, 40 MB |
| **chrony** | 4.4+ | NTP time sync | Pool: `pool.ntp.org`, fallback to GPS |
| **AppArmor** | Kernel | Mandatory access control for services | Profiles for `udara-edge` service |
| **ufw** | Latest | Firewall | Allow: 22, 80, 443, 8001, 6379 (local only) |

```yaml
# balena/balena.yml — Fleet configuration
name: udara-edge
type: "sw.arm64"
docker-compose:
  - docker-compose.edge.yml

# balena/docker-compose.edge.yml — Service definition
version: "3.8"
services:
  edge-api:
    build:
      context: ./edge
      dockerfile: Dockerfile
    restart: always
    ports:
      - "8001:8001"
    volumes:
      - udara-data:/data
      - udara-models:/models
      - udara-backup:/backup
    environment:
      - UDARA_ENV=${UDARA_ENV}
      - UDARA_SECRET_KEY=${UDARA_SECRET_KEY}
      - EDGE_NODE_ID=${EDGE_NODE_ID}
      - REDIS_HOST=redis
    depends_on:
      redis:
        condition: service_healthy
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:8001/health"]
      interval: 30s
      timeout: 10s
      retries: 3
      start_period: 60s
    deploy:
      resources:
        limits:
          memory: 6G
          cpus: "3.5"

  redis:
    image: redis:7-alpine
    restart: always
    volumes:
      - redis-data:/data
    command: redis-server --maxmemory 256mb --maxmemory-policy allkeys-lru --appendonly yes
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 10s
      timeout: 5s
      retries: 5

volumes:
  udara-data:
  udara-models:
  udara-backup:
  redis-data:
```

### Application Runtime

| Technology | Version | Purpose | Configuration |
|-----------|---------|---------|---------------|
| **Python** | 3.11.x | Application runtime | `uv` package manager, `--no-cache-dir` builds |
| **FastAPI** | 0.115+ | ASGI web framework | `workers=4`, `limit_concurrency=100` |
| **Uvicorn** | 0.32+ | ASGI server | `--host 0.0.0.0 --port 8001 --loop uvloop` |
| **uvloop** | Latest | Fast async event loop | `asyncio.set_event_loop_policy(uvloop.EventLoopPolicy())` |
| **Pydantic** | 2.x | Data validation & settings | `model_config = ConfigDict(strict=True)` |
| **SQLAlchemy** | 2.x | ORM for SQLite | `create_engine("sqlite+aiosqlite:///data/reports.db")` |
| **aiosqlite** | 0.20+ | Async SQLite driver | WAL mode, busy_timeout=5000 |
| **httpx** | 0.27+ | Async HTTP client | For cloud sync, Africa's Talking API |
| **Pillow** | 10.x | Image processing | OCR preprocessing pipeline |
| **numpy** | 1.26+ | Numerical computing | AI model preprocessing |

```toml
# edge/pyproject.toml
[project]
name = "udara-edge"
version = "0.1.0"
requires-python = ">=3.11"
dependencies = [
    # Web framework
    "fastapi>=0.115.0",
    "uvicorn[standard]>=0.32.0",
    "uvloop>=0.21.0",
    "pydantic>=2.10.0",
    "pydantic-settings>=2.7.0",
    
    # Database
    "sqlalchemy[asyncio]>=2.0.36",
    "aiosqlite>=0.20.0",
    "alembic>=1.14.0",
    
    # Storage
    "chromadb>=0.5.0",
    "redis>=5.0.0",
    
    # AI/ML
    "torch>=2.5.0",
    "transformers>=4.46.0",
    "onnxruntime>=1.20.0",
    "paddleocr>=2.7.0",
    "paddlepaddle>=2.6.0",
    "pymc>=5.16.0",
    "scipy>=1.14.0",
    "sentence-transformers>=3.3.0",
    "llama-cpp-python>=0.3.0",
    "fasttext-wheel>=0.9.2",
    "whoosh>=2.7.4",
    
    # NLP
    "spacy>=3.7.0",
    "presidio-analyzer>=2.2.0",
    "presidio-anonymizer>=2.2.0",
    
    # Communications
    "africastalking>=1.2.0",
    "aiogram>=3.13.0",
    "httpx>=0.27.0",
    
    # Crypto
    "cryptography>=44.0.0",
    
    # Utilities
    "numpy>=1.26.0",
    "pillow>=10.4.0",
    "pyyaml>=6.0.0",
    "tenacity>=9.0.0",
    
    # Monitoring
    "prometheus-client>=0.21.0",
    "sentry-sdk>=2.19.0",
]

[tool.uv]
dev-dependencies = [
    "pytest>=8.0.0",
    "pytest-asyncio>=0.24.0",
    "pytest-cov>=6.0.0",
    "ruff>=0.8.0",
    "mypy>=1.13.0",
]
```

### Database & Storage

| Technology | Version | Purpose | Configuration |
|-----------|---------|---------|---------------|
| **SQLite** | 3.44+ | Primary edge database | WAL mode, busy_timeout=5000ms, journal_mode=WAL |
| **ChromaDB** | 0.5+ | Local vector embeddings store | `chromadb.PersistentClient(path="/data/chroma")` |
| **Redis** | 7.x | Session cache, rate limiting | `maxmemory 256mb`, `allkeys-lru` eviction |
| **Whoosh** | 2.7.4 | Full-text BM25 search index | `index_dir="/data/whoosh_index"` |

```python
# edge/app/db/session.py
"""SQLite WAL mode configuration for edge database."""

from sqlalchemy import event, text
from sqlalchemy.ext.asyncio import create_async_engine, async_sessionmaker, AsyncSession

engine = create_async_engine(
    "sqlite+aiosqlite:///data/reports.db",
    echo=False,
    pool_pre_ping=True,
    pool_size=5,
    max_overflow=10,
)


@event.listens_for(engine.sync_engine, "connect")
def set_sqlite_pragmas(dbapi_conn, connection_record):
    """Set SQLite pragmas for performance and safety."""
    cursor = dbapi_conn.cursor()
    cursor.execute("PRAGMA journal_mode=WAL")          # Concurrent reads during writes
    cursor.execute("PRAGMA synchronous=NORMAL")        # Durability + performance balance
    cursor.execute("PRAGMA busy_timeout=5000")         # Wait up to 5s for lock
    cursor.execute("PRAGMA cache_size=-64000")          # 64 MB cache
    cursor.execute("PRAGMA temp_store=MEMORY")          # Temp tables in RAM
    cursor.execute("PRAGMA mmap_size=268435456")        # 256 MB memory-mapped I/O
    cursor.execute("PRAGMA foreign_keys=ON")           # Enforce FK constraints
    cursor.execute("PRAGMA wal_autocheckpoint=1000")    # Checkpoint every 1000 pages
    cursor.close()


SessionLocal = async_sessionmaker(engine, class_=AsyncSession, expire_on_commit=False)
```

### AI/ML Runtime

| Technology | Version | Purpose | Size on Disk |
|-----------|---------|---------|-------------|
| **ONNX Runtime** | 1.20+ | Inference engine for INT8 models | ~25 MB |
| **llama.cpp** | Latest (Python bindings) | LLM inference for Llama 3.2 3B | ~5 MB (lib) + 1.8 GB (model) |
| **PyTorch** | 2.5+ (CPU only) | Model loading, preprocessing | ~200 MB (CPU-only) |
| **HuggingFace Transformers** | 4.46+ | Model loading, tokenizers | ~50 MB (core) |
| **PaddlePaddle** | 2.6+ (CPU only) | OCR inference backend | ~150 MB (CPU-only) |
| **PaddleOCR** | 2.7+ | Text extraction pipeline | ~100 MB (core) + ~50 MB (models) |
| **PyMC** | 5.16+ | Bayesian inference engine | ~30 MB |
| **scipy** | 1.14+ | Statistical distributions | ~60 MB |
| **sentence-transformers** | 3.3+ | Embedding model wrapper | ~20 MB (wrapper) |
| **fastText** | 0.9.2 | Language identification | ~5 MB (lib) + 2 MB (model) |
| **Whoosh** | 2.7.4 | BM25 full-text search | ~2 MB |
| **spaCy** | 3.7+ | Rule-based NER (stage 1) | ~80 MB (en_core_web_sm) |
| **Presidio** | 2.2+ | PII detection and redaction | ~15 MB |

```python
# edge/app/models/registry.py (excerpt — loading strategy)
"""
Model loading and memory management for edge AI models.

Strategy:
- Always-loaded models: NER, LID, Embeddings, spaCy, Whoosh (~810 MB)
- On-demand models: OCR, ASR, LLM, PyMC (~2,350 MB)
- Budget: 3.5 GB max for all models
- Eviction: LRU-based for on-demand models
"""

import threading
import time
import logging
from dataclasses import dataclass, field
from enum import Enum
from typing import Any, Callable

logger = logging.getLogger(__name__)

MODEL_BUDGET_MB = 3500  # 3.5 GB hard limit


class LoadStrategy(str, Enum):
    ALWAYS_LOADED = "always_loaded"   # Loaded on startup, never evicted
    ON_DEMAND = "on_demand"           # Loaded when needed, evicted by LRU
    SCHEDULED = "scheduled"           # Loaded/unloaded by schedule


@dataclass
class ModelSpec:
    """Specification for an AI model in the registry."""
    name: str
    version: str
    load_strategy: LoadStrategy
    estimated_ram_mb: int
    load_fn: Callable[[], Any]
    unload_fn: Callable[[Any], None] | None = None
    loaded: bool = False
    instance: Any = None
    last_used: float = 0.0
    load_count: int = 0


# Model definitions (abbreviated — see 04-edge-ai-models.md for full details)
MODEL_SPECS = {
    "afrobert_ner": ModelSpec(
        name="afrobert_ner",
        version="1.2.0",
        load_strategy=LoadStrategy.ALWAYS_LOADED,
        estimated_ram_mb=320,
        load_fn=lambda: None,  # See full implementation in 04-edge-ai-models.md
    ),
    "fasttext_lid": ModelSpec(
        name="fasttext_lid",
        version="1.0.0",
        load_strategy=LoadStrategy.ALWAYS_LOADED,
        estimated_ram_mb=50,
        load_fn=lambda: None,
    ),
    "multilingual_e5": ModelSpec(
        name="multilingual_e5",
        version="1.0.0",
        load_strategy=LoadStrategy.ALWAYS_LOADED,
        estimated_ram_mb=500,
        load_fn=lambda: None,
    ),
    "paddle_ocr": ModelSpec(
        name="paddle_ocr",
        version="1.0.0",
        load_strategy=LoadStrategy.ON_DEMAND,
        estimated_ram_mb=200,
        load_fn=lambda: None,
    ),
    "mms_asr": ModelSpec(
        name="mms_asr",
        version="1.0.0",
        load_strategy=LoadStrategy.ON_DEMAND,
        estimated_ram_mb=400,
        load_fn=lambda: None,
    ),
    "llama_3_2_3b": ModelSpec(
        name="llama_3_2_3b",
        version="1.0.0",
        load_strategy=LoadStrategy.ON_DEMAND,
        estimated_ram_mb=2000,
        load_fn=lambda: None,
    ),
    "pymc_bayesian": ModelSpec(
        name="pymc_bayesian",
        version="1.0.0",
        load_strategy=LoadStrategy.ON_DEMAND,
        estimated_ram_mb=80,
        load_fn=lambda: None,
    ),
}
```

### Communication & Bot Frameworks

| Technology | Version | Purpose | Channel |
|-----------|---------|---------|---------|
| **africastalking-python** | 1.2+ | USSD + SMS via Africa's Talking | Door 1 (USSD/SMS) |
| **aiogram** | 3.13+ | Telegram Bot API (async) | Door 2 (Telegram) |
| **httpx** | 0.27+ | WhatsApp Cloud API calls | Door 2 (WhatsApp) |
| **websockets** | 13+ | WebSocket support for real-time | Door 3 (Dashboard) |

### Security & Privacy

| Technology | Version | Purpose | Notes |
|-----------|---------|---------|-------|
| **cryptography** | 44+ | AES-256-GCM encryption for sync payloads | `SyncEncryptor` class |
| **python-jose** | 3.3+ | JWT validation (Keycloak tokens) | RS256 algorithm |
| **presidio-analyzer** | 2.2+ | PII detection (names, phone numbers, IDs) | Custom recognizers for Nigerian IDs |
| **presidio-anonymizer** | 2.2+ | PII redaction before sync to cloud | Replaces PII with `<PATIENT_NAME>` etc. |
| **bcrypt** | 4.2+ | Password hashing (local admin accounts) | cost factor 12 |
| **limiter** | 1.2+ | Rate limiting (per-CHW, per-endpoint) | Redis-backed sliding window |

---

## Cloud Layer

### Infrastructure

| Technology | Version | Purpose | Configuration |
|-----------|---------|---------|---------------|
| **AWS Region** | af-south-1 (Cape Town) | Primary region for all cloud resources | Lowest latency for sub-Saharan Africa |
| **Terraform** | 1.9+ | Infrastructure as Code | Modules: ECS, RDS, ElastiCache, CloudFront, S3, Monitoring |
| **AWS CLI** | 2.x | Infrastructure management | `~/.aws/credentials` with MFA |

```hcl
# infra/main.tf (excerpt)
provider "aws" {
  region = var.aws_region

  default_tags {
    tags = {
      Project     = "udara-ai"
      Environment = var.environment
      ManagedBy   = "terraform"
    }
  }
}

module "ecs" {
  source = "./modules/ecs"

  environment = var.environment
  region      = var.aws_region
  
  service_name = "udara-cloud-api"
  cpu          = 1024     # 1 vCPU
  memory       = 4096     # 4 GB
  desired_count = 2
  
  # Auto-scaling
  min_capacity = 2
  max_capacity = 10
  cpu_target   = 70
}
```

### Compute

| Technology | Version | Purpose | Configuration |
|-----------|---------|---------|---------------|
| **ECS Fargate** | Latest | Serverless container orchestration | 2+ tasks, 4 GB RAM, 1 vCPU each |
| **ECS Service Connect** | Latest | Internal service mesh | Service discovery between API and workers |
| **Application Load Balancer** | Latest | HTTPS termination, routing | ACM certificate, WAF rules |
| **AWS WAF** | Latest | DDoS protection, rate limiting | Rate-based rules (2000 req/5min) |

### Database & Storage

| Technology | Version | Purpose | Configuration |
|-----------|---------|---------|---------------|
| **PostgreSQL** | 16.x | Primary relational database | RDS Multi-AZ, db.r6g.large |
| **TimescaleDB** | 2.14+ | Time-series extension for AMR metrics | Hypertables for `report_timeseries`, `resistance_estimates` |
| **PostGIS** | 3.4+ | Geospatial queries for map layer | ST_DWithin, ST_ClusterDBSCAN |
| **Qdrant** | 1.12+ | Vector database for cross-facility search | qdrant/qdrant Docker, 1 CPU, 4 GB |
| **S3** | Standard + Glacier | Object storage (images, audio, exports) | Versioning, lifecycle to Glacier after 90d |
| **CloudFront** | Latest | CDN for dashboard + static assets | Custom domain, TLS 1.3 |

```sql
-- Cloud: TimescaleDB hypertable setup
CREATE EXTENSION IF NOT EXISTS timescaledb;
CREATE EXTENSION IF NOT EXISTS postgis;

-- Reports time-series (hypertable for efficient time-range queries)
CREATE TABLE reports (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    edge_node_id    TEXT NOT NULL,
    chw_id          TEXT NOT NULL,
    facility_id     UUID NOT NULL REFERENCES facilities(id),
    patient_age_group TEXT NOT NULL,
    symptoms        JSONB NOT NULL,
    symptom_duration_days INT,
    drug_prescribed TEXT,
    drug_dosage     TEXT,
    treatment_outcome TEXT,
    source_channel  TEXT NOT NULL,
    language        TEXT,
    ner_confidence  FLOAT,
    location        GEOGRAPHY(POINT, 4326),
    entities        JSONB,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    synced_at       TIMESTAMPTZ,
    -- PostGIS index for spatial queries
    -- TimescaleDB index for time queries
);

SELECT create_hypertable('reports', 'created_at', chunk_time_interval => INTERVAL '1 week');

-- Spatial index
CREATE INDEX idx_reports_location ON reports USING GIST (location);

-- TimescaleDB continuous aggregates for resistance analytics
CREATE MATERIALIZED VIEW resistance_weekly
WITH (timescaledb.continuous) AS
SELECT
    time_bucket('1 week', created_at) AS week,
    drug_prescribed AS drug,
    facility_id,
    COUNT(*) AS total_reports,
    COUNT(*) FILTER (WHERE treatment_outcome = 'treatment_failure') AS failures,
    ROUND(
        COUNT(*) FILTER (WHERE treatment_outcome = 'treatment_failure')::FLOAT 
        / NULLIF(COUNT(*), 0) * 100, 
        2
    ) AS resistance_rate
FROM reports
WHERE drug_prescribed IS NOT NULL
GROUP BY week, drug_prescribed, facility_id;

-- Auto-refresh continuous aggregate
SELECT add_continuous_aggregate_policy('resistance_weekly',
    start_offset    => INTERVAL '3 days',
    end_offset      => INTERVAL '1 hour',
    schedule_interval => INTERVAL '1 hour');
```

### Caching & Messaging

| Technology | Version | Purpose | Configuration |
|-----------|---------|---------|---------------|
| **Redis 7** | 7.2+ | Application cache, Celery broker | ElastiCache r6g.large, 3-node cluster |
| **Celery** | 5.4+ | Distributed task queue | Redis broker, `--concurrency=4` |
| **Celery Beat** | 5.4+ | Periodic task scheduler | Sync processing, alert evaluation, model training |

```python
# cloud/workers/celery_app.py
"""Celery configuration for cloud async tasks."""

from celery import Celery
from celery.schedules import crontab

app = Celery("udara-cloud")

app.conf.update(
    broker_url="redis://redis:6379/0",
    result_backend="redis://redis:6379/1",
    task_serializer="json",
    result_serializer="json",
    accept_content=["json"],
    timezone="Africa/Lagos",
    enable_utc=True,
    task_track_started=True,
    task_acks_late=True,         # Retry on worker crash
    worker_prefetch_multiplier=1,  # One task per worker at a time
    task_soft_time_limit=300,    # 5 min soft limit
    task_time_limit=360,        # 6 min hard limit
)

# Scheduled tasks (Celery Beat)
app.conf.beat_schedule = {
    # Process incoming edge syncs every 60 seconds
    "process-sync-batch": {
        "task": "cloud.workers.tasks.sync_processing.process_pending_syncs",
        "schedule": 60.0,
    },
    # Evaluate alert rules every 5 minutes
    "evaluate-alerts": {
        "task": "cloud.workers.tasks.alert_evaluation.evaluate_all_rules",
        "schedule": 300.0,
    },
    # Generate weekly resistance report (every Monday 06:00 WAT)
    "weekly-resistance-report": {
        "task": "cloud.workers.tasks.report_generation.generate_weekly_report",
        "schedule": crontab(hour=6, minute=0, day_of_week=1),
    },
    # Fine-tune models monthly (1st of month, 02:00 WAT)
    "monthly-model-retraining": {
        "task": "cloud.workers.tasks.model_training.fine_tune_all_models",
        "schedule": crontab(hour=2, minute=0, day_of_month=1),
    },
    # Refresh TimescaleDB continuous aggregates (hourly)
    "refresh-materialized-views": {
        "task": "cloud.workers.tasks.sync_processing.refresh_continuous_aggregates",
        "schedule": 3600.0,
    },
}
```

### Identity & Access Management

| Technology | Version | Purpose | Configuration |
|-----------|---------|---------|---------------|
| **Keycloak** | 23+ | OIDC identity provider | Docker, PostgreSQL backend |
| **Keycloak Themes** | Custom | Branded login pages | UDARA logo, colours |
| **OIDC** | 1.0 | Authentication protocol | RS256 signing, 15 min access token TTL |
| **RBAC** | Custom | Role-based access control | 3 roles: facility_admin, state_admin, national_admin |

```yaml
# Keycloak realm export (excerpt)
realm: udara
roles:
  realm:
    - name: facility_admin
      description: "Access to single facility data"
    - name: state_admin
      description: "Access to all facilities in assigned state"
    - name: national_admin
      description: "Full platform access"
    - name: chw
      description: "Community Health Worker (bot user)"
    - name: researcher
      description: "Read-only access for research"

clients:
  - clientId: udara-web
    publicClient: true
    redirectUris: ["https://dashboard.udara.ai/*"]
    webOrigins: ["https://dashboard.udara.ai"]
    standardFlowEnabled: true
    directAccessGrantsEnabled: false
    
  - clientId: udara-api
    publicClient: false
    serviceAccountsEnabled: true
    authorizationServicesEnabled: true
```

### Secret Management

| Technology | Version | Purpose | Configuration |
|-----------|---------|---------|---------------|
| **HashiCorp Vault** | 1.17+ | Secret storage, key management | Auto-unseal with AWS KMS |
| **Vault Transit** | — | Encryption-as-a-service for sync keys | AES-256-GCM, key rotation every 90 days |
| **AWS KMS** | — | Vault auto-unseal | `aws-kms://` seal config |
| **AWS Secrets Manager** | — | Environment secrets backup | Cross-region replication |

### CDN & Content Delivery

| Technology | Version | Purpose | Configuration |
|-----------|---------|---------|---------------|
| **CloudFront** | Latest | CDN for Next.js dashboard | Origin: S3 + ECS ALB |
| **ACM** | — | TLS certificate management | `*.udara.ai` wildcard cert |
| **S3** | Standard | Static asset storage | Origin for CloudFront |

### ML Training Infrastructure

| Technology | Version | Purpose | Configuration |
|-----------|---------|---------|---------------|
| **PyTorch** | 2.5+ | Model training | CUDA-enabled on p3.2xlarge (V100 GPU) |
| **HuggingFace Transformers** | 4.46+ | Transformer fine-tuning | `Trainer` API, mixed precision (fp16) |
| **MLflow** | 2.17+ | Experiment tracking + model registry | S3 artifact store, PostgreSQL tracking |
| **Weights & Biases** | — | Optional experiment visualization | Integration with MLflow |
| **ONNX** | Latest | Model export for edge deployment | `torch.onnx.export()` + INT8 quantization |

```python
# cloud/ml/training.py (excerpt)
"""Fine-tuning pipeline for AfroBERT NER model."""

import mlflow
import torch
from transformers import (
    AutoTokenizer,
    AutoModelForTokenClassification,
    TrainingArguments,
    Trainer,
    DataCollatorForTokenClassification,
)

# MLflow tracking
mlflow.set_experiment("udara-ner-finetuning")
mlflow.autolog()

def fine_tune_ner(
    base_model: str = "xlm-roberta-base",
    dataset_path: str = "s3://udara-data-af-south-1/ner_training/",
    output_dir: str = "./models/afrobert_ner",
    epochs: int = 3,
    batch_size: int = 16,
    learning_rate: float = 2e-5,
):
    """Fine-tune XLM-RoBERTa for AMR NER on African health data."""
    
    tokenizer = AutoTokenizer.from_pretrained(base_model)
    model = AutoModelForTokenClassification.from_pretrained(
        base_model,
        num_labels=len(ENTITY_TYPES),
    )
    
    training_args = TrainingArguments(
        output_dir=output_dir,
        eval_strategy="epoch",
        learning_rate=learning_rate,
        per_device_train_batch_size=batch_size,
        per_device_eval_batch_size=batch_size,
        num_train_epochs=epochs,
        weight_decay=0.01,
        fp16=True,  # Mixed precision on V100
        logging_steps=50,
        save_strategy="epoch",
        load_best_model_at_end=True,
        metric_for_best_model="eval_f1",
    )
    
    trainer = Trainer(
        model=model,
        args=training_args,
        train_dataset=train_dataset,
        eval_dataset=eval_dataset,
        tokenizer=tokenizer,
        data_collator=DataCollatorForTokenClassification(tokenizer),
        compute_metrics=compute_ner_metrics,
    )
    
    trainer.train()
    
    # Export to ONNX for edge deployment
    export_to_onnx(model, tokenizer, output_dir)
    
    # Register in MLflow
    mlflow.register_model(
        f"{output_dir}/onnx",
        "afrobert-ner-edge",
    )
```

### Observability

| Technology | Version | Purpose | Configuration |
|-----------|---------|---------|---------------|
| **Prometheus** | 2.54+ | Metrics collection | `scrape_interval=15s`, retention=30d |
| **Grafana** | 11+ | Dashboard visualisation | Provisioned dashboards for all services |
| **Loki** | 2.9+ | Log aggregation | Promtail agent on each service |
| **AlertManager** | 0.27+ | Alert routing & dispatch | Slack, email, PagerDuty channels |
| **Sentry** | SaaS | Error tracking & performance | Python + JavaScript SDKs |
| **CloudWatch** | — | AWS-native metrics & logs | Backup for Prometheus/Loki |

---

## Frontend Layer

### Core Framework

| Technology | Version | Purpose | Configuration |
|-----------|---------|---------|---------------|
| **Next.js** | 14.x (App Router) | React framework with SSR/SSG | `output: "standalone"` for Docker |
| **TypeScript** | 5.x | Type-safe JavaScript | `strict: true`, `noUncheckedIndexedAccess: true` |
| **React** | 18.x | UI library | Concurrent features enabled |

```json
// web/package.json (excerpt)
{
  "name": "udara-web",
  "version": "0.1.0",
  "private": true,
  "scripts": {
    "dev": "next dev",
    "build": "next build",
    "start": "next start",
    "lint": "next lint",
    "type-check": "tsc --noEmit",
    "test": "jest",
    "test:e2e": "playwright test"
  },
  "dependencies": {
    "next": "^14.2.0",
    "react": "^18.3.0",
    "react-dom": "^18.3.0",
    "@tanstack/react-table": "^8.20.0",
    "@tanstack/react-query": "^5.60.0",
    "maplibre-gl": "^4.7.0",
    "echarts": "^5.5.0",
    "echarts-for-react": "^3.0.2",
    "zustand": "^5.0.0",
    "react-hook-form": "^7.53.0",
    "@hookform/resolvers": "^3.9.0",
    "zod": "^3.23.0",
    "lucide-react": "^0.460.0",
    "@keycloak/keycloak-js": "^23.0.0",
    "axios": "^1.7.0",
    "date-fns": "^4.1.0",
    "clsx": "^2.1.0",
    "tailwind-merge": "^2.6.0"
  },
  "devDependencies": {
    "typescript": "^5.6.0",
    "@types/react": "^18.3.0",
    "@types/node": "^22.0.0",
    "tailwindcss": "^4.0.0",
    "eslint": "^9.15.0",
    "eslint-config-next": "^14.2.0",
    "prettier": "^3.4.0",
    "jest": "^29.7.0",
    "@playwright/test": "^1.48.0"
  }
}
```

### UI Component Library

| Technology | Version | Purpose | Components Used |
|-----------|---------|---------|----------------|
| **shadcn/ui** | Latest | Accessible, composable components | Button, Card, Dialog, Table, Badge, Select, Input, Tabs, Sheet, DropdownMenu, Command, Toast, AlertDialog, Separator, ScrollArea, Skeleton, Tooltip |

### Styling

| Technology | Version | Purpose | Configuration |
|-----------|---------|---------|---------------|
| **Tailwind CSS** | 4.x | Utility-first CSS framework | `tailwind.config.ts` with custom UDARA theme |
| **class-variance-authority** | 0.7+ | Variant-based component styles | shadcn/ui uses this internally |
| **tailwind-merge** | 2.6+ | Merge Tailwind classes without conflicts | `cn()` utility function |
| **clsx** | 2.1+ | Conditional class names | Used with `tailwind-merge` |

### Map Library

| Technology | Version | Purpose | Configuration |
|-----------|---------|---------|---------------|
| **MapLibre GL JS** | 4.7+ | WebGL-based interactive maps | Custom tiles from OpenStreetMap, GeoJSON layers |
| **maplibre-gl-legend** | — | Map legend component | Resistance index legend |

```typescript
// web/src/components/map/MapView.tsx (excerpt)
import maplibregl from "maplibre-gl";
import "maplibre-gl/dist/maplibre-gl.css";

export function MapView({ facilities, resistanceData }: MapViewProps) {
  const mapContainer = useRef<HTMLDivElement>(null);
  const map = useRef<maplibregl.Map | null>(null);

  useEffect(() => {
    if (!mapContainer.current || map.current) return;

    map.current = new maplibregl.Map({
      container: mapContainer.current,
      style: {
        version: 8,
        sources: {
          osm: {
            type: "raster",
            tiles: ["https://tile.openstreetmap.org/{z}/{x}/{y}.png"],
            tileSize: 256,
            attribution: "© OpenStreetMap contributors",
          },
        },
        layers: [
          {
            id: "osm-tiles",
            type: "raster",
            source: "osm",
            minzoom: 0,
            maxzoom: 19,
          },
        ],
      },
      center: [3.379, 6.524], // Lagos
      zoom: 6,
    });

    // Add facility markers layer
    map.current.on("load", () => {
      map.current!.addSource("facilities", {
        type: "geojson",
        data: facilitiesGeoJSON,
      });

      map.current!.addLayer({
        id: "facility-circles",
        type: "circle",
        source: "facilities",
        paint: {
          "circle-radius": ["interpolate", ["linear"], ["get", "resistance_rate"],
            0, 8, 20, 8, 40, 16, 60, 24, 80, 32, 100, 40
          ],
          "circle-color": [
            "interpolate", ["linear"], ["get", "resistance_rate"],
            0, "#22c55e",  // green
            20, "#eab308",  // yellow
            40, "#f97316",  // orange
            60, "#ef4444",  // red
            80, "#dc2626",  // dark red
          ],
          "circle-stroke-width": 2,
          "circle-stroke-color": "#ffffff",
        },
      });
    });

    return () => map.current?.remove();
  }, []);

  return <div ref={mapContainer} className="w-full h-[500px] rounded-lg" />;
}
```

### Charting Library

| Technology | Version | Purpose | Chart Types Used |
|-----------|---------|---------|-----------------|
| **Apache ECharts** | 5.5+ | Interactive data visualisation | Line, Bar, Pie, Heatmap, Gauge, Scatter, Sankey |
| **echarts-for-react** | 3.0+ | React wrapper for ECharts | Responsive, themeable wrapper |

```typescript
// web/src/components/charts/ResistanceIndexChart.tsx (excerpt)
import ReactECharts from "echarts-for-react";

export function ResistanceIndexChart({ data }: Props) {
  const option: EChartsOption = {
    tooltip: { trigger: "axis" },
    legend: { data: ["Amoxicillin", "Ciprofloxacin", "Co-trimoxazole"] },
    xAxis: {
      type: "category",
      data: data.map((d) => d.week),
      axisLabel: { rotate: 45 },
    },
    yAxis: {
      type: "value",
      name: "Resistance Rate (%)",
      max: 100,
      axisLabel: { formatter: "{value}%" },
    },
    series: [
      {
        name: "Amoxicillin",
        type: "line",
        data: data.map((d) => d.amoxicillin),
        smooth: true,
        lineStyle: { width: 3 },
        areaStyle: { opacity: 0.1 },
        itemStyle: { color: "#ef4444" },
      },
      {
        name: "Ciprofloxacin",
        type: "line",
        data: data.map((d) => d.ciprofloxacin),
        smooth: true,
        lineStyle: { width: 3 },
        areaStyle: { opacity: 0.1 },
        itemStyle: { color: "#22c55e" },
      },
      {
        name: "Co-trimoxazole",
        type: "line",
        data: data.map((d) => d.co_trimoxazole),
        smooth: true,
        lineStyle: { width: 3 },
        areaStyle: { opacity: 0.1 },
        itemStyle: { color: "#f97316" },
      },
    ],
    grid: { left: "3%", right: "4%", bottom: "15%", contain: true },
  };

  return <ReactECharts option={option} style={{ height: 400 }} />;
}
```

### Data Table

| Technology | Version | Purpose | Configuration |
|-----------|---------|---------|---------------|
| **TanStack Table** | 8.20+ | Headless, performant data table | Virtual scrolling for 10,000+ rows, sorting, filtering, pagination |

### Data Fetching & State

| Technology | Version | Purpose | Configuration |
|-----------|---------|---------|---------------|
| **TanStack Query** | 5.60+ | Server state management, caching | Stale time: 30s, retry: 2, cache time: 5min |
| **Zustand** | 5.0+ | Client state management | `auth-store`, `filters-store`, `ui-store` |
| **Axios** | 1.7+ | HTTP client with interceptors | Auth token injection, refresh, error handling |

```typescript
// web/src/lib/query-client.ts
import { QueryClient } from "@tanstack/react-query";

export const queryClient = new QueryClient({
  defaultOptions: {
    queries: {
      staleTime: 30 * 1000,        // 30 seconds
      gcTime: 5 * 60 * 1000,       // 5 minutes
      retry: 2,
      refetchOnWindowFocus: false,  // Don't refetch on tab switch
    },
    mutations: {
      retry: 1,
    },
  },
});
```

### Form Handling

| Technology | Version | Purpose | Configuration |
|-----------|---------|---------|---------------|
| **React Hook Form** | 7.53+ | Performant form state management | Uncontrolled components, minimal re-renders |
| **Zod** | 3.23+ | Schema validation | `z.object()`, `z.enum()`, custom refinements |

### Icons

| Technology | Version | Purpose |
|-----------|---------|---------|
| **Lucide React** | 0.460+ | Open-source icon library (900+ icons) |

### Authentication

| Technology | Version | Purpose |
|-----------|---------|---------|
| **@keycloak/keycloak-js** | 23+ | OIDC client for React, token management |

### PWA

| Technology | Version | Purpose |
|-----------|---------|---------|
| **next-pwa** | 5.6+ | PWA support for Next.js |
| **Workbox** | 7.x | Service worker library (caching, offline) |

---

## Security Layer

| Technology | Version | Purpose | Details |
|-----------|---------|---------|---------|
| **TLS 1.3** | — | All communications encrypted | CloudFront → ALB → ECS, Edge → Cloud |
| **AES-256-GCM** | — | Sync payload encryption | 256-bit key, 96-bit nonce, AAD with node ID |
| **Keycloak OIDC** | 23+ | Authentication | RS256 JWT, 15 min access TTL, 8h refresh TTL |
| **HashiCorp Vault** | 1.17+ | Secret management | Transit engine, 90-day key rotation |
| **Presidio** | 2.2+ | PII detection & redaction | Names, phones, Nigerian NINs, addresses |
| **Rate Limiting** | — | Abuse prevention | Redis sliding window: 100 req/min per CHW |
| **CORS** | — | Cross-origin policy | Whitelist `*.udara.ai` only |
| **CSP** | — | Content Security Policy | Strict CSP headers on all responses |
| **Audit Logging** | — | Access tracking | All API calls logged with user, action, resource, timestamp |
| **Input Validation** | — | Injection prevention | Pydantic (backend), Zod (frontend), parameterized SQL |

---

## DevOps & CI/CD

| Technology | Version | Purpose | Configuration |
|-----------|---------|---------|---------------|
| **GitHub Actions** | — | CI/CD pipelines | `.github/workflows/*.yml` |
| **Docker** | 24+ | Containerization | Multi-stage builds, multi-arch (amd64 + arm64) |
| **Docker Buildx** | Latest | Multi-platform builds | `--platform linux/amd64,linux/arm64` |
| **Balena CLI** | Latest | Edge fleet deployment | `balena push udara-edge` |
| **Terraform** | 1.9+ | Infrastructure as Code | `terraform plan -apply` via GitHub Actions |
| **Prometheus** | 2.54+ | Metrics collection | Scrape configs for all services |
| **Grafana** | 11+ | Dashboards & alerting | Provisioned dashboards |
| **Loki** | 2.9+ | Log aggregation | Promtail agents |
| **Sentry** | SaaS | Error tracking | Python + JS SDKs, source maps uploaded |
| **Trivy** | Latest | Container security scanning | Weekly full scan, PR scan on push |
| **Renovate** | Latest | Dependency updates | Auto-merge patch updates, PR for minors |

```yaml
# .github/workflows/ci.yml (excerpt)
name: CI

on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main]

jobs:
  edge-test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-python@v5
        with:
          python-version: "3.11"
      - name: Install uv
        run: pip install uv
      - name: Install dependencies
        working-directory: edge
        run: uv sync --dev
      - name: Lint
        working-directory: edge
        run: uv run ruff check . && uv run mypy app/
      - name: Test
        working-directory: edge
        run: uv run pytest --cov=app --cov-report=xml
      - name: Upload coverage
        uses: codecov/codecov-action@v4
        with:
          file: edge/coverage.xml

  cloud-test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-python@v5
        with:
          python-version: "3.12"
      - name: Install uv
        run: pip install uv
      - name: Install dependencies
        working-directory: cloud
        run: uv sync --dev
      - name: Lint
        working-directory: cloud
        run: uv run ruff check . && uv run mypy app/
      - name: Test
        working-directory: cloud
        run: uv run pytest --cov=app --cov-report=xml

  web-test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: "20"
          cache: "npm"
          cache-dependency-path: web/package-lock.json
      - name: Install dependencies
        working-directory: web
        run: npm ci
      - name: Lint
        working-directory: web
        run: npm run lint && npm run type-check
      - name: Test
        working-directory: web
        run: npm test
      - name: Build
        working-directory: web
        run: npm run build

  security-scan:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Trivy scan (edge)
        uses: aquasecurity/trivy-action@master
        with:
          scan-type: "fs"
          scan-ref: "edge/"
          severity: "CRITICAL,HIGH"
      - name: Python bandit (edge)
        working-directory: edge
        run: pip install bandit && bandit -r app/ -ll
```

---

## Shared Libraries

### Python Shared (`shared/python/udara_shared/`)

| Module | Purpose |
|--------|---------|
| `schemas.py` | Pydantic models shared between edge and cloud (Report, Facility, User, SyncPayload) |
| `constants.py` | Shared enums (Drug, Symptom, Outcome, Channel), drug lists, ICD-10 AMR codes |
| `crypto.py` | `SyncEncryptor` class (AES-256-GCM encrypt/decrypt) |
| `sync.py` | Delta format definition, sync protocol constants |
| `validators.py` | Shared validation rules (phone number, drug name, date format) |

### TypeScript Shared (`shared/typescript/src/`)

| Module | Purpose |
|--------|---------|
| `types.ts` | TypeScript interfaces mirroring Pydantic schemas |
| `constants.ts` | Shared enums, drug lists, API route paths |
| `api-routes.ts` | Centralised API route definitions with typed responses |

---

## Development Tools

| Tool | Version | Purpose |
|------|---------|---------|
| **ruff** | 0.8+ | Python linter + formatter (replaces flake8, black, isort) |
| **mypy** | 1.13+ | Python static type checking |
| **pytest** | 8.x | Python testing framework |
| **pytest-asyncio** | 0.24+ | Async test support |
| **pytest-cov** | 6.x | Coverage reporting |
| **ESLint** | 9.x | JavaScript/TypeScript linting |
| **Prettier** | 3.4+ | JavaScript/TypeScript formatting |
| **Jest** | 29.x | JavaScript testing |
| **Playwright** | 1.48+ | End-to-end testing |
| **Docker Compose** | 2.20+ | Multi-container orchestration for dev |
| **Make** | 4.3+ | Task runner for common developer commands |
| **pre-commit** | 3.x | Git pre-commit hooks (lint, format, type-check) |

---

## Complete Version Pinning Summary

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    COMPLETE TECHNOLOGY INVENTORY                             │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  EDGE NODE                                                                 │
│  ├── Hardware: RPi 5 (8 GB), 128 GB SD, 256 GB SSD, UPS                   │
│  ├── OS: Ubuntu Core 22.04 + Balena Cloud                                 │
│  ├── Runtime: Python 3.11 + uv                                            │
│  ├── Web: FastAPI 0.115 + Uvicorn 0.32 + uvloop                           │
│  ├── DB: SQLite 3.44 (WAL) + ChromaDB 0.5 + Redis 7 + Whoosh 2.7        │
│  ├── AI: ONNX Runtime 1.20 + llama.cpp + PyTorch 2.5 (CPU)               │
│  ├── NLP: spaCy 3.7 + Transformers 4.46 + fastText 0.9.2                │
│  ├── OCR: PaddleOCR 2.7 + PaddlePaddle 2.6 (CPU)                         │
│  ├── ASR: MMS-ASR (latest)                                                │
│  ├── Bayes: PyMC 5.16 + scipy 1.14                                       │
│  ├── Embeddings: sentence-transformers 3.3 (multilingual-e5)              │
│  ├── LLM: llama-cpp-python 0.3 (Llama 3.2 3B INT4)                       │
│  ├── PII: Presidio 2.2                                                    │
│  ├── Bots: aiogram 3.13 (Telegram) + africastalking 1.2 (USSD/SMS)       │
│  ├── Crypto: cryptography 44                                              │
│  └── Monitoring: prometheus-client 0.21 + sentry-sdk 2.19                │
│                                                                             │
│  CLOUD                                                                      │
│  ├── Infra: AWS af-south-1 + Terraform 1.9                                │
│  ├── Compute: ECS Fargate (2+ tasks, 4 GB, 1 vCPU)                       │
│  ├── LB: Application Load Balancer + WAF                                  │
│  ├── Runtime: Python 3.12 + uv                                            │
│  ├── Web: FastAPI 0.115 + Uvicorn 0.32                                    │
│  ├── Tasks: Celery 5.4 + Celery Beat                                      │
│  ├── DB: PostgreSQL 16 + TimescaleDB 2.14 + PostGIS 3.4                  │
│  ├── Vector: Qdrant 1.12                                                  │
│  ├── Cache: Redis 7 (ElastiCache)                                         │
│  ├── Auth: Keycloak 23 (OIDC)                                             │
│  ├── Secrets: HashiCorp Vault 1.17 + AWS KMS                              │
│  ├── CDN: CloudFront + S3 + ACM                                           │
│  ├── ML: PyTorch 2.5 (CUDA) + HF Transformers 4.46 + MLflow 2.17         │
│  ├── Observability: Prometheus 2.54 + Grafana 11 + Loki 2.9 + Sentry     │
│  └── CI/CD: GitHub Actions + Docker 24 + Trivy + Renovate                │
│                                                                             │
│  FRONTEND                                                                   │
│  ├── Framework: Next.js 14 (App Router) + TypeScript 5.6                 │
│  ├── UI: shadcn/ui + Tailwind CSS 4 + Lucide React                       │
│  ├── Maps: MapLibre GL JS 4.7                                             │
│  ├── Charts: ECharts 5.5 + echarts-for-react 3.0                         │
│  ├── Tables: TanStack Table 8.20                                           │
│  ├── Data: TanStack Query 5.60 + Zustand 5 + Axios 1.7                    │
│  ├── Forms: React Hook Form 7.53 + Zod 3.23                               │
│  ├── Auth: @keycloak/keycloak-js 23                                       │
│  ├── PWA: next-pwa 5.6 + Workbox 7                                        │
│  ├── Test: Jest 29 + Playwright 1.48                                      │
│  └── Lint: ESLint 9 + Prettier 3.4                                       │
│                                                                             │
│  SHARED                                                                     │
│  ├── Python: udara_shared (schemas, constants, crypto, sync, validators)  │
│  └── TypeScript: @udara/shared (types, constants, api-routes)            │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

> **Next:** [04 — Team Overview & Sprint Plan](04-team-overview-sprint-plan.md)
> **Prev:** [02 — Three-Door Interface](02-three-door-interface.md)
