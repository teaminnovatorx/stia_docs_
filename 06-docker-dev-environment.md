# 06 — Docker Development Environment

> **UDARA AI** — AMR Surveillance Platform for Sub-Saharan Africa  
> Document Version: 2.1.0 | Last Updated: 2025-01-15 | Status: Production

---

## Table of Contents

1. [Why Docker for UDARA Development](#1-why-docker-for-udara-development)
2. [3-Phase Development Workflow](#2-3-phase-development-workflow)
3. [Complete docker-compose.yml](#3-complete-docker-composeyml)
4. [Dockerfile Template](#4-dockerfile-template)
5. [Multi-Architecture Build](#5-multi-architecture-build)
6. [Development Workflow Commands](#6-development-workflow-commands)
7. [.env.example Configuration](#7-envexample-configuration)
8. [Common Issues & Solutions](#8-common-issues--solutions)
9. [CI/CD Integration](#9-cicd-integration)
10. [Appendix: Service Dependency Map](#10-appendix-service-dependency-map)

---

## 1. Why Docker for UDARA Development

### 1.1 The Challenge

UDARA runs on Raspberry Pi 4B devices deployed in rural health facilities across
sub-Saharan Africa. These devices have constrained resources (ARM64, 4GB RAM,
no GPU, SD card storage, cellular connectivity). Development must target this
exact environment, but not every developer has a Raspberry Pi on their desk.

### 1.2 What Docker Simulates (≈90% of Production)

```
┌─────────────────────────────────────────────────────────────────────┐
│              DOCKER DEVELOPMENT COVERAGE vs REAL PI                  │
├──────────────────────────┬────────────┬────────────────────────────┤
│ Aspect                   │ Docker     │ Real Raspberry Pi          │
├──────────────────────────┼────────────┼────────────────────────────┤
│ Python runtime           │ ✅ Identical│ ✅ Identical (3.11 ARM64) │
│ SQLite WAL mode          │ ✅ Identical│ ✅ Identical               │
│ Redis sessions           │ ✅ Identical│ ✅ Identical               │
│ Agent A/B/C APIs         │ ✅ Identical│ ✅ Identical               │
│ Gateway routing          │ ✅ Identical│ ✅ Identical               │
│ Sync protocol            │ ✅ Identical│ ✅ Identical               │
│ USSD handler             │ ✅ Identical│ ✅ Identical               │
│ ONNX model inference     │ ✅ Close    │ ✅ Native                  │
│ Integration tests        │ ✅ Full     │ ✅ Full                    │
│ Multi-agent orchestration│ ✅ Full     │ ✅ Full                    │
│ Memory constraints       │ ✅ Limits   │ ⚠️ Hard limits (4GB)      │
├──────────────────────────┼────────────┼────────────────────────────┤
│ ARM64 performance        │ ❌ Emulated │ ✅ Native                   │
│ Thermal throttling       │ ❌ N/A      │ ✅ Real (70°C throttle)    │
│ SD card I/O speed        │ ❌ NVMe SSD │ ⚠️ Class 10 (25 MB/s)      │
│ Cellular latency         │ ❌ Loopback │ ⚠️ 200-2000ms real         │
│ Power loss resilience    │ ❌ Clean    │ ⚠️ Unclean shutdowns       │
│ USB device I/O           │ ❌ Bind mnt │ ✅ Real USB mass storage    │
│ GPS/Bluetooth            │ ❌ Simulated│ ✅ Real hardware            │
│ Real-time clock drift    │ ❌ Host sync│ ⚠️ No RTC battery drain     │
└──────────────────────────┴────────────┴────────────────────────────┘
```

### 1.3 Development Environment Philosophy

```
┌─────────────────────────────────────────────────────────────────┐
│                  UDARA 3-PHASE DEV WORKFLOW                      │
│                                                                  │
│  PHASE 1: DOCKER           PHASE 2: REAL PI       PHASE 3: FIELD│
│  (Daily Development)       (Weekly Testing)      (Pilot Deploy) │
│  ┌────────────────┐        ┌──────────────┐     ┌──────────────┐│
│  │ Fast iteration │───────>│ Hardware val- │────>│ Real CHWs    ││
│  │ Multi-service  │        │ idation, ARM │     │ Real networks││
│  │ Full coverage  │        │ native perf  │     │ Real UX      ││
│  │ ~90% accuracy  │        │ ~97% accuracy│     │ 100% reality ││
│  └────────────────┘        └──────────────┘     └──────────────┘│
│  Developer laptop          Office Pi test bench Pilot sites in   │
│  docker compose up         (1-2 units)           Kenya, Ghana    │
│                                                                  │
│  Frequency: Daily           Frequency: Weekly     Frequency:      │
│  Duration: minutes          Duration: hours       Sprint cycles  │
└─────────────────────────────────────────────────────────────────┘
```

**Key rule:** If it works in Docker, it WILL work on the Pi for code-level
issues. But Docker cannot catch hardware-specific failures (thermal, SD card,
power loss). That's why Phase 2 exists.

---

## 2. 3-Phase Development Workflow

### 2.1 Phase 1: Local Docker (Daily)

**Who:** Every developer, every day.  
**What:** Full microservices stack running on developer laptop.  
**Goal:** Feature development, bug fixes, unit tests, integration tests.

```bash
# Morning workflow
git checkout main && git pull
docker compose pull                          # Update base images
docker compose up -d                          # Start all services
docker compose logs -f gateway               # Watch gateway logs

# Run tests against live stack
docker compose exec agent-a python -m pytest tests/ -v
docker compose exec gateway python -m pytest tests/integration/ -v

# Evening workflow
docker compose down -v                        # Clean shutdown
```

### 2.2 Phase 2: Real Pi (Weekly)

**Who:** Tech lead + 1 engineer rotation.  
**What:** Flash SD card, deploy to physical Pi, test real-world conditions.  
**Goal:** Catch hardware-specific issues, validate ARM64 native performance.

```bash
# Build ARM64 image on dev machine (cross-compile)
docker buildx build --platform linux/arm64 -t udara-agent-a:latest ./agents/a/

# Deploy to Pi via SSH
docker save udara-agent-a:latest | gzip | ssh pi@192.168.1.42 \
  "docker load && docker compose -f /opt/udara/docker-compose.yml up -d agent-a"

# Run hardware validation suite on Pi
ssh pi@192.168.1.42 "python3 /opt/udara/tests/hardware_validation.py"
```

### 2.3 Phase 3: Field Pi (Pilot)

**Who:** Field engineering team.  
**What:** Deploy to actual health facilities with real CHWs.  
**Goal:** Validate UX, network resilience, power management.

```bash
# Field deployment (from operations laptop at health facility)
ssh pi@192.168.1.42 "udara-deploy --env pilot --facility KE-NRB-FAC-017"

# Monitor remotely via VPN tunnel
udara-monitor --device udara-edge-KE-NRB-0042 --dashboard
```

---

## 3. Complete docker-compose.yml

```yaml
# docker-compose.yml — UDARA AI Development Stack
# Version: 2.1.0
# Usage: docker compose up -d
#
# Services:
#   1. gateway     — API Gateway (FastAPI + routing)
#   2. agent-a     — Clinical NLP Agent (symptom/drug extraction)
#   3. agent-b     — Resistance Analytics Agent (AMR rates)
#   4. agent-c     — Treatment Guidance Agent (recommendations)
#   5. redis       — Session store, USSD state, sync queue bridge
#   6. storage     — MinIO (S3-compatible) for media storage
#   7. sync-daemon — Edge sync daemon (simulates RPi sync process)
#   8. timescaledb — PostgreSQL + TimescaleDB (cloud simulation)

version: "3.9"

x-common-env: &common-env
  UDARA_ENV: ${UDARA_ENV:-development}
  UDARA_LOG_LEVEL: ${UDARA_LOG_LEVEL:-DEBUG}
  UDARA_LOG_FORMAT: ${UDARA_LOG_FORMAT:-json}
  UDARA_REDIS_URL: redis://redis:6379/0
  UDARA_DEVICE_ID: ${UDARA_DEVICE_ID:-udara-dev-docker-001}
  UDARA_FACILITY_CODE: ${UDARA_FACILITY_CODE:-DEV-FAC-001}
  OTLP_ENDPOINT: ${OTLP_ENDPOINT:-}

services:
  # ─── API GATEWAY ─────────────────────────────────────────────────
  gateway:
    build:
      context: .
      dockerfile: Dockerfile
      target: gateway
      args:
        PYTHON_VERSION: "3.11"
    container_name: udara-gateway
    hostname: gateway
    restart: unless-stopped
    ports:
      - "${GATEWAY_PORT:-8000}:8000"       # Main API
      - "${GATEWAY_METRICS:-9090}:9090"     # Prometheus metrics
    environment:
      <<: *common-env
      UDARA_SERVICE_NAME: gateway
      UDARA_AGENT_A_URL: http://agent-a:8001
      UDARA_AGENT_B_URL: http://agent-b:8002
      UDARA_AGENT_C_URL: http://agent-c:8003
      UDARA_TIMESCALEDB_URL: postgresql://udara:udara_dev_pass@timescaledb:5432/udara
      UDARA_STORAGE_ENDPOINT: http://storage:9000
      UDARA_STORAGE_ACCESS_KEY: ${MINIO_ROOT_USER:-udara_minio}
      UDARA_STORAGE_SECRET_KEY: ${MINIO_ROOT_PASSWORD:-udara_minio_secret}
      UDARA_STORAGE_BUCKET: ${STORAGE_BUCKET:-udara-media}
      UDARA_CORS_ORIGINS: '["http://localhost:3000","http://localhost:8000"]'
      UDARA_API_KEY: ${UDARA_API_KEY:-dev-api-key-change-me}
      UDARA_JWT_SECRET: ${UDARA_JWT_SECRET:-dev-jwt-secret-change-me}
    volumes:
      - ./gateway:/app/gateway:ro           # Code mount (read-only)
      - ./shared:/app/shared:ro              # Shared libraries
      - ./tests:/app/tests:ro                # Test files
      - gateway-data:/app/data               # Persistent data
    healthcheck:
      test: ["CMD", "python", "-c", "import urllib.request; urllib.request.urlopen('http://localhost:8000/health')"]
      interval: 15s
      timeout: 5s
      retries: 3
      start_period: 20s
    deploy:
      resources:
        limits:
          memory: 512M
          cpus: "1.0"
        reservations:
          memory: 128M
          cpus: "0.25"
    depends_on:
      redis:
        condition: service_healthy
      timescaledb:
        condition: service_healthy
      storage:
        condition: service_healthy
    networks:
      - udara-net

  # ─── AGENT A: Clinical NLP ──────────────────────────────────────
  agent-a:
    build:
      context: .
      dockerfile: Dockerfile
      target: agent-a
      args:
        PYTHON_VERSION: "3.11"
    container_name: udara-agent-a
    hostname: agent-a
    restart: unless-stopped
    ports:
      - "${AGENT_A_PORT:-8001}:8001"
      - "${AGENT_A_METRICS:-9091}:9091"
    environment:
      <<: *common-env
      UDARA_SERVICE_NAME: agent-a
      UDARA_MODEL_PATH: /app/models/agent-a
      UDARA_MODEL_VERSION: ${AGENT_A_MODEL_VERSION:-2.3.1}
      UDARA_ONNX_RUNTIME: ${UDARA_ONNX_RUNTIME:-cpu}
      UDARA_MAX_CONCURRENT_REQUESTS: ${AGENT_A_MAX_CONCURRENCY:-4}
      UDARA_TIMEOUT_SECONDS: ${AGENT_A_TIMEOUT:-30}
      UDARA_SWAHILI_MODEL_PATH: /app/models/agent-a-sw
      UDARA_STORAGE_ENDPOINT: http://storage:9000
      UDARA_STORAGE_ACCESS_KEY: ${MINIO_ROOT_USER:-udara_minio}
      UDARA_STORAGE_SECRET_KEY: ${MINIO_ROOT_PASSWORD:-udara_minio_secret}
    volumes:
      - ./agents/a:/app/agent:ro
      - ./shared:/app/shared:ro
      - ./tests:/app/tests:ro
      - model-data:/app/models:ro           # Pre-loaded models
      - agent-a-data:/app/data
    healthcheck:
      test: ["CMD", "python", "-c", "import urllib.request; urllib.request.urlopen('http://localhost:8001/health')"]
      interval: 15s
      timeout: 5s
      retries: 3
      start_period: 30s  # Slower start: loads ONNX model
    deploy:
      resources:
        limits:
          memory: 1024M                      # Needs more RAM for NLP models
          cpus: "2.0"
        reservations:
          memory: 512M
          cpus: "0.5"
    depends_on:
      redis:
        condition: service_healthy
    networks:
      - udara-net

  # ─── AGENT B: Resistance Analytics ──────────────────────────────
  agent-b:
    build:
      context: .
      dockerfile: Dockerfile
      target: agent-b
      args:
        PYTHON_VERSION: "3.11"
    container_name: udara-agent-b
    hostname: agent-b
    restart: unless-stopped
    ports:
      - "${AGENT_B_PORT:-8002}:8002"
      - "${AGENT_B_METRICS:-9092}:9092"
    environment:
      <<: *common-env
      UDARA_SERVICE_NAME: agent-b
      UDARA_TIMESCALEDB_URL: postgresql://udara:udara_dev_pass@timescaledb:5432/udara
      UDARA_CACHE_TTL_SECONDS: ${AGENT_B_CACHE_TTL:-300}
      UDARA_MIN_CASES_FOR_ESTIMATE: ${AGENT_B_MIN_CASES:-10}
      UDARA_CONFIDENCE_LEVEL: ${AGENT_B_CONFIDENCE:-0.95}
    volumes:
      - ./agents/b:/app/agent:ro
      - ./shared:/app/shared:ro
      - ./tests:/app/tests:ro
      - agent-b-data:/app/data
    healthcheck:
      test: ["CMD", "python", "-c", "import urllib.request; urllib.request.urlopen('http://localhost:8002/health')"]
      interval: 15s
      timeout: 5s
      retries: 3
      start_period: 15s
    deploy:
      resources:
        limits:
          memory: 512M
          cpus: "1.0"
        reservations:
          memory: 128M
          cpus: "0.25"
    depends_on:
      redis:
        condition: service_healthy
      timescaledb:
        condition: service_healthy
    networks:
      - udara-net

  # ─── AGENT C: Treatment Guidance ────────────────────────────────
  agent-c:
    build:
      context: .
      dockerfile: Dockerfile
      target: agent-c
      args:
        PYTHON_VERSION: "3.11"
    container_name: udara-agent-c
    hostname: agent-c
    restart: unless-stopped
    ports:
      - "${AGENT_C_PORT:-8003}:8003"
      - "${AGENT_C_METRICS:-9093}:9093"
    environment:
      <<: *common-env
      UDARA_SERVICE_NAME: agent-c
      UDARA_AGENT_B_URL: http://agent-b:8002
      UDARA_WHO_GUIDELINES_VERSION: ${WHO_GUIDELINES_VERSION:-2024}
      UDARA_MAX_ALTERNATIVES: ${AGENT_C_MAX_ALTERNATIVES:-5}
      UDARA_CONFIDENCE_THRESHOLD: ${AGENT_C_CONFIDENCE_THRESHOLD:-0.7}
    volumes:
      - ./agents/c:/app/agent:ro
      - ./shared:/app/shared:ro
      - ./tests:/app/tests:ro
      - agent-c-data:/app/data
    healthcheck:
      test: ["CMD", "python", "-c", "import urllib.request; urllib.request.urlopen('http://localhost:8003/health')"]
      interval: 15s
      timeout: 5s
      retries: 3
      start_period: 15s
    deploy:
      resources:
        limits:
          memory: 512M
          cpus: "1.0"
        reservations:
          memory: 128M
          cpus: "0.25"
    depends_on:
      redis:
        condition: service_healthy
    networks:
      - udara-net

  # ─── REDIS: Session Store ───────────────────────────────────────
  redis:
    image: redis:7.2-alpine
    container_name: udara-redis
    hostname: redis
    restart: unless-stopped
    ports:
      - "${REDIS_PORT:-6379}:6379"
    command: >
      redis-server
      --maxmemory ${REDIS_MAXMEMORY:-256mb}
      --maxmemory-policy allkeys-lru
      --appendonly yes
      --appendfsync everysec
      --save 60 1000
      --save 300 100
      --hash-max-ziplist-entries 512
      --hash-max-ziplist-value 64
      --tcp-keepalive 300
      --loglevel warning
    volumes:
      - redis-data:/data
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 10s
      timeout: 3s
      retries: 3
      start_period: 5s
    deploy:
      resources:
        limits:
          memory: 384M
          cpus: "0.5"
    networks:
      - udara-net

  # ─── MINIO: S3-Compatible Object Storage ────────────────────────
  storage:
    image: minio/minio:latest
    container_name: udara-storage
    hostname: storage
    restart: unless-stopped
    ports:
      - "${MINIO_API_PORT:-9000}:9000"
      - "${MINIO_CONSOLE_PORT:-9001}:9001"
    environment:
      MINIO_ROOT_USER: ${MINIO_ROOT_USER:-udara_minio}
      MINIO_ROOT_PASSWORD: ${MINIO_ROOT_PASSWORD:-udara_minio_secret}
      MINIO_BROWSER: "on"
    volumes:
      - storage-data:/data
    command: server /data --console-address ":9001"
    healthcheck:
      test: ["CMD", "mc", "ready", "local"]
      interval: 10s
      timeout: 5s
      retries: 3
      start_period: 10s
    deploy:
      resources:
        limits:
          memory: 512M
          cpus: "0.5"
    networks:
      - udara-net

  # ─── SYNC DAEMON: Edge Sync Simulation ──────────────────────────
  sync-daemon:
    build:
      context: .
      dockerfile: Dockerfile
      target: sync-daemon
      args:
        PYTHON_VERSION: "3.11"
    container_name: udara-sync-daemon
    hostname: sync-daemon
    restart: unless-stopped
    ports:
      - "${SYNC_DAEMON_METRICS:-9094}:9094"
    environment:
      <<: *common-env
      UDARA_SERVICE_NAME: sync-daemon
      UDARA_CLOUD_URL: http://gateway:8000
      UDARA_EDGE_DB: /app/data/edge.db
      UDARA_SYNC_INTERVAL: ${SYNC_INTERVAL:-60}       # Faster in dev (production: 300)
      UDARA_BATCH_SIZE: ${SYNC_BATCH_SIZE:-10}
      UDARA_SYNC_KEY: ${UDARA_SYNC_KEY:-a1b2c3d4e5f6a7b8c9d0e1f2a3b4c5d6e7f8a9b0c1d2e3f4a5b6c7d8e9f0a1b2}
      UDARA_USB_MOUNT: /app/data/usb_sim
      UDARA_USB_FALLBACK_HOURS: ${USB_FALLBACK_HOURS:-0.1}  # 6 min in dev (production: 48)
      UDARA_API_KEY: ${UDARA_API_KEY:-dev-api-key-change-me}
    volumes:
      - ./sync:/app/sync:ro
      - ./shared:/app/shared:ro
      - sync-data:/app/data
    healthcheck:
      test: ["CMD", "python", "-c", "
        import json, urllib.request;
        r = json.loads(urllib.request.urlopen('http://localhost:9094/metrics').read().decode());
        assert 'udara_sync_queue_size' in str(r)
      "]
      interval: 30s
      timeout: 5s
      retries: 3
      start_period: 15s
    deploy:
      resources:
        limits:
          memory: 256M
          cpus: "0.5"
    depends_on:
      gateway:
        condition: service_healthy
    networks:
      - udara-net

  # ─── TIMESCALEDB: Cloud Database Simulation ─────────────────────
  timescaledb:
    image: timescale/timescaledb:latest-pg16
    container_name: udara-timescaledb
    hostname: timescaledb
    restart: unless-stopped
    ports:
      - "${TIMESCALEDB_PORT:-5432}:5432"
    environment:
      POSTGRES_USER: udara
      POSTGRES_PASSWORD: udara_dev_pass
      POSTGRES_DB: udara
    volumes:
      - timescaledb-data:/var/lib/postgresql/data
      - ./scripts/init-db.sql:/docker-entrypoint-initdb.d/001_init.sql:ro
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U udara -d udara"]
      interval: 10s
      timeout: 3s
      retries: 5
      start_period: 30s
    deploy:
      resources:
        limits:
          memory: 1024M
          cpus: "1.0"
    networks:
      - udara-net

# ─── VOLUMES ────────────────────────────────────────────────────────
volumes:
  gateway-data:
    name: udara-gateway-data
  agent-a-data:
    name: udara-agent-a-data
  agent-b-data:
    name: udara-agent-b-data
  agent-c-data:
    name: udara-agent-c-data
  model-data:
    name: udara-model-data
  redis-data:
    name: udara-redis-data
  storage-data:
    name: udara-storage-data
  sync-data:
    name: udara-sync-data
  timescaledb-data:
    name: udara-timescaledb-data

# ─── NETWORKS ───────────────────────────────────────────────────────
networks:
  udara-net:
    name: udara-dev-network
    driver: bridge
    ipam:
      config:
        - subnet: 172.28.0.0/16
```

---

## 4. Dockerfile Template

```dockerfile
# Dockerfile — UDARA AI Multi-Service Build
# Supports: linux/amd64, linux/arm64
# Build: docker buildx build --platform linux/amd64,linux/arm64 -t udara .
#
# Usage:
#   docker build --target gateway -t udara-gateway .
#   docker build --target agent-a -t udara-agent-a .
#   docker build --target agent-b -t udara-agent-b .
#   docker build --target agent-c -t udara-agent-c .
#   docker build --target sync-daemon -t udara-sync-daemon .

# ─── ARGUMENTS ─────────────────────────────────────────────────────
ARG PYTHON_VERSION=3.11

# ─── BASE: Shared Python Runtime ──────────────────────────────────
FROM python:${PYTHON_VERSION}-slim AS base

# Install system dependencies required by all services
RUN apt-get update && apt-get install -y --no-install-recommends \
    # Networking
    curl \
    wget \
    ca-certificates \
    # Database
    libsqlite3-dev \
    libpq-dev \
    # Image/audio processing
    libgl1-mesa-glx \
    libglib2.0-0 \
    libsndfile1 \
    # Utilities
    tini \
    su-exec \
    && rm -rf /var/lib/apt/lists/*

# Create non-root user for security
RUN groupadd -r udara && useradd -r -g udara -d /app -s /sbin/nologin udara

# Set working directory
WORKDIR /app

# Install Python dependencies (shared layer, cached across services)
COPY requirements/base.txt ./requirements-base.txt
COPY requirements/common.txt ./requirements-common.txt

RUN pip install --no-cache-dir -r requirements-base.txt && \
    pip install --no-cache-dir -r requirements-common.txt

# Copy shared library code
COPY shared/ ./shared/

# Set environment
ENV PYTHONPATH=/app \
    PYTHONUNBUFFERED=1 \
    PYTHONDONTWRITEBYTECODE=1 \
    PIP_NO_CACHE_DIR=1 \
    PIP_DISABLE_PIP_VERSION_CHECK=1

# ─── GATEWAY ───────────────────────────────────────────────────────
FROM base AS gateway

COPY requirements/gateway.txt ./requirements-gateway.txt
RUN pip install --no-cache-dir -r requirements-gateway.txt

COPY gateway/ ./gateway/
COPY contracts/ ./contracts/

EXPOSE 8000 9090

HEALTHCHECK --interval=15s --timeout=5s --retries=3 \
    CMD curl -f http://localhost:8000/health || exit 1

CMD ["tini", "--", "python", "-m", "gateway.main"]

# ─── AGENT A: Clinical NLP ─────────────────────────────────────────
FROM base AS agent-a

# Install ONNX Runtime (CPU only for RPi compatibility)
COPY requirements/agent-a.txt ./requirements-agent-a.txt
RUN pip install --no-cache-dir -r requirements-agent-a.txt

# Pre-download spaCy model for English
RUN python -m spacy download en_core_web_sm 2>/dev/null || true

COPY agents/a/ ./agent/
COPY shared/ ./shared/

# Model files copied at runtime via volume mount
# Expected structure: /app/models/agent-a/model.onnx
ENV UDARA_MODEL_PATH=/app/models/agent-a

EXPOSE 8001 9091

HEALTHCHECK --interval=15s --timeout=5s --retries=3 \
    CMD curl -f http://localhost:8001/health || exit 1

CMD ["tini", "--", "python", "-m", "agent.main"]

# ─── AGENT B: Resistance Analytics ─────────────────────────────────
FROM base AS agent-b

COPY requirements/agent-b.txt ./requirements-agent-b.txt
RUN pip install --no-cache-dir -r requirements-agent-b.txt

COPY agents/b/ ./agent/
COPY shared/ ./shared/

EXPOSE 8002 9092

HEALTHCHECK --interval=15s --timeout=5s --retries=3 \
    CMD curl -f http://localhost:8002/health || exit 1

CMD ["tini", "--", "python", "-m", "agent.main"]

# ─── AGENT C: Treatment Guidance ──────────────────────────────────
FROM base AS agent-c

COPY requirements/agent-c.txt ./requirements-agent-c.txt
RUN pip install --no-cache-dir -r requirements-agent-c.txt

COPY agents/c/ ./agent/
COPY shared/ ./shared/

EXPOSE 8003 9093

HEALTHCHECK --interval=15s --timeout=5s --retries=3 \
    CMD curl -f http://localhost:8003/health || exit 1

CMD ["tini", "--", "python", "-m", "agent.main"]

# ─── SYNC DAEMON ───────────────────────────────────────────────────
FROM base AS sync-daemon

COPY requirements/sync.txt ./requirements-sync.txt
RUN pip install --no-cache-dir -r requirements-sync.txt

COPY sync/ ./sync/
COPY shared/ ./shared/

EXPOSE 9094

CMD ["tini", "--", "python", "-m", "sync.engine"]

# ─── DEVELOPMENT: All services in one image for debugging ──────────
FROM base AS development

# Install all service dependencies
COPY requirements/requirements.txt ./requirements-all.txt
RUN pip install --no-cache-dir -r requirements-all.txt

# Install development tools
RUN pip install --no-cache-dir \
    pytest \
    pytest-asyncio \
    pytest-cov \
    pytest-timeout \
    httpie \
    ipython \
    black \
    ruff \
    mypy

# Copy everything
COPY . .

EXPOSE 8000 8001 8002 8003 9090 9091 9092 9093 9094

CMD ["tini", "--", "python", "-m", "dev.server"]
```

### 4.1 Requirements Files Structure

```
requirements/
├── base.txt              # Core: fastapi, uvicorn, pydantic, httpx
├── common.txt            # Shared: structlog, prometheus-client, etc.
├── gateway.txt           # Gateway-specific: jinja2, aiohttp, s3fs
├── agent-a.txt           # Agent A: onnxruntime, spacy, pillow, librosa
├── agent-b.txt           # Agent B: psycopg2-binary, numpy, scipy
├── agent-c.txt           # Agent C: numpy, pydantic (already in base)
├── sync.txt              # Sync: cryptography, gzip (stdlib), aioredis
└── requirements.txt      # All combined (for development target)
```

**requirements/base.txt:**
```txt
# Core web framework
fastapi==0.115.6
uvicorn[standard]==0.34.0
pydantic==2.10.4
pydantic-settings==2.7.1

# Async HTTP client
httpx==0.28.1

# Serialization
msgspec==0.18.6
orjson==3.10.15

# Configuration
python-dotenv==1.0.1

# Logging
structlog==24.4.0
```

**requirements/agent-a.txt:**
```txt
# NLP & ML
onnxruntime==1.20.1
spacy==3.8.4

# Image processing
Pillow==11.1.0

# Audio processing
librosa==0.10.2.post1
soundfile==0.12.1

# Geohash
pygeohash==1.2.1

# Redis (for caching)
redis==5.2.1
```

---

## 5. Multi-Architecture Build

### 5.1 Buildx Setup

```bash
# Install Docker Buildx (included with modern Docker Desktop)
docker buildx version

# Create a multi-arch builder instance
docker buildx create --name udara-builder --driver docker-container --use
docker buildx inspect --bootstrap

# Verify supported platforms
docker buildx inspect --bootstrap | grep Platforms
# Platforms: linux/amd64, linux/arm64, linux/arm/v7
```

### 5.2 Build Commands

```bash
# ── Build for current architecture (fast, for daily dev)
docker compose build

# ── Build specific service
docker compose build agent-a
docker compose build --no-cache gateway  # Force rebuild

# ── Cross-compile for ARM64 (for Pi deployment)
docker buildx build \
  --platform linux/arm64 \
  --target agent-a \
  -t udara-agent-a:arm64-v2.3.1 \
  --load \
  .

# ── Build AND push multi-arch manifest to registry
docker buildx build \
  --platform linux/amd64,linux/arm64 \
  --target agent-a \
  -t ghcr.io/udara-health/agent-a:v2.3.1 \
  -t ghcr.io/udara-health/agent-a:latest \
  --push \
  .

# ── Build all services for both architectures
for service in gateway agent-a agent-b agent-c sync-daemon; do
  echo "=== Building $service for amd64 + arm64 ==="
  docker buildx build \
    --platform linux/amd64,linux/arm64 \
    --target $service \
    -t ghcr.io/udara-health/$service:${VERSION:-latest} \
    --push \
    .
done

# ── Build with BuildKit cache (much faster incremental builds)
docker buildx build \
  --platform linux/arm64 \
  --target agent-a \
  --cache-from type=gha \
  --cache-to type=gha,mode=max \
  -t udara-agent-a:arm64 \
  .
```

### 5.3 Architecture-Specific Optimizations

```dockerfile
# In Dockerfile, after FROM base:
ARG TARGETARCH

# ARM64-specific optimizations for Raspberry Pi
RUN if [ "$TARGETARCH" = "arm64" ]; then \
      # Use ARM-optimized BLAS for numpy/scipy
      pip install --no-cache-dir numpy==1.26.4; \
    else \
      pip install --no-cache-dir numpy==1.26.4; \
    fi

# ONNX Runtime: use CPU-optimized build for ARM
RUN if [ "$TARGETARCH" = "arm64" ]; then \
      pip install --no-cache-dir onnxruntime==1.20.1; \
    else \
      pip install --no-cache-dir onnxruntime==1.20.1; \
    fi
```

### 5.4 Image Size Comparison

```
Service        amd64 Size    arm64 Size    Notes
─────────────  ───────────   ───────────   ──────────────────────
base           145 MB        132 MB        Slim Python 3.11
gateway        198 MB        185 MB        + FastAPI, PostgreSQL
agent-a        892 MB        845 MB        + ONNX, spaCy, librosa
agent-b        267 MB        254 MB        + NumPy, SciPy
agent-c        234 MB        221 MB        + NumPy
sync-daemon    203 MB        191 MB        + cryptography
─────────────  ───────────   ───────────   ──────────────────────
TOTAL          ~1,994 MB     ~1,828 MB     (all services combined)
```

> **Tip:** Use `docker images --format "{{.Repository}}:{{.Tag}} {{.Size}}"` to check sizes.

---

## 6. Development Workflow Commands

### 6.1 Quick Reference

```bash
# ── STARTUP ───────────────────────────────────────────────────────
docker compose up -d                          # Start all services (detached)
docker compose up -d gateway redis            # Start specific services
docker compose up --build -d                  # Rebuild and start
docker compose up --force-recreate -d         # Force recreate all containers

# ── MONITORING ────────────────────────────────────────────────────
docker compose ps                             # List all services with status
docker compose logs -f                        # Follow all logs
docker compose logs -f gateway                 # Follow specific service
docker compose logs --since 5m gateway         # Last 5 minutes
docker compose logs --tail 100 agent-a         # Last 100 lines

# ── TESTING ───────────────────────────────────────────────────────
docker compose exec gateway python -m pytest tests/ -v
docker compose exec agent-a python -m pytest tests/ -v --cov=agent
docker compose exec agent-b python -m pytest tests/ --timeout=30
docker compose run --rm gateway python -m pytest tests/integration/ -v

# ── EXEC / DEBUG ──────────────────────────────────────────────────
docker compose exec gateway bash              # Shell in running container
docker compose exec agent-a python -c "import agent; print(agent.__version__)"
docker compose exec redis redis-cli MONITOR    # Redis monitor
docker compose exec timescaledb psql -U udara -c "\dt"  # List tables

# ── DATABASE ──────────────────────────────────────────────────────
docker compose exec timescaledb psql -U udara -d udara
docker compose exec timescaledb pg_dump -U udara udara > backup.sql
docker compose exec -T timescaledb psql -U udara -d udara < restore.sql

# ── CLEANUP ───────────────────────────────────────────────────────
docker compose down                            # Stop and remove containers
docker compose down -v                         # Also remove volumes (DESTROYS DATA)
docker compose down --rmi all                  # Also remove images
docker system prune -f                         # Clean dangling resources

# ── NETWORK DEBUG ─────────────────────────────────────────────────
docker compose exec gateway curl -s http://agent-a:8001/health | jq
docker compose exec gateway curl -s http://agent-b:8002/health | jq
docker compose exec gateway curl -s http://redis:6379/ | head
docker compose exec gateway python -c "
import httpx
r = httpx.get('http://agent-a:8001/health')
print(r.status_code, r.json())
"
```

### 6.2 End-to-End Test Workflow

```bash
#!/bin/bash
# e2e-test.sh — Run full end-to-end test against Docker stack

set -euo pipefail

echo "=== Step 1: Ensure stack is running ==="
docker compose up -d --wait
echo "All services healthy: $(docker compose ps --format json | jq -r '.[] | .Name' | wc -l) services"

echo "=== Step 2: Run unit tests ==="
for service in gateway agent-a agent-b agent-c; do
  echo "--- Unit tests: $service ---"
  docker compose exec $service python -m pytest tests/unit/ -v --tb=short -q
done

echo "=== Step 3: Run integration tests ==="
docker compose exec gateway python -m pytest tests/integration/ -v --tb=short

echo "=== Step 4: Test clinical workflow ==="
# Ingest a text case through the gateway
RESPONSE=$(docker compose exec gateway curl -s -X POST http://localhost:8000/ingest/text \
  -H "Content-Type: application/json" \
  -H "X-API-Key: dev-api-key-change-me" \
  -d '{
    "text": "Patient is a 28 year old female presenting with fever for 3 days, dysuria, and flank pain. She was given ciprofloxacin 500mg twice daily.",
    "language": "en",
    "source": "clinical_note",
    "facility_code": "DEV-FAC-001"
  }')

echo "Gateway response: $RESPONSE" | python -m json.tool

CASE_ID=$(echo "$RESPONSE" | python -c "import sys,json; print(json.load(sys.stdin)['case_id'])")
echo "Created case: $CASE_ID"

echo "=== Step 5: Test resistance lookup ==="
docker compose exec gateway curl -s \
  "http://localhost:8000/resistance/ciprofloxacin/DEV-DIST-001?age_group=25-34" \
  -H "X-API-Key: dev-api-key-change-me" | python -m json.tool

echo "=== Step 6: Test treatment guidance ==="
docker compose exec gateway curl -s -X POST http://localhost:8000/guidance \
  -H "Content-Type: application/json" \
  -H "X-API-Key: dev-api-key-change-me" \
  -d "{
    \"case_id\": \"$CASE_ID\",
    \"district\": \"DEV-DIST-001\",
    \"suggested_drug\": \"ciprofloxacin\"
  }" | python -m json.tool

echo "=== Step 7: Verify sync queue ==="
docker compose exec sync-daemon python -c "
import sqlite3
conn = sqlite3.connect('/app/data/edge.db')
count = conn.execute('SELECT COUNT(*) FROM sync_queue').fetchone()[0]
print(f'Records in sync queue: {count}')
"

echo "=== All E2E tests passed ==="
```

### 6.3 Performance Testing

```bash
# Install load testing tool
pip install locust

# Run load test against gateway
locust -f tests/load/locustfile.py \
  --host=http://localhost:8000 \
  --users=50 \
  --spawn-rate=5 \
  --run-time=5m \
  --headless

# Memory profiling on agent-a (largest service)
docker stats udara-agent-a --no-stream --format "table {{.Name}}\t{{.CPUPerc}}\t{{.MemUsage}}\t{{.MemPerc}}"

# Profile a single request
docker compose exec agent-a python -c "
import cProfile
import pstats
import io
from agent.main import app

pr = cProfile.Profile()
pr.enable()
# ... make request ...
pr.disable()
s = io.StringIO()
ps = pstats.Stats(pr, stream=s).sort_stats('cumulative')
ps.print_stats(20)
print(s.getvalue())
"
```

---

## 7. .env.example Configuration

```bash
# .env.example — UDARA AI Docker Environment Configuration
# Copy to .env and customize: cp .env.example .env
#
# NEVER commit .env to version control!
# Add to .gitignore: echo ".env" >> .gitignore

# ═══════════════════════════════════════════════════════════════════
# GENERAL
# ═══════════════════════════════════════════════════════════════════
UDARA_ENV=development                           # development|staging|production
UDARA_LOG_LEVEL=DEBUG                           # DEBUG|INFO|WARNING|ERROR
UDARA_LOG_FORMAT=json                           # json|text
UDARA_DEVICE_ID=udara-dev-docker-001
UDARA_FACILITY_CODE=DEV-FAC-001
UDARA_API_KEY=dev-api-key-change-me             # CHANGE IN PRODUCTION!
UDARA_JWT_SECRET=dev-jwt-secret-change-me       # CHANGE IN PRODUCTION!
VERSION=2.3.1

# ═══════════════════════════════════════════════════════════════════
# SERVICE PORTS (host-side; change if ports conflict)
# ═══════════════════════════════════════════════════════════════════
GATEWAY_PORT=8000
GATEWAY_METRICS=9090
AGENT_A_PORT=8001
AGENT_A_METRICS=9091
AGENT_B_PORT=8002
AGENT_B_METRICS=9092
AGENT_C_PORT=8003
AGENT_C_METRICS=9093
SYNC_DAEMON_METRICS=9094

# ═══════════════════════════════════════════════════════════════════
# REDIS
# ═══════════════════════════════════════════════════════════════════
REDIS_PORT=6379
REDIS_MAXMEMORY=256mb                           # Match RPi available RAM

# ═══════════════════════════════════════════════════════════════════
# MINIO (S3-compatible Storage)
# ═══════════════════════════════════════════════════════════════════
MINIO_API_PORT=9000
MINIO_CONSOLE_PORT=9001
MINIO_ROOT_USER=udara_minio
MINIO_ROOT_PASSWORD=udara_minio_secret
STORAGE_BUCKET=udara-media

# ═══════════════════════════════════════════════════════════════════
# TIMESCALEDB
# ═══════════════════════════════════════════════════════════════════
TIMESCALEDB_PORT=5432
# Note: credentials set in docker-compose.yml environment section

# ═══════════════════════════════════════════════════════════════════
# AGENT A — Clinical NLP
# ═══════════════════════════════════════════════════════════════════
AGENT_A_MODEL_VERSION=2.3.1
AGENT_A_MAX_CONCURRENCY=4                      # Parallel inference requests
AGENT_A_TIMEOUT=30                              # Seconds per request
UDARA_ONNX_RUNTIME=cpu                         # cpu|cuda (always cpu on RPi)

# ═══════════════════════════════════════════════════════════════════
# AGENT B — Resistance Analytics
# ═══════════════════════════════════════════════════════════════════
AGENT_B_CACHE_TTL=300                          # Seconds: resistance cache TTL
AGENT_B_MIN_CASES=10                           # Minimum cases for rate estimate
AGENT_B_CONFIDENCE=0.95                        # Wilson score CI level

# ═══════════════════════════════════════════════════════════════════
# AGENT C — Treatment Guidance
# ═══════════════════════════════════════════════════════════════════
AGENT_C_MAX_ALTERNATIVES=5
AGENT_C_CONFIDENCE_THRESHOLD=0.7
WHO_GUIDELINES_VERSION=2024

# ═══════════════════════════════════════════════════════════════════
# SYNC DAEMON
# ═══════════════════════════════════════════════════════════════════
SYNC_INTERVAL=60                               # Seconds (dev: 60, prod: 300)
SYNC_BATCH_SIZE=10                             # Records per batch (dev: 10, prod: 50)
UDARA_SYNC_KEY=a1b2c3d4e5f6a7b8c9d0e1f2a3b4c5d6e7f8a9b0c1d2e3f4a5b6c7d8e9f0a1b2
USB_FALLBACK_HOURS=0.1                         # Hours (dev: 0.1, prod: 48)

# ═══════════════════════════════════════════════════════════════════
# TELEMETRY (Optional)
# ═══════════════════════════════════════════════════════════════════
OTLP_ENDPOINT=                                  # OpenTelemetry endpoint (empty=disabled)
# OTLP_ENDPOINT=http://otel-collector:4318    # Example with local collector
```

---

## 8. Common Issues & Solutions

| # | Problem | Cause | Solution |
|---|---------|-------|----------|
| 1 | `gateway` exits immediately | Port 8000 already in use | `lsof -i :8000` then kill or change `GATEWAY_PORT` |
| 2 | `agent-a` OOM killed (exit code 137) | spaCy model + ONNX > memory limit | Increase `memory: 1024M` → `1536M` in compose |
| 3 | `timescaledb` healthcheck never passes | Slow startup on low-RAM machine | Increase `start_period: 30s` → `60s`; check `docker stats` |
| 4 | Redis `OOM command not allowed` | Data exceeds `maxmemory` | Increase `REDIS_MAXMEMORY` or change policy to `volatile-lru` |
| 5 | `agent-a` takes 60+ seconds to start | Large ONNX model loading | Expected first time; subsequent starts use Docker layer cache |
| 6 | Cross-container DNS resolution fails | Custom network not created | Run `docker compose down && docker compose up -d` to recreate |
| 7 | `Permission denied` on volume mount | Container runs as non-root but volume owned by root | Add `user: "${UID}:${GID}"` to service, or `chown` volume dir |
| 8 | ARM64 build fails on M1 Mac | QEMU emulation issues | Use native ARM: `docker buildx build --platform linux/arm64` |
| 9 | `pip install` very slow in build | No pip cache across builds | Use `--mount=type=cache,target=/root/.cache/pip` in Dockerfile |
| 10 | MinIO healthcheck fails | `mc` not in minimal image | Use `curl -f http://localhost:9000/minio/health/live` instead |
| 11 | Tests fail with connection refused | Service not ready when tests run | Add `depends_on:` with `condition: service_healthy` |
| 12 | `sqlite3.OperationalError: disk I/O error` | WAl file on Docker Desktop with small disk | Use named volume instead of bind mount for SQLite |
| 13 | Stale code after editing files | Docker caches old code | Use `:ro` bind mounts (already in compose.yml); run `docker compose restart` |
| 14 | Multi-arch push fails | Not logged into registry | `docker login ghcr.io`; check `~/.docker/config.json` |
| 15 | `ImportError: No module named 'shared'` | PYTHONPATH not set | Verify `ENV PYTHONPATH=/app` in Dockerfile base stage |

### 8.1 Diagnostic Commands

```bash
# Check all container statuses and health
docker compose ps -a

# Resource usage for all services
docker stats --no-stream --format "table {{.Name}}\t{{.CPUPerc}}\t{{.MemUsage}}\t{{.NetIO}}\t{{.BlockIO}}"

# Check container logs for errors
docker compose logs --since 1h 2>&1 | grep -i "error\|warning\|traceback" | tail -50

# Test internal network connectivity
docker compose exec gateway python -c "
import httpx
services = {
    'agent-a': 8001, 'agent-b': 8002, 'agent-c': 8003,
    'redis': 6379, 'storage': 9000
}
for name, port in services.items():
    try:
        r = httpx.get(f'http://{name}:{port}/', timeout=2)
        print(f'  ✅ {name}:{port} → {r.status_code}')
    except Exception as e:
        print(f'  ❌ {name}:{port} → {e}')
"

# Check disk usage of volumes
docker system df -v

# Inspect a container's environment
docker compose exec gateway env | sort | grep UDARA
```

---

## 9. CI/CD Integration

### 9.1 GitHub Actions Build Pipeline

```yaml
# .github/workflows/docker-build.yml
name: Docker Build & Test

on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main]

env:
  REGISTRY: ghcr.io
  IMAGE_PREFIX: udara-health

jobs:
  build-and-test:
    runs-on: ubuntu-latest
    strategy:
      matrix:
        platform: [linux/amd64, linux/arm64]
        service: [gateway, agent-a, agent-b, agent-c, sync-daemon]

    steps:
      - uses: actions/checkout@v4

      - name: Set up Docker Buildx
        uses: docker/setup-buildx-action@v3

      - name: Log in to Container Registry
        uses: docker/login-action@v3
        with:
          registry: ${{ env.REGISTRY }}
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}

      - name: Build ${{ matrix.service }} (${{ matrix.platform }})
        uses: docker/build-push-action@v5
        with:
          context: .
          file: Dockerfile
          target: ${{ matrix.service }}
          platforms: ${{ matrix.platform }}
          push: ${{ github.ref == 'refs/heads/main' }}
          tags: |
            ${{ env.REGISTRY }}/${{ env.IMAGE_PREFIX }}/${{ matrix.service }}:${{ github.sha }}
            ${{ env.REGISTRY }}/${{ env.IMAGE_PREFIX }}/${{ matrix.service }}:latest
          cache-from: type=gha
          cache-to: type=gha,mode=max

      - name: Run tests in container
        if: matrix.platform == 'linux/amd64'
        run: |
          docker build --target ${{ matrix.service }} -t test-${{ matrix.service }} .
          docker run --rm test-${{ matrix.service }} python -m pytest tests/ -v --tb=short

  integration-test:
    needs: build-and-test
    runs-on: ubuntu-latest

    steps:
      - uses: actions/checkout@v4

      - name: Start full stack
        run: docker compose up -d --wait

      - name: Run integration tests
        run: docker compose exec gateway python -m pytest tests/integration/ -v

      - name: Run E2E test
        run: bash e2e-test.sh

      - name: Collect logs on failure
        if: failure()
        run: docker compose logs --since 10m > failure-logs.txt

      - name: Cleanup
        if: always()
        run: docker compose down -v
```

---

## 10. Appendix: Service Dependency Map

```
                    ┌─────────────────────────────────────────┐
                    │         SERVICE DEPENDENCY MAP          │
                    └─────────────────────────────────────────┘

                        ┌──────────────┐
                        │   CLIENT     │
                        │  (CHW app,   │
                        │   USSD, API) │
                        └──────┬───────┘
                               │
                               ▼
                        ┌──────────────┐
                        │   GATEWAY    │
                        │  :8000       │
                        │  (FastAPI)   │
                        └──┬───┬───┬───┘
                           │   │   │
              ┌────────────┘   │   └────────────┐
              ▼                ▼                ▼
       ┌────────────┐  ┌────────────┐  ┌────────────┐
       │  AGENT A   │  │  AGENT B   │  │  AGENT C   │
       │  :8001     │  │  :8002     │  │  :8003     │
       │ (Clinical  │  │(Resistance)│  │(Guidance)  │
       │  NLP)      │  │            │  │            │
       └──┬─────────┘  └──┬─────────┘  └──┬─────────┘
          │               │               │
          │         ┌─────┘               │
          │         ▼                     │
          │  ┌──────────────┐             │
          │  │ TIMESCALEDB  │             │
          │  │  :5432       │             │
          │  │ (PostgreSQL) │             │
          │  └──────────────┘             │
          │                              │
          │  ┌──────────────┐             │
          │  │   STORAGE    │             │
          │  │  :9000       │             │
          │  │  (MinIO)     │             │
          │  └──────────────┘             │
          │                              │
     ┌────┴──────────────────────────────┴────┐
     │              REDIS :6379               │
     │   (Sessions, USSD state, cache, pub/sub)│
     └──────────────────┬────────────────────┘
                        │
                 ┌──────┴──────┐
                 │ SYNC DAEMON │
                 │  :9094      │
                 │ (metrics)   │
                 └─────────────┘

    LEGEND:
    ──────
    ────  HTTP dependency (service calls)
    ─ ──  Data store connection
    ╌╌╌╌  Event/queue based
```

### 10.1 Startup Order

The healthchecks in `docker-compose.yml` enforce this startup order:

```
Time    Event
────    ─────
0s      redis, timescaledb, storage start in parallel
5s      redis healthy → services depending on redis can start
10s     storage healthy → gateway can start
15s     timescaledb healthy → agent-b, gateway can start
20s     gateway healthy → agent-a, agent-c, sync-daemon can start
30s     agent-a healthy (after model loading) → system fully operational
```
