# 07 — Database Schemas

> **UDARA AI** — AMR Surveillance Platform for Sub-Saharan Africa  
> Document Version: 2.1.0 | Last Updated: 2026-05-27 | Status: Production

---

## Table of Contents

1. [Architecture Overview](#1-architecture-overview)
2. [Edge SQLite: Configuration & WAL Mode](#2-edge-sqlite-configuration--wal-mode)
3. [Edge SQLite: Complete Schema (6 Tables)](#3-edge-sqlite-complete-schema-6-tables)
4. [Cloud PostgreSQL+TimescaleDB+PostGIS](#4-cloud-postgresqltimescaledbpostgis)
5. [ASCII ER Diagram](#5-ascii-er-diagram)
6. [Data Type Notes](#6-data-type-notes)
7. [Migration Strategy](#7-migration-strategy)
8. [Seed Data & Test Fixtures](#8-seed-data--test-fixtures)
9. [Query Patterns & Performance](#9-query-patterns--performance)
10. [Backup & Recovery](#10-backup--recovery)

---

## 1. Architecture Overview

UDARA uses a **dual-database architecture**: SQLite on edge devices (Raspberry Pi)
and PostgreSQL+TimescaleDB+PostGIS in the cloud. This design ensures the edge can
operate fully offline while the cloud provides time-series analytics, geospatial
queries, and multi-facility aggregation.

```
┌─────────────────────────────────────────────────────────────────────┐
│                    UDARA DATABASE ARCHITECTURE                       │
│                                                                      │
│   EDGE (Raspberry Pi)                    CLOUD (AWS/GCP)             │
│   ───────────────────                    ───────────────             │
│                                                                      │
│   ┌──────────────────────┐              ┌────────────────────────┐   │
│   │     SQLite 3.x       │              │  PostgreSQL 16 +       │   │
│   │     WAL mode         │   sync       │  TimescaleDB 2.x      │   │
│   │                      │ ─────────>   │  + PostGIS 3.x         │   │
│   │  Tables:             │              │                        │   │
│   │  • cases             │              │  Tables:               │   │
│   │  • resistance_est.   │              │  • national_cases      │   │
│   │  • sync_queue        │              │    (hypertable)        │   │
│   │  • chw_profiles      │              │  • resistance_weekly   │   │
│   │  • alert_history     │              │    (materialized view) │   │
│   │  • ussd_sessions     │              │  • devices             │   │
│   │                      │              │  • sync_audit_log      │   │
│   │  File: edge.db       │              │  • raw_ingestion_log   │   │
│   │  Size: ~50-500 MB    │              │                        │   │
│   └──────────────────────┘              │  File: Managed RDS     │
│                                         │  Size: ~10-100 GB      │   │
│                                         └────────────────────────┘   │
│                                                                      │
│   Characteristics:                      Characteristics:             │
│   • Zero-config embedded                 • Time-series optimized     │
│   • Single-writer, many-reader           • Geospatial queries        │
│   • ACID compliant                      • Multi-tenant isolation     │
│   • Survives power loss (WAL)           • Horizontal read replicas   │
│   • No network required                 • Point-in-time recovery    │
└─────────────────────────────────────────────────────────────────────┘
```

---

## 2. Edge SQLite: Configuration & WAL Mode

### 2.1 SQLite PRAGMA Configuration

The following PRAGMAs are set on every database connection. They are critical
for reliability on Raspberry Pi devices that may experience unexpected power
loss or SD card corruption.

```python
# shared/database/edge_connection.py

import sqlite3
from pathlib import Path
from typing import Optional
import logging

logger = logging.getLogger("udara.db.edge")

EDGE_DB_PRAGMAS = {
    # ── Journal Mode ──────────────────────────────────────────────
    # WAL mode allows concurrent reads while a write is in progress.
    # Crucial for the sync daemon writing while agents read.
    # WAL mode is MUCH more resilient to power loss than DELETE/JOURNAL modes.
    # The -wal and -shm files are created alongside edge.db.
    "journal_mode": "WAL",

    # ── Synchronous ──────────────────────────────────────────────
    # NORMAL = flush to OS buffer cache, not disk. Much faster than
    # FULL on SD cards while still being crash-safe (WAL guarantees).
    # FULL would be safer but 5-10x slower on Class 10 SD cards.
    "synchronous": "NORMAL",

    # ── Foreign Keys ─────────────────────────────────────────────
    # Enable FK constraint enforcement. SQLite disables this by default.
    "foreign_keys": "ON",

    # ── Busy Timeout ─────────────────────────────────────────────
    # If a write transaction is blocked by a reader (WAL allows this
    # but the writer still needs exclusive lock briefly), wait up to
    # 30 seconds before raising SQLITE_BUSY. On RPi with concurrent
    # sync + agent access, brief contention is normal.
    "busy_timeout": "30000",

    # ── Cache Size ───────────────────────────────────────────────
    # 64MB page cache (in negative KB). Default is ~2MB.
    # RPi 4B has 4GB RAM; 64MB is a reasonable trade-off.
    # More cache = fewer disk reads = faster queries.
    "cache_size": "-65536",

    # ── Page Size ────────────────────────────────────────────────
    # 4096 bytes matches most SD card sector sizes and ARM64 page size.
    # Reduces wasted space compared to default 1024.
    "page_size": "4096",

    # ── Auto-Vacuum ──────────────────────────────────────────────
    # INCREMENTAL reclaims freed pages when explicitly vacuumed.
    # AUTO_VACUUM=FULL would be automatic but causes write amplification.
    "auto_vacuum": "INCREMENTAL",

    # ── Temp Store ───────────────────────────────────────────────
    # MEMORY for temp tables and indices. Faster than disk.
    "temp_store": "MEMORY",

    # ── Mmap Size ────────────────────────────────────────────────
    # Memory-mapped I/O for reads. 256MB max.
    # Avoids syscall overhead for large sequential reads.
    "mmap_size": "268435456",

    # ── WAL Auto-Checkpoint ──────────────────────────────────────
    # Not a PRAGMA but a configuration: auto-checkpoint every 1000 pages
    # (default). We control this via wal_autocheckpoint.
    "wal_autocheckpoint": "1000",
}


def get_edge_connection(db_path: str = "/var/lib/udara/edge.db") -> sqlite3.Connection:
    """
    Create a configured SQLite connection for the edge database.

    This function MUST be called for every new connection.
    Do NOT share connections across threads.
    """
    db_file = Path(db_path)
    db_file.parent.mkdir(parents=True, exist_ok=True)

    conn = sqlite3.connect(
        str(db_file),
        timeout=30.0,
        isolation_level=None,  # Autocommit mode; we manage transactions explicitly
        check_same_thread=False,  # Allow sharing across threads (use with care)
    )
    conn.row_factory = sqlite3.Row

    # Apply all PRAGMAs
    for pragma, value in EDGE_DB_PRAGMAS.items():
        conn.execute(f"PRAGMA {pragma} = {value}")
        logger.debug("Set PRAGMA %s = %s", pragma, value)

    logger.info("Edge database connection established: %s", db_path)
    return conn
```

### 2.2 Auto-Migration System

```python
# shared/database/migrations.py

import sqlite3
import logging
from pathlib import Path
from typing import List

logger = logging.getLogger("udara.db.migrations")

MIGRATIONS_DIR = Path(__file__).parent.parent.parent / "migrations" / "edge"


class EdgeMigrationRunner:
    """
    Automatic migration system for edge SQLite database.
    Migrations are numbered sequentially: 001_initial_schema.sql, 002_..., etc.
    Only forward migrations are supported (no rollback on edge).
    """

    def __init__(self, conn: sqlite3.Connection):
        self.conn = conn

    def ensure_migration_table(self):
        """Create the migration tracking table if it doesn't exist."""
        self.conn.execute("""
            CREATE TABLE IF NOT EXISTS _schema_migrations (
                version     INTEGER PRIMARY KEY,
                name       TEXT    NOT NULL,
                applied_at TEXT    NOT NULL DEFAULT (datetime('now')),
                checksum   TEXT
            )
        """)

    def get_current_version(self) -> int:
        """Return the highest applied migration version number."""
        row = self.conn.execute(
            "SELECT COALESCE(MAX(version), 0) FROM _schema_migrations"
        ).fetchone()
        return row[0]

    def get_pending_migrations(self) -> List[Path]:
        """List migration files that have not yet been applied."""
        current = self.get_current_version()
        pending = []

        if not MIGRATIONS_DIR.exists():
            logger.warning("Migrations directory not found: %s", MIGRATIONS_DIR)
            return pending

        for migration_file in sorted(MIGRATIONS_DIR.glob("*.sql")):
            # Extract version number from filename: "001_initial_schema.sql" → 1
            try:
                version = int(migration_file.stem.split("_")[0])
            except (ValueError, IndexError):
                logger.warning("Skipping malformed migration file: %s", migration_file.name)
                continue

            if version > current:
                pending.append(migration_file)

        return pending

    def run_migrations(self) -> int:
        """
        Run all pending migrations in order.

        Returns:
            Number of migrations applied.
        """
        self.ensure_migration_table()

        pending = self.get_pending_migrations()
        if not pending:
            logger.info("Database is up to date (version %d)", self.get_current_version())
            return 0

        applied = 0
        for migration_file in pending:
            version = int(migration_file.stem.split("_")[0])
            sql = migration_file.read_text(encoding="utf-8")

            logger.info(
                "Applying migration %d: %s",
                version, migration_file.name,
            )

            try:
                # Execute within a transaction
                self.conn.execute("BEGIN")
                self.conn.executescript(sql)

                # Record the migration
                import hashlib
                checksum = hashlib.sha256(sql.encode("utf-8")).hexdigest()[:16]
                self.conn.execute(
                    "INSERT INTO _schema_migrations (version, name, checksum) VALUES (?, ?, ?)",
                    (version, migration_file.name, checksum),
                )
                self.conn.execute("COMMIT")

                applied += 1
                logger.info("Migration %d applied successfully", version)

            except Exception as e:
                self.conn.execute("ROLLBACK")
                logger.error("Migration %d FAILED: %s", version, e)
                raise RuntimeError(
                    f"Migration {version} ({migration_file.name}) failed: {e}"
                ) from e

        logger.info(
            "All migrations complete: %d applied, now at version %d",
            applied, self.get_current_version(),
        )
        return applied


def migrate_edge_db(conn: sqlite3.Connection) -> None:
    """Convenience function: run migrations on a connection."""
    runner = EdgeMigrationRunner(conn)
    runner.run_migrations()
```

---

## 3. Edge SQLite: Complete Schema (6 Tables)

### 3.1 Migration File: `001_initial_schema.sql`

```sql
-- ============================================================================
-- 001_initial_schema.sql
-- UDARA Edge Database — Initial Schema
-- Target: SQLite 3.x on Raspberry Pi 4B
-- ============================================================================

-- ── 1. CLINICAL CASES ───────────────────────────────────────────────────────
-- Core table: every clinical encounter extracted by Agent A.

CREATE TABLE IF NOT EXISTS cases (
    -- Identity
    id                  TEXT PRIMARY KEY,              -- UUID v4
    facility_code       TEXT NOT NULL,                  -- e.g., "KE-NRB-FAC-017"
    district            TEXT NOT NULL,                  -- e.g., "Nairobi"
    county              TEXT,                           -- e.g., "Nairobi"
    country             TEXT NOT NULL DEFAULT 'KEN',    -- ISO 3166-1 alpha-3

    -- Patient demographics (ANONYMIZED — no PII)
    patient_age_years   INTEGER,                        -- Integer age
    patient_age_group   TEXT,                           -- "25-34", "1-4", "65+", etc.
    patient_sex         TEXT CHECK(patient_sex IN ('M','F','O','U')),

    -- Encounter metadata
    encounter_date      TEXT NOT NULL,                  -- ISO 8601 date: "2026-05-27"
    source_type         TEXT NOT NULL DEFAULT 'clinical_note',
                                                 CHECK(source_type IN (
                                                    'clinical_note', 'voice',
                                                    'image', 'ussd', 'multimodal'
                                                 )),
    language            TEXT DEFAULT 'en',              -- 'en', 'sw', 'am', 'fr'

    -- Extracted clinical data (JSON — flexible schema for varied encounters)
    symptoms            TEXT,                           -- JSON array of symptoms
    diagnosis_suggestion TEXT,                          -- AI-suggested diagnosis
    drugs_prescribed    TEXT,                           -- JSON array of drugs
    clinical_notes      TEXT,                           -- Free-text notes (anonymized)

    -- AI confidence and metadata
    amr_risk_score      REAL CHECK(amr_risk_score >= 0 AND amr_risk_score <= 1),
    extraction_confidence REAL CHECK(extraction_confidence >= 0 AND extraction_confidence <= 1),
    extracted_by        TEXT,                           -- Agent version: "agent-a-v2.3.1"

    -- Location (anonymized)
    geohash             TEXT,                           -- 5-char precision ≈ 5km

    -- Sync metadata
    synced              INTEGER NOT NULL DEFAULT 0,     -- 0=pending, 1=synced
    synced_at           TEXT,                           -- ISO 8601 datetime
    sync_batch_id       TEXT,                           -- UUID of sync batch

    -- Record metadata
    created_at          TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at          TEXT NOT NULL DEFAULT (datetime('now'))
);

-- Indexes for common query patterns
CREATE INDEX IF NOT EXISTS idx_cases_facility ON cases(facility_code);
CREATE INDEX IF NOT EXISTS idx_cases_district ON cases(district);
CREATE INDEX IF NOT EXISTS idx_cases_county ON cases(county);
CREATE INDEX IF NOT EXISTS idx_cases_encounter_date ON cases(encounter_date);
CREATE INDEX IF NOT EXISTS idx_cases_synced ON cases(synced);
CREATE INDEX IF NOT EXISTS idx_cases_created_at ON cases(created_at);
CREATE INDEX IF NOT EXISTS idx_cases_diagnosis ON cases(diagnosis_suggestion);
CREATE INDEX IF NOT EXISTS idx_cases_amr_risk ON cases(amr_risk_score)
    WHERE amr_risk_score IS NOT NULL;

-- Composite index for the most common query: "unsynced cases for a facility"
CREATE INDEX IF NOT EXISTS idx_cases_unsynced_facility ON cases(facility_code, synced)
    WHERE synced = 0;

-- Geohash prefix index for proximity queries
CREATE INDEX IF NOT EXISTS idx_cases_geohash ON cases(geohash)
    WHERE geohash IS NOT NULL;

-- ── 2. RESISTANCE ESTIMATES ─────────────────────────────────────────────────
-- Computed by Agent B: drug resistance rates by district and time period.

CREATE TABLE IF NOT EXISTS resistance_estimates (
    id                  INTEGER PRIMARY KEY AUTOINCREMENT,

    -- Dimensions
    drug                TEXT NOT NULL,                  -- "ciprofloxacin"
    district            TEXT NOT NULL,
    county              TEXT,
    country             TEXT NOT NULL DEFAULT 'KEN',

    -- Time period (ISO week)
    period_start        TEXT NOT NULL,                  -- "2025-01-06" (Monday)
    period_end          TEXT NOT NULL,                  -- "2025-01-12" (Sunday)
    iso_year_week       TEXT NOT NULL,                  -- "2025-W02"

    -- Estimates
    total_cases         INTEGER NOT NULL DEFAULT 0,
    resistant_cases     INTEGER NOT NULL DEFAULT 0,
    rate                REAL,                           -- resistant/total
    ci_lower            REAL,                           -- Wilson score CI lower
    ci_upper            REAL,                           -- Wilson score CI upper
    trend               TEXT CHECK(trend IN ('increasing','stable','decreasing','unknown')),
    trend_slope         REAL,                           -- Linear regression slope

    -- Metadata
    sample_size_met     INTEGER NOT NULL DEFAULT 0,     -- 1 if total >= min threshold
    computed_at         TEXT NOT NULL DEFAULT (datetime('now')),

    -- Sync
    synced              INTEGER NOT NULL DEFAULT 0,
    synced_at           TEXT,
    sync_batch_id       TEXT,

    created_at          TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at          TEXT NOT NULL DEFAULT (datetime('now')),

    -- Unique constraint: one estimate per drug per district per week
    CONSTRAINT uq_resistance UNIQUE (drug, district, period_start, period_end)
);

-- Indexes
CREATE INDEX IF NOT EXISTS idx_resistance_drug ON resistance_estimates(drug);
CREATE INDEX IF NOT EXISTS idx_resistance_district ON resistance_estimates(district);
CREATE INDEX IF NOT EXISTS idx_resistance_period ON resistance_estimates(period_start, period_end);
CREATE INDEX IF NOT EXISTS idx_resistance_synced ON resistance_estimates(synced);
CREATE UNIQUE INDEX IF NOT EXISTS idx_resistance_unique
    ON resistance_estimates(drug, district, period_start, period_end);

-- ── 3. SYNC QUEUE ───────────────────────────────────────────────────────────
-- Tracks records pending upload to cloud. Managed by the SyncEngine.

CREATE TABLE IF NOT EXISTS sync_queue (
    id                  INTEGER PRIMARY KEY AUTOINCREMENT,
    record_id           TEXT NOT NULL,                  -- UUID of source record
    record_type         TEXT NOT NULL CHECK(record_type IN (
                           'case', 'resistance', 'alert', 'ussd_session'
                       )),
    operation           TEXT NOT NULL CHECK(operation IN ('INSERT','UPDATE','DELETE')),
    payload             TEXT NOT NULL,                  -- Full JSON snapshot
    hlc_timestamp       TEXT NOT NULL,                  -- Hybrid Logical Clock timestamp
    priority            INTEGER NOT NULL DEFAULT 5 CHECK(priority BETWEEN 1 AND 10),
    retries             INTEGER NOT NULL DEFAULT 0,
    max_retries         INTEGER NOT NULL DEFAULT 10,
    next_retry_at       TEXT,                           -- NULL = ready now
    batch_id            TEXT,                           -- Set when batched
    last_error          TEXT,
    created_at          TEXT NOT NULL DEFAULT (datetime('now')),
    encrypted_size      INTEGER                         -- Bytes after compression+encryption
);

-- Priority-ordered index: pick highest priority first, then oldest
CREATE INDEX IF NOT EXISTS idx_sync_queue_ready
    ON sync_queue(priority ASC, created_at ASC)
    WHERE next_retry_at IS NULL OR next_retry_at <= datetime('now');

-- Track retries for monitoring
CREATE INDEX IF NOT EXISTS idx_sync_queue_retries ON sync_queue(retries);

-- ── 4. CHW PROFILES ─────────────────────────────────────────────────────────
-- Community Health Worker profiles (for audit trail and workload tracking).

CREATE TABLE IF NOT EXISTS chw_profiles (
    chw_id              TEXT PRIMARY KEY,              -- UUID or facility-assigned ID
    facility_code       TEXT NOT NULL,
    district            TEXT NOT NULL,
    name_hash           TEXT,                           -- SHA-256 of name (no PII stored)
    role                TEXT DEFAULT 'chw',             -- 'chw', 'nurse', 'pharmacist'
    phone_hash          TEXT,                           -- SHA-256 of phone (for dedup only)
    is_active           INTEGER NOT NULL DEFAULT 1,

    -- Stats
    total_cases_entered INTEGER NOT NULL DEFAULT 0,
    last_active_at      TEXT,

    -- Device association
    assigned_device_id  TEXT,                           -- Which RPi this CHW uses

    created_at          TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at          TEXT NOT NULL DEFAULT (datetime('now')),

    FOREIGN KEY (facility_code) REFERENCES cases(facility_code)
);

CREATE INDEX IF NOT EXISTS idx_chw_facility ON chw_profiles(facility_code);
CREATE INDEX IF NOT EXISTS idx_chw_active ON chw_profiles(is_active) WHERE is_active = 1;

-- ── 5. ALERT HISTORY ────────────────────────────────────────────────────────
-- AMR alerts generated when resistance thresholds are exceeded.

CREATE TABLE IF NOT EXISTS alert_history (
    id                  TEXT PRIMARY KEY,              -- UUID v4
    alert_type          TEXT NOT NULL CHECK(alert_type IN (
                           'RESISTANCE_THRESHOLD',
                           'UNUSUAL_PATTERN',
                           'NEW_RESISTANCE_CASE',
                           'DEVICE_OFFLINE',
                           'SYNC_BACKLOG',
                           'DATA_QUALITY'
                       )),
    severity            TEXT NOT NULL CHECK(severity IN ('LOW','MEDIUM','HIGH','CRITICAL')),

    -- Context
    drug                TEXT,                           -- For resistance alerts
    district            TEXT,
    county              TEXT,
    current_rate        REAL,                           -- Current resistance rate
    threshold_rate      REAL,                           -- Threshold that was exceeded

    -- Details
    message             TEXT NOT NULL,                  -- Human-readable alert message
    recommended_action  TEXT,                           -- Suggested response
    recommended_alternatives TEXT,                      -- JSON array of alt drugs

    -- Status
    acknowledged        INTEGER NOT NULL DEFAULT 0,
    acknowledged_by     TEXT,                           -- CHW ID or system
    acknowledged_at     TEXT,

    -- Sync
    synced              INTEGER NOT NULL DEFAULT 0,
    synced_at           TEXT,
    sync_batch_id       TEXT,

    created_at          TEXT NOT NULL DEFAULT (datetime('now'))
);

CREATE INDEX IF NOT EXISTS idx_alert_type ON alert_history(alert_type);
CREATE INDEX IF NOT EXISTS idx_alert_severity ON alert_history(severity);
CREATE INDEX IF NOT EXISTS idx_alert_district ON alert_history(district);
CREATE INDEX IF NOT EXISTS idx_alert_acknowledged ON alert_history(acknowledged)
    WHERE acknowledged = 0;
CREATE INDEX IF NOT EXISTS idx_alert_created ON alert_history(created_at);
CREATE INDEX IF NOT EXISTS idx_alert_synced ON alert_history(synced);

-- ── 6. USSD SESSIONS ───────────────────────────────────────────────────────
-- Tracks USSD interaction sessions for feature phone users.

CREATE TABLE IF NOT EXISTS ussd_sessions (
    id                  TEXT PRIMARY KEY,              -- UUID v4
    session_id          TEXT NOT NULL,                  -- Telco session ID
    phone_number_hash   TEXT NOT NULL,                  -- SHA-256 of MSISDN
    network_operator    TEXT,                           -- "safaricom", "airtel", "mtn"
    country_code        TEXT NOT NULL DEFAULT '254',    -- International dialing code

    -- Session state machine
    current_state       TEXT NOT NULL DEFAULT 'MAIN',
    session_data        TEXT DEFAULT '{}',              -- JSON: accumulated form data

    -- Outcome
    completed           INTEGER NOT NULL DEFAULT 0,     -- 1 = reached CONFIRM state
    case_id             TEXT,                           -- Linked case if completed
    final_submission    TEXT,                           -- JSON: final submitted data

    -- Timing
    started_at          TEXT NOT NULL DEFAULT (datetime('now')),
    last_interaction_at TEXT NOT NULL DEFAULT (datetime('now')),
    ended_at            TEXT,                           -- NULL if session still active
    duration_seconds    INTEGER,                        -- Calculated on session end

    -- Sync
    synced              INTEGER NOT NULL DEFAULT 0,
    synced_at           TEXT,

    created_at          TEXT NOT NULL DEFAULT (datetime('now')),

    FOREIGN KEY (case_id) REFERENCES cases(id)
);

CREATE INDEX IF NOT EXISTS idx_ussd_session_id ON ussd_sessions(session_id);
CREATE INDEX IF NOT EXISTS idx_ussd_phone ON ussd_sessions(phone_number_hash);
CREATE INDEX IF NOT EXISTS idx_ussd_state ON ussd_sessions(current_state);
CREATE INDEX IF NOT EXISTS idx_ussd_completed ON ussd_sessions(completed);
CREATE INDEX IF NOT EXISTS idx_ussd_started ON ussd_sessions(started_at);
CREATE INDEX IF NOT EXISTS idx_ussd_synced ON ussd_sessions(synced);

-- ── SUPPORTING TABLES (created by system, not migrations) ───────────────────

-- Sync conflict log (populated by SyncEngine CRDT resolution)
CREATE TABLE IF NOT EXISTS sync_conflict_log (
    id                  INTEGER PRIMARY KEY AUTOINCREMENT,
    batch_id            TEXT NOT NULL,
    record_id           TEXT NOT NULL,
    field               TEXT,
    action              TEXT,                           -- 'local_wins', 'remote_wins', 'union_merge'
    local_value         TEXT,
    remote_value        TEXT,
    resolved_at         TEXT NOT NULL DEFAULT (datetime('now'))
);

CREATE INDEX IF NOT EXISTS idx_conflict_batch ON sync_conflict_log(batch_id);
CREATE INDEX IF NOT EXISTS idx_conflict_record ON sync_conflict_log(record_id);
CREATE INDEX IF NOT EXISTS idx_conflict_resolved ON sync_conflict_log(resolved_at);

-- Dead letter queue: records that exceeded max retries
CREATE TABLE IF NOT EXISTS sync_dead_letter (
    id                  INTEGER PRIMARY KEY AUTOINCREMENT,
    source_id           INTEGER NOT NULL,               -- Original sync_queue.id
    record_id           TEXT NOT NULL,
    payload             TEXT NOT NULL,
    error               TEXT,
    retries             INTEGER NOT NULL,
    failed_at           TEXT NOT NULL DEFAULT (datetime('now')),
    resolved            INTEGER NOT NULL DEFAULT 0,
    resolved_at         TEXT,
    resolution_note     TEXT
);

CREATE INDEX IF NOT EXISTS idx_dead_letter_resolved ON sync_dead_letter(resolved)
    WHERE resolved = 0;

-- ── TRIGGERS ─────────────────────────────────────────────────────────────────

-- Auto-update updated_at timestamp
CREATE TRIGGER IF NOT EXISTS trg_cases_updated_at
    AFTER UPDATE ON cases
    FOR EACH ROW
    BEGIN
        UPDATE cases SET updated_at = datetime('now') WHERE id = OLD.id;
    END;

CREATE TRIGGER IF NOT EXISTS trg_resistance_updated_at
    AFTER UPDATE ON resistance_estimates
    FOR EACH ROW
    BEGIN
        UPDATE resistance_estimates SET updated_at = datetime('now') WHERE id = OLD.id;
    END;

CREATE TRIGGER IF NOT EXISTS trg_ussd_last_interaction
    AFTER UPDATE ON ussd_sessions
    FOR EACH ROW
    BEGIN
        UPDATE ussd_sessions SET last_interaction_at = datetime('now') WHERE id = OLD.id;
    END;

-- Auto-compute duration on session end
CREATE TRIGGER IF NOT EXISTS trg_ussd_compute_duration
    AFTER UPDATE ON ussd_sessions
    FOR EACH ROW
    WHEN NEW.ended_at IS NOT NULL AND OLD.ended_at IS NULL
    BEGIN
        UPDATE ussd_sessions SET
            duration_seconds = CAST(
                (julianday(NEW.ended_at) - julianday(NEW.started_at)) * 86400 AS INTEGER
            )
        WHERE id = NEW.id;
    END;
```

### 3.2 Sample Symptoms JSON Format

The `symptoms` column in `cases` stores a JSON array:

```json
[
  {
    "name": "fever",
    "category": "SYSTEMIC",
    "duration_days": 5,
    "severity": "moderate",
    "onset": "acute",
    "location": null,
    "associated": ["headache", "chills"]
  },
  {
    "name": "dysuria",
    "category": "GENITOURINARY",
    "duration_days": 3,
    "severity": "severe",
    "onset": "acute",
    "location": null,
    "associated": []
  },
  {
    "name": "flank_pain",
    "category": "GENITOURINARY",
    "duration_days": 3,
    "severity": "moderate",
    "onset": "acute",
    "location": "left",
    "associated": ["nausea"]
  }
]
```

### 3.3 Sample Drugs JSON Format

The `drugs_prescribed` column in `cases` stores a JSON array:

```json
[
  {
    "name": "ciprofloxacin",
    "generic_name": "ciprofloxacin",
    "dose_mg": 500,
    "frequency": "bid",
    "duration_days": 7,
    "source": "prescription",
    "route": "oral",
    "brand_name": null
  },
  {
    "name": "paracetamol",
    "generic_name": "paracetamol",
    "dose_mg": 1000,
    "frequency": "qid",
    "duration_days": 5,
    "source": "otc",
    "route": "oral",
    "brand_name": "Panadol"
  }
]
```

---

## 4. Cloud PostgreSQL+TimescaleDB+PostGIS

### 4.1 Extensions Setup

```sql
-- ============================================================================
-- Cloud Database: Extension Setup
-- Target: PostgreSQL 16 + TimescaleDB 2.x + PostGIS 3.x
-- Managed service: AWS RDS / Google Cloud SQL / Azure Database for PostgreSQL
-- ============================================================================

-- TimescaleDB: time-series hypertables with automatic partitioning
CREATE EXTENSION IF NOT EXISTS timescaledb;

-- PostGIS: geospatial queries for location-based resistance mapping
CREATE EXTENSION IF NOT EXISTS postgis;
CREATE EXTENSION IF NOT EXISTS postgis_topology;

-- pgcrypto: for hashing and encryption utilities
CREATE EXTENSION IF NOT EXISTS pgcrypto;

-- uuid-ossp: UUID generation functions
CREATE EXTENSION IF NOT EXISTS "uuid-ossp";
```

### 4.2 National Cases Hypertable

```sql
-- ============================================================================
-- national_cases: All clinical cases aggregated from edge devices
-- Uses TimescaleDB hypertable for automatic time-based partitioning
-- ============================================================================

CREATE TABLE IF NOT EXISTS national_cases (
    -- Identity
    id                  UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    case_id            TEXT NOT NULL,                  -- Original edge case UUID
    device_id           TEXT NOT NULL,                  -- Source edge device
    facility_code       TEXT NOT NULL,
    district            TEXT NOT NULL,
    county              TEXT,
    country             TEXT NOT NULL DEFAULT 'KEN',
    national_id         TEXT,                           -- National health ID (if assigned)

    -- Patient demographics (anonymized)
    patient_age_years   INTEGER,
    patient_age_group   TEXT,
    patient_sex         TEXT CHECK(patient_sex IN ('M','F','O','U')),

    -- Encounter
    encounter_date      DATE NOT NULL,
    source_type         TEXT NOT NULL,
    language            TEXT DEFAULT 'en',

    -- Clinical data (JSONB for flexible schema)
    symptoms            JSONB,                          -- Array of symptom objects
    diagnosis_suggestion TEXT,
    drugs_prescribed    JSONB,                          -- Array of drug objects
    clinical_notes      TEXT,

    -- AI metadata
    amr_risk_score      REAL CHECK(amr_risk_score >= 0 AND amr_risk_score <= 1),
    extraction_confidence REAL,
    extracted_by        TEXT,

    -- Geospatial (PostGIS GEOGRAPHY type for accurate distance calculations)
    location            GEOGRAPHY(POINT, 4326),         -- Lat/lon as geography
    geohash             TEXT,

    -- Metadata
    received_at         TIMESTAMPTZ NOT NULL DEFAULT NOW(),  -- When cloud received this
    created_at_edge     TIMESTAMPTZ,                    -- When edge created this

    -- Data quality
    quality_score       REAL,                           -- 0-1 data quality assessment
    is_duplicate        BOOLEAN NOT NULL DEFAULT FALSE,
    duplicate_of        UUID,

    -- Partitioning time column (TimescaleDB requirement)
    -- We use received_at as the time column for partitioning
    created_at          TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

-- Convert to TimescaleDB hypertable, partitioned by received_at
-- chunk_time_interval: 1 week per chunk (balances query performance vs chunk count)
-- For ~50,000 cases/month, this creates ~4 chunks/month
SELECT create_hypertable('national_cases', 'received_at',
    chunk_time_interval => INTERVAL '1 week'
);

-- Add space partitioning on country for multi-country deployments
-- This creates separate chunk groups per country
SELECT add_dimension('national_cases', 'country', number_partitions => 4);

-- Indexes
CREATE INDEX idx_national_cases_device ON national_cases(device_id);
CREATE INDEX idx_national_cases_facility ON national_cases(facility_code);
CREATE INDEX idx_national_cases_district ON national_cases(district);
CREATE INDEX idx_national_cases_county ON national_cases(county);
CREATE INDEX idx_national_cases_date ON national_cases(encounter_date);
CREATE INDEX idx_national_cases_country ON national_cases(country);

-- JSONB indexes for symptom and drug queries
CREATE INDEX idx_national_cases_symptoms ON national_cases USING GIN (symptoms);
CREATE INDEX idx_national_cases_drugs ON national_cases USING GIN (drugs_prescribed);

-- Geospatial index (GiST for GEOGRAPHY type)
CREATE INDEX idx_national_cases_location ON national_cases USING GIST (location);

-- AMR risk index for filtering high-risk cases
CREATE INDEX idx_national_cases_amr_risk ON national_cases(amr_risk_score)
    WHERE amr_risk_score IS NOT NULL;

-- Duplicate detection index
CREATE INDEX idx_national_cases_duplicates ON national_cases(is_duplicate, duplicate_of)
    WHERE is_duplicate = FALSE;

-- Continuous aggregate: daily case counts by district
CREATE MATERIALIZED VIEW national_cases_daily
WITH (timescaledb.continuous) AS
SELECT
    time_bucket('1 day', received_at) AS day,
    district,
    county,
    country,
    source_type,
    COUNT(*) AS total_cases,
    COUNT(*) FILTER (WHERE amr_risk_score >= 0.7) AS high_risk_cases,
    AVG(amr_risk_score) AS avg_amr_risk,
    COUNT(DISTINCT facility_code) AS active_facilities,
    COUNT(DISTINCT device_id) AS active_devices
FROM national_cases
WHERE is_duplicate = FALSE
GROUP BY day, district, county, country, source_type;

-- Auto-refresh the materialized view every hour
SELECT add_continuous_aggregate_policy('national_cases_daily',
    start_offset   => INTERVAL '3 hours',
    end_offset     => INTERVAL '1 hour',
    schedule_interval => INTERVAL '1 hour'
);

-- Data retention: keep raw data for 5 years, aggregated for longer
SELECT add_retention_policy('national_cases', INTERVAL '5 years');
```

### 4.3 Weekly Resistance Summary (Materialized View)

```sql
-- ============================================================================
-- weekly_resistance_summary: Pre-computed resistance rates by drug and district
-- Refreshed weekly; used by the dashboard and Agent B queries
-- ============================================================================

CREATE TABLE weekly_resistance_summary (
    id                  BIGSERIAL,
    drug                TEXT NOT NULL,
    district            TEXT NOT NULL,
    county              TEXT,
    country             TEXT NOT NULL DEFAULT 'KEN',
    week_start          DATE NOT NULL,
    week_end            DATE NOT NULL,
    iso_year_week       TEXT NOT NULL,

    -- Metrics
    total_encounters    INTEGER NOT NULL DEFAULT 0,
    drug_prescribed     INTEGER NOT NULL DEFAULT 0,     -- Cases where this drug was given
    resistant_cases     INTEGER NOT NULL DEFAULT 0,     -- Cases where this drug showed resistance
    rate                REAL,                           -- resistance / drug_prescribed
    ci_lower            REAL,                           -- 95% Wilson CI lower
    ci_upper            REAL,                           -- 95% Wilson CI upper

    -- Trend
    prev_week_rate      REAL,                           -- Rate from previous week
    trend               TEXT CHECK(trend IN ('increasing','stable','decreasing','new','unknown')),
    trend_change_pct    REAL,                           -- Percentage change from prev week

    -- Data quality
    sample_size_met     BOOLEAN NOT NULL DEFAULT FALSE, -- TRUE if >= 30 encounters
    min_sample_size     INTEGER NOT NULL DEFAULT 30,

    -- Metadata
    computed_at         TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    source_facilities   INTEGER[],                      -- Array of facility codes contributing

    PRIMARY KEY (drug, district, week_start)
);

-- Convert to hypertable
SELECT create_hypertable('weekly_resistance_summary', 'week_start',
    chunk_time_interval => INTERVAL '1 month'
);

-- Indexes
CREATE INDEX idx_weekly_resistance_drug ON weekly_resistance_summary(drug);
CREATE INDEX idx_weekly_resistance_district ON weekly_resistance_summary(district);
CREATE INDEX idx_weekly_resistance_country ON weekly_resistance_summary(country);
CREATE INDEX idx_weekly_resistance_sample ON weekly_resistance_summary(sample_size_met)
    WHERE sample_size_met = TRUE;

-- Function to compute and refresh the weekly summary
CREATE OR REPLACE FUNCTION refresh_weekly_resistance_summary(
    p_drug TEXT DEFAULT NULL,
    p_district TEXT DEFAULT NULL,
    p_week_start DATE DEFAULT NULL
)
RETURNS INTEGER AS $$
DECLARE
    v_count INTEGER;
BEGIN
    INSERT INTO weekly_resistance_summary (
        drug, district, county, country, week_start, week_end, iso_year_week,
        total_encounters, drug_prescribed, resistant_cases, rate,
        ci_lower, ci_upper, prev_week_rate, trend, trend_change_pct,
        sample_size_met, source_facilities, computed_at
    )
    SELECT
        drug_unnest AS drug,
        nc.district,
        nc.county,
        nc.country,
        week_start,
        week_start + 6 AS week_end,
        TO_CHAR(week_start, 'IYYY-"W"IW') AS iso_year_week,

        -- Total encounters in this district/week
        total_enc.total AS total_encounters,

        -- Encounters where this specific drug was prescribed
        COUNT(*) AS drug_prescribed,

        -- Resistant cases (where amr_risk_score >= 0.7 for this drug)
        COUNT(*) FILTER (
            WHERE nc.amr_risk_score >= 0.7
        ) AS resistant_cases,

        -- Rate with NULLIF to avoid division by zero
        CASE
            WHEN COUNT(*) > 0 THEN
                COUNT(*) FILTER (WHERE nc.amr_risk_score >= 0.7)::REAL / COUNT(*)
            ELSE NULL
        END AS rate,

        -- Wilson score confidence interval (95%)
        CASE WHEN COUNT(*) > 0 THEN
            -- Wilson score CI lower bound
            (COUNT(*) FILTER (WHERE nc.amr_risk_score >= 0.7) + 1.96*1.96/2
             - 1.96 * SQRT(
                 (COUNT(*) FILTER (WHERE nc.amr_risk_score >= 0.7) *
                  (COUNT(*) - COUNT(*) FILTER (WHERE nc.amr_risk_score >= 0.7)) +
                  1.96*1.96*COUNT(*) / 4) / (COUNT()*COUNT())
               )
            ) / (COUNT(*) + 1.96*1.96)
        ELSE NULL END AS ci_lower,

        CASE WHEN COUNT(*) > 0 THEN
            (COUNT(*) FILTER (WHERE nc.amr_risk_score >= 0.7) + 1.96*1.96/2
             + 1.96 * SQRT(
                 (COUNT(*) FILTER (WHERE nc.amr_risk_score >= 0.7) *
                  (COUNT(*) - COUNT(*) FILTER (WHERE nc.amr_risk_score >= 0.7)) +
                  1.96*1.96*COUNT(*) / 4) / (COUNT()*COUNT())
               )
            ) / (COUNT(*) + 1.96*1.96)
        ELSE NULL END AS ci_upper,

        -- Previous week rate for trend calculation
        prev_wk.rate AS prev_week_rate,

        -- Trend classification
        CASE
            WHEN prev_wk.rate IS NULL THEN 'new'
            WHEN ABS(COALESCE(rate, 0) - prev_wk.rate) < 0.05 THEN 'stable'
            WHEN COALESCE(rate, 0) > prev_wk.rate + 0.05 THEN 'increasing'
            ELSE 'decreasing'
        END AS trend,

        -- Percentage change
        CASE WHEN prev_wk.rate IS NOT NULL AND prev_wk.rate > 0 THEN
            ROUND((COALESCE(rate, 0) - prev_wk.rate) / prev_wk.rate * 100, 1)
        ELSE NULL END AS trend_change_pct,

        -- Sample size threshold
        COUNT(*) >= 30 AS sample_size_met,

        -- Contributing facilities
        ARRAY_AGG(DISTINCT nc.facility_code) AS source_facilities,

        NOW() AS computed_at

    FROM
        national_cases nc
    CROSS JOIN LATERAL UNNEST(nc.drugs_prescribed) AS drug_obj
    CROSS JOIN LATERAL (SELECT (drug_obj->>'name')::TEXT AS drug_unnest) AS d
    CROSS JOIN LATERAL (
        SELECT date_trunc('week', nc.encounter_date)::DATE AS week_start
    ) AS wk
    LEFT JOIN LATERAL (
        SELECT rate FROM weekly_resistance_summary wrs
        WHERE wrs.drug = d.drug_unnest
          AND wrs.district = nc.district
          AND wrs.week_start = wk.week_start - INTERVAL '1 week'
        LIMIT 1
    ) prev_wk ON TRUE

    WHERE nc.is_duplicate = FALSE
      AND nc.drugs_prescribed IS NOT NULL
      AND (p_drug IS NULL OR d.drug_unnest = p_drug)
      AND (p_district IS NULL OR nc.district = p_district)
      AND (p_week_start IS NULL OR wk.week_start = p_week_start)

    GROUP BY d.drug_unnest, nc.district, nc.county, nc.country,
             wk.week_start, total_enc.total, prev_wk.rate

    ON CONFLICT (drug, district, week_start)
    DO UPDATE SET
        total_encounters = EXCLUDED.total_encounters,
        drug_prescribed = EXCLUDED.drug_prescribed,
        resistant_cases = EXCLUDED.resistant_cases,
        rate = EXCLUDED.rate,
        ci_lower = EXCLUDED.ci_lower,
        ci_upper = EXCLUDED.ci_upper,
        prev_week_rate = EXCLUDED.prev_week_rate,
        trend = EXCLUDED.trend,
        trend_change_pct = EXCLUDED.trend_change_pct,
        sample_size_met = EXCLUDED.sample_size_met,
        source_facilities = EXCLUDED.source_facilities,
        computed_at = EXCLUDED.computed_at;

    GET DIAGNOSTICS v_count = ROW_COUNT;
    RETURN v_count;
END;
$$ LANGUAGE plpgsql;

-- Refresh policy: run every Monday at 02:00 UTC
-- (After weekend data has synced from edge devices)
SELECT cron.schedule(
    'refresh_weekly_resistance',
    '0 2 * * 1',  -- Every Monday at 02:00 UTC
    $$SELECT refresh_weekly_resistance_summary()$$
);
```

### 4.4 Additional Cloud Tables

```sql
-- ============================================================================
-- Device registry and heartbeat tracking
-- ============================================================================

CREATE TABLE IF NOT EXISTS devices (
    device_id           TEXT PRIMARY KEY,
    facility_code       TEXT NOT NULL,
    district            TEXT NOT NULL,
    county              TEXT,
    country             TEXT NOT NULL DEFAULT 'KEN',

    -- Device metadata
    device_model        TEXT,                           -- "Raspberry Pi 4B"
    device_arch         TEXT,                           -- "arm64"
    os_version          TEXT,                           -- "Ubuntu 24.04 LTS"
    agent_versions      JSONB DEFAULT '{}',             -- {"agent-a": "2.3.1", ...}

    -- Connectivity
    last_heartbeat      TIMESTAMPTZ,
    last_sync           TIMESTAMPTZ,
    last_model_update   TIMESTAMPTZ,
    connectivity_tier   TEXT CHECK(connectivity_tier IN ('1','2','3','4')),

    -- Status
    is_active           BOOLEAN NOT NULL DEFAULT TRUE,
    alerts_count        INTEGER NOT NULL DEFAULT 0,

    -- Location
    location            GEOGRAPHY(POINT, 4326),

    registered_at       TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_devices_facility ON devices(facility_code);
CREATE INDEX idx_devices_active ON devices(is_active) WHERE is_active = TRUE;
CREATE INDEX idx_devices_heartbeat ON devices(last_heartbeat);
CREATE INDEX idx_devices_location ON devices USING GIST (location);

-- ============================================================================
-- Sync audit log (every sync batch from every device)
-- ============================================================================

CREATE TABLE IF NOT EXISTS sync_audit_log (
    id                  BIGSERIAL PRIMARY KEY,
    batch_id            UUID NOT NULL,
    device_id           TEXT NOT NULL,
    facility_code       TEXT NOT NULL,

    -- Batch info
    record_count        INTEGER NOT NULL,
    applied_count       INTEGER NOT NULL DEFAULT 0,
    conflict_count      INTEGER NOT NULL DEFAULT 0,
    rejected_count      INTEGER NOT NULL DEFAULT 0,

    -- Payload info
    compressed_size_kb  REAL,
    encrypted           BOOLEAN NOT NULL DEFAULT TRUE,

    -- Timing
    received_at         TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    processing_time_ms  INTEGER,

    -- Status
    status              TEXT NOT NULL CHECK(status IN (
        'accepted', 'partial', 'rejected', 'error'
    )),
    error_message       TEXT,

    -- Client info
    client_ip           INET,
    user_agent          TEXT,

    FOREIGN KEY (device_id) REFERENCES devices(device_id)
);

SELECT create_hypertable('sync_audit_log', 'received_at',
    chunk_time_interval => INTERVAL '1 month'
);

CREATE INDEX idx_sync_audit_device ON sync_audit_log(device_id);
CREATE INDEX idx_sync_audit_status ON sync_audit_log(status);
CREATE INDEX idx_sync_audit_batch ON sync_audit_log(batch_id);
```

---

## 5. ASCII ER Diagram

```
╔════════════════════════════════════════════════════════════════════════════════╗
║                          UDARA DATABASE ER DIAGRAM                            ║
║                          (Edge SQLite + Cloud PostgreSQL)                      ║
╠════════════════════════════════════════════════════════════════════════════════╣
║                                                                                 ║
║  EDGE (SQLite)                    CLOUD (PostgreSQL+TimescaleDB+PostGIS)       ║
║  ─────────────                    ──────────────────────────────────────       ║
║                                                                                 ║
║  ┌──────────────────────┐                                                    ║
║  │       CASES          │ ═══ SYNC ═══>  ┌───────────────────────────────┐    ║
║  ├──────────────────────┤                  │     NATIONAL_CASES            │    ║
║  │ PK  id (TEXT/UUID)   │                  │  (HYPERTABLE by received_at)  │    ║
║  │     facility_code    │                  ├───────────────────────────────┤    ║
║  │     district         │                  │ PK  id (UUID)                 │    ║
║  │     county           │                  │ FK  case_id ──────────────────┼──┐ ║
║  │     country          │                  │     device_id ───┐            │  │ ║
║  │     patient_age_*    │                  │     facility_code │            │  │ ║
║  │     encounter_date   │                  │     district      │            │  │ ║
║  │     symptoms (JSON)  │                  │     encounter_date│            │  │ ║
║  │     drugs (JSON)     │                  │     symptoms (JSONB)           │  │ ║
║  │     amr_risk_score   │                  │     drugs (JSONB)              │  │ ║
║  │     geohash          │                  │     location (GEOGRAPHY)      │  │ ║
║  │     synced           │                  │     amr_risk_score            │  │ ║
║  └──────────┬───────────┘                  └──────────────┬────────────────┘  │ ║
║             │                                              │                  │ ║
║             │ 1:N                                          │ 1:N              │ ║
║             ▼                                              ▼                  │ ║
║  ┌──────────────────────┐                  ┌───────────────────────────────┐ │ ║
║  │ RESISTANCE_ESTIMATES │ ═══ SYNC ═══>  │  WEEKLY_RESISTANCE_SUMMARY    │ │ ║
║  ├──────────────────────┤                  │  (HYPERTABLE by week_start)   │ │ ║
║  │ PK  id               │                  ├───────────────────────────────┤ │ ║
║  │ UQ  drug+district+   │                  │ PK  drug+district+week_start  │ │ ║
║  │     period_start+    │                  │     rate, ci_lower, ci_upper   │ │ ║
║  │     period_end       │                  │     trend, trend_change_pct   │ │ ║
║  │     rate             │                  │     sample_size_met           │ │ ║
║  │     ci_lower/upper   │                  │     source_facilities[]       │ │ ║
║  │     trend            │                  └───────────────────────────────┘ │ ║
║  │     synced           │                                                  │ ║
║  └──────────────────────┘                                                  │ ║
║                                                                             │ ║
║  ┌──────────────────────┐                  ┌───────────────────────────────┐ │ ║
║  │     SYNC_QUEUE       │ ═══ SYNC ═══>  │     SYNC_AUDIT_LOG            │ │ ║
║  ├──────────────────────┤                  │  (HYPERTABLE by received_at)   │ │ ║
║  │ PK  id               │                  ├───────────────────────────────┤ │ ║
║  │ FK  record_id ───────┼───────┐         │ PK  id (BIGSERIAL)             │ │ ║
║  │     record_type      │       │         │     batch_id (UUID)           │ │ ║
║  │     operation        │       │         │ FK  device_id ────────────────┼─┘ ║
║  │     payload (JSON)   │       │         │     record_count              │    ║
║  │     hlc_timestamp    │       │         │     status                    │    ║
║  │     priority         │       │         │     processing_time_ms       │    ║
║  └──────────────────────┘       │         └──────────────┬────────────────┘    ║
║                                 │                        │                     ║
║           ┌─────────────────────┘                        │                     ║
║           │    (references cases.id,                      │                     ║
║           │     resistance_estimates.id,                  ▼                     ║
║           │     alert_history.id,           ┌───────────────────────────────┐ ║
║           │     ussd_sessions.id)           │         DEVICES               │ ║
║           │                                ├───────────────────────────────┤ ║
║           │                                │ PK  device_id                 │ ║
║           │                                │     facility_code             │ ║
║           │                                │     district                  │ ║
║  ┌────────┴──────────────────┐            │     last_heartbeat             │ ║
║  │     ALERT_HISTORY         │            │     location (GEOGRAPHY)       │ ║
║  ├──────────────────────────┤            └───────────────────────────────┘ ║
║  │ PK  id (TEXT/UUID)       │                                                    ║
║  │     alert_type            │                                                    ║
║  │     severity              │                                                    ║
║  │     drug                  │                                                    ║
║  │     district              │                                                    ║
║  │     message               │                                                    ║
║  │     acknowledged          │                                                    ║
║  └──────────────────────────┘                                                    ║
║                                                                                    ║
║  ┌──────────────────────┐                                                        ║
║  │     CHW_PROFILES     │                                                        ║
║  ├──────────────────────┤                                                        ║
║  │ PK  chw_id            │                                                        ║
║  │ FK  facility_code ────┼──> cases.facility_code (logical ref)                   ║
║  │     name_hash          │                                                        ║
║  │     total_cases_entered│                                                       ║
║  │     assigned_device_id─┼──> devices.device_id (logical ref)                    ║
║  └──────────────────────┘                                                        ║
║                                                                                    ║
║  ┌──────────────────────┐                                                        ║
║  │    USSD_SESSIONS      │                                                        ║
║  ├──────────────────────┤                                                        ║
║  │ PK  id (TEXT/UUID)    │                                                        ║
║  │     session_id         │                                                        ║
║  │     phone_number_hash  │                                                        ║
║  │     current_state      │                                                        ║
║  │ FK  case_id ───────────┼──> cases.id                                           ║
║  │     session_data(JSON) │                                                        ║
║  │     completed          │                                                        ║
║  └──────────────────────┘                                                        ║
║                                                                                    ║
║  ┌──────────────────────┐  (system tables, not user-facing)                        ║
║  │  _SCHEMA_MIGRATIONS  │                                                        ║
║  │  SYNC_CONFLICT_LOG   │                                                        ║
║  │  SYNC_DEAD_LETTER    │                                                        ║
║  └──────────────────────┘                                                        ║
║                                                                                    ║
╠════════════════════════════════════════════════════════════════════════════════╣
║  LEGEND                                                                           ║
║  ──────                                                                           ║
║  PK  = Primary Key        FK  = Foreign Key        UQ  = Unique Constraint       ║
║  ═══ = Sync relationship (edge→cloud, async)                                     ║
║  ───> = Foreign Key reference (within same database)                              ║
║  ()  = Data type note                                                             ║
╚════════════════════════════════════════════════════════════════════════════════╝
```

---

## 6. Data Type Notes

### 6.1 Type Mapping: SQLite ↔ PostgreSQL

| Concept | Edge (SQLite) | Cloud (PostgreSQL) | Notes |
|---------|--------------|--------------------|----|
| Primary key UUID | `TEXT` | `UUID` | SQLite has no native UUID; store as TEXT with validation |
| Timestamp | `TEXT` (ISO 8601) | `TIMESTAMPTZ` | SQLite stores as string; PostgreSQL has native time zone awareness |
| JSON data | `TEXT` (JSON string) | `JSONB` | PostgreSQL JSONB supports GIN indexes and `->>` operators |
| Geospatial point | `TEXT` (geohash) | `GEOGRAPHY(POINT, 4326)` | Edge stores geohash (no PostGIS); cloud stores full geometry |
| Boolean | `INTEGER` (0/1) | `BOOLEAN` | SQLite convention; PostgreSQL native boolean |
| Enum/Check | `CHECK(col IN (...))` | `CHECK(col IN (...))` or `ENUM` type | Both support CHECK constraints |
| Array | `TEXT` (JSON array) | `INTEGER[]` or `JSONB` | Cloud uses native array for facility lists |
| Auto-increment | `INTEGER PRIMARY KEY AUTOINCREMENT` | `BIGSERIAL` or `GENERATED ALWAYS AS IDENTITY` | Different syntax, same concept |

### 6.2 JSONB Column Usage Patterns (Cloud)

```sql
-- Query symptoms by category (GIN index enabled)
SELECT COUNT(*)
FROM national_cases
WHERE symptoms @> '[{"category": "GENITOURINARY"}]';

-- Extract all unique drug names prescribed
SELECT DISTINCT jsonb_array_elements(drugs_prescribed)->>'name' AS drug_name
FROM national_cases
WHERE drugs_prescribed IS NOT NULL;

-- Cases where ciprofloxacin was prescribed
SELECT *
FROM national_cases
WHERE drugs_prescribed @> '[{"name": "ciprofloxacin"}]';

-- Average AMR risk score for cases with specific symptom
SELECT
    jsonb_array_elements(symptoms)->>'name' AS symptom_name,
    AVG(amr_risk_score) AS avg_risk,
    COUNT(*) AS case_count
FROM national_cases, jsonb_array_elements(symptoms)
WHERE amr_risk_score IS NOT NULL
GROUP BY symptom_name
ORDER BY avg_risk DESC
LIMIT 20;

-- Geospatial query: cases within 50km of a point
SELECT id, district, amr_risk_score, encounter_date
FROM national_cases
WHERE ST_DWithin(location, ST_MakePoint(36.8219, -1.2921)::geography, 50000)
ORDER BY encounter_date DESC
LIMIT 100;
```

### 6.3 Geohash Precision Table

| Precision | Width | Height | Use Case |
|-----------|-------|--------|----------|
| 4 chars | ~39 km × 19 km | Regional | Country-level aggregation |
| **5 chars** | **~5 km × 5 km** | **District-level** | **Edge storage (default)** |
| 6 chars | ~1.2 km × 0.6 km | Facility catchment | Cloud detailed queries |
| 7 chars | ~150 m × 150 m | Neighborhood | Not used (too precise for privacy) |

---

## 7. Migration Strategy

### 7.1 Naming Convention

```
migrations/
├── edge/
│   ├── 001_initial_schema.sql
│   ├── 002_add_ussd_sessions.sql
│   ├── 003_add_sync_dead_letter.sql
│   ├── 004_add_quality_score_to_cases.sql
│   └── 005_add_ussd_network_operator.sql
└── cloud/
    ├── 001_initial_schema.sql
    ├── 002_add_continuous_aggregates.sql
    ├── 003_add_devices_table.sql
    ├── 004_add_retention_policies.sql
    └── 005_add_multi_country_support.sql
```

### 7.2 Migration File Template

```sql
-- ============================================================================
-- 006_add_case_resolution_field.sql
-- Purpose: Add treatment resolution tracking to cases
-- Author: @developer-name
-- Date: 2026-05-27
-- Backwards compatible: YES
-- ============================================================================

-- Step 1: Add new column (NULL allowed for existing rows)
ALTER TABLE cases ADD COLUMN resolution_outcome TEXT
    CHECK(resolution_outcome IN ('improved', 'no_change', 'worsened', 'lost_followup', 'unknown'));

-- Step 2: Add column for follow-up date
ALTER TABLE cases ADD COLUMN follow_up_date TEXT;

-- Step 3: Add index for resolution queries
CREATE INDEX IF NOT EXISTS idx_cases_resolution ON cases(resolution_outcome)
    WHERE resolution_outcome IS NOT NULL;

-- Step 4: Backfill existing rows (set NULL to 'unknown')
-- NOTE: Run this separately with a LIMIT for large databases
-- UPDATE cases SET resolution_outcome = 'unknown' WHERE resolution_outcome IS NULL;
```

### 7.3 Migration Rules

| Rule | Description |
|------|-------------|
| **Forward-only on edge** | No rollback migrations on edge devices. If a migration breaks things, the cloud sync will eventually correct it. |
| **NULL-safe additions** | Always add new columns as `NULL`-able first. Backfill separately. Only add `NOT NULL` constraints after backfill. |
| **No column drops** | Never drop columns on edge (existing code may reference them). Deprecate instead. |
| **Test on SQLite 3.39+** | Some ALTER TABLE features are not supported in older SQLite versions. |
| **Lock-free migrations** | Avoid long-running DDL. SQLite only supports one writer at a time. |
| **Version bump** | After a migration, update the `UDARA_DB_VERSION` constant in code. |

---

## 8. Seed Data & Test Fixtures

### 8.1 Edge SQLite Seed

```sql
-- seed_edge.sql — Test data for development

-- Insert test facility (referenced by all test data)
INSERT OR IGNORE INTO chw_profiles (chw_id, facility_code, district, name_hash, role)
VALUES
    ('chw-test-001', 'DEV-FAC-001', 'Nairobi', 'a1b2c3d4e5f6...', 'chw'),
    ('chw-test-002', 'DEV-FAC-002', 'Kisumu', 'f6e5d4c3b2a1...', 'nurse');

-- Insert test cases
INSERT OR IGNORE INTO cases (id, facility_code, district, county, country,
    patient_age_years, patient_age_group, patient_sex, encounter_date, source_type,
    symptoms, diagnosis_suggestion, drugs_prescribed, amr_risk_score,
    extraction_confidence, extracted_by, geohash)
VALUES
    ('case-test-001', 'DEV-FAC-001', 'Nairobi', 'Nairobi', 'KEN',
     28, '25-34', 'F', '2026-05-27', 'clinical_note',
     '[{"name":"fever","category":"SYSTEMIC","duration_days":5,"severity":"moderate"},
       {"name":"dysuria","category":"GENITOURINARY","duration_days":3,"severity":"severe"}]',
     'urinary_tract_infection',
     '[{"name":"ciprofloxacin","dose_mg":500,"frequency":"bid","duration_days":7,"source":"prescription"}]',
     0.72, 0.89, 'agent-a-v2.3.1', 'k17fqv'),

    ('case-test-002', 'DEV-FAC-001', 'Nairobi', 'Nairobi', 'KEN',
     45, '45-54', 'M', '2026-05-27', 'clinical_note',
     '[{"name":"cough","category":"RESPIRATORY","duration_days":14,"severity":"moderate"},
       {"name":"sputum","category":"RESPIRATORY","duration_days":10,"severity":"mild"}]',
     'respiratory_infection',
     '[{"name":"amoxicillin","dose_mg":500,"frequency":"tid","duration_days":7,"source":"prescription"}]',
     0.35, 0.92, 'agent-a-v2.3.1', 'k17fqv'),

    ('case-test-003', 'DEV-FAC-002', 'Kisumu', 'Kisumu', 'KEN',
     6, '5-14', 'M', '2025-01-14', 'ussd',
     '[{"name":"fever","category":"SYSTEMIC","duration_days":2,"severity":"mild"},
       {"name":"diarrhea","category":"GASTROINTESTINAL","duration_days":1,"severity":"moderate"}]',
     'gastroenteritis',
     '[{"name":"metronidazole","dose_mg":250,"frequency":"tid","duration_days":5,"source":"otc"}]',
     0.18, 0.75, 'agent-a-v2.3.1', 'k17fwr');

-- Insert test resistance estimate
INSERT OR IGNORE INTO resistance_estimates (drug, district, county, country,
    period_start, period_end, iso_year_week, total_cases, resistant_cases,
    rate, ci_lower, ci_upper, trend, sample_size_met)
VALUES
    ('ciprofloxacin', 'Nairobi', 'Nairobi', 'KEN',
     '2025-01-06', '2025-01-12', '2025-W02',
     47, 32, 0.681, 0.618, 0.739, 'increasing', 1),

    ('amoxicillin', 'Nairobi', 'Nairobi', 'KEN',
     '2025-01-06', '2025-01-12', '2025-W02',
     63, 15, 0.238, 0.164, 0.329, 'stable', 1),

    ('ciprofloxacin', 'Kisumu', 'Kisumu', 'KEN',
     '2025-01-06', '2025-01-12', '2025-W02',
     31, 25, 0.806, 0.694, 0.882, 'increasing', 1);

-- Insert test alert
INSERT OR IGNORE INTO alert_history (id, alert_type, severity, drug, district,
    current_rate, threshold_rate, message, recommended_action)
VALUES
    ('alert-test-001', 'RESISTANCE_THRESHOLD', 'HIGH', 'ciprofloxacin', 'Nairobi',
     0.681, 0.50,
     'Ciprofloxacin resistance in Nairobi (68.1%) exceeds 50% threshold',
     'Consider alternative antibiotics: nitrofurantoin, fosfomycin');
```

---

## 9. Query Patterns & Performance

### 9.1 Common Edge Queries

```sql
-- 1. Get unsynced cases for a facility (most common sync query)
SELECT id, facility_code, encounter_date, symptoms, drugs_prescribed, amr_risk_score
FROM cases
WHERE synced = 0 AND facility_code = 'KE-NRB-FAC-017'
ORDER BY created_at ASC
LIMIT 50;

-- 2. Count cases by district for the last 7 days
SELECT district, COUNT(*) AS case_count
FROM cases
WHERE encounter_date >= date('now', '-7 days')
GROUP BY district
ORDER BY case_count DESC;

-- 3. Get current resistance rate for a drug in a district
SELECT drug, district, rate, ci_lower, ci_upper, trend, iso_year_week
FROM resistance_estimates
WHERE drug = 'ciprofloxacin' AND district = 'Nairobi'
ORDER BY period_start DESC
LIMIT 1;

-- 4. Search symptoms across cases (JSON search via LIKE)
SELECT id, encounter_date, symptoms
FROM cases
WHERE symptoms LIKE '%fever%'
  AND encounter_date >= date('now', '-30 days')
LIMIT 20;

-- 5. Get high-risk unsynced cases (priority sync)
SELECT id, amr_risk_score, district, encounter_date
FROM cases
WHERE synced = 0 AND amr_risk_score >= 0.7
ORDER BY amr_risk_score DESC, created_at ASC;

-- 6. CHW activity summary
SELECT
    chw_id,
    COUNT(*) AS total_cases,
    MAX(cases.created_at) AS last_activity
FROM cases
JOIN chw_profiles ON cases.facility_code = chw_profiles.facility_code
WHERE chw_profiles.is_active = 1
GROUP BY chw_id;
```

### 9.2 Query Performance Targets (Edge SQLite on RPi 4B)

| Query Type | Target Latency | Notes |
|------------|---------------|-------|
| Single case by ID | < 5ms | Indexed PK lookup |
| Unsynced cases (50) | < 50ms | Composite index on (facility, synced) |
| District summary (7 days) | < 100ms | Date range scan + GROUP BY |
| Resistance rate lookup | < 10ms | Unique index on (drug, district, period) |
| JSON symptom search | < 500ms | LIKE on JSON string (not ideal, consider FTS5) |
| Full-text search (FTS5) | < 20ms | Use virtual table (see below) |

### 9.3 Full-Text Search Extension (Optional)

```sql
-- FTS5 virtual table for symptom searching (faster than LIKE)
CREATE VIRTUAL TABLE IF NOT EXISTS cases_fts USING fts5(
    symptoms_text,
    diagnosis_suggestion,
    clinical_notes,
    content=cases,
    content_rowid=rowid
);

-- Triggers to keep FTS in sync
CREATE TRIGGER IF NOT EXISTS trg_cases_fts_insert AFTER INSERT ON cases BEGIN
    INSERT INTO cases_fts(rowid, symptoms_text, diagnosis_suggestion, clinical_notes)
    VALUES (NEW.rowid, NEW.symptoms, NEW.diagnosis_suggestion, NEW.clinical_notes);
END;

CREATE TRIGGER IF NOT EXISTS trg_cases_fts_delete AFTER DELETE ON cases BEGIN
    INSERT INTO cases_fts(cases_fts, rowid, symptoms_text, diagnosis_suggestion, clinical_notes)
    VALUES ('delete', OLD.rowid, OLD.symptoms, OLD.diagnosis_suggestion, OLD.clinical_notes);
END;

CREATE TRIGGER IF NOT EXISTS trg_cases_fts_update AFTER UPDATE ON cases BEGIN
    INSERT INTO cases_fts(cases_fts, rowid, symptoms_text, diagnosis_suggestion, clinical_notes)
    VALUES ('delete', OLD.rowid, OLD.symptoms, OLD.diagnosis_suggestion, OLD.clinical_notes);
    INSERT INTO cases_fts(rowid, symptoms_text, diagnosis_suggestion, clinical_notes)
    VALUES (NEW.rowid, NEW.symptoms, NEW.diagnosis_suggestion, NEW.clinical_notes);
END;

-- Usage:
-- SELECT * FROM cases_fts WHERE cases_fts MATCH 'fever dysuria' ORDER BY rank;
```

---

## 10. Backup & Recovery

### 10.1 Edge SQLite Backup

```python
# shared/database/backup.py

import sqlite3
import shutil
from pathlib import Path
from datetime import datetime
import logging

logger = logging.getLogger("udara.db.backup")


def backup_edge_db(
    db_path: str = "/var/lib/udara/edge.db",
    backup_dir: str = "/var/lib/udara/backups",
    keep_count: int = 7,
) -> str:
    """
    Create a consistent backup of the edge SQLite database.

    Uses SQLite's built-in backup API (VACUUM INTO) which creates
    a consistent snapshot even with concurrent readers/writers.

    Args:
        db_path: Path to the source database.
        backup_dir: Directory to store backups.
        keep_count: Number of backups to retain (auto-cleanup old ones).

    Returns:
        Path to the created backup file.
    """
    db_file = Path(db_path)
    backup_path = Path(backup_dir)
    backup_path.mkdir(parents=True, exist_ok=True)

    timestamp = datetime.now().strftime("%Y%m%d_%H%M%S")
    backup_file = backup_path / f"edge_{timestamp}.db"

    # Use SQLite backup API for consistent snapshot
    conn = sqlite3.connect(str(db_file))
    try:
        conn.execute(f"VACUUM INTO '{backup_file}'")
        logger.info("Database backup created: %s (%d bytes)", backup_file, backup_file.stat().st_size)
    finally:
        conn.close()

    # Cleanup old backups
    existing = sorted(backup_path.glob("edge_*.db"))
    if len(existing) > keep_count:
        for old_file in existing[:-keep_count]:
            old_file.unlink()
            logger.info("Removed old backup: %s", old_file)

    return str(backup_file)


def verify_edge_db(db_path: str = "/var/lib/udara/edge.db") -> dict:
    """
    Run integrity check on the edge database.

    Returns:
        {"ok": bool, "error": str|None, "size_mb": float, "wal_size_mb": float}
    """
    result = {"ok": False, "error": None, "size_mb": 0, "wal_size_mb": 0}

    db_file = Path(db_path)
    if not db_file.exists():
        result["error"] = f"Database file not found: {db_path}"
        return result

    result["size_mb"] = round(db_file.stat().st_size / (1024 * 1024), 2)
    wal_file = db_file.with_suffix(".db-wal")
    if wal_file.exists():
        result["wal_size_mb"] = round(wal_file.stat().st_size / (1024 * 1024), 2)

    conn = sqlite3.connect(str(db_file))
    try:
        check = conn.execute("PRAGMA integrity_check").fetchone()
        result["ok"] = check[0] == "ok"
        if not result["ok"]:
            result["error"] = check[0]
    finally:
        conn.close()

    return result
```

### 10.2 Cloud Database Backup

```sql
-- Cloud backup is handled by the managed service (AWS RDS, GCP Cloud SQL).
-- Additional logical backup for portability:

-- Full pg_dump (run from a bastion host)
-- pg_dump -h udara-cloud-db.xxx.rds.amazonaws.com -U udara -d udara \
--     --format=custom --compress=9 \
--     --file=udara_backup_$(date +%Y%m%d).dump

-- Schema-only dump
-- pg_dump -h ... -U udara -d udara --schema-only --no-owner > schema.sql

-- Data-only dump (for migration)
-- pg_dump -h ... -U udara -d udara --data-only --no-owner > data.sql
```
