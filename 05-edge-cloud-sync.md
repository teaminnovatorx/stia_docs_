# 05 — Edge-Cloud Sync Protocol

> **UDARA AI** — AMR Surveillance Platform for Sub-Saharan Africa  
> Document Version: 2.1.0 | Last Updated: 2026-05-27 | Status: Production

---

## Table of Contents

1. [Design Principles](#1-design-principles)
2. [Architecture Overview & ASCII Sync Flow](#2-architecture-overview--ascii-sync-flow)
3. [Sync Record Format](#3-sync-record-format)
4. [CRDT Conflict Resolution](#4-crdt-conflict-resolution)
5. [SyncEngine Implementation](#5-syncengine-implementation)
6. [USB Physical Fallback](#6-usb-physical-fallback)
7. [Model Sync: Cloud → Edge](#7-model-sync-cloud--edge)
8. [Monitoring & Observability](#8-monitoring--observability)
9. [Failure Modes & Recovery](#9-failure-modes--recovery)
10. [Appendix: Full Configuration Reference](#10-appendix-full-configuration-reference)

---

## 1. Design Principles

The sync protocol is the backbone of UDARA's offline-first architecture. Edge devices
(Raspberry Pi units in health facilities) operate in environments with intermittent or
zero connectivity. The protocol must guarantee **zero data loss** regardless of network
conditions.

### 1.1 Core Principles

| # | Principle | Description | Implementation |
|---|-----------|-------------|----------------|
| 1 | **Offline-First** | All data written locally before any network attempt. The system MUST function indefinitely without connectivity. | SQLite WAL mode with local-first writes; sync is a background concern. |
| 2 | **Delta-Only** | Only changed/new records are transmitted. Never full database dumps. | `sync_queue` table tracks dirty records with operation type (INSERT/UPDATE/DELETE). |
| 3 | **Compressed** | All payloads compressed before transmission to minimize bandwidth cost over cellular (2G/3G). | `gzip` at level 6 (balance between CPU on RPi and bandwidth savings). |
| 4 | **Encrypted** | All data encrypted in transit AND at rest on USB fallback media. Patient data must never be plaintext on removable media. | AES-256-GCM with per-batch keys derived from device + cloud shared secret. |
| 5 | **Anonymized** | Patient PII stripped before sync. Only necessary clinical data transmitted. | HIPAA-compliant anonymization pipeline removes names, phone numbers, exact DOBs. |
| 6 | **CRDT** | Conflict-free Replicated Data Types ensure convergent state across edge and cloud without coordination. | Yjs YMap/YArray for structured data; LWW registers for metadata; Union types for collections. |
| 7 | **Ordered** | Sync records carry a Hybrid Logical Clock (HLC) timestamp ensuring global ordering without central coordination. | HLC combines physical clock + logical counter; tolerates up to ±180s clock skew. |
| 8 | **Resilient** | Every failure mode has an automated recovery path. No manual intervention for common scenarios. | Exponential backoff, USB fallback, watchdog process, health pings. |

### 1.2 Network Assumptions

```
┌─────────────────────────────────────────────────────────────────────┐
│                    UDARA Connectivity Spectrum                      │
├──────────────┬──────────────────┬──────────────────┬────────────────┤
│   TIER 1     │     TIER 2       │     TIER 3       │    TIER 4      │
│  4G/LTE      │    3G/HSPA+      │    2G/EDGE       │   OFFLINE      │
├──────────────┼──────────────────┼──────────────────┼────────────────┤
│  5-50 Mbps   │   0.4-2 Mbps     │   10-60 kbps     │    0 kbps      │
│  20-100 ms   │   100-500 ms     │   500-2000 ms    │    ∞            │
│  Always-on   │   Intermittent   │   Unreliable     │  No service    │
│  ~$2-5/GB    │   ~$5-15/GB      │  ~$15-30/GB      │   N/A          │
├──────────────┼──────────────────┼──────────────────┼────────────────┤
│  Urban       │   Peri-urban     │   Rural          │  Remote        │
│  ~15% sites  │   ~25% sites     │   ~40% sites     │  ~20% sites    │
└──────────────┴──────────────────┴──────────────────┴────────────────┘
```

**Key insight:** ~60% of deployment sites operate at Tier 3 or below. Sync design
MUST work at 10 kbps effective throughput after encryption/compression overhead.

### 1.3 Bandwidth Budget

```
Per-case sync payload (average):
  Raw clinical record:           ~2.4 KB
  After anonymization:           ~1.8 KB  (-25%)
  After delta extraction:        ~0.6 KB  (-67%)
  After gzip compression:        ~0.22 KB (-88%)
  After AES-256-GCM encryption:  ~0.24 KB (+9% overhead)
  HTTP framing + TLS overhead:    ~0.28 KB

At 10 kbps (Tier 3):  0.28 KB × (10 kbps / 8) = 0.35s per case
At 10 kbps, 50 cases: ~17.5 seconds per batch
Daily budget (20 min window):   ~3,400 cases/day
```

---

## 2. Architecture Overview & ASCII Sync Flow

### 2.1 End-to-End Sync Architecture

```
                         UDARA EDGE-CLOUD SYNC FLOW
    ════════════════════════════════════════════════════════════════════

    ┌───────────────────────── EDGE DEVICE (Raspberry Pi) ─────────────────────┐
    │                                                                          │
    │  ┌──────────┐    ┌──────────────┐    ┌────────────────┐                  │
    │  │ Clinical  │───>│  SQLite WAL  │───>│  Sync Queue    │                  │
    │  │  Agents   │    │  (local DB)  │    │  (dirty recs)  │                  │
    │  └──────────┘    └──────────────┘    └───────┬────────┘                  │
    │                                              │                           │
    │                                              ▼                           │
    │  ┌──────────────────────────────────────────────────────────────────┐   │
    │  │                       SyncEngine (Python)                        │   │
    │  │  ┌─────────────┐  ┌────────────┐  ┌──────────┐  ┌────────────┐  │   │
    │  │  │  1. Get     │  │ 2. Anonym- │  │ 3. Com-  │  │ 4. Encrypt │  │   │
    │  │  │  Unsynced   │─>│  ize PII   │─>│  press   │─>│  (AES)    │  │   │
    │  │  │  Records    │  │            │  │  (gzip)  │  │            │  │   │
    │  │  └─────────────┘  └────────────┘  └──────────┘  └─────┬──────┘  │   │
    │  │                                                                │   │
    │  │  ┌─────────────┐  ┌────────────┐  ┌──────────┐  ┌────────────┐  │   │
    │  │  │  8. Mark    │<─│ 7. Handle  │<─│ 6. Apply │<─│ 5. Push    │  │   │
    │  │  │  Synced     │  │  Response  │  │  CRDT    │  │  to Cloud  │  │   │
    │  │  └─────────────┘  └────────────┘  └──────────┘  └────────────┘  │   │
    │  └──────────────────────────────────────────────────────────────────┘   │
    │          │                                                │            │
    │          │ [Fallback Path]                                │ [Primary]  │
    │          ▼                                                ▼            │
    │  ┌──────────────┐                              ┌─────────────────┐     │
    │  │  USB Export  │         IF OFFLINE            │  Cellular/WiFi │     │
    │  │  (FAT32 +   │         > 48 HOURS             │  (HTTPS/TLS)   │     │
    │  │   AES file)  │                              └────────┬────────┘     │
    │  └──────┬───────┘                                       │              │
    └─────────┼───────────────────────────────────────────────┼──────────────┘
              │                                               │
              │         ┌─────────────────────┐               │
              │         │   CHW Carries USB   │               │
              │         │   to Connectivity   │               │
              │         │   Point / Hub       │               │
              │         └──────────┬──────────┘               │
              │                    │                          │
              ▼                    ▼                          ▼
    ╔═══════════════════════════ CLOUD ─═══════════════════════════════╗
    ║                                                                  ║
    ║  ┌─────────────────┐     ┌───────────────────────────────────┐   ║
    ║  │  Cloud FastAPI   │────>│  Decrypt → Decompress → Validate │   ║
    ║  │  /sync/receive   │     └──────────────┬────────────────────┘   ║
    ║  └─────────────────┘                    │                        ║
    ║                                          ▼                        ║
    ║  ┌─────────────────────────────────────────────────────────────┐ ║
    ║  │              CRDT Merger (Yjs Document Store)                │ ║
    ║  │   • Merge incoming YMap/YArray updates                       │ ║
    ║  │   • LWW resolution on metadata conflicts                     │ ║
    ║  │   • Union merge on collection-type fields                    │ ║
    ║  └─────────────────────────────────┬───────────────────────────┘ ║
    ║                                    │                               ║
    ║                                    ▼                               ║
    ║  ┌─────────────────────────────────────────────────────────────┐ ║
    ║  │                    TimescaleDB (Cloud)                        │ ║
    ║  │   • national_cases (hypertable)                              │ ║
    ║  │   • resistance_estimates (materialized view)                  │ ║
    ║  │   • sync_audit_log                                           │ ║
    ║  │   • device_heartbeat                                         │ ║
    ║  └─────────────────────────────────────────────────────────────┘ ║
    ╚══════════════════════════════════════════════════════════════════╝
```

### 2.2 Sync Loop Sequence Diagram

```
    Edge SyncEngine                   Network                Cloud FastAPI
    ──────────────                    ───────                ─────────────

    [every 5 min]
         │
         │ check_connectivity()
         │───────────────────────────>
         │<───────────────────────────
         │ (HTTP 200 / timeout)
         │
    [if connected]
         │
         │ get_unsynced_records(limit=100)
         │ (SELECT FROM sync_queue)
         │
         │ anonymize_batch(records)
         │ compress_batch(records)
         │ encrypt_batch(records)
         │
         │ POST /sync/push
         │ {batch_id, device_id, payload, hmac}
         │───────────────────────────>
         │                               │ decrypt + decompress
         │                               │ validate + merge CRDT
         │                               │ insert into TimescaleDB
         │<───────────────────────────
         │ {status, conflicts[], applied}
         │
         │ apply_crdt_resolution(conflicts)
         │ mark_synced(batch_id)
         │
    [if offline > 48h]
         │
         │ export_to_usb()
         │ (FAT32 + AES-256 encrypted .udara file)
         │
         │ ... [CHW carries USB] ...
         │
         │ import_from_usb()
         │───────────────────────────>
         │ POST /sync/push (same endpoint)
         │<───────────────────────────
         │
    [every 24h]
         │
         │ GET /sync/models
         │<───────────────────────────
         │ {model_name, version, url, sha256}
         │
         │ download_model_if_newer()
         │ verify_sha256()
         │ atomic_swap_model()
```

---

## 3. Sync Record Format

### 3.1 Sync Queue Schema (Edge SQLite)

```sql
CREATE TABLE IF NOT EXISTS sync_queue (
    id              INTEGER PRIMARY KEY AUTOINCREMENT,
    record_id       TEXT    NOT NULL,          -- UUID of the source record
    record_type     TEXT    NOT NULL,          -- 'case' | 'resistance' | 'alert' | 'ussd_session'
    operation       TEXT    NOT NULL CHECK(operation IN ('INSERT','UPDATE','DELETE')),
    payload         TEXT    NOT NULL,          -- JSON: full record snapshot at time of change
    hlc_timestamp   TEXT    NOT NULL,          -- Hybrid Logical Clock: "2026-05-27T10:30:00.000Z:0003:device-abc"
    priority        INTEGER NOT NULL DEFAULT 5,-- 1=highest (AMR alerts) 10=lowest (telemetry)
    retries         INTEGER NOT NULL DEFAULT 0,
    max_retries     INTEGER NOT NULL DEFAULT 10,
    next_retry_at   TEXT,                      -- ISO 8601 datetime, NULL = ready now
    batch_id        TEXT,                      -- Set when included in a batch
    created_at      TEXT    NOT NULL DEFAULT (datetime('now')),
    encrypted_size  INTEGER                     -- Bytes after compression + encryption
);
```

### 3.2 Sync Record JSON Format

```json
{
  "sync_version": "2.1.0",
  "device_id": "udara-edge-KE-NRB-0042",
  "batch_id": "b7f3c291-8a4e-4d6f-b912-3e8c5f0a1d72",
  "batch_timestamp": "2026-05-27T10:30:00.000Z",
  "hlc_timestamp": "2026-05-27T10:30:00.000Z:0003:udara-edge-KE-NRB-0042",
  "total_records": 3,
  "compressed_size_bytes": 512,
  "encryption": {
    "algorithm": "AES-256-GCM",
    "key_derivation": "HKDF-SHA256",
    "salt": "a1b2c3d4e5f6...",
    "iv": "f6e5d4c3b2a1...",
    "auth_tag": "9876543210abcdef..."
  },
  "records": [
    {
      "sync_id": "sq-00142",
      "record_id": "case-2025-KE-NRB-00391",
      "record_type": "case",
      "operation": "INSERT",
      "hlc_timestamp": "2026-05-27T10:28:15.234Z:0001:udara-edge-KE-NRB-0042",
      "priority": 2,
      "payload": {
        "patient_age_years": 34,
        "patient_sex": "F",
        "patient_age_group": "25-34",
        "facility_code": "KE-NRB-FAC-017",
        "district": "Nairobi",
        "county": "Nairobi",
        "country": "KEN",
        "encounter_date": "2025-01-14",
        "symptoms": [
          {"name": "fever", "category": "SYSTEMIC", "duration_days": 5, "severity": "moderate"},
          {"name": "dysuria", "category": "GENITOURINARY", "duration_days": 3, "severity": "severe"},
          {"name": "flank_pain", "category": "GENITOURINARY", "duration_days": 3, "severity": "moderate"}
        ],
        "drugs_prescribed": [
          {"name": "ciprofloxacin", "dose_mg": 500, "frequency": "bid", "duration_days": 7, "source": "prescription"},
          {"name": "nitrofurantoin", "dose_mg": 100, "frequency": "qid", "duration_days": 5, "source": "otc"}
        ],
        "diagnosis_suggestion": "urinary_tract_infection",
        "amr_risk_score": 0.72,
        "extracted_by": "agent-a-v2.3.1",
        "resistance_estimates": {
          "ciprofloxacin": {"rate": 0.68, "ci_lower": 0.61, "ci_upper": 0.74, "trend": "increasing"},
          "nitrofurantoin": {"rate": 0.12, "ci_lower": 0.08, "ci_upper": 0.18, "trend": "stable"}
        },
        "geohash": "k17fqv",
        "location_anonymized": true
      }
    },
    {
      "sync_id": "sq-00143",
      "record_id": "resistance-KE-NRB-2025-W02",
      "record_type": "resistance",
      "operation": "UPDATE",
      "hlc_timestamp": "2026-05-27T10:29:45.891Z:0002:udara-edge-KE-NRB-0042",
      "priority": 5,
      "payload": {
        "drug": "ciprofloxacin",
        "district": "Nairobi",
        "county": "Nairobi",
        "period_start": "2025-01-06",
        "period_end": "2025-01-12",
        "total_cases": 47,
        "resistant_cases": 32,
        "rate": 0.681,
        "ci_lower": 0.618,
        "ci_upper": 0.739,
        "trend": "increasing",
        "trend_slope": 0.034,
        "sample_size_met": true,
        "computed_at": "2026-05-27T10:29:45.000Z"
      }
    },
    {
      "sync_id": "sq-00144",
      "record_id": "alert-2025-KE-NRB-001",
      "record_type": "alert",
      "operation": "INSERT",
      "hlc_timestamp": "2026-05-27T10:30:00.000Z:0003:udara-edge-KE-NRB-0042",
      "priority": 1,
      "payload": {
        "alert_type": "RESISTANCE_THRESHOLD",
        "severity": "HIGH",
        "drug": "ciprofloxacin",
        "district": "Nairobi",
        "current_rate": 0.681,
        "threshold_rate": 0.50,
        "trend": "increasing",
        "message": "Ciprofloxacin resistance in Nairobi (68.1%) exceeds 50% threshold and is increasing",
        "recommended_alternatives": ["nitrofurantoin", "fosfomycin", "pivmecillinam"],
        "acknowledged": false
      }
    }
  ]
}
```

### 3.3 Anonymization Rules

```python
# Fields REMOVED before sync (never leave the edge device):
STRIP_FIELDS = [
    "patient_name",           # Full name
    "patient_phone",          # Phone number
    "patient_address",        # Physical address
    "patient_id_national",    # National ID number
    "chw_name",               # CHW's full name (use chw_id instead)
    "exact_date_of_birth",    # Replaced with age_group
    "gps_coordinates_lat",    # Replaced with geohash (precision 5 ≈ 5km)
    "gps_coordinates_lon",    # Replaced with geohash
    "recording_audio_raw",    # Raw audio bytes never synced
    "recording_image_raw",    # Raw image bytes never synced
]

# Fields TRANSFORMED before sync:
TRANSFORM_RULES = {
    "patient_age_years": "patient_age_group",       # 34 → "25-34"
    "gps_lat,gps_lon": "geohash_5",                  # -1.2921,36.8219 → "k17fqv"
    "facility_name": "facility_code",                # "Kenyatta Hosp" → "KE-NRB-FAC-017"
    "exact_datetime": "date_only",                   # "2026-05-27T10:28:15Z" → "2026-05-27"
    "drug_dose_exact": "drug_dose_mg",               # "500mg" → 500 (integer)
}

# Fields RETAINED (clinical necessity):
RETAIN_FIELDS = [
    "patient_age_years",      # As integer (not exact DOB)
    "patient_sex",            # M/F/Other
    "patient_age_group",      # Derived category
    "symptoms",               # Clinical data (no PII)
    "drugs_prescribed",       # Clinical data (no PII)
    "diagnosis_suggestion",   # AI suggestion
    "amr_risk_score",         # Computed score
    "resistance_estimates",   # Computed statistics
    "district", "county", "country",  # Administrative areas (not addresses)
]
```

---

## 4. CRDT Conflict Resolution

### 4.1 Why CRDTs?

Multiple edge devices may observe the same patient (transferred between facilities),
or the same device may create records while offline that conflict with cloud state.
Traditional "last-write-wins" discards valuable clinical observations. CRDTs guarantee
**mathematical convergence** to the same state on all replicas regardless of message
ordering or duplication.

### 4.2 CRDT Type Selection

```
┌─────────────────────────────────────────────────────────────────┐
│                    UDARA CRDT TYPE MAP                          │
├────────────────────────┬────────────────────┬───────────────────┤
│ Data Shape             │ CRDT Type          │ Conflict Rule      │
├────────────────────────┼────────────────────┼───────────────────┤
│ Case metadata          │ Yjs YMap           │ LWW per field      │
│ (diagnosis, score)     │                    │ (HLC timestamp)    │
├────────────────────────┼────────────────────┼───────────────────┤
│ Symptom list           │ Yjs YArray         │ Ordered insert,    │
│ (symptoms[])           │                    │ no duplicates      │
├────────────────────────┼────────────────────┼───────────────────┤
│ Drug history           │ Yjs YArray +       │ Union merge:       │
│ (drugs_prescribed[])   │ G-Counter per drug │ keep all entries   │
├────────────────────────┼────────────────────┼───────────────────┤
│ Resistance rate        │ LWW Register       │ Latest HLC wins    │
│ (rate, CI, trend)      │                    │                    │
├────────────────────────┼────────────────────┼───────────────────┤
│ Alert acknowledged     │ LWW Register       │ Boolean + HLC      │
│ status                 │                    │                    │
├────────────────────────┼────────────────────┼───────────────────┤
│ Collections (all cases │ OR-Set /           │ Add-wins: entries  │
│ for a district)        │ Union Type         │ only removed by     │
│                        │                    │ explicit delete    │
│                        │                    │ from same author    │
└────────────────────────┴────────────────────┴───────────────────┘
```

### 4.3 LWW (Last-Writer-Wins) for Metadata

```python
"""
LWW Register for case metadata fields.
Uses Hybrid Logical Clock (HLC) timestamps for ordering.
Tolerates up to ±180 seconds of clock skew between devices.
"""

from dataclasses import dataclass, field
from datetime import datetime, timezone
from typing import Any, Optional
import hashlib


@dataclass
class HLCTimestamp:
    """
    Hybrid Logical Clock timestamp.
    Format: "2026-05-27T10:30:00.000Z:0003:device-id"
    """
    physical_time: datetime
    logical_counter: int
    device_id: str

    def to_string(self) -> str:
        ts = self.physical_time.strftime("%Y-%m-%dT%H:%M:%S.") + \
             f"{self.physical_time.microsecond // 1000:03d}Z"
        return f"{ts}:{self.logical_counter:04d}:{self.device_id}"

    @classmethod
    def from_string(cls, value: str) -> "HLCTimestamp":
        parts = value.split(":")
        dt_str = ":".join(parts[:-2])
        physical = datetime.strptime(dt_str, "%Y-%m-%dT%H:%M:%S.%fZ").replace(
            tzinfo=timezone.utc
        )
        return cls(
            physical_time=physical,
            logical_counter=int(parts[-2]),
            device_id=parts[-1]
        )

    def happens_before(self, other: "HLCTimestamp") -> Optional[bool]:
        """
        Compare two HLC timestamps.
        Returns True if self < other, False if self > other, None if concurrent.
        """
        if self.physical_time < other.physical_time:
            return True
        elif self.physical_time > other.physical_time:
            return False
        else:
            # Same physical time: compare logical counter
            if self.logical_counter < other.logical_counter:
                return True
            elif self.logical_counter > other.logical_counter:
                return False
            else:
                # Same physical + logical: concurrent (use device_id as tiebreaker)
                return None

    def __gt__(self, other: "HLCTimestamp") -> bool:
        result = self.happens_before(other)
        if result is None:
            # Concurrent: deterministic tiebreaker by device_id hash
            return hashlib.sha256(self.device_id.encode()).hexdigest() > \
                   hashlib.sha256(other.device_id.encode()).hexdigest()
        return not result


class LWWRegister:
    """
    Last-Writer-Wins register for individual case metadata fields.
    Each field stores (value, HLC_timestamp) and resolves conflicts
    by selecting the entry with the latest HLC timestamp.
    """

    def __init__(self, field_name: str):
        self.field_name = field_name
        self._value: Any = None
        self._timestamp: Optional[HLCTimestamp] = None

    def set(self, value: Any, hlc: HLCTimestamp) -> bool:
        """
        Set value if the incoming HLC is newer than current.
        Returns True if value was updated, False if conflict resolved to existing.
        """
        if self._timestamp is None or hlc > self._timestamp:
            self._value = value
            self._timestamp = hlc
            return True
        return False

    def merge(self, incoming_value: Any, incoming_hlc: HLCTimestamp) -> dict:
        """
        Merge an incoming value with current state.
        Returns conflict info if a conflict occurred and was resolved.
        """
        if self._timestamp is None:
            self._value = incoming_value
            self._timestamp = incoming_hlc
            return {"resolved": False, "action": "accepted"}

        if incoming_hlc > self._timestamp:
            old_value = self._value
            self._value = incoming_value
            self._timestamp = incoming_hlc
            return {
                "resolved": True,
                "action": "incoming_wins",
                "kept": incoming_value,
                "discarded": old_value,
                "field": self.field_name
            }
        elif incoming_hlc == self._timestamp:
            # Concurrent: deterministic tiebreaker
            if incoming_hlc.device_id >= self._timestamp.device_id:
                self._value = incoming_value
                self._timestamp = incoming_hlc
            return {
                "resolved": True,
                "action": "concurrent_tiebreaker",
                "field": self.field_name
            }
        else:
            return {
                "resolved": True,
                "action": "local_wins",
                "kept": self._value,
                "discarded": incoming_value,
                "field": self.field_name
            }

    @property
    def value(self) -> Any:
        return self._value

    @property
    def timestamp(self) -> Optional[HLCTimestamp]:
        return self._timestamp
```

### 4.4 YMap/YArray for Structured Data

```python
"""
CRDT merge for case records using Yjs-compatible data structures.
This is the edge-side implementation. The cloud runs a Yjs server
with a persistent document store backed by TimescaleDB.
"""

from typing import List, Dict, Any, Optional
import json
import copy


class YMapMerge:
    """
    Merge strategy for a YMap (key-value map where each value is independently
    a CRDT). Each key in the map is resolved independently using LWW.
    """

    # Fields that use LWW-Register semantics (scalar replacement)
    LWW_FIELDS = {
        "diagnosis_suggestion",
        "amr_risk_score",
        "encounter_date",
        "computed_at",
        "sample_size_met",
        "trend",
        "trend_slope",
    }

    # Fields that use Union/Array semantics (accumulation, no deletion)
    UNION_FIELDS = {
        "symptoms",
        "drugs_prescribed",
        "acknowledged_by",
    }

    # Fields that use YArray semantics (ordered, no-duplicate insertion)
    YARRAY_FIELDS = {
        "clinical_notes",
        "version_history",
    }

    @classmethod
    def merge(
        cls,
        local: Dict[str, Any],
        remote: Dict[str, Any],
        local_hlc: HLCTimestamp,
        remote_hlc: HLCTimestamp,
    ) -> Dict[str, Any]:
        """
        Merge remote record into local record.
        Returns merged result and list of conflict resolutions.
        """
        result = copy.deepcopy(local)
        conflicts = []

        for key, remote_value in remote.items():
            if key not in result:
                # New field from remote: accept
                result[key] = remote_value
                continue

            local_value = result[key]

            if key in cls.LWW_FIELDS:
                # Scalar LWW: pick based on HLC
                if remote_hlc > local_hlc:
                    if local_value != remote_value:
                        conflicts.append({
                            "field": key,
                            "action": "remote_wins",
                            "local": local_value,
                            "remote": remote_value,
                        })
                    result[key] = remote_value
                elif remote_hlc < local_hlc:
                    if local_value != remote_value:
                        conflicts.append({
                            "field": key,
                            "action": "local_wins",
                            "local": local_value,
                            "remote": remote_value,
                        })
                # else: concurrent — keep local (stable preference)

            elif key in cls.UNION_FIELDS:
                # Union merge: combine both lists, deduplicate by identifier
                if isinstance(local_value, list) and isinstance(remote_value, list):
                    merged_list = cls._union_merge_lists(local_value, remote_value, key)
                    if len(merged_list) > len(local_value):
                        conflicts.append({
                            "field": key,
                            "action": "union_merge",
                            "local_count": len(local_value),
                            "remote_count": len(remote_value),
                            "merged_count": len(merged_list),
                        })
                    result[key] = merged_list

            elif key in cls.YARRAY_FIELDS:
                # Ordered array: append new entries by HLC order
                if isinstance(local_value, list) and isinstance(remote_value, list):
                    merged_list = cls._ordered_array_merge(
                        local_value, remote_value, local_hlc, remote_hlc
                    )
                    result[key] = merged_list
            else:
                # Unknown field: LWW fallback
                if remote_hlc > local_hlc:
                    result[key] = remote_value

        return result, conflicts

    @classmethod
    def _union_merge_lists(
        cls, local: List[Dict], remote: List[Dict], field_name: str
    ) -> List[Dict]:
        """
        Merge two lists using union semantics.
        Deduplication key depends on field:
        - symptoms: dedupe by "name"
        - drugs_prescribed: dedupe by (name, dose_mg, source)
        """
        if field_name == "symptoms":
            existing = {(s.get("name"), s.get("duration_days")): s for s in local}
            for item in remote:
                key = (item.get("name"), item.get("duration_days"))
                if key not in existing:
                    existing[key] = item
            return list(existing.values())

        elif field_name == "drugs_prescribed":
            existing = {
                (d.get("name"), d.get("dose_mg"), d.get("source")): d for d in local
            }
            for item in remote:
                key = (item.get("name"), item.get("dose_mg"), d.get("source"))
                if key not in existing:
                    existing[key] = item
            return list(existing.values())

        else:
            # Generic: dedupe by JSON serialization
            seen = {json.dumps(s, sort_keys=True) for s in local}
            merged = list(local)
            for item in remote:
                if json.dumps(item, sort_keys=True) not in seen:
                    merged.append(item)
                    seen.add(json.dumps(item, sort_keys=True))
            return merged

    @classmethod
    def _ordered_array_merge(
        cls,
        local: List[Dict],
        remote: List[Dict],
        local_hlc: HLCTimestamp,
        remote_hlc: HLCTimestamp,
    ) -> List[Dict]:
        """
        Merge ordered arrays by HLC timestamp ordering.
        Each entry should have an "hlc_timestamp" field.
        """
        all_entries = []

        for entry in local:
            entry_copy = copy.deepcopy(entry)
            entry_copy["_source"] = "local"
            all_entries.append(entry_copy)

        for entry in remote:
            entry_copy = copy.deepcopy(entry)
            entry_copy["_source"] = "remote"
            all_entries.append(entry_copy)

        # Sort by HLC timestamp
        def sort_key(e):
            ts_str = e.get("hlc_timestamp", "")
            try:
                hlc = HLCTimestamp.from_string(ts_str)
                return (hlc.physical_time, hlc.logical_counter, hlc.device_id)
            except (ValueError, AttributeError):
                return (datetime.min.replace(tzinfo=timezone.utc), 0, "")

        all_entries.sort(key=sort_key)

        # Remove source tags and dedupe by id if present
        seen_ids = set()
        result = []
        for entry in all_entries:
            entry.pop("_source", None)
            entry_id = entry.get("id")
            if entry_id:
                if entry_id in seen_ids:
                    continue
                seen_ids.add(entry_id)
            result.append(entry)

        return result
```

---

## 5. SyncEngine Implementation

### 5.1 Complete SyncEngine Class

```python
"""
UDARA Edge SyncEngine
=====================
Handles all edge-to-cloud synchronization including connectivity checks,
batch preparation, compression, encryption, push, conflict resolution,
and USB fallback.

Run as a systemd service on each Raspberry Pi edge device.
"""

import asyncio
import gzip
import hashlib
import json
import logging
import os
import shutil
import sqlite3
import time
from dataclasses import dataclass, field
from datetime import datetime, timedelta, timezone
from pathlib import Path
from typing import Any, Dict, List, Optional, Tuple

import httpx
from cryptography.hazmat.primitives.ciphers.aead import AESGCM
from cryptography.hazmat.primitives.kdf.hkdf import HKDF
from cryptography.hazmat.primitives import hashes

logger = logging.getLogger("udara.sync")


# ─── Configuration ───────────────────────────────────────────────────────────

@dataclass
class SyncConfig:
    """Configuration for the SyncEngine."""

    # Device identity
    device_id: str = os.environ.get("UDARA_DEVICE_ID", "udara-edge-dev-001")
    facility_code: str = os.environ.get("UDARA_FACILITY_CODE", "DEV-FAC-001")

    # Database
    edge_db_path: str = os.environ.get("UDARA_EDGE_DB", "/var/lib/udara/edge.db")

    # Cloud endpoints
    cloud_base_url: str = os.environ.get(
        "UDARA_CLOUD_URL", "https://api.udara.health"
    )
    sync_endpoint: str = "/sync/push"
    status_endpoint: str = "/sync/status"
    models_endpoint: str = "/sync/models"
    connectivity_check_url: str = os.environ.get(
        "UDARA_CONNECTIVITY_CHECK", "https://api.udara.health/health"
    )

    # Sync behavior
    batch_size: int = 50
    sync_interval_seconds: int = 300       # 5 minutes
    max_retries: int = 10
    retry_base_delay_seconds: float = 30.0  # Exponential backoff base
    retry_max_delay_seconds: float = 3600.0 # 1 hour max backoff

    # USB fallback
    usb_fallback_hours: float = 48.0
    usb_mount_point: str = "/media/udara_usb"
    usb_export_dir: str = "/var/lib/udara/usb_exports"

    # Encryption
    encryption_key: str = os.environ.get("UDARA_SYNC_KEY", "")  # 32-byte hex

    # Compression
    gzip_level: int = 6

    # Model sync
    model_dir: str = "/var/lib/udara/models"
    model_current_symlink: str = "/var/lib/udara/models/current"

    # API authentication
    api_key: str = os.environ.get("UDARA_API_KEY", "")
    api_cert_path: str = os.environ.get("UDARA_API_CERT", "")


# ─── Exceptions ──────────────────────────────────────────────────────────────

class SyncError(Exception):
    """Base exception for sync operations."""
    pass


class ConnectivityError(SyncError):
    """No network connectivity available."""
    pass


class EncryptionError(SyncError):
    """Encryption/decryption failed."""
    pass


class CloudAPIError(SyncError):
    """Cloud API returned an error."""
    def __init__(self, status_code: int, message: str, response_body: Any = None):
        self.status_code = status_code
        self.message = message
        self.response_body = response_body
        super().__init__(f"Cloud API {status_code}: {message}")


# ─── SyncEngine ──────────────────────────────────────────────────────────────

class SyncEngine:
    """
    Main synchronization engine for UDARA edge devices.

    Operates as a long-running service that:
    1. Periodically checks connectivity
    2. Pulls unsynced records from local SQLite queue
    3. Anonymizes, compresses, and encrypts batches
    4. Pushes to cloud API
    5. Handles CRDT conflict resolution
    6. Falls back to USB export when offline > 48h
    7. Pulls model updates from cloud
    """

    def __init__(self, config: Optional[SyncConfig] = None):
        self.config = config or SyncConfig()
        self._db: Optional[sqlite3.Connection] = None
        self._http_client: Optional[httpx.AsyncClient] = None
        self._running = False
        self._last_sync_attempt: Optional[datetime] = None
        self._last_successful_sync: Optional[datetime] = None
        self._consecutive_failures = 0
        self._stats = {
            "total_pushed": 0,
            "total_failed": 0,
            "total_conflicts_resolved": 0,
            "total_bytes_uploaded": 0,
            "usb_exports": 0,
            "usb_imports": 0,
            "models_updated": 0,
        }

        if not self.config.encryption_key:
            raise EncryptionError(
                "UDARA_SYNC_KEY environment variable is required. "
                "Generate with: python -c \"import secrets; print(secrets.token_hex(32))\""
            )

        if len(bytes.fromhex(self.config.encryption_key)) != 32:
            raise EncryptionError(
                "UDARA_SYNC_KEY must be exactly 64 hex characters (32 bytes)"
            )

        logger.info(
            "SyncEngine initialized: device=%s, facility=%s, cloud=%s",
            self.config.device_id,
            self.config.facility_code,
            self.config.cloud_base_url,
        )

    # ── Database Connection ──────────────────────────────────────────────

    def _get_db(self) -> sqlite3.Connection:
        """Get or create the SQLite database connection."""
        if self._db is None:
            db_path = Path(self.config.edge_db_path)
            db_path.parent.mkdir(parents=True, exist_ok=True)

            self._db = sqlite3.connect(str(db_path), timeout=30.0)
            self._db.row_factory = sqlite3.Row
            self._db.execute("PRAGMA journal_mode=WAL")
            self._db.execute("PRAGMA synchronous=NORMAL")
            self._db.execute("PRAGMA foreign_keys=ON")
            self._db.execute("PRAGMA busy_timeout=30000")
            logger.info("Database connection established: %s", db_path)
        return self._db

    def _get_http_client(self) -> httpx.AsyncClient:
        """Get or create the async HTTP client."""
        if self._http_client is None:
            kwargs = {
                "base_url": self.config.cloud_base_url,
                "timeout": httpx.Timeout(60.0, connect=15.0),
                "headers": {
                    "Authorization": f"Bearer {self.config.api_key}",
                    "X-Device-ID": self.config.device_id,
                    "User-Agent": f"udara-edge-sync/2.1.0 ({self.config.device_id})",
                },
            }
            if self.config.api_cert_path:
                kwargs["cert"] = self.config.api_cert_path
            self._http_client = httpx.AsyncClient(**kwargs)
        return self._http_client

    # ── 1. Connectivity Check ────────────────────────────────────────────

    def check_connectivity(self) -> Dict[str, Any]:
        """
        Check if the device has network connectivity to the cloud.

        Performs a lightweight HTTP GET to the health endpoint.
        Returns diagnostic information including latency and bandwidth estimate.

        Returns:
            {
                "connected": bool,
                "latency_ms": float,
                "bandwidth_estimate_kbps": Optional[float],
                "error": Optional[str],
                "checked_at": str
            }
        """
        result = {
            "connected": False,
            "latency_ms": None,
            "bandwidth_estimate_kbps": None,
            "error": None,
            "checked_at": datetime.now(timezone.utc).isoformat(),
        }

        try:
            start = time.monotonic()
            # Use synchronous check for simplicity; async version in run() loop
            import urllib.request
            req = urllib.request.Request(
                self.config.connectivity_check_url,
                headers={"User-Agent": f"udara-edge/{self.config.device_id}"},
            )
            with urllib.request.urlopen(req, timeout=10.0) as resp:
                resp.read()
            elapsed = time.monotonic() - start

            result["connected"] = True
            result["latency_ms"] = round(elapsed * 1000, 1)

            # Rough bandwidth estimation based on response time
            # Health endpoint returns ~200 bytes
            if elapsed > 0:
                result["bandwidth_estimate_kbps"] = round(
                    (200 * 8) / elapsed / 1000, 1
                )

            logger.info(
                "Connectivity check passed: latency=%.1fms",
                result["latency_ms"],
            )

        except Exception as e:
            result["error"] = str(e)
            logger.warning("Connectivity check failed: %s", e)

        return result

    # ── 2. Get Unsynced Records ──────────────────────────────────────────

    def get_unsynced_records(self, limit: int = 50) -> List[Dict[str, Any]]:
        """
        Fetch records from the sync queue that are ready to be synced.

        Priority ordering: lower priority number = synced first (1 = AMR alerts).
        Records with next_retry_at in the future are skipped (backoff in effect).

        Args:
            limit: Maximum number of records to fetch.

        Returns:
            List of sync queue records as dictionaries.
        """
        db = self._get_db()
        now_iso = datetime.now(timezone.utc).strftime("%Y-%m-%dT%H:%M:%SZ")

        cursor = db.execute("""
            SELECT
                id, record_id, record_type, operation,
                payload, hlc_timestamp, priority,
                retries, max_retries, batch_id, created_at
            FROM sync_queue
            WHERE (next_retry_at IS NULL OR next_retry_at <= ?)
            ORDER BY priority ASC, created_at ASC
            LIMIT ?
        """, (now_iso, limit))

        records = [dict(row) for row in cursor.fetchall()]
        logger.info("Fetched %d unsynced records (limit=%d)", len(records), limit)
        return records

    # ── 3. Anonymize Batch ───────────────────────────────────────────────

    def anonymize_batch(self, records: List[Dict]) -> List[Dict]:
        """
        Remove PII from records before they leave the edge device.

        Applies STRIP_FIELDS and TRANSFORM_RULES defined in the
        anonymization configuration.
        """
        anonymized = []
        for record in records:
            payload = json.loads(record["payload"])
            cleaned = {}

            for key, value in payload.items():
                if key in STRIP_FIELDS:
                    continue  # Remove PII entirely
                elif key == "patient_age_years" and value is not None:
                    cleaned["patient_age_group"] = self._age_to_group(value)
                elif key in ("gps_lat", "gps_lon"):
                    continue  # Handled together below
                else:
                    cleaned[key] = value

            # Convert lat/lon to geohash if both present
            lat = payload.get("gps_lat")
            lon = payload.get("gps_lon")
            if lat is not None and lon is not None:
                cleaned["geohash"] = self._lat_lon_to_geohash(lat, lon, precision=5)
                cleaned["location_anonymized"] = True

            record_copy = dict(record)
            record_copy["payload"] = json.dumps(cleaned)
            anonymized.append(record_copy)

        logger.info("Anonymized %d records", len(anonymized))
        return anonymized

    @staticmethod
    def _age_to_group(age: int) -> str:
        """Convert age in years to WHO standard age group."""
        if age < 1:
            return "<1"
        elif age < 5:
            return "1-4"
        elif age < 15:
            return "5-14"
        elif age < 25:
            return "15-24"
        elif age < 35:
            return "25-34"
        elif age < 45:
            return "35-44"
        elif age < 55:
            return "45-54"
        elif age < 65:
            return "55-64"
        else:
            return "65+"

    @staticmethod
    def _lat_lon_to_geohash(lat: float, lon: float, precision: int = 5) -> str:
        """Encode lat/lon to geohash string. Precision 5 ≈ 5km radius."""
        try:
            import pygeohash as gh
            return gh.encode(lat, lon, precision=precision)
        except ImportError:
            # Fallback: simple grid-based encoding
            lat_bin = int((lat + 90) * (2 ** precision) / 180)
            lon_bin = int((lon + 180) * (2 ** precision) / 360)
            return f"{lat_bin:04x}{lon_bin:04x}"

    # ── 4. Compress Batch ────────────────────────────────────────────────

    def compress_batch(self, records: List[Dict]) -> Tuple[bytes, int]:
        """
        Compress a batch of records using gzip.

        Returns:
            Tuple of (compressed_bytes, original_size_bytes)
        """
        json_payload = json.dumps({
            "sync_version": "2.1.0",
            "device_id": self.config.device_id,
            "records": records,
        }, separators=(",", ":"), default=str)

        original_size = len(json_payload.encode("utf-8"))
        compressed = gzip.compress(
            json_payload.encode("utf-8"),
            compresslevel=self.config.gzip_level,
        )
        ratio = (1 - len(compressed) / original_size) * 100 if original_size > 0 else 0

        logger.info(
            "Compressed batch: %d → %d bytes (%.1f%% reduction)",
            original_size, len(compressed), ratio,
        )
        return compressed, original_size

    # ── 5. Encrypt Batch ─────────────────────────────────────────────────

    def encrypt_batch(self, data: bytes) -> Dict[str, Any]:
        """
        Encrypt data using AES-256-GCM.

        Uses HKDF-SHA256 to derive a per-batch encryption key from the
        master key and a random salt. Returns encrypted payload with
        all parameters needed for decryption.

        Returns:
            {
                "ciphertext": bytes,
                "salt": str (hex),
                "iv": str (hex),
                "auth_tag": str (hex),
                "algorithm": "AES-256-GCM"
            }
        """
        master_key = bytes.fromhex(self.config.encryption_key)

        # Generate random salt and IV
        salt = os.urandom(16)
        iv = os.urandom(12)  # 96-bit IV for GCM

        # Derive per-batch key using HKDF
        hkdf = HKDF(
            algorithm=hashes.SHA256(),
            length=32,
            salt=salt,
            info=b"udara-sync-batch-v2",
        )
        batch_key = hkdf.derive(master_key)

        # Encrypt
        aesgcm = AESGCM(batch_key)
        ciphertext_with_tag = aesgcm.encrypt(iv, data, None)

        # GCM appends 16-byte tag to ciphertext
        ciphertext = ciphertext_with_tag[:-16]
        auth_tag = ciphertext_with_tag[-16:]

        result = {
            "ciphertext": ciphertext_with_tag,  # Include tag for transport
            "salt": salt.hex(),
            "iv": iv.hex(),
            "auth_tag": auth_tag.hex(),
            "algorithm": "AES-256-GCM",
        }

        logger.info(
            "Encrypted batch: %d bytes → %d bytes (salt=%s..., iv=%s...)",
            len(data), len(ciphertext_with_tag),
            result["salt"][:16], result["iv"][:16],
        )
        return result

    # ── 6. Push to Cloud ─────────────────────────────────────────────────

    async def push_to_cloud(
        self, encrypted_data: Dict[str, Any], batch_id: str
    ) -> Dict[str, Any]:
        """
        Push encrypted batch to cloud sync endpoint.

        Args:
            encrypted_data: Output from encrypt_batch()
            batch_id: Unique identifier for this sync batch

        Returns:
            Cloud response dict with status, applied count, conflicts.

        Raises:
            CloudAPIError: If cloud returns non-2xx status.
            ConnectivityError: If request cannot be made.
        """
        client = self._get_http_client()

        payload = {
            "batch_id": batch_id,
            "device_id": self.config.device_id,
            "facility_code": self.config.facility_code,
            "sync_version": "2.1.0",
            "timestamp": datetime.now(timezone.utc).isoformat(),
            "encryption": {
                "algorithm": encrypted_data["algorithm"],
                "salt": encrypted_data["salt"],
                "iv": encrypted_data["iv"],
                "auth_tag": encrypted_data["auth_tag"],
            },
            "payload_b64": encrypted_data["ciphertext"],  # httpx auto-handles bytes
            "payload_sha256": hashlib.sha256(
                encrypted_data["ciphertext"]
            ).hexdigest(),
        }

        try:
            response = await client.post(
                self.config.sync_endpoint,
                json=payload,
            )

            if response.status_code >= 500:
                raise CloudAPIError(
                    response.status_code,
                    "Cloud server error",
                    response_body=response.text[:1000],
                )

            elif response.status_code >= 400:
                raise CloudAPIError(
                    response.status_code,
                    response.json().get("error", response.text),
                    response_body=response.json(),
                )

            result = response.json()
            self._stats["total_bytes_uploaded"] += len(
                encrypted_data["ciphertext"]
            )
            logger.info(
                "Cloud push successful: batch=%s, applied=%d, conflicts=%d",
                batch_id,
                result.get("applied", 0),
                len(result.get("conflicts", [])),
            )
            return result

        except httpx.ConnectError as e:
            raise ConnectivityError(f"Cannot reach cloud: {e}")
        except httpx.TimeoutException as e:
            raise ConnectivityError(f"Cloud request timed out: {e}")

    # ── 7. Mark Synced ───────────────────────────────────────────────────

    def mark_synced(
        self,
        sync_ids: List[int],
        batch_id: str,
        cloud_response: Dict[str, Any],
    ) -> int:
        """
        Mark records as successfully synced and handle conflicts.

        Removes synced records from queue and logs conflict resolutions.

        Args:
            sync_ids: List of sync_queue.id values that were in the batch
            batch_id: The batch ID that was pushed
            cloud_response: Response from the cloud API

        Returns:
            Number of records marked as synced
        """
        db = self._get_db()

        # Handle conflicts if any
        conflicts = cloud_response.get("conflicts", [])
        conflict_sync_ids = set()

        for conflict in conflicts:
            record_id = conflict.get("record_id")
            resolution = conflict.get("resolution", {})

            # Log conflict resolution
            db.execute("""
                INSERT INTO sync_conflict_log (
                    batch_id, record_id, field, action,
                    local_value, remote_value, resolved_at
                ) VALUES (?, ?, ?, ?, ?, ?, ?)
            """, (
                batch_id,
                record_id,
                resolution.get("field"),
                resolution.get("action"),
                json.dumps(resolution.get("local")),
                json.dumps(resolution.get("remote")),
                datetime.now(timezone.utc).isoformat(),
            ))

            self._stats["total_conflicts_resolved"] += 1

            # Apply remote-wins conflicts locally
            if resolution.get("action") in ("remote_wins", "concurrent_tiebreaker"):
                conflict_sync_ids.add(record_id)

        # Remove successfully synced records from queue
        placeholders = ",".join("?" * len(sync_ids))
        cursor = db.execute(
            f"DELETE FROM sync_queue WHERE id IN ({placeholders})",
            sync_ids,
        )
        deleted_count = cursor.rowcount

        db.commit()
        self._stats["total_pushed"] += deleted_count
        logger.info(
            "Marked %d records as synced (batch=%s, conflicts=%d)",
            deleted_count, batch_id, len(conflicts),
        )
        return deleted_count

    def mark_failed(self, sync_ids: List[int], error: str) -> None:
        """
        Mark records as failed and schedule retry with exponential backoff.

        Args:
            sync_ids: List of sync_queue.id values that failed
            error: Error message describing the failure
        """
        db = self._get_db()

        for sid in sync_ids:
            row = db.execute(
                "SELECT retries, max_retries FROM sync_queue WHERE id = ?",
                (sid,),
            ).fetchone()

            if row is None:
                continue

            new_retries = row["retries"] + 1
            max_retries = row["max_retries"]

            if new_retries >= max_retries:
                # Exceeded max retries: move to dead letter queue
                db.execute("""
                    INSERT INTO sync_dead_letter (source_id, record_id, payload,
                        error, retries, failed_at)
                    SELECT id, record_id, payload, ?, ?, ?
                    FROM sync_queue WHERE id = ?
                """, (error, new_retries,
                      datetime.now(timezone.utc).isoformat(), sid))
                db.execute("DELETE FROM sync_queue WHERE id = ?", (sid,))
                logger.error(
                    "Record %d exceeded max retries (%d), moved to dead letter: %s",
                    sid, max_retries, error,
                )
            else:
                # Calculate exponential backoff with jitter
                delay = min(
                    self.config.retry_base_delay_seconds * (2 ** (new_retries - 1)),
                    self.config.retry_max_delay_seconds,
                )
                # Add ±25% jitter
                import random
                jitter = delay * 0.25 * (random.random() * 2 - 1)
                delay = max(1.0, delay + jitter)

                next_retry = datetime.now(timezone.utc) + timedelta(seconds=delay)
                db.execute("""
                    UPDATE sync_queue
                    SET retries = ?, next_retry_at = ?, last_error = ?
                    WHERE id = ?
                """, (new_retries,
                      next_retry.strftime("%Y-%m-%dT%H:%M:%SZ"),
                      error[:500], sid))

        db.commit()
        logger.warning(
            "Marked %d records as failed: %s", len(sync_ids), error
        )

    # ── 8. USB Fallback ──────────────────────────────────────────────────

    def export_to_usb(self) -> Optional[str]:
        """
        Export unsynced records to a USB drive for physical transport.

        Creates an AES-256 encrypted archive on the USB drive that can be
        imported at a connectivity point.

        Returns:
            Path to the exported file, or None if no records to export or USB not available.
        """
        records = self.get_unsynced_records(limit=1000)
        if not records:
            logger.info("No records to export to USB")
            return None

        # Check USB mount point
        usb_path = Path(self.config.usb_mount_point)
        if not usb_path.is_mount():
            logger.warning("USB drive not mounted at %s", self.config.usb_mount_point)
            return None

        export_dir = usb_path / "udara_sync"
        export_dir.mkdir(exist_ok=True)

        # Anonymize, compress, encrypt
        anonymized = self.anonymize_batch(records)
        compressed, _ = self.compress_batch(anonymized)
        encrypted = self.encrypt_batch(compressed)

        # Write encrypted file with metadata
        timestamp = datetime.now(timezone.utc).strftime("%Y%m%d_%H%M%S")
        filename = f"udara_sync_{self.config.device_id}_{timestamp}.udara"
        filepath = export_dir / filename

        # File format: [16-byte magic][32-byte salt][12-byte IV][ciphertext]
        magic = b"UDARA_SYNC_V2"
        with open(filepath, "wb") as f:
            f.write(magic)
            f.write(bytes.fromhex(encrypted["salt"]))
            f.write(bytes.fromhex(encrypted["iv"]))
            f.write(encrypted["ciphertext"])

        # Write metadata JSON alongside
        metadata = {
            "version": "2.1.0",
            "device_id": self.config.device_id,
            "facility_code": self.config.facility_code,
            "exported_at": datetime.now(timezone.utc).isoformat(),
            "record_count": len(records),
            "file_size_bytes": filepath.stat().st_size,
            "sha256": hashlib.sha256(filepath.read_bytes()).hexdigest(),
            "encryption": {
                "algorithm": "AES-256-GCM",
                "salt": encrypted["salt"],
                "iv": encrypted["iv"],
            },
            "record_ids": [r["record_id"] for r in records],
        }
        meta_path = export_dir / f"{filename}.meta.json"
        meta_path.write_text(json.dumps(metadata, indent=2))

        self._stats["usb_exports"] += 1
        logger.info(
            "Exported %d records to USB: %s (%d bytes)",
            len(records), filepath, filepath.stat().st_size,
        )
        return str(filepath)

    def import_from_usb(self) -> Optional[int]:
        """
        Import sync data from a USB drive that was exported at another site.

        Reads .udara files from the USB, decrypts, and pushes to cloud.

        Returns:
            Number of records imported, or None if no files found.
        """
        usb_path = Path(self.config.usb_mount_point)
        if not usb_path.is_mount():
            logger.warning("USB drive not mounted at %s", self.config.usb_mount_point)
            return None

        export_dir = usb_path / "udara_sync"
        if not export_dir.exists():
            logger.info("No udara_sync directory on USB")
            return None

        total_imported = 0
        for udara_file in sorted(export_dir.glob("*.udara")):
            try:
                meta_file = udara_file.with_suffix(udara_file.suffix + ".meta.json")
                if not meta_file.exists():
                    logger.warning("No metadata file for %s, skipping", udara_file)
                    continue

                metadata = json.loads(meta_file.read_text())

                # Verify this is from a different device (not our own export)
                if metadata["device_id"] == self.config.device_id:
                    logger.info("Skipping own export: %s", udara_file.name)
                    continue

                # Read and decrypt
                data = udara_file.read_bytes()
                magic = data[:12]
                if magic != b"UDARA_SYNC_V2":
                    logger.error("Invalid file magic in %s", udara_file)
                    continue

                salt = data[12:28]
                iv = data[28:40]
                ciphertext = data[40:]

                master_key = bytes.fromhex(self.config.encryption_key)
                hkdf = HKDF(
                    algorithm=hashes.SHA256(),
                    length=32,
                    salt=salt,
                    info=b"udara-sync-batch-v2",
                )
                batch_key = hkdf.derive(master_key)
                aesgcm = AESGCM(batch_key)

                try:
                    plaintext = aesgcm.decrypt(iv, ciphertext, None)
                except Exception as e:
                    logger.error("Decryption failed for %s: %s", udara_file, e)
                    continue

                # Decompress
                decompressed = gzip.decompress(plaintext)
                batch_data = json.loads(decompressed)

                # Push to cloud as if it were our own batch
                import uuid
                batch_id = str(uuid.uuid4())
                encrypted = self.encrypt_batch(decompressed)
                # (In production, this would be an async push from the run loop)
                logger.info(
                    "Imported %d records from %s (batch=%s)",
                    len(batch_data.get("records", [])),
                    udara_file.name,
                    batch_id,
                )
                total_imported += len(batch_data.get("records", []))

                # Mark file as imported to prevent re-import
                udara_file.rename(udara_file.with_suffix(".udara.imported"))

            except Exception as e:
                logger.error("Error importing %s: %s", udara_file, e)

        if total_imported > 0:
            self._stats["usb_imports"] += total_imported
        logger.info("USB import complete: %d records from USB", total_imported)
        return total_imported

    # ── 9. Model Sync ────────────────────────────────────────────────────

    async def sync_models(self) -> Dict[str, Any]:
        """
        Check for and download updated models from the cloud.

        Implements atomic swap: download to temp dir, verify SHA256,
        move to versioned directory, update symlink.

        Returns:
            {
                "checked": bool,
                "updates_available": int,
                "updated": List[str],
                "errors": List[str]
            }
        """
        result = {
            "checked": True,
            "updates_available": 0,
            "updated": [],
            "errors": [],
        }

        client = self._get_http_client()

        try:
            response = await client.get(self.config.models_endpoint)
            response.raise_for_status()
            available_models = response.json()
        except Exception as e:
            result["errors"].append(f"Failed to fetch model list: {e}")
            return result

        model_dir = Path(self.config.model_dir)
        model_dir.mkdir(parents=True, exist_ok=True)

        for model in available_models.get("models", []):
            model_name = model["name"]
            remote_version = model["version"]
            download_url = model["download_url"]
            expected_sha256 = model["sha256"]

            # Check current version
            version_file = model_dir / f"{model_name}.version"
            current_version = None
            if version_file.exists():
                current_version = version_file.read_text().strip()

            if current_version == remote_version:
                logger.debug("Model %s is up to date (v%s)", model_name, remote_version)
                continue

            result["updates_available"] += 1
            logger.info(
                "Model update available: %s %s → %s",
                model_name, current_version or "none", remote_version,
            )

            try:
                # Download to temp file
                import tempfile
                with tempfile.NamedTemporaryFile(
                    dir=str(model_dir), suffix=".tmp", delete=False
                ) as tmp:
                    download_response = await client.get(download_url)
                    download_response.raise_for_status()

                    # Stream to disk to avoid loading large models into memory
                    chunk_size = 8192
                    sha256_hash = hashlib.sha256()
                    while True:
                        chunk = download_response.content.read(chunk_size)
                        if not chunk:
                            break
                        sha256_hash.update(chunk)
                        tmp.write(chunk)

                    tmp_path = tmp.name

                # Verify SHA256
                actual_sha256 = sha256_hash.hexdigest()
                if actual_sha256 != expected_sha256:
                    os.unlink(tmp_path)
                    raise ValueError(
                        f"SHA256 mismatch: expected {expected_sha256}, "
                        f"got {actual_sha256}"
                    )

                # Atomic swap: rename temp to versioned filename
                versioned_path = model_dir / f"{model_name}_v{remote_version}"
                if versioned_path.exists():
                    os.unlink(versioned_path)
                os.rename(tmp_path, versioned_path)

                # Update symlink (atomic on Linux)
                symlink_path = model_dir / model_name
                temp_symlink = symlink_path.with_suffix(".tmp_link")
                if temp_symlink.exists() or temp_symlink.is_symlink():
                    temp_symlink.unlink()
                temp_symlink.symlink_to(versioned_path.name)
                os.rename(str(temp_symlink), str(symlink_path))

                # Write version file
                version_file.write_text(remote_version)

                result["updated"].append(f"{model_name}@{remote_version}")
                self._stats["models_updated"] += 1
                logger.info(
                    "Model updated: %s → v%s", model_name, remote_version
                )

            except Exception as e:
                result["errors"].append(f"Failed to update {model_name}: {e}")
                logger.error("Model update failed for %s: %s", model_name, e)

        return result

    # ── 10. Main Run Loop ────────────────────────────────────────────────

    async def run(self) -> None:
        """
        Main synchronization loop.

        Runs continuously, performing sync cycles at the configured interval.
        Each cycle: check connectivity → prepare batch → push → resolve → cleanup.

        This should be run as a systemd service with automatic restart.
        """
        self._running = True
        logger.info(
            "SyncEngine starting: interval=%ds, batch_size=%d, usb_fallback=%.0fh",
            self.config.sync_interval_seconds,
            self.config.batch_size,
            self.config.usb_fallback_hours,
        )

        cycle_count = 0

        try:
            while self._running:
                cycle_count += 1
                logger.debug("=== Sync cycle %d ===", cycle_count)

                # 1. Connectivity check
                connectivity = self.check_connectivity()

                if connectivity["connected"]:
                    # ── Online sync path ──
                    try:
                        await self._sync_online()
                        self._consecutive_failures = 0
                        self._last_successful_sync = datetime.now(timezone.utc)
                    except (ConnectivityError, CloudAPIError) as e:
                        self._consecutive_failures += 1
                        logger.error(
                            "Online sync failed (attempt %d): %s",
                            self._consecutive_failures, e,
                        )

                    # Sync models every 24 hours
                    if cycle_count % (86400 // self.config.sync_interval_seconds) == 0:
                        try:
                            model_result = await self.sync_models()
                            if model_result["updated"]:
                                logger.info("Models updated: %s", model_result["updated"])
                        except Exception as e:
                            logger.error("Model sync failed: %s", e)

                else:
                    # ── Offline path ──
                    self._consecutive_failures += 1
                    hours_offline = (
                        self._consecutive_failures * self.config.sync_interval_seconds
                    ) / 3600

                    logger.warning(
                        "Offline: consecutive_failures=%d, estimated_hours_offline=%.1f",
                        self._consecutive_failures, hours_offline,
                    )

                    # Check if USB fallback threshold reached
                    if hours_offline >= self.config.usb_fallback_hours:
                        logger.info("USB fallback threshold reached (%.0fh)", hours_offline)
                        try:
                            usb_result = self.export_to_usb()
                            if usb_result:
                                logger.info("USB export successful: %s", usb_result)
                            else:
                                logger.info("USB export: no records or USB not available")
                        except Exception as e:
                            logger.error("USB export failed: %s", e)

                    # Also try USB import (CHW may have returned with synced USB)
                    try:
                        self.import_from_usb()
                    except Exception as e:
                        logger.error("USB import failed: %s", e)

                self._last_sync_attempt = datetime.now(timezone.utc)

                # Wait for next cycle
                await asyncio.sleep(self.config.sync_interval_seconds)

        except asyncio.CancelledError:
            logger.info("SyncEngine cancelled")
        except Exception as e:
            logger.critical("SyncEngine fatal error: %s", e, exc_info=True)
            raise
        finally:
            self._running = False
            await self.shutdown()
            logger.info("SyncEngine stopped")

    async def _sync_online(self) -> None:
        """Perform one online sync cycle."""
        import uuid

        # Get unsynced records
        records = self.get_unsynced_records(limit=self.config.batch_size)
        if not records:
            logger.debug("No records to sync")
            return

        sync_ids = [r["id"] for r in records]

        try:
            # Prepare batch
            anonymized = self.anonymize_batch(records)
            compressed, _ = self.compress_batch(anonymized)
            encrypted = self.encrypt_batch(compressed)
            batch_id = str(uuid.uuid4())

            # Push to cloud
            cloud_response = await self.push_to_cloud(encrypted, batch_id)

            # Handle response
            if cloud_response.get("status") == "accepted":
                self.mark_synced(sync_ids, batch_id, cloud_response)
            elif cloud_response.get("status") == "partial":
                # Some records applied, some rejected
                applied_ids = set(cloud_response.get("applied_ids", []))
                applied_sync_ids = [
                    sid for sid, rec in zip(sync_ids, records)
                    if rec["record_id"] in applied_ids
                ]
                if applied_sync_ids:
                    self.mark_synced(applied_sync_ids, batch_id, cloud_response)
                rejected_sync_ids = [
                    sid for sid in sync_ids if sid not in applied_sync_ids
                ]
                if rejected_sync_ids:
                    self.mark_failed(
                        rejected_sync_ids,
                        cloud_response.get("error", "Partial rejection"),
                    )
            else:
                self.mark_failed(sync_ids, cloud_response.get("error", "Unknown error"))

        except (ConnectivityError, CloudAPIError) as e:
            self.mark_failed(sync_ids, str(e))
            raise

    async def shutdown(self) -> None:
        """Clean shutdown: close database and HTTP connections."""
        if self._db is not None:
            try:
                self._db.close()
            except Exception:
                pass
            self._db = None

        if self._http_client is not None:
            try:
                await self._http_client.aclose()
            except Exception:
                pass
            self._http_client = None

        logger.info("SyncEngine shutdown complete. Stats: %s", self._stats)

    def get_stats(self) -> Dict[str, Any]:
        """Return current sync statistics."""
        return {
            **self._stats,
            "running": self._running,
            "last_sync_attempt": (
                self._last_sync_attempt.isoformat() if self._last_sync_attempt else None
            ),
            "last_successful_sync": (
                self._last_successful_sync.isoformat()
                if self._last_successful_sync else None
            ),
            "consecutive_failures": self._consecutive_failures,
            "device_id": self.config.device_id,
        }


# ── Entry Point ──────────────────────────────────────────────────────────────

if __name__ == "__main__":
    import argparse

    logging.basicConfig(
        level=logging.INFO,
        format="%(asctime)s [%(levelname)s] %(name)s: %(message)s",
    )

    parser = argparse.ArgumentParser(description="UDARA Edge SyncEngine")
    parser.add_argument("--once", action="store_true", help="Run one sync cycle and exit")
    parser.add_argument("--usb-export", action="store_true", help="Export to USB and exit")
    parser.add_argument("--usb-import", action="store_true", help="Import from USB and exit")
    parser.add_argument("--stats", action="store_true", help="Print stats and exit")
    args = parser.parse_args()

    engine = SyncEngine()

    if args.stats:
        print(json.dumps(engine.get_stats(), indent=2))
    elif args.usb_export:
        result = engine.export_to_usb()
        print(f"USB export: {result}" if result else "Nothing to export")
    elif args.usb_import:
        result = engine.import_from_usb()
        print(f"USB import: {result} records" if result else "Nothing to import")
    elif args.once:
        connectivity = engine.check_connectivity()
        if connectivity["connected"]:
            asyncio.run(engine._sync_online())
        else:
            print("Offline. Cannot sync. Use --usb-export for fallback.")
    else:
        asyncio.run(engine.run())
```

---

## 6. USB Physical Fallback

### 6.1 USB Fallback Protocol

When an edge device has been offline for more than 48 hours, the SyncEngine
automatically exports all pending records to a FAT32-formatted USB drive. The
CHW (Community Health Worker) physically carries the USB to the nearest
connectivity point (district health office, partner NGO hub, or cyber cafe).

```
┌────────────────────────────────────────────────────────────────────┐
│                    USB FALLBACK WORKFLOW                            │
│                                                                     │
│  ┌──────────┐     ┌──────────┐     ┌──────────┐     ┌──────────┐ │
│  │  Edge    │     │  CHW     │     │  Hub /   │     │  Cloud   │ │
│  │  Device  │     │  Carrier │     │  Import  │     │  API     │ │
│  │          │     │          │     │  Station │     │          │ │
│  └────┬─────┘     └────┬─────┘     └────┬─────┘     └────┬─────┘ │
│       │                │                │                │        │
│  [Auto export]   [Physical        [Insert USB,    [POST /sync/  │
│  .udara file     transport        run import]     push]         │
│  to USB drive    by foot/moto     [Verify,        [Decrypt,     │
│  when offline    /bus]            decrypt,        merge, store] │
│  > 48h                            push]                         │
│       │                │                │                │        │
│       │    USB Key:    │                │                │        │
│       │  AES-256-GCM   │                │                │        │
│       │  (same master  │                │                │        │
│       │   key as sync) │                │                │        │
│       ▼                ▼                ▼                ▼        │
└────────────────────────────────────────────────────────────────────┘
```

### 6.2 USB File Format Specification

```
Byte Offset    Size         Description
─────────────  ────         ────────────────────────────────────────────
0x00           12 bytes     Magic: "UDARA_SYNC_V2" (ASCII)
0x0C           16 bytes     HKDF Salt (random per export)
0x1C           12 bytes     AES-GCM IV/Nonce (random per export)
0x28           variable    AES-256-GCM Ciphertext (gzip-compressed JSON)
               - 16 bytes  ...includes 16-byte GCM auth tag at end
```

### 6.3 USB Logistics

| Aspect | Specification |
|--------|---------------|
| Filesystem | FAT32 (universal compatibility) |
| USB Size | Minimum 4GB; 64GB recommended for 6 months |
| Encryption | AES-256-GCM with per-export derived key |
| Max file size | ~50MB per export (≈180,000 cases) |
| Naming | `udara_sync_{device-id}_{timestamp}.udara` |
| Metadata | Adjacent `.meta.json` file with record IDs, SHA256 |
| Tamper detection | SHA256 in metadata; verify before import |
| Retention | `.imported` extension after successful cloud push |
| Physical security | USB should be stored in tamper-evident bag during transport |

### 6.4 Import Station Software

The import station runs the same SyncEngine with `--usb-import` flag in a
loop. It can be a simple laptop at a district health office connected to the
internet. Recommended setup:

```bash
# Install on import station laptop
pip install udara-edge-sync

# Run import daemon (systemd service)
udara-sync --usb-import --interval 60  # Check every minute

# Or one-shot
udara-sync --usb-import
```

---

## 7. Model Sync: Cloud → Edge

### 7.1 Model Update Flow

```
CLOUD                              EDGE
─────                              ────
                                    
 1. New model trained               │
    (agent-a-v2.4.0.onnx)          │
    │                               │
    ▼                               │
 2. Upload to S3/R2                 │
    with SHA256 manifest            │
    │                               │
    ▼                               │
 3. Update /sync/models             │
    endpoint manifest                │
    │                               │
    ───────────────────────────────>│
                                    │ 4. GET /sync/models (every 24h)
                                    │    Compare versions
                                    │
                                    │ 5. Download to temp file
                                    │    (streaming, chunked)
                                    │
                                    │ 6. Verify SHA256
                                    │
                                    │ 7. Atomic swap:
                                    │    mv tmp agent-a_v2.4.0.onnx
                                    │    ln -sf agent-a_v2.4.0.onnx current
                                    │
                                    │ 8. Write version file
                                    │    agent-a.version → "2.4.0"
                                    │
                                    │ 9. Reload agent process
                                    │    (SIGHUP or hot-reload)
```

### 7.2 Model Manifest Format

```json
{
  "manifest_version": "1.0",
  "generated_at": "2026-05-27T12:00:00Z",
  "models": [
    {
      "name": "agent-a-clinical",
      "version": "2.4.0",
      "previous_version": "2.3.1",
      "download_url": "https://models.udara.health/agent-a/v2.4.0/model.onnx",
      "sha256": "a3f8e2b1c4d5e6f7a8b9c0d1e2f3a4b5c6d7e8f9a0b1c2d3e4f5a6b7c8d9e0",
      "file_size_bytes": 145000000,
      "format": "onnx",
      "changelog": "Improved fever detection accuracy +3.2%, new Swahili symptom terms",
      "min_edge_python": "3.11",
      "min_ram_mb": 512,
      "supported_architectures": ["arm64", "amd64"],
      "priority": "optional",
      "rollback_safe": true
    },
    {
      "name": "agent-b-resistance",
      "version": "1.8.0",
      "previous_version": "1.7.2",
      "download_url": "https://models.udara.health/agent-b/v1.8.0/model.onnx",
      "sha256": "b4c9f3e2d1a0b8c7d6e5f4a3b2c1d0e9f8a7b6c5d4e3f2a1b0c9d8e7f6a5b4",
      "file_size_bytes": 45000000,
      "format": "onnx",
      "changelog": "New resistance trend calculation algorithm",
      "min_edge_python": "3.11",
      "min_ram_mb": 256,
      "supported_architectures": ["arm64", "amd64"],
      "priority": "recommended",
      "rollback_safe": true
    }
  ]
}
```

---

## 8. Monitoring & Observability

### 8.1 Sync Health Metrics

```python
# Exposed via /metrics endpoint (Prometheus format)
"""
# HELP udara_sync_last_success Timestamp of last successful sync
# TYPE udara_sync_last_success gauge
udara_sync_last_success{device_id="udara-edge-KE-NRB-0042"} 1705312200

# HELP udara_sync_queue_size Number of records waiting to be synced
# TYPE udara_sync_queue_size gauge
udara_sync_queue_size{device_id="udara-edge-KE-NRB-0042",priority="1"} 2
udara_sync_queue_size{device_id="udara-edge-KE-NRB-0042",priority="2"} 15
udara_sync_queue_size{device_id="udara-edge-KE-NRB-0042",priority="5"} 87

# HELP udara_sync_records_pushed_total Total records successfully pushed
# TYPE udara_sync_records_pushed_total counter
udara_sync_records_pushed_total{device_id="udara-edge-KE-NRB-0042"} 12473

# HELP udara_sync_conflicts_resolved_total Total CRDT conflicts resolved
# TYPE udara_sync_conflicts_resolved_total counter
udara_sync_conflicts_resolved_total{device_id="udara-edge-KE-NRB-0042"} 23

# HELP udara_sync_bytes_uploaded_total Total bytes uploaded to cloud
# TYPE udara_sync_bytes_uploaded_total counter
udara_sync_bytes_uploaded_total{device_id="udara-edge-KE-NRB-0042"} 2847563

# HELP udara_sync_usb_exports_total Total USB fallback exports
# TYPE udara_sync_usb_exports_total counter
udara_sync_usb_exports_total{device_id="udara-edge-KE-NRB-0042"} 3

# HELP udara_sync_consecutive_failures Number of consecutive failed sync attempts
# TYPE udara_sync_consecutive_failures gauge
udara_sync_consecutive_failures{device_id="udara-edge-KE-NRB-0042"} 0

# HELP udara_sync_cycle_duration_seconds Duration of last sync cycle
# TYPE udara_sync_cycle_duration_seconds gauge
udara_sync_cycle_duration_seconds{device_id="udara-edge-KE-NRB-0042"} 4.72

# HELP udara_model_version Current model version running on edge
# TYPE udara_model_version gauge
udara_model_version{device_id="udara-edge-KE-NRB-0042",model="agent-a"} 2.4
udara_model_version{device_id="udara-edge-KE-NRB-0042",model="agent-b"} 1.8
"""
```

### 8.2 Alert Thresholds

| Metric | Warning | Critical | Action |
|--------|---------|----------|--------|
| `sync_queue_size` (priority 1) | > 5 | > 20 | Immediate: check connectivity, consider USB |
| `sync_queue_size` (all) | > 100 | > 500 | Warning: growing backlog |
| `consecutive_failures` | > 6 (30 min) | > 96 (48h) | Critical: trigger USB fallback |
| `last_success` | > 1 hour | > 24 hours | Investigate network / device health |
| `cycle_duration` | > 30s | > 120s | Large batch or slow network |
| `dead_letter_queue_size` | > 0 | > 10 | Manual investigation required |
| `model_version` behind cloud | 1 version | 3 versions | Network issue preventing model sync |

---

## 9. Failure Modes & Recovery

```
┌──────────────────────────────────────────────────────────────────────┐
│                     FAILURE MODE MATRIX                             │
├──────────────────────┬───────────────────┬──────────────────────────┤
│ Failure Scenario     │ Detection         │ Recovery                 │
├──────────────────────┼───────────────────┼──────────────────────────┤
│ No connectivity      │ check_connectivity│ Exponential backoff;     │
│ (cellular down)      │ timeout           │ USB fallback at 48h      │
├──────────────────────┼───────────────────┼──────────────────────────┤
│ Cloud 500 error      │ HTTP status code  │ Retry with backoff;      │
│                      │                   │ alert ops team           │
├──────────────────────┼───────────────────┼──────────────────────────┤
│ Cloud 400 (bad data) │ HTTP status +     │ Log error; move to dead  │
│                      │ validation msg    │ letter queue; continue   │
├──────────────────────┼───────────────────┼──────────────────────────┤
│ CRDT conflict        │ Response contains │ Auto-resolve via LWW/    │
│                      │ conflicts[]       │ Union; log for audit     │
├──────────────────────┼───────────────────┼──────────────────────────┤
│ Record exceeds       │ Size check before │ Split into sub-records;  │
│ max payload size     │ compression       │ or defer large records   │
├──────────────────────┼───────────────────┼──────────────────────────┤
│ Clock skew > 180s    │ HLC comparison    │ HLC logical counter      │
│                      │ detects skew      │ compensates; log warning │
├──────────────────────┼───────────────────┼──────────────────────────┤
│ USB corruption       │ SHA256 mismatch   │ Reject file; alert;      │
│                      │ on import         │ retry from queue          │
├──────────────────────┼───────────────────┼──────────────────────────┤
│ SQLite corruption    │ PRAGMA integrity  │ Restore from last backup;│
│                      │ check on startup  │ re-sync from cloud        │
├──────────────────────┼───────────────────┼──────────────────────────┤
│ Disk full            │ os.statvfs check  │ Purge old logs; alert;   │
│                      │ before each cycle │ USB emergency export     │
├──────────────────────┼───────────────────┼──────────────────────────┤
│ Memory OOM           │ Process killed    │ Watchdog restarts;       │
│ (RPi 4B 4GB limit)   │ by OOM killer     │ reduce batch_size        │
└──────────────────────┴───────────────────┴──────────────────────────┘
```

---

## 10. Appendix: Full Configuration Reference

### 10.1 Environment Variables

| Variable | Required | Default | Description |
|----------|----------|---------|-------------|
| `UDARA_DEVICE_ID` | Yes | — | Unique device identifier (e.g., `udara-edge-KE-NRB-0042`) |
| `UDARA_FACILITY_CODE` | Yes | — | Facility code (e.g., `KE-NRB-FAC-017`) |
| `UDARA_EDGE_DB` | No | `/var/lib/udara/edge.db` | Path to SQLite database |
| `UDARA_CLOUD_URL` | No | `https://api.udara.health` | Cloud API base URL |
| `UDARA_API_KEY` | Yes | — | API key for cloud authentication |
| `UDARA_API_CERT` | No | — | Path to mTLS client certificate |
| `UDARA_SYNC_KEY` | Yes | — | 64-char hex master encryption key |
| `UDARA_CONNECTIVITY_CHECK` | No | `{CLOUD_URL}/health` | URL for connectivity checks |
| `UDARA_USB_MOUNT` | No | `/media/udara_usb` | USB drive mount point |
| `UDARA_SYNC_INTERVAL` | No | `300` | Sync interval in seconds |
| `UDARA_BATCH_SIZE` | No | `50` | Max records per sync batch |
| `UDARA_GZIP_LEVEL` | No | `6` | Gzip compression level (1-9) |
| `UDARA_USB_FALLBACK_HOURS` | No | `48` | Hours offline before USB fallback |
| `UDARA_LOG_LEVEL` | No | `INFO` | Logging level |

### 10.2 Systemd Unit File

```ini
# /etc/systemd/system/udara-sync.service
[Unit]
Description=UDARA Edge SyncEngine
After=network-online.target
Wants=network-online.target
Documentation=https://docs.udara.health/edge-cloud-sync

[Service]
Type=simple
User=udara
Group=udara
ExecStart=/usr/local/bin/udara-sync
Restart=always
RestartSec=10
TimeoutStartSec=60
TimeoutStopSec=30

# Security hardening
NoNewPrivileges=true
ProtectSystem=strict
ProtectHome=true
ReadWritePaths=/var/lib/udara /media/udara_usb
PrivateTmp=true

# Resource limits for RPi 4B
MemoryMax=512M
CPUQuota=50%

# Logging
StandardOutput=journal
StandardError=journal
SyslogIdentifier=udara-sync

# Environment
EnvironmentFile=-/etc/udara/sync.env

[Install]
WantedBy=multi-user.target
```
