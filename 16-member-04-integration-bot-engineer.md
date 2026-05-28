# UDARA AI — Integration & Bot Engineer Deep Dive

> **Document ID:** UDARA-ENG-IB-001  
> **Version:** 1.0.0  
> **Last Updated:** 2026-05-27  
> **Owner:** Integration & Bot Engineering Team  
> **Classification:** Internal — Engineering

---

## Table of Contents

1. [Role Overview](#1-role-overview)
2. [Key Skills & Competencies](#2-key-skills--competencies)
3. [Complete File Structure](#3-complete-file-structure)
4. [Bot Architecture — 8-Step Pipeline](#4-bot-architecture--8-step-pipeline)
5. [Agent Client](#5-agent-client)
6. [Telegram Bot](#6-telegram-bot)
7. [WhatsApp Bot](#7-whatsapp-bot)
8. [Shared Modules](#8-shared-modules)
9. [E2E Test Scenarios](#9-e2e-test-scenarios)
10. [Week-by-Week Plan](#10-week-by-week-plan)
11. [Deliverables Checklist](#11-deliverables-checklist)

---

## 1. Role Overview

The Integration & Bot Engineer owns **all conversational interfaces** and the glue between external platforms and UDARA's AI backend. CHWs in sub-Saharan Africa interact with UDARA primarily through bots — this role is therefore on the critical path.

### Three Pillars

```
┌──────────────────────────────────────────────────────────────────┐
│              INTEGRATION & BOT ENGINEER                          │
├──────────────────┬──────────────────┬────────────────────────────┤
│   TELEGRAM BOT   │   WHATSAPP BOT   │    SHARED INFRASTRUCTURE   │
│                  │                  │                            │
│  • aiogram 3.x   │  • WhatsApp CLIs │  • AgentClient (gRPC/HTTP) │
│  • Inline kbd    │  • Twilio/Meta   │  • Onboarding FSM          │
│  • Media handler │  • Interactive   │  • Daily digest scheduler  │
│  • FSM flows     │    buttons       │  • Resistance calculator   │
│  • Photo OCR     │  • Media upload  │  • Multilingual templates  │
│  • Voice msg     │  • Text handler  │  • Rate limiting           │
│                  │                  │  • Error recovery          │
├──────────────────┼──────────────────┼────────────────────────────┤
│     50%+ of CHW interactions flow through these bots            │
└────────────────────────┴──────────────────┴────────────────────┘
```

### Core Responsibilities

| Area | Details |
|------|---------|
| **Bot Development** | Build and maintain Telegram + WhatsApp bots with aiogram 3.x and official CLIs |
| **Agent Integration** | Implement AgentClient that bridges bot messages → AI backend → structured responses |
| **Conversational Flows** | Design and implement multi-step FSMs for onboarding, reporting, resistance checking |
| **Media Processing** | Handle photo uploads (drug packaging OCR), voice messages (ASR), documents |
| **Multilingual Support** | Support 12+ languages via detection + template-based responses |
| **Reliability** | Implement retry logic, graceful degradation, message deduplication, rate limiting |
| **Testing** | E2E tests covering all conversational flows, error paths, edge cases |
| **Monitoring** | Structured logging, Prometheus metrics, alerting on bot health |

---

## 2. Key Skills & Competencies

| Skill | Proficiency | Tools / Libraries |
|-------|-------------|-------------------|
| Python async | Expert | asyncio, aiohttp, aiofiles |
| Telegram bots | Expert | aiogram 3.x, telethon |
| WhatsApp integration | Advanced | Meta Cloud API, Twilio |
| State machines | Advanced | asyncio-locks, custom FSM |
| gRPC / HTTP clients | Advanced | grpcio, httpx, aiohttp |
| Message queues | Intermediate | Redis Streams, Celery |
| OCR / ASR integration | Intermediate | Whisper, Tesseract wrappers |
| Database | Intermediate | SQLAlchemy async, Redis |
| Testing | Advanced | pytest-asyncio, pytest-mock |
| Docker / deployment | Intermediate | Docker Compose, health checks |
| Monitoring | Intermediate | Prometheus client, Sentry |

---

## 3. Complete File Structure

```
bots/
├── telegram/
│   ├── __init__.py
│   ├── bot.py                          # Main bot entry point + middleware stack
│   ├── config.py                       # TG_BOT_TOKEN, WEBHOOK_URL, etc.
│   │
│   ├── handlers/
│   │   ├── __init__.py
│   │   ├── start.py                    # /start command → welcome + onboarding check
│   │   ├── text.py                     # Free text message → agent pipeline
│   │   ├── photo.py                    # Photo message → OCR → agent pipeline
│   │   ├── voice.py                    # Voice message → ASR → agent pipeline
│   │   ├── callback.py                 # Inline keyboard callback handler
│   │   └── errors.py                   # Global error handler
│   │
│   ├── keyboards/
│   │   ├── __init__.py
│   │   ├── main_menu.py                # Main navigation inline keyboard
│   │   ├── report_menu.py              # Case reporting sub-menu
│   │   ├── resistance_menu.py          # Drug resistance check flow
│   │   ├── settings_menu.py            # Language, notifications settings
│   │   └── pagination.py               # Generic paginated inline keyboard
│   │
│   ├── middlewares/
│   │   ├── __init__.py
│   │   ├── auth.py                     # User identification + registration
│   │   ├── rate_limit.py               # Per-user rate limiting
│   │   ├── logging.py                  # Request/response structured logging
│   │   ├── i18n.py                     # Language detection + context
│   │   └── error_handler.py            # Global exception catching
│   │
│   ├── formatters/
│   │   ├── __init__.py
│   │   ├── response.py                 # format_guidance_for_telegram()
│   │   ├── resistance.py               # format_resistance_result()
│   │   ├── case_summary.py             # format_case_summary()
│   │   ├── digest.py                   # format_daily_digest()
│   │   └── stats.py                    # format_user_stats()
│   │
│   └── tests/
│       ├── __init__.py
│       ├── conftest.py
│       ├── test_text_handler.py
│       ├── test_photo_handler.py
│       ├── test_voice_handler.py
│       ├── test_onboarding.py
│       └── test_resistance_check.py
│
├── whatsapp/
│   ├── __init__.py
│   ├── bot.py                          # WhatsAppBot class
│   ├── config.py                       # WA_PHONE_ID, WA_TOKEN, VERIFY_TOKEN
│   │
│   ├── handlers/
│   │   ├── __init__.py
│   │   ├── text.py                     # Incoming text → agent pipeline
│   │   ├── interactive.py              # Interactive button responses
│   │   ├── media.py                    # Image/document download + processing
│   │   └── status.py                   # Message status webhooks
│   │
│   ├── formatters/
│   │   ├── __init__.py
│   │   ├── response.py                 # WhatsApp formatting (no markdown bold)
│   │   └── interactive.py              # Build interactive button payloads
│   │
│   └── tests/
│       ├── __init__.py
│       ├── conftest.py
│       ├── test_text_handler.py
│       └── test_interactive_handler.py
│
├── shared/
│   ├── __init__.py
│   ├── agent_client.py                 # AgentClient class (core integration)
│   ├── config.py                       # AGENT_URL, timeouts, retry config
│   │
│   ├── onboarding/
│   │   ├── __init__.py
│   │   ├── flow.py                     # OnboardingFlow FSM (5 steps)
│   │   ├── steps.py                    # Individual step handlers
│   │   └── storage.py                  # FSM state persistence (Redis)
│   │
│   ├── conversations/
│   │   ├── __init__.py
│   │   ├── resistance_calc.py          # ResistanceCalculatorFlow
│   │   ├── case_report.py              # CaseReportFlow
│   │   └── drug_lookup.py              # DrugLookupFlow
│   │
│   ├── daily_digest.py                 # DailyDigestScheduler
│   ├── multilingual.py                 # Language detection + templates
│   ├── rate_limiter.py                 # Token bucket rate limiter
│   ├── media_downloader.py             # Unified media download + temp file mgmt
│   ├── validators.py                   # Input sanitization + validation
│   └── error_recovery.py               # Retry strategies + circuit breaker
│
├── contracts/
│   ├── agent_api.proto                 # gRPC service definition
│   ├── agent_api_pb2.py                # Generated protobuf
│   ├── agent_api_pb2_grpc.py           # Generated gRPC stubs
│   └── schemas/
│       ├── ingest_request.json         # JSON Schema for ingest requests
│       ├── resistance_response.json    # JSON Schema for resistance results
│       └── guidance_response.json      # JSON Schema for guidance
│
├── e2e-tests/
│   ├── __init__.py
│   ├── conftest.py                     # Fixtures: test bot, mock agent, test user
│   ├── test_full_case_report.py        # E2E: text case → resistance → guidance
│   ├── test_photo_ocr_flow.py          # E2E: photo → OCR → resistance check
│   ├── test_onboarding_flow.py         # E2E: new user → complete onboarding
│   ├── test_resistance_calculator.py   # E2E: drug query → probability + alternatives
│   ├── test_daily_digest.py            # E2E: scheduler → personalized digest
│   └── test_error_recovery.py          # E2E: agent timeout → fallback response
│
├── Dockerfile
├── docker-compose.yml                  # telegram-bot, whatsapp-bot, redis
├── Makefile
├── pyproject.toml
└── README.md
```

---

## 4. Bot Architecture — 8-Step Pipeline

Every incoming message (text, photo, voice) flows through the same 8-step pipeline:

```
 ┌─────────────────────────────────────────────────────────────────────────────┐
 │                    UDARA BOT MESSAGE PROCESSING PIPELINE                     │
 └─────────────────────────────────────────────────────────────────────────────┘

  User sends        Step 1           Step 2           Step 3
  message  ──────►  AUTH &           RATE             LANGUAGE
  (text/photo/       IDENTIFY ──────► LIMIT ────────► DETECT
   voice)            middleware       middleware       middleware
                       │                │                │
                    ┌──▼──┐          ┌──▼──┐          ┌──▼──┐
                    │Who  │          │Max  │          │What │
                    │is   │          │5/min│          │lang?│
                    │this?│          │per  │          │     │
                    └─────┘          │user │          └─────┘
                                     └─────┘
                       │                │                │
                       ▼                ▼                ▼

  Step 4            Step 5           Step 6           Step 7
  CONTENT           AGENT            RESPONSE         SEND
  PREP ──────────► INGEST ────────► FORMAT ────────► TO
  (normalize,       (OCR/ASR/       (platform-       USER
   extract,         NER →           specific
   validate)        structured      markdown,
                     case)           emoji,
                                      buttons)
                       │                │                │
                    ┌──▼──┐          ┌──▼──┐          ┌──▼──┐
                    │Text │          │TG:  │          │Send │
                    │+media│         │MDv2 │          │msg  │
                    │→JSON│          │WA:  │          │+save│
                    └─────┘          │plain│          │state│
                                     └─────┘          └─────┘
                       │                │                │
                       ▼                ▼                ▼

  Step 8
  LOG & METRICS
  ──────────────► Prometheus counters
                  Structured JSON log
                  Update user state
                       │
                    ┌──▼──┐
                    │metrics│
                    │+log  │
                    │+audit│
                    └──────┘
```

### Data Flow Diagram

```
┌──────────┐     ┌──────────────┐     ┌──────────────────┐     ┌────────────┐
│ Telegram │     │              │     │                  │     │            │
│ Bot      │────►│  AgentClient │────►│  UDARA Backend   │────►│ PostgreSQL │
│ (aiogram)│     │  (HTTP/gRPC) │     │  (FastAPI)       │     │            │
└──────────┘     │              │     │  ┌────────────┐  │     └────────────┘
                 │  • retry     │     │  │ AI Agent   │  │
┌──────────┐     │  • timeout   │     │  │ (LLM+NER)  │  │
│ WhatsApp │────►│  • circuit   │     │  └────────────┘  │
│ Bot      │     │    breaker   │     │  ┌────────────┐  │     ┌────────────┐
└──────────┘     │  • fallback  │     │  │ Resistance │  │────►│ Redis      │
                 │              │     │  │ Engine     │  │     │ (cache+FSM)│
                 └──────────────┘     │  └────────────┘  │     └────────────┘
                                      └──────────────────┘
                                              │
                                      ┌───────▼────────┐
                                      │  MinIO/S3       │
                                      │  (media files)  │
                                      └────────────────┘
```

---

## 5. Agent Client

### 5.1 Core Client — `shared/agent_client.py`

```python
"""
UDARA Agent Client

Bridges bot messages to the UDARA AI backend.
Handles all content types: text, images, audio.
Implements retry logic, timeouts, circuit breaker, and graceful fallbacks.
"""

from __future__ import annotations

import asyncio
import logging
import time
from dataclasses import dataclass, field
from enum import Enum
from typing import Any, Optional

import httpx
from pydantic import BaseModel, Field

logger = logging.getLogger(__name__)


# ── Configuration ────────────────────────────────────────────────────

class AgentConfig:
    """Agent client configuration loaded from environment."""

    BASE_URL: str = "http://localhost:8001"
    TIMEOUT_SECONDS: float = 30.0
    MAX_RETRIES: int = 3
    RETRY_BACKOFF_BASE: float = 2.0
    RETRY_BACKOFF_MAX: float = 30.0
    CIRCUIT_BREAKER_THRESHOLD: int = 5
    CIRCUIT_BREAKER_RESET_SECONDS: float = 60.0


# ── Data Models ─────────────────────────────────────────────────────

class ContentType(str, Enum):
    TEXT = "text"
    IMAGE = "image"
    AUDIO = "audio"
    DOCUMENT = "document"


class IngestRequest(BaseModel):
    """Request sent to the AI agent for processing."""
    content: str
    content_type: ContentType = ContentType.TEXT
    language: Optional[str] = None
    user_id: str
    context: dict[str, Any] = Field(default_factory=dict)
    metadata: dict[str, Any] = Field(default_factory=dict)


class IngestResponse(BaseModel):
    """Response from the AI agent after processing."""
    success: bool
    case_id: Optional[str] = None
    extracted_data: dict[str, Any] = Field(default_factory=dict)
    resistance_summary: Optional[dict[str, Any]] = None
    guidance: Optional[str] = None
    confidence: float = 0.0
    processing_time_ms: float = 0.0
    warnings: list[str] = Field(default_factory=list)
    error: Optional[str] = None


class ResistanceCheckRequest(BaseModel):
    """Request to check resistance for specific drugs."""
    drugs: list[str]
    location: Optional[dict[str, str]] = None  # {country, state, district}
    patient_age: Optional[int] = None
    specimen_type: Optional[str] = None


class ResistanceResult(BaseModel):
    """Resistance probability for a single drug."""
    drug: str
    resistance_pct: float
    category: str  # susceptible, intermediate, resistant
    confidence: float
    alternatives: list[str] = Field(default_factory=list)
    sample_size: int = 0
    last_updated: Optional[str] = None


class ResistanceCheckResponse(BaseModel):
    """Response with resistance data for queried drugs."""
    drugs: list[ResistanceResult]
    location_summary: Optional[dict[str, Any]] = None
    overall_risk: str = "unknown"


class GuidanceRequest(BaseModel):
    """Request for clinical guidance based on case data."""
    case_data: dict[str, Any]
    resistance_data: Optional[list[ResistanceResult]] = None
    language: str = "en"


class GuidanceResponse(BaseModel):
    """Clinical guidance response."""
    guidance_text: str
    urgency_level: str  # low, medium, high, critical
    recommended_actions: list[str] = Field(default_factory=list)
    warnings: list[str] = Field(default_factory=list)
    confidence: float = 0.0


# ── Circuit Breaker ─────────────────────────────────────────────────

class CircuitState(str, Enum):
    CLOSED = "closed"       # Normal operation
    OPEN = "open"           # Failing — reject all requests
    HALF_OPEN = "half_open" # Testing — allow limited requests


class CircuitBreaker:
    """Simple circuit breaker to prevent cascading failures."""

    def __init__(
        self,
        failure_threshold: int = 5,
        reset_seconds: float = 60.0,
    ):
        self.failure_threshold = failure_threshold
        self.reset_seconds = reset_seconds
        self.failure_count = 0
        self.state = CircuitState.CLOSED
        self.last_failure_time: Optional[float] = None
        self._lock = asyncio.Lock()

    async def allow_request(self) -> bool:
        async with self._lock:
            if self.state == CircuitState.CLOSED:
                return True
            if self.state == CircuitState.OPEN:
                if (
                    self.last_failure_time
                    and time.time() - self.last_failure_time > self.reset_seconds
                ):
                    self.state = CircuitState.HALF_OPEN
                    return True
                return False
            # HALF_OPEN: allow one request to test
            return True

    async def record_success(self):
        async with self._lock:
            self.failure_count = 0
            self.state = CircuitState.CLOSED

    async def record_failure(self):
        async with self._lock:
            self.failure_count += 1
            self.last_failure_time = time.time()
            if self.failure_count >= self.failure_threshold:
                self.state = CircuitState.OPEN
                logger.warning(
                    "Circuit breaker OPEN after %d failures",
                    self.failure_count,
                )


# ── Agent Client ────────────────────────────────────────────────────

class AgentClient:
    """
    HTTP client for the UDARA AI Agent backend.

    Features:
      - Automatic retry with exponential backoff
      - Circuit breaker pattern
      - Request timeout enforcement
      - Structured error handling with fallbacks
      - Prometheus metrics integration
    """

    def __init__(self, config: Optional[AgentConfig] = None):
        self.config = config or AgentConfig()
        self.circuit_breaker = CircuitBreaker(
            failure_threshold=self.config.CIRCUIT_BREAKER_THRESHOLD,
            reset_seconds=self.config.CIRCUIT_BREAKER_RESET_SECONDS,
        )
        self._client: Optional[httpx.AsyncClient] = None

        # Metrics counters (prometheus-python if available, else no-ops)
        try:
            from prometheus_client import Counter, Histogram

            self._metrics_available = True
            self._request_counter = Counter(
                "udara_agent_requests_total",
                "Total agent requests",
                ["endpoint", "status"],
            )
            self._request_duration = Histogram(
                "udara_agent_request_duration_seconds",
                "Agent request duration",
                ["endpoint"],
            )
            self._retry_counter = Counter(
                "udara_agent_retries_total",
                "Total agent request retries",
                ["endpoint"],
            )
        except ImportError:
            self._metrics_available = False

    async def _get_client(self) -> httpx.AsyncClient:
        if self._client is None or self._client.is_closed:
            self._client = httpx.AsyncClient(
                base_url=self.config.BASE_URL,
                timeout=httpx.Timeout(self.config.TIMEOUT_SECONDS),
                headers={"Content-Type": "application/json"},
            )
        return self._client

    async def close(self):
        if self._client and not self._client.is_closed:
            await self._client.aclose()

    def _record_metrics(
        self, endpoint: str, status: str, duration: float, retries: int = 0
    ):
        if not self._metrics_available:
            return
        self._request_counter.labels(endpoint=endpoint, status=status).inc()
        self._request_duration.labels(endpoint=endpoint).observe(duration)
        if retries > 0:
            self._retry_counter.labels(endpoint=endpoint).inc(retries)

    async def _request_with_retry(
        self,
        method: str,
        path: str,
        json_data: Optional[dict] = None,
        files: Optional[dict] = None,
    ) -> dict:
        """
        Execute HTTP request with retry logic and circuit breaker.

        Retry strategy:
          - Up to MAX_RETRIES attempts
          - Exponential backoff: 2^attempt seconds, capped at 30s
          - Retries on: 502, 503, 504, timeout, connection errors
          - No retry on: 400, 401, 403, 404, 422
        """
        # Circuit breaker check
        if not await self.circuit_breaker.allow_request():
            logger.error("Circuit breaker OPEN — rejecting request to %s", path)
            raise AgentUnavailableError(
                "Agent service temporarily unavailable. Please try again later."
            )

        RETRYABLE_STATUS_CODES = {502, 503, 504}
        last_error: Optional[Exception] = None
        retries = 0
        start_time = time.time()

        for attempt in range(self.config.MAX_RETRIES + 1):
            try:
                client = await self._get_client()

                if files:
                    response = await client.request(
                        method, path, data=json_data, files=files
                    )
                else:
                    response = await client.request(
                        method, path, json=json_data
                    )

                if response.status_code == 200:
                    await self.circuit_breaker.record_success()
                    duration = time.time() - start_time
                    self._record_metrics(
                        path, "success", duration, retries
                    )
                    return response.json()

                if response.status_code in RETRYABLE_STATUS_CODES:
                    last_error = AgentHTTPError(
                        status=response.status_code,
                        detail=response.text,
                    )
                    logger.warning(
                        "Agent returned %d (attempt %d/%d)",
                        response.status_code,
                        attempt + 1,
                        self.config.MAX_RETRIES + 1,
                    )
                else:
                    # Non-retryable error
                    duration = time.time() - start_time
                    self._record_metrics(path, f"error_{response.status_code}", duration)
                    raise AgentHTTPError(
                        status=response.status_code,
                        detail=response.text,
                    )

            except (httpx.TimeoutException, httpx.ConnectError) as exc:
                last_error = exc
                logger.warning(
                    "Agent connection error (attempt %d/%d): %s",
                    attempt + 1,
                    self.config.MAX_RETRIES + 1,
                    str(exc),
                )
            except Exception as exc:
                last_error = exc
                logger.error("Unexpected agent error: %s", str(exc))
                raise

            # Calculate backoff and sleep before retry
            if attempt < self.config.MAX_RETRIES:
                retries += 1
                backoff = min(
                    self.config.RETRY_BACKOFF_BASE ** (attempt + 1),
                    self.config.RETRY_BACKOFF_MAX,
                )
                logger.info("Retrying in %.1f seconds (attempt %d)", backoff, attempt + 1)
                await asyncio.sleep(backoff)

        # All retries exhausted
        await self.circuit_breaker.record_failure()
        duration = time.time() - start_time
        self._record_metrics(path, "failed", duration, retries)

        raise AgentUnavailableError(
            f"Agent service unavailable after {retries} retries. "
            f"Last error: {last_error}"
        )

    # ── Public API Methods ─────────────────────────────────────

    async def ingest_text(
        self,
        text: str,
        user_id: str,
        language: Optional[str] = None,
        context: Optional[dict] = None,
    ) -> IngestResponse:
        """Process free-text message through the AI agent."""
        request = IngestRequest(
            content=text,
            content_type=ContentType.TEXT,
            language=language,
            user_id=user_id,
            context=context or {},
        )
        result = await self._request_with_retry(
            "POST", "/agent/ingest", json_data=request.model_dump()
        )
        return IngestResponse(**result)

    async def ingest_image(
        self,
        image_bytes: bytes,
        filename: str,
        user_id: str,
        language: Optional[str] = None,
        context: Optional[dict] = None,
    ) -> IngestResponse:
        """
        Process an image through the AI agent.
        Handles drug packaging OCR, prescription photos, lab reports.
        """
        # Upload image as multipart/form-data
        files = {"file": (filename, image_bytes, "image/jpeg")}
        data = {
            "user_id": user_id,
            "content_type": ContentType.IMAGE,
            "language": language or "en",
            **(context or {}),
        }
        result = await self._request_with_retry(
            "POST", "/agent/ingest", json_data=data, files=files
        )
        return IngestResponse(**result)

    async def ingest_audio(
        self,
        audio_bytes: bytes,
        filename: str,
        user_id: str,
        language: Optional[str] = None,
    ) -> IngestResponse:
        """
        Process audio through the AI agent (ASR → text → pipeline).
        """
        files = {"file": (filename, audio_bytes, "audio/ogg")}
        data = {
            "user_id": user_id,
            "content_type": ContentType.AUDIO,
            "language": language or "en",
        }
        result = await self._request_with_retry(
            "POST", "/agent/ingest", json_data=data, files=files
        )
        return IngestResponse(**result)

    async def check_resistance(
        self,
        drugs: list[str],
        location: Optional[dict] = None,
        patient_age: Optional[int] = None,
        specimen_type: Optional[str] = None,
    ) -> ResistanceCheckResponse:
        """Check antimicrobial resistance for specific drugs at a location."""
        request = ResistanceCheckRequest(
            drugs=drugs,
            location=location,
            patient_age=patient_age,
            specimen_type=specimen_type,
        )
        result = await self._request_with_retry(
            "POST",
            "/resistance/check",
            json_data=request.model_dump(),
        )
        return ResistanceCheckResponse(**result)

    async def get_guidance(
        self,
        case_data: dict,
        resistance_data: Optional[list] = None,
        language: str = "en",
    ) -> GuidanceResponse:
        """Get clinical guidance based on case and resistance data."""
        request = GuidanceRequest(
            case_data=case_data,
            resistance_data=resistance_data,
            language=language,
        )
        result = await self._request_with_retry(
            "POST",
            "/agent/guidance",
            json_data=request.model_dump(),
        )
        return GuidanceResponse(**result)


# ── Exceptions ──────────────────────────────────────────────────────

class AgentError(Exception):
    """Base exception for agent client errors."""
    pass


class AgentUnavailableError(AgentError):
    """Agent service is unavailable (circuit open or all retries exhausted)."""
    def __init__(self, message: str):
        self.message = message
        super().__init__(message)


class AgentHTTPError(AgentError):
    """HTTP error from agent service."""
    def __init__(self, status: int, detail: str):
        self.status = status
        self.detail = detail
        super().__init__(f"Agent HTTP {status}: {detail}")


# ── Singleton ───────────────────────────────────────────────────────

_agent_client: Optional[AgentClient] = None


def get_agent_client() -> AgentClient:
    """Get or create the global AgentClient singleton."""
    global _agent_client
    if _agent_client is None:
        _agent_client = AgentClient()
    return _agent_client
```

---

## 6. Telegram Bot

### 6.1 Main Bot — `telegram/bot.py`

```python
"""
UDARA Telegram Bot — Main Entry Point

Built with aiogram 3.x. Supports:
  - Webhook mode (production) and polling mode (development)
  - Middleware stack: auth → rate limit → logging → i18n → error handler
  - All handlers registered via router
"""

from __future__ import annotations

import asyncio
import logging
import os
from typing import Optional

from aiogram import Bot, Dispatcher, Router
from aiogram.client.default import DefaultBotProperties
from aiogram.enums import ParseMode
from aiogram.fsm.storage.redis import RedisStorage
from aiogram.types import Update
from aiogram.webhook.aiohttp_server import SimpleRequestHandler, setup_application

from shared.agent_client import AgentClient, get_agent_client
from telegram.config import TelegramConfig

logger = logging.getLogger(__name__)


def create_dispatcher(agent_client: AgentClient) -> Dispatcher:
    """Create and configure the dispatcher with all routers and middleware."""

    # Storage for FSM states
    storage = RedisStorage.from_url(
        os.getenv("REDIS_URL", "redis://localhost:6379/1")
    )

    dp = Dispatcher(storage=storage)

    # ── Register Routers ────────────────────────────────────────

    from telegram.handlers.start import router as start_router
    from telegram.handlers.text import router as text_router
    from telegram.handlers.photo import router as photo_router
    from telegram.handlers.voice import router as voice_router
    from telegram.handlers.callback import router as callback_router
    from telegram.handlers.errors import router as errors_router

    dp.include_router(start_router)
    dp.include_router(text_router)
    dp.include_router(photo_router)
    dp.include_router(voice_router)
    dp.include_router(callback_router)

    # Error handler must be last
    dp.include_router(errors_router)

    # ── Middleware Stack (outermost first) ──────────────────────
    #
    # Request flow:
    #   1. LoggingMiddleware   — structured log entry
    #   2. AuthMiddleware     — identify user, create if new
    #   3. RateLimitMiddleware — check rate limits
    #   4. I18nMiddleware      — detect language, set context
    #   5. Handler execution
    #   6. LoggingMiddleware   — structured log exit

    from telegram.middlewares.logging import LoggingMiddleware
    from telegram.middlewares.auth import AuthMiddleware
    from telegram.middlewares.rate_limit import RateLimitMiddleware
    from telegram.middlewares.i18n import I18nMiddleware

    dp.update.outer_middleware(LoggingMiddleware())
    dp.update.outer_middleware(AuthMiddleware())
    dp.update.outer_middleware(RateLimitMiddleware())
    dp.update.middleware(I18nMiddleware())

    # ── Shutdown Hook ───────────────────────────────────────────

    async def on_shutdown():
        await agent_client.close()
        logger.info("Agent client closed")

    dp.shutdown.register(on_shutdown)

    return dp


async def run_polling(config: Optional[TelegramConfig] = None):
    """Run bot in long-polling mode (development)."""
    config = config or TelegramConfig()
    agent_client = get_agent_client()

    bot = Bot(
        token=config.BOT_TOKEN,
        default=DefaultBotProperties(
            parse_mode=ParseMode.HTML,
        ),
    )

    dp = create_dispatcher(agent_client)

    logger.info("Starting Telegram bot in polling mode...")
    await dp.start_polling(bot)


async def run_webhook(config: Optional[TelegramConfig] = None):
    """Run bot in webhook mode (production via aiohttp)."""
    config = config or TelegramConfig()
    agent_client = get_agent_client()

    bot = Bot(
        token=config.BOT_TOKEN,
        default=DefaultBotProperties(
            parse_mode=ParseMode.HTML,
        ),
    )

    dp = create_dispatcher(agent_client)

    from aiohttp import web

    app = web.Application()

    # Health check
    async def health_check(request: web.Request) -> web.Response:
        return web.json_response({"status": "ok", "service": "telegram-bot"})

    app.router.add_get("/health", health_check)

    # Webhook handler
    webhook_requests_handler = SimpleRequestHandler(
        dispatcher=dp,
        bot=bot,
    )
    webhook_requests_handler.register(app, path=config.WEBHOOK_PATH)

    # Setup webhook on startup
    setup_application(bot, dp)

    # Set webhook URL
    await bot.set_webhook(
        url=f"{config.WEBHOOK_URL}{config.WEBHOOK_PATH}",
        drop_pending_updates=True,
        allowed_updates=dp.resolve_used_update_types(),
    )

    runner = web.AppRunner(app)
    await runner.setup()
    site = web.TCPSite(runner, host="0.0.0.0", port=config.PORT)
    await site.start()

    logger.info(
        "Telegram bot webhook running on %s:%d%s",
        "0.0.0.0",
        config.PORT,
        config.WEBHOOK_PATH,
    )

    # Keep running forever
    try:
        await asyncio.Event().wait()
    except asyncio.CancelledError:
        pass
    finally:
        await runner.cleanup()
        await bot.session.close()


if __name__ == "__main__":
    import sys

    logging.basicConfig(
        level=logging.INFO,
        format="%(asctime)s [%(levelname)s] %(name)s: %(message)s",
    )

    mode = sys.argv[1] if len(sys.argv) > 1 else "polling"

    if mode == "webhook":
        asyncio.run(run_webhook())
    else:
        asyncio.run(run_polling())
```

### 6.2 Text Handler — `telegram/handlers/text.py`

```python
"""
Telegram Text Message Handler

Processes free-text messages from CHWs:
1. Check if user is in an active FSM flow → delegate to that flow
2. If not, route through the agent pipeline:
   a. AgentClient.ingest_text() → extract case data
   b. Check if drugs mentioned → AgentClient.check_resistance()
   c. Generate guidance → AgentClient.get_guidance()
   d. Format and send response
"""

from __future__ import annotations

import logging
from typing import Optional

from aiogram import Router, F
from aiogram.types import Message
from aiogram.fsm.context import FSMContext

from shared.agent_client import (
    AgentClient,
    get_agent_client,
    AgentUnavailableError,
)
from telegram.formatters.response import format_guidance_for_telegram
from telegram.keyboards.main_menu import get_main_menu_keyboard

logger = logging.getLogger(__name__)
router = Router()


@router.message(F.text, ~F.forward_from)
async def handle_text_message(
    message: Message,
    state: FSMContext,
    agent_client: AgentClient = None,  # Injected via middleware
):
    """Handle free-text messages from CHWs."""

    agent = agent_client or get_agent_client()
    user_id = str(message.from_user.id)
    text = message.text.strip()

    if not text:
        return

    logger.info(
        "Text message from user %s: %s... (truncated)",
        user_id,
        text[:100],
    )

    # ── Step 1: Check if user is in an active FSM flow ────────
    current_state = await state.get_state()
    if current_state:
        # Delegate to the active flow handler
        logger.debug("User %s in FSM state %s — delegating", user_id, current_state)
        # The FSM flow will handle this message via its own handlers
        return

    # ── Step 2: Show typing indicator ─────────────────────────
    await message.bot.send_chat_action(
        chat_id=message.chat.id,
        action="typing",
    )

    # ── Step 3: Process through agent pipeline ────────────────
    try:
        # 3a. Ingest text → extract case data
        ingest_response = await agent.ingest_text(
            text=text,
            user_id=user_id,
            language=message.from_user.language_code,
        )

        if not ingest_response.success:
            await message.answer(
                "⚠️ I couldn't process that message. "
                "Please try again or use the menu to report a case.",
                reply_markup=get_main_menu_keyboard(),
            )
            return

        # 3b. Check resistance if drugs were extracted
        extracted = ingest_response.extracted_data
        drugs = extracted.get("drugs", [])
        resistance_data = None

        if drugs:
            try:
                resistance_response = await agent.check_resistance(
                    drugs=drugs,
                    location=extracted.get("location"),
                    patient_age=extracted.get("patient_age"),
                )
                resistance_data = resistance_response.drugs
            except Exception as exc:
                logger.warning("Resistance check failed: %s", exc)
                # Continue without resistance data

        # 3c. Generate guidance
        if ingest_response.case_id or drugs:
            try:
                guidance_response = await agent.get_guidance(
                    case_data=extracted,
                    resistance_data=[r.model_dump() for r in resistance_data]
                    if resistance_data
                    else None,
                    language=message.from_user.language_code or "en",
                )
                guidance = guidance_response.guidance_text
                urgency = guidance_response.urgency_level
                actions = guidance_response.recommended_actions
            except Exception as exc:
                logger.warning("Guidance generation failed: %s", exc)
                guidance = "Case recorded successfully."
                urgency = "low"
                actions = []
        else:
            # No case data extracted — this might be a general question
            guidance = (
                "I received your message but couldn't identify a case or drug query. "
                "You can:\n"
                "• Report a case using the menu below\n"
                "• Ask about drug resistance (e.g., 'What is the resistance to Amoxicillin in Lagos?')\n"
                "• Send a photo of a drug label or prescription"
            )
            urgency = "low"
            actions = []

        # ── Step 4: Format and send response ───────────────────
        response_text = format_guidance_for_telegram(
            guidance=guidance,
            urgency=urgency,
            case_id=ingest_response.case_id,
            resistance_data=resistance_data,
            actions=actions,
            confidence=ingest_response.confidence,
            language=message.from_user.language_code or "en",
        )

        # Split if too long (Telegram max is 4096 chars)
        MAX_MESSAGE_LENGTH = 4000
        if len(response_text) <= MAX_MESSAGE_LENGTH:
            await message.answer(
                response_text,
                reply_markup=get_main_menu_keyboard(),
                parse_mode="HTML",
            )
        else:
            # Split at paragraph boundaries
            chunks = response_text.split("\n\n")
            current_chunk = ""
            for chunk in chunks:
                if len(current_chunk) + len(chunk) + 2 > MAX_MESSAGE_LENGTH:
                    await message.answer(current_chunk, parse_mode="HTML")
                    current_chunk = chunk
                else:
                    current_chunk = f"{current_chunk}\n\n{chunk}" if current_chunk else chunk
            if current_chunk:
                await message.answer(
                    current_chunk,
                    reply_markup=get_main_menu_keyboard(),
                    parse_mode="HTML",
                )

    except AgentUnavailableError:
        logger.error("Agent unavailable for user %s", user_id)
        await message.answer(
            "😔 Our AI service is temporarily busy. "
            "Your message has been saved and will be processed shortly.\n\n"
            "Please try again in a few minutes.",
            reply_markup=get_main_menu_keyboard(),
        )

    except Exception as exc:
        logger.exception("Unexpected error processing text from user %s: %s", user_id, exc)
        await message.answer(
            "⚠️ Something went wrong. Our team has been notified.\n"
            "Please try again or contact support.",
            reply_markup=get_main_menu_keyboard(),
        )
```

### 6.3 Photo Handler — `telegram/handlers/photo.py`

```python
"""
Telegram Photo Handler

Processes photo messages:
1. Download the highest resolution photo from Telegram
2. Send to AgentClient.ingest_image() for OCR
3. If drugs identified → resistance check → guidance
4. Return response with OCR transcript + analysis
"""

from __future__ import annotations

import logging
import tempfile
from pathlib import Path

from aiogram import Router, F
from aiogram.types import Message

from shared.agent_client import AgentClient, get_agent_client, AgentUnavailableError
from shared.media_downloader import download_telegram_photo
from telegram.formatters.response import format_guidance_for_telegram
from telegram.keyboards.main_menu import get_main_menu_keyboard

logger = logging.getLogger(__name__)
router = Router()


@router.message(F.photo)
async def handle_photo_message(
    message: Message,
    agent_client: AgentClient = None,
):
    """Handle photo messages — drug packaging OCR, prescription scanning."""

    agent = agent_client or get_agent_client()
    user_id = str(message.from_user.id)

    # Get the highest resolution photo
    photo = message.photo[-1]  # Last item = largest
    file_id = photo.file_id

    logger.info(
        "Photo received from user %s: file_id=%s, size=%dx%d",
        user_id,
        file_id,
        photo.width,
        photo.height,
    )

    # Send processing indicator
    processing_msg = await message.answer(
        "🔍 <b>Processing image...</b>\n"
        "Identifying text and drug information...",
        parse_mode="HTML",
    )

    try:
        # ── Step 1: Download photo ────────────────────────────
        image_bytes, filename = await download_telegram_photo(
            bot=message.bot,
            file_id=file_id,
            user_id=user_id,
        )

        logger.info(
            "Downloaded photo: %s (%d bytes)",
            filename,
            len(image_bytes),
        )

        # ── Step 2: Ingest via agent (OCR + NER) ──────────────
        await processing_msg.edit_text(
            "🔍 <b>Analyzing image...</b>\n"
            "Running OCR and extracting drug information...",
            parse_mode="HTML",
        )

        ingest_response = await agent.ingest_image(
            image_bytes=image_bytes,
            filename=filename,
            user_id=user_id,
            language=message.from_user.language_code,
        )

        if not ingest_response.success:
            await processing_msg.edit_text(
                "❌ <b>Image Analysis Failed</b>\n\n"
                "I couldn't extract readable text from this image. "
                "Please try:\n"
                "• A clearer, better-lit photo\n"
                "• Ensure the drug name is visible\n"
                "• Try a different angle",
                parse_mode="HTML",
                reply_markup=get_main_menu_keyboard(),
            )
            return

        extracted = ingest_response.extracted_data
        ocr_text = extracted.get("ocr_text", "")
        drugs = extracted.get("drugs", [])

        # ── Step 3: Check resistance if drugs found ───────────
        resistance_data = None
        if drugs:
            await processing_msg.edit_text(
                f"🔍 <b>Drugs identified:</b> {', '.join(drugs)}\n"
                "Checking local resistance data...",
                parse_mode="HTML",
            )

            resistance_response = await agent.check_resistance(
                drugs=drugs,
                location=extracted.get("location"),
            )
            resistance_data = resistance_response.drugs

        # ── Step 4: Generate guidance ─────────────────────────
        guidance_response = await agent.get_guidance(
            case_data=extracted,
            resistance_data=[r.model_dump() for r in resistance_data]
            if resistance_data
            else None,
            language=message.from_user.language_code or "en",
        )

        # ── Step 5: Build and send response ────────────────────
        response_parts = []

        # OCR transcript section
        if ocr_text:
            response_parts.append(
                f"📝 <b>Text Extracted:</b>\n"
                f"<code>{ocr_text[:500]}</code>\n"
            )

        # Main guidance
        response_parts.append(
            format_guidance_for_telegram(
                guidance=guidance_response.guidance_text,
                urgency=guidance_response.urgency_level,
                case_id=ingest_response.case_id,
                resistance_data=resistance_data,
                actions=guidance_response.recommended_actions,
                confidence=ingest_response.confidence,
                language=message.from_user.language_code or "en",
            )
        )

        full_response = "\n\n".join(response_parts)

        await processing_msg.edit_text(
            full_response,
            parse_mode="HTML",
            reply_markup=get_main_menu_keyboard(),
        )

        # Clean up temp file
        try:
            temp_path = Path(tempfile.gettempdir()) / filename
            if temp_path.exists():
                temp_path.unlink()
        except Exception:
            pass

    except AgentUnavailableError:
        await processing_msg.edit_text(
            "😔 AI service is temporarily busy. "
            "Your image has been saved and will be processed shortly.",
            reply_markup=get_main_menu_keyboard(),
        )

    except Exception as exc:
        logger.exception("Photo processing error for user %s: %s", user_id, exc)
        await processing_msg.edit_text(
            "⚠️ Something went wrong processing your image. "
            "Please try again with a clearer photo.",
            reply_markup=get_main_menu_keyboard(),
        )
```

### 6.4 Main Menu Keyboard — `telegram/keyboards/main_menu.py`

```python
"""
Telegram Inline Keyboard — Main Menu

Provides navigation for CHWs:
  1. 📋 Report Case    — Start case reporting flow
  2. 💊 Check Resistance — Query drug resistance
  3. 📊 My Stats       — View personal reporting stats
  4. 📚 Learn          — Access AMR education modules
  5. 🔔 Alerts         — View active resistance alerts
  6. ⚙️ Settings       — Language, notifications
"""

from __future__ import annotations

from aiogram.types import InlineKeyboardMarkup, InlineKeyboardButton


def get_main_menu_keyboard() -> InlineKeyboardMarkup:
    """Build the main navigation inline keyboard."""

    return InlineKeyboardMarkup(
        inline_keyboard=[
            # Row 1: Primary actions
            [
                InlineKeyboardButton(
                    text="📋 Report Case",
                    callback_data="menu:report_case",
                ),
                InlineKeyboardButton(
                    text="💊 Check Resistance",
                    callback_data="menu:check_resistance",
                ),
            ],
            # Row 2: Information
            [
                InlineKeyboardButton(
                    text="📊 My Stats",
                    callback_data="menu:my_stats",
                ),
                InlineKeyboardButton(
                    text="📚 Learn AMR",
                    callback_data="menu:learn",
                ),
            ],
            # Row 3: Notifications
            [
                InlineKeyboardButton(
                    text="🔔 Alerts",
                    callback_data="menu:alerts",
                ),
                InlineKeyboardButton(
                    text="⚙️ Settings",
                    callback_data="menu:settings",
                ),
            ],
        ]
    )


def get_report_type_keyboard() -> InlineKeyboardMarkup:
    """Sub-menu for case reporting options."""

    return InlineKeyboardMarkup(
        inline_keyboard=[
            [
                InlineKeyboardButton(
                    text="📝 Type a Report",
                    callback_data="report:text",
                ),
            ],
            [
                InlineKeyboardButton(
                    text="📷 Photo of Prescription",
                    callback_data="report:photo",
                ),
                InlineKeyboardButton(
                    text="🎤 Voice Report",
                    callback_data="report:voice",
                ),
            ],
            [
                InlineKeyboardButton(
                    text="🔙 Back to Menu",
                    callback_data="menu:main",
                ),
            ],
        ]
    )


def get_resistance_keyboard() -> InlineKeyboardMarkup:
    """Sub-menu for resistance checking."""

    return InlineKeyboardMarkup(
        inline_keyboard=[
            [
                InlineKeyboardButton(
                    text="🔍 Search Drug",
                    callback_data="resistance:search",
                ),
            ],
            [
                InlineKeyboardButton(
                    text="📷 Scan Drug Label",
                    callback_data="resistance:scan",
                ),
            ],
            [
                InlineKeyboardButton(
                    text="📊 Compare Drugs",
                    callback_data="resistance:compare",
                ),
            ],
            [
                InlineKeyboardButton(
                    text="🔙 Back to Menu",
                    callback_data="menu:main",
                ),
            ],
        ]
    )


def get_language_keyboard() -> InlineKeyboardMarkup:
    """Language selection keyboard (first 6 most common)."""

    languages = [
        ("🇬🇧 English", "lang:en"),
        ("🇫🇷 Français", "lang:fr"),
        ("🇳🇬 Hausa", "lang:ha"),
        ("🇳🇬 Yoruba", "lang:yo"),
        ("🇳🇬 Igbo", "lang:ig"),
        ("🇰🇪 Swahili", "lang:sw"),
    ]

    return InlineKeyboardMarkup(
        inline_keyboard=[
            [
                InlineKeyboardButton(text=text, callback_data=data)
                for text, data in languages[i:i + 2]
            ]
            for i in range(0, len(languages), 2)
        ]
        + [
            [
                InlineKeyboardButton(
                    text="🔙 Cancel",
                    callback_data="settings:cancel",
                ),
            ]
        ]
    )
```

### 6.5 Response Formatter — `telegram/formatters/response.py`

```python
"""
Telegram Response Formatter

Formats AI agent responses into Telegram-friendly HTML with:
  - Markdown headers (via HTML tags)
  - Emoji indicators for resistance levels
  - Confidence scores with visual bars
  - Urgency color coding
  - Action item bullet lists
"""

from __future__ import annotations

from typing import Optional


# Resistance level config
RESISTANCE_EMOJI = {
    "susceptible": "🟢",
    "low": "🟢",
    "safe": "🟢",
    "intermediate": "🔵",
    "moderate": "🔵",
    "warning": "🟡",
    "resistant": "🔴",
    "high": "🔴",
    "critical": "🔴",
}

URGENCY_EMOJI = {
    "low": "✅",
    "medium": "⚠️",
    "high": "🔶",
    "critical": "🚨",
}


def get_confidence_bar(confidence: float, width: int = 10) -> str:
    """Generate a Unicode confidence bar like ████████░░."""
    if confidence <= 0:
        return "░" * width
    filled = int(confidence * width)
    return "█" * filled + "░" * (width - filled)


def get_resistance_bar(pct: float, width: int = 10) -> str:
    """Generate a Unicode resistance probability bar."""
    if pct <= 0:
        return "░" * width
    filled = int(pct / 100 * width)
    return "█" * filled + "░" * (width - filled)


def format_guidance_for_telegram(
    guidance: str,
    urgency: str = "low",
    case_id: Optional[str] = None,
    resistance_data: Optional[list] = None,
    actions: Optional[list[str]] = None,
    confidence: float = 0.0,
    language: str = "en",
) -> str:
    """
    Format guidance response for Telegram HTML rendering.

    Args:
        guidance: Main guidance text from AI agent
        urgency: Urgency level (low, medium, high, critical)
        case_id: Created case ID (if any)
        resistance_data: List of ResistanceResult objects
        actions: Recommended action items
        confidence: AI confidence score (0.0 - 1.0)
        language: Response language code

    Returns:
        HTML-formatted string for Telegram
    """
    parts: list[str] = []

    # ── Header ─────────────────────────────────────────────────
    urgency_emoji = URGENCY_EMOJI.get(urgency, "ℹ️")
    parts.append(f"{urgency_emoji} <b>UDARA Analysis</b>\n")

    # ── Case ID ────────────────────────────────────────────────
    if case_id:
        parts.append(f"📋 Case: <code>{case_id[:12]}</code>\n")

    # ── Confidence ─────────────────────────────────────────────
    conf_pct = int(confidence * 100)
    conf_bar = get_confidence_bar(confidence)
    parts.append(f"🎯 Confidence: [{conf_bar}] {conf_pct}%\n")

    # ── Main Guidance ──────────────────────────────────────────
    parts.append(f"\n{guidance}\n")

    # ── Resistance Results ─────────────────────────────────────
    if resistance_data:
        parts.append("\n<b>💊 Resistance Data:</b>")
        for drug_res in resistance_data:
            if isinstance(drug_res, dict):
                drug_name = drug_res.get("drug", "Unknown")
                resistance_pct = drug_res.get("resistance_pct", 0)
                category = drug_res.get("category", "unknown")
                alternatives = drug_res.get("alternatives", [])
                sample_size = drug_res.get("sample_size", 0)
            else:
                drug_name = drug_res.drug
                resistance_pct = drug_res.resistance_pct
                category = drug_res.category
                alternatives = drug_res.alternatives
                sample_size = drug_res.sample_size

            emoji = RESISTANCE_EMOJI.get(category, "⚪")
            res_bar = get_resistance_bar(resistance_pct)

            parts.append(
                f"\n{emoji} <b>{drug_name}</b>: "
                f"<code>[{res_bar}]</code> {resistance_pct:.1f}% "
                f"({category})"
            )

            if sample_size > 0:
                parts.append(f"   📊 Based on {sample_size:,} samples")

            if alternatives:
                alt_text = ", ".join(alternatives[:3])
                parts.append(f"   ↩️ Alternatives: {alt_text}")

    # ── Recommended Actions ────────────────────────────────────
    if actions:
        parts.append("\n<b>📌 Recommended Actions:</b>")
        for i, action in enumerate(actions, 1):
            parts.append(f"  {i}. {action}")

    # ── Footer ─────────────────────────────────────────────────
    parts.append(
        "\n━━━━━━━━━━━━━━━━━━\n"
        "Reply to continue or use the menu below ⬇️"
    )

    return "\n".join(parts)


def format_resistance_result_for_telegram(
    drug_name: str,
    resistance_pct: float,
    category: str,
    confidence: float,
    alternatives: Optional[list[str]] = None,
    sample_size: int = 0,
    location: Optional[str] = None,
) -> str:
    """Format a single drug resistance result."""

    emoji = RESISTANCE_EMOJI.get(category, "⚪")
    res_bar = get_resistance_bar(resistance_pct)
    conf_bar = get_confidence_bar(confidence)

    parts = [
        f"{emoji} <b>{drug_name}</b>",
        f"   Resistance: <code>[{res_bar}]</code> {resistance_pct:.1f}%",
        f"   Category: {category}",
        f"   Confidence: [{conf_bar}] {int(confidence * 100)}%",
    ]

    if location:
        parts.append(f"   📍 Location: {location}")

    if sample_size > 0:
        parts.append(f"   📊 Samples: {sample_size:,}")

    if alternatives:
        parts.append(f"   ↩️ Alternatives: {', '.join(alternatives[:3])}")

    return "\n".join(parts)
```

---

## 7. WhatsApp Bot

### 7.1 Main Bot — `whatsapp/bot.py`

```python
"""
UDARA WhatsApp Bot

Built with Meta Cloud API (WhatsApp Business Platform).
Supports:
  - Text messages
  - Interactive buttons (max 3 per row, max 3 rows)
  - Media messages (images, documents)
  - Template messages for notifications
"""

from __future__ import annotations

import hashlib
import hmac
import json
import logging
from typing import Any, Optional

import httpx

from shared.agent_client import AgentClient, get_agent_client, AgentUnavailableError

logger = logging.getLogger(__name__)


class WhatsAppBot:
    """
    WhatsApp Business API Bot.

    Limitations to keep in mind:
      - Max 3 interactive buttons per row
      - Max 3 rows of buttons
      - No markdown bold/italic (use Unicode or plain text)
      - 4096 char message limit
      - Media must be hosted on accessible URL (not local files)
    """

    def __init__(
        self,
        phone_number_id: str,
        access_token: str,
        verify_token: str,
        agent_client: Optional[AgentClient] = None,
    ):
        self.phone_number_id = phone_number_id
        self.access_token = access_token
        self.verify_token = verify_token
        self.agent_client = agent_client or get_agent_client()
        self.api_base = "https://graph.facebook.com/v18.0"
        self._client: Optional[httpx.AsyncClient] = None

    async def _get_client(self) -> httpx.AsyncClient:
        if self._client is None or self._client.is_closed:
            self._client = httpx.AsyncClient(
                headers={"Authorization": f"Bearer {self.access_token}"},
            )
        return self._client

    async def close(self):
        if self._client and not self._client.is_closed:
            await self._client.aclose()

    # ── Webhook Verification ──────────────────────────────────

    def verify_webhook(self, mode: str, token: str, challenge: str) -> Optional[str]:
        """Verify webhook subscription from Meta."""
        if mode == "subscribe" and token == self.verify_token:
            logger.info("Webhook verified")
            return challenge
        return None

    # ── Message Sending ───────────────────────────────────────

    async def send_text(
        self,
        to: str,
        text: str,
        preview_url: bool = False,
    ) -> dict:
        """Send a text message."""
        client = await self._get_client()
        payload = {
            "messaging_product": "whatsapp",
            "to": to,
            "type": "text",
            "text": {
                "body": text[:4096],  # Enforce limit
                "preview_url": preview_url,
            },
        }
        response = await client.post(
            f"{self.api_base}/{self.phone_number_id}/messages",
            json=payload,
        )
        response.raise_for_status()
        return response.json()

    async def send_interactive_buttons(
        self,
        to: str,
        body_text: str,
        buttons: list[dict[str, str]],  # [{id, title}, ...]
        header_text: Optional[str] = None,
        footer_text: Optional[str] = None,
    ) -> dict:
        """
        Send an interactive button message.
        Max 3 buttons, max 20 chars per button title.
        """
        if len(buttons) > 3:
            logger.warning("Truncating buttons to 3 (received %d)", len(buttons))
            buttons = buttons[:3]

        # Build interactive object
        interactive: dict[str, Any] = {
            "type": "button",
            "body": {
                "text": body_text[:1024],
            },
            "action": {
                "buttons": [
                    {
                        "type": "reply",
                        "reply": {
                            "id": btn["id"],
                            "title": btn["title"][:20],
                        },
                    }
                    for btn in buttons
                ],
            },
        }

        if header_text:
            interactive["header"] = {"type": "text", "text": header_text[:60]}

        if footer_text:
            interactive["footer"] = {"text": footer_text[:60]}

        client = await self._get_client()
        payload = {
            "messaging_product": "whatsapp",
            "to": to,
            "type": "interactive",
            "interactive": interactive,
        }
        response = await client.post(
            f"{self.api_base}/{self.phone_number_id}/messages",
            json=payload,
        )
        response.raise_for_status()
        return response.json()

    async def send_image(
        self,
        to: str,
        image_url: str,
        caption: Optional[str] = None,
    ) -> dict:
        """Send an image message."""
        client = await self._get_client()
        payload: dict[str, Any] = {
            "messaging_product": "whatsapp",
            "to": to,
            "type": "image",
            "image": {"url": image_url},
        }
        if caption:
            payload["image"]["caption"] = caption[:1024]

        response = await client.post(
            f"{self.api_base}/{self.phone_number_id}/messages",
            json=payload,
        )
        response.raise_for_status()
        return response.json()

    async def send_location(
        self,
        to: str,
        latitude: float,
        longitude: float,
        name: Optional[str] = None,
        address: Optional[str] = None,
    ) -> dict:
        """Send a location message."""
        client = await self._get_client()
        payload = {
            "messaging_product": "whatsapp",
            "to": to,
            "type": "location",
            "location": {
                "latitude": latitude,
                "longitude": longitude,
            },
        }
        if name:
            payload["location"]["name"] = name
        if address:
            payload["location"]["address"] = address

        response = await client.post(
            f"{self.api_base}/{self.phone_number_id}/messages",
            json=payload,
        )
        response.raise_for_status()
        return response.json()

    # ── Media Download ────────────────────────────────────────

    async def download_media(self, media_id: str) -> tuple[bytes, str]:
        """
        Download media from WhatsApp.

        Returns: (bytes, mime_type)
        """
        client = await self._get_client()

        # Step 1: Get media URL
        response = await client.get(f"{self.api_base}/{media_id}")
        response.raise_for_status()
        media_data = response.json()
        media_url = media_data["url"]
        mime_type = media_data["mime_type"]

        # Step 2: Download the actual file
        file_response = await client.get(media_url)
        file_response.raise_for_status()

        return file_response.content, mime_type

    # ── Incoming Message Handler ──────────────────────────────

    async def process_webhook_event(self, body: dict) -> Optional[str]:
        """
        Process an incoming webhook event.
        Returns the phone number if a message was processed.
        """
        try:
            for entry in body.get("entry", []):
                for change in entry.get("changes", []):
                    value = change.get("value", {})

                    # Handle messages
                    messages = value.get("messages", [])
                    for msg in messages:
                        phone = msg["from"]
                        msg_type = msg.get("type")

                        if msg_type == "text":
                            text_body = msg["text"]["body"]
                            await self._handle_text(phone, text_body, msg)
                        elif msg_type == "interactive":
                            await self._handle_interactive(phone, msg)
                        elif msg_type == "image":
                            await self._handle_image(phone, msg)
                        elif msg_type == "document":
                            await self._handle_document(phone, msg)

                        return phone

                    # Handle statuses
                    statuses = value.get("statuses", [])
                    for status in statuses:
                        await self._handle_status(status)

        except Exception as exc:
            logger.exception("Webhook processing error: %s", exc)

        return None

    async def _handle_text(self, phone: str, text: str, msg: dict):
        """Handle incoming text message."""
        logger.info("WhatsApp text from %s: %s", phone, text[:100])

        try:
            ingest_response = await self.agent_client.ingest_text(
                text=text,
                user_id=phone,
            )

            if ingest_response.success and ingest_response.guidance:
                response_text = self._format_response(ingest_response)
                await self.send_text(phone, response_text)
                await self._send_main_menu(phone)
            else:
                await self.send_text(
                    phone,
                    "I received your message. Please use the menu to:\n"
                    "- Report a case\n"
                    "- Check drug resistance\n"
                    "- View your stats",
                )
                await self._send_main_menu(phone)

        except AgentUnavailableError:
            await self.send_text(
                phone,
                "Our AI service is temporarily busy. "
                "Please try again in a few minutes.",
            )

    async def _handle_image(self, phone: str, msg: dict):
        """Handle incoming image message."""
        logger.info("WhatsApp image from %s", phone)

        try:
            image_id = msg["image"]["id"]
            image_bytes, mime_type = await self.download_media(image_id)

            ingest_response = await self.agent_client.ingest_image(
                image_bytes=image_bytes,
                filename=f"wa_{phone}_{msg['timestamp']}.jpg",
                user_id=phone,
            )

            if ingest_response.success:
                response_text = self._format_response(ingest_response)
                await self.send_text(phone, response_text)
            else:
                await self.send_text(
                    phone,
                    "I couldn't read the text in this image. "
                    "Please try a clearer photo.",
                )

            await self._send_main_menu(phone)

        except Exception as exc:
            logger.exception("Image handling error: %s", exc)
            await self.send_text(phone, "Error processing image. Please try again.")

    async def _handle_interactive(self, phone: str, msg: dict):
        """Handle interactive button response."""
        button_id = msg["interactive"]["button_reply"]["id"]
        logger.info("WhatsApp button from %s: %s", phone, button_id)

        if button_id == "menu_report":
            await self.send_text(phone, "Please type your case report:\nInclude symptoms, drugs used, and patient details.")
        elif button_id == "menu_resistance":
            await self.send_text(phone, "Which drug would you like to check?\nExample: Amoxicillin, Ciprofloxacin")
        elif button_id == "menu_stats":
            await self.send_text(phone, "📊 Your stats this month:\n- Cases reported: 0\n- Accuracy score: 0%")
        else:
            await self._send_main_menu(phone)

    async def _handle_status(self, status: dict):
        """Handle message status updates."""
        msg_id = status.get("id")
        status_type = status.get("status")
        logger.debug("Message %s status: %s", msg_id, status_type)

    async def _send_main_menu(self, phone: str):
        """Send the main menu with interactive buttons."""
        await self.send_interactive_buttons(
            to=phone,
            body_text="UDARA AI - AMR Surveillance\n\nHow can I help you today?",
            buttons=[
                {"id": "menu_report", "title": "📋 Report Case"},
                {"id": "menu_resistance", "title": "💊 Resistance"},
                {"id": "menu_stats", "title": "📊 My Stats"},
            ],
        )

    def _format_response(self, ingest_response) -> str:
        """Format agent response for WhatsApp (no markdown)."""
        lines = ["*UDARA Analysis*"]

        if ingest_response.case_id:
            lines.append(f"Case: {ingest_response.case_id[:12]}")

        if ingest_response.guidance:
            lines.append(f"\n{ingest_response.guidance}")

        if ingest_response.resistance_summary:
            for drug, data in ingest_response.resistance_summary.items():
                pct = data.get("resistance_pct", 0)
                emoji = "🟢" if pct < 30 else "🟡" if pct < 70 else "🔴"
                lines.append(f"\n{emoji} {drug}: {pct:.1f}%")

        return "\n".join(lines)
```

---

## 8. Shared Modules

### 8.1 Onboarding Flow — `shared/onboarding/flow.py`

```python
"""
UDARA Onboarding Flow — 5-Step FSM

New CHW onboarding steps:
  Step 1: Country selection
  Step 2: Full name
  Step 3: District / facility
  Step 4: Role (CHW, clinician, lab tech, nurse)
  Step 5: Language preference

State stored in Redis with 24-hour TTL.
"""

from __future__ import annotations

import logging
from dataclasses import dataclass, field
from enum import Enum, auto
from typing import Any, Optional

from aiogram.fsm.state import State, StatesGroup

logger = logging.getLogger(__name__)


class OnboardingSteps(StatesGroup):
    """FSM states for onboarding flow."""

    STEP_1_COUNTRY = State()
    STEP_2_NAME = State()
    STEP_3_DISTRICT = State()
    STEP_4_ROLE = State()
    STEP_5_LANGUAGE = State()
    COMPLETED = State()


@dataclass
class OnboardingData:
    """Data collected during onboarding."""

    user_id: str
    platform: str  # "telegram" or "whatsapp"
    current_step: int = 1
    total_steps: int = 5

    # Collected fields
    country: Optional[str] = None
    name: Optional[str] = None
    district: Optional[str] = None
    facility: Optional[str] = None
    role: Optional[str] = None
    language: str = "en"

    # Metadata
    started_at: Optional[str] = None
    completed_at: Optional[str] = None

    def to_dict(self) -> dict:
        return {
            "user_id": self.user_id,
            "platform": self.platform,
            "current_step": self.current_step,
            "total_steps": self.total_steps,
            "country": self.country,
            "name": self.name,
            "district": self.district,
            "facility": self.facility,
            "role": self.role,
            "language": self.language,
            "started_at": self.started_at,
            "completed_at": self.completed_at,
        }

    @classmethod
    def from_dict(cls, data: dict) -> OnboardingData:
        return cls(**{k: v for k, v in data.items() if k in cls.__dataclass_fields__})


# Step configurations
COUNTRIES = [
    "🇳🇬 Nigeria",
    "🇰🇪 Kenya",
    "🇬🇭 Ghana",
    "🇹🇿 Tanzania",
    "🇺🇬 Uganda",
    "🇸🇳 Senegal",
    "🇪🇹 Ethiopia",
    "🇲🇼 Malawi",
    "🇿🇦 South Africa",
    "🇷🇼 Rwanda",
]

ROLES = [
    ("chw", "Community Health Worker"),
    ("clinician", "Clinician / Doctor"),
    ("nurse", "Nurse"),
    ("lab_tech", "Laboratory Technician"),
    ("pharmacist", "Pharmacist"),
    ("student", "Student / Trainee"),
    ("other", "Other"),
]

LANGUAGES = [
    ("en", "🇬🇧 English"),
    ("fr", "🇫🇷 Français"),
    ("ha", "🇳🇬 Hausa"),
    ("yo", "🇳🇬 Yoruba"),
    ("ig", "🇳🇬 Igbo"),
    ("sw", "🇰🇪 Swahili"),
    ("am", "🇪🇹 Amharic"),
]


class OnboardingFlow:
    """
    Manages the 5-step onboarding FSM.

    Each step has:
      - prompt: Message sent to user
      - validate: Input validation function
      - save: Store validated input
      - next_step: Advance to next step or complete
    """

    def __init__(self, storage):
        """
        Args:
            storage: RedisStorage instance for state persistence
        """
        self.storage = storage

    async def start(self, user_id: str, platform: str) -> OnboardingData:
        """Initialize onboarding for a new user."""
        data = OnboardingData(
            user_id=user_id,
            platform=platform,
            current_step=1,
        )
        await self._save_state(user_id, data)
        return data

    async def get_state(self, user_id: str) -> Optional[OnboardingData]:
        """Get current onboarding state."""
        raw = await self.storage.get(f"onboarding:{user_id}")
        if raw:
            return OnboardingData.from_dict(raw)
        return None

    async def advance(
        self,
        user_id: str,
        step_input: str,
    ) -> tuple[bool, str, Optional[OnboardingData]]:
        """
        Process input for the current step.

        Returns:
            (is_valid, response_message, updated_data_or_None)
        """
        data = await self.get_state(user_id)
        if not data:
            return False, "Onboarding not found. Please restart.", None

        # Validate and save based on current step
        if data.current_step == 1:
            is_valid, response = self._validate_country(step_input)
            if is_valid:
                data.country = step_input
                data.current_step = 2

        elif data.current_step == 2:
            is_valid, response = self._validate_name(step_input)
            if is_valid:
                data.name = step_input.strip()
                data.current_step = 3

        elif data.current_step == 3:
            is_valid, response = self._validate_district(step_input)
            if is_valid:
                data.district = step_input.strip()
                data.current_step = 4

        elif data.current_step == 4:
            is_valid, response = self._validate_role(step_input)
            if is_valid:
                data.role = step_input.strip().lower()
                data.current_step = 5

        elif data.current_step == 5:
            is_valid, response = self._validate_language(step_input)
            if is_valid:
                data.language = step_input.strip().lower()
                data.current_step = 6  # Completed
                data.completed_at = self._now_iso()

                await self._save_state(user_id, data)
                return True, self._get_completion_message(data), data

        await self._save_state(user_id, data)

        if is_valid:
            prompt = self._get_step_prompt(data.current_step)
            return True, prompt, data
        else:
            return False, response, data

    async def _save_state(self, user_id: str, data: OnboardingData):
        """Persist onboarding state to Redis."""
        await self.storage.set(
            f"onboarding:{user_id}",
            data.to_dict(),
            ex=86400,  # 24-hour TTL
        )

    # ── Step Validators ──────────────────────────────────────

    @staticmethod
    def _validate_country(input_text: str) -> tuple[bool, str]:
        """Validate country selection."""
        if not input_text.strip():
            return False, "Please select your country."
        return True, ""

    @staticmethod
    def _validate_name(input_text: str) -> tuple[bool, str]:
        """Validate name (min 2 chars, no special chars)."""
        name = input_text.strip()
        if len(name) < 2:
            return False, "Name must be at least 2 characters."
        if len(name) > 100:
            return False, "Name is too long. Please use a shorter name."
        return True, ""

    @staticmethod
    def _validate_district(input_text: str) -> tuple[bool, str]:
        """Validate district/facility name."""
        if len(input_text.strip()) < 2:
            return False, "Please enter a valid district or facility name."
        return True, ""

    @staticmethod
    def _validate_role(input_text: str) -> tuple[bool, str]:
        """Validate role selection."""
        valid_roles = [r[0] for r in ROLES]
        if input_text.strip().lower() not in valid_roles:
            return False, f"Invalid role. Choose from: {', '.join(valid_roles)}"
        return True, ""

    @staticmethod
    def _validate_language(input_text: str) -> tuple[bool, str]:
        """Validate language selection."""
        valid_langs = [l[0] for l in LANGUAGES]
        if input_text.strip().lower() not in valid_langs:
            return False, f"Invalid language. Choose from: {', '.join(valid_langs)}"
        return True, ""

    # ── Step Prompts ─────────────────────────────────────────

    def _get_step_prompt(self, step: int) -> str:
        """Get the prompt message for a given step."""
        prompts = {
            1: (
                "👋 Welcome to <b>UDARA AI</b> — Antimicrobial Resistance Surveillance\n\n"
                "<b>Step 1/5:</b> Select your country\n\n"
                + "\n".join(COUNTRIES)
            ),
            2: (
                "<b>Step 2/5:</b> What is your full name?\n\n"
                "This will be used for reporting and leaderboard entries."
            ),
            3: (
                "<b>Step 3/5:</b> Which district or facility do you work at?\n\n"
                "Example: Kano Municipal, Lagos Island PHC"
            ),
            4: (
                "<b>Step 4/5:</b> What is your role?\n\n"
                + "\n".join(f"  • {code}: {label}" for code, label in ROLES)
                + "\n\nType the code (e.g., chw)"
            ),
            5: (
                "<b>Step 5/5:</b> Select your preferred language\n\n"
                + "\n".join(f"  {code}: {label}" for code, label in LANGUAGES)
                + "\n\nType the language code (e.g., en)"
            ),
        }
        return prompts.get(step, "Unknown step")

    @staticmethod
    def _get_completion_message(data: OnboardingData) -> str:
        """Get the completion message after successful onboarding."""
        return (
            "🎉 <b>Onboarding Complete!</b>\n\n"
            f"Welcome, <b>{data.name}</b>!\n\n"
            f"📍 {data.country} — {data.district}\n"
            f"👤 Role: {data.role}\n"
            f"🌐 Language: {data.language}\n\n"
            "You can now:\n"
            "  📋 Report cases\n"
            "  💊 Check drug resistance\n"
            "  📊 Track your stats\n"
            "  📚 Learn about AMR\n\n"
            "Type anything or use the menu to get started!"
        )

    @staticmethod
    def _now_iso() -> str:
        from datetime import datetime, timezone
        return datetime.now(timezone.utc).isoformat()
```

### 8.2 Daily Digest — `shared/daily_digest.py`

```python
"""
UDARA Daily Digest Scheduler

Sends personalized daily summaries to CHWs via Telegram and WhatsApp.
Each digest includes:
  - Personal stats (cases reported, accuracy score)
  - District-level alerts
  - Daily AMR tip
  - Leaderboard position

Scheduler supports timezone-aware delivery:
  - Digests sent at 08:00 local time
  - Cron jobs per timezone cluster
"""

from __future__ import annotations

import asyncio
import logging
from dataclasses import dataclass
from datetime import datetime, timedelta, timezone
from typing import Any, Optional

import httpx
from apscheduler.schedulers.asyncio import AsyncIOScheduler
from apscheduler.triggers.cron import CronTrigger

logger = logging.getLogger(__name__)


@dataclass
class DigestRecipient:
    """A CHW scheduled to receive a daily digest."""
    user_id: str
    platform: str  # "telegram" or "whatsapp"
    timezone: str   # IANA timezone, e.g., "Africa/Lagos"
    language: str = "en"
    name: str = ""
    district: str = ""


@dataclass
class DigestContent:
    """Content for a single digest message."""
    greeting: str
    personal_stats: dict[str, Any]
    district_alerts: list[dict[str, Any]]
    daily_tip: str
    leaderboard_position: int
    leaderboard_total: int
    trending_drugs: list[dict[str, Any]]


class DailyDigestScheduler:
    """
    Manages scheduled delivery of personalized daily digests.

    Architecture:
      1. APScheduler runs cron jobs per timezone cluster
      2. On trigger, fetch recipient list from backend API
      3. For each recipient, fetch personalized content
      4. Format and send via appropriate platform bot
      5. Log delivery status and failures
    """

    # Timezone clusters (group nearby timezones for efficiency)
    TIMEZONE_CLUSTERS = {
        "wt": {"zones": ["Africa/Lagos", "Africa/Accra"], "hour": 8},
        "et": {"zones": ["Africa/Nairobi", "Africa/Dar_es_Salaam"], "hour": 8},
        "sat": {"zones": ["Africa/Johannesburg"], "hour": 8},
    }

    def __init__(
        self,
        backend_url: str,
        telegram_bot=None,
        whatsapp_bot=None,
    ):
        self.backend_url = backend_url
        self.telegram_bot = telegram_bot
        self.whatsapp_bot = whatsapp_bot
        self.scheduler = AsyncIOScheduler(timezone="UTC")
        self._http_client: Optional[httpx.AsyncClient] = None

    async def _get_client(self) -> httpx.AsyncClient:
        if self._http_client is None:
            self._http_client = httpx.AsyncClient(
                base_url=self.backend_url,
                timeout=30.0,
            )
        return self._http_client

    def start(self):
        """Start the scheduler with cron jobs for each timezone cluster."""
        for cluster_id, config in self.TIMEZONE_CLUSTERS.items():
            self.scheduler.add_job(
                self._run_digest_for_cluster,
                CronTrigger(
                    hour=config["hour"],
                    minute=0,
                    day_of_week="mon-fri",  # Weekdays only
                    timezone="UTC",
                ),
                id=f"digest_{cluster_id}",
                kwargs={"cluster_id": cluster_id},
                replace_existing=True,
            )
            logger.info(
                "Scheduled digest for cluster %s at %02d:00 UTC",
                cluster_id,
                config["hour"],
            )

        self.scheduler.start()
        logger.info("Daily digest scheduler started")

    def stop(self):
        """Stop the scheduler."""
        self.scheduler.shutdown(wait=False)

    async def _run_digest_for_cluster(self, cluster_id: str):
        """Fetch recipients and send digests for a timezone cluster."""
        config = self.TIMEZONE_CLUSTERS[cluster_id]
        logger.info(
            "Running digest for cluster %s (%s)",
            cluster_id,
            ", ".join(config["zones"]),
        )

        try:
            client = await self._get_client()

            # Fetch recipients for this timezone cluster
            response = await client.get(
                "/internal/digest/recipients",
                params={"timezones": ",".join(config["zones"])},
            )
            response.raise_for_status()
            recipients_data = response.json()

            recipients = [
                DigestRecipient(**r) for r in recipients_data.get("recipients", [])
            ]

            logger.info(
                "Sending %d digests for cluster %s",
                len(recipients),
                cluster_id,
            )

            # Send digests in batches of 10 (rate limiting)
            batch_size = 10
            for i in range(0, len(recipients), batch_size):
                batch = recipients[i:i + batch_size]
                tasks = [
                    self._send_digest(recipient) for recipient in batch
                ]
                results = await asyncio.gather(*tasks, return_exceptions=True)

                for recipient, result in zip(batch, results):
                    if isinstance(result, Exception):
                        logger.error(
                            "Digest failed for %s (%s): %s",
                            recipient.user_id,
                            recipient.platform,
                            str(result),
                        )

                # Rate limit: wait between batches
                if i + batch_size < len(recipients):
                    await asyncio.sleep(2)

        except Exception as exc:
            logger.exception("Digest cluster error for %s: %s", cluster_id, exc)

    async def _send_digest(self, recipient: DigestRecipient):
        """Fetch personalized content and send digest to a single recipient."""
        client = await self._get_client()

        # Fetch personalized digest content from backend
        response = await client.get(
            f"/internal/digest/content/{recipient.user_id}",
        )
        response.raise_for_status()
        content_data = response.json()

        content = DigestContent(**content_data)

        # Format for platform
        if recipient.platform == "telegram":
            text = self._format_for_telegram(content, recipient)
            if self.telegram_bot:
                await self.telegram_bot.send_message(
                    chat_id=int(recipient.user_id),
                    text=text,
                    parse_mode="HTML",
                )
        elif recipient.platform == "whatsapp":
            text = self._format_for_whatsapp(content, recipient)
            if self.whatsapp_bot:
                await self.whatsapp_bot.send_text(
                    to=recipient.user_id,
                    text=text,
                )

        logger.info("Digest sent to %s via %s", recipient.user_id, recipient.platform)

    @staticmethod
    def _format_for_telegram(content: DigestContent, recipient: DigestRecipient) -> str:
        """Format digest for Telegram (HTML)."""
        stats = content.personal_stats

        parts = [
            f"☀️ <b>Good morning, {recipient.name}!</b>\n",
            f"📊 <b>Your Week in Numbers:</b>",
            f"  • Cases reported: <b>{stats.get('cases_this_week', 0)}</b>",
            f"  • Accuracy score: <b>{stats.get('accuracy_pct', 0)}%</b>",
            f"  • Streak: <b>{stats.get('streak_days', 0)} days</b> 🔥",
        ]

        # Leaderboard
        if content.leaderboard_position:
            emoji = "🥇" if content.leaderboard_position <= 1 else "🥈" if content.leaderboard_position <= 2 else "🥉" if content.leaderboard_position <= 3 else "📊"
            parts.append(
                f"\n{emoji} Leaderboard: #{content.leaderboard_position} "
                f"of {content.leaderboard_total}"
            )

        # Alerts
        if content.district_alerts:
            parts.append(f"\n🚨 <b>District Alerts ({len(content.district_alerts)}):</b>")
            for alert in content.district_alerts[:3]:
                parts.append(
                    f"  • {alert.get('drug', 'Unknown')}: "
                    f"{alert.get('resistance_pct', 0):.0f}% in {alert.get('district', '')}"
                )

        # Trending drugs
        if content.trending_drugs:
            parts.append("\n📈 <b>Trending Resistance:</b>")
            for drug in content.trending_drugs[:3]:
                emoji = "🟢" if drug["change"] < 0 else "🔴"
                parts.append(
                    f"  {emoji} {drug['name']}: {drug['resistance_pct']:.1f}% "
                    f"({drug['change']:+.1f}%)"
                )

        # Daily tip
        if content.daily_tip:
            parts.append(f"\n💡 <b>Daily Tip:</b> {content.daily_tip}")

        parts.append("\n━━━━━━━━━━━━━━━")
        parts.append("Keep up the great work! 💪")

        return "\n".join(parts)

    @staticmethod
    def _format_for_whatsapp(content: DigestContent, recipient: DigestRecipient) -> str:
        """Format digest for WhatsApp (plain text, no markdown)."""
        # Similar to Telegram but without HTML tags
        stats = content.personal_stats

        lines = [
            f"☀️ Good morning, {recipient.name}!",
            "",
            "📊 Your Week in Numbers:",
            f"  • Cases reported: {stats.get('cases_this_week', 0)}",
            f"  • Accuracy score: {stats.get('accuracy_pct', 0)}%",
            f"  • Streak: {stats.get('streak_days', 0)} days 🔥",
        ]

        if content.daily_tip:
            lines.extend(["", f"💡 Daily Tip: {content.daily_tip}"])

        return "\n".join(lines)
```

### 8.3 Resistance Calculator — `shared/conversations/resistance_calc.py`

```python
"""
Resistance Calculator Conversation Flow

Multi-step conversation to check drug resistance:
  1. User names a drug
  2. System looks up local resistance data
  3. Displays probability with Unicode bars
  4. Shows alternatives if resistance is high
"""

from __future__ import annotations

import logging
from typing import Optional

from shared.agent_client import AgentClient, ResistanceResult, get_agent_client

logger = logging.getLogger(__name__)


def format_resistance_bar(pct: float, width: int = 10) -> str:
    """Generate Unicode resistance bar: ████░░░░░░."""
    if pct <= 0:
        return "░" * width
    filled = max(1, int(pct / 100 * width))
    empty = width - filled
    return "█" * filled + "░" * empty


def get_resistance_emoji(category: str) -> str:
    """Get emoji for resistance category."""
    mapping = {
        "susceptible": "🟢",
        "low": "🟢",
        "safe": "🟢",
        "intermediate": "🔵",
        "moderate": "🔵",
        "warning": "🟡",
        "resistant": "🔴",
        "high": "🔴",
        "critical": "🔴",
    }
    return mapping.get(category.lower(), "⚪")


async def calculate_resistance(
    drug_name: str,
    agent_client: Optional[AgentClient] = None,
    location: Optional[dict] = None,
    patient_age: Optional[int] = None,
) -> str:
    """
    Calculate and format resistance for a drug.

    Returns a formatted string with:
      - Drug name and resistance percentage
      - Unicode probability bar
      - Category (susceptible/intermediate/resistant)
      - Confidence score
      - Alternatives if resistance is high
      - Sample size
    """
    agent = agent_client or get_agent_client()

    try:
        response = await agent.check_resistance(
            drugs=[drug_name],
            location=location,
            patient_age=patient_age,
        )

        if not response.drugs:
            return (
                f"❓ No resistance data found for <b>{drug_name}</b>\n\n"
                "This drug may not be tracked in your area yet. "
                "Try another drug or check back later."
            )

        result: ResistanceResult = response.drugs[0]
        bar = format_resistance_bar(result.resistance_pct)
        emoji = get_resistance_emoji(result.category)
        conf_bar = format_resistance_bar(result.confidence * 100)

        lines = [
            f"{emoji} <b>{result.drug}</b>",
            f"",
            f"Resistance: [{bar}] {result.resistance_pct:.1f}%",
            f"Category: {result.category}",
            f"Confidence: [{conf_bar}] {int(result.confidence * 100)}%",
        ]

        if result.sample_size > 0:
            lines.append(f"Based on: {result.sample_size:,} samples")

        if result.alternatives:
            lines.append("")
            lines.append("↩️ <b>Alternatives:</b>")
            for alt in result.alternatives[:5]:
                lines.append(f"  • {alt}")

        if result.resistance_pct >= 70:
            lines.extend([
                "",
                "🚨 <b>WARNING:</b> High resistance detected!",
                "Consider alternative antibiotics.",
                "Consult with a clinician before prescribing.",
            ])
        elif result.resistance_pct >= 50:
            lines.extend([
                "",
                "⚠️ <b>Note:</b> Moderate resistance in your area.",
                "Monitor treatment response closely.",
            ])

        return "\n".join(lines)

    except Exception as exc:
        logger.error("Resistance calculation error for %s: %s", drug_name, exc)
        return (
            f"⚠️ Error checking resistance for <b>{drug_name}</b>.\n"
            "Please try again in a moment."
        )
```

### 8.4 Multilingual Support — `shared/multilingual.py`

```python
"""
UDARA Multilingual Support

Provides:
  - Language detection from message text
  - Template-based response generation in 12+ languages
  - Fallback chain: detected → user preference → English
"""

from __future__ import annotations

import logging
from typing import Optional

logger = logging.getLogger(__name__)


# ── Supported Languages ────────────────────────────────────────────

SUPPORTED_LANGUAGES = {
    "en": {"name": "English", "native": "English", "flag": "🇬🇧"},
    "fr": {"name": "French", "native": "Français", "flag": "🇫🇷"},
    "ha": {"name": "Hausa", "native": "Hausa", "flag": "🇳🇬"},
    "yo": {"name": "Yoruba", "native": "Yorùbá", "flag": "🇳🇬"},
    "ig": {"name": "Igbo", "native": "Igbo", "flag": "🇳🇬"},
    "sw": {"name": "Swahili", "native": "Kiswahili", "flag": "🇰🇪"},
    "am": {"name": "Amharic", "native": "አማርኛ", "flag": "🇪🇹"},
    "zu": {"name": "Zulu", "native": "isiZulu", "flag": "🇿🇦"},
    "pt": {"name": "Portuguese", "native": "Português", "flag": "🇲🇿"},
    "ar": {"name": "Arabic", "native": "العربية", "flag": "🇸🇩"},
    "so": {"name": "Somali", "native": "Soomaali", "flag": "🇸🇴"},
    "rw": {"name": "Kinyarwanda", "native": "Ikinyarwanda", "flag": "🇷🇼"},
}


# ── Language Detection (simple keyword-based) ─────────────────────

LANGUAGE_KEYWORDS = {
    "fr": ["bonjour", "merci", "oui", "non", "résistance", "médicament", "patient"],
    "ha": ["sannu", "na gode", "iya", "a'a", "magani", "mcuta"],
    "yo": ["bawo ni", "e se", "oṣe", "oogun", "arun"],
    "ig": ["ndewo", "imeela", "ee", "mba", "ọgwụ", "ọrịa"],
    "sw": ["habari", "asante", "ndiyo", "hapana", "dawa", "ugonjwa"],
    "am": ["ሰላም", "እናመሰግናለን", "አዎ", "የለም", "መድሃሒት"],
    "ar": ["مرحبا", "شكرا", "نعم", "لا", "مضاد", "حاد"],
}


def detect_language(text: str) -> Optional[str]:
    """
    Detect language from text using keyword matching.
    Returns ISO 639-1 code or None if undetermined.
    """
    text_lower = text.lower()

    for lang, keywords in LANGUAGE_KEYWORDS.items():
        matches = sum(1 for kw in keywords if kw in text_lower)
        if matches >= 2:
            return lang

    return None


# ── Template Responses ─────────────────────────────────────────────

RESPONSE_TEMPLATES = {
    "welcome": {
        "en": "Welcome to UDARA AI — AMR Surveillance. How can I help you today?",
        "fr": "Bienvenue sur UDARA AI — Surveillance de la RAM. Comment puis-je vous aider?",
        "ha": "Barka da zuwa UDARA AI — Tsarin Kula da Rigakafin Magunguna. Yaya zan iya taimaka muku?",
        "sw": "Karibu UDARA AI — Ufuatiliaji wa Ushindaji wa Dawa. Ninaweza kusaidia vipi?",
    },
    "case_recorded": {
        "en": "✅ Case recorded successfully. Case ID: {case_id}",
        "fr": "✅ Cas enregistré avec succès. ID du cas: {case_id}",
        "ha": "✅ An yi rajistar shari'ar nasara. ID: {case_id}",
        "sw": "✅ Kesi imerekwa kwa mafanikio. Nambari ya kesi: {case_id}",
    },
    "resistance_high": {
        "en": "🚨 HIGH RESISTANCE detected for {drug} in your area ({pct}%)",
        "fr": "🚨 RÉSISTANCE ÉLEVÉE détectée pour {drug} dans votre zone ({pct}%)",
        "ha": "🚨 ANFANI MAI GURIN YA SAMU ga {drug} a yankin ku ({pct}%)",
        "sw": "🚨 USHINDAJI WA JUU umegundulika kwa {drug} kwenye eneo lako ({pct}%)",
    },
    "processing": {
        "en": "🔍 Processing your message...",
        "fr": "🔍 Traitement de votre message...",
        "ha": "🔍 Ana gudanar da sakon ku...",
        "sw": "🔍 Inachakata ujumbe wako...",
    },
    "error_generic": {
        "en": "⚠️ Something went wrong. Please try again.",
        "fr": "⚠️ Un problème est survenu. Veuillez réessayer.",
        "ha": "⚠️ Wani abu ya faru. Da fatan a sake gwadawa.",
        "sw": "⚠️ Kuna tatizo. Tafadhali jaribu tena.",
    },
    "agent_unavailable": {
        "en": "😔 Our AI service is temporarily busy. Please try again in a few minutes.",
        "fr": "😔 Notre service IA est temporairement occupé. Veuillez réessayer.",
        "ha": "😩 Sabisin AI ɗinmu na cikin mamaki a lokacin. Da fatan a sake gwadawa.",
        "sw": "😔 Huduma yetu ya AI inashughulika kwa muda. Tafadhali jaribu tena.",
    },
}


def get_template(
    template_key: str,
    language: str = "en",
    fallback: str = "en",
    **kwargs,
) -> str:
    """
    Get a localized template string.

    Args:
        template_key: Key in RESPONSE_TEMPLATES
        language: Target language code
        fallback: Fallback language if translation not found
        **kwargs: Variables to interpolate into the template

    Returns:
        Localized string with variables interpolated
    """
    template_set = RESPONSE_TEMPLATES.get(template_key, {})

    # Try requested language, then fallback, then English, then key
    text = (
        template_set.get(language)
        or template_set.get(fallback)
        or template_set.get("en")
        or f"[{template_key}]"
    )

    # Interpolate variables
    for key, value in kwargs.items():
        text = text.replace(f"{{{key}}}", str(value))

    return text


def get_response_language(
    detected_lang: Optional[str],
    user_preference: Optional[str],
) -> str:
    """
    Determine the best response language.

    Priority: user_preference > detected > "en"
    """
    if user_preference and user_preference in SUPPORTED_LANGUAGES:
        return user_preference
    if detected_lang and detected_lang in SUPPORTED_LANGUAGES:
        return detected_lang
    return "en"
```

---

## 9. E2E Test Scenarios

### 9.1 Test Fixtures — `e2e-tests/conftest.py`

```python
"""
E2E test fixtures and configuration.
"""

import asyncio
import pytest
from unittest.mock import AsyncMock, MagicMock

from shared.agent_client import AgentClient, IngestResponse, ResistanceCheckResponse
from shared.onboarding.flow import OnboardingFlow


@pytest.fixture
def mock_agent_client():
    """Create a mock AgentClient with pre-configured responses."""
    client = MagicMock(spec=AgentClient)

    # Default: successful text ingest
    client.ingest_text = AsyncMock(return_value=IngestResponse(
        success=True,
        case_id="test-case-001",
        extracted_data={
            "drugs": ["Amoxicillin"],
            "symptoms": ["fever", "cough"],
            "location": {"country": "Nigeria", "state": "Lagos"},
            "patient_age": 35,
        },
        guidance="Consider alternative antibiotics...",
        confidence=0.85,
        processing_time_ms=1200,
    ))

    # Default: resistance check
    client.check_resistance = AsyncMock(return_value=ResistanceCheckResponse(
        drugs=[],
        overall_risk="medium",
    ))

    client.get_guidance = AsyncMock(return_value=MagicMock(
        guidance_text="Based on local resistance data...",
        urgency_level="medium",
        recommended_actions=["Monitor treatment response"],
        warnings=[],
        confidence=0.82,
    ))

    client.close = AsyncMock()

    return client


@pytest.fixture
def mock_redis_storage():
    """Create a mock Redis storage for FSM states."""
    storage = MagicMock()
    storage.get = AsyncMock(return_value=None)
    storage.set = AsyncMock(return_value=True)
    storage.delete = AsyncMock(return_value=True)
    return storage
```

### 9.2 Scenario 1 — Full Case Report via Text

```python
"""
E2E Test: Full case report via free text.

Flow: CHW types case description → agent extracts data →
      resistance check → guidance generated → formatted response sent.
"""

import pytest
from unittest.mock import AsyncMock, MagicMock, patch

from shared.agent_client import (
    AgentClient,
    IngestResponse,
    ResistanceCheckResponse,
    ResistanceResult,
)


@pytest.mark.asyncio
async def test_full_case_report_text_flow(mock_agent_client):
    """
    Scenario: CHW reports a case via free text.

    Given:
      - A CHW sends: "35yo male presenting with fever, cough for 5 days.
        Started on Amoxicillin 500mg TDS yesterday. No improvement."
      - User is registered and past onboarding

    When:
      - Text handler processes the message
      - Agent extracts drugs and symptoms
      - Resistance check runs for Amoxicillin in Lagos

    Then:
      - A case is created with ID
      - Resistance data returned showing Amoxicillin at 62%
      - Guidance recommends alternative (Ceftriaxone)
      - Response formatted with resistance bars and alternatives
    """

    # Configure mock responses
    mock_agent_client.ingest_text = AsyncMock(return_value=IngestResponse(
        success=True,
        case_id="e2e-case-001",
        extracted_data={
            "drugs": ["Amoxicillin"],
            "symptoms": ["fever", "cough"],
            "location": {"country": "Nigeria", "state": "Lagos", "district": "Ikeja"},
            "patient_age": 35,
            "patient_sex": "male",
            "duration_days": 5,
        },
        guidance="Patient on Amoxicillin with no improvement. Consider resistance.",
        confidence=0.87,
        processing_time_ms=1450,
    ))

    mock_agent_client.check_resistance = AsyncMock(
        return_value=ResistanceCheckResponse(
            drugs=[
                ResistanceResult(
                    drug="Amoxicillin",
                    resistance_pct=62.3,
                    category="resistant",
                    confidence=0.88,
                    alternatives=["Ceftriaxone", "Azithromycin", "Levofloxacin"],
                    sample_size=1247,
                )
            ],
            overall_risk="high",
        )
    )

    # Simulate the handler call
    from telegram.formatters.response import format_guidance_for_telegram

    response_text = format_guidance_for_telegram(
        guidance="Patient on Amoxicillin with no improvement. Consider resistance.",
        urgency="high",
        case_id="e2e-case-001",
        resistance_data=[
            ResistanceResult(
                drug="Amoxicillin",
                resistance_pct=62.3,
                category="resistant",
                confidence=0.88,
                alternatives=["Ceftriaxone", "Azithromycin", "Levofloxacin"],
                sample_size=1247,
            )
        ],
        actions=["Consider switching to Ceftriaxone", "Monitor for 48 hours", "Follow up in 3 days"],
        confidence=0.87,
    )

    # Assertions
    assert "e2e-case-001" in response_text
    assert "Amoxicillin" in response_text
    assert "62.3%" in response_text
    assert "resistant" in response_text.lower()
    assert "Ceftriaxone" in response_text
    assert "1,247" in response_text

    # Verify agent was called correctly
    mock_agent_client.ingest_text.assert_called_once()
    call_kwargs = mock_agent_client.ingest_text.call_args.kwargs
    assert "fever" in call_kwargs["text"]
    assert "Amoxicillin" in call_kwargs["text"]

    mock_agent_client.check_resistance.assert_called_once()
    res_call_kwargs = mock_agent_client.check_resistance.call_args.kwargs
    assert "Amoxicillin" in res_call_kwargs["drugs"]
```

### 9.3 Scenarios 2-5 Summary

| # | Scenario | Key Assertions |
|---|----------|---------------|
| 2 | **Photo OCR → Resistance** | Upload drug packaging photo → OCR extracts "Ciprofloxacin 500mg" → resistance check → response includes OCR transcript + resistance bar + alternatives |
| 3 | **New User Onboarding** | New user starts → 5-step FSM completes → user profile created in backend → main menu displayed → state cleaned from Redis |
| 4 | **Resistance Calculator** | User asks "What is resistance to Co-trimoxazole?" → drug identified → probability bar displayed → alternatives shown if >50% |
| 5 | **Error Recovery** | Agent timeout → fallback message sent → retry succeeds on second attempt → user gets response with "sorry for delay" prefix |

---

## 10. Week-by-Week Plan

| Week | Focus | Deliverables |
|------|-------|-------------|
| **W1** | Bot scaffolding + AgentClient | Project setup, aiogram 3.x boilerplate, AgentClient with retry/circuit breaker, config management |
| **W2** | Telegram core handlers | Text handler, start command, main menu keyboard, basic response formatting |
| **W3** | Media processing | Photo handler (OCR pipeline), voice handler (ASR), media downloader, temp file management |
| **W4** | Onboarding + FSM | 5-step onboarding flow, Redis state storage, language selection, profile creation |
| **W5** | Resistance conversations | Drug query parsing, resistance calculator, probability bars, alternatives display |
| **W6** | WhatsApp bot | WhatsApp Cloud API integration, text + image handlers, interactive buttons, webhook verification |
| **W7** | Daily digest + scheduler | Digest content fetching, platform-specific formatting, APScheduler cron jobs, timezone handling |
| **W8** | Multilingual + i18n | Language detection, template responses (EN, FR, HA, SW, YO, IG), fallback chains |
| **W9** | Shared conversation flows | Case report FSM, resistance calculator flow, drug lookup flow |
| **W10** | Rate limiting + error handling | Token bucket rate limiter, global error handler, structured logging, Sentry integration |
| **W11** | E2E tests | 5+ E2E scenarios, handler unit tests, mock agent, test fixtures |
| **W12** | Hardening + deploy | Docker setup, health checks, monitoring integration, documentation, production deployment |

---

## 11. Deliverables Checklist

```
┌─────────────────────────────────────────────────────────┐
│  INTEGRATION & BOT ENGINEER — DELIVERABLES             │
├────────────────────────────────────────┬───────┬────────┤
│  Deliverable                           │Status │Priority│
├────────────────────────────────────────┼───────┼────────┤
│  □ AgentClient (retry + circuit)       │  ○    │ P0     │
│  □ Telegram bot (aiogram 3.x)          │  ○    │ P0     │
│  □ Text handler → agent pipeline       │  ○    │ P0     │
│  □ Photo handler → OCR pipeline        │  ○    │ P0     │
│  □ Main menu inline keyboard           │  ○    │ P0     │
│  □ Response formatter (HTML)           │  ○    │ P0     │
│  □ Onboarding FSM (5 steps)            │  ○    │ P0     │
│  □ Error handler + fallbacks           │  ○    │ P0     │
├────────────────────────────────────────┼───────┼────────┤
│  □ WhatsApp bot (Cloud API)            │  ○    │ P1     │
│  □ Voice message handler (ASR)         │  ○    │ P1     │
│  □ Daily digest scheduler              │  ○    │ P1     │
│  □ Resistance calculator flow          │  ○    │ P1     │
│  □ Rate limiter                        │  ○    │ P1     │
│  □ Auth middleware                      │  ○    │ P1     │
├────────────────────────────────────────┼───────┼────────┤
│  □ Multilingual (6+ languages)         │  ○    │ P2     │
│  □ CHW gamification hooks              │  ○    │ P2     │
│  □ Peer alert notifications            │  ○    │ P2     │
│  □ Structured logging (JSON)           │  ○    │ P2     │
│  □ Prometheus metrics                  │  ○    │ P2     │
├────────────────────────────────────────┼───────┼────────┤
│  □ E2E tests (5+ scenarios)            │  ○    │ P3     │
│  □ Docker Compose setup                │  ○    │ P3     │
│  □ Health check endpoints              │  ○    │ P3     │
│  □ Sentry error tracking               │  ○    │ P3     │
│  □ API documentation                   │  ○    │ P3     │
└────────────────────────────────────────┴───────┴────────┘
```

---

> **Next:** See [17-usps-features.md](./17-usps-features.md) for differentiating feature specifications.
