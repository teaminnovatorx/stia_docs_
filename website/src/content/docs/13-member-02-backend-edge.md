# Member 02 — Backend + Edge Engineer Deep Dive

> **Document ID:** 13-MEMBER-02-BACKEND  
> **Version:** 1.0.0  
> **Last Updated:** 2026-05-27  
> **Status:** Active  
> **Classification:** Internal — Engineering Reference

---

## Table of Contents

1. [Role Overview](#1-role-overview)
2. [Directory Structure](#2-directory-structure)
3. [FastAPI Gateway](#3-fastapi-gateway)
4. [Edge Runtime on RPi 5](#4-edge-runtime-on-rpi-5)
5. [USSD Session Manager](#5-ussd-session-manager)
6. [Sync Protocol Deep Dive](#6-sync-protocol-deep-dive)
7. [Database Schemas](#7-database-schemas)
8. [Africa's Talking Integration](#8-africas-talking-integration)
9. [Complete Code Examples](#9-complete-code-examples)
10. [Week-by-Week Tasks](#10-week-by-week-tasks)
11. [Testing Strategy](#11-testing-strategy)
12. [Monitoring & Observability](#12-monitoring--observability)
13. [Appendices](#13-appendices)

---

## 1. Role Overview

### 1.1 What This Engineer Owns

The Backend + Edge Engineer is the **infrastructure backbone** of UDARA AI. Every request from every Door flows through systems built by this engineer. They own the API gateway, the edge runtime on RPi 5, the synchronization protocol between edge and cloud, all database schemas, and the USSD session management system that enables Door 1 for feature phone users.

```
┌─────────────────────────────────────────────────────────────────────┐
│           MEMBER 02 — SPHERE OF OWNERSHIP                           │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  INFRASTRUCTURE LAYER:                                              │
│  ├── FastAPI Gateway (REST + WebSocket)                            │
│  ├── Authentication (JWT + Keycloak SSO)                           │
│  ├── Rate limiting & middleware stack                              │
│  ├── CORS & security headers                                       │
│  └── API versioning & documentation (OpenAPI 3.1)                   │
│                                                                     │
│  EDGE LAYER (RPi 5):                                               │
│  ├── Edge runtime orchestrator                                     │
│  ├── Model lifecycle management (load/unload/swap)                  │
│  ├── SQLite WAL mode for concurrent persistence                    │
│  ├── Hardware monitoring (CPU temp, RAM, storage)                   │
│  ├── Graceful degradation strategy                                 │
│  └── Balena Fleet management integration                           │
│                                                                     │
│  SYNC PROTOCOL:                                                     │
│  ├── Delta JSON generation (SQLite → Cloud)                        │
│  ├── CRDT conflict resolution (Yjs)                               │
│  ├── gzip compression for bandwidth efficiency                     │
│  ├── Retry with exponential backoff                                │
│  ├── USB fallback (encrypted tarball)                              │
│  └── Sync state machine (6 states)                                 │
│                                                                     │
│  DATA LAYER:                                                        │
│  ├── Edge: SQLite (4 tables, WAL mode)                            │
│  ├── Cloud: PostgreSQL 16 + TimescaleDB + PostGIS                  │
│  ├── Knowledge Graph: Neo4j 5 (drug-pathogen-resistance)          │
│  └── Redis 7 (session cache, rate limits)                         │
│                                                                     │
│  USSD SYSTEM:                                                       │
│  ├── Redis-backed session manager (120s TTL)                      │
│  ├── State machine for multi-step menu navigation                  │
│  ├── Africa's Talking API integration                             │
│  └── Input validation for feature phones                           │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

### 1.2 Design Principles

| Principle | Description | How We Apply It |
|-----------|-------------|-----------------|
| **API-First** | Every capability exposed via well-documented REST API | OpenAPI 3.1 auto-generated from FastAPI code |
| **Offline-First** | Edge works fully offline, syncs when connected | SQLite at edge, delta sync, USB fallback |
| **Graceful Degradation** | System degrades predictably under constraints | Memory monitoring drops models; USSD has minimal data |
| **Eventual Consistency** | Edge and cloud converge over time | Delta sync + CRDT for conflict resolution |
| **Defense in Depth** | Multiple security layers | JWT + Keycloak + AES-256 + Presidio PII scrubbing |
| **Observable** | Every component emits metrics and logs | Prometheus + Grafana + structured logging |

### 1.3 Key Performance Targets

| Metric | Target | Why It Matters |
|--------|--------|---------------|
| API response time (p50) | < 200ms | Responsive UI and bot interactions |
| API response time (p99) | < 1s | No user should wait > 1s for simple queries |
| Sync completion time | < 30s (1000 records) | Fits within typical connectivity windows |
| Sync bandwidth | < 500KB (100 records) | Works on 2G/3G connections |
| USSD session response | < 3s (telco requirement) | USSD sessions timeout after ~20s |
| RPi memory usage | < 6.5GB | Headroom for ML models and OS |
| SQLite write latency | < 10ms | Fast local persistence |
| API availability | > 99.5% (cloud), 100% (edge) | Critical health infrastructure |

---

## 2. Directory Structure

```
backend/
├── api/
│   ├── main.py                        # FastAPI app entry point
│   ├── routes/
│   │   ├── cases.py                   # Case CRUD endpoints
│   │   ├── resistance.py              # Resistance query endpoints
│   │   ├── reports.py                 # Report generation
│   │   ├── sync.py                    # Sync protocol endpoints
│   │   ├── ussd.py                    # USSD callback endpoints
│   │   ├── ocr.py                     # OCR processing endpoints
│   │   ├── asr.py                     # ASR processing endpoints
│   │   ├── guidance.py                # RAG guidance endpoints
│   │   └── health.py                  # Health checks & metrics
│   ├── middleware/
│   │   ├── auth.py                    # JWT validation middleware
│   │   ├── rate_limit.py             # Rate limiting (Redis-backed)
│   │   ├── logging.py                 # Request/response logging
│   │   ├── cors.py                    # CORS configuration
│   │   └── error_handler.py           # Global error handling
│   └── dependencies.py                # Shared FastAPI dependencies
├── edge/
│   ├── runtime.py                     # Edge runtime orchestrator
│   ├── local_db.py                   # SQLite operations layer
│   ├── model_manager.py              # Load/unload models on RPi
│   ├── usb_sync.py                   # USB fallback sync mechanism
│   ├── health_monitor.py             # RPi hardware health monitoring
│   └── graceful_degradation.py       # Memory-based model dropping
├── sync/
│   ├── delta_generator.py             # Generate delta JSON from SQLite
│   ├── conflict_resolver.py           # CRDT conflict resolution (Yjs)
│   ├── compressor.py                 # gzip compression for sync payloads
│   ├── orchestrator.py               # Sync state machine & orchestrator
│   ├── retry.py                      # Retry with exponential backoff
│   └── usb_transport.py             # Encrypted USB tarball transport
├── db/
│   ├── postgres/
│   │   ├── connection.py             # PostgreSQL connection pool
│   │   ├── cases.py                  # Cases table CRUD
│   │   ├── resistance.py             # Resistance data queries
│   │   ├── users.py                  # Users & CHW profiles
│   │   ├── facilities.py             # Health facility registry
│   │   ├── spatial.py               # PostGIS spatial functions
│   │   └── migrations/              # Alembic migration scripts
│   ├── timescale/
│   │   ├── hypertable_setup.py      # TimescaleDB hypertable config
│   │   ├── resistance_timeseries.py  # Time-series resistance queries
│   │   └── continuous_aggregates.py  # Pre-aggregated resistance stats
│   ├── neo4j/
│   │   ├── connection.py             # Neo4j driver configuration
│   │   ├── knowledge_graph.py       # Drug-pathogen-resistance graph
│   │   ├── graph_queries.py         # Cypher query library
│   │   └── seed_data.py             # Initial graph seeding
│   └── redis/
│       ├── connection.py             # Redis connection pool
│       ├── session_store.py          # USSD session management
│       ├── rate_limit_store.py       # Rate limit counters
│       └── cache_store.py            # General-purpose cache
├── ussd/
│   ├── session_manager.py            # Redis-backed USSD sessions
│   ├── menu_router.py               # Menu flow routing engine
│   ├── flows/
│   │   ├── report_case.py           # USSD case reporting flow
│   │   ├── check_resistance.py      # USSD resistance query flow
│   │   ├── registration.py          # CHW registration flow
│   │   └── help.py                  # Help & information flow
│   ├── input_validator.py           # USSD keypad input validation
│   └── africa_talking.py            # Africa's Talking API client
├── tests/
│   ├── test_sync.py                  # Sync protocol tests
│   ├── test_ussd.py                  # USSD session & flow tests
│   ├── test_api.py                   # API endpoint tests
│   ├── test_edge.py                  # Edge runtime tests
│   ├── test_db.py                    # Database tests
│   └── conftest.py                   # Shared test fixtures
├── Dockerfile                        # Backend Docker image
├── Dockerfile.edge                   # Edge Docker image (RPi 5)
├── docker-compose.yml                # Full stack orchestration
├── requirements.txt                  # Python dependencies
└── pyproject.toml                    # Project metadata
```

### 2.1 File Ownership Map

```
FILES OWNED vs INTERFACES:
═══════════════════════════════════════════════════════════════

EXCLUSIVELY OWNED BY MEMBER 02:
  backend/api/         — All API routes, middleware, dependencies
  backend/edge/        — Edge runtime, model manager, health monitoring
  backend/sync/        — Sync protocol, delta generator, CRDT
  backend/db/          — All database schemas and connections
  backend/ussd/        — USSD sessions, flows, Africa's Talking
  backend/tests/       — Backend-specific tests

INTERFACES (defined by Member 02, consumed by others):
  ├── API endpoints     → Member 03 (Frontend) calls these
  ├── Sync endpoint     → Edge devices POST deltas here
  ├── USSD callback     → Africa's Talking POSTs here
  └── ML model manager  → Member 01's models loaded via this

CALLED FROM (dependencies):
  ├── ml/agent/case_builder.py → persists via API
  ├── ml/models/resistance/   → reads/writes via API
  └── bots/bot_core.py       → calls API for case queries
```

---

## 3. FastAPI Gateway

### 3.1 Application Architecture

```
FASTAPI GATEWAY ARCHITECTURE:
═══════════════════════════════════════════════════════════════

                    ┌──────────────────────┐
                    │    Client Request    │
                    │  (Door 1/2/3/Edge)  │
                    └──────────┬───────────┘
                               │
                    ┌──────────▼───────────┐
                    │   Nginx / Caddy      │
                    │   (Reverse Proxy)    │
                    │   SSL Termination    │
                    │   Rate Limit (L4)    │
                    └──────────┬───────────┘
                               │
              ┌────────────────▼────────────────┐
              │       FASTAPI APPLICATION        │
              │                                  │
              │  ┌────────────────────────────┐  │
              │  │      MIDDLEWARE STACK      │  │
              │  │  1. CORS                   │  │
              │  │  2. Request Logging        │  │
              │  │  3. Request ID (UUID)      │  │
              │  │  4. JWT Validation         │  │
              │  │  5. Rate Limiting (L7)     │  │
              │  │  6. PII Scrubbing (audit)  │  │
              │  └────────────┬───────────────┘  │
              │               │                  │
              │  ┌────────────▼───────────────┐  │
              │  │      ROUTER                │  │
              │  │  /api/v1/cases             │  │
              │  │  /api/v1/resistance        │  │
              │  │  /api/v1/sync              │  │
              │  │  /api/v1/ussd              │  │
              │  │  /api/v1/ocr               │  │
              │  │  /api/v1/asr               │  │
              │  │  /api/v1/guidance          │  │
              │  │  /health                   │  │
              │  └────────────┬───────────────┘  │
              │               │                  │
              │  ┌────────────▼───────────────┐  │
              │  │   DEPENDENCY INJECTION     │  │
              │  │  • get_db_session()       │  │
              │  │  • get_redis()            │  │
              │  │  • get_current_user()     │  │
              │  │  • get_neo4j_driver()     │  │
              │  └────────────┬───────────────┘  │
              │               │                  │
              │  ┌────────────▼───────────────┐  │
              │  │   RESPONSE FORMATTER       │  │
              │  │  • JSON envelope           │  │
              │  │  • Error format            │  │
              │  │  • Compression (gzip)     │  │
              │  └────────────────────────────┘  │
              └──────────────────────────────────┘
                               │
              ┌────────────────┼────────────────┐
              │                │                │
       ┌──────▼──────┐ ┌──────▼──────┐ ┌──────▼──────┐
       │ PostgreSQL  │ │   Redis 7   │ │   Neo4j 5   │
       │ + Timescale │ │ (Sessions,  │ │ (Knowledge  │
       │ + PostGIS   │ │  Cache,    │ │  Graph)     │
       │             │ │  Limits)   │ │             │
       └─────────────┘ └────────────┘ └─────────────┘
```

### 3.2 FastAPI Main Application

```python
# backend/api/main.py
"""
UDARA AI — FastAPI Gateway.

Central API entry point serving all Three Doors and edge sync.
"""

from __future__ import annotations

import logging
import time
import uuid
from contextlib import asynccontextmanager
from typing import Optional

from fastapi import FastAPI, Request, Response, status
from fastapi.middleware.cors import CORSMiddleware
from fastapi.middleware.gzip import GZipMiddleware
from fastapi.responses import JSONResponse

from api.middleware.auth import AuthMiddleware
from api.middleware.logging import RequestLoggingMiddleware
from api.middleware.rate_limit import RateLimitMiddleware
from api.routes import (
    cases, resistance, reports, sync,
    ussd, ocr, asr, guidance, health,
)


# ─── Logging Setup ──────────────────────────────────────────────────────

logging.basicConfig(
    level=logging.INFO,
    format="%(asctime)s | %(levelname)s | %(name)s | %(message)s",
)
logger = logging.getLogger("udara.api")


# ─── App Lifespan ────────────────────────────────────────────────────────

@asynccontextmanager
async def lifespan(app: FastAPI):
    """Application lifespan: startup and shutdown events."""
    logger.info("🚀 UDARA AI API starting up...")

    # Startup
    await startup_db_connections()
    await startup_cache_connections()
    logger.info("✅ All connections established")

    yield

    # Shutdown
    await shutdown_db_connections()
    await shutdown_cache_connections()
    logger.info("👋 UDARA AI API shut down")


async def startup_db_connections():
    """Initialize database connection pools."""
    from db.postgres.connection import init_postgres_pool
    from db.neo4j.connection import init_neo4j_driver
    from db.redis.connection import init_redis_pool

    await init_postgres_pool()
    init_neo4j_driver()
    await init_redis_pool()


async def shutdown_db_connections():
    """Close database connection pools."""
    from db.postgres.connection import close_postgres_pool
    from db.neo4j.connection import close_neo4j_driver
    from db.redis.connection import close_redis_pool

    await close_postgres_pool()
    close_neo4j_driver()
    await close_redis_pool()


# ─── App Creation ───────────────────────────────────────────────────────

app = FastAPI(
    title="UDARA AI API",
    description="""
    Antimicrobial Resistance Surveillance Platform for Sub-Saharan Africa.

    ## Three-Door Interface
    - **Door 1**: USSD (feature phones) via Africa's Talking
    - **Door 2**: WhatsApp + Telegram bots
    - **Door 3**: Web dashboard (Next.js)

    ## Features
    - AMR case reporting and management
    - Resistance probability queries (Bayesian)
    - Drug label OCR validation
    - Voice transcription (12+ African languages)
    - RAG-powered clinical guidance
    - Spatial resistance maps
    - Edge-to-cloud synchronization
    """,
    version="1.0.0",
    lifespan=lifespan,
    docs_url="/docs",
    redoc_url="/redoc",
    openapi_url="/openapi.json",
)


# ─── Middleware Stack ───────────────────────────────────────────────────

# 1. CORS — Allow all Three Doors to access the API
app.add_middleware(
    CORSMiddleware,
    allow_origins=[
        "http://localhost:3000",      # Next.js dev
        "https://udara.ai",           # Production
        "https://app.udara.ai",       # Web app
    ],
    allow_credentials=True,
    allow_methods=["*"],
    allow_headers=["*"],
)

# 2. GZip compression for responses > 1KB
app.add_middleware(GZipMiddleware, minimum_size=1000)

# 3. Request ID injection
@app.middleware("http")
async def request_id_middleware(request: Request, call_next):
    """Inject unique request ID for tracing."""
    request_id = request.headers.get(
        "X-Request-ID", str(uuid.uuid4())
    )
    request.state.request_id = request_id

    start_time = time.time()
    response = await call_next(request)
    duration_ms = (time.time() - start_time) * 1000

    response.headers["X-Request-ID"] = request_id
    response.headers["X-Response-Time"] = f"{duration_ms:.1f}ms"
    return response

# 4. Custom middleware classes
app.add_middleware(RequestLoggingMiddleware)
app.add_middleware(AuthMiddleware)
app.add_middleware(RateLimitMiddleware)


# ─── Global Exception Handler ───────────────────────────────────────────

@app.exception_handler(Exception)
async def global_exception_handler(request: Request, exc: Exception):
    """Catch-all exception handler for consistent error responses."""
    logger.error(
        f"Unhandled exception: {exc}",
        exc_info=True,
        extra={"request_id": getattr(request.state, 'request_id', None)},
    )
    return JSONResponse(
        status_code=status.HTTP_500_INTERNAL_SERVER_ERROR,
        content={
            "error": "Internal Server Error",
            "detail": str(exc) if app.debug else "An error occurred",
            "request_id": getattr(request.state, 'request_id', None),
        },
    )


# ─── Router Registration ────────────────────────────────────────────────

app.include_router(health.router, prefix="/health", tags=["Health"])
app.include_router(cases.router, prefix="/api/v1/cases", tags=["Cases"])
app.include_router(
    resistance.router, prefix="/api/v1/resistance", tags=["Resistance"]
)
app.include_router(
    reports.router, prefix="/api/v1/reports", tags=["Reports"]
)
app.include_router(sync.router, prefix="/api/v1/sync", tags=["Sync"])
app.include_router(ussd.router, prefix="/api/v1/ussd", tags=["USSD"])
app.include_router(ocr.router, prefix="/api/v1/ocr", tags=["OCR"])
app.include_router(asr.router, prefix="/api/v1/asr", tags=["ASR"])
app.include_router(
    guidance.router, prefix="/api/v1/guidance", tags=["Guidance"]
)


# ─── Root Endpoint ──────────────────────────────────────────────────────

@app.get("/", tags=["Root"])
async def root():
    """API root — service information."""
    return {
        "service": "UDARA AI",
        "version": "1.0.0",
        "status": "operational",
        "doors": {
            "ussd": "active",
            "telegram": "active",
            "whatsapp": "active",
            "web": "active",
        },
        "docs": "/docs",
    }
```

### 3.3 Shared Dependencies

```python
# backend/api/dependencies.py
"""
FastAPI dependency injection for shared resources.

Provides database sessions, Redis clients, and auth context.
"""

from __future__ import annotations

from typing import AsyncGenerator, Optional

from fastapi import Depends, HTTPException, Request, status


async def get_db_session():
    """Get a PostgreSQL database session."""
    from db.postgres.connection import get_pool
    async with get_pool().acquire() as conn:
        yield conn


def get_redis():
    """Get a Redis connection."""
    from db.redis.connection import get_redis_client
    return get_redis_client()


def get_neo4j():
    """Get a Neo4j driver session."""
    from db.neo4j.connection import get_driver
    return get_driver()


async def get_current_user(request: Request) -> dict:
    """
    Get the current authenticated user from JWT.

    Raises HTTPException if not authenticated.
    """
    user = getattr(request.state, "user", None)
    if user is None:
        raise HTTPException(
            status_code=status.HTTP_401_UNAUTHORIZED,
            detail="Not authenticated",
        )
    return user


async def get_optional_user(request: Request) -> Optional[dict]:
    """Get current user or None if not authenticated."""
    return getattr(request.state, "user", None)


def get_edge_node_id(request: Request) -> Optional[str]:
    """Extract edge node ID from request header."""
    return request.headers.get("X-Edge-Node-ID")
```

### 3.4 Case CRUD Endpoints

```python
# backend/api/routes/cases.py
"""
Case CRUD endpoints.

POST   /api/v1/cases          — Create a new AMR case
GET    /api/v1/cases           — List cases (paginated, filterable)
GET    /api/v1/cases/{id}      — Get a specific case
PUT    /api/v1/cases/{id}      — Update a case
DELETE /api/v1/cases/{id}      — Delete a case (soft delete)
GET    /api/v1/cases/search    — Search cases by text
"""

from __future__ import annotations

import logging
from typing import Optional

from fastapi import (
    APIRouter, Depends, HTTPException, Query, Request,
    Response, status,
)
from pydantic import BaseModel, Field

logger = logging.getLogger(__name__)
router = APIRouter()


# ─── Request/Response Models ────────────────────────────────────────────

class CaseCreateRequest(BaseModel):
    """Request body for creating a new AMR case."""
    source: str = Field(..., description="ussd, telegram, whatsapp, web")
    language: str = Field(default="en")
    patient_age_group: Optional[str] = None
    patient_age_value: Optional[int] = None
    specimen: Optional[str] = None
    body_site: Optional[str] = None
    pathogen: Optional[str] = None
    resistance_pattern: Optional[str] = None
    test_result: Optional[str] = None
    drugs_prescribed: list[str] = Field(default_factory=list)
    dosages: list[str] = Field(default_factory=list)
    clinical_outcome: Optional[str] = None
    district: str = Field(..., description="District of reporting facility")
    facility: Optional[str] = None
    clinical_notes: Optional[str] = None


class CaseResponse(BaseModel):
    """Response for a case."""
    case_id: str
    source: str
    language: str
    timestamp: str
    patient_age_group: Optional[str]
    patient_age_value: Optional[int]
    specimen: Optional[str]
    body_site: Optional[str]
    pathogen: Optional[str]
    resistance_pattern: Optional[str]
    test_result: Optional[str]
    drugs_prescribed: list[str]
    dosages: list[str]
    clinical_outcome: Optional[str]
    district: str
    facility: Optional[str]
    confidence: str
    resistance_flag: bool
    created_at: str


class PaginatedCaseResponse(BaseModel):
    """Paginated list of cases."""
    cases: list[CaseResponse]
    total: int
    page: int
    page_size: int
    has_more: bool


# ─── Endpoints ──────────────────────────────────────────────────────────

@router.post(
    "",
    response_model=CaseResponse,
    status_code=status.HTTP_201_CREATED,
    summary="Create AMR case",
    description="Submit a new AMR case report from any Door.",
)
async def create_case(
    request: Request,
    body: CaseCreateRequest,
    db=Depends(get_db_session),
):
    """Create a new AMR case."""
    import uuid
    from datetime import datetime, timezone

    case_id = str(uuid.uuid4())
    timestamp = datetime.now(timezone.utc).isoformat()

    # Check for resistance flag
    resistance_flag = body.resistance_pattern in (
        "ESBL", "MRSA", "CRE", "XDR", "MDR-TB", "carbapenemase"
    )

    # Persist to PostgreSQL
    await db.execute(
        """
        INSERT INTO cases (
            case_id, source, language, timestamp,
            patient_age_group, patient_age_value,
            specimen, body_site, pathogen,
            resistance_pattern, test_result,
            drugs_prescribed, dosages, clinical_outcome,
            district, facility, clinical_notes,
            confidence, resistance_flag, created_at
        ) VALUES (
            $1, $2, $3, $4, $5, $6, $7, $8, $9,
            $10, $11, $12, $13, $14, $15, $16, $17, $18,
            $19, $20, $21
        )
        """,
        case_id, body.source, body.language, timestamp,
        body.patient_age_group, body.patient_age_value,
        body.specimen, body.body_site, body.pathogen,
        body.resistance_pattern, body.test_result,
        body.drugs_prescribed, body.dosages, body.clinical_outcome,
        body.district, body.facility, body.clinical_notes,
        "medium", resistance_flag, timestamp,
    )

    # Queue for resistance engine update (async)
    # (In production: publish to message queue)

    logger.info(
        f"Case created: {case_id} "
        f"(source={body.source}, district={body.district}, "
        f"resistance_flag={resistance_flag})"
    )

    return CaseResponse(
        case_id=case_id,
        source=body.source,
        language=body.language,
        timestamp=timestamp,
        patient_age_group=body.patient_age_group,
        patient_age_value=body.patient_age_value,
        specimen=body.specimen,
        body_site=body.body_site,
        pathogen=body.pathogen,
        resistance_pattern=body.resistance_pattern,
        test_result=body.test_result,
        drugs_prescribed=body.drugs_prescribed,
        dosages=body.dosages,
        clinical_outcome=body.clinical_outcome,
        district=body.district,
        facility=body.facility,
        confidence="medium",
        resistance_flag=resistance_flag,
        created_at=timestamp,
    )


@router.get(
    "/{case_id}",
    response_model=CaseResponse,
    summary="Get case by ID",
)
async def get_case(
    case_id: str,
    db=Depends(get_db_session),
):
    """Retrieve a specific case by its ID."""
    row = await db.fetchrow(
        "SELECT * FROM cases WHERE case_id = $1 AND deleted_at IS NULL",
        case_id,
    )
    if row is None:
        raise HTTPException(status_code=404, detail="Case not found")
    return CaseResponse(**dict(row))


@router.get(
    "",
    response_model=PaginatedCaseResponse,
    summary="List cases",
)
async def list_cases(
    page: int = Query(1, ge=1),
    page_size: int = Query(20, ge=1, le=100),
    district: Optional[str] = Query(None),
    pathogen: Optional[str] = Query(None),
    drug: Optional[str] = Query(None),
    resistance_flag: Optional[bool] = Query(None),
    db=Depends(get_db_session),
):
    """List cases with pagination and filters."""
    offset = (page - 1) * page_size

    # Build query with filters
    conditions = ["deleted_at IS NULL"]
    params = []
    param_idx = 1

    if district:
        conditions.append(f"district = ${param_idx}")
        params.append(district)
        param_idx += 1
    if pathogen:
        conditions.append(f"pathogen = ${param_idx}")
        params.append(pathogen)
        param_idx += 1
    if resistance_flag is not None:
        conditions.append(f"resistance_flag = ${param_idx}")
        params.append(resistance_flag)
        param_idx += 1

    where = " AND ".join(conditions)

    # Count total
    count_row = await db.fetchrow(
        f"SELECT COUNT(*) as total FROM cases WHERE {where}",
        *params,
    )
    total = count_row["total"]

    # Fetch page
    rows = await db.fetch(
        f"SELECT * FROM cases WHERE {where} "
        f"ORDER BY created_at DESC LIMIT ${param_idx} OFFSET ${param_idx + 1}",
        *params, page_size, offset,
    )

    return PaginatedCaseResponse(
        cases=[CaseResponse(**dict(r)) for r in rows],
        total=total,
        page=page,
        page_size=page_size,
        has_more=(offset + page_size) < total,
    )
```

---

## 4. Edge Runtime on RPi 5

### 4.1 Edge Runtime Architecture

```
EDGE RUNTIME ON RASPBERRY PI 5 (8GB):
═══════════════════════════════════════════════════════════════

┌──────────────────────────────────────────────────────────────┐
│                     RASPBERRY PI 5                           │
│                   (Balena OS, Docker)                        │
│                                                              │
│  ┌────────────────────────────────────────────────────────┐  │
│  │              EDGE RUNTIME (Python)                      │  │
│  │                                                        │  │
│  │  ┌──────────────┐    ┌───────────────────────────┐    │  │
│  │  │  FastAPI      │    │   MODEL MANAGER          │    │  │
│  │  │  (Local API)  │    │   ┌─────┐ ┌─────┐        │    │  │
│  │  │  :8080        │    │   │Llama│ │AfroBERT│       │    │  │
│  │  │               │    │   │INT4  │ │INT8   │       │    │  │
│  │  │  • /cases     │    │   └─────┘ └─────┘        │    │  │
│  │  │  • /ocr       │    │   ┌─────┐ ┌─────┐        │    │  │
│  │  │  • /asr       │    │   │OCR   │ │ASR   │       │    │  │
│  │  │  • /health    │    │   │INT8  │ │INT8  │       │    │  │
│  │  │  • /sync      │    │   └─────┘ └─────┘        │    │  │
│  │  └──────┬───────┘    └──────────┬────────────────┘    │  │
│  │         │                       │                      │  │
│  │  ┌──────▼───────────────────────▼────────────────┐    │  │
│  │  │           LOCAL DATABASE (SQLite)              │    │  │
│  │  │           WAL Mode enabled                     │    │  │
│  │  │           /data/udara_local.db                 │    │  │
│  │  │                                                │    │  │
│  │  │   Tables: cases, resistance_data,              │    │  │
│  │  │           sync_meta, local_users               │    │  │
│  │  └──────────────────────┬─────────────────────────┘    │  │
│  │                         │                               │  │
│  │  ┌──────────────────────▼─────────────────────────┐    │  │
│  │  │           SYNC ORCHESTRATOR                     │    │  │
│  │  │   Delta JSON → CRDT → gzip → Upload            │    │  │
│  │  │   Retry with exponential backoff                │    │  │
│  │  │   USB fallback (encrypted tarball)              │    │  │
│  │  └────────────────────────────────────────────────┘    │  │
│  │                                                        │  │
│  │  ┌────────────────────────────────────────────────┐    │  │
│  │  │         HEALTH MONITOR                         │    │  │
│  │  │   CPU temp, RAM usage, disk space, uptime      │    │  │
│  │  │   → POST /health every 60s                      │    │  │
│  │  └────────────────────────────────────────────────┘    │  │
│  └────────────────────────────────────────────────────────┘  │
│                                                              │
│  STORAGE:                                                    │
│  ├── /data/udara_local.db         (~50MB typical)           │
│  ├── /data/models/                 (~4.4GB total)            │
│  └── /data/sync/                   (~10MB delta queue)      │
│                                                              │
└──────────────────────────────────────────────────────────────┘
```

### 4.2 Edge Runtime Implementation

```python
# backend/edge/runtime.py
"""
Edge Runtime Orchestrator for RPi 5.

Manages the complete edge stack: API server, model lifecycle,
local database, sync protocol, and hardware health monitoring.
"""

from __future__ import annotations

import logging
import os
import signal
import sys
import time
from dataclasses import dataclass, field
from pathlib import Path
from typing import Optional

import uvicorn

logger = logging.getLogger(__name__)


@dataclass
class EdgeConfig:
    """Edge runtime configuration."""
    data_dir: str = "/data"
    model_dir: str = "/data/models"
    db_path: str = "/data/udara_local.db"
    api_host: str = "0.0.0.0"
    api_port: int = 8080
    sync_interval_seconds: int = 300     # 5 minutes
    health_check_interval: int = 60      # 1 minute
    max_memory_percent: float = 80.0     # Graceful degradation threshold
    node_id: str = "edge-001"
    cloud_sync_url: str = "https://api.udara.ai/api/v1/sync"


class EdgeRuntime:
    """
    Main edge runtime orchestrator.

    Responsibilities:
    1. Start local FastAPI server
    2. Initialize SQLite database
    3. Manage model lifecycle
    4. Run periodic sync with cloud
    5. Monitor hardware health
    6. Handle graceful shutdown
    """

    def __init__(self, config: Optional[EdgeConfig] = None):
        self.config = config or EdgeConfig()
        self._running = False
        self._model_manager = None
        self._health_monitor = None
        self._sync_orchestrator = None

    def initialize(self):
        """Initialize all edge subsystems."""
        logger.info("=" * 60)
        logger.info("  UDARA AI — Edge Runtime v1.0.0")
        logger.info(f"  Node: {self.config.node_id}")
        logger.info(f"  Data: {self.config.data_dir}")
        logger.info("=" * 60)

        # Ensure directories exist
        for path in [self.config.data_dir, self.config.model_dir]:
            Path(path).mkdir(parents=True, exist_ok=True)

        # Initialize SQLite
        self._init_database()

        # Initialize model manager
        self._init_model_manager()

        # Initialize health monitor
        self._init_health_monitor()

        # Initialize sync orchestrator
        self._init_sync()

        logger.info("✅ Edge runtime initialized")

    def _init_database(self):
        """Initialize SQLite with WAL mode."""
        from edge.local_db import LocalDatabase

        self.db = LocalDatabase(self.config.db_path)
        self.db.initialize_schema()
        logger.info(
            f"SQLite initialized at {self.config.db_path} (WAL mode)"
        )

    def _init_model_manager(self):
        """Initialize model lifecycle manager."""
        from edge.model_manager import ModelManager

        self.model_manager = ModelManager(
            model_dir=self.config.model_dir,
            max_memory_percent=self.config.max_memory_percent,
        )
        logger.info("Model manager initialized")

    def _init_health_monitor(self):
        """Initialize hardware health monitor."""
        from edge.health_monitor import HealthMonitor

        self.health_monitor = HealthMonitor(
            interval=self.config.health_check_interval,
        )
        logger.info("Health monitor initialized")

    def _init_sync(self):
        """Initialize sync orchestrator."""
        from sync.orchestrator import SyncOrchestrator

        self.sync_orchestrator = SyncOrchestrator(
            db=self.db,
            cloud_url=self.config.cloud_sync_url,
            node_id=self.config.node_id,
            interval=self.config.sync_interval_seconds,
        )
        logger.info("Sync orchestrator initialized")

    def start(self):
        """Start the edge runtime."""
        self._running = True
        self.initialize()

        # Start background services
        self.health_monitor.start()
        self.sync_orchestrator.start()

        # Start FastAPI server (blocking)
        logger.info(
            f"Starting API server on {self.config.api_host}:"
            f"{self.config.api_port}"
        )

        uvicorn.run(
            "api.main:app",
            host=self.config.api_host,
            port=self.config.api_port,
            log_level="info",
            workers=1,  # Single worker for RPi
        )

    def stop(self):
        """Gracefully stop the edge runtime."""
        logger.info("Shutting down edge runtime...")
        self._running = False

        if self.sync_orchestrator:
            self.sync_orchestrator.stop()
        if self.health_monitor:
            self.health_monitor.stop()
        if self.model_manager:
            self.model_manager.unload_all()

        logger.info("Edge runtime stopped")


# ─── Signal Handlers ────────────────────────────────────────────────────

runtime: Optional[EdgeRuntime] = None


def signal_handler(sig, frame):
    """Handle SIGTERM/SIGINT for graceful shutdown."""
    logger.info(f"Received signal {sig}, shutting down...")
    if runtime:
        runtime.stop()
    sys.exit(0)


def main():
    """Entry point for edge runtime."""
    global runtime

    signal.signal(signal.SIGTERM, signal_handler)
    signal.signal(signal.SIGINT, signal_handler)

    runtime = EdgeRuntime()
    runtime.start()


if __name__ == "__main__":
    main()
```

### 4.3 Model Manager

```python
# backend/edge/model_manager.py
"""
Model lifecycle management on RPi 5.

Handles lazy loading, memory monitoring, and graceful degradation
of AI/ML models to stay within RPi 5's 8GB RAM constraint.
"""

from __future__ import annotations

import logging
import os
import psutil
from dataclasses import dataclass
from enum import Enum
from typing import Optional

logger = logging.getLogger(__name__)


class ModelPriority(Enum):
    """Priority for model loading/unloading decisions."""
    CRITICAL = 1    # Always loaded (Llama, AfroBERT, ChromaDB)
    HIGH = 2        # Load when available (SQLite, FastAPI)
    MEDIUM = 3      # Lazy load on demand (OCR, ASR)
    LOW = 4         # Only load if ample memory (Bayesian)


@dataclass
class ModelInfo:
    """Metadata about a loaded model."""
    name: str
    priority: ModelPriority
    estimated_ram_mb: int
    loaded: bool = False
    last_used: float = 0.0
    load_fn: Optional[callable] = None
    unload_fn: Optional[callable] = None


class ModelManager:
    """
    Manages AI/ML model lifecycle on RPi 5.

    Features:
    - Lazy loading: models loaded on first request
    - LRU unloading: least recently used models unloaded first
    - Memory monitoring: tracks RAM, triggers degradation
    - Graceful degradation: drops low-priority models under pressure
    """

    def __init__(
        self,
        model_dir: str = "/data/models",
        max_memory_percent: float = 80.0,
    ):
        self.model_dir = model_dir
        self.max_memory_percent = max_memory_percent
        self._models: dict[str, ModelInfo] = {}
        self._load_count = 0

    def register_model(
        self,
        name: str,
        priority: ModelPriority,
        estimated_ram_mb: int,
        load_fn: callable,
        unload_fn: Optional[callable] = None,
    ):
        """Register a model for lifecycle management."""
        self._models[name] = ModelInfo(
            name=name,
            priority=priority,
            estimated_ram_mb=estimated_ram_mb,
            load_fn=load_fn,
            unload_fn=unload_fn,
        )
        logger.info(
            f"Registered model: {name} "
            f"(priority={priority.name}, ~{estimated_ram_mb}MB)"
        )

    def get_model(self, name: str):
        """
        Get a model, loading it if necessary.

        Returns the model object (whatever load_fn returns).
        Raises RuntimeError if model cannot be loaded.
        """
        import time

        if name not in self._models:
            raise KeyError(f"Model '{name}' not registered")

        model_info = self._models[name]

        if model_info.loaded:
            model_info.last_used = time.time()
            return model_info  # Already loaded, return ref

        # Check memory before loading
        if not self._has_sufficient_memory(model_info.estimated_ram_mb):
            self._free_memory(model_info.estimated_ram_mb)

        # Load the model
        logger.info(f"Loading model: {name} (~{model_info.estimated_ram_mb}MB)")
        try:
            model_info.load_fn()
            model_info.loaded = True
            model_info.last_used = time.time()
            self._load_count += 1
            logger.info(f"Model {name} loaded successfully")
            return model_info
        except Exception as e:
            logger.error(f"Failed to load model {name}: {e}")
            raise RuntimeError(f"Cannot load model {name}: {e}")

    def unload_model(self, name: str):
        """Unload a model to free memory."""
        if name not in self._models:
            return

        model_info = self._models[name]
        if not model_info.loaded:
            return

        if model_info.unload_fn:
            logger.info(f"Unloading model: {name}")
            model_info.unload_fn()
        model_info.loaded = False
        logger.info(f"Model {name} unloaded, ~{model_info.estimated_ram_mb}MB freed")

    def unload_all(self):
        """Unload all models (for shutdown)."""
        for name in list(self._models.keys()):
            self.unload_model(name)

    def _get_available_ram_mb(self) -> float:
        """Get available RAM in MB."""
        return psutil.virtual_memory().available / (1024 * 1024)

    def _get_used_memory_percent(self) -> float:
        """Get current memory usage percentage."""
        return psutil.virtual_memory().percent

    def _has_sufficient_memory(self, needed_mb: int) -> bool:
        """Check if enough RAM is available."""
        available = self._get_available_ram_mb()
        headroom_mb = 512  # Keep 512MB headroom
        return (available - headroom_mb) > needed_mb

    def _free_memory(self, needed_mb: int):
        """
        Free memory by unloading models in priority order.

        Strategy: unload LOW → MEDIUM priority models first.
        Never unload CRITICAL priority models.
        """
        current_available = self._get_available_ram_mb()
        deficit = (needed_mb + 512) - current_available

        if deficit <= 0:
            return  # No need to free

        # Sort models by priority (LOW first) then by last_used (oldest first)
        unloadable = sorted(
            [
                m for m in self._models.values()
                if m.loaded and m.priority != ModelPriority.CRITICAL
            ],
            key=lambda m: (-m.priority.value, m.last_used),
        )

        freed = 0
        for model in unloadable:
            if freed >= deficit:
                break
            self.unload_model(model.name)
            freed += model.estimated_ram_mb
            logger.warning(
                f"Graceful degradation: unloaded {model.name} "
                f"(freed ~{model.estimated_ram_mb}MB, "
                f"still need {max(0, deficit - freed)}MB)"
            )

        if freed < deficit:
            logger.error(
                f"Cannot free enough memory! "
                f"Need {deficit}MB, freed {freed}MB"
            )

    def get_status(self) -> dict:
        """Get current model manager status."""
        return {
            "total_models": len(self._models),
            "loaded_models": sum(1 for m in self._models.values() if m.loaded),
            "load_count": self._load_count,
            "ram_available_mb": round(self._get_available_ram_mb(), 1),
            "ram_used_percent": round(self._get_used_memory_percent(), 1),
            "models": {
                name: {
                    "loaded": m.loaded,
                    "priority": m.priority.name,
                    "estimated_ram_mb": m.estimated_ram_mb,
                }
                for name, m in self._models.items()
            },
        }
```

### 4.4 Local Database (SQLite)

```python
# backend/edge/local_db.py
"""
SQLite local database for edge persistence.

Runs on RPi 5 with WAL mode for concurrent read/write support.
"""

from __future__ import annotations

import logging
import sqlite3
from contextlib import contextmanager
from pathlib import Path
from typing import Generator, Optional

logger = logging.getLogger(__name__)


class LocalDatabase:
    """
    SQLite database for edge persistence.

    Features:
    - WAL mode for concurrent read/write
    - Foreign key enforcement
    - Auto-vacuum to manage disk space
    - 4 core tables: cases, resistance_data, sync_meta, local_users
    """

    def __init__(self, db_path: str):
        self.db_path = db_path
        self._conn: Optional[sqlite3.Connection] = None

    def initialize_schema(self):
        """Create tables if they don't exist."""
        Path(self.db_path).parent.mkdir(parents=True, exist_ok=True)

        with self._get_connection() as conn:
            conn.executescript("""
                -- Enable WAL mode for concurrent read/write
                PRAGMA journal_mode = WAL;
                PRAGMA foreign_keys = ON;
                PRAGMA auto_vacuum = INCREMENTAL;
                PRAGMA busy_timeout = 5000;
                PRAGMA synchronous = NORMAL;

                -- Cases table (local AMR case records)
                CREATE TABLE IF NOT EXISTS cases (
                    case_id TEXT PRIMARY KEY,
                    source TEXT NOT NULL,
                    language TEXT DEFAULT 'en',
                    timestamp TEXT NOT NULL,
                    patient_age_group TEXT,
                    patient_age_value INTEGER,
                    specimen TEXT,
                    body_site TEXT,
                    pathogen TEXT,
                    resistance_pattern TEXT,
                    test_result TEXT,
                    drugs_prescribed TEXT,       -- JSON array
                    dosages TEXT,                -- JSON array
                    clinical_outcome TEXT,
                    district TEXT NOT NULL,
                    facility TEXT,
                    clinical_notes TEXT,
                    confidence TEXT DEFAULT 'medium',
                    resistance_flag INTEGER DEFAULT 0,
                    raw_text TEXT,
                    synced INTEGER DEFAULT 0,     -- 0=unsynced, 1=synced
                    sync_timestamp TEXT,
                    created_at TEXT DEFAULT (datetime('now')),
                    updated_at TEXT DEFAULT (datetime('now'))
                );

                -- Resistance data (local resistance observations)
                CREATE TABLE IF NOT EXISTS resistance_data (
                    id INTEGER PRIMARY KEY AUTOINCREMENT,
                    district TEXT NOT NULL,
                    drug TEXT NOT NULL,
                    pathogen TEXT NOT NULL,
                    total_tested INTEGER NOT NULL DEFAULT 0,
                    resistant_count INTEGER NOT NULL DEFAULT 0,
                    year INTEGER NOT NULL,
                    quarter INTEGER NOT NULL,
                    last_updated TEXT DEFAULT (datetime('now')),
                    synced INTEGER DEFAULT 0,
                    UNIQUE(district, drug, pathogen, year, quarter)
                );

                -- Sync metadata
                CREATE TABLE IF NOT EXISTS sync_meta (
                    key TEXT PRIMARY KEY,
                    value TEXT NOT NULL
                );

                -- Local users (CHW profiles cached at edge)
                CREATE TABLE IF NOT EXISTS local_users (
                    user_id TEXT PRIMARY KEY,
                    phone_number TEXT UNIQUE,
                    name TEXT,
                    district TEXT,
                    facility TEXT,
                    role TEXT DEFAULT 'chw',
                    points INTEGER DEFAULT 0,
                    badges TEXT,               -- JSON array
                    last_active TEXT,
                    created_at TEXT DEFAULT (datetime('now'))
                );

                -- Indexes for common queries
                CREATE INDEX IF NOT EXISTS idx_cases_district
                    ON cases(district);
                CREATE INDEX IF NOT EXISTS idx_cases_synced
                    ON cases(synced);
                CREATE INDEX IF NOT EXISTS idx_cases_created
                    ON cases(created_at);
                CREATE INDEX IF NOT EXISTS idx_resistance_district_drug
                    ON resistance_data(district, drug);
            """)
            logger.info("Database schema initialized")

    @contextmanager
    def _get_connection(self) -> Generator[sqlite3.Connection, None, None]:
        """Get a SQLite connection (context manager)."""
        conn = sqlite3.connect(self.db_path)
        conn.row_factory = sqlite3.Row
        conn.execute("PRAGMA journal_mode = WAL")
        conn.execute("PRAGMA foreign_keys = ON")
        try:
            yield conn
            conn.commit()
        except Exception:
            conn.rollback()
            raise
        finally:
            conn.close()

    def insert_case(self, case: dict) -> str:
        """Insert a case and return the case_id."""
        import json

        with self._get_connection() as conn:
            conn.execute(
                """INSERT INTO cases (
                    case_id, source, language, timestamp,
                    patient_age_group, patient_age_value,
                    specimen, body_site, pathogen,
                    resistance_pattern, test_result,
                    drugs_prescribed, dosages, clinical_outcome,
                    district, facility, clinical_notes,
                    confidence, resistance_flag, raw_text
                ) VALUES (?, ?, ?, ?, ?, ?, ?, ?, ?, ?, ?, ?, ?, ?, ?, ?, ?, ?, ?, ?, ?)""",
                (
                    case.get("case_id"),
                    case.get("source", "web"),
                    case.get("language", "en"),
                    case.get("timestamp", ""),
                    case.get("patient_age_group"),
                    case.get("patient_age_value"),
                    case.get("specimen"),
                    case.get("body_site"),
                    case.get("pathogen"),
                    case.get("resistance_pattern"),
                    case.get("test_result"),
                    json.dumps(case.get("drugs_prescribed", [])),
                    json.dumps(case.get("dosages", [])),
                    case.get("clinical_outcome"),
                    case.get("district"),
                    case.get("facility"),
                    case.get("clinical_notes"),
                    case.get("confidence", "medium"),
                    1 if case.get("resistance_flag") else 0,
                    case.get("raw_text"),
                ),
            )
            return case.get("case_id", "")

    def get_unsynced_cases(self) -> list[dict]:
        """Get all cases that haven't been synced to cloud."""
        import json

        with self._get_connection() as conn:
            rows = conn.execute(
                "SELECT * FROM cases WHERE synced = 0 ORDER BY created_at"
            ).fetchall()

            cases = []
            for row in rows:
                case = dict(row)
                case["drugs_prescribed"] = json.loads(
                    case.get("drugs_prescribed", "[]")
                )
                case["dosages"] = json.loads(case.get("dosages", "[]"))
                cases.append(case)
            return cases

    def mark_cases_synced(self, case_ids: list[str]):
        """Mark cases as synced to cloud."""
        import json
        from datetime import datetime

        with self._get_connection() as conn:
            placeholders = ",".join("?" * len(case_ids))
            conn.execute(
                f"""UPDATE cases SET synced = 1, sync_timestamp = ?
                    WHERE case_id IN ({placeholders})""",
                [datetime.utcnow().isoformat()] + case_ids,
            )

    def get_sync_meta(self, key: str, default: str = "") -> str:
        """Get a sync metadata value."""
        with self._get_connection() as conn:
            row = conn.execute(
                "SELECT value FROM sync_meta WHERE key = ?", (key,)
            ).fetchone()
            return row["value"] if row else default

    def set_sync_meta(self, key: str, value: str):
        """Set a sync metadata value (upsert)."""
        with self._get_connection() as conn:
            conn.execute(
                """INSERT INTO sync_meta (key, value) VALUES (?, ?)
                   ON CONFLICT(key) DO UPDATE SET value = ?""",
                (key, value, value),
            )

    def get_stats(self) -> dict:
        """Get database statistics."""
        with self._get_connection() as conn:
            total_cases = conn.execute(
                "SELECT COUNT(*) FROM cases"
            ).fetchone()[0]
            unsynced = conn.execute(
                "SELECT COUNT(*) FROM cases WHERE synced = 0"
            ).fetchone()[0]
            return {
                "db_path": self.db_path,
                "total_cases": total_cases,
                "unsynced_cases": unsynced,
                "db_size_mb": round(
                    Path(self.db_path).stat().st_size / (1024 * 1024), 2
                ),
            }
```

---

## 5. USSD Session Manager

### 5.1 USSD Architecture

```
USSD SESSION MANAGEMENT:
═══════════════════════════════════════════════════════════════

Feature Phone (USSD)          Africa's Talking        UDARA Cloud
┌──────────────┐              ┌──────────────┐        ┌──────────┐
│  *384*12#    │───dial──────►│  USSD        │──POST──►│  /ussd/  │
│              │              │  Gateway     │        │ callback │
│  Response:   │◄──response───│  (callback)  │◄200OK──│          │
│  CON menu    │              │              │        │          │
│              │              │              │        │ Redis    │
│  1. Report   │───input─────►│  Session ID  │──POST──►│ Session  │
│     Case     │              │  in headers  │        │ Manager  │
│              │              │              │        │          │
│  Enter pathogen│◄──menu─────│              │◄GET────│  State   │
│  > 1. E.coli  │              │              │        │  Machine │
│              │              │              │        │          │
│  Enter age:   │───input─────►│              │──POST──►│  Route   │
│  > 45        │              │              │        │  to      │
│              │              │              │        │  Flow    │
│  [Continue]  │◄──step──────│              │◄200OK──│          │
│              │              │              │        │          │
│  Confirm?    │              │              │        │          │
│  1. Yes      │───confirm───►│              │──POST──►│ Persist │
│              │              │              │        │  Case    │
│  Thank you!  │◄──END───────│              │◄200OK──│          │
└──────────────┘              └──────────────┘        └──────────┘

SESSION FLOW:
1. User dials USSD code → AT sends HTTP POST to /api/v1/ussd
2. API creates Redis session (120s TTL)
3. API returns USSD menu text (CON or END)
4. User responds → AT sends POST with session ID
5. API looks up Redis session, advances state machine
6. Repeat until flow completes or session expires

REDIS SESSION STRUCTURE:
{
    "session_id": "ussd_abc123",
    "phone": "+256700123456",
    "flow": "report_case",
    "step": 3,
    "data": {
        "pathogen": "E. coli",
        "patient_age": "45",
        "specimen": "urine"
    },
    "created_at": "2026-05-27T10:30:00Z",
    "expires_at": "2026-05-27T10:32:00Z",
    "district": "kampala_central"
}
```

### 5.2 Session Manager Implementation

```python
# backend/ussd/session_manager.py
"""
Redis-backed USSD session manager.

Manages multi-step USSD conversation sessions with TTL,
state tracking, and data accumulation across steps.
"""

from __future__ import annotations

import json
import logging
import time
import uuid
from dataclasses import dataclass, field
from typing import Any, Optional

logger = logging.getLogger(__name__)

SESSION_TTL_SECONDS = 120  # 2 minutes (USSD telco limit)


@dataclass
class USSDSession:
    """A USSD conversation session."""
    session_id: str
    phone_number: str
    flow: str                      # "report_case", "check_resistance", etc.
    step: int = 0
    data: dict = field(default_factory=dict)
    district: str = ""
    created_at: float = 0.0
    expires_at: float = 0.0


class USSDSessionManager:
    """
    Redis-backed session manager for USSD conversations.

    Features:
    - 120s TTL per session (telco timeout constraint)
    - State machine tracking (flow + step)
    - Data accumulation across steps
    - Automatic cleanup on expiry
    """

    def __init__(self, redis_client, ttl: int = SESSION_TTL_SECONDS):
        self.redis = redis_client
        self.ttl = ttl
        self._prefix = "ussd:session:"

    def create_session(
        self,
        phone_number: str,
        flow: str = "main_menu",
        district: str = "",
    ) -> USSDSession:
        """
        Create a new USSD session.

        Args:
            phone_number: User's phone number.
            flow: Initial flow name.
            district: User's district (if known).

        Returns:
            New USSDSession.
        """
        session = USSDSession(
            session_id=f"ussd_{uuid.uuid4().hex[:12]}",
            phone_number=phone_number,
            flow=flow,
            step=0,
            data={},
            district=district,
            created_at=time.time(),
            expires_at=time.time() + self.ttl,
        )

        self._save_session(session)
        logger.info(
            f"USSD session created: {session.session_id} "
            f"(phone={phone_number}, flow={flow})"
        )
        return session

    def get_session(self, session_id: str) -> Optional[USSDSession]:
        """Get a session by ID, returns None if expired."""
        key = f"{self._prefix}{session_id}"
        data = self.redis.get(key)

        if data is None:
            logger.debug(f"Session not found or expired: {session_id}")
            return None

        session_data = json.loads(data)

        # Check if expired
        if time.time() > session_data["expires_at"]:
            self.delete_session(session_id)
            logger.info(f"Session expired: {session_id}")
            return None

        return USSDSession(**session_data)

    def update_session(
        self,
        session_id: str,
        step: Optional[int] = None,
        data: Optional[dict] = None,
        flow: Optional[str] = None,
    ) -> Optional[USSDSession]:
        """Update session state."""
        session = self.get_session(session_id)
        if session is None:
            return None

        if step is not None:
            session.step = step
        if data is not None:
            session.data.update(data)
        if flow is not None:
            session.flow = flow

        self._save_session(session)
        return session

    def delete_session(self, session_id: str):
        """Delete a session."""
        key = f"{self._prefix}{session_id}"
        self.redis.delete(key)
        logger.debug(f"Session deleted: {session_id}")

    def extend_session(self, session_id: str, extra_seconds: int = 60):
        """Extend session TTL."""
        session = self.get_session(session_id)
        if session:
            session.expires_at = time.time() + extra_seconds
            self._save_session(session)

    def get_active_session_by_phone(
        self, phone_number: str
    ) -> Optional[USSDSession]:
        """Find active session for a phone number."""
        pattern = f"{self._prefix}*"
        for key in self.redis.scan_iter(pattern):
            data = self.redis.get(key)
            if data:
                session_data = json.loads(data)
                if (session_data["phone_number"] == phone_number
                        and time.time() < session_data["expires_at"]):
                    return USSDSession(**session_data)
        return None

    def _save_session(self, session: USSDSession):
        """Serialize and save session to Redis with TTL."""
        key = f"{self._prefix}{session.session_id}"
        ttl_remaining = max(0, session.expires_at - time.time())
        self.redis.setex(
            key,
            int(ttl_remaining),
            json.dumps({
                "session_id": session.session_id,
                "phone_number": session.phone_number,
                "flow": session.flow,
                "step": session.step,
                "data": session.data,
                "district": session.district,
                "created_at": session.created_at,
                "expires_at": session.expires_at,
            }),
        )
```

---

## 6. Sync Protocol Deep Dive

### 6.1 Protocol Overview

```
SYNC PROTOCOL ARCHITECTURE:
═══════════════════════════════════════════════════════════════

┌─────────────────────┐                    ┌─────────────────────┐
│    EDGE (RPi 5)     │                    │    CLOUD (AWS)      │
│                     │                    │                     │
│  ┌───────────────┐  │                    │  ┌───────────────┐  │
│  │  SQLite       │  │   DELTA JSON      │  │  PostgreSQL    │  │
│  │  (local)      │  │   + gzip          │  │  (cloud)       │  │
│  └───────┬───────┘  │                    │  └───────▲───────┘  │
│          │          │                    │          │          │
│          ▼          │                    │          │          │
│  ┌───────────────┐  │   ┌──────────┐     │  ┌───────┴───────┐  │
│  │  Delta        │──┼──►│  HTTP    │────┼─►│  Sync         │  │
│  │  Generator    │  │   │  POST    │     │  │  Receiver     │  │
│  └───────────────┘  │   │  /sync   │     │  │  (upsert +    │  │
│                     │   └──────────┘     │  │   conflict    │  │
│  ┌───────────────┐  │                    │  │   resolution) │  │
│  │  CRDT         │  │   ◄───────────────┼──┤               │  │
│  │  (Yjs)        │  │   200 OK          │  └───────────────┘  │
│  │  Conflict     │  │   + server delta  │                    │
│  │  Resolution   │  │                    │                    │
│  └───────────────┘  │                    │                    │
│                     │                    │                    │
│  ┌───────────────┐  │                    │                    │
│  │  Compressor   │  │   gzip level 6    │                    │
│  │  (gzip)       │  │   ~70% reduction  │                    │
│  └───────────────┘  │                    │                    │
│                     │                    │                    │
│  ┌───────────────┐  │                    │                    │
│  │  Retry        │  │   1s → 2s → 4s    │                    │
│  │  (backoff)    │  │   → 8s → 16s max  │                    │
│  └───────────────┘  │                    │                    │
│                     │                    │                    │
│  ┌───────────────┐  │   USB FALLBACK    │                    │
│  │  USB Sync     │  │   (no internet)   │                    │
│  │  (encrypted   │  │                    │                    │
│  │   tarball)    │  │                    │                    │
│  └───────────────┘  │                    │                    │
└─────────────────────┘                    └─────────────────────┘
```

### 6.2 Sync State Machine

```
SYNC STATE MACHINE:
═══════════════════════════════════════════════════════════════

         ┌──────────┐
         │          │
         ▼          │
    ┌─────────┐    │    Success: start next sync cycle
    │  IDLE   │────┘
    │         │◄──────────────────────────────┐
    └────┬────┘                               │
         │ Timer (5 min) / Trigger           │
         ▼                                    │
    ┌──────────┐                              │
    │GENERATING│  Generate delta JSON          │
    │          │  from SQLite (unsynced rows) │
    └────┬─────┘                              │
         │ Delta ready                         │
         ▼                                    │
    ┌──────────┐                              │
    │COMPRESSING│ gzip level 6                │
    │          │                              │
    └────┬─────┘                              │
         │ Compressed                          │
         ▼                                    │
    ┌──────────┐                              │
    │UPLOADING │ HTTP POST to cloud            │
    │          │                              │
    └──┬───┬──┘                              │
       │   │                                  │
   Success  │ Failure                          │
       │   │                                  │
       │   ▼                                  │
       │ ┌──────────┐                          │
       │ │RETRYING  │ exponential backoff      │
       │ │          │ 1s → 2s → 4s → 8s → 16s │
       │ └────┬─────┘                          │
       │      │ Max retries (5) exceeded       │
       │      ▼                                │
       │ ┌──────────┐                          │
       │ │ FAILED   │ Log error, queue USB     │
       │ └──────────┘                          │
       │                                      │
       ▼                                      │
    ┌──────────┐                              │
    │CONFIRMING│ Parse server response,        │
    │          │ update sync_meta, mark synced │
    └──────────┘──────────────────────────────┘

SYNC STATES (enum):
    IDLE = "idle"
    GENERATING = "generating"
    COMPRESSING = "compressing"
    UPLOADING = "uploading"
    CONFIRMING = "confirming"
    RETRYING = "retrying"
    FAILED = "failed"
```

### 6.3 Sync Orchestrator Implementation

```python
# backend/sync/orchestrator.py
"""
Sync orchestrator — manages edge-to-cloud synchronization.

Implements the sync state machine with delta generation,
compression, retry logic, and USB fallback.
"""

from __future__ import annotations

import asyncio
import gzip
import json
import logging
import time
from dataclasses import dataclass, field
from enum import Enum
from typing import Optional

import httpx

logger = logging.getLogger(__name__)


class SyncState(str, Enum):
    """Sync state machine states."""
    IDLE = "idle"
    GENERATING = "generating"
    COMPRESSING = "compressing"
    UPLOADING = "uploading"
    CONFIRMING = "confirming"
    RETRYING = "retrying"
    FAILED = "failed"


@dataclass
class SyncResult:
    """Result of a sync operation."""
    success: bool
    state: SyncState
    records_synced: int = 0
    bytes_sent: int = 0
    duration_ms: float = 0.0
    error: Optional[str] = None


class SyncOrchestrator:
    """
    Edge-to-cloud sync orchestrator.

    Manages the complete sync lifecycle:
    1. Generate delta JSON from unsynced SQLite rows
    2. Compress with gzip
    3. Upload to cloud API
    4. Handle conflicts via CRDT
    5. Confirm and mark synced
    6. Retry on failure with exponential backoff
    """

    MAX_RETRIES = 5
    BACKOFF_BASE = 2  # seconds
    BACKOFF_MAX = 60  # seconds

    def __init__(
        self,
        db,
        cloud_url: str,
        node_id: str,
        interval: int = 300,
    ):
        self.db = db
        self.cloud_url = cloud_url
        self.node_id = node_id
        self.interval = interval
        self.state = SyncState.IDLE
        self._retry_count = 0
        self._last_sync_time = 0.0
        self._running = False
        self._task: Optional[asyncio.Task] = None

    def start(self):
        """Start the periodic sync loop."""
        self._running = True
        self._task = asyncio.create_task(self._sync_loop())
        logger.info(
            f"Sync orchestrator started "
            f"(interval={self.interval}s, node={self.node_id})"
        )

    def stop(self):
        """Stop the sync loop."""
        self._running = False
        if self._task:
            self._task.cancel()
        logger.info("Sync orchestrator stopped")

    async def _sync_loop(self):
        """Main sync loop — runs periodically."""
        while self._running:
            try:
                await asyncio.sleep(self.interval)
                if self.state == SyncState.IDLE:
                    result = await self.sync()
                    if not result.success:
                        logger.warning(
                            f"Sync failed: {result.error}"
                        )
            except asyncio.CancelledError:
                break
            except Exception as e:
                logger.error(f"Sync loop error: {e}")
                await asyncio.sleep(60)

    async def sync(self) -> SyncResult:
        """
        Execute a complete sync cycle.

        State machine: IDLE → GENERATING → COMPRESSING →
                       UPLOADING → CONFIRMING → IDLE
        """
        start = time.time()

        try:
            # Step 1: GENERATE delta
            self.state = SyncState.GENERATING
            delta = self._generate_delta()
            if not delta["records"]:
                self.state = SyncState.IDLE
                return SyncResult(
                    success=True, state=SyncState.IDLE
                )

            # Step 2: COMPRESS
            self.state = SyncState.COMPRESSING
            compressed = self._compress_delta(delta)

            # Step 3: UPLOAD
            self.state = SyncState.UPLOADING
            response = await self._upload_delta(compressed)

            # Step 4: CONFIRM
            self.state = SyncState.CONFIRMING
            self._confirm_sync(delta["case_ids"], response)

            # Success
            self.state = SyncState.IDLE
            self._retry_count = 0
            self._last_sync_time = time.time()

            elapsed_ms = (time.time() - start) * 1000
            logger.info(
                f"Sync complete: {len(delta['records'])} records, "
                f"{len(compressed)} bytes, {elapsed_ms:.0f}ms"
            )

            return SyncResult(
                success=True,
                state=SyncState.IDLE,
                records_synced=len(delta["records"]),
                bytes_sent=len(compressed),
                duration_ms=elapsed_ms,
            )

        except Exception as e:
            self._retry_count += 1
            error_msg = str(e)

            if self._retry_count <= self.MAX_RETRIES:
                self.state = SyncState.RETRYING
                backoff = min(
                    self.BACKOFF_BASE ** self._retry_count,
                    self.BACKOFF_MAX,
                )
                logger.warning(
                    f"Sync failed (attempt {self._retry_count}), "
                    f"retrying in {backoff}s: {error_msg}"
                )
                await asyncio.sleep(backoff)
                return await self.sync()  # Recursive retry
            else:
                self.state = SyncState.FAILED
                logger.error(
                    f"Sync failed after {self.MAX_RETRIES} retries: "
                    f"{error_msg}"
                )
                return SyncResult(
                    success=False,
                    state=SyncState.FAILED,
                    error=error_msg,
                )

    def _generate_delta(self) -> dict:
        """Generate delta JSON from unsynced SQLite records."""
        unsynced = self.db.get_unsynced_cases()

        records = []
        case_ids = []
        for case in unsynced:
            records.append({
                "case_id": case["case_id"],
                "table": "cases",
                "operation": "upsert",
                "data": {
                    k: v for k, v in case.items()
                    if k not in ("synced", "sync_timestamp")
                },
            })
            case_ids.append(case["case_id"])

        last_sync = self.db.get_sync_meta(
            "last_sync_timestamp", "1970-01-01T00:00:00Z"
        )

        delta = {
            "node_id": self.node_id,
            "timestamp": time.time(),
            "last_sync": last_sync,
            "record_count": len(records),
            "records": records,
            "case_ids": case_ids,
        }

        logger.info(
            f"Delta generated: {len(records)} unsynced records"
        )
        return delta

    def _compress_delta(self, delta: dict) -> bytes:
        """Compress delta JSON with gzip level 6."""
        json_bytes = json.dumps(delta, default=str).encode("utf-8")
        compressed = gzip.compress(json_bytes, compresslevel=6)
        ratio = (1 - len(compressed) / len(json_bytes)) * 100
        logger.info(
            f"Compressed {len(json_bytes)} → {len(compressed)} bytes "
            f"({ratio:.1f}% reduction)"
        )
        return compressed

    async def _upload_delta(self, compressed: bytes) -> dict:
        """Upload compressed delta to cloud."""
        headers = {
            "Content-Type": "application/gzip",
            "Content-Encoding": "gzip",
            "X-Edge-Node-ID": self.node_id,
        }

        async with httpx.AsyncClient(timeout=30.0) as client:
            response = await client.post(
                f"{self.cloud_url}/upload",
                content=compressed,
                headers=headers,
            )
            response.raise_for_status()
            return response.json()

    def _confirm_sync(
        self, case_ids: list[str], response: dict
    ):
        """Mark synced records and update metadata."""
        self.db.mark_cases_synced(case_ids)
        self.db.set_sync_meta(
            "last_sync_timestamp",
            time.strftime("%Y-%m-%dT%H:%M:%SZ", time.gmtime()),
        )
        logger.info(f"Confirmed sync for {len(case_ids)} cases")
```

---

## 7. Database Schemas

### 7.1 Edge vs Cloud Schema Comparison

```
EDGE (SQLite)              CLOUD (PostgreSQL + TimescaleDB + PostGIS + Neo4j)
═════════════════════      ════════════════════════════════════════════════════

4 Tables                   12+ Tables + Graph

cases (local)              cases (cloud, partitioned by year)
resistance_data (local)    resistance_observations (TimescaleDB hypertable)
sync_meta (key-value)      users (CHW profiles, Keycloak-linked)
local_users (cache)        facilities (health facility registry)
                           districts (PostGIS geometry)
                           resistance_summary (materialized view)
                           sync_log (audit trail)
                           api_keys (bot credentials)
                           notifications (push/alert queue)
                           gamification (points, badges, leaderboard)
                           drug_labels (OCR results, media storage)
                           graph (Neo4j: drug-pathogen-resistance KG)

KEY DIFFERENCES:
- Edge: Simple, flat, no foreign keys to cloud tables
- Cloud: Normalized, indexed, partitioned, spatially enabled
- Edge: JSON strings for arrays (SQLite limitation)
- Cloud: Native JSONB, ARRAY types, PostGIS geometry
- Edge: No time-series — cloud uses TimescaleDB hypertables
- Edge: No graph — cloud uses Neo4j for drug-pathogen relationships
```

### 7.2 Cloud PostgreSQL Schema

```sql
-- Cloud PostgreSQL Schema for UDARA AI
-- PostgreSQL 16 + TimescaleDB + PostGIS

-- Enable extensions
CREATE EXTENSION IF NOT EXISTS "uuid-ossp";
CREATE EXTENSION IF NOT EXISTS "postgis";
CREATE EXTENSION IF NOT EXISTS "timescaledb";

-- ═══════════════════════════════════════════
-- TABLE: cases (AMR case reports)
-- ═══════════════════════════════════════════
CREATE TABLE cases (
    case_id          UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    source           VARCHAR(20) NOT NULL,  -- ussd, telegram, whatsapp, web
    language         VARCHAR(10) DEFAULT 'en',
    timestamp        TIMESTAMPTZ NOT NULL,
    patient_age_group VARCHAR(20),
    patient_age_value INTEGER,
    specimen         VARCHAR(50),
    body_site        VARCHAR(100),
    pathogen         VARCHAR(100),
    resistance_pattern VARCHAR(50),
    test_result      VARCHAR(50),
    drugs_prescribed JSONB DEFAULT '[]'::jsonb,
    dosages          JSONB DEFAULT '[]'::jsonb,
    clinical_outcome VARCHAR(50),
    district         VARCHAR(100) NOT NULL,
    facility         VARCHAR(200),
    clinical_notes   TEXT,
    confidence       VARCHAR(10) DEFAULT 'medium',
    resistance_flag  BOOLEAN DEFAULT FALSE,
    reporter_id      UUID REFERENCES users(user_id),
    raw_text         TEXT,
    created_at       TIMESTAMPTZ DEFAULT NOW(),
    updated_at       TIMESTAMPTZ DEFAULT NOW(),
    deleted_at       TIMESTAMPTZ  -- Soft delete
);

-- Partition by year for performance
CREATE INDEX idx_cases_district ON cases(district);
CREATE INDEX idx_cases_pathogen ON cases(pathogen);
CREATE INDEX idx_cases_created ON cases(created_at DESC);
CREATE INDEX idx_cases_resistance ON cases(resistance_flag) WHERE resistance_flag = TRUE;
CREATE INDEX idx_cases_drugs ON cases USING GIN(drugs_prescribed);

-- ═══════════════════════════════════════════
-- TABLE: resistance_observations (time-series)
-- ═══════════════════════════════════════════
CREATE TABLE resistance_observations (
    id               BIGSERIAL,
    district         VARCHAR(100) NOT NULL,
    drug             VARCHAR(100) NOT NULL,
    pathogen         VARCHAR(100) NOT NULL,
    total_tested     INTEGER NOT NULL DEFAULT 0,
    resistant_count  INTEGER NOT NULL DEFAULT 0,
    resistance_pct   NUMERIC(5,2) GENERATED ALWAYS AS (
        CASE WHEN total_tested > 0
        THEN ROUND(resistant_count::NUMERIC / total_tested * 100, 2)
        ELSE NULL END
    ) STORED,
    year             INTEGER NOT NULL,
    quarter          INTEGER NOT NULL CHECK (quarter BETWEEN 1 AND 4),
    source_node      VARCHAR(50),          -- Which edge node reported
    created_at       TIMESTAMPTZ DEFAULT NOW(),
    PRIMARY KEY (id, year)
);

-- Convert to TimescaleDB hypertable (partitioned by time)
SELECT create_hypertable('resistance_observations', 'created_at');

-- Continuous aggregate: quarterly resistance summary
CREATE MATERIALIZED VIEW resistance_quarterly
WITH (timescaledb.continuous) AS
SELECT
    district,
    drug,
    pathogen,
    year,
    quarter,
    SUM(total_tested) as total_tested,
    SUM(resistant_count) as resistant_count,
    CASE WHEN SUM(total_tested) > 0
        THEN ROUND(SUM(resistant_count)::NUMERIC / SUM(total_tested) * 100, 2)
        ELSE NULL END as resistance_pct
FROM resistance_observations
GROUP BY district, drug, pathogen, year, quarter;

-- ═══════════════════════════════════════════
-- TABLE: districts (spatial — PostGIS)
-- ═══════════════════════════════════════════
CREATE TABLE districts (
    district_id   VARCHAR(50) PRIMARY KEY,
    district_name VARCHAR(200) NOT NULL,
    country       VARCHAR(100) NOT NULL,
    region        VARCHAR(100),
    population    INTEGER,
    geometry      GEOMETRY(MultiPolygon, 4326),  -- WGS84
    centroid      GEOMETRY(Point, 4326),
    created_at    TIMESTAMPTZ DEFAULT NOW()
);

-- Spatial index for efficient geo queries
CREATE INDEX idx_districts_geometry ON districts USING GIST(geometry);
CREATE INDEX idx_districts_centroid ON districts USING GIST(centroid);

-- ═══════════════════════════════════════════
-- TABLE: users (CHW profiles)
-- ═══════════════════════════════════════════
CREATE TABLE users (
    user_id        UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    keycloak_id    VARCHAR(100) UNIQUE,
    phone_number   VARCHAR(20) UNIQUE,
    name           VARCHAR(200),
    district       VARCHAR(100) REFERENCES districts(district_id),
    facility       VARCHAR(200),
    role           VARCHAR(20) DEFAULT 'chw',  -- chw, clinician, lab_tech, admin
    points         INTEGER DEFAULT 0,
    badges         JSONB DEFAULT '[]'::jsonb,
    streak_days    INTEGER DEFAULT 0,
    last_active    TIMESTAMPTZ,
    is_active      BOOLEAN DEFAULT TRUE,
    created_at     TIMESTAMPTZ DEFAULT NOW()
);

-- ═══════════════════════════════════════════
-- TABLE: sync_log (audit trail)
-- ═══════════════════════════════════════════
CREATE TABLE sync_log (
    sync_id        UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    node_id        VARCHAR(50) NOT NULL,
    direction      VARCHAR(10) NOT NULL,  -- upload, download
    records_count  INTEGER NOT NULL,
    bytes_transferred INTEGER,
    status         VARCHAR(20) NOT NULL,  -- success, failed, partial
    error_message  TEXT,
    started_at     TIMESTAMPTZ NOT NULL,
    completed_at   TIMESTAMPTZ
);
```

### 7.3 Neo4j Knowledge Graph Schema

```cypher
// Neo4j Knowledge Graph: Drug-Pathogen-Resistance relationships

// Node types
CREATE CONSTRAINT drug_name IF NOT EXISTS
FOR (d:Drug) REQUIRE d.name IS UNIQUE;

CREATE CONSTRAINT pathogen_species IF NOT EXISTS
FOR (p:Pathogen) REQUIRE p.species IS UNIQUE;

CREATE CONSTRAINT resistance_type IF NOT EXISTS
FOR (r:ResistanceType) REQUIRE r.type IS UNIQUE;

CREATE CONSTRAINT drug_class IF NOT EXISTS
FOR (c:DrugClass) REQUIRE c.name IS UNIQUE;

// Sample data seeding
// Drug nodes
CREATE (d1:Drug {name: "amoxicillin", class: "penicillin"})
CREATE (d2:Drug {name: "ciprofloxacin", class: "fluoroquinolone"})
CREATE (d3:Drug {name: "cotrimoxazole", class: "sulfonamide"})
CREATE (d4:Drug {name: "ceftriaxone", class: "cephalosporin"})
CREATE (d5:Drug {name: "gentamicin", class: "aminoglycoside"})

// Drug class hierarchy
CREATE (c1:DrugClass {name: "penicillin"})
CREATE (c2:DrugClass {name: "fluoroquinolone"})
CREATE (c3:DrugClass {name: "sulfonamide"})
CREATE (c4:DrugClass {name: "cephalosporin"})

CREATE (d1)-[:BELONGS_TO]->(c1)
CREATE (d2)-[:BELONGS_TO]->(c2)
CREATE (d3)-[:BELONGS_TO]->(c3)
CREATE (d4)-[:BELONGS_TO]->(c4)

// Pathogen nodes
CREATE (p1:Pathogen {species: "Escherichia coli", gram: "negative"})
CREATE (p2:Pathogen {species: "Staphylococcus aureus", gram: "positive"})
CREATE (p3:Pathogen {species: "Klebsiella pneumoniae", gram: "negative"})
CREATE (p4:Pathogen {species: "Salmonella typhi", gram: "negative"})

// Resistance type nodes
CREATE (r1:ResistanceType {type: "ESBL"})
CREATE (r2:ResistanceType {type: "MRSA"})
CREATE (r3:ResistanceType {type: "CRE"})
CREATE (r4:ResistanceType {type: "MDR"})

// Resistance relationships
// (pathogen)-[:RESISTS {district, year, pct}]->(drug)
CREATE (p1)-[:RESISTS {
    district: "kampala", year: 2024, quarter: 1,
    pct: 45.2, total_tested: 120, resistant: 54
}]->(d1)

// Pathogen → Resistance Type
CREATE (p1)-[:EXHIBITS]->(r1)  // E. coli → ESBL
CREATE (p2)-[:EXHIBITS]->(r2)  // S. aureus → MRSA

// Alternative drug recommendations
// drug)-[:ALTERNATIVE_FOR]->(drug) based on resistance
CREATE (d4)-[:ALTERNATIVE_FOR {
    context: "ESBL urinary_tract_infection"
}]->(d1)  // ceftriaxone alt for amoxicillin when ESBL

// Query examples
// 1. Find resistance % for amoxicillin against E. coli in Kampala
MATCH (p:Pathogen {species: "Escherichia coli"})
      -[r:RESISTS {district: "kampala"}]->(d:Drug {name: "amoxicillin"})
RETURN r.pct, r.total_tested, r.year

// 2. Find alternative drugs when E. coli is resistant to amoxicillin
MATCH (p:Pathogen {species: "Escherichia coli"})
      -[:RESISTS]->(d1:Drug {name: "amoxicillin"}),
      (d2:Drug)-[:ALTERNATIVE_FOR]->(d1)
WHERE NOT EXISTS((
    MATCH (p)-[r:RESISTS]->(d2) WHERE r.pct > 30
))
RETURN d2.name, d2.class
```

---

## 8. Africa's Talking Integration

### 8.1 API Client

```python
# backend/ussd/africa_talking.py
"""
Africa's Talking API client for USSD and SMS.

Handles callback processing, USSD session creation,
and SMS delivery for AMR notifications.
"""

from __future__ import annotations

import hashlib
import hmac
import json
import logging
import time
from dataclasses import dataclass
from typing import Optional

import httpx

logger = logging.getLogger(__name__)


@dataclass
class USSDCallback:
    """Parsed USSD callback from Africa's Talking."""
    session_id: str
    phone_number: str
    text: str                          # User's input
    service_code: str                   # USSD short code (*384*12#)
    channel: str = "ussd"


@dataclass
class USSDResponse:
    """Response to send back via USSD."""
    text: str
    is_terminal: bool = False          # True = END, False = CON

    def to_africastalking_format(self) -> str:
        """Format for Africa's Talking callback response."""
        if self.is_terminal:
            return self.text           # Plain text = END session
        else:
            return f"CON {self.text}"  # CON = Continue session


class AfricasTalkingClient:
    """
    Africa's Talking API integration.

    Handles:
    - USSD callback parsing
    - SMS sending (notifications, reminders)
    - Signature validation (security)
    """

    BASE_URL = "https://api.africastalking.com/v1"
    USSD_CALLBACK_PATH = "/api/v1/ussd"

    def __init__(
        self,
        username: str,
        api_key: str,
        shortcode: str,
        from_phone: str,
    ):
        self.username = username
        self.api_key = api_key
        self.shortcode = shortcode
        self.from_phone = from_phone

    def parse_ussd_callback(self, form_data: dict) -> USSDCallback:
        """Parse incoming USSD callback from Africa's Talking."""
        return USSDCallback(
            session_id=form_data.get("sessionId", ""),
            phone_number=form_data.get("phoneNumber", ""),
            text=form_data.get("text", ""),
            service_code=form_data.get("serviceCode", ""),
            channel=form_data.get("channel", "ussd"),
        )

    def validate_signature(
        self,
        form_data: dict,
        signature: str,
        api_key: Optional[str] = None,
    ) -> bool:
        """
        Validate webhook signature from Africa's Talking.

        Security: prevents spoofed USSD callbacks.
        """
        key = api_key or self.api_key

        # Build the signature string
        sorted_items = sorted(form_data.items())
        signature_string = "".join(
            f"{k}{v}" for k, v in sorted_items
        )

        # HMAC-SHA256
        expected = hmac.new(
            key.encode("utf-8"),
            signature_string.encode("utf-8"),
            hashlib.sha256,
        ).hexdigest()

        return hmac.compare_digest(expected, signature)

    async def send_sms(
        self,
        to: str,
        message: str,
    ) -> dict:
        """
        Send SMS via Africa's Talking.

        Used for: AMR alerts, sync notifications, training reminders.
        """
        url = f"{self.BASE_URL}/messaging"
        headers = {
            "apiKey": self.api_key,
            "Content-Type": "application/x-www-form-urlencoded",
            "Accept": "application/json",
        }
        data = {
            "username": self.username,
            "to": to,
            "message": message,
            "from": self.from_phone,
        }

        async with httpx.AsyncClient(timeout=30.0) as client:
            response = await client.post(
                url, headers=headers, data=data
            )
            response.raise_for_status()
            result = response.json()

        logger.info(f"SMS sent to {to}: {result}")
        return result
```

---

## 9. Complete Code Examples

*See previous sections for complete code examples covering:*

- `main.py` — FastAPI application setup (Section 3.2)
- `dependencies.py` — Dependency injection (Section 3.3)
- `routes/cases.py` — Case CRUD endpoints (Section 3.4)
- `edge/runtime.py` — Edge runtime orchestrator (Section 4.2)
- `edge/model_manager.py` — Model lifecycle management (Section 4.3)
- `edge/local_db.py` — SQLite edge database (Section 4.4)
- `ussd/session_manager.py` — USSD session management (Section 5.2)
- `sync/orchestrator.py` — Sync state machine (Section 6.3)
- `africa_talking.py` — Africa's Talking client (Section 8.1)

---

## 10. Week-by-Week Tasks

### Week 1: Foundation

| Day | Task | Deliverable |
|-----|------|-------------|
| Mon | Create monorepo structure + Docker Compose | `docker compose up` starts all services |
| Tue | FastAPI scaffold + first routes | /health, basic /cases endpoints |
| Wed | PostgreSQL + PostGIS + TimescaleDB setup | All extensions enabled, test tables created |
| Thu | Redis + Neo4j containers | Connection pools working |
| Fri | SQLite edge schema | 4 tables created, WAL mode verified |

### Week 2: Core ML + Backend

| Day | Task | Deliverable |
|-----|------|-------------|
| Mon | Edge Docker image for RPi 5 | Builds and runs on ARM64 |
| Tue | Edge runtime on RPi 5 | FastAPI running on :8080 |
| Wed | SQLite operations + sync meta | CRUD working, sync_meta read/write |
| Thu | Delta JSON generator v1 | Generates delta from unsynced rows |
| Fri | gzip compression + basic upload | Compressed delta uploads to cloud |

### Week 3: Interfaces

| Day | Task | Deliverable |
|-----|------|-------------|
| Mon | Africa's Talking sandbox setup | Callback URL receives USSD events |
| Tue | Redis session manager | Sessions create/expire correctly |
| Wed | USSD menu routing | Main menu → report case / check resistance |
| Thu | Report case flow (6 steps) | Complete case submission via USSD |
| Fri | Check resistance flow | Query resistance by drug + district |

### Week 4: Intelligence Layer

| Day | Task | Deliverable |
|-----|------|-------------|
| Mon | CRDT conflict resolution (Yjs) | Concurrent edits resolve correctly |
| Tue | Full sync protocol | IDLE→GENERATING→COMPRESSING→UPLOADING→CONFIRMING |
| Wed | Retry + USB fallback | Backoff works, tarball generates |
| Thu | Neo4j knowledge graph | Drug-pathogen-resistance graph seeded |
| Fri | PostGIS spatial queries | District geometry queries working |

### Week 5: Polish + Pilot Prep

| Day | Task | Deliverable |
|-----|------|-------------|
| Mon | End-to-end sync test | Edge → Cloud → Edge roundtrip |
| Tue | Balena Fleet setup | 3 RPi devices in fleet dashboard |
| Wed | RPi deployment to 3 units | All nodes reporting health |
| Thu | Monitoring + alerting | Grafana dashboards, Prometheus metrics |
| Fri | Operations runbook | Deployment, monitoring, incident docs |

---

## 11. Testing Strategy

### 11.1 Test Types

```
TESTING PYRAMID — BACKEND ENGINEER:
═══════════════════════════════════════════════════════════════

           ┌───────────┐
           │   E2E      │  ← Full system tests (10 tests)
           │  Tests     │    USSD → API → SQLite → Sync → Cloud
           ├───────────┤
           │Integration │  ← Multi-component (20 tests)
           │  Tests     │    API + DB, Sync + CRDT, USSD + Redis
           ├───────────┤
           │   Unit     │  ← Function-level (80+ tests)
           │   Tests    │    Individual routes, sync logic, DB ops
           └───────────┘
```

### 11.2 Sample Tests

```python
# backend/tests/test_sync.py
"""Tests for the sync protocol."""

import pytest
import gzip
import json
from backend.sync.orchestrator import SyncOrchestrator, SyncState


class TestDeltaGeneration:

    def test_empty_delta(self, edge_db):
        """When no unsynced records, delta should be empty."""
        delta = edge_db.get_unsynced_cases()
        assert delta == []

    def test_delta_includes_unsynced(self, edge_db):
        """Delta should include all unsynced cases."""
        edge_db.insert_case({
            "case_id": "test-001",
            "source": "ussd",
            "district": "kampala",
            "pathogen": "E. coli",
            "drugs_prescribed": ["amoxicillin"],
        })

        delta = edge_db.get_unsynced_cases()
        assert len(delta) == 1
        assert delta[0]["case_id"] == "test-001"

    def test_delta_excludes_synced(self, edge_db):
        """Delta should exclude already-synced cases."""
        edge_db.insert_case({
            "case_id": "test-synced",
            "source": "web",
            "district": "accra",
        })
        edge_db.mark_cases_synced(["test-synced"])

        delta = edge_db.get_unsynced_cases()
        assert len(delta) == 0


class TestCompression:

    def test_gzip_compression(self):
        """Compressed delta should be smaller than raw."""
        delta = {
            "records": [{"case_id": f"case-{i}"} for i in range(100)],
            "timestamp": "2026-05-27T10:00:00Z",
        }

        raw = json.dumps(delta).encode("utf-8")
        compressed = gzip.compress(raw, compresslevel=6)

        assert len(compressed) < len(raw)
        assert len(compressed) / len(raw) < 0.5  # > 50% reduction

    def test_roundtrip(self):
        """Compressed data should decompress correctly."""
        original = b'{"test": "data"}'
        compressed = gzip.compress(original)
        decompressed = gzip.decompress(compressed)
        assert decompressed == original


class TestStateMachine:

    def test_initial_state(self):
        """Orchestrator should start in IDLE state."""
        orch = SyncOrchestrator(
            db=None, cloud_url="http://test", node_id="test"
        )
        assert orch.state == SyncState.IDLE

    def test_max_retries(self):
        """Should stop retrying after MAX_RETRIES."""
        orch = SyncOrchestrator(
            db=None, cloud_url="http://invalid", node_id="test"
        )
        assert orch.MAX_RETRIES == 5
```

```python
# backend/tests/test_ussd.py
"""Tests for USSD session management."""

import pytest
import time
from unittest.mock import MagicMock
from backend.ussd.session_manager import USSDSessionManager


class TestUSSDSessionManager:

    def test_create_session(self, redis_client):
        """Should create session with correct TTL."""
        mgr = USSDSessionManager(redis_client)
        session = mgr.create_session(
            phone_number="+256700123456",
            flow="report_case",
            district="kampala",
        )
        assert session.phone_number == "+256700123456"
        assert session.flow == "report_case"
        assert session.step == 0
        assert session.data == {}

    def test_get_session(self, redis_client):
        """Should retrieve session by ID."""
        mgr = USSDSessionManager(redis_client)
        created = mgr.create_session(phone_number="+256700123456")
        retrieved = mgr.get_session(created.session_id)
        assert retrieved is not None
        assert retrieved.session_id == created.session_id

    def test_update_session_step(self, redis_client):
        """Should advance session step."""
        mgr = USSDSessionManager(redis_client)
        session = mgr.create_session(phone_number="+256700123456")
        updated = mgr.update_session(
            session.session_id, step=2
        )
        assert updated.step == 2

    def test_session_expiry(self, redis_client):
        """Should not return expired sessions."""
        mgr = USSDSessionManager(redis_client, ttl=1)
        session = mgr.create_session(phone_number="+256700123456")
        time.sleep(1.1)
        expired = mgr.get_session(session.session_id)
        assert expired is None
```

---

## 12. Monitoring & Observability

### 12.1 Metrics Collected

```
METRICS — BACKEND + EDGE:
═══════════════════════════════════════════════════════════════

CLOUD (Prometheus + Grafana):
┌────────────────────────────────────────────────────────────┐
│ API Metrics:                                               │
│   udara_api_requests_total{method, endpoint, status}       │
│   udara_api_request_duration_seconds{endpoint, quantile}   │
│   udara_api_active_connections                             │
│                                                            │
│ Database Metrics:                                          │
│   udara_db_query_duration_seconds{query_type}              │
│   udara_db_connections_active                              │
│   udara_db_connections_max                                 │
│   udara_db_rows_affected{table, operation}                 │
│                                                            │
│ Sync Metrics:                                              │
│   udara_sync_records_synced_total{node_id, direction}      │
│   udara_sync_duration_seconds{node_id, status}             │
│   udara_sync_bytes_transferred_total{node_id, direction}   │
│   udara_sync_queue_size{node_id}                           │
│                                                            │
│ USSD Metrics:                                              │
│   udara_ussd_sessions_active                               │
│   udara_ussd_sessions_created_total                        │
│   udara_ussd_session_duration_seconds{flow, outcome}        │
│   udara_ussd_steps_completed{flow}                         │
└────────────────────────────────────────────────────────────┘

EDGE (local Prometheus + Balena dashboard):
┌────────────────────────────────────────────────────────────┐
│ Hardware Metrics:                                          │
│   udara_edge_cpu_temp_celsius                              │
│   udara_edge_cpu_percent                                   │
│   udara_edge_ram_available_mb                              │
│   udara_edge_ram_used_percent                              │
│   udara_edge_disk_used_percent                             │
│   udara_edge_uptime_seconds                                │
│                                                            │
│ Model Metrics:                                             │
│   udara_model_loaded{name}                                 │
│   udara_model_inference_duration_seconds{name}             │
│   udara_model_memory_usage_mb{name}                        │
│   udara_model_unload_total{name, reason}                   │
│                                                            │
│ Sync Metrics:                                              │
│   udara_sync_last_success_timestamp{node_id}               │
│   udara_sync_queue_size{node_id}                           │
│   udara_sync_unsynced_cases{node_id}                       │
└────────────────────────────────────────────────────────────┘
```

---

## 13. Appendices

### Appendix A: Docker Compose

```yaml
# docker-compose.yml
version: "3.9"

services:
  api:
    build:
      context: ./backend
      dockerfile: Dockerfile
    ports:
      - "8000:8000"
    environment:
      - DATABASE_URL=postgresql://udara:secret@postgres:5432/udara
      - REDIS_URL=redis://redis:6379/0
      - NEO4J_URI=bolt://neo4j:7687
    depends_on:
      - postgres
      - redis
      - neo4j
    volumes:
      - ./backend:/app
    restart: unless-stopped

  postgres:
    image: timescale/timescaledb:latest-pg16
    environment:
      POSTGRES_DB: udara
      POSTGRES_USER: udara
      POSTGRES_PASSWORD: secret
    ports:
      - "5432:5432"
    volumes:
      - postgres_data:/var/lib/postgresql/data
    restart: unless-stopped

  redis:
    image: redis:7-alpine
    ports:
      - "6379:6379"
    volumes:
      - redis_data:/data
    restart: unless-stopped

  neo4j:
    image: neo4j:5-community
    environment:
      NEO4J_AUTH: neo4j/secret
    ports:
      - "7474:7474"
      - "7687:7687"
    volumes:
      - neo4j_data:/data
    restart: unless-stopped

  qdrant:
    image: qdrant/qdrant:latest
    ports:
      - "6333:6333"
      - "6334:6334"
    volumes:
      - qdrant_data:/qdrant/storage
    restart: unless-stopped

volumes:
  postgres_data:
  redis_data:
  neo4j_data:
  qdrant_data:
```

### Appendix B: Edge Dockerfile

```dockerfile
# backend/Dockerfile.edge
# Optimized for Raspberry Pi 5 (ARM64)

FROM python:3.11-slim-bookworm

# System dependencies for ML models
RUN apt-get update && apt-get install -y --no-install-recommends \
    build-essential \
    cmake \
    libopenblas-dev \
    libomp-dev \
    && rm -rf /var/lib/apt/lists/*

WORKDIR /app

# Install Python dependencies
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

# Copy application code
COPY . .

# Pre-create data directories
RUN mkdir -p /data/models /data/sync

# Health check
HEALTHCHECK --interval=30s --timeout=10s --retries=3 \
    CMD python -c "import requests; requests.get('http://localhost:8080/health')" || exit 1

# Run edge runtime
CMD ["python", "-m", "edge.runtime"]
```

---

> **Document End**  
> Next: [14-usps-features.md](./14-usps-features.md) | Prev: [12-member-01-ml-ai.md](./12-member-01-ml-ai.md)
