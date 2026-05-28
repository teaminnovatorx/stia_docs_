# 08 — API Contracts

> **UDARA AI** — AMR Surveillance Platform for Sub-Saharan Africa  
> Document Version: 2.1.0 | Last Updated: 2026-05-27 | Status: Production

---

## Table of Contents

1. [Contract Philosophy](#1-contract-philosophy)
2. [Gateway API: Endpoints](#2-gateway-api-endpoints)
3. [Agent A Contract: Clinical Extraction](#3-agent-a-contract-clinical-extraction)
4. [Agent B Contract: Resistance Analytics](#4-agent-b-contract-resistance-analytics)
5. [Agent C Contract: Treatment Guidance](#5-agent-c-contract-treatment-guidance)
6. [Shared Pydantic Schemas](#6-shared-pydantic-schemas)
7. [Sync API](#7-sync-api)
8. [Error Handling & Status Codes](#8-error-handling--status-codes)
9. [OpenAPI Specification](#9-openapi-specification)
10. [Versioning & Deprecation Policy](#10-versioning--deprecation-policy)

---

## 1. Contract Philosophy

### 1.1 Design Principles

UDARA's API architecture follows strict design principles tailored to the
operational constraints of sub-Saharan Africa:

| Principle | Implementation |
|-----------|---------------|
| **REST over HTTP/JSON** | All inter-agent communication uses RESTful HTTP with JSON payloads. No gRPC (too complex for RPi), no GraphQL (too bandwidth-heavy). |
| **Gateway-mediated routing** | External clients never call agents directly. The gateway handles routing, auth, rate limiting, and request/response transformation. |
| **OpenAPI as source of truth** | Every endpoint has a corresponding OpenAPI 3.1 YAML in `contracts/`. Code is generated from these specs. |
| **Idempotent where possible** | All `POST` endpoints accept an optional `Idempotency-Key` header to prevent duplicate processing on unreliable networks. |
| **Versioned via URL prefix** | API versions are URL-based (`/v2/...`). Header-based versioning adds complexity for CHW-facing USSD. |
| **Graceful degradation** | Every endpoint returns a `degraded` response if a dependency (Agent B/C) is unavailable, rather than failing the entire request. |
| **Bandwidth-aware** | Response payloads are minimized. Large objects (images, audio) use pre-signed URLs to object storage, not inline base64. |

### 1.2 Architecture Diagram

```
┌─────────────────────────────────────────────────────────────────────┐
│                    UDARA API ARCHITECTURE                           │
│                                                                      │
│  EXTERNAL CLIENTS               API GATEWAY (:8000)                  │
│  ─────────────────              ──────────────────────               │
│                                                                      │
│  ┌──────────┐                 ┌──────────────────────────┐          │
│  │ CHW App  │──HTTP/JSON────>│  FastAPI Gateway          │          │
│  │ (React)  │                 │                          │          │
│  └──────────┘                 │  ┌────────┐  ┌────────┐  │          │
│  ┌──────────┐                 │  │ Auth   │  │ Rate   │  │          │
│  │ USSD     │──Webhook──────>│  │ MW     │  │ Limit  │  │          │
│  │ Gateway  │                 │  └────────┘  └────────┘  │          │
│  └──────────┘                 │  ┌────────┐  ┌────────┐  │          │
│  ┌──────────┐                 │  │ CORS   │  │ Logging│  │          │
│  │ Telegram │──Webhook──────>│  └────────┘  └────────┘  │          │
│  │ Bot      │                 │                          │          │
│  └──────────┘                 │  ┌─────────────────────┐  │          │
│  ┌──────────┐                 │  │   Route Dispatcher  │  │          │
│  │ WhatsApp │──Webhook──────>│  │                     │  │          │
│  │ Business │                 │  │  /ingest/*  → A    │  │          │
│  │ API      │                 │  │  /resistance/* → B │  │          │
│  └──────────┘                 │  │  /guidance/*  → C  │  │          │
│                                │  │  /sync/*     → DB │  │          │
│  ┌──────────┐                 │  └─────────────────────┘  │          │
│  │ National │──REST─────────>│                          │          │
│  │ Dashboard│                 └──┬───────┬───────┬──────┘          │
│  └──────────┘                    │       │       │                   │
│                                   ▼       ▼       ▼                   │
│                          ┌──────────┐┌──────────┐┌──────────┐      │
│                          │ Agent A  ││ Agent B  ││ Agent C  │      │
│                          │ :8001    ││ :8002    ││ :8003    │      │
│                          │ Clinical ││ Resistance││ Guidance │      │
│                          │ NLP      ││ Analytics││ Treatment│      │
│                          └──────────┘└──────────┘└──────────┘      │
└─────────────────────────────────────────────────────────────────────┘
```

### 1.3 Request Flow Example

```
CHW App sends:
  POST https://api.udara.health/v2/ingest/text
  Authorization: Bearer <jwt>
  X-Idempotency-Key: <uuid>
  Content-Type: application/json
  {
    "text": "28F, fever 5 days, dysuria, given ciprofloxacin 500mg bid",
    "language": "en",
    "source": "clinical_note",
    "facility_code": "KE-NRB-FAC-017"
  }

Gateway flow:
  1. Verify JWT token → extract chw_id, facility_code
  2. Check rate limit → allow (150/min per CHW)
  3. Validate request body against StructuredCase schema
  4. Forward to Agent A: POST http://agent-a:8001/ingest/text
  5. Agent A returns: {case_id, symptoms, drugs, diagnosis, amr_risk_score}
  6. Gateway stores case in TimescaleDB
  7. If amr_risk_score >= 0.5:
       a. Query Agent B: GET http://agent-b:8002/resistance/ciprofloxacin/district
       b. If resistance > threshold: query Agent C for alternatives
  8. Enqueue case to sync_queue for edge sync
  9. Return combined response to client
```

---

## 2. Gateway API: Endpoints

### 2.1 Gateway Endpoint Table

| Method | Path | Description | Agent | Auth | Rate Limit |
|--------|------|-------------|-------|------|------------|
| `POST` | `/v2/ingest/text` | Ingest clinical text note | A | JWT | 150/min |
| `POST` | `/v2/ingest/image` | Ingest clinical image (photo) | A | JWT | 30/min |
| `POST` | `/v2/ingest/audio` | Ingest voice recording | A | JWT | 30/min |
| `POST` | `/v2/ingest/multimodal` | Ingest combined text+image+audio | A | JWT | 20/min |
| `POST` | `/v2/ingest/ussd` | Ingest USSD session data | A | API Key | 200/min |
| `GET` | `/v2/resistance/{drug}/{district}` | Get resistance rate for drug+district | B | JWT | 300/min |
| `POST` | `/v2/resistance/update` | Trigger resistance recalculation | B | JWT | 10/min |
| `GET` | `/v2/resistance/alerts` | Get active AMR alerts | B | JWT | 60/min |
| `POST` | `/v2/guidance` | Get treatment recommendation | C | JWT | 60/min |
| `POST` | `/v2/webhooks/ussd` | USSD webhook from Africa's Talking | — | API Key | 500/min |
| `POST` | `/v2/webhooks/telegram` | Telegram bot webhook | — | API Key | 100/min |
| `POST` | `/v2/webhooks/whatsapp` | WhatsApp Business API webhook | — | API Key | 100/min |
| `GET` | `/v2/health` | Health check (all services) | — | None | None |
| `GET` | `/v2/metrics` | Prometheus metrics | — | Internal | None |

### 2.2 Gateway Ingest Endpoints

#### `POST /v2/ingest/text`

```bash
# Request
curl -X POST https://api.udara.health/v2/ingest/text \
  -H "Authorization: Bearer eyJhbGciOiJIUzI1NiJ9..." \
  -H "Content-Type: application/json" \
  -H "X-Idempotency-Key: 550e8400-e29b-41d4-a716-446655440000" \
  -d '{
    "text": "Patient is a 28 year old female presenting with fever for 5 days, dysuria, and flank pain. She was given ciprofloxacin 500mg twice daily for 7 days and paracetamol 1000mg QID for pain relief.",
    "language": "en",
    "source": "clinical_note",
    "facility_code": "KE-NRB-FAC-017",
    "chw_id": "chw-KE-NRB-0042",
    "metadata": {
      "encounter_date": "2026-05-27",
      "patient_age_group": "25-34"
    }
  }'
```

```json
// Response (200 OK)
{
  "status": "success",
  "case_id": "a1b2c3d4-e5f6-7890-abcd-ef1234567890",
  "extraction": {
    "patient_age_years": 28,
    "patient_sex": "F",
    "symptoms": [
      {"name": "fever", "category": "SYSTEMIC", "duration_days": 5, "severity": "moderate"},
      {"name": "dysuria", "category": "GENITOURINARY", "duration_days": 5, "severity": null},
      {"name": "flank_pain", "category": "GENITOURINARY", "duration_days": null, "severity": null}
    ],
    "drugs_prescribed": [
      {"name": "ciprofloxacin", "dose_mg": 500, "frequency": "bid", "duration_days": 7, "source": "prescription"},
      {"name": "paracetamol", "dose_mg": 1000, "frequency": "qid", "duration_days": null, "source": "prescription"}
    ],
    "diagnosis_suggestion": "urinary_tract_infection",
    "extraction_confidence": 0.91
  },
  "resistance_check": {
    "ciprofloxacin": {
      "district_rate": 0.681,
      "confidence_interval": [0.618, 0.739],
      "trend": "increasing",
      "threshold_exceeded": true,
      "threshold": 0.50
    }
  },
  "amr_risk_score": 0.72,
  "guidance": {
    "recommended": "nitrofurantoin 100mg qid x 5 days",
    "alternatives": ["fosfomycin 3g single dose", "pivmecillinam 400mg bid x 3 days"],
    "confidence": 0.85,
    "sources": ["WHO Essential Medicines List 2024", "Kenya AMR Surveillance Report 2024-W02"]
  },
  "warnings": [
    {
      "code": "HIGH_RESISTANCE",
      "message": "Ciprofloxacin resistance in Nairobi is 68.1% (above 50% threshold)",
      "severity": "HIGH"
    }
  ],
  "processing_time_ms": 342,
  "timestamp": "2026-05-27T10:30:00.000Z"
}
```

#### `POST /v2/ingest/image`

```bash
# Request (multipart/form-data)
curl -X POST https://api.udara.health/v2/ingest/image \
  -H "Authorization: Bearer eyJhbGciOiJIUzI1NiJ9..." \
  -F "file=@prescription_photo.jpg" \
  -F "source=prescription_photo" \
  -F "facility_code=KE-NRB-FAC-017" \
  -F "language=en"
```

```json
// Response (200 OK)
{
  "status": "success",
  "case_id": "b2c3d4e5-f6a7-8901-bcde-f12345678901",
  "extraction": {
    "patient_age_years": null,
    "patient_sex": "M",
    "symptoms": [],
    "drugs_prescribed": [
      {"name": "amoxicillin", "dose_mg": 500, "frequency": "tid", "duration_days": 7, "source": "prescription"}
    ],
    "diagnosis_suggestion": null,
    "extraction_confidence": 0.76
  },
  "image_analysis": {
    "type": "prescription",
    "ocr_text": "Amoxicillin 500mg TID x 7d",
    "quality_score": 0.82,
    "image_hash": "sha256:abc123..."
  },
  "amr_risk_score": 0.28,
  "processing_time_ms": 1820,
  "timestamp": "2026-05-27T10:31:00.000Z"
}
```

#### `POST /v2/ingest/audio`

```bash
# Request (multipart/form-data)
curl -X POST https://api.udara.health/v2/ingest/audio \
  -H "Authorization: Bearer eyJhbGciOiJIUzI1NiJ9..." \
  -F "file=@voice_note.ogg" \
  -F "source=voice_note" \
  -F "facility_code=KE-NRB-FAC-017" \
  -F "language=sw"
```

```json
// Response (200 OK)
{
  "status": "success",
  "case_id": "c3d4e5f6-a7b8-9012-cdef-123456789012",
  "extraction": {
    "patient_age_years": 34,
    "patient_sex": "F",
    "symptoms": [
      {"name": "fever", "category": "SYSTEMIC", "duration_days": 3, "severity": "moderate"},
      {"name": "headache", "category": "NEUROLOGICAL", "duration_days": 3, "severity": "mild"}
    ],
    "drugs_prescribed": [],
    "diagnosis_suggestion": null,
    "extraction_confidence": 0.68
  },
  "transcription": {
    "text": "Mgonjwa amekaa siku tatu, homa na kichwa inauma",
    "language_detected": "sw",
    "confidence": 0.82,
    "duration_seconds": 8.5
  },
  "amr_risk_score": 0.15,
  "processing_time_ms": 3200,
  "timestamp": "2026-05-27T10:32:00.000Z"
}
```

#### `POST /v2/ingest/multimodal`

```bash
# Request (multipart/form-data with multiple files)
curl -X POST https://api.udara.health/v2/ingest/multimodal \
  -H "Authorization: Bearer eyJhbGciOiJIUzI1NiJ9..." \
  -F "file=@voice_note.ogg" \
  -F "file=@photo.jpg" \
  -F "text=28F with fever and cough" \
  -F "source=multimodal" \
  -F "facility_code=KE-NRB-FAC-017" \
  -F "language=en"
```

```json
// Response (200 OK)
{
  "status": "success",
  "case_id": "d4e5f6a7-b8c9-0123-defa-234567890123",
  "extraction": {
    "patient_age_years": 28,
    "patient_sex": "F",
    "symptoms": [
      {"name": "fever", "category": "SYSTEMIC", "duration_days": null, "severity": "moderate"},
      {"name": "cough", "category": "RESPIRATORY", "duration_days": null, "severity": null}
    ],
    "drugs_prescribed": [],
    "diagnosis_suggestion": "respiratory_infection",
    "extraction_confidence": 0.88
  },
  "modality_results": {
    "text_extraction": {"symptoms_found": 2, "drugs_found": 0, "confidence": 0.85},
    "audio_transcription": {"transcribed": true, "symptoms_found": 0, "confidence": 0.68},
    "image_analysis": {"type": null, "symptoms_found": 0, "confidence": 0.0}
  },
  "fusion_strategy": "text_primary_audio_supplement",
  "amr_risk_score": 0.22,
  "processing_time_ms": 4100,
  "timestamp": "2026-05-27T10:33:00.000Z"
}
```

### 2.3 Webhook Endpoints

#### `POST /v2/webhooks/ussd` (Africa's Talking)

```bash
# Incoming from Africa's Talking when CHW sends USSD input
# Content-Type: application/x-www-form-urlencoded

curl -X POST https://api.udara.health/v2/webhooks/ussd \
  -d "sessionId=ATUid_1234567890abcdef" \
  -d "serviceCode=*384*12345#" \
  -d "phoneNumber=+254712345678" \
  -d "text=1" \
  -d "networkCode=63902"
```

```json
// Response (200 OK) — Africa's Talking expects plain text
// This is NOT JSON — it's plain text that gets sent to the CHW's phone
// Content-Type: text/plain
//
// CON (continue) prefix keeps the session open
// END prefix terminates the session
//
// Example response body:
"CON Welcome to UDARA AMR Surveillance\n1. Report case\n2. Check resistance\n3. Help\n\nReply 1,2,3"
```

---

## 3. Agent A Contract: Clinical Extraction

### 3.1 Agent A Internal Endpoints

These endpoints are called by the gateway, never directly by external clients.

#### `POST /ingest/text`

```python
# Agent A internal contract (gateway → agent-a)

# REQUEST
{
    "request_id": "uuid-v4",              # Gateway-assigned tracking ID
    "text": "28F, fever 5 days, dysuria, given ciprofloxacin 500mg bid",
    "language": "en",
    "source": "clinical_note",
    "facility_code": "KE-NRB-FAC-017",
    "chw_id": "chw-KE-NRB-0042",
    "metadata": {
        "encounter_date": "2026-05-27",
        "patient_age_group": "25-34"
    }
}

# RESPONSE (200)
{
    "request_id": "uuid-v4",
    "case_id": "uuid-v4",                  # Agent A generates the case UUID
    "status": "success",
    "patient": {
        "age_years": 28,
        "sex": "F",
        "age_group": "25-34"
    },
    "symptoms": [
        {
            "name": "fever",
            "category": "SYSTEMIC",
            "duration_days": 5,
            "severity": "moderate",
            "onset": null,
            "location": null,
            "confidence": 0.95
        },
        {
            "name": "dysuria",
            "category": "GENITOURINARY",
            "duration_days": null,
            "severity": null,
            "onset": null,
            "location": null,
            "confidence": 0.88
        }
    ],
    "drugs_prescribed": [
        {
            "name": "ciprofloxacin",
            "generic_name": "ciprofloxacin",
            "dose_mg": 500,
            "frequency": "bid",
            "duration_days": null,
            "source": "prescription",
            "route": "oral",
            "confidence": 0.92
        }
    ],
    "diagnosis_suggestion": "urinary_tract_infection",
    "diagnosis_confidence": 0.76,
    "extraction_confidence": 0.91,        # Overall confidence score
    "amr_risk_score": 0.72,               # 0-1: higher = more AMR concern
    "model_version": "agent-a-v2.3.1",
    "processing_time_ms": 180
}
```

#### `POST /ingest/image`

```python
# REQUEST (gateway forwards multipart files to Agent A)
# Content-Type: multipart/form-data
# Fields: file (binary), source (string), facility_code (string), language (string)

# RESPONSE (200)
{
    "request_id": "uuid-v4",
    "case_id": "uuid-v4",
    "status": "success",
    "patient": {
        "age_years": null,
        "sex": "M",
        "age_group": null
    },
    "symptoms": [],
    "drugs_prescribed": [
        {
            "name": "amoxicillin",
            "dose_mg": 500,
            "frequency": "tid",
            "duration_days": 7,
            "source": "prescription",
            "route": "oral",
            "confidence": 0.85
        }
    ],
    "image_analysis": {
        "type": "prescription",
        "ocr_text": "Amoxicillin 500mg TID x 7d\nDr. Ochieng",
        "quality_score": 0.82,
        "image_dimensions": {"width": 3024, "height": 4032},
        "image_hash": "sha256:abc123def456..."
    },
    "extraction_confidence": 0.76,
    "amr_risk_score": 0.28,
    "model_version": "agent-a-v2.3.1",
    "processing_time_ms": 1450
}
```

#### `POST /ingest/audio`

```python
# REQUEST (gateway forwards audio file to Agent A)
# Content-Type: multipart/form-data
# Fields: file (binary), source (string), facility_code (string), language (string)

# RESPONSE (200)
{
    "request_id": "uuid-v4",
    "case_id": "uuid-v4",
    "status": "success",
    "patient": {
        "age_years": 34,
        "sex": "F",
        "age_group": "25-34"
    },
    "transcription": {
        "text": "Mgonjwa amekaa siku tatu, homa na kichwa inauma",
        "language_detected": "sw",
        "language_confidence": 0.93,
        "segments": [
            {"start": 0.0, "end": 2.1, "text": "Mgonjwa amekaa siku tatu"},
            {"start": 2.3, "end": 4.5, "text": "homa na kichwa inauma"}
        ],
        "duration_seconds": 8.5
    },
    "symptoms": [
        {
            "name": "fever",
            "category": "SYSTEMIC",
            "duration_days": 3,
            "severity": "moderate",
            "confidence": 0.78
        },
        {
            "name": "headache",
            "category": "NEUROLOGICAL",
            "duration_days": 3,
            "severity": "mild",
            "confidence": 0.72
        }
    ],
    "drugs_prescribed": [],
    "extraction_confidence": 0.68,
    "amr_risk_score": 0.15,
    "model_version": "agent-a-v2.3.1",
    "processing_time_ms": 2800
}
```

---

## 4. Agent B Contract: Resistance Analytics

### 4.1 `GET /resistance/{drug}/{district}`

Get current resistance rate for a specific drug in a specific district.

```bash
# Request
curl -X GET "https://api.udara.health/v2/resistance/ciprofloxacin/Nairobi?age_group=25-34&county=Nairobi" \
  -H "Authorization: Bearer eyJhbGciOiJIUzI1NiJ9..."
```

```json
// Response (200 OK)
{
  "status": "success",
  "drug": "ciprofloxacin",
  "district": "Nairobi",
  "county": "Nairobi",
  "country": "KEN",
  "current_period": {
    "start": "2025-01-06",
    "end": "2025-01-12",
    "iso_week": "2025-W02"
  },
  "estimate": {
    "rate": 0.681,
    "rate_percentage": 68.1,
    "confidence_interval": {
      "lower": 0.618,
      "upper": 0.739,
      "level": 0.95,
      "method": "wilson_score"
    },
    "total_cases": 47,
    "resistant_cases": 32,
    "sample_size_met": true,
    "minimum_sample_size": 30
  },
  "trend": {
    "direction": "increasing",
    "slope": 0.034,
    "previous_week_rate": 0.647,
    "change_percentage": 5.3,
    "weeks_analyzed": 12,
    "r_squared": 0.72
  },
  "threshold": {
    "rate": 0.50,
    "exceeded": true,
    "severity": "HIGH"
  },
  "geographic_coverage": {
    "facilities_contributing": 8,
    "total_facilities_in_district": 12,
    "coverage_percentage": 66.7
  },
  "last_computed": "2026-05-27T10:29:45.000Z",
  "cache_ttl_seconds": 300
}
```

### 4.2 `POST /resistance/update`

Trigger recalculation of resistance estimates. Used after a batch of new cases is synced.

```bash
# Request
curl -X POST https://api.udara.health/v2/resistance/update \
  -H "Authorization: Bearer eyJhbGciOiJIUzI1NiJ9..." \
  -H "Content-Type: application/json" \
  -d '{
    "drugs": ["ciprofloxacin", "amoxicillin", "nitrofurantoin"],
    "districts": ["Nairobi", "Kisumu"],
    "period_start": "2025-01-06",
    "period_end": "2025-01-12"
  }'
```

```json
// Response (202 Accepted)
{
  "status": "accepted",
  "job_id": "job-abc123",
  "message": "Resistance recalculation queued for 6 drug-district combinations",
  "estimated_time_seconds": 15
}
```

### 4.3 `GET /resistance/alerts`

Get all active AMR alerts, sorted by severity.

```bash
# Request
curl -X GET "https://api.udara.health/v2/resistance/alerts?severity=HIGH&district=Nairobi" \
  -H "Authorization: Bearer eyJhbGciOiJIUzI1NiJ9..."
```

```json
// Response (200 OK)
{
  "status": "success",
  "alerts": [
    {
      "id": "alert-2025-KE-NRB-001",
      "alert_type": "RESISTANCE_THRESHOLD",
      "severity": "HIGH",
      "drug": "ciprofloxacin",
      "district": "Nairobi",
      "current_rate": 0.681,
      "threshold_rate": 0.50,
      "trend": "increasing",
      "message": "Ciprofloxacin resistance in Nairobi (68.1%) exceeds 50% threshold and is increasing",
      "recommended_alternatives": ["nitrofurantoin", "fosfomycin", "pivmecillinam"],
      "facilities_affected": 8,
      "created_at": "2026-05-27T08:00:00.000Z",
      "acknowledged": false
    },
    {
      "id": "alert-2025-KE-NRB-002",
      "alert_type": "RESISTANCE_THRESHOLD",
      "severity": "HIGH",
      "drug": "amoxicillin",
      "district": "Nairobi",
      "current_rate": 0.452,
      "threshold_rate": 0.40,
      "trend": "stable",
      "message": "Amoxicillin resistance in Nairobi (45.2%) exceeds 40% threshold",
      "recommended_alternatives": ["amoxicillin-clavulanate", "doxycycline"],
      "facilities_affected": 10,
      "created_at": "2025-01-14T08:00:00.000Z",
      "acknowledged": true,
      "acknowledged_by": "dr-adm-001",
      "acknowledged_at": "2025-01-14T10:30:00.000Z"
    }
  ],
  "total": 2,
  "unacknowledged": 1,
  "filters_applied": {"severity": "HIGH", "district": "Nairobi"}
}
```

---

## 5. Agent C Contract: Treatment Guidance

### 5.1 `POST /guidance`

Get evidence-based treatment guidance for a clinical scenario.

```bash
# Request
curl -X POST https://api.udara.health/v2/guidance \
  -H "Authorization: Bearer eyJhbGciOiJIUzI1NiJ9..." \
  -H "Content-Type: application/json" \
  -d '{
    "case_id": "a1b2c3d4-e5f6-7890-abcd-ef1234567890",
    "patient": {
      "age_years": 28,
      "sex": "F",
      "age_group": "25-34",
      "pregnancy_status": "not_pregnant",
      "allergies": [],
      "renal_impairment": false
    },
    "diagnosis": {
      "suggestion": "urinary_tract_infection",
      "confidence": 0.76
    },
    "symptoms": [
      {"name": "fever", "category": "SYSTEMIC", "duration_days": 5},
      {"name": "dysuria", "category": "GENITOURINARY", "duration_days": 5}
    ],
    "current_drug": {
      "name": "ciprofloxacin",
      "dose_mg": 500,
      "frequency": "bid"
    },
    "resistance_context": {
      "drug": "ciprofloxacin",
      "district": "Nairobi",
      "rate": 0.681,
      "trend": "increasing",
      "threshold_exceeded": true
    },
    "context": {
      "facility_level": "primary",
      "country": "KEN",
      "district": "Nairobi",
      "availability": ["nitrofurantoin", "fosfomycin", "trimethoprim-sulfamethoxazole"]
    }
  }'
```

```json
// Response (200 OK)
{
  "status": "success",
  "guidance_id": "guid-uuid-v4",
  "case_id": "a1b2c3d4-e5f6-7890-abcd-ef1234567890",
  "recommendation": {
    "action": "SWITCH_RECOMMENDED",
    "confidence": 0.87,
    "reasoning": "Ciprofloxacin resistance in Nairobi (68.1%) exceeds WHO threshold of 20% for UTI. Nitrofurantoin has low resistance (12%) and is WHO first-line for uncomplicated UTI."
  },
  "primary_recommendation": {
    "drug": "nitrofurantoin",
    "dose_mg": 100,
    "frequency": "qid",
    "duration_days": 5,
    "route": "oral",
    "resistance_rate": 0.12,
    "resistance_trend": "stable",
    "evidence_level": "A",
    "who_essential": true
  },
  "alternatives": [
    {
      "drug": "fosfomycin",
      "dose_mg": 3000,
      "frequency": "single_dose",
      "duration_days": 1,
      "route": "oral",
      "resistance_rate": 0.08,
      "resistance_trend": "stable",
      "evidence_level": "A",
      "who_essential": true,
      "note": "Convenient single-dose option"
    },
    {
      "drug": "pivmecillinam",
      "dose_mg": 400,
      "frequency": "bid",
      "duration_days": 3,
      "route": "oral",
      "resistance_rate": null,
      "resistance_trend": "unknown",
      "evidence_level": "B",
      "who_essential": false,
      "note": "Insufficient local resistance data"
    }
  ],
  "current_drug_assessment": {
    "drug": "ciprofloxacin",
    "resistance_rate": 0.681,
    "recommendation": "AVOID",
    "reason": "Resistance exceeds 50% — high probability of treatment failure"
  },
  "sources": [
    {
      "name": "WHO Model List of Essential Medicines",
      "version": "2024",
      "url": "https://www.who.int/publications/i/item/WHO-MVP-EMP-IAU-2023.01"
    },
    {
      "name": "Kenya AMR Surveillance Report",
      "version": "2025-W02",
      "url": null
    },
    {
      "name": "IDSA UTI Treatment Guidelines",
      "version": "2023",
      "url": "https://www.idsociety.org/practice-guidelines/"
    }
  ],
  "warnings": [
    {
      "code": "ALLERGY_CHECK",
      "message": "Verify patient has no nitrofurantoin allergy before prescribing",
      "severity": "MEDIUM"
    },
    {
      "code": "RENAL_CHECK",
      "message": "Nitrofurantoin contraindicated if eGFR < 30 mL/min",
      "severity": "HIGH"
    },
    {
      "code": "DATA_QUALITY",
      "message": "Resistance data for pivmecillinam in this district is limited",
      "severity": "LOW"
    }
  ],
  "model_version": "agent-c-v1.5.0",
  "processing_time_ms": 95
}
```

---

## 6. Shared Pydantic Schemas

### 6.1 Core Schemas

```python
# shared/schemas/base.py

from datetime import date, datetime
from enum import Enum
from typing import List, Optional, Dict, Any
from pydantic import BaseModel, Field, field_validator
from pydantic.config import ConfigDict


class SymptomCategory(str, Enum):
    """Standardized symptom categories for AMR-relevant conditions."""
    SYSTEMIC = "SYSTEMIC"                     # Fever, malaise, fatigue
    RESPIRATORY = "RESPIRATORY"               # Cough, dyspnea, sputum
    GASTROINTESTINAL = "GASTROINTESTINAL"     # Diarrhea, nausea, vomiting
    GENITOURINARY = "GENITOURINARY"           # Dysuria, frequency, flank pain
    SKIN_SOFT_TISSUE = "SKIN_SOFT_TISSUE"     # Wound infection, cellulitis
    NEUROLOGICAL = "NEUROLOGICAL"             # Headache, altered consciousness
    ENT = "ENT"                               # Ear pain, sore throat, rhinitis
    OTHER = "OTHER"


class SeverityLevel(str, Enum):
    """Symptom severity classification."""
    MILD = "mild"
    MODERATE = "moderate"
    SEVERE = "severe"
    CRITICAL = "critical"


class DrugSource(str, Enum):
    """How the drug was obtained/prescribed."""
    PRESCRIPTION = "prescription"
    OTC = "otc"                               # Over-the-counter
    TRADITIONAL = "traditional"               # Traditional/herbal medicine
    SHARED = "shared"                         # Shared from friend/family
    LEFTOVER = "leftover"                     # Leftover from previous illness
    UNKNOWN = "unknown"


class DrugRoute(str, Enum):
    """Drug administration route."""
    ORAL = "oral"
    INTRAVENOUS = "intravenous"
    INTRAMUSCULAR = "intramuscular"
    TOPICAL = "topical"
    RECTAL = "rectal"
    INHALATION = "inhalation"
    SUBCUTANEOUS = "subcutaneous"


class ExtractedSymptom(BaseModel):
    """A single symptom extracted from a clinical encounter."""
    name: str = Field(..., min_length=1, max_length=200, description="Canonical symptom name")
    category: SymptomCategory = Field(..., description="Standardized category")
    duration_days: Optional[int] = Field(None, ge=0, le=365, description="Duration in days")
    severity: Optional[SeverityLevel] = Field(None, description="Severity level")
    onset: Optional[str] = Field(None, description="Acute/subacute/chronic")
    location: Optional[str] = Field(None, max_length=200, description="Body location")
    associated: Optional[List[str]] = Field(default_factory=list, description="Related symptoms")
    confidence: float = Field(..., ge=0.0, le=1.0, description="Extraction confidence")

    model_config = ConfigDict(
        json_schema_extra={
            "examples": [
                {
                    "name": "fever",
                    "category": "SYSTEMIC",
                    "duration_days": 5,
                    "severity": "moderate",
                    "onset": "acute",
                    "location": None,
                    "associated": ["headache", "chills"],
                    "confidence": 0.95
                }
            ]
        }
    )


class ExtractedDrug(BaseModel):
    """A drug extracted from a clinical encounter."""
    name: str = Field(..., min_length=1, max_length=200, description="Drug name (generic preferred)")
    generic_name: Optional[str] = Field(None, max_length=200, description="Standard generic name")
    dose_mg: Optional[int] = Field(None, gt=0, le=5000, description="Dose in milligrams")
    frequency: Optional[str] = Field(None, max_length=50, description="Dosing frequency (e.g., 'bid', 'tid')")
    duration_days: Optional[int] = Field(None, ge=1, le=365, description="Treatment duration in days")
    source: DrugSource = Field(DrugSource.UNKNOWN, description="How the drug was obtained")
    route: DrugRoute = Field(DrugRoute.ORAL, description="Administration route")
    brand_name: Optional[str] = Field(None, max_length=200, description="Brand name if known")
    confidence: float = Field(0.5, ge=0.0, le=1.0, description="Extraction confidence")

    @field_validator("name")
    @classmethod
    def normalize_drug_name(cls, v: str) -> str:
        """Normalize drug name to lowercase with underscores."""
        return v.lower().strip().replace(" ", "_")


class PatientInfo(BaseModel):
    """Anonymized patient demographics."""
    age_years: Optional[int] = Field(None, ge=0, le=120)
    sex: Optional[str] = Field(None, pattern=r"^[MFOU]$")
    age_group: Optional[str] = Field(None, max_length=20)
    pregnancy_status: Optional[str] = Field(None, description="pregnant, lactating, not_pregnant, unknown")
    allergies: Optional[List[str]] = Field(default_factory=list)
    renal_impairment: Optional[bool] = None
    hepatic_impairment: Optional[bool] = None


class StructuredCase(BaseModel):
    """A fully structured clinical case, as produced by Agent A."""
    case_id: str = Field(..., description="UUID of the case")
    facility_code: str = Field(..., min_length=1, max_length=50)
    district: str = Field(..., min_length=1, max_length=100)
    county: Optional[str] = Field(None, max_length=100)
    country: str = Field("KEN", min_length=2, max_length=10)

    patient: PatientInfo

    encounter_date: date
    source_type: str = Field("clinical_note")
    language: str = Field("en", min_length=2, max_length=10)

    symptoms: List[ExtractedSymptom] = Field(default_factory=list)
    drugs_prescribed: List[ExtractedDrug] = Field(default_factory=list)
    diagnosis_suggestion: Optional[str] = Field(None, max_length=500)
    diagnosis_confidence: Optional[float] = Field(None, ge=0.0, le=1.0)
    clinical_notes: Optional[str] = Field(None, max_length=5000)

    amr_risk_score: float = Field(..., ge=0.0, le=1.0)
    extraction_confidence: float = Field(..., ge=0.0, le=1.0)
    extracted_by: str = Field(..., description="Agent version that extracted this case")

    geohash: Optional[str] = Field(None, min_length=4, max_length=12)

    model_config = ConfigDict(
        json_schema_extra={
            "examples": [{
                "case_id": "a1b2c3d4-e5f6-7890-abcd-ef1234567890",
                "facility_code": "KE-NRB-FAC-017",
                "district": "Nairobi",
                "county": "Nairobi",
                "country": "KEN",
                "patient": {
                    "age_years": 28,
                    "sex": "F",
                    "age_group": "25-34",
                    "pregnancy_status": "not_pregnant",
                    "allergies": [],
                    "renal_impairment": False
                },
                "encounter_date": "2026-05-27",
                "source_type": "clinical_note",
                "language": "en",
                "symptoms": [
                    {
                        "name": "fever",
                        "category": "SYSTEMIC",
                        "duration_days": 5,
                        "severity": "moderate",
                        "onset": "acute",
                        "location": None,
                        "associated": ["headache"],
                        "confidence": 0.95
                    }
                ],
                "drugs_prescribed": [
                    {
                        "name": "ciprofloxacin",
                        "generic_name": "ciprofloxacin",
                        "dose_mg": 500,
                        "frequency": "bid",
                        "duration_days": 7,
                        "source": "prescription",
                        "route": "oral",
                        "brand_name": None,
                        "confidence": 0.92
                    }
                ],
                "diagnosis_suggestion": "urinary_tract_infection",
                "diagnosis_confidence": 0.76,
                "clinical_notes": None,
                "amr_risk_score": 0.72,
                "extraction_confidence": 0.91,
                "extracted_by": "agent-a-v2.3.1",
                "geohash": "k17fqv"
            }]
        }
    )


class ResistanceEstimate(BaseModel):
    """Resistance rate estimate for a drug in a location and time period."""
    drug: str
    district: str
    county: Optional[str] = None
    country: str = "KEN"
    period_start: date
    period_end: date
    iso_year_week: str

    rate: float = Field(..., ge=0.0, le=1.0)
    ci_lower: float = Field(..., ge=0.0, le=1.0)
    ci_upper: float = Field(..., ge=0.0, le=1.0)
    confidence_level: float = Field(0.95, description="CI confidence level")

    total_cases: int = Field(..., ge=0)
    resistant_cases: int = Field(..., ge=0)
    sample_size_met: bool = False
    minimum_sample_size: int = 30

    trend: Optional[str] = Field(None, pattern=r"^(increasing|stable|decreasing|unknown|new)$")
    trend_slope: Optional[float] = None
    previous_week_rate: Optional[float] = None
    change_percentage: Optional[float] = None

    computed_at: datetime


class GuidanceResponse(BaseModel):
    """Treatment guidance response from Agent C."""
    guidance_id: str
    case_id: str
    recommendation: Dict[str, Any] = Field(..., description="Primary recommendation with action and reasoning")
    primary_recommendation: Dict[str, Any] = Field(..., description="Recommended drug details")
    alternatives: List[Dict[str, Any]] = Field(default_factory=list, description="Alternative drug options")
    current_drug_assessment: Optional[Dict[str, Any]] = Field(None)
    sources: List[Dict[str, str]] = Field(default_factory=list, description="Evidence sources")
    warnings: List[Dict[str, str]] = Field(default_factory=list)
    model_version: str


class IngestTextRequest(BaseModel):
    """Request body for POST /v2/ingest/text."""
    text: str = Field(..., min_length=1, max_length=10000, description="Clinical text to process")
    language: str = Field("en", pattern=r"^(en|sw|am|fr|pt|ar)$")
    source: str = Field("clinical_note", pattern=r"^(clinical_note|voice|image|ussd|multimodal)$")
    facility_code: str = Field(..., min_length=1, max_length=50)
    chw_id: Optional[str] = Field(None, max_length=100)
    metadata: Optional[Dict[str, Any]] = Field(None)


class IngestResponse(BaseModel):
    """Standard response for all ingest endpoints."""
    status: str = Field(..., pattern=r"^(success|partial|error)$")
    case_id: str
    extraction: Optional[Dict[str, Any]] = None
    resistance_check: Optional[Dict[str, Any]] = None
    guidance: Optional[Dict[str, Any]] = None
    warnings: Optional[List[Dict[str, Any]]] = Field(default_factory=list)
    processing_time_ms: int
    timestamp: datetime


class ErrorResponse(BaseModel):
    """Standard error response."""
    status: str = Field("error")
    error_code: str = Field(..., min_length=1, max_length=50, description="Machine-readable error code")
    message: str = Field(..., min_length=1, max_length=500, description="Human-readable error message")
    details: Optional[Dict[str, Any]] = Field(None, description="Additional error context")
    request_id: Optional[str] = Field(None, description="Request tracking ID for debugging")
    timestamp: datetime = Field(default_factory=lambda: datetime.utcnow())
```

---

## 7. Sync API

### 7.1 `POST /v2/sync/push`

Receive a sync batch from an edge device. This is the primary edge→cloud sync endpoint.

```bash
# Request
curl -X POST https://api.udara.health/v2/sync/push \
  -H "Authorization: Bearer <device-api-key>" \
  -H "Content-Type: application/json" \
  -d '{
    "batch_id": "b7f3c291-8a4e-4d6f-b912-3e8c5f0a1d72",
    "device_id": "udara-edge-KE-NRB-0042",
    "facility_code": "KE-NRB-FAC-017",
    "sync_version": "2.1.0",
    "timestamp": "2026-05-27T10:30:00.000Z",
    "encryption": {
      "algorithm": "AES-256-GCM",
      "salt": "a1b2c3d4e5f6a7b8c9d0e1f2a3b4c5d6",
      "iv": "f6e5d4c3b2a1e0f9",
      "auth_tag": "9876543210abcdef9876543210abcdef"
    },
    "payload_b64": "<base64-encoded-encrypted-gzip-json>",
    "payload_sha256": "sha256-hex-digest",
    "record_count": 3,
    "compression": "gzip",
    "gzip_level": 6
  }'
```

```json
// Response (200 OK)
{
  "status": "accepted",
  "batch_id": "b7f3c291-8a4e-4d6f-b912-3e8c5f0a1d72",
  "applied": 2,
  "rejected": 0,
  "conflicts": [
    {
      "record_id": "case-2025-KE-NRB-00391",
      "field": "drugs_prescribed",
      "action": "union_merge",
      "detail": "Added 1 drug entry from remote that was not in local copy"
    }
  ],
  "processing_time_ms": 45,
  "server_timestamp": "2026-05-27T10:30:01.234Z"
}
```

### 7.2 `GET /v2/sync/status`

Get sync status for a specific device.

```bash
# Request
curl -X GET "https://api.udara.health/v2/sync/status?device_id=udara-edge-KE-NRB-0042" \
  -H "Authorization: Bearer <device-api-key>"
```

```json
// Response (200 OK)
{
  "device_id": "udara-edge-KE-NRB-0042",
  "facility_code": "KE-NRB-FAC-017",
  "last_sync": "2026-05-27T10:30:01.000Z",
  "last_heartbeat": "2026-05-27T10:35:00.000Z",
  "queue": {
    "total_records": 12,
    "priority_1_alerts": 0,
    "priority_2_cases": 5,
    "priority_5_resistance": 7,
    "oldest_record_age_minutes": 180
  },
  "connectivity_tier": "3",
  "estimated_bandwidth_kbps": 45,
  "model_versions": {
    "agent-a": "2.3.1",
    "agent-b": "1.8.0",
    "agent-c": "1.5.0"
  },
  "system": {
    "disk_free_mb": 8420,
    "memory_available_mb": 1820,
    "cpu_temperature_c": 52.3,
    "uptime_seconds": 864000
  }
}
```

### 7.3 `GET /v2/sync/models`

Get available model updates for edge devices.

```bash
# Request
curl -X GET "https://api.udara.health/v2/sync/models?current_agent_a=2.3.1&current_agent_b=1.7.2&arch=arm64" \
  -H "Authorization: Bearer <device-api-key>"
```

```json
// Response (200 OK)
{
  "models": [
    {
      "name": "agent-a-clinical",
      "current_version": "2.3.1",
      "latest_version": "2.4.0",
      "update_available": true,
      "download_url": "https://models.udara.health/agent-a/v2.4.0/model.onnx",
      "sha256": "a3f8e2b1c4d5e6f7a8b9c0d1e2f3a4b5c6d7e8f9a0b1c2d3e4f5a6b7c8d9e0",
      "file_size_mb": 138,
      "changelog": "Improved fever detection +3.2%, new Swahili symptom terms",
      "mandatory": false,
      "rollback_safe": true
    },
    {
      "name": "agent-b-resistance",
      "current_version": "1.7.2",
      "latest_version": "1.8.0",
      "update_available": true,
      "download_url": "https://models.udara.health/agent-b/v1.8.0/model.onnx",
      "sha256": "b4c9f3e2d1a0b8c7d6e5f4a3b2c1d0e9f8a7b6c5d4e3f2a1b0c9d8e7f6a5b4",
      "file_size_mb": 43,
      "changelog": "New trend calculation algorithm",
      "mandatory": true,
      "rollback_safe": true
    },
    {
      "name": "agent-c-guidance",
      "current_version": "1.5.0",
      "latest_version": "1.5.0",
      "update_available": false
    }
  ]
}
```

---

## 8. Error Handling & Status Codes

### 8.1 HTTP Status Code Strategy

| Status Code | Meaning | When Used |
|-------------|---------|-----------|
| `200` | OK | Successful GET/POST with result |
| `201` | Created | Resource created (new case, new alert) |
| `202` | Accepted | Async operation accepted (resistance recalc) |
| `400` | Bad Request | Invalid request body, missing fields, bad enum |
| `401` | Unauthorized | Missing or invalid JWT/token |
| `403` | Forbidden | Valid auth but insufficient permissions |
| `404` | Not Found | Resource not found (case, drug, district) |
| `409` | Conflict | Idempotency key already used (duplicate request) |
| `413` | Payload Too Large | File exceeds 10MB limit |
| `422` | Unprocessable Entity | Valid JSON but fails business validation |
| `429` | Too Many Requests | Rate limit exceeded |
| `500` | Internal Server Error | Unexpected server error |
| `502` | Bad Gateway | Upstream agent unreachable |
| `503` | Service Unavailable | Agent degraded or in maintenance |
| `504` | Gateway Timeout | Upstream agent timed out |

### 8.2 Error Response Format

```json
{
  "status": "error",
  "error_code": "INVALID_LANGUAGE",
  "message": "Unsupported language code: 'de'. Supported: en, sw, am, fr, pt, ar.",
  "details": {
    "field": "language",
    "provided_value": "de",
    "valid_options": ["en", "sw", "am", "fr", "pt", "ar"]
  },
  "request_id": "req-550e8400-e29b-41d4-a716-446655440000",
  "timestamp": "2026-05-27T10:30:00.000Z"
}
```

### 8.3 Error Code Registry

| Error Code | HTTP Status | Description |
|------------|-------------|-------------|
| `AUTH_REQUIRED` | 401 | Missing Authorization header |
| `AUTH_INVALID` | 401 | JWT token invalid or expired |
| `AUTH_FORBIDDEN` | 403 | Insufficient permissions for this action |
| `RATE_LIMITED` | 429 | Rate limit exceeded; retry after `Retry-After` seconds |
| `INVALID_REQUEST` | 400 | Generic request validation failure |
| `INVALID_LANGUAGE` | 400 | Language code not supported |
| `INVALID_FACILITY` | 400 | Facility code not recognized |
| `INVALID_DRUG` | 400 | Drug name not in knowledge base |
| `INVALID_DISTRICT` | 404 | District code not found |
| `TEXT_TOO_LONG` | 400 | Input text exceeds 10,000 character limit |
| `TEXT_TOO_SHORT` | 400 | Input text is empty or too short |
| `FILE_TOO_LARGE` | 413 | Uploaded file exceeds size limit |
| `UNSUPPORTED_MEDIA_TYPE` | 400 | File format not supported |
| `IDEMPOTENCY_CONFLICT` | 409 | Idempotency-Key already used |
| `EXTRACTION_FAILED` | 422 | Agent A could not extract clinical data |
| `EXTRACTION_LOW_CONFIDENCE` | 422 | Extraction confidence below threshold |
| `RESISTANCE_DATA_UNAVAILABLE` | 404 | No resistance data for this drug/district |
| `RESISTANCE_INSUFFICIENT_SAMPLE` | 422 | Sample size below minimum threshold |
| `GUIDANCE_UNAVAILABLE` | 503 | Agent C is unavailable |
| `AGENT_UNREACHABLE` | 502 | Upstream agent not responding |
| `AGENT_TIMEOUT` | 504 | Upstream agent response timed out |
| `DEGRADED_RESPONSE` | 200 (with warning) | Partial response due to dependency failure |
| `SYNC_DECRYPTION_FAILED` | 400 | Could not decrypt sync payload |
| `SYNC_INTEGRITY_FAILED` | 400 | Payload SHA256 mismatch |
| `SYNC_BATCH_CONFLICT` | 409 | Batch ID already processed |

### 8.4 Degraded Response Pattern

When a dependency agent is unavailable, the gateway returns a degraded response
rather than failing the entire request. The `degraded` flag and `degraded_services`
field inform the client.

```json
{
  "status": "success",
  "case_id": "a1b2c3d4-e5f6-7890-abcd-ef1234567890",
  "extraction": {
    "symptoms": [...],
    "drugs_prescribed": [...],
    "diagnosis_suggestion": "urinary_tract_infection",
    "extraction_confidence": 0.91
  },
  "amr_risk_score": 0.72,
  "degraded": true,
  "degraded_services": ["resistance_check", "guidance"],
  "degradation_reason": "Agent B and Agent C are currently unreachable",
  "warnings": [
    {
      "code": "DEGRADED_RESPONSE",
      "message": "Resistance data and treatment guidance unavailable. Try again later.",
      "severity": "MEDIUM"
    }
  ],
  "processing_time_ms": 180
}
```

---

## 9. OpenAPI Specification

### 9.1 File Organization

```
contracts/
├── openapi.yaml                  # Root: $ref to all sub-specs
├── gateway/
│   ├── ingest.yaml               # /v2/ingest/* endpoints
│   ├── resistance.yaml           # /v2/resistance/* endpoints
│   ├── guidance.yaml             # /v2/guidance endpoint
│   └── webhooks.yaml             # /v2/webhooks/* endpoints
├── agents/
│   ├── agent-a.yaml              # Agent A internal API
│   ├── agent-b.yaml              # Agent B internal API
│   └── agent-c.yaml              # Agent C internal API
├── sync/
│   └── sync.yaml                 # /v2/sync/* endpoints
├── schemas/
│   ├── case.yaml                 # StructuredCase, ExtractedSymptom, etc.
│   ├── resistance.yaml           # ResistanceEstimate
│   ├── guidance.yaml             # GuidanceResponse
│   ├── errors.yaml               # ErrorResponse, error codes
│   └── common.yaml               # Shared types (pagination, etc.)
└── examples/
    ├── ingest-text-request.json
    ├── ingest-text-response.json
    ├── resistance-response.json
    └── guidance-response.json
```

### 9.2 Root OpenAPI Spec

```yaml
# contracts/openapi.yaml
openapi: 3.1.0
info:
  title: UDARA AI API
  version: 2.1.0
  description: |
    AMR Surveillance Platform API for sub-Saharan Africa.
    Provides clinical data extraction, resistance analytics,
    and treatment guidance for Community Health Workers.
  contact:
    name: UDARA API Team
    email: api@udara.health
  license:
    name: AGPL-3.0
    url: https://www.gnu.org/licenses/agpl-3.0.en.html

servers:
  - url: https://api.udara.health
    description: Production
  - url: https://staging-api.udara.health
    description: Staging
  - url: http://localhost:8000
    description: Local development

security:
  - BearerAuth: []
  - ApiKeyAuth: []

tags:
  - name: Ingest
    description: Clinical data ingestion endpoints
  - name: Resistance
    description: AMR resistance analytics endpoints
  - name: Guidance
    description: Treatment guidance endpoint
  - name: Sync
    description: Edge-cloud synchronization endpoints
  - name: Webhooks
    description: External webhook receivers (USSD, Telegram, WhatsApp)
  - name: System
    description: Health, metrics, and diagnostics

paths:
  /v2/ingest/text:
    $ref: gateway/ingest.yaml#/paths/~1v2~1ingest~1text
  /v2/ingest/image:
    $ref: gateway/ingest.yaml#/paths/~1v2~1ingest~1image
  /v2/ingest/audio:
    $ref: gateway/ingest.yaml#/paths/~1v2~1ingest~1audio
  /v2/ingest/multimodal:
    $ref: gateway/ingest.yaml#/paths/~1v2~1ingest~1multimodal
  /v2/resistance/{drug}/{district}:
    $ref: gateway/resistance.yaml#/paths/~1v2~1resistance~1{drug}~1{district}
  /v2/resistance/alerts:
    $ref: gateway/resistance.yaml#/paths/~1v2~1resistance~1alerts
  /v2/guidance:
    $ref: gateway/guidance.yaml#/paths/~1v2~1guidance
  /v2/sync/push:
    $ref: sync/sync.yaml#/paths/~1v2~1sync~1push
  /v2/sync/status:
    $ref: sync/sync.yaml#/paths/~1v2~1sync~1status
  /v2/health:
    $ref: gateway/ingest.yaml#/paths/~1v2~1health

components:
  securitySchemes:
    BearerAuth:
      type: http
      scheme: bearer
      bearerFormat: JWT
      description: "JWT token issued by UDARA auth service"
    ApiKeyAuth:
      type: apiKey
      in: header
      name: X-API-Key
      description: "API key for device-to-cloud and webhook authentication"

  schemas:
    StructuredCase:
      $ref: schemas/case.yaml#/components/schemas/StructuredCase
    ExtractedSymptom:
      $ref: schemas/case.yaml#/components/schemas/ExtractedSymptom
    ExtractedDrug:
      $ref: schemas/case.yaml#/components/schemas/ExtractedDrug
    ErrorResponse:
      $ref: schemas/errors.yaml#/components/schemas/ErrorResponse
    ResistanceEstimate:
      $ref: schemas/resistance.yaml#/components/schemas/ResistanceEstimate
    GuidanceResponse:
      $ref: schemas/guidance.yaml#/components/schemas/GuidanceResponse
```

---

## 10. Versioning & Deprecation Policy

### 10.1 Version Strategy

```
URL Versioning Pattern:
  /v1/...  → Deprecated (sunsetting 2025-06-01)
  /v2/...  → Current (stable since 2024-09-01)
  /v3/...  → Beta (preview features, not production-ready)
```

### 10.2 Deprecation Timeline

| Phase | Duration | Description |
|-------|----------|-------------|
| **Active** | — | Current version, receives all updates |
| **Deprecated** | 6 months | Still works, returns `Deprecation` header, no new features |
| **Sunset Notice** | 3 months | Returns `410 Gone` with migration guide |
| **Removed** | — | Endpoint returns `404` |

### 10.3 Deprecation Headers

```http
# When accessing a deprecated endpoint:
HTTP/1.1 200 OK
Content-Type: application/json
Deprecation: true
Sunset: Sat, 01 Jun 2025 00:00:00 GMT
Link: </v2/ingest/text>; rel="successor-version"
X-Deprecation-Notice: "v1 endpoints will be removed on 2025-06-01. Migrate to /v2/."
```

### 10.4 Breaking Change Policy

A change is considered "breaking" if it:
- Removes or renames a field from the response
- Changes a field's data type
- Adds a new required field to the request
- Changes an HTTP status code for a known scenario
- Changes the URL structure

**All breaking changes require a new major version (v2 → v3).** Non-breaking
changes (new optional fields, new endpoints) are made within the current version
with a minor bump in the OpenAPI spec.
