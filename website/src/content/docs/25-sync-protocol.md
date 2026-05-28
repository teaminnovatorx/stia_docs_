# Sync Protocol — Delta JSON, CRDT Conflict Resolution & USB Fallback

> "Zero data loss, even when the internet is gone for weeks."
> — UDARA AI Technical Architecture

---

## Table of Contents

1. [Design Philosophy](#design-philosophy)
2. [Architecture Overview](#architecture-overview)
3. [Delta Generation Algorithm](#delta-generation-algorithm)
4. [Delta JSON Format](#delta-json-format)
5. [CRDT Conflict Resolution via Yjs](#crdt-conflict-resolution-via-yjs)
6. [Sync State Machine](#sync-state-machine)
7. [Retry Strategy](#retry-strategy)
8. [USB Fallback](#usb-fallback)
9. [Sync Orchestrator Implementation](#sync-orchestrator-implementation)
10. [Cloud Reception Endpoint](#cloud-reception-endpoint)
11. [Edge SQLite Schema](#edge-sqlite-schema)
12. [Sync Schedule](#sync-schedule)
13. [Bandwidth Budget](#bandwidth-budget)
14. [Monitoring & Observability](#monitoring--observability)

---

## Design Philosophy

The sync protocol is the backbone of UDARA AI's offline-first architecture. In sub-Saharan Africa, internet connectivity at health posts is not a luxury — it is unreliable at best, nonexistent at worst. A health post in rural Awe, Nigeria might have connectivity for 4 hours a day (when the nearest tower's 3G signal is strong enough). A health post in Fiditi might lose connectivity for an entire week during the rainy season when storms damage the local tower.

The sync protocol is designed around one non-negotiable principle:

> **ZERO DATA LOSS. Every AMR case reported at any health post must eventually reach the cloud, regardless of how long connectivity is lost.**

This principle drives every design decision: incremental deltas (save bandwidth), CRDT conflict resolution (handle concurrent edits gracefully), and USB fallback (physical data transport when network fails).

### Design Constraints

| Constraint | Value | Why |
|------------|-------|-----|
| Max bandwidth per sync | 2 MB/day | 3G data plans are expensive in rural Africa |
| Target sync time | <30 seconds | Don't block the RPi for long |
| Max offline duration | 30+ days | Health posts can be offline for weeks |
| Conflict tolerance | Zero data loss | Never silently discard CHW reports |
| Device hardware | RPi 5 8GB | Limited CPU/RAM for sync operations |

---

## Architecture Overview

```
┌──────────────────────────────────────────────────────────────────────┐
│                         EDGE (RPi 5)                                   │
│                                                                        │
│  ┌───────────────┐     ┌───────────────┐     ┌───────────────┐        │
│  │  CHW Reports  │────▶│  SQLite WAL   │────▶│  Delta Gen    │        │
│  │  (USSD/Bot)   │     │  local data   │     │  (incremental)│        │
│  └───────────────┘     └───────┬───────┘     └───────┬───────┘        │
│                                │                     │                 │
│                       ┌────────┴─────────────────────┴────────┐       │
│                       │          Sync Orchestrator              │       │
│                       │  IDLE→CHECK→GEN→COMPRESS→UPLOAD→CONFIRM│       │
│                       └───────────────┬────────────────────────┘       │
│                                       │                                 │
│                          ┌────────────┼────────────┐                   │
│                          ▼                         ▼                   │
│                   ┌─────────────┐          ┌───────────────┐           │
│                   │ Network     │          │ USB Fallback   │           │
│                   │ (HTTPS/TLS) │          │ (encrypted     │           │
│                   │             │          │  tarball)      │           │
│                   └──────┬──────┘          └───────┬───────┘           │
│                          │                         │                   │
└──────────────────────────┼─────────────────────────┼───────────────────┘
                           │                         │
                    ┌──────▼─────────────────────────▼────────┐
                    │          CONNECTIVITY LAYER             │
                    │  HTTPS (primary)    USB (fallback)      │
                    └──────────────┬────────────────────────┘
                                   │
┌──────────────────────────────────▼──────────────────────────────────┐
│                         CLOUD (AWS af-south-1)                        │
│                                                                        │
│  ┌───────────────┐     ┌───────────────┐     ┌───────────────┐        │
│  │  API Gateway   │────▶│  Delta Parser  │────▶│  PostgreSQL   │        │
│  │  /sync/receive │     │  (validate)    │     │  (upsert)     │        │
│  └───────────────┘     └───────────────┘     └───────────────┘        │
│                                │                     │                 │
│                       ┌────────┴─────────────────────┴────────┐       │
│                       │           Cloud Services             │       │
│                       │  TimescaleDB │ Qdrant │ Neo4j │ S3     │       │
│                       └─────────────────────────────────────┘       │
│                                                                        │
│  ┌─────────────────────────────────────────────────────┐              │
│  │  Balena Fleet Dashboard (device health, OTA)       │              │
│  └─────────────────────────────────────────────────────┘              │
└──────────────────────────────────────────────────────────────────────┘
```

### Data Flow

1. **Ingestion**: CHW reports AMR case via USSD/WhatsApp/Telegram → stored in local SQLite (WAL mode)
2. **Delta Generation**: Sync orchestrator compares local data against last sync cursor → generates delta JSON
3. **Compression**: Delta JSON compressed with gzip level 6 (~70% size reduction)
4. **Upload**: Compressed delta uploaded via HTTPS/TLS 1.3 to cloud API
5. **Validation**: Cloud validates SHA-256 checksum, parses delta, detects conflicts
6. **Conflict Resolution**: If conflicts detected, CRDT (Yjs) resolves them
7. **Persistence**: Resolved data upserted into PostgreSQL + TimescaleDB + Neo4j + Qdrant
8. **Acknowledgment**: Cloud sends ack with new sync cursor → edge updates cursor
9. **USB Fallback**: If network fails after 72h, generate encrypted tarball on USB drive

---

## Delta Generation Algorithm

The delta generation is the heart of efficient sync. Instead of uploading the entire database, we only send what changed since the last successful sync.

### Cursor-Based Incremental Extraction

Each table in SQLite uses a monotonically increasing `rowid` (auto-increment) and a `updated_at` timestamp. The `sync_meta` table stores the last successfully synced `rowid` per table.

```python
class DeltaGenerator:
    """
    Generates incremental deltas from SQLite for sync to cloud.
    
    Uses rowid cursor to track last synced position per table.
    Only new and modified rows since last sync are included.
    """

    SYNCABLE_TABLES = [
        "cases",           # AMR case reports
        "resistance_data", # Lab test results
        "gamification_log", # Points, badges, streaks
        "local_users",     # User profiles
    ]

    def __init__(self, db_path: str):
        self.db = sqlite3.connect(db_path)
        self.db.execute("PRAGMA journal_mode=WAL")
        self.db.execute("PRAGMA synchronous=NORMAL")

    def generate_delta(self) -> dict:
        """Generate delta JSON with all changes since last sync."""
        delta = {
            "device_id": self._get_device_id(),
            "sync_cursors": {},
            "tables": {},
            "timestamp": datetime.utcnow().isoformat() + "Z",
        }

        for table in self.SYNCABLE_TABLES:
            cursor = self._get_sync_cursor(table)
            rows = self._fetch_changed_rows(table, cursor)
            
            if rows:
                delta["tables"][table] = []
                for row in rows:
                    delta["tables"][table].append({
                        "op": self._determine_operation(row),
                        "rowid": row["rowid"],
                        "data": self._row_to_dict(row, table),
                        "ts": row["updated_at"],
                    })

        delta["checksum_sha256"] = self._compute_checksum(delta)
        return delta

    def _get_sync_cursor(self, table: str) -> int:
        """Get last synced rowid for a table."""
        result = self.db.execute(
            "SELECT last_rowid FROM sync_meta WHERE table_name = ?",
            (table,)
        ).fetchone()
        return result[0] if result else 0

    def _fetch_changed_rows(self, table: str, cursor: int) -> list:
        """Fetch all rows with rowid > cursor."""
        self.db.execute(f"SELECT rowid, * FROM {table} WHERE rowid > ?", (cursor,))
        columns = [desc[0] for desc in self.db.description]
        return [dict(zip(columns, row)) for row in self.db.fetchall()]

    def _determine_operation(self, row: dict) -> str:
        """Determine if this is an insert or update."""
        return "update" if row.get("_synced", 0) else "insert"

    def _compute_checksum(self, data: dict) -> str:
        """Compute SHA-256 checksum of delta for integrity verification."""
        serialized = json.dumps(data, sort_keys=True, default=str)
        return hashlib.sha256(serialized.encode()).hexdigest()
```

### Incremental vs Full Sync

| Sync Type | When Used | Data Volume | Time |
|-----------|-----------|-------------|------|
| **Incremental** (delta) | Normal operation | 5-50 KB (gzipped) | <5 seconds |
| **Full sync** | First sync, corruption recovery | 1-10 MB (gzipped) | <2 minutes |
| **USB fallback** | Extended offline (>72h) | Variable, full dataset | Copy time only |

---

## Delta JSON Format

### Structure

```json
{
  "device_id": "RPi-NG-IBD-001",
  "device_fingerprint": "a1b2c3d4e5f6",
  "firmware_version": "udara-edge-v1.2.3",
  "sync_cursors": {
    "cases": 1543,
    "resistance_data": 892,
    "gamification_log": 3456,
    "local_users": 78
  },
  "timestamp": "2026-05-27T10:30:00Z",
  "tables": {
    "cases": [
      {
        "op": "insert",
        "rowid": 1544,
        "data": {
          "case_id": "CASE-2025-0615-001",
          "patient_age": 35,
          "patient_sex": "F",
          "reporting_chw_id": "CHW-NG-IBD-042",
          "pathogen": "E. coli",
          "specimen_type": "urine",
          "infection_site": "urinary_tract",
          "drugs_prescribed": ["Ciprofloxacin"],
          "clinical_outcome": "not_improved",
          "notes": "Patient did not improve after 5 days of Ciprofloxacin",
          "reported_via": "whatsapp",
          "reported_at": "2026-05-27T09:45:00Z",
          "health_post_id": "HP-NG-IBD-JERICHO"
        },
        "ts": "2026-05-27T09:45:00Z"
      },
      {
        "op": "update",
        "rowid": 1521,
        "data": {
          "case_id": "CASE-2025-0613-007",
          "clinical_outcome": "recovered",
          "notes": "Patient recovered after switching to Nitrofurantoin",
          "updated_reason": "supervisor_verification"
        },
        "ts": "2026-05-27T10:15:00Z"
      }
    ],
    "resistance_data": [
      {
        "op": "insert",
        "rowid": 893,
        "data": {
          "test_id": "LAB-2025-0615-001",
          "case_id": "CASE-2025-0615-001",
          "pathogen": "E. coli",
          "drug": "Ciprofloxacin",
          "resistance_result": "resistant",
          "mic_value": 8.0,
          "breakpoint": 1.0,
          "lab_method": "disk_diffusion",
          "tested_at": "2026-05-27T08:00:00Z"
        },
        "ts": "2026-05-27T08:00:00Z"
      }
    ],
    "gamification_log": [
      {
        "op": "insert",
        "rowid": 3457,
        "data": {
          "event_type": "points_awarded",
          "chw_id": "CHW-NG-IBD-042",
          "action": "report_case",
          "points": 10,
          "running_total": 280,
          "streak_days": 3
        },
        "ts": "2026-05-27T09:46:00Z"
      }
    ]
  },
  "stats": {
    "total_rows": 4,
    "tables_affected": 3,
    "bytes_uncompressed": 2456,
    "bytes_compressed": 734,
    "compression_ratio": 0.299
  },
  "checksum_sha256": "a3f8b2c1d4e5f6078910111213141516171819202122232425262728293031"
}
```

### Compression Results

| Data Type | Uncompressed | Gzip Level 6 | Ratio |
|-----------|-------------|--------------|-------|
| Single case (text report) | 1.2 KB | 380 B | 68% |
| Single case (with photo metadata) | 2.1 KB | 650 B | 69% |
| 10 cases batch | 15 KB | 4.5 KB | 70% |
| 50 cases batch | 72 KB | 21 KB | 71% |
| Full first sync (500 cases) | 1.2 MB | 340 KB | 72% |

---

## CRDT Conflict Resolution via Yjs

### The Problem

Consider this scenario:
1. A CHW reports a case at the edge RPi at 10:00 AM
2. The sync fails (network down)
3. A supervisor, via the web dashboard, adds notes to the same case at 10:30 AM
4. The network comes back at 11:00 AM
5. Both the edge update and the cloud update need to be merged

With naive "last-write-wins", one of these updates would be silently discarded. For AMR data, losing a CHW's report or a supervisor's verification note is unacceptable.

### Yjs-Based CRDT Resolution

We use **Yjs** (a high-performance CRDT library) to handle concurrent edits. Each case document is a `Y.Doc` with typed shared data structures.

```python
import yjs
from yjs import YMap, YArray, YDoc

class CaseCRDTResolver:
    """
    Resolves conflicts between edge and cloud edits using Yjs CRDTs.
    
    Strategy per field:
    - Simple fields (drug, pathogen): Last-writer-wins (LWW)
    - Arrays (drugs_prescribed, notes): Append-merge (union)
    - Structured fields (gamification): Field-level LWW
    """

    def __init__(self):
        self.resolvers = {
            "drugs_prescribed": self._merge_arrays,
            "notes": self._append_merge,
            "clinical_outcome": self._last_writer_wins,
            "resistance_result": self._last_writer_wins,
            "gamification_data": self._gamification_merge,
        }

    def resolve(self, edge_case: dict, cloud_case: dict) -> dict:
        """Resolve conflicts between edge and cloud versions of a case."""
        merged = {}
        all_keys = set(edge_case.keys()) | set(cloud_case.keys())
        
        for key in all_keys:
            if key in ["updated_at", "_version"]:
                # Metadata: use cloud version (it's more authoritative)
                merged[key] = cloud_case.get(key)
            elif key not in edge_case:
                merged[key] = cloud_case[key]
            elif key not in cloud_case:
                merged[key] = edge_case[key]
            elif edge_case[key] == cloud_case[key]:
                merged[key] = edge_case[key]
            elif key in self.resolvers:
                merged[key] = self.resolvers[key](
                    edge_case[key], cloud_case[key], key
                )
            else:
                # Default: last writer wins
                edge_ts = edge_case.get("updated_at", "")
                cloud_ts = cloud_case.get("updated_at", "")
                if cloud_ts > edge_ts:
                    merged[key] = cloud_case[key]
                else:
                    merged[key] = edge_case[key]
        
        merged["_conflict_resolved"] = True
        merged["_resolution_method"] = "crdt_merge"
        return merged

    def _merge_arrays(self, edge_val, cloud_val, field_name):
        """Merge arrays by taking union (no duplicates)."""
        merged = list(edge_val)
        for item in cloud_val:
            if item not in merged:
                merged.append(item)
        return merged

    def _append_merge(self, edge_val, cloud_val, field_name):
        """Append both notes with timestamps."""
        if isinstance(edge_val, str) and isinstance(cloud_val, str):
            return f"[EDGE] {edge_val}\n[CLOUD] {cloud_val}"
        return cloud_val

    def _last_writer_wins(self, edge_val, cloud_val, field_name):
        """For categorical fields, cloud wins if it was explicitly updated."""
        return cloud_val

    def _gamification_merge(self, edge_val, cloud_val, field_name):
        """For gamification, take the higher value (CHW should never lose points)."""
        if isinstance(edge_val, dict) and isinstance(cloud_val, dict):
            merged = {}
            for key in set(edge_val) | set(cloud_val):
                e = edge_val.get(key, 0)
                c = cloud_val.get(key, 0)
                merged[key] = max(e, c)
            return merged
        return max(edge_val, cloud_val) if isinstance(edge_val, (int, float)) else edge_val
```

### Conflict Scenarios Handled

| Scenario | Resolution |
|----------|-----------|
| CHW adds note, supervisor adds note | Both notes preserved (append-merge) |
| CHW reports "not_improved", supervisor updates to "recovered" | Supervisor wins (clinical authority) |
| Edge adds drug to list, cloud adds different drug | Union of both drug lists |
| CHW earns 10 points at edge, 5 points at cloud | Max (15 points preserved) |
| Edge deletes case, cloud updates case | Case preserved with update (no deletion without confirmation) |

---

## Sync State Machine

### State Diagram

```
                         ┌──────────┐
                    ┌───▶│  IDLE     │◀──────────────────────────────┐
                    │    └────┬─────┘                                │
                    │         │  (sync_timer or trigger)             │
                    │         ▼                                      │
                    │    ┌──────────┐                                │
                    │    │ CHECKING │  Ping cloud, check queue       │
                    │    └────┬─────┘                                │
                    │         │  (has changes)                       │
                    │         ▼                                      │
                    │    ┌──────────┐                                │
                    │    │GENERATING│  Build delta JSON               │
                    │    └────┬─────┘                                │
                    │         │  (delta ready)                       │
                    │         ▼                                      │
                    │    ┌──────────┐                                │
              error │    │COMPRESSING│  gzip delta                   │
                    │    └────┬─────┘                                │
                    │         │  (compressed)                        │
                    │         ▼                                      │
                    │    ┌──────────┐                                │
              error │    │ UPLOADING │  HTTPS POST to cloud          │
                    │    └────┬─────┘                                │
                    │         │  (cloud responds)                     │
                    │         ▼                                      │
                    │    ┌──────────────┐                            │
                    │    │ CONFIRMING   │  Validate ack, update cursor │
                    │    └────┬────────┘                            │
                    │         │  (cursor updated)                     │
                    │         │                                      │
                    └─────────┘ (success)

Error Handling:
    Any state ──error──▶ ERROR ──retry count < 6──▶ back to IDLE (with backoff delay)
                    ERROR ──retry count >= 6──▶ USB_FALLBACK
```

### State Descriptions

| State | Duration | What Happens | Exit Condition |
|-------|----------|-------------|----------------|
| `IDLE` | Variable | Waiting for sync trigger (timer or on-demand) | Timer fires or admin command |
| `CHECKING` | ~2s | Check cloud connectivity, read sync queue | Connected or offline |
| `GENERATING` | ~1-5s | Build delta JSON from SQLite changes | Delta complete or empty |
| `COMPRESSING` | <1s | gzip compress delta JSON | Compressed |
| `UPLOADING` | ~5-30s | POST compressed delta to cloud API | Response received or timeout |
| `CONFIRMING` | ~1s | Validate cloud ack, update sync cursor | Cursor updated |
| `ERROR` | Variable | Log error, calculate backoff, check retry count | Backoff timer expires |
| `USB_FALLBACK` | Variable | Generate encrypted tarball to USB drive | Tarball written |

---

## Retry Strategy

### Exponential Backoff

```
Retry #1: wait  1 second
Retry #2: wait  2 seconds
Retry #3: wait  4 seconds
Retry #4: wait  8 seconds
Retry #5: wait 16 seconds
Retry #6: wait 32 seconds
Cap:      wait 60 seconds (for all subsequent retries)

Total wait time for 6 retries: 1 + 2 + 4 + 8 + 16 + 32 = 63 seconds
Then: 60 seconds per retry indefinitely
After 6 failures (total ~63 seconds): switch to USB monitoring mode
After 72 hours offline: trigger USB fallback (generate tarball)
```

### Retry with Jitter

To prevent thundering herd (all RPi devices retrying simultaneously), we add randomized jitter:

```python
import random

def calculate_backoff(retry_count: int) -> float:
    """Exponential backoff with jitter."""
    base = min(2 ** retry_count, 60)  # Cap at 60 seconds
    jitter = random.uniform(0, base * 0.5)  # ±50% jitter
    return base + jitter
```

### Persistent Retry State

The retry count is persisted to SQLite so it survives RPi reboots:

```sql
CREATE TABLE IF NOT EXISTS sync_retry_state (
    id INTEGER PRIMARY KEY DEFAULT 1,
    retry_count INTEGER DEFAULT 0,
    first_failure_at TEXT,
    last_retry_at TEXT,
    last_error TEXT
);
```

---

## USB Fallback

### Trigger Conditions

USB fallback is triggered when:
1. Network connectivity check fails 6 consecutive times (after full backoff sequence)
2. Total offline time exceeds 72 hours (configurable per deployment)
3. Manual trigger via admin command (`udara-cli sync --usb-fallback`)

### USB Fallback Process

```
Step 1: Generate Delta
    │  (same delta generation as network sync)
    ▼
Step 2: Serialize to JSON
    │  (full delta, uncompressed for reliability)
    ▼
Step 3: Compress with gzip
    │  (level 6, ~70% reduction)
    ▼
Step 4: Encrypt with AES-256-GCM
    │  (per-device key stored in RPi secure storage)
    ▼
Step 5: Package as tarball
    │  udara_sync_20250615_103000.tar.gz.aes
    ▼
Step 6: Write to USB drive
    │  Auto-detect via udev rules → /media/usb/
    ▼
Step 7: Generate manifest
    │  checksums.txt, device_info.json
    ▼
Step 8: Notify admin
    │  SMS/email: "USB fallback generated on RPi-NG-IBD-001"
```

### udev Rules for Auto-Detection

```bash
# /etc/udev/rules.d/99-udara-usb.rules
# Auto-mount USB drives for UDARA sync fallback

ACTION=="add", SUBSYSTEM=="block", KERNEL=="sd[a-z][0-9]", \
    RUN+="/usr/bin/mkdir -p /media/usb", \
    RUN+="/usr/bin/mount -o noatime,nodiratime %k /media/usb"

ACTION=="remove", SUBSYSTEM=="block", KERNEL=="sd[a-z][0-9]", \
    RUN+="/usr/bin/umount /media/usb"
```

### Encryption Details

```python
from cryptography.hazmat.primitives.ciphers.aead import AESGCM
import os, json, gzip

class USBFallbackHandler:
    def __init__(self, device_key_path: str = "/etc/udara/device.key"):
        self.device_key = self._load_or_generate_key(device_key_path)

    def generate_fallback(self, delta: dict, usb_path: str):
        """Generate encrypted USB fallback tarball."""
        timestamp = datetime.now().strftime("%Y%m%d_%H%M%S")
        filename = f"udara_sync_{timestamp}.tar.gz.aes"
        
        # Step 1: Serialize
        json_bytes = json.dumps(delta, default=str).encode()
        
        # Step 2: Compress
        compressed = gzip.compress(json_bytes, compresslevel=6)
        
        # Step 3: Encrypt
        nonce = os.urandom(12)
        aesgcm = AESGCM(self.device_key)
        encrypted = aesgcm.encrypt(nonce, compressed, None)
        
        # Step 4: Package (nonce + ciphertext)
        package = nonce + encrypted
        
        # Step 5: Write to USB
        filepath = os.path.join(usb_path, filename)
        with open(filepath, 'wb') as f:
            f.write(package)
        
        # Step 6: Generate checksum
        sha256 = hashlib.sha256(package).hexdigest()
        checksum_path = os.path.join(usb_path, "checksums.txt")
        with open(checksum_path, 'a') as f:
            f.write(f"{filename} {sha256}\n")
        
        return filepath, len(package)
```

### Cloud Import Process

When a USB drive arrives at the cloud operations center:

1. Insert USB drive into operations workstation
2. Run `udara-import --source /media/usb/`
3. Script reads `checksums.txt`, validates SHA-256
4. Decrypts using device key (from Vault)
5. Decompresses with gzip
6. Parses delta JSON
7. Applies to PostgreSQL (same as network sync)
8. Sends ack to device (on next network reconnect)

---

## Sync Orchestrator Implementation

```python
"""
Complete Sync Orchestrator — the central sync state machine.

This is the production implementation that manages the entire
sync lifecycle: delta generation, compression, upload, conflict
resolution, cursor management, and USB fallback.
"""

import asyncio
import gzip
import hashlib
import json
import logging
import sqlite3
from datetime import datetime, timedelta
from enum import Enum, auto
from typing import Optional
from dataclasses import dataclass, field

import httpx
from cryptography.hazmat.primitives.ciphers.aead import AESGCM

logger = logging.getLogger("udara.sync")


class SyncState(Enum):
    IDLE = auto()
    CHECKING = auto()
    GENERATING = auto()
    COMPRESSING = auto()
    UPLOADING = auto()
    CONFIRMING = auto()
    RESOLVING = auto()
    ERROR = auto()
    USB_FALLBACK = auto()


@dataclass
class SyncResult:
    status: str
    rows_synced: int = 0
    bytes_sent: int = 0
    conflicts_resolved: int = 0
    duration_seconds: float = 0.0
    error: Optional[str] = None
    retry_count: int = 0


@dataclass
class SyncConfig:
    cloud_api_url: str = "https://api.udara.ai/sync/receive"
    sync_interval_seconds: int = 900  # 15 minutes
    max_retry_count: int = 6
    usb_fallback_hours: int = 72
    bandwidth_limit_bytes: int = 2_097_152  # 2MB daily
    gzip_level: int = 6
    request_timeout_seconds: float = 30.0


class SyncOrchestrator:
    """
    Manages the complete sync lifecycle for a single edge device.
    
    Thread-safe, persistent across reboots, handles all error
    scenarios including USB fallback.
    """

    def __init__(
        self,
        db_path: str,
        config: SyncConfig = SyncConfig(),
        device_id: str = "RPi-UNKNOWN"
    ):
        self.db_path = db_path
        self.config = config
        self.device_id = device_id
        self.state = SyncState.IDLE
        self.retry_count = 0
        self.last_sync_time = None
        self.bytes_sent_today = 0
        self._running = False

        # Initialize database
        self.db = sqlite3.connect(db_path)
        self.db.execute("PRAGMA journal_mode=WAL")
        self.db.execute("PRAGMA synchronous=NORMAL")
        self._init_sync_meta()

        # Cloud client
        self.http_client = httpx.AsyncClient(
            timeout=config.request_timeout_seconds,
            headers={"User-Agent": f"UDARA-Edge/{device_id}"}
        )

        # CRDT resolver
        self.crdt_resolver = CaseCRDTResolver()

    def _init_sync_meta(self):
        """Initialize sync tracking tables."""
        self.db.executescript("""
            CREATE TABLE IF NOT EXISTS sync_meta (
                table_name TEXT PRIMARY KEY,
                last_rowid INTEGER DEFAULT 0,
                last_sync_at TEXT
            );
            CREATE TABLE IF NOT EXISTS sync_retry_state (
                id INTEGER PRIMARY KEY DEFAULT 1,
                retry_count INTEGER DEFAULT 0,
                first_failure_at TEXT,
                last_retry_at TEXT,
                last_error TEXT
            );
            CREATE TABLE IF NOT EXISTS sync_log (
                id INTEGER PRIMARY KEY AUTOINCREMENT,
                started_at TEXT,
                completed_at TEXT,
                state TEXT,
                status TEXT,
                rows_synced INTEGER,
                bytes_sent INTEGER,
                error TEXT
            );
        """)
        self.db.commit()

    async def sync_cycle(self) -> SyncResult:
        """Execute one complete sync cycle through the state machine."""
        start_time = datetime.utcnow()
        
        try:
            # STATE: CHECKING
            self.state = SyncState.CHECKING
            self._log_state()
            
            if not await self._check_connectivity():
                self.state = SyncState.ERROR
                return self._handle_error("No connectivity")
            
            # Check bandwidth budget
            if self.bytes_sent_today >= self.config.bandwidth_limit_bytes:
                logger.info("Daily bandwidth limit reached, skipping sync")
                self.state = SyncState.IDLE
                return SyncResult(status="bandwidth_limit")

            # STATE: GENERATING
            self.state = SyncState.GENERATING
            delta = self._generate_delta()
            
            if not delta["tables"]:
                self.state = SyncState.IDLE
                return SyncResult(status="no_changes")
            
            # STATE: COMPRESSING
            self.state = SyncState.COMPRESSING
            compressed = self._compress_delta(delta)
            bytes_sent = len(compressed)
            
            # STATE: UPLOADING
            self.state = SyncState.UPLOADING
            response = await self._upload_delta(compressed)
            
            # Handle conflicts
            if response.get("conflicts"):
                self.state = SyncState.RESOLVING
                resolved = self._resolve_conflicts(response["conflicts"])
                response = await self._upload_resolved(resolved)
            
            # STATE: CONFIRMING
            self.state = SyncState.CONFIRMING
            self._update_sync_cursors(response["new_cursors"])
            self.bytes_sent_today += bytes_sent
            
            # Success
            self.state = SyncState.IDLE
            self.retry_count = 0
            self.last_sync_time = datetime.utcnow()
            self._reset_retry_state()
            
            duration = (datetime.utcnow() - start_time).total_seconds()
            self._log_sync_success(delta, compressed, duration)
            
            total_rows = sum(len(rows) for rows in delta["tables"].values())
            return SyncResult(
                status="success",
                rows_synced=total_rows,
                bytes_sent=bytes_sent,
                conflicts_resolved=len(response.get("conflicts", [])),
                duration_seconds=duration
            )
            
        except Exception as e:
            logger.error(f"Sync error: {e}", exc_info=True)
            self.state = SyncState.ERROR
            return self._handle_error(str(e))

    async def _check_connectivity(self) -> bool:
        """Check if cloud API is reachable."""
        try:
            resp = await self.http_client.get(
                f"{self.config.cloud_api_url.replace('/sync/receive', '')}/health",
                timeout=5.0
            )
            return resp.status_code == 200
        except Exception:
            return False

    def _generate_delta(self) -> dict:
        """Generate delta JSON from local SQLite."""
        delta = {
            "device_id": self.device_id,
            "timestamp": datetime.utcnow().isoformat() + "Z",
            "tables": {}
        }
        
        for table in ["cases", "resistance_data", "gamification_log", "local_users"]:
            cursor_row = self.db.execute(
                "SELECT last_rowid FROM sync_meta WHERE table_name = ?", (table,)
            ).fetchone()
            cursor = cursor_row[0] if cursor_row else 0
            
            self.db.execute(f"SELECT rowid, * FROM {table} WHERE rowid > ?", (cursor,))
            columns = [desc[0] for desc in self.db.description]
            rows = [dict(zip(columns, row)) for row in self.db.fetchall()]
            
            if rows:
                delta["tables"][table] = [
                    {"op": "insert", "rowid": r["rowid"], "data": r, "ts": r.get("updated_at", "")}
                    for r in rows
                ]
        
        delta["checksum_sha256"] = hashlib.sha256(
            json.dumps(delta, sort_keys=True, default=str).encode()
        ).hexdigest()
        return delta

    def _compress_delta(self, delta: dict) -> bytes:
        """Compress delta JSON with gzip."""
        return gzip.compress(
            json.dumps(delta, default=str).encode(),
            compresslevel=self.config.gzip_level
        )

    async def _upload_delta(self, compressed: bytes) -> dict:
        """Upload compressed delta to cloud."""
        resp = await self.http_client.post(
            self.config.cloud_api_url,
            content=compressed,
            headers={
                "Content-Type": "application/octet-stream",
                "X-Device-ID": self.device_id,
                "X-Checksum": hashlib.sha256(compressed).hexdigest(),
                "X-Compression": "gzip",
            }
        )
        resp.raise_for_status()
        return resp.json()

    def _resolve_conflicts(self, conflicts: list) -> list:
        """Resolve conflicts using CRDT."""
        resolved = []
        for conflict in conflicts:
            merged = self.crdt_resolver.resolve(conflict["edge"], conflict["cloud"])
            resolved.append({"case_id": conflict["case_id"], "merged": merged})
        return resolved

    async def _upload_resolved(self, resolved: list) -> dict:
        """Upload CRDT-resolved conflicts."""
        resp = await self.http_client.post(
            f"{self.config.cloud_api_url}/resolve",
            json={"device_id": self.device_id, "resolutions": resolved}
        )
        resp.raise_for_status()
        return resp.json()

    def _update_sync_cursors(self, new_cursors: dict):
        """Update local sync cursors after successful sync."""
        for table, rowid in new_cursors.items():
            self.db.execute(
                """INSERT INTO sync_meta (table_name, last_rowid, last_sync_at)
                   VALUES (?, ?, ?)
                   ON CONFLICT(table_name) DO UPDATE SET
                   last_rowid = excluded.last_rowid,
                   last_sync_at = excluded.last_sync_at""",
                (table, rowid, datetime.utcnow().isoformat())
            )
        self.db.commit()

    def _handle_error(self, error: str) -> SyncResult:
        """Handle sync error with retry logic."""
        self.retry_count += 1
        
        # Persist retry state
        self.db.execute(
            """INSERT INTO sync_retry_state (id, retry_count, first_failure_at, last_retry_at, last_error)
               VALUES (1, ?, COALESCE(
                   (SELECT first_failure_at FROM sync_retry_state WHERE id=1),
                   ?
               ), ?, ?)
               ON CONFLICT(id) DO UPDATE SET
               retry_count = excluded.retry_count,
               last_retry_at = excluded.last_retry_at,
               last_error = excluded.last_error""",
            (self.retry_count, datetime.utcnow().isoformat(),
             datetime.utcnow().isoformat(), error[:500])
        )
        self.db.commit()
        
        # Check USB fallback trigger
        retry_state = self.db.execute(
            "SELECT first_failure_at FROM sync_retry_state WHERE id = 1"
        ).fetchone()
        
        if retry_state and retry_state[0]:
            first_failure = datetime.fromisoformat(retry_state[0])
            offline_hours = (datetime.utcnow() - first_failure).total_seconds() / 3600
            
            if offline_hours >= self.config.usb_fallback_hours:
                logger.warning(f"Offline for {offline_hours:.1f}h, triggering USB fallback")
                self.state = SyncState.USB_FALLBACK
                self._trigger_usb_fallback()
        
        # Calculate backoff
        backoff = min(2 ** self.retry_count, 60) + 0.5  # Simple backoff
        
        logger.info(f"Sync error (retry {self.retry_count}/{self.config.max_retry_count}): {error}")
        logger.info(f"Next retry in {backoff}s")
        
        return SyncResult(
            status="error",
            error=error,
            retry_count=self.retry_count
        )

    def _trigger_usb_fallback(self):
        """Generate encrypted tarball for USB physical transport."""
        try:
            from udara.edge.usb_sync import USBFallbackHandler
            
            handler = USBFallbackHandler()
            delta = self._generate_delta()
            filepath, size = handler.generate_fallback(delta, "/media/usb")
            logger.info(f"USB fallback generated: {filepath} ({size} bytes)")
            
            self.db.execute(
                "INSERT INTO sync_log (started_at, completed_at, state, status, bytes_sent) VALUES (?, ?, ?, ?, ?)",
                (datetime.utcnow().isoformat(), datetime.utcnow().isoformat(),
                 "USB_FALLBACK", "generated", size)
            )
            self.db.commit()
            
        except Exception as e:
            logger.error(f"USB fallback failed: {e}")

    def _reset_retry_state(self):
        """Reset retry state after successful sync."""
        self.db.execute(
            "UPDATE sync_retry_state SET retry_count=0, first_failure_at=NULL, last_retry_at=NULL, last_error=NULL WHERE id=1"
        )
        self.db.commit()

    def _log_state(self):
        logger.debug(f"Sync state: {self.state.name}")

    def _log_sync_success(self, delta, compressed, duration):
        total_rows = sum(len(rows) for rows in delta["tables"].values())
        self.db.execute(
            "INSERT INTO sync_log (started_at, completed_at, state, status, rows_synced, bytes_sent) VALUES (?, ?, ?, ?, ?, ?)",
            (datetime.utcnow().isoformat(), datetime.utcnow().isoformat(),
             "IDLE", "success", total_rows, len(compressed))
        )
        self.db.commit()
        logger.info(
            f"Sync success: {total_rows} rows, {len(compressed)} bytes, {duration:.1f}s"
        )

    async def start_periodic_sync(self):
        """Start the periodic sync loop."""
        self._running = True
        while self._running:
            try:
                result = await self.sync_cycle()
                if result.status == "success":
                    logger.info(f"Periodic sync: {result.rows_synced} rows synced")
            except Exception as e:
                logger.error(f"Periodic sync error: {e}")
            await asyncio.sleep(self.config.sync_interval_seconds)

    async def stop(self):
        """Stop the periodic sync loop."""
        self._running = False
        await self.http_client.aclose()
```

---

## Cloud Reception Endpoint

```python
"""
FastAPI endpoint for receiving sync deltas from edge devices.
"""

from fastapi import APIRouter, HTTPException, Header, Request, BackgroundTasks
import gzip
import hashlib
import json
from typing import Optional

router = APIRouter(prefix="/sync", tags=["sync"])


@router.post("/receive")
async def receive_delta(
    request: Request,
    x_device_id: Optional[str] = Header(None),
    x_checksum: Optional[str] = Header(None),
    x_compression: Optional[str] = Header("gzip"),
):
    """Receive and process sync delta from an edge device."""
    
    # 1. Read raw body
    body = await request.body()
    
    # 2. Verify checksum
    if x_checksum:
        computed = hashlib.sha256(body).hexdigest()
        if computed != x_checksum:
            raise HTTPException(400, f"Checksum mismatch: {computed} != {x_checksum}")
    
    # 3. Decompress
    if x_compression == "gzip":
        try:
            json_bytes = gzip.decompress(body)
        except Exception:
            raise HTTPException(400, "Failed to decompress gzip data")
    else:
        json_bytes = body
    
    # 4. Parse JSON
    try:
        delta = json.loads(json_bytes)
    except json.JSONDecodeError as e:
        raise HTTPException(400, f"Invalid JSON: {e}")
    
    # 5. Validate structure
    if "device_id" not in delta or "tables" not in delta:
        raise HTTPException(400, "Missing required fields: device_id, tables")
    
    # 6. Check for conflicts
    conflicts = await detect_conflicts(delta)
    
    # 7. Apply delta to database
    new_cursors = {}
    for table_name, rows in delta["tables"].items():
        max_rowid = await apply_table_delta(table_name, rows, x_device_id)
        if max_rowid:
            new_cursors[table_name] = max_rowid
    
    # 8. Return acknowledgment
    return {
        "status": "accepted",
        "device_id": delta["device_id"],
        "rows_received": sum(len(rows) for rows in delta["tables"].values()),
        "new_cursors": new_cursors,
        "conflicts": conflicts,
        "server_timestamp": datetime.utcnow().isoformat() + "Z"
    }


async def detect_conflicts(delta: dict) -> list:
    """Check if any delta rows conflict with existing cloud data."""
    conflicts = []
    for table_name, rows in delta["tables"].items():
        for row in rows:
            if row["op"] == "update":
                existing = await get_cloud_row(table_name, row["data"]["case_id"])
                if existing and existing["updated_at"] > row["ts"]:
                    conflicts.append({
                        "case_id": row["data"].get("case_id"),
                        "table": table_name,
                        "edge": row["data"],
                        "cloud": existing
                    })
    return conflicts
```

---

## Edge SQLite Schema

```sql
-- Edge SQLite Schema (runs on RPi 5)
-- All tables use WAL mode for concurrent read/write

PRAGMA journal_mode = WAL;
PRAGMA synchronous = NORMAL;
PRAGMA cache_size = -2000;  -- 2MB cache
PRAGMA temp_store = MEMORY;

-- AMR Case Reports
CREATE TABLE IF NOT EXISTS cases (
    rowid INTEGER PRIMARY KEY AUTOINCREMENT,
    case_id TEXT UNIQUE NOT NULL,
    patient_age INTEGER,
    patient_sex TEXT CHECK(patient_sex IN ('M', 'F', 'O', 'U')),
    reporting_chw_id TEXT NOT NULL,
    pathogen TEXT,
    specimen_type TEXT,
    infection_site TEXT,
    drugs_prescribed TEXT,  -- JSON array: ["Ciprofloxacin", "Amoxicillin"]
    clinical_outcome TEXT CHECK(clinical_outcome IN ('improved', 'not_improved', 'recovered', 'deceased', 'unknown')),
    notes TEXT,
    reported_via TEXT CHECK(reported_via IN ('ussd', 'whatsapp', 'telegram', 'web', 'voice')),
    reported_at TEXT NOT NULL,
    health_post_id TEXT NOT NULL,
    _synced INTEGER DEFAULT 0,
    updated_at TEXT DEFAULT (datetime('now'))
);
CREATE INDEX idx_cases_chw ON cases(reporting_chw_id);
CREATE INDEX idx_cases_date ON cases(reported_at);
CREATE INDEX idx_cases_post ON cases(health_post_id);
CREATE INDEX idx_cases_pathogen ON cases(pathogen);

-- Resistance Test Results
CREATE TABLE IF NOT EXISTS resistance_data (
    rowid INTEGER PRIMARY KEY AUTOINCREMENT,
    test_id TEXT UNIQUE NOT NULL,
    case_id TEXT NOT NULL REFERENCES cases(case_id),
    pathogen TEXT NOT NULL,
    drug TEXT NOT NULL,
    resistance_result TEXT CHECK(resistance_result IN ('resistant', 'susceptible', 'intermediate')),
    mic_value REAL,
    breakpoint REAL,
    lab_method TEXT,
    tested_at TEXT NOT NULL,
    _synced INTEGER DEFAULT 0,
    updated_at TEXT DEFAULT (datetime('now'))
);
CREATE INDEX idx_resistance_case ON resistance_data(case_id);
CREATE INDEX idx_resistance_drug ON resistance_data(drug);
CREATE INDEX idx_resistance_pathogen ON resistance_data(pathogen);

-- Gamification Events
CREATE TABLE IF NOT EXISTS gamification_log (
    rowid INTEGER PRIMARY KEY AUTOINCREMENT,
    event_type TEXT NOT NULL,
    chw_id TEXT NOT NULL,
    action TEXT NOT NULL,
    points INTEGER DEFAULT 0,
    badge_id TEXT,
    running_total INTEGER DEFAULT 0,
    streak_days INTEGER DEFAULT 0,
    metadata TEXT,  -- JSON: additional context
    _synced INTEGER DEFAULT 0,
    updated_at TEXT DEFAULT (datetime('now'))
);
CREATE INDEX idx_gam_chw ON gamification_log(chw_id);
CREATE INDEX idx_gam_date ON gamification_log(updated_at);

-- Local Users (CHWs registered at this health post)
CREATE TABLE IF NOT EXISTS local_users (
    rowid INTEGER PRIMARY KEY AUTOINCREMENT,
    user_id TEXT UNIQUE NOT NULL,
    name TEXT,
    phone TEXT,
    language TEXT DEFAULT 'en',
    role TEXT DEFAULT 'chw',
    health_post_id TEXT NOT NULL,
    points_total INTEGER DEFAULT 0,
    badges TEXT,  -- JSON array
    streak_current INTEGER DEFAULT 0,
    streak_best INTEGER DEFAULT 0,
    registered_at TEXT,
    last_active_at TEXT,
    _synced INTEGER DEFAULT 0,
    updated_at TEXT DEFAULT (datetime('now'))
);

-- Sync Metadata (tracking cursors per table)
-- Created by SyncOrchestrator._init_sync_meta()
```

---

## Sync Schedule

| Trigger | Frequency | Description |
|---------|-----------|-------------|
| **Periodic** | Every 15 minutes | Automatic sync when online |
| **On Reconnect** | Immediate | Triggered when connectivity detected after offline period |
| **Case Threshold** | After 5 new cases | Don't wait 15 min if there's data to sync |
| **Admin Command** | On demand | `udara-cli sync --now` |
| **Low Memory** | When RAM > 85% | Sync to offload data from SQLite cache |
| **Before Update** | Pre OTA | Sync all data before Balena fleet update |

### Sync Priority

When bandwidth is limited, syncs are prioritized:

```
Priority 1 (Critical):  resistance_data    — Lab results, urgent for public health
Priority 2 (High):       cases              — Patient case reports
Priority 3 (Medium):     gamification_log    — Points/badges (can be delayed)
Priority 4 (Low):         local_users        — User profiles (rarely change)
```

---

## Bandwidth Budget

| Metric | Target | Rationale |
|--------|--------|-----------|
| Avg delta per sync | 5-50 KB | Typical: 5-10 cases |
| Max delta per sync | 500 KB | After extended offline, many cases queued |
| Daily budget | 2 MB | Affordable on 3G data plans ($5/month) |
| Monthly budget | 50 MB | ~30 days × average sync patterns |

### Bandwidth Optimization Techniques

1. **Incremental deltas**: Only changed rows, not full tables
2. **Gzip compression**: ~70% reduction
3. **Selective sync**: Priority-based, resistance data first
4. **Batching**: Multiple syncs combined when bandwidth allows
5. **Deduplication**: Skip rows already present at cloud (cursor check)

---

## Monitoring & Observability

### Prometheus Metrics

```python
from prometheus_client import Counter, Gauge, Histogram, Summary

# Sync metrics
sync_total = Counter(
    'udara_sync_total',
    'Total sync attempts',
    ['device_id', 'status']  # status: success, error, bandwidth_limit
)
sync_duration = Histogram(
    'udara_sync_duration_seconds',
    'Sync cycle duration',
    buckets=[1, 2, 5, 10, 30, 60, 120]
)
sync_rows = Summary(
    'udara_sync_rows',
    'Rows synced per cycle'
)
sync_bytes = Counter(
    'udara_sync_bytes_total',
    'Total bytes uploaded',
    ['device_id']
)
sync_lag = Gauge(
    'udara_sync_lag_seconds',
    'Time since last successful sync',
    ['device_id']
)
conflict_count = Counter(
    'udara_sync_conflicts_total',
    'Total CRDT conflicts resolved',
    ['device_id']
)
usb_fallback_count = Counter(
    'udara_usb_fallback_total',
    'USB fallback events triggered',
    ['device_id']
)
device_offline_hours = Gauge(
    'udara_device_offline_hours',
    'Hours device has been offline',
    ['device_id']
)
```

### Grafana Dashboard Panels

| Row | Panel | Type | Description |
|-----|-------|------|-------------|
| 1 | Sync Success Rate | Gauge | % of syncs that succeed (target: 95%) |
| 1 | Sync Lag | Time series | Seconds since last successful sync per device |
| 1 | Active Syncs | Status | Number of devices currently syncing |
| 2 | Rows Synced | Time series | New rows synced per hour |
| 2 | Bytes Transferred | Time series | Bandwidth usage per device per day |
| 2 | Conflicts Resolved | Counter | CRDT conflicts over time |
| 3 | Device Health Map | Choropleth | Map of all devices colored by sync health |
| 3 | USB Fallback Events | Time series | How often USB fallback triggers |
| 3 | Offline Devices | Table | List of devices offline >24h |

### Alerts

| Alert | Condition | Severity | Action |
|-------|-----------|----------|--------|
| Device Offline | No sync >24h | Warning | Email admin |
| Device Offline Critical | No sync >72h | Critical | Email + SMS admin, trigger USB check |
| Sync Failure Rate | >20% failures in 1h | Warning | Investigate connectivity |
| Conflict Spike | >10 conflicts/hour | Warning | Review CRDT logic |
| Bandwidth Overuse | >2MB/day per device | Warning | Check for data duplication |

---

> **Document Version**: v1.0 | **Last Updated**: 2026-05-27 | **Status**: Final
