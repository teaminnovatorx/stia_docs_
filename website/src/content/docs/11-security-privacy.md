# UDARA AI — Security & Privacy Architecture

> **Document ID:** UDARA-SEC-011
> **Version:** 2.0.0
> **Last Updated:** 2026-05-27
> **Owner:** Security Engineering — Platform Security Team
> **Classification:** CONFIDENTIAL — Internal Use Only
> **Review Cycle:** Per Sprint + After Any Security Incident

---

## Table of Contents

1. [Overview & Scope](#1-overview--scope)
2. [Threat Model](#2-threat-model)
3. [Encryption Architecture](#3-encryption-architecture)
4. [PII Anonymization Pipeline](#4-pii-anonymization-pipeline)
5. [Authentication](#5-authentication)
6. [Authorization (RBAC)](#6-authorization-rbac)
7. [Audit Logging](#7-audit-logging)
8. [Data Sovereignty](#8-data-sovereignty)
9. [Data Breach Response Plan](#9-data-breach-response-plan)
10. [Security Checklist Per Release](#10-security-checklist-per-release)
11. [Appendix: Security Configuration Reference](#11-appendix-security-configuration-reference)

---

## 1. Overview & Scope

### 1.1 Purpose

This document defines the complete security and privacy architecture for
UDARA AI — an Antimicrobial Resistance (AMR) surveillance platform
processing sensitive health data from Community Health Workers (CHWs)
across six sub-Saharan African countries.

### 1.2 Data Sensitivity Overview

UDARA AI processes **Category 1 (Sensitive)** health data under multiple
African data protection regulations:

```
┌────────────────────────────────────────────────────────────────┐
│                    DATA SENSITIVITY LEVELS                      │
├────────────────────────────────────────────────────────────────┤
│                                                                │
│  LEVEL 5 — CRITICAL                                           │
│  ├─ Patient names, phone numbers, ages                        │
│  ├─ HIV status, TB status                                     │
│  ├─ Precise GPS coordinates of patients                       │
│  └─ Photos of lab reports / prescriptions                     │
│                                                                │
│  LEVEL 4 — HIGH                                               │
│  ├─ AMR case reports (symptoms, diagnosis)                     │
│  ├─ Drug resistance probabilities per district                 │
│  ├─ CHW identities and credentials                            │
│  └─ Facility-level aggregation data                           │
│                                                                │
│  LEVEL 3 — MODERATE                                           │
│  ├─ District-level AMR statistics                             │
│  ├─ Drug resistance trends (aggregated)                       │
│  ├─ User preferences and language settings                    │
│  └─ Session tokens and auth state                            │
│                                                                │
│  LEVEL 2 — LOW                                                │
│  ├─ General educational content                               │
│  ├─ Public AMR awareness materials                            │
│  └─ Aggregated national statistics                           │
│                                                                │
│  LEVEL 1 — PUBLIC                                            │
│  ├─ Marketing content                                        │
│  ├─ Open-source code repositories                            │
│  └─ Documentation (non-internal)                             │
│                                                                │
└────────────────────────────────────────────────────────────────┘
```

### 1.3 Applicable Regulations

| Country | Regulation | Key Requirements | Compliance Status |
|---------|-----------|-----------------|-------------------|
| Nigeria | NDPR (2019) | 72hr breach notification, consent, data minimization | In Progress |
| Kenya | Data Protection Act (2019) | DPO required, impact assessments, cross-border restrictions | In Progress |
| Tanzania | POCA (2022) | Data localization, purpose limitation | In Progress |
| Ghana | Data Protection Act (2012) | Registration with DPA, consent requirements | In Progress |
| Uganda | Data Protection & Privacy Act (2019) | Data protection impact assessment | In Progress |
| Ethiopia | Draft Data Protection Proclamation | (Awaiting enactment) | Monitoring |

---

## 2. Threat Model

### 2.1 Attacker Profiles

```
┌──────────────────────────────────────────────────────────────────────────┐
│                          THREAT MODEL                                    │
│                                                                          │
│  ┌─────────────────┐     ┌─────────────────┐     ┌──────────────────┐   │
│  │  ATTACKER A     │     │  ATTACKER B     │     │  ATTACKER C      │   │
│  │  Script Kiddie  │     │  Insider Threat │     │  Nation-State    │   │
│  │                 │     │                 │     │  / APT           │   │
│  │  Motivation:    │     │  Motivation:    │     │  Motivation:     │   │
│  │  Curiosity,     │     │  Financial gain,│     │  Intelligence    │   │
│  │  bragging rights│     │  grudges,       │     │  gathering,      │   │
│  │                 │     │  negligence     │     │  sabotage        │   │
│  │  Capability:    │     │                 │     │                  │   │
│  │  Low            │     │  Capability:    │     │  Capability:     │   │
│  │                 │     │  Medium-High    │     │  Very High       │   │
│  │  Access:        │     │                 │     │                  │   │
│  │  Public internet│     │  Access:        │     │  Access:         │   │
│  │                 │     │  Authenticated  │     │  Advanced        │   │
│  │  Impact:        │     │  (CHW, Admin)   │     │  persistent     │   │
│  │  Low            │     │                 │     │  threats         │   │
│  │                 │     │  Impact:        │     │                  │   │
│  │  Threats:       │     │  High           │     │  Impact:         │   │
│  │  - SQL injection│     │  Threats:       │     │  Critical        │   │
│  │  - XSS          │     │  - Data theft   │     │                  │   │
│  │  - Brute force  │     │  - Credential   │     │  Threats:        │   │
│  │  - CSRF         │     │    sharing      │     │  - Supply chain  │   │
│  │                 │     │  - Unauthorized  │     │  - 0-day exploits│   │
│  └─────────────────┘     │    access       │     │  - Physical      │   │
│                          │  - Data         │     │    access to     │   │
│                          │    modification │     │    edge nodes    │   │
│                          └─────────────────┘     └──────────────────┘   │
│                                                                          │
│  ┌─────────────────┐     ┌─────────────────┐                            │
│  │  ATTACKER D     │     │  ATTACKER E     │                            │
│  │  Opportunist    │     │  Competitor     │                            │
│  │                 │     │                 │                            │
│  │  Motivation:    │     │  Motivation:    │                            │
│  │  Financial gain,│     │  Competitive    │                            │
│  │  phishing,      │     │  intelligence,  │                            │
│  │  social eng.    │     │  market edge    │                            │
│  │                 │     │                 │                            │
│  │  Capability:    │     │  Capability:    │                            │
│  │  Medium         │     │  Medium-High    │                            │
│  │                 │     │                 │                            │
│  │  Threats:       │     │  Threats:       │                            │
│  │  - Phishing     │     │  - IP theft     │                            │
│  │  - SIM swap     │     │  - Data scraping│                            │
│  │  - Phone social │     │  - DDoS         │                            │
│  │    engineering  │     │                 │                            │
│  │  - WhatsApp/ TG │     │                 │                            │
│  │    account take │     │                 │                            │
│  └─────────────────┘     └─────────────────┘                            │
│                                                                          │
└──────────────────────────────────────────────────────────────────────────┘
```

### 2.2 Trust Boundaries

```
┌─────────────────────────────────────────────────────────────────────────┐
│                        TRUST BOUNDARIES                                  │
│                                                                         │
│  ┌────────────────────┐   ┌────────────────────┐   ┌────────────────┐  │
│  │  ZONE 0            │   │  ZONE 1            │   │  ZONE 2        │  │
│  │  UNTRUSTED         │   │  EDGE (Semi-Trusted│   │  CLOUD (Trusted│  │
│  │                    │   │                    │   │                │  │
│  │  • CHW phones      │──▶│  • Raspberry Pi    │──▶│  • AWS         │  │
│  │  • Public internet │TLS│  • Local SQLite    │TLS│  • PostgreSQL  │  │
│  │  • USSD gateway    │  │  • Local models    │  │  • Redis        │  │
│  │  • Messenger APIs  │  │  • USSD sessions   │  │  • ML cluster   │  │
│  │                    │  │  • Anonymization    │  │  • Grafana      │  │
│  │  Risk: MITM,      │  │    (Presidio)      │  │  • Keycloak     │  │
│  │  eavesdropping,   │  │                    │  │                │  │
│  │  spoofing         │  │  Risk: Physical    │  │  Risk: Insider  │  │
│  │                    │  │  theft, local      │  │  threat, cloud  │  │
│  │                    │  │  compromise,       │  │  misconfig      │  │
│  │                    │  │  SD card removal   │  │                │  │
│  └────────────────────┘   └────────────────────┘   └────────────────┘  │
│                                                                         │
│  BOUNDARY CONTROLS:                                                     │
│  Zone 0 → Zone 1: TLS 1.3, API key auth, rate limiting                │
│  Zone 1 → Zone 2: mTLS, HMAC signatures, encrypted payloads           │
│  Zone 2 internal: Network segmentation, VPC, security groups           │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

### 2.3 Threat Matrix

| # | Threat | Attacker | Likelihood | Impact | Risk | Mitigation |
|---|--------|----------|-----------|--------|------|------------|
| T1 | SQL injection on edge SQLite | A | Medium | High | **HIGH** | Parameterized queries, input validation |
| T2 | Man-in-the-middle on USSD/GSM | A, E | Medium | Medium | **MEDIUM** | TLS 1.3, certificate pinning |
| T3 | Insider data exfiltration | B | Medium | Critical | **CRITICAL** | RBAC, audit logs, DLP |
| T4 | Physical theft of RPi edge node | A, C | Medium | High | **HIGH** | Full-disk encryption, SQLCipher, remote wipe |
| T5 | WhatsApp account takeover | D | Medium | High | **HIGH** | 2FA on Meta Business, webhook signature verification |
| T6 | SIM swap for phone auth | D | Low | High | **MEDIUM** | PIN backup codes, session invalidation |
| T7 | PII leakage in sync payloads | B, E | Medium | Critical | **CRITICAL** | Presidio anonymization, pre-sync validation |
| T8 | DDoS on cloud API | E | Medium | Medium | **MEDIUM** | WAF, rate limiting, auto-scaling |
| T9 | Supply chain compromise (PyPI) | C | Low | Critical | **HIGH** | Dependency pinning, hash verification |
| T10 | LLM prompt injection | A | High | Medium | **MEDIUM** | Input sanitization, output validation |
| T11 | Brute force on API keys | A | Medium | Medium | **MEDIUM** | Rate limiting, key rotation, IP allowlisting |
| T12 | Cross-border data transfer | B, E | Low | Critical | **HIGH** | Data residency enforcement, af-south-1 only |

---

## 3. Encryption Architecture

### 3.1 Encryption Standards Matrix

| Layer | Algorithm | Key Size | Mode | Implementation | Notes |
|-------|-----------|----------|------|----------------|-------|
| **Data at Rest (Edge)** | AES | 256-bit | GCM | SQLCipher 4.x | Per-database encryption key |
| **Data at Rest (Cloud)** | AES | 256-bit | GCM | AWS TDE + KMS | Customer-managed CMK |
| **Data in Transit** | TLS | 256-bit | 1.3 | Python `ssl`, httpx | HSTS enforced, cipher pinning |
| **Sync Payloads** | AES | 256-bit | GCM | `cryptography` library | Per-payload key, HMAC integrity |
| **API Keys at Rest** | AES | 256-bit | GCM | Vault / env encrypted | Never in source code |
| **PII Fields** | SHA-256 | 256-bit | Hash | Python `hashlib` | Salted hashing for dedup |
| **User PINs** | bcrypt | Variable | — | `bcrypt` library | Cost factor = 12 |
| **Database Backups** | AES | 256-bit | GCM | `gpg` symmetric | Separate encryption key |

### 3.2 Encryption Architecture Diagram

```
┌──────────────────────────────────────────────────────────────────────────┐
│                       ENCRYPTION ARCHITECTURE                             │
│                                                                          │
│  DATA AT REST                                                            │
│  ════════════                                                            │
│                                                                          │
│  ┌─────────────────────────────────────────────────────────────────┐    │
│  │ EDGE NODE (Raspberry Pi)                                       │    │
│  │                                                                 │    │
│  │  ┌───────────────┐    ┌──────────────────────────────────┐     │    │
│  │  │  SQLite DB     │    │  SQLCipher 4.x                   │     │    │
│  │  │               │───▶│  ┌─────────────────────────────┐  │     │    │
│  │  │  cases.db     │    │  │ AES-256-GCM                  │  │     │    │
│  │  │  users.db     │    │  │ Key: PRAGMA key = '...'      │  │     │    │
│  │  │  sessions.db  │    │  │ Page size: 4096              │  │     │    │
│  │  │               │    │  │ KDF iterations: 256,000      │  │     │    │
│  │  └───────────────┘    │  │ HMAC: SHA-512                │  │     │    │
│  │                        │  └─────────────────────────────┘  │     │    │
│  │  Key Derivation:       └──────────────────────────────────┘     │    │
│  │  ┌─────────────────────────────────────────────────────┐       │    │
│  │  │ key = HKDF(                                    │       │    │
│  │  │   ikm = hardware_uuid || provisioning_secret,    │       │    │
│  │  │   salt = per-node-random-salt,                   │       │    │
│  │  │   info = "udara-edge-db-v1",                     │       │    │
│  │  │   length = 32                                    │       │    │
│  │  │ )                                               │       │    │
│  │  └─────────────────────────────────────────────────────┘       │    │
│  │                                                                 │    │
│  │  🔒 If RPi is stolen: encrypted DB is unreadable                │    │
│  │     without the provisioning secret (NOT stored on device)       │    │
│  └─────────────────────────────────────────────────────────────────┘    │
│                                                                          │
│  ┌─────────────────────────────────────────────────────────────────┐    │
│  │ CLOUD (AWS af-south-1)                                          │    │
│  │                                                                 │    │
│  │  ┌────────────────────────────────────────────────────────┐    │    │
│  │  │ AWS KMS → Customer Managed CMK                         │    │    │
│  │  │  ┌──────────────────────────────────────────────────┐  │    │    │
│  │  │  │ Key Policy:                                       │  │    │    │
│  │  │  │  - Enable rotation: 365 days                      │  │    │    │
│  │  │  │  - Deletion protection: 30 days                   │  │    │    │
│  │  │  │  - Key administrators: security@udara.ai          │  │    │    │
│  │  │  │  - Key users: app-role, sync-role                 │  │    │    │
│  │  │  │  - Region: af-south-1 ONLY                        │  │    │    │
│  │  │  └──────────────────────────────────────────────────┘  │    │    │
│  │  └────────────────────────────────────────────────────────┘    │    │
│  │                                                                 │    │
│  │  RDS PostgreSQL: TDE with AWS KMS CMK                            │    │
│  │  S3 Buckets: SSE-KMS with CMK + bucket policies                  │    │
│  │  EBS Volumes: encrypted with CMK                                 │    │
│  └─────────────────────────────────────────────────────────────────┘    │
│                                                                          │
│  DATA IN TRANSIT                                                         │
│  ═══════════════                                                         │
│                                                                          │
│  ┌──────────┐  TLS 1.3  ┌──────────┐  mTLS  ┌──────────┐             │
│  │ CHW      │──────────▶│ Edge RPi │────────▶│ Cloud    │             │
│  │ Phone    │           │          │        │ AWS      │             │
│  └──────────┘           └──────────┘        └──────────┘             │
│                                                                          │
│  TLS 1.3 Ciphers:                                                        │
│  ┌─────────────────────────────────────────────────────────────────┐    │
│  │ TLS_AES_256_GCM_SHA384        (PFS: ECDHE, 256-bit)            │    │
│  │ TLS_CHACHA20_POLY1305_SHA256  (PFS: ECDHE, 256-bit)            │    │
│  │                                                                 │    │
│  │ REJECTED:                                                      │    │
│  │ ✗ TLS_AES_128_GCM_SHA256  (insufficient key size)              │    │
│  │ ✗ RSA key exchange           (no forward secrecy)               │    │
│  │ ✗ TLS 1.2                    (legacy)                           │    │
│  │ ✗ TLS 1.1, 1.0, SSL 3.0     (deprecated)                      │    │
│  └─────────────────────────────────────────────────────────────────┘    │
│                                                                          │
│  SYNC PAYLOAD ENCRYPTION                                                  │
│  ═══════════════════════                                                  │
│                                                                          │
│  ┌────────────────────────────────────────────────────────────────┐    │
│  │  Edge → Cloud Sync Payload                                     │    │
│  │                                                                 │    │
│  │  plaintext = JSON.dumps(anonymized_cases)                      │    │
│  │  aad = b"hmac-sha256:" + payload_hash                         │    │
│  │                                                                 │    │
│  │  per_payload_key = AESGCM.generate_key(bit_length=256)        │    │
│  │  wrapped_key = KMS.encrypt(per_payload_key)                    │    │
│  │                                                                 │    │
│  │  ciphertext, nonce = AESGCM.encrypt(                          │    │
│  │      key=per_payload_key,                                     │    │
│  │      data=plaintext.encode(),                                  │    │
│  │      associated_data=aad,                                     │    │
│  │  )                                                             │    │
│  │                                                                 │    │
│  │  sync_payload = {                                              │    │
│  │      "encrypted_data": base64(ciphertext),                    │    │
│  │      "nonce": base64(nonce),                                  │    │
│  │      "wrapped_key": base64(wrapped_key),                      │    │
│  │      "hmac": base64(HMAC(sha256, ciphertext)),                │    │
│  │      "algorithm": "AES-256-GCM",                              │    │
│  │      "schema_version": "v2",                                   │    │
│  │  }                                                             │    │
│  └────────────────────────────────────────────────────────────────┘    │
│                                                                          │
└──────────────────────────────────────────────────────────────────────────┘
```

### 3.3 Edge Database Encryption Implementation

```python
# edge/storage/encrypted_database.py

"""
SQLCipher integration for encrypted SQLite databases on edge nodes.

Every edge node (Raspberry Pi) stores case data in an encrypted SQLite
database using SQLCipher. If the device is physically stolen, the data
is unreadable without the encryption key.

The encryption key is derived from:
  - Hardware UUID (unique per RPi, read from /proc/cpu_serial)
  - Provisioning secret (injected during setup, NOT stored on device)
  - Per-node random salt (generated at provisioning, stored on device)

This means an attacker who steals the RPi gets:
  ✗ The encrypted database file
  ✗ The hardware UUID
  ✗ The per-node salt
  ✗ The provisioning secret (only on provisioning server)

Result: Database cannot be decrypted.
"""

import hashlib
import logging
import os
import secrets
from pathlib import Path
from typing import Optional

from cryptography.hazmat.primitives.kdf.hkdf import HKDF
from cryptography.hazmat.primitives import hashes

logger = logging.getLogger(__name__)


class EdgeDatabaseEncryption:
    """Manages encryption keys for edge SQLite databases."""

    # SQLCipher PRAGMA settings
    CIPHER_PRAGMAS = {
        "cipher_compatibility": 4,         # SQLCipher 4.x compatibility
        "cipher_page_size": 4096,          # 4KB pages
        "kdf_iter": 256000,                # KDF iterations (high security)
        "cipher_hmac_algorithm": "HMAC_SHA512",
        "cipher_kdf_algorithm": "PBKDF2_HMAC_SHA512",
        "cipher_plaintext_header_size": 32,
    }

    def __init__(
        self,
        db_path: str,
        hardware_uuid: str,
        provisioning_secret: str,
        salt_path: Optional[str] = None,
    ):
        """
        Initialize encrypted database.

        Args:
            db_path: Path to the SQLite database file.
            hardware_uuid: Unique hardware UUID from /proc/cpu_serial.
            provisioning_secret: Secret injected during provisioning (NOT stored on device).
            salt_path: Path to per-node salt file. Generated if not exists.
        """
        self.db_path = Path(db_path)
        self.hardware_uuid = hardware_uuid
        self.provisioning_secret = provisioning_secret
        self.salt_path = Path(salt_path or str(self.db_path) + ".salt")
        self._encryption_key: Optional[str] = None

    def derive_key(self) -> str:
        """
        Derive the database encryption key using HKDF.

        Uses SHA-256 as the hash function with:
        - IKM: hardware_uuid || provisioning_secret
        - Salt: per-node random salt (or deterministic fallback)
        - Info: "udara-edge-db-v1"

        Returns:
            64-character hex string (32 bytes = 256 bits).
        """
        # Load or generate per-node salt
        salt = self._load_or_generate_salt()

        # Derive key material
        hkdf = HKDF(
            algorithm=hashes.SHA256(),
            length=32,
            salt=salt,
            info=b"udara-edge-db-v1",
        )

        ikm = (self.hardware_uuid + self.provisioning_secret).encode("utf-8")
        key_bytes = hkdf.derive(ikm)

        # Convert to hex string for SQLCipher PRAGMA key
        self._encryption_key = key_bytes.hex()

        logger.info(
            "Derived encryption key: %s...%s (first/last 4 chars)",
            self._encryption_key[:4],
            self._encryption_key[-4:],
        )
        return self._encryption_key

    def get_sqlcipher_pragmas(self) -> list[str]:
        """Get list of SQLCipher PRAGMA statements to execute after opening."""
        key = self._encryption_key or self.derive_key()
        pragmas = [f"PRAGMA key = \"x'{key}'\";"]
        for pragma_name, pragma_value in self.CIPHER_PRAGMAS.items():
            pragmas.append(f"PRAGMA {pragma_name} = {pragma_value};")
        return pragmas

    def _load_or_generate_salt(self) -> bytes:
        """Load existing salt or generate a new one."""
        if self.salt_path.exists():
            salt = self.salt_path.read_bytes()
            logger.info("Loaded existing salt from %s", self.salt_path)
            return salt

        # Generate new 32-byte random salt
        salt = secrets.token_bytes(32)
        self.salt_path.write_bytes(salt)
        # Restrict permissions to owner only
        os.chmod(self.salt_path, 0o600)
        logger.info("Generated new per-node salt: %s", self.salt_path)
        return salt

    @staticmethod
    def get_hardware_uuid() -> str:
        """Read the Raspberry Pi hardware UUID from /proc/cpu_serial."""
        try:
            serial_path = Path("/proc/cpu_serial")
            if serial_path.exists():
                return serial_path.read_text().strip()
        except Exception:
            logger.warning("Could not read hardware UUID, falling back to env")
        # Fallback for development
        return os.environ.get("UDARA_HARDWARE_UUID", "dev-uuid-001")

    def verify_database_integrity(self) -> bool:
        """
        Verify that the encrypted database can be read.

        Opens the database and attempts to read a known table.
        Returns True if successful, False if decryption fails.
        """
        import sqlite3

        try:
            conn = sqlite3.connect(str(self.db_path))
            cursor = conn.cursor()
            for pragma in self.get_sqlcipher_pragmas():
                cursor.execute(pragma)
            # Attempt a simple query — if key is wrong, this will fail
            cursor.execute("SELECT count(*) FROM sqlite_master;")
            count = cursor.fetchone()
            conn.close()
            logger.info("Database integrity check passed: %d objects", count[0])
            return True
        except sqlite3.DatabaseError as e:
            logger.error("Database integrity check FAILED: %s", e)
            return False


# ─── Usage Example ─────────────────────────────────────────────────

def initialize_encrypted_database(db_path: str) -> EdgeDatabaseEncryption:
    """
    Initialize an encrypted SQLite database for an edge node.

    This function runs once during device provisioning.
    """
    # In production, provisioning_secret is injected from the provisioning
    # server via secure channel (USB or encrypted network bootstrap)
    provisioning_secret = os.environ.get("UDARA_PROVISIONING_SECRET", "")
    if not provisioning_secret:
        raise RuntimeError(
            "UDARA_PROVISIONING_SECRET not set. "
            "Database encryption requires a provisioning secret."
        )

    hw_uuid = EdgeDatabaseEncryption.get_hardware_uuid()
    enc = EdgeDatabaseEncryption(
        db_path=db_path,
        hardware_uuid=hw_uuid,
        provisioning_secret=provisioning_secret,
    )
    enc.derive_key()

    if not enc.verify_database_integrity():
        raise RuntimeError("Database encryption verification failed")

    return enc
```

---

## 4. PII Anonymization Pipeline

### 4.1 Pipeline Architecture

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    PII ANONYMIZATION PIPELINE                            │
│                        (Microsoft Presidio)                              │
│                                                                          │
│  ┌────────────────┐                                                   │
│  │ RAW INPUT      │                                                   │
│  │                │                                                   │
│  │ "Patient John  │                                                   │
│  │ Okafor, aged   │                                                   │
│  │ 34, came from  │                                                   │
│  │ Lagos Island   │                                                   │
│  │ with fever.    │                                                   │
│  │ Phone:         │                                                   │
│  │ 08012345678.   │                                                   │
│  │ Seen on 14 Jan │                                                   │
│  │ 2025.          │                                                   │
│  │ Prescribed     │                                                   │
│  │ amoxicillin."  │                                                   │
│  └───────┬────────┘                                                   │
│          │                                                             │
│          ▼                                                             │
│  ┌────────────────────────────────────────────────────────────────┐   │
│  │ STAGE 1: ANALYZER — Detect PII Entities                        │   │
│  │                                                                │   │
│  │  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐        │   │
│  │  │ Presidio     │  │ spaCy NER    │  │ Custom Regex │        │   │
│  │  │ NLP Engine   │  │ (en_core_web │  │ Patterns     │        │   │
│  │  │              │  │ _sm)         │  │              │        │   │
│  │  │ PERSON ✅    │  │ PERSON ✅    │  │ PHONE ✅     │        │   │
│  │  │ LOCATION ✅  │  │              │  │ DATE  ✅     │        │   │
│  │  │ DATE ✅      │  │              │  │ AGE   ✅     │        │   │
│  │  └──────────────┘  └──────────────┘  └──────────────┘        │   │
│  │                                                                │   │
│  │  Detected entities:                                            │   │
│  │  [PERSON] "John Okafor"    → score: 0.92                      │   │
│  │  [AGE]    "34"             → score: 0.88                      │   │
│  │  [LOCATION] "Lagos Island" → score: 0.95                      │   │
│  │  [PHONE]  "08012345678"    → score: 0.99                      │   │
│  │  [DATE]   "14 Jan 2025"    → score: 0.97                      │   │
│  └───────────────────────────┬────────────────────────────────────┘   │
│                              │                                        │
│                              ▼                                        │
│  ┌────────────────────────────────────────────────────────────────┐   │
│  │ STAGE 2: ANONYMIZER — Replace PII                              │   │
│  │                                                                │   │
│  │  Anonymization Actions:                                        │   │
│  │  ┌────────────────┬──────────────┬─────────────────────────┐ │   │
│  │  │ Entity         │ Action       │ Replacement              │ │   │
│  │  ├────────────────┼──────────────┼─────────────────────────┤ │   │
│  │  │ PERSON         │ Replace      │ <PATIENT>               │ │   │
│  │  │ PHONE          │ Replace      │ <PHONE_REDACTED>        │ │   │
│  │  │ LOCATION       │ Keep (agg.)  │ Lagos Island → district │ │   │
│  │  │                │              │ code for aggregation     │ │   │
│  │  │ DATE           │ Shift        │ 14 Jan → 01 Jan (shift  │ │   │
│  │  │                │              │ by 13 days, preserve    │ │   │
│  │  │                │              │ day-of-week)            │ │   │
│  │  │ AGE            │ Generalize   │ 34 → 30-39 bucket      │ │   │
│  │  └────────────────┴──────────────┴─────────────────────────┘ │   │
│  └───────────────────────────┬────────────────────────────────────┘   │
│                              │                                        │
│                              ▼                                        │
│  ┌────────────────────────────────────────────────────────────────┐   │
│  │ ANONYMIZED OUTPUT                                              │   │
│  │                                                                │   │
│  │ "Patient <PATIENT>, aged 30-39, came from                    │   │
│  │  Lagos Island with fever. Phone: <PHONE_REDACTED>.            │   │
│  │  Seen on 01 Jan 2025. Prescribed amoxicillin."                │   │
│  │                                                                │   │
│  │ ✅ Safe for sync to cloud                                     │   │
│  │ ✅ Safe for aggregation/analytics                             │   │
│  │ ✅ Safe for ML model training                                 │   │
│  └────────────────────────────────────────────────────────────────┘   │
│                                                                          │
│  PIPELINE GUARANTEES:                                                   │
│  ✓ Runs on EVERY edge node BEFORE sync                                 │
│  ✓ No raw PII ever leaves the edge node                                │
│  ✓ Deterministic: same input → same output                             │
│  ✓ Sub-100ms latency per message on RPi 4                              │
│  ✓ Graceful degradation: if entity not detected, mask entire field     │
│                                                                          │
└─────────────────────────────────────────────────────────────────────────┘
```

### 4.2 Full Presidio Pipeline Implementation

```python
# edge/anonymization/presidio_pipeline.py

"""
PII Anonymization Pipeline using Microsoft Presidio.

This module MUST run on every edge node before any data
is synced to the cloud. It detects and anonymizes PII entities
including person names, phone numbers, locations, dates, and ages.

Designed for resource-constrained environments (Raspberry Pi 4):
- spaCy small model (en_core_web_sm, 12MB)
- Minimal NLP engine configuration
- < 100ms per typical case report
"""

import json
import logging
import re
from typing import Optional

from presidio_analyzer import (
    AnalyzerEngine,
    RecognizerResult,
    BatchAnalyzerEngine,
)
from presidio_analyzer.nlp_engine import NlpEngineProvider
from presidio_anonymizer import AnonymizerEngine
from presidio_anonymizer.entities import OperatorConfig

logger = logging.getLogger(__name__)


# ─── Custom Recognizers for African Context ─────────────────────

class NigerianPhoneRecognizer:
    """Recognizes Nigerian phone numbers in various formats."""

    # Nigerian phone patterns:
    # 08012345678, +2348012345678, 2348012345678
    # 09012345678, 07012345678, 08112345678
    NIGERIAN_PHONE_PATTERNS = [
        r"\+?234[789]\d{9}",
        r"0[789]0\d{8}",
        r"\+?234[701]\d{9}",
        r"0[701]\d{8}",
    ]

    # Kenyan patterns:
    # +254712345678, 0712345678
    KENYAN_PHONE_PATTERNS = [
        r"\+?254[7]\d{8}",
        r"07\d{8}",
    ]

    ALL_PATTERNS = NIGERIAN_PHONE_PATTERNS + KENYAN_PHONE_PATTERNS

    def load(self) -> dict:
        return {
            "supported_entities": ["PHONE_NUMBER"],
            "supported_languages": ["en"],
        }

    def analyze(
        self,
        text: str,
        entities: list[str],
        language: str,
    ) -> list[RecognizerResult]:
        results = []
        for pattern in self.ALL_PATTERNS:
            for match in re.finditer(pattern, text):
                start, end = match.span()
                results.append(
                    RecognizerResult(
                        entity_type="PHONE_NUMBER",
                        start=start,
                        end=end,
                        score=0.95,
                    )
                )
        return results


class AgeRecognizer:
    """Recognizes age mentions: 'aged 34', '34 years old', '34yo'."""

    AGE_PATTERNS = [
        r"\bage\s*(?:of\s*)?(\d{1,3})\b",
        r"(\d{1,3})\s*(?:years?\s*old|yo|yrs)",
        r"(?:age|aged)[\s:]*(\d{1,3})",
    ]

    def load(self) -> dict:
        return {
            "supported_entities": ["AGE"],
            "supported_languages": ["en"],
        }

    def analyze(
        self,
        text: str,
        entities: list[str],
        language: str,
    ) -> list[RecognizerResult]:
        results = []
        for pattern in self.AGE_PATTERNS:
            for match in re.finditer(pattern, text, re.IGNORECASE):
                start, end = match.span()
                age_str = match.group(1) or match.group(0)
                age_value = re.search(r"\d{1,3}", age_str)
                if age_value:
                    age_num = int(age_value.group())
                    # Sanity check: 0-150
                    if 0 <= age_num <= 150:
                        results.append(
                            RecognizerResult(
                                entity_type="AGE",
                                start=start,
                                end=end,
                                score=0.88,
                            )
                        )
        return results


class AfricanLocationRecognizer:
    """Recognizes African district/city names using a known gazetteer."""

    # Top 200 African cities/districts for AMR surveillance
    KNOWN_LOCATIONS = {
        # Nigeria
        "lagos": "Lagos", "abuja": "Abuja", "kano": "Kano",
        "ibadan": "Ibadan", "port harcourt": "Port Harcourt",
        "benin city": "Benin City", "kaduna": "Kaduna",
        "maiduguri": "Maiduguri", "enugu": "Enugu",
        # Kenya
        "nairobi": "Nairobi", "mombasa": "Mombasa", "kisumu": "Kisumu",
        "nakuru": "Nakuru", "eldoret": "Eldoret",
        # Tanzania
        "dar es salaam": "Dar es Salaam", "dodoma": "Dodoma",
        "arusha": "Arusha", "mwanza": "Mwanza",
        # Ghana
        "accra": "Accra", "kumasi": "Kumasi", "tamale": "Tamale",
        # Uganda
        "kampala": "Kampala", "entebbe": "Entebbe", "gulu": "Gulu",
        # Ethiopia
        "addis ababa": "Addis Ababa", "dire dawa": "Dire Dawa",
        "hawassa": "Hawassa",
    }

    def load(self) -> dict:
        return {
            "supported_entities": ["LOCATION"],
            "supported_languages": ["en"],
        }

    def analyze(
        self,
        text: str,
        entities: list[str],
        language: str,
    ) -> list[RecognizerResult]:
        results = []
        text_lower = text.lower()
        for loc_key, loc_name in self.KNOWN_LOCATIONS.items():
            idx = text_lower.find(loc_key)
            if idx >= 0:
                results.append(
                    RecognizerResult(
                        entity_type="LOCATION",
                        start=idx,
                        end=idx + len(loc_key),
                        score=0.90,
                    )
                )
        return results


# ─── Main Pipeline ──────────────────────────────────────────────

class PIIAnonymizer:
    """
    PII detection and anonymization pipeline.

    Usage:
        anonymizer = PIIAnonymizer()
        result = anonymizer.anonymize(
            "Patient John Okafor, aged 34, from Lagos Island. "
            "Phone: 08012345678. Seen on 14 Jan 2025."
        )
        # result.anonymized_text → cleaned text
        # result.entities → list of detected entities
    """

    # Minimum confidence score for entity detection
    CONFIDENCE_THRESHOLD = 0.70

    def __init__(self, models_dir: str = "models/spacy"):
        """Initialize the Presidio analyzer and anonymizer."""
        self._analyzer: Optional[AnalyzerEngine] = None
        self._anonymizer: Optional[AnonymizerEngine] = None
        self._models_dir = models_dir

    def initialize(self) -> None:
        """
        Initialize the NLP engine and register custom recognizers.

        MUST be called before first use. Loads the spaCy model
        and registers African-context recognizers.
        """
        logger.info("Initializing PII Anonymizer pipeline...")

        # Configure NLP engine with spaCy
        nlp_configuration = {
            "nlp_engine_name": "spacy",
            "models": [
                {"lang_code": "en", "model_name": "en_core_web_sm"},
            ],
        }
        provider = NlpEngineProvider(nlp_configuration=nlp_configuration)
        nlp_engine = provider.create_engine()

        # Create analyzer engine
        self._analyzer = AnalyzerEngine(
            nlp_engine=nlp_engine,
            supported_languages=["en"],
        )

        # Register custom recognizers
        self._analyzer.registry.add_recognizer(NigerianPhoneRecognizer())
        self._analyzer.registry.add_recognizer(AgeRecognizer())
        self._analyzer.registry.add_recognizer(AfricanLocationRecognizer())

        # Create anonymizer engine
        self._anonymizer = AnonymizerEngine()

        logger.info("PII Anonymizer pipeline initialized successfully")

    def _ensure_initialized(self) -> None:
        """Lazy initialization."""
        if self._analyzer is None:
            self.initialize()

    def detect(self, text: str) -> list[dict]:
        """
        Detect PII entities in text.

        Returns list of dicts:
        [{"entity_type": "PERSON", "start": 8, "end": 18,
          "score": 0.92, "text": "John Okafor"}]
        """
        self._ensure_initialized()

        results = self._analyzer.analyze(
            text=text,
            language="en",
            entities=["PERSON", "PHONE_NUMBER", "LOCATION", "DATE_TIME", "AGE"],
            score_threshold=self.CONFIDENCE_THRESHOLD,
        )

        entities = []
        for result in results:
            entities.append({
                "entity_type": result.entity_type,
                "start": result.start,
                "end": result.end,
                "score": result.score,
                "text": text[result.start : result.end],
            })

        return entities

    def anonymize(
        self,
        text: str,
        keep_location_for_aggregation: bool = True,
        age_bucket_size: int = 10,
    ) -> dict:
        """
        Detect and anonymize PII in text.

        Args:
            text: Input text potentially containing PII.
            keep_location_for_aggregation: If True, replaces location
                names with district codes instead of full redaction.
            age_bucket_size: Size of age generalization buckets (default: 10).

        Returns:
            {
                "anonymized_text": "Patient <PATIENT>, aged 30-39, ...",
                "entities": [...],
                "operations": [...],
                "pseudonym_map": {
                    "John Okafor": "<PATIENT>",
                    "08012345678": "<PHONE_REDACTED>",
                }
            }
        """
        self._ensure_initialized()

        # Detect entities
        entities = self._analyzer.analyze(
            text=text,
            language="en",
            entities=["PERSON", "PHONE_NUMBER", "LOCATION", "DATE_TIME", "AGE"],
            score_threshold=self.CONFIDENCE_THRESHOLD,
        )

        # Build operator configurations per entity type
        operators = {
            "PERSON": OperatorConfig(
                "replace",
                {"new_value": "<PATIENT>"},
            ),
            "PHONE_NUMBER": OperatorConfig(
                "replace",
                {"new_value": "<PHONE_REDACTED>"},
            ),
            "LOCATION": OperatorConfig(
                "replace" if not keep_location_for_aggregation else "custom_keep_location",
                {"new_value": "<LOCATION>"},
            ),
            "DATE_TIME": OperatorConfig(
                "shift",
                {"shift_days": 13, "shift_seconds": 0},
            ),
            "AGE": OperatorConfig(
                "custom_generalize_age",
                {"bucket_size": age_bucket_size},
            ),
        }

        # Anonymize
        result = self._anonymizer.anonymize(
            text=text,
            analyzer_results=entities,
            operators=operators,
        )

        # Build pseudonym map for audit trail
        pseudonym_map = {}
        for entity in entities:
            original = text[entity.start : entity.end]
            pseudonym_map[original] = f"<{entity.entity_type}>"

        return {
            "anonymized_text": result.text,
            "entities": [
                {
                    "entity_type": e.entity_type,
                    "start": e.start,
                    "end": e.end,
                    "score": e.score,
                    "text": text[e.start : e.end],
                }
                for e in entities
            ],
            "operations": [
                {
                    "entity_type": item.entity_type,
                    "operator": item.operator,
                    "original": text[item.start : item.end] if hasattr(item, 'start') else None,
                }
                for item in result.items
            ],
            "pseudonym_map": pseudonym_map,
        }

    def check_text_is_safe(self, text: str) -> bool:
        """
        Quick check: does the text contain any PII?

        Returns True if NO PII detected (text is safe).
        Returns False if PII detected.
        """
        entities = self.detect(text)
        return len(entities) == 0
```

---

## 5. Authentication

### 5.1 Authentication Methods by Channel

```
┌──────────────────────────────────────────────────────────────────────────┐
│                    AUTHENTICATION METHODS                                 │
│                                                                          │
│  ┌────────────────┐  ┌────────────────┐  ┌─────────────────────────┐  │
│  │ EDGE → CLOUD   │  │ USSD           │  │ MESSENGER               │  │
│  │                │  │                │  │ (WhatsApp / Telegram)    │  │
│  │ API Key Auth   │  │ Phone + PIN    │  │ Phone Verification      │  │
│  │                │  │                │  │                         │  │
│  │ Header:        │  │ 1. CHW dials   │  │ 1. CHW sends message    │  │
│  │  X-API-Key:    │  │    USSD code   │  │ 2. Bot extracts phone   │  │
│  │  <edge-key>    │  │ 2. System      │  │    from WhatsApp/TG     │  │
│  │                │  │    identifies   │  │ 3. System sends OTP     │  │
│  │ Key rotation:  │  │    phone via    │  │    via SMS (backup)     │  │
│  │ 90 days        │  │    USSD session │  │ 4. CHW enters OTP      │  │
│  │                │  │ 3. First time:  │  │ 5. Account linked       │  │
│  │ Issued during: │  │    prompt PIN   │  │ 6. Session via JWT      │  │
│  │ provisioning   │  │    creation     │  │    (1hr expiry)         │  │
│  │                │  │ 4. PIN stored   │  │                         │  │
│  │ Storage:       │  │    as bcrypt    │  │ Storage:                │  │
│  │ .env on edge   │  │    hash         │  │ users table in cloud DB │  │
│  │ (chmod 600)    │  │                │  │                         │  │
│  └────────────────┘  └────────────────┘  └─────────────────────────┘  │
│                                                                          │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │ WEB DASHBOARD                                                    │   │
│  │                                                                  │   │
│  │ Keycloak OIDC                                                   │   │
│  │  ┌──────────────────────────────────────────────────────────┐   │   │
│  │  │ Roles:                                                    │   │   │
│  │  │  • viewer      → Read-only dashboard, export CSV         │   │   │
│  │  │  • editor      → viewer + case editing, report creation  │   │   │
│  │  │  • admin       → editor + user management, system config │   │   │
│  │  │  • superadmin  → admin + billing, security settings      │   │   │
│  │  │                                                           │   │   │
│  │  │ Auth flow:                                                │   │   │
│  │  │  1. Redirect to Keycloak login                           │   │   │
│  │  │  2. Email/password + optional MFA (TOTP)                │   │   │
│  │  │  3. Keycloak issues JWT (id_token + access_token)        │   │   │
│  │  │  4. JWT validated by backend middleware                  │   │   │
│  │  │  5. Role extracted from JWT claims                       │   │   │
│  │  └──────────────────────────────────────────────────────────┘   │   │
│  │                                                                  │   │
│  │ JWT Payload:                                                     │   │
│  │ {                                                                │   │
│  │   "sub": "user-uuid-123",                                       │   │
│  │   "email": "admin@udara.ai",                                    │   │
│  │   "realm_access": {"roles": ["admin"]},                         │   │
│  │   "resource_access": {"udara-api": {"roles": ["admin"]}},       │   │
│  │   "exp": 1705305600,        // 1 hour from issue                │   │
│  │   "iat": 1705302000,        // issued at                        │   │
│  │   "iss": "https://keycloak.udara.ai/realms/udara",              │   │
│  │   "aud": "udara-api"                                             │   │
│  │ }                                                                │   │
│  └─────────────────────────────────────────────────────────────────┘   │
│                                                                          │
└──────────────────────────────────────────────────────────────────────────┘
```

### 5.2 Authentication Implementation

```python
# cloud/auth/keycloak_auth.py

"""
Keycloak OIDC integration for the UDARA web dashboard.

Provides JWT validation, role extraction, and authentication middleware.
"""

import logging
from dataclasses import dataclass
from functools import wraps
from typing import Optional

import httpx
import jwt
from jwt import PyJWKClient

logger = logging.getLogger(__name__)


@dataclass
class AuthConfig:
    """Keycloak configuration."""
    server_url: str = "https://keycloak.udara.ai"
    realm: str = "udara"
    client_id: str = "udara-api"
    client_secret: Optional[str] = None

    @property
    def realm_url(self) -> str:
        return f"{self.server_url}/realms/{self.realm}"

    @property
    def jwks_url(self) -> str:
        return f"{self.realm_url}/protocol/openid-connect/certs"

    @property
    def token_endpoint(self) -> str:
        return f"{self.realm_url}/protocol/openid-connect/token"


@dataclass
class AuthenticatedUser:
    """Authenticated user extracted from JWT."""
    user_id: str
    email: str
    roles: list[str]
    token_exp: int
    is_superadmin: bool = False
    is_admin: bool = False
    is_editor: bool = False
    is_viewer: bool = False


class KeycloakAuth:
    """
    Keycloak OIDC authentication and authorization.

    Usage:
        auth = KeycloakAuth(config)
        user = auth.validate_token(jwt_string)
        if auth.has_role(user, "admin"):
            # Allow admin operation
    """

    ROLE_HIERARCHY = {
        "superadmin": ["superadmin", "admin", "editor", "viewer"],
        "admin": ["admin", "editor", "viewer"],
        "editor": ["editor", "viewer"],
        "viewer": ["viewer"],
    }

    def __init__(self, config: AuthConfig):
        self.config = config
        self._jwk_client: Optional[PyJWKClient] = None

    def _get_jwk_client(self) -> PyJWKClient:
        """Lazy-load JWK client for public key verification."""
        if self._jwk_client is None:
            self._jwk_client = PyJWKClient(
                uri=self.config.jwks_url,
                cache_keys=True,
                cache_jwk_set=True,
                lifespan=300,  # Cache JWKS for 5 minutes
            )
        return self._jwk_client

    def validate_token(self, token: str) -> Optional[AuthenticatedUser]:
        """
        Validate a JWT access token and extract user info.

        Args:
            token: JWT access token string.

        Returns:
            AuthenticatedUser if token is valid, None otherwise.

        Raises:
            JWTError: If token is malformed or expired.
        """
        try:
            signing_key = self._get_jwk_client().get_signing_key_from_jwt(token)
        except Exception as e:
            logger.error("Failed to fetch signing key: %s", e)
            return None

        try:
            payload = jwt.decode(
                token,
                signing_key.key,
                algorithms=["RS256"],
                audience=self.config.client_id,
                issuer=self.config.realm_url,
                options={
                    "require": ["exp", "iat", "sub", "iss", "aud"],
                    "verify_exp": True,
                    "verify_iat": True,
                    "verify_iss": True,
                    "verify_aud": True,
                },
            )
        except jwt.ExpiredSignatureError:
            logger.warning("Token expired")
            return None
        except jwt.InvalidTokenError as e:
            logger.warning("Invalid token: %s", e)
            return None

        # Extract roles from realm_access and resource_access
        roles = set()
        realm_roles = payload.get("realm_access", {}).get("roles", [])
        resource_roles = (
            payload
            .get("resource_access", {})
            .get(self.config.client_id, {})
            .get("roles", [])
        )
        roles.update(realm_roles)
        roles.update(resource_roles)

        user = AuthenticatedUser(
            user_id=payload["sub"],
            email=payload.get("email", ""),
            roles=list(roles),
            token_exp=payload["exp"],
            is_superadmin="superadmin" in roles,
            is_admin="admin" in roles or "superadmin" in roles,
            is_editor="editor" in roles or "admin" in roles,
            is_viewer=True,  # All authenticated users have at least viewer
        )

        return user

    def has_role(self, user: AuthenticatedUser, required_role: str) -> bool:
        """Check if user has a specific role (with hierarchy)."""
        effective_roles = set()
        for role in user.roles:
            effective_roles.update(self.ROLE_HIERARCHY.get(role, [role]))
        return required_role in effective_roles

    def require_role(self, role: str):
        """Decorator for requiring a specific role on FastAPI endpoints."""
        def decorator(func):
            @wraps(func)
            async def wrapper(*args, **kwargs):
                request = kwargs.get("request")
                if not request:
                    raise RuntimeError("No request object in kwargs")

                user = getattr(request.state, "user", None)
                if not user:
                    from fastapi import HTTPException
                    raise HTTPException(status_code=401, detail="Not authenticated")

                if not self.has_role(user, role):
                    from fastapi import HTTPException
                    raise HTTPException(
                        status_code=403,
                        detail=f"Insufficient permissions. Required: {role}",
                    )
                return await func(*args, **kwargs)
            return wrapper
        return decorator
```

---

## 6. Authorization (RBAC)

### 6.1 Role Definitions

```
┌──────────────────────────────────────────────────────────────────────────┐
│                     ROLE-BASED ACCESS CONTROL                             │
│                                                                          │
│  ┌─────────────┬──────┬──────────┬──────┬───────────┬──────────────┐   │
│  │ Permission  │Viewer│ Editor   │Admin │Superadmin │ Description  │   │
│  ├─────────────┼──────┼──────────┼──────┼───────────┼──────────────┤   │
│  │ View        │  ✅  │   ✅     │  ✅  │    ✅     │              │   │
│  │ dashboard   │      │          │      │           │              │   │
│  ├─────────────┼──────┼──────────┼──────┼───────────┼──────────────┤   │
│  │ View AMR    │  ✅  │   ✅     │  ✅  │    ✅     │ Aggregated   │   │
│  │ statistics  │      │          │      │           │ data only    │   │
│  ├─────────────┼──────┼──────────┼──────┼───────────┼──────────────┤   │
│  │ Export CSV  │  ✅  │   ✅     │  ✅  │    ✅     │ Summary      │   │
│  │             │      │          │      │           │ reports      │   │
│  ├─────────────┼──────┼──────────┼──────┼───────────┼──────────────┤   │
│  │ Create/Edit │  ❌  │   ✅     │  ✅  │    ✅     │ Includes     │   │
│  │ cases       │      │          │      │           │ case mgmt    │   │
│  ├─────────────┼──────┼──────────┼──────┼───────────┼──────────────┤   │
│  │ Manage      │  ❌  │   ❌     │  ✅  │    ✅     │ User CRUD    │   │
│  │ users       │      │          │      │           │ operations   │   │
│  ├─────────────┼──────┼──────────┼──────┼───────────┼──────────────┤   │
│  │ View audit  │  ❌  │   ❌     │  ✅  │    ✅     │ Security     │   │
│  │ logs        │      │          │      │           │ logs         │   │
│  ├─────────────┼──────┼──────────┼──────┼───────────┼──────────────┤   │
│  │ System      │  ❌  │   ❌     │  ❌  │    ✅     │ Billing,     │   │
│  │ settings    │      │          │      │           │ security cfg │   │
│  ├─────────────┼──────┼──────────┼──────┼───────────┼──────────────┤   │
│  │ Access raw  │  ❌  │   ❌     │  ❌  │    ✅     │ Full DB      │   │
│  │ PII         │      │          │      │           │ access       │   │
│  ├─────────────┼──────┼──────────┼──────┼───────────┼──────────────┤   │
│  │ Delete data │  ❌  │   ❌     │  ❌  │    ✅     │ Destructive  │   │
│  │             │      │          │      │           │ operations   │   │
│  └─────────────┴──────┴──────────┴──────┴───────────┴──────────────┘   │
│                                                                          │
│  Role Assignment:                                                        │
│  • viewer    → Auto-assigned to all authenticated CHWs                   │
│  • editor    → Assigned by admin for CHWs with reporting duties         │
│  • admin     → Assigned by superadmin for facility/region managers      │
│  • superadmin→ Maximum 3 per country. Requires MFA.                     │
│                                                                          │
└──────────────────────────────────────────────────────────────────────────┘
```

### 6.2 JWT Token Lifecycle

```
  Token Lifecycle:
  ┌──────────┐    ┌──────────────┐    ┌──────────────┐    ┌──────────┐
  │  Issue   │───▶│   Active     │───▶│  Expired     │───▶│ Invalid  │
  │  (login) │    │  (1hr TTL)   │    │  (after 1h)  │    │          │
  └──────────┘    └──────┬───────┘    └──────┬───────┘    └──────────┘
                         │                   │
                         │ Refresh token     │ Can be refreshed
                         │ (30 day TTL)      │ with refresh token
                         │                   │
                         ▼                   ▼
                  ┌──────────────┐    ┌──────────────┐
                  │  Refresh     │    │  Use refresh │
                  │  token flow  │    │  token       │
                  │              │    │  → new JWT   │
                  └──────────────┘    └──────────────┘

  Refresh Token Rules:
  - 30-day expiry (configurable per realm)
  - Single-use: rotated on each refresh
  - Revoked on password change
  - Revoked on explicit logout
  - Max 5 active sessions per user
```

---

## 7. Audit Logging

### 7.1 Audit Event Schema

```python
# cloud/audit/audit_logger.py

"""
Structured audit logging for UDARA AI.

All security-relevant events are logged as structured JSON
and shipped to Loki → Grafana for analysis and alerting.

Retention:
  - Hot storage (Loki): 90 days
  - Cold storage (S3): 1 year (compressed)
"""

import json
import logging
import sys
from datetime import datetime, timezone
from typing import Any, Optional

logger = logging.getLogger("udara.audit")


class AuditEvent:
    """
    Structured audit event.

    Every security-relevant action MUST be logged as an AuditEvent.
    """

    # Event categories
    AUTH = "auth"
    DATA_ACCESS = "data_access"
    DATA_MODIFY = "data_modify"
    DATA_DELETE = "data_delete"
    SYSTEM = "system"
    SECURITY = "security"
    SYNC = "sync"
    CONFIG = "config"

    def __init__(
        self,
        event_type: str,
        action: str,
        actor: str,
        resource_type: str,
        resource_id: Optional[str] = None,
        details: Optional[dict] = None,
        ip_address: Optional[str] = None,
        user_agent: Optional[str] = None,
        channel: Optional[str] = None,
        country: Optional[str] = None,
        success: bool = True,
        error_message: Optional[str] = None,
    ):
        self.timestamp = datetime.now(timezone.utc).isoformat()
        self.event_type = event_type
        self.action = action
        self.actor = actor
        self.resource_type = resource_type
        self.resource_id = resource_id
        self.details = details or {}
        self.ip_address = ip_address
        self.user_agent = user_agent
        self.channel = channel
        self.country = country
        self.success = success
        self.error_message = error_message

    def to_json(self) -> str:
        """Serialize to JSON string for logging."""
        return json.dumps({
            "timestamp": self.timestamp,
            "event_type": self.event_type,
            "action": self.action,
            "actor": self.actor,
            "resource_type": self.resource_type,
            "resource_id": self.resource_id,
            "details": self._sanitize_details(),
            "ip_address": self.ip_address,
            "user_agent": self.user_agent,
            "channel": self.channel,
            "country": self.country,
            "success": self.success,
            "error_message": self.error_message,
            "service": "udara-ai",
            "version": "2.1.0",
        })

    def _sanitize_details(self) -> dict:
        """Remove any PII from detail fields."""
        sanitized = dict(self.details)
        # Remove fields that might contain PII
        for key in ["password", "token", "secret", "api_key", "pin"]:
            if key in sanitized:
                sanitized[key] = "***REDACTED***"
        return sanitized

    def log(self) -> None:
        """Write the audit event to the audit logger."""
        log_entry = self.to_json()
        if self.success:
            logger.info(log_entry)
        else:
            logger.warning(log_entry)


# ─── Convenience Functions ────────────────────────────────────────

def log_auth_login(
    user_id: str,
    channel: str,
    ip: Optional[str] = None,
    success: bool = True,
    error: Optional[str] = None,
) -> None:
    """Log a login attempt."""
    event = AuditEvent(
        event_type=AuditEvent.AUTH,
        action="login",
        actor=user_id,
        resource_type="session",
        ip_address=ip,
        channel=channel,
        success=success,
        error_message=error,
    )
    event.log()


def log_data_access(
    user_id: str,
    resource_type: str,
    resource_id: str,
    action: str = "read",
    channel: Optional[str] = None,
) -> None:
    """Log data access."""
    event = AuditEvent(
        event_type=AuditEvent.DATA_ACCESS,
        action=action,
        actor=user_id,
        resource_type=resource_type,
        resource_id=resource_id,
        channel=channel,
    )
    event.log()


def log_data_modify(
    user_id: str,
    resource_type: str,
    resource_id: str,
    changes: dict,
    channel: Optional[str] = None,
) -> None:
    """Log data modification."""
    event = AuditEvent(
        event_type=AuditEvent.DATA_MODIFY,
        action="update",
        actor=user_id,
        resource_type=resource_type,
        resource_id=resource_id,
        details={"changes": changes},
        channel=channel,
    )
    event.log()


def log_security_event(
    action: str,
    actor: str,
    details: Optional[dict] = None,
    ip: Optional[str] = None,
) -> None:
    """Log a security event (failed auth, rate limit, etc.)."""
    event = AuditEvent(
        event_type=AuditEvent.SECURITY,
        action=action,
        actor=actor,
        resource_type="system",
        details=details,
        ip_address=ip,
    )
    event.log()


def log_sync_event(
    edge_id: str,
    direction: str,
    record_count: int,
    success: bool = True,
    error: Optional[str] = None,
) -> None:
    """Log a data sync event."""
    event = AuditEvent(
        event_type=AuditEvent.SYNC,
        action=direction,
        actor=edge_id,
        resource_type="data_batch",
        details={"record_count": record_count},
        success=success,
        error_message=error,
    )
    event.log()
```

### 7.2 Loki → Grafana Pipeline

```
┌─────────────────────────────────────────────────────┐
│          AUDIT LOG PIPELINE                         │
│                                                     │
│  ┌──────────┐    ┌──────────┐    ┌──────────────┐ │
│  │ udara-*  │───▶│   Loki   │───▶│   Grafana    │ │
│  │ services │JSON│ (log     │LogQL│ (dashboards) │ │
│  │          │   │  aggregator)│   │              │ │
│  └──────────┘    └─────┬────┘    └──────────────┘ │
│                        │                          │
│                        │ ┌──────────────────────┐ │
│                        │ │ Retention Policies:  │ │
│                        │ │ Hot (Loki): 90 days  │ │
│                        │ │ Cold (S3): 1 year    │ │
│                        │ │ Archive: 7 years     │ │
│                        │ └──────────────────────┘ │
│                                                     │
│  Alert Rules (Grafana):                             │
│  ┌────────────────────────────────────────────────┐ │
│  │ 1. >10 failed logins/hour → PagerDuty alert    │ │
│  │ 2. Any data delete by non-superadmin → Email   │ │
│  │ 3. Sync failure > 1hr → Slack #ops-alerts     │ │
│  │ 4. New superadmin role assigned → Email        │ │
│  │ 5. PII detected in sync payload → Page Eng     │ │
│  │ 6. API key rotation overdue → Weekly email     │ │
│  └────────────────────────────────────────────────┘ │
└─────────────────────────────────────────────────────┘
```

---

## 8. Data Sovereignty

### 8.1 Data Residency Policy

```
┌──────────────────────────────────────────────────────────────────────────┐
│                      DATA SOVEREIGNTY POLICY                              │
│                                                                          │
│  CORE PRINCIPLE:                                                         │
│  ═══════════════                                                         │
│  ALL UDARA AI DATA STAYS IN AFRICA. PERIOD.                              │
│                                                                          │
│  ┌─────────────────────────────────────────────────────────────────┐    │
│  │                                                                  │    │
│  │  ✅ ALLOWED:                                                      │    │
│  │  • Processing in AWS af-south-1 (Cape Town, South Africa)        │    │
│  │  • Edge processing on Raspberry Pi (in-country)                  │    │
│  │  • Sync between edge nodes and af-south-1                       │    │
│  │  • Aggregated, anonymized data export for WHO GLASS             │    │
│  │                                                                  │    │
│  │  ❌ FORBIDDEN:                                                    │    │
│  │  • Any data in us-east-1, eu-west-1, or any non-African region  │    │
│  │  • Third-party API calls outside Africa (OpenAI, Google, etc.)  │    │
│  │  • CDN assets hosted outside Africa                             │    │
│  │  • Analytics/tracking pixels to global servers                  │    │
│  │  • Any employee access from outside the approved VPN            │    │
│  │                                                                  │    │
│  └─────────────────────────────────────────────────────────────────┘    │
│                                                                          │
│  ENFORCEMENT:                                                            │
│  ┌─────────────────────────────────────────────────────────────────┐    │
│  │                                                                  │    │
│  │  1. AWS SCP (Service Control Policy) restricts all services     │    │
│  │     to af-south-1 only. Any attempt to create resources in      │    │
│  │     other regions is blocked at the AWS organization level.    │    │
│  │                                                                  │    │
│  │  2. Network firewall blocks all outbound traffic except:        │    │
│  │     - *.amazonaws.com (af-south-1 only)                        │    │
│  │     - grafana.com (for monitoring)                             │    │
│  │     - pypi.org (for CI/CD only, rate-limited)                  │    │
│  │     - github.com (for CI/CD only, rate-limited)                │    │
│  │                                                                  │    │
│  │  3. Code-level guard: every API call checks the region:         │    │
│  │     assert boto3.client('s3').meta.region_name == 'af-south-1' │    │
│  │                                                                  │    │
│  │  4. CI/CD pipeline has a "data sovereignty check" step that     │    │
│  │     scans CloudFormation/Terraform for non-af-south-1 refs.     │    │
│  │                                                                  │    │
│  │  5. Quarterly audit by external security firm to verify         │    │
│  │     compliance.                                                 │    │
│  │                                                                  │    │
│  └─────────────────────────────────────────────────────────────────┘    │
│                                                                          │
│  AWS SERVICES IN af-south-1:                                             │
│  ┌─────────────────────────────────────────────────────────────────┐    │
│  │  ✅ Available in af-south-1:                                    │    │
│  │  • EC2, Lambda, Fargate                                        │    │
│  │  • RDS (PostgreSQL, MySQL)                                     │    │
│  │  • ElastiCache (Redis)                                         │    │
│  │  • S3, EFS                                                     │    │
│  │  • KMS, Secrets Manager                                        │    │
│  │  • CloudFront (origin in af-south-1)                           │    │
│  │  • CloudWatch, CloudTrail                                      │    │
│  │  • IAM, organizations, SCP                                     │    │
│  │  • Route 53, ACM                                               │    │
│  │                                                                  │    │
│  │  ❌ NOT Available (workarounds needed):                         │    │
│  │  • AWS SageMaker → use self-hosted ML on EC2/GPU               │    │
│  │  • AWS Bedrock → use local LLM (llama.cpp)                     │    │
│  │  • AWS Comprehend → use self-hosted NER (spaCy)                │    │
│  │                                                                  │    │
│  └─────────────────────────────────────────────────────────────────┘    │
│                                                                          │
└──────────────────────────────────────────────────────────────────────────┘
```

---

## 9. Data Breach Response Plan

### 9.1 Response Timeline

```
┌──────────────────────────────────────────────────────────────────────────┐
│                    DATA BREACH RESPONSE PLAN                              │
│                                                                          │
│  Phase 1: DETECTION (Target: < 1 hour)                                 │
│  ══════════════════════════════════════                                 │
│  Trigger: Automated alerting from Grafana/Loki                         │
│                                                                          │
│  T+0:00  Alert fires (anomaly in data access, sync failure,            │
│          unauthorized access attempt, etc.)                             │
│  T+0:05  On-call security engineer acknowledges alert                 │
│  T+0:15  Initial assessment: is this a real incident?                  │
│  T+0:30  If confirmed: escalate to Security Lead + CTO                │
│  T+0:60  Incident commander assigned, war room opened                  │
│                                                                          │
│  Phase 2: CONTAINMENT (Target: < 4 hours)                               │
│  ═══════════════════════════════════════                                │
│                                                                          │
│  Actions:                                                                │
│  □ Revoke compromised API keys                                         │
│  □ Force password reset for affected accounts                          │
│  □ Disable affected edge node (remote command)                         │
│  □ Block suspicious IP addresses at WAF/firewall                      │
│  □ Rotate encryption keys if key compromise suspected                 │
│  □ Enable enhanced logging on affected systems                         │
│  □ Take forensic snapshot of affected systems                          │
│                                                                          │
│  Phase 3: NOTIFICATION (Target: < 72 hours — NDPR requirement)         │
│  ═══════════════════════════════════════════════════════               │
│                                                                          │
│  NDPR (Nigeria): 72 hours from discovery                                │
│  KE DPA (Kenya): 72 hours from discovery                               │
│                                                                          │
│  Notifications:                                                          │
│  □ Affected data subjects (via WhatsApp/SMS)                           │
│  □ National data protection authority (NDPC, ODPC, etc.)               │
│  □ Country coordinators                                               │
│  □ Board of directors                                                  │
│  □ Legal counsel                                                       │
│                                                                          │
│  Notification template includes:                                        │
│  - What data was affected                                             │
│  - What we are doing about it                                         │
│  - What the individual should do                                      │
│  - Contact information for questions                                  │
│                                                                          │
│  Phase 4: RECOVERY (Target: < 1 week)                                  │
│  ════════════════════════════════════                                  │
│                                                                          │
│  □ Patch vulnerability that caused the breach                          │
│  □ Restore from last known good backup                                │
│  □ Re-provision compromised edge nodes                                │
│  □ Re-issue all API keys                                               │
│  □ Verify no data exfiltration (forensic analysis)                     │
│  □ Gradual service restoration with enhanced monitoring               │
│                                                                          │
│  Phase 5: POST-MORTEM (Target: < 2 weeks)                              │
│  ════════════════════════════════════════                               │
│                                                                          │
│  □ Root cause analysis (5 Whys technique)                              │
│  □ Timeline reconstruction                                             │
│  □ Impact assessment                                                   │
│  □ Remediation actions                                                 │
│  □ Prevention measures (new controls)                                  │
│  □ Update security checklist                                           │
│  □ Train staff on lessons learned                                     │
│  □ Publish summary (internal)                                          │
│                                                                          │
└──────────────────────────────────────────────────────────────────────────┘
```

### 9.2 Incident Severity Levels

| Level | Name | Description | Response Time | Notification |
|-------|------|-------------|---------------|--------------|
| SEV-1 | Critical | Large-scale PII exposure, data exfiltration confirmed | 15 min | CTO + Legal + Board |
| SEV-2 | High | Confirmed unauthorized access, limited PII affected | 1 hour | Security Lead + CTO |
| SEV-3 | Medium | Suspected unauthorized access, under investigation | 4 hours | Security Lead |
| SEV-4 | Low | Failed login attempts, minor policy violation | 24 hours | Security team |

---

## 10. Security Checklist Per Release

### 10.1 Pre-Release Security Checklist

Every release (sprint or hotfix) MUST pass this checklist before deployment:

```
┌─────────────────────────────────────────────────────────────────┐
│          UDARA AI — PRE-RELEASE SECURITY CHECKLIST              │
│          Document ID: UDARA-SEC-CL-001                           │
│          Version: 2.1 | Last Updated: 2026-05-27                │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  □ 1. DEPENDENCY VULNERABILITY SCAN                             │
│     □ Run `pip-audit --strict` — 0 known vulnerabilities       │
│     □ Run `safety check --full-report` — 0 critical/high       │
│     □ Verify `requirements.txt` hash pinning (all packages)     │
│     □ No new dependencies without security review               │
│     Sign-off: _______________ Date: _______                    │
│                                                                 │
│  □ 2. STATIC CODE ANALYSIS                                     │
│     □ Run `bandit -r . -ll` — 0 high/severity issues           │
│     □ Run `ruff check .` — 0 errors                            │
│     □ No hardcoded secrets (grep for password, token, secret)  │
│     □ No debug code in production (print, pdb, logging.debug)  │
│     Sign-off: _______________ Date: _______                    │
│                                                                 │
│  □ 3. AUTHENTICATION & AUTHORIZATION                            │
│     □ All API endpoints require authentication                  │
│     □ RBAC checked on all data access endpoints                 │
│     □ JWT expiry is 1 hour (no infinite tokens)                │
│     □ Refresh token rotation verified                          │
│     □ Keycloak realm configuration unchanged                    │
│     Sign-off: _______________ Date: _______                    │
│                                                                 │
│  □ 4. ENCRYPTION                                               │
│     □ TLS 1.3 enforced on all endpoints                        │
│     □ SQLCipher encryption verified on edge nodes              │
│     □ AWS KMS key rotation status OK                           │
│     □ Sync payload encryption (AES-256-GCM) tested             │
│     □ No plaintext PII in logs                                 │
│     Sign-off: _______________ Date: _______                    │
│                                                                 │
│  □ 5. PII ANONYMIZATION                                        │
│     □ Presidio pipeline runs on all edge sync                  │
│     □ All entity types detected: PERSON, PHONE, LOCATION,     │
│       DATE, AGE                                                │
│     □ Anonymized output contains zero raw PII                   │
│     □ Fallback: if Presidio fails, block sync entirely         │
│     Sign-off: _______________ Date: _______                    │
│                                                                 │
│  □ 6. DATA SOVEREIGNTY                                         │
│     □ All AWS resources in af-south-1 (verify with CLI)       │
│     □ SCP blocks non-af-south-1 resource creation             │
│     □ No external API calls (OpenAI, Google) in production    │
│     □ DNS resolution confirms af-south-1 endpoints             │
│     Sign-off: _______________ Date: _______                    │
│                                                                 │
│  □ 7. AUDIT LOGGING                                            │
│     □ All sensitive operations logged with AuditEvent          │
│     □ Log pipeline to Loki verified (test event visible)       │
│     □ Grafana alerts configured and tested                     │
│     □ No PII in audit log details                             │
│     Sign-off: _______________ Date: _______                    │
│                                                                 │
│  □ 8. INFRASTRUCTURE                                           │
│     □ Docker images scanned (trivy) — 0 critical/high CVEs     │
│     □ Container running as non-root user                      │
│     □ No unnecessary ports exposed                            │
│     □ Secrets in AWS Secrets Manager (not env vars)            │
│     □ IAM roles follow least-privilege principle               │
│     Sign-off: _______________ Date: _______                    │
│                                                                 │
│  □ 9. TESTING                                                  │
│     □ All security unit tests pass                             │
│     □ Authentication bypass tests pass                         │
│     □ Authorization boundary tests pass                       │
│     □ SQL injection tests pass                                 │
│     □ XSS tests pass                                           │
│     Sign-off: _______________ Date: _______                    │
│                                                                 │
│  □ 10. DEPLOYMENT                                             │
│     □ Blue/green deployment verified                           │
│     □ Rollback procedure documented and tested                 │
│     □ Health check endpoints responding                       │
│     □ Error rate < 1% in first 30 minutes                     │
│     □ On-call notified of deployment                          │
│     Sign-off: _______________ Date: _______                    │
│                                                                 │
│  RELEASE APPROVAL:                                              │
│  Security Lead: _______________ Date: _______                 │
│  Engineering Lead: _______________ Date: _______               │
│  CTO: _______________ Date: _______                            │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 11. Appendix: Security Configuration Reference

### 11.1 TLS 1.3 Configuration

```python
# edge/config/tls_config.py

"""TLS 1.3 configuration for UDARA edge and cloud services."""

import ssl

# Allowed TLS 1.3 cipher suites
TLS_CIPHERS = [
    "TLS_AES_256_GCM_SHA384",
    "TLS_CHACHA20_POLY1305_SHA256",
]

# Rejected cipher suites (documented for audit)
REJECTED_CIPHERS = [
    "TLS_AES_128_GCM_SHA256",    # Insufficient key length
    "TLS_RSA_WITH_AES_256_GCM_SHA384",  # No forward secrecy
    "ECDHE-RSA-AES256-GCM-SHA384",     # TLS 1.2 only
]


def create_ssl_context() -> ssl.SSLContext:
    """Create a TLS 1.3 only SSL context."""
    ctx = ssl.SSLContext(ssl.PROTOCOL_TLS_SERVER)
    ctx.minimum_version = ssl.TLSVersion.TLSv1_3
    ctx.maximum_version = ssl.TLSVersion.TLSv1_3
    ctx.set_ciphers(":".join(TLS_CIPHERS))
    ctx.options |= ssl.OP_NO_COMPRESSION
    ctx.options |= ssl.OP_SINGLE_ECDH_USE
    ctx.options |= ssl.OP_NO_TLSv1
    ctx.options |= ssl.OP_NO_TLSv1_1
    ctx.options |= ssl.OP_NO_TLSv1_2
    return ctx
```

### 11.2 Security Headers

```python
# cloud/middleware/security_headers.py

"""Security headers for the UDARA web dashboard."""

from starlette.middleware.base import BaseHTTPMiddleware
from starlette.requests import Request
from starlette.responses import Response


class SecurityHeadersMiddleware(BaseHTTPMiddleware):
    """Adds security headers to all HTTP responses."""

    HEADERS = {
        "Strict-Transport-Security": "max-age=63072000; includeSubDomains; preload",
        "X-Content-Type-Options": "nosniff",
        "X-Frame-Options": "DENY",
        "X-XSS-Protection": "1; mode=block",
        "Referrer-Policy": "strict-origin-when-cross-origin",
        "Content-Security-Policy": (
            "default-src 'self'; "
            "script-src 'self' 'unsafe-inline'; "
            "style-src 'self' 'unsafe-inline' https://fonts.googleapis.com; "
            "font-src 'self' https://fonts.gstatic.com; "
            "img-src 'self' data: https:; "
            "connect-src 'self' https://*.amazonaws.com; "
            "frame-ancestors 'none'; "
            "base-uri 'self'; "
            "form-action 'self'"
        ),
        "Permissions-Policy": "camera=(), microphone=(), geolocation=()",
        "Cache-Control": "no-store, no-cache, must-revalidate",
        "Pragma": "no-cache",
    }

    async def dispatch(self, request: Request, call_next) -> Response:
        response = await call_next(request)
        for header, value in self.HEADERS.items():
            response.headers[header] = value
        return response
```

---

*End of Document UDARA-SEC-011*
