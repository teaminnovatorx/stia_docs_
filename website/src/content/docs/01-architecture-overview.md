# 01 — Architecture Overview

> **UDARA AI** is a hybrid edge-cloud system purpose-built for AMR surveillance in
> environments with intermittent connectivity, limited bandwidth, and diverse device
> capabilities across sub-Saharan Africa.

---

## Table of Contents

- [Design Philosophy](#design-philosophy)
  - [Why Edge + Cloud (Not Pure Either)](#why-edge--cloud-not-pure-either)
  - [The Tree Mental Model](#the-tree-mental-model)
  - [Design Principles](#design-principles)
- [Architecture Diagram](#architecture-diagram)
- [What Runs Where](#what-runs-where)
- [Data Flow](#data-flow)
  - [Full 10-Step Sequence](#full-10-step-sequence)
  - [Data Flow Diagram](#data-flow-diagram)
- [Sync Architecture](#sync-architecture)
  - [Delta-Only JSON](#delta-only-json)
  - [Compression](#compression)
  - [Encryption](#encryption)
  - [CRDT via Yjs](#crdt-via-yjs)
  - [Exponential Backoff](#exponential-backoff)
  - [USB Fallback](#usb-fallback)
- [Edge Node Specifications](#edge-node-specifications)
  - [Hardware](#hardware)
  - [Software Stack](#software-stack)
  - [RAM Budget](#ram-budget)
- [Failure Modes](#failure-modes)
  - [Connectivity Lost](#connectivity-lost)
  - [Pi Reboots Unexpectedly](#pi-reboots-unexpectedly)
  - [Model Out of Memory](#model-out-of-memory)
  - [SD Card Corruption](#sd-card-corruption)
  - [Database Lock Contention](#database-lock-contention)
  - [Clock Drift](#clock-drift)
- [Key Architectural Decisions](#key-architectural-decisions)

---

## Design Philosophy

### Why Edge + Cloud (Not Pure Either)

The decision to use a hybrid edge-cloud architecture was driven by the operational
realities of deploying AMR surveillance in sub-Saharan Africa. Neither a pure cloud
nor a pure edge approach would work.

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    WHY PURE CLOUD FAILS HERE                                │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  Problem 1: Connectivity                                                    │
│  ┌────────────┐     ┌──────────┐     ┌───────────┐     ┌──────────┐         │
│  │  CHW       │───▶│ Cellular │───▶│ 3G/4G     │───▶│ Internet │         │
│  │  submits   │     │ tower    │     │ (spotty)  │     │ (AWS)    │         │
│  └────────────┘     └──────────┘     └───────────┘     └──────────┘         │
│                       ▲                  ▲                                  │
│                       │                  │                                  │
│                 30-60% downtime     20-40% uptime                           │
│                 in rural areas      during business hours                   │
│                                                                             │
│  Problem 2: Bandwidth                                                       │
│  - Average rural 3G: 0.5-2 Mbps                                             │
│  - Image upload (photo of prescription): 2-5 MB raw                         │
│  - Audio message: 0.5-3 MB                                                  │
│  - OCR/ASR on cloud: every report requires round-trip                       │
│                                                                             │
│  Problem 3: Latency                                                         │
│  - Round-trip to af-south-1: 150-400ms (good), 2000ms+ (bad)                │
│  - USSD session timeout: 120 seconds                                        │
│  - CHW cannot wait 5-10 seconds for cloud NER response                      │
│                                                                             │
│  Problem 4: Data Sovereignty                                                │
│  - Nigeria NDPR requires health data to be processed locally                │
│  - Some reports contain patient PII that should not leave facility          │
│  - Cloud-only means all data traverses international links                  │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────────────┐
│                    WHY PURE EDGE FAILS HERE                                 │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  Problem 1: Cross-Facility Intelligence                                     │
│  - AMR patterns span multiple facilities and states                         │
│  - Outbreak detection requires aggregated data                              │
│  - A single Pi sees only its own facility's reports                         │
│                                                                             │
│  Problem 2: Model Updates                                                   │
│  - Models need periodic retraining on new data                              │
│  - Pi cannot train AfroBERT (requires GPU cluster)                          │
│  - New resistance patterns emerge from global data                          │
│                                                                             │
│  Problem 3: Official Reporting                                              │
│  - WHO GLASS, NCDC require national-level data                              │
│  - PDF/CSV exports, API access for researchers                              │
│  - Audit trail and data provenance                                          │
│                                                                             │
│  Problem 4: Storage & Backup                                                │
│  - RPi SD card: limited capacity, prone to corruption                       │
│  - No RAID, no redundancy at edge                                           │
│  - Need durable cloud storage for regulatory compliance                     │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────────────┐
│                    WHY EDGE + CLOUD WORKS                                   │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  Edge handles:                    Cloud handles:                            │
│  ✅ Real-time NER/OCR/ASR         ✅ Cross-facility analytics               │
│  ✅ USSD sessions (120s TTL)     ✅ Model training & distribution           │
│  ✅ Offline data collection       ✅ Official dashboards (Door 3)           │
│  ✅ Local Bayesian estimates       ✅ WHO GLASS exports                     │
│  ✅ PII stays on-device           ✅ Long-term durable storage              │
│  ✅ Sub-second response time      ✅ Alerting across facilities             │
│  ✅ Works during outages          ✅ User management (Keycloak)             │
│                                                                             │
│  Sync connects them:                                                        │
│  ✅ Delta-only (send changes, not full dump)                                │
│  ✅ AES-256-GCM encrypted in transit                                        │
│  ✅ CRDT-based conflict resolution                                          │
│  ✅ USB fallback for extended outages                                       │
│  ✅ Exponential backoff with jitter                                         │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

### The Tree Mental Model

We use a **tree** as the mental model for the entire system. This metaphor helps every
team member understand the architecture regardless of their technical background.

```
                        ╔═══════════════════════╗
                        ║   CANOPY (CLOUD)      ║
                        ║                       ║
                        ║   Cross-facility AI   ║
                        ║   National dashboard  ║
                        ║   Model training      ║
                        ║   WHO GLASS exports   ║
                        ║   Alert dispatch      ║
                        ╚════════╤══════════════╝
                                 │
                        ┌────────┴────────┐
                        │    SYNC LAYER   │
                        │ (Delta + CRDT + │
                        │  AES-256 + USB) │
                        └────────┬────────┘
                                 │
              ┌──────────────────┼──────────────────┐
              │                  │                  │
        ╔═════╧══════╗    ╔═════╧══════╗    ╔═════╧══════╗
        ║  BRANCH A  ║    ║  BRANCH B  ║    ║  BRANCH C  ║
        ║  Facility A║    ║  Facility B║    ║  Facility C║
        ║  (Edge Pi) ║    ║  (Edge Pi) ║    ║  (Edge Pi) ║
        ║            ║    ║            ║    ║            ║
        ║  • NER     ║    ║  • NER     ║    ║  • NER     ║
        ║  • OCR     ║    ║  • OCR     ║    ║  • OCR     ║
        ║  • ASR     ║    ║  • ASR     ║    ║  • ASR     ║
        ║  • LLM     ║    ║  • LLM     ║    ║  • LLM     ║
        ║  • SQLite  ║    ║  • SQLite  ║    ║  • SQLite  ║
        ╚═════╤══════╝    ╚═════╤══════╝    ╚═════╤══════╝
              │                 │                 │
         ┌────┴────┐       ┌────┴────┐       ┌────┴────┐
         │ TRUNK   │       │ TRUNK   │       │ TRUNK   │
         │ (Local  │       │ (Local  │       │ (Local  │
         │  WiFi)  │       │  WiFi)  │       │  WiFi)  │
         └────┬────┘       └────┬────┘       └────┬────┘
              │                 │                 │
        ╔═════╧═════╗     ╔═════╧═════╗     ╔═════╧═════╗
        ║   ROOTS   ║     ║   ROOTS   ║     ║   ROOTS   ║
        ║           ║     ║           ║     ║           ║
        ║  CHWs     ║     ║  CHWs     ║     ║  CHWs     ║
        ║  Patients ║     ║  Patients ║     ║  Patients ║
        ║  Community║     ║  Community║     ║  Community║
        ╚═══════════╝     ╚═══════════╝     ╚═══════════╝
```

| Layer | Component | Analogy | Purpose |
|-------|-----------|---------|---------|
| **Roots** | CHWs, patients, community | Roots absorb water & nutrients | Absorb raw AMR observations from the community |
| **Trunk** | Local WiFi, Africa's Talking, WhatsApp, Telegram | Trunk transports water upward | Transport data from CHWs to the edge node |
| **Branches** | Edge nodes (RPi 5), local SQLite, AI models | Branches process & distribute sap | Process, enrich, and store data locally |
| **Canopy** | Cloud (AWS), PostgreSQL, MLflow, dashboard | Canopy captures sunlight & photosynthesizes | Aggregate, analyze, train models, and report nationally |

### Design Principles

1. **Offline-First** — Every critical feature works without internet. Edge nodes
   can collect, process, store, and display AMR data for weeks without connectivity.

2. **Progressive Enhancement** — Start with USSD (lowest common denominator), add
   features as device capability increases: WhatsApp → PWA → Web Dashboard.

3. **Data Minimisation** — Only sync deltas, not full datasets. PII stays on the
   edge whenever possible. Cloud receives anonymised aggregates by default.

4. **Graceful Degradation** — If the LLM model OOMs, fall back to rule-based
   extraction. If OCR fails, accept manual text entry. If sync fails, queue locally.

5. **Community Ownership** — CHWs see the value of their data through local
   resistance dashboards and alert feedback loops.

6. **WHO GLASS Compatibility** — All data structures align with WHO GLASS
   standards for antimicrobial resistance surveillance.

---

## Architecture Diagram

### Full System Architecture

```
┌───────────────────────────────────────────────────────────────────────────────────┐
│                              UDARA AI — FULL ARCHITECTURE                         │
│                                                                                   │
│  ┌─────────────────────────── CHANNEL LAYER ──────────────────────────┐           │
│  │                                                                    │           │
│  │  ┌─────────────┐ ┌─────────────┐ ┌─────────────┐ ┌─────────────┐   │           │
│  │  │ DOOR 1      │ │ DOOR 2      │ │ DOOR 2      │ │ DOOR 2      │   │           │
│  │  │ USSD+SMS    │ │ WhatsApp    │ │ Telegram    │ │ PWA         │   │           │
│  │  │ (Feature    │ │ Business API│ │ Bot API     │ │ (Service    │   │           │
│  │  │  phones)    │ │             │ │             │ │  Worker)    │   │           │
│  │  └──────┬──────┘ └──────┬──────┘ └──────┬──────┘ └──────┬──────┘   │           │
│  │         └───────────────┼───────────────┼───────────────┘          │           │
│  │                         │               │                          │           │
│  └─────────────────────────┼───────────────┼──────────────────────────┘           │
│                            │               │                                      │
│                   ┌────────┴───────────────┴────────┐                             │
│                   │         GATEWAY LAYER           │                             │
│                   │                                 │                             │
│                   │  ┌──────────────────────────┐   │                             │
│                   │  │     MESSAGE ROUTER       │   │                             │
│                   │  │                          │   │                             │
│                   │  │  • Channel identification│   │                             │
│                   │  │  • Session management    │   │                             │
│                   │  │  • Rate limiting         │   │                             │
│                   │  │  • Input validation      │   │                             │
│                   │  │  • Load balancing        │   │                             │
│                   │  │  • Health checks         │   │                             │
│                   │  └─────────────┬────────────┘   │                             │
│                   └────────────────┼────────────────┘                             │
│                                    │                                              │
│                   ┌────────────────┼────────────────┐                             │
│                   │                │                │                             │
│         ┌─────────┴───-──┐  ┌──────┴───────┐  ┌────┴────-──┐                      │
│         │  EDGE NODE A   │  │ EDGE NODE B  │  │ EDGE N     │                      │
│         │  ┌───────────┐ │  │ ┌──────────┐ │  │            │                      │
│         │  │ FastAPI   │ │  │ │ FastAPI  │ │  │   ...      │                      │
│         │  │ :8001     │ │  │ │ :8001    │ │  │            │                      │
│         │  └─────┬─────┘ │  │ └────┬─────┘ │  │            │                      │
│         │        │       │  │      │       │  │            │                      │
│         │  ┌─────┴─────┐ │  │ ┌────┴─────┐ │  │            │                      │
│         │  │ AI MODELS │ │  │ │ AI MODELS│ │  │            │                      │
│         │  │ ┌───────┐ │ │  │ │ ┌──────┐ │ │  │            │                      │
│         │  │ │NER    │ │ │  │ │ │NER   │ │ │  │            │                      │
│         │  │ │(280MB)│ │ │  │ │ │      │ │ │  │            │                      │
│         │  │ ├───────┤ │ │  │ │ ├──────┤ │ │  │            │                      │
│         │  │ │OCR    │ │ │  │ │ │OCR   │ │ │  │            │                      │
│         │  │ │(150MB)│ │ │  │ │ │      │ │ │  │            │                      │
│         │  │ ├───────┤ │ │  │ │ ├──────┤ │ │  │            │                      │
│         │  │ │ASR    │ │ │  │ │ │ASR   │ │ │  │            │                      │
│         │  │ │(1.1GB)│ │ │  │ │ │      │ │ │  │            │                      │
│         │  │ ├───────┤ │ │  │ │ ├──────┤ │ │  │            │                      │
│         │  │ │LLM    │ │ │  │ │ │LLM   │ │ │  │            │                      │
│         │  │ │(1.8GB)│ │ │  │ │ │      │ │ │  │            │                      │
│         │  │ ├───────┤ │ │  │ │ ├──────┤ │ │  │            │                      │
│         │  │ │Bayes  │ │ │  │ │ │Bayes │ │ │  │            │                      │
│         │  │ │(50MB) │ │ │  │ │ │      │ │ │  │            │                      │
│         │  │ └───────┘ │ │  │ │ └──────┘ │ │  │            │                      │
│         │  └───────────┘ │  │ └──────────┘ │  │            │                      │
│         │  ┌───────────┐ │  │ ┌──────────┐ │  │            │                      │
│         │  │ SQLite    │ │  │ │ SQLite   │ │  │            │                      │
│         │  │ WAL       │ │  │ │ WAL      │ │  │            │                      │
│         │  │ ChromaDB  │ │  │ │ ChromaDB │ │  │            │                      │
│         │  │ Redis     │ │  │ │ Redis    │ │  │            │                      │
│         │  └───────────┘ │  │ └──────────┘ │  │            │                      │
│         └────────┬───────┘  └──────┬───────┘  └────┬───────┘                      │
│                  │                 │               │                              │
│                  └────────────────┬┘               │                              │
│                                   │                │                              │
│              ┌────────────────────┴────────────────┴──────────────┐               │
│              │              SYNC PROTOCOL LAYER                   │               │
│              │                                                    │               │
│              │  Delta JSON ──▶ gzip ──▶ AES-256-GCM ──▶ HTTPS  │               │
│              │  Conflict: Yjs CRDT (last-writer-wins + merge)     │               │
│              │  Backoff: exponential (1s → 2s → 4s → ... 5min)    │               │
│              │  Fallback: USB drive sync (extended outages)       │               │
│              └────────────────────┬───────────────────────────────┘               │
│                                   │                                               │
│         ┌─────────────────────────┼──────────────────────────────┐                │
│         │                    CLOUD LAYER                         │                │
│         │                                                        │                │
│         │  ┌──────────────────────────────────────────────────┐  │                │
│         │  │              API LAYER (ECS Fargate)             │  │                │
│         │  │                                                  │  │                │
│         │  │  ┌──────────┐  ┌──────────┐  ┌───────────────┐   │  │                │
│         │  │  │ FastAPI  │  │ Celery   │  │ Celery Beat   │   │  │                │
│         │  │  │ REST+WS  │  │ Workers  │  │ (Scheduler)   │   │  │                │
│         │  │  └──────────┘  └──────────┘  └───────────────┘   │  │                │
│         │  └──────────────────────┬───────────────────────────┘  │                │
│         │                         │                              │                │
│         │  ┌──────────────────────┴───────────────────────────┐  │                │
│         │  │              SERVICE LAYER                       │  │                │
│         │  │                                                  │  │                │
│         │  │  ┌────────────┐ ┌────────────┐ ┌─────────────┐   │  │                │
│         │  │  │ Report     │ │ Analytics  │ │ Alert       │   │  │                │
│         │  │  │ Service    │ │ Engine     │ │ Rules Engine│   │  │                │
│         │  │  └────────────┘ └────────────┘ └─────────────┘   │  │                │
│         │  │  ┌────────────┐ ┌────────────┐ ┌─────────────┐   │  │                │
│         │  │  │ Sync       │ │ ML         │ │ Export      │   │  │                │
│         │  │  │ Aggregator │ │ Pipeline   │ │ Service     │   │  │                │
│         │  │  └────────────┘ └────────────┘ └─────────────┘   │  │                │
│         │  └──────────────────────┬───────────────────────────┘  │                │
│         │                         │                              │                │
│         │  ┌──────────────────────┴───────────────────────────┐  │                │
│         │  │              IDENTITY LAYER                      │  │                │
│         │  │                                                  │  │                │
│         │  │  ┌──────────────────────────────────────────┐    │  │                │
│         │  │  │  Keycloak (OIDC + SAML + LDAP)           │    │  │                │
│         │  │  │  • Facility Admin, State Admin, National │    │  │                │
│         │  │  │  • MFA support (TOTP + SMS)              │    │  │                │
│         │  │  │  • Session management                    │    │  │                │
│         │  │  └──────────────────────────────────────────┘    │  │                │
│         │  └──────────────────────────────────────────────────┘  │                │
│         │                                                        │                │
│         │  ┌──────────────────────────────────────────────────┐  │                │
│         │  │              DATA LAYER                          │  │                │
│         │  │                                                  │  │                │
│         │  │  ┌──────────┐  ┌──────────┐  ┌──────────────┐    │  │                │
│         │  │  │ Postgres │  │ Qdrant   │  │ Redis 7      │    │  │                │
│         │  │  │ +TSDB    │  │ (Vector) │  │ (Cache+Queue)│    │  │                │
│         │  │  │ +PostGIS │  │          │  │              │    │  │                │
│         │  │  └──────────┘  └──────────┘  └──────────────┘    │  │                │
│         │  │  ┌──────────┐  ┌──────────┐  ┌──────────────┐    │  │                │
│         │  │  │ S3       │  │CloudFront│  │ Vault        │    │  │                │
│         │  │  │ (Blobs)  │  │ (CDN)    │  │ (Secrets)    │    │  │                │
│         │  │  └──────────┘  └──────────┘  └──────────────┘    │  │                │
│         │  └──────────────────────────────────────────────────┘  │                │
│         │                                                        │                │
│         │  ┌──────────────────────────────────────────────────┐  │                │
│         │  │              ML LAYER                            │  │                │
│         │  │                                                  │  │                │
│         │  │  ┌──────────────┐  ┌──────────────────────┐      │  │                │
│         │  │  │ PyTorch + HF │  │ MLflow               │      │  │                │
│         │  │  │ (Training)   │  │ (Registry + Tracking)│      │  │                │
│         │  │  └──────────────┘  └──────────────────────┘      │  │                │
│         │  └──────────────────────────────────────────────────┘  │                │
│         └────────────────────────────────────────────────────────┘                │
│                                   │                                               │
│         ┌─────────────────────────┼───────────────────────────────┐               │
│         │                   PRESENTATION LAYER                    │               │
│         │                                                         │               │
│         │  ┌───────────────────────────────────────────────────┐  │               │
│         │  │          DOOR 3: WEB DASHBOARD (Next.js 14)       │  │               │
│         │  │                                                   │  │               │
│         │  │  ┌────────────┐ ┌──────────┐ ┌───────────────┐    │  │               │
│         │  │  │ MapLibre   │ │ ECharts  │ │ TanStack      │    │  │               │
│         │  │  │ GL (Maps)  │ │ (Charts) │ │ Table (Data)  │    │  │               │
│         │  │  └────────────┘ └──────────┘ └───────────────┘    │  │               │
│         │  │  ┌────────────┐ ┌──────────┐ ┌───────────────┐    │  │               │
│         │  │  │ shadcn/ui  │ │ Zustand  │ │ React Hook    │    │  │               │
│         │  │  │ (Design)   │ │ (State)  │ │ Form + Zod    │    │  │               │
│         │  │  └────────────┘ └──────────┘ └───────────────┘    │  │               │
│         │  └───────────────────────────────────────────────────┘  │               │
│         └─────────────────────────────────────────────────────────┘               │
│                                                                                   │
│         ┌───────────────────────────────────────────────────────┐                 │
│         │                   OBSERVABILITY LAYER                 │                 │
│         │                                                       │                 │
│         │  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────-─┐ │                 │
│         │  │Prometheus│  │ Grafana  │  │   Loki   │  │Sentry  │ │                 │
│         │  │(Metrics) │  │(Dashbd)  │  │ (Logs)   │  │(Errors)│ │                 │
│         │  └──────────┘  └──────────┘  └──────────┘  └──────-─┘ │                 │
│         └───────────────────────────────────────────────────────┘                 │
│                                                                                   │
└───────────────────────────────────────────────────────────────────────────────────┘
```

---

## What Runs Where

| # | Component | Edge | Cloud | Why |
|---|-----------|------|-------|-----|
| 1 | **FastAPI REST API** | ✅ | ✅ | Both expose REST; edge handles local requests, cloud handles cross-facility |
| 2 | **USSD Session Handler** | ✅ | ❌ | Must respond within 120s; cloud latency too risky |
| 3 | **WhatsApp/Telegram Webhooks** | ✅ | ❌ | Same reason as USSD; also works offline |
| 4 | **AfroBERT NER** | ✅ | ❌ | 280MB model; keeps PII on-device; sub-100ms inference |
| 5 | **PaddleOCR Pipeline** | ✅ | ❌ | 150MB; image processing bandwidth-intensive |
| 6 | **MMS-ASR** | ✅ | ❌ | 1.1GB; audio files large for cloud upload |
| 7 | **Llama 3.2 3B INT4** | ✅ | ❌ | 1.8GB; enables offline chatbot, report summarisation |
| 8 | **Bayesian Resistance Engine** | ✅ | ✅ | Edge for local estimates; cloud for cross-facility aggregation |
| 9 | **Hybrid RAG Retriever** | ✅ | ❌ | ChromaDB is local; embeddings for facility-specific knowledge |
| 10 | **SQLite WAL** | ✅ | ❌ | Local persistent storage on Pi |
| 11 | **ChromaDB** | ✅ | ❌ | Local vector store for RAG |
| 12 | **Redis** | ✅ | ✅ | Edge: session cache, USSD state; Cloud: cache, Celery broker |
| 13 | **PostgreSQL + TimescaleDB + PostGIS** | ❌ | ✅ | Requires managed database for durability and geospatial queries |
| 14 | **Qdrant** | ❌ | ✅ | Cloud vector store for cross-facility search |
| 15 | **Keycloak** | ❌ | ✅ | Centralised identity; too heavy for Pi |
| 16 | **HashiCorp Vault** | ❌ | ✅ | Secret management; edge uses env vars + encrypted config |
| 17 | **Celery Workers** | ❌ | ✅ | Long-running tasks (training, exports); edge uses asyncio |
| 18 | **MLflow** | ❌ | ✅ | Model registry and experiment tracking |
| 19 | **PyTorch Training** | ❌ | ✅ | GPU-intensive; not possible on Pi |
| 20 | **Next.js Dashboard** | ❌ | ✅ | Served via CloudFront CDN |
| 21 | **Prometheus + Grafana** | ✅ | ✅ | Edge: self-monitoring; Cloud: centralised observability |
| 22 | **Sentry** | ✅ | ✅ | Edge: buffered error reports; Cloud: real-time |

---

## Data Flow

### Full 10-Step Sequence

A single AMR report follows this path from CHW submission to dashboard:

```
┌─────────────────────────────────────────────────────────────────────────────┐
│  STEP-BY-STEP: CHW Report → Dashboard                                       │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  Step 1: CHW SUBMISSION                                                     │
│  ┌──────────────────────────────────────────────────────────────────┐       │
│  │ CHW dials *384# (USSD) or sends WhatsApp message:                │       │
│  │                                                                  │       │
│  │ "Patient with fever, cough for 5 days. Given amoxicillin but     │       │
│  │  no improvement after 3 days. Suspect resistance."               │       │
│  └──────────────────────────────┬───────────────────────────────────┘       │
│                                 │                                           │
│  Step 2: GATEWAY RECEIPT                                                    │
│  ┌──────────────────────────────┴───────────────────────────────────┐       │
│  │ Gateway receives raw text. Identifies:                           │       │
│  │   • Channel (USSD/WhatsApp/Telegram/PWA)                         │       │
│  │   • Phone number → look up CHW ID                                │       │
│  │   • Session (new or existing)                                    │       │
│  │   • Language (auto-detect or from profile)                       │       │
│  │   • Timestamp (server receive time)                              │       │
│  └──────────────────────────────┬───────────────────────────────────┘       │
│                                 │                                           │
│  Step 3: LANGUAGE IDENTIFICATION                                            │
│  ┌──────────────────────────────┴───────────────────────────────────┐       │
│  │ fastText LID model classifies:                                   │       │
│  │   • "en" (English) → confidence: 0.92                            │       │
│  │   • Fallback: "yo" (Yoruba), "ha" (Hausa), "ig" (Igbo)           │       │
│  │   • Sets NER language parameter                                  │       │
│  └──────────────────────────────┬───────────────────────────────────┘       │
│                                 │                                           │
│  Step 4: NAMED ENTITY RECOGNITION (AfroBERT)                                │
│  ┌──────────────────────────────┴───────────────────────────────────┐       │
│  │ Two-stage NER pipeline:                                          │       │
│  │   Stage 1: spaCy rule-based matcher (fast, high precision)       │       │
│  │   Stage 2: AfroBERT transformer (captures context, slang)        │       │
│  │   Merge: deduplicate, confidence-weighted combination            │       │
│  │                                                                  │       │
│  │ Extracted entities:                                              │       │
│  │   • SYMPTOM: "fever", "cough"                                    │       │
│  │   • DURATION: "5 days"                                           │       │
│  │   • DRUG: "amoxicillin"                                          │       │
│  │   • OUTCOME: "no improvement"                                    │       │
│  │   • TREATMENT_OUTCOME: "treatment_failure"                       │       │
│  └──────────────────────────────┬───────────────────────────────────┘       │
│                                 │                                           │
│  Step 5: STRUCTURED REPORT CREATION                                         │
│  ┌──────────────────────────────┴───────────────────────────────────┐       │
│  │ Convert raw text + entities into structured Report object:       │       │
│  │                                                                  │       │
│  │ {                                                                │       │
│  │   "id": "rpt_a1b2c3d4",                                          │       │
│  │   "chw_id": "chw_001",                                           │       │
│  │   "facility_id": "fac_lagos_01",                                 │       │
│  │   "patient_age_group": "adult",                                  │       │
│  │   "symptoms": ["fever", "cough"],                                │       │
│  │   "symptom_duration_days": 5,                                    │       │
│  │   "drug_prescribed": "amoxicillin",                              │       │
│  │   "treatment_outcome": "treatment_failure",                      │       │
│  │   "language": "en",                                              │       │
│  │   "source_channel": "ussd",                                      │       │
│  │   "ner_confidence": 0.87,                                        │       │
│  │   "created_at": "2024-01-15T10:30:00Z",                          │       │
│  │   "location": { "lat": 6.524, "lon": 3.379 }                     │       │
│  │ }                                                                │       │
│  └──────────────────────────────┬───────────────────────────────────┘       │
│                                 │                                           │
│  Step 6: LOCAL STORAGE (SQLite WAL)                                         │
│  ┌──────────────────────────────┴───────────────────────────────────┐       │
│  │ Store in local SQLite with WAL mode:                             │       │
│  │   • reports table (structured data)                              │       │
│  │   • report_entities table (NER results)                          │       │
│  │   • sync_queue table (marks as "pending_sync")                   │       │
│  │   • ChromaDB: embed report for local RAG queries                 │       │
│  │                                                                  │       │
│  │ Acknowledge to CHW: "Report received. Thank you!"                │       │
│  └──────────────────────────────┬───────────────────────────────────┘       │
│                                 │                                           │
│  Step 7: BAYESIAN RESISTANCE UPDATE                                         │
│  ┌──────────────────────────────┴───────────────────────────────────┐       │
│  │ PyMC Bayesian engine updates posterior:                          │       │
│  │   • Drug: amoxicillin                                            │       │
│  │   • District: Lagos Mainland                                     │       │
│  │   • Prior: Beta(2, 8) → P(resistance) ≈ 20%                      │       │
│  │   • After N observations: Beta(2+n_fail, 8+n_success)            │       │
│  │   • New posterior: Beta(5, 10) → P(resistance) ≈ 33%             │       │
│  │   • 95% CI: [15%, 55%]                                           │       │
│  │                                                                  │       │
│  │ If P(resistance) > threshold (40%): generate local alert         │       │
│  └──────────────────────────────┬───────────────────────────────────┘       │
│                                 │                                           │
│  Step 8: DELTA SYNC TO CLOUD (when connectivity available)                  │
│  ┌──────────────────────────────┴───────────────────────────────────┐       │
│  │ Sync process (runs every 30s when online):                       │       │
│  │   1. Query sync_queue WHERE status = 'pending'                   │       │
│  │   2. Serialize as delta JSON (only changed rows)                 │       │
│  │   3. Compress with gzip (typical: 70-80% reduction)              │       │
│  │   4. Encrypt with AES-256-GCM (key from Vault/env)               │       │
│  │   5. POST to cloud /api/v1/sync/receive                          │       │
│  │   6. Cloud decrypts, decompresses, applies CRDT merge            │       │
│  │   7. Cloud returns ACK with server_timestamp                     │       │
│  │   8. Edge marks rows as 'synced'                                 │       │
│  │                                                                  │       │
│  │ If sync fails: exponential backoff (1s, 2s, 4s, ..., max 5min)   │       │
│  └──────────────────────────────┬───────────────────────────────────┘       │
│                                 │                                           │
│  Step 9: CLOUD PROCESSING & AGGREGATION                                     │
│  ┌──────────────────────────────┴───────────────────────────────────┐       │
│  │ Cloud services process the synced data:                          │       │
│  │   • Sync Aggregator: merge deltas from all edge nodes            │       │
│  │   • Analytics Engine:                                            │       │
│  │     - Compute resistance index per drug/district/period          │       │
│  │     - Run spatial clustering (DBSCAN) for outbreak detection     │       │
│  │     - Update TimescaleDB hypertables for time-series queries     │       │
│  │     - Generate GeoJSON for map layers                            │       │
│  │   • Alert Rules Engine:                                          │       │
│  │     - Evaluate threshold rules                                   │       │
│  │     - Dispatch alerts (email, SMS, dashboard notification)       │       │
│  │   • ML Pipeline:                                                 │       │
│  │     - Queue data for model retraining (weekly)                   │       │
│  │     - Track evaluation metrics in MLflow                         │       │
│  └──────────────────────────────┬───────────────────────────────────┘       │
│                                 │                                           │
│  Step 10: DASHBOARD VISUALISATION                                           │
│  ┌──────────────────────────────┴───────────────────────────────────┐       │
│  │ Web Dashboard (Door 3) displays:                                 │       │
│  │   • Interactive MapLibre GL map with facility markers            │       │
│  │     - Color-coded by resistance index (green → yellow → red)     │       │
│  │     - Click for facility detail popup                            │       │
│  │     - Filter by drug, date range, resistance level               │       │
│  │   • ECharts trend lines:                                         │       │
│  │     - Resistance index over time (per drug, district, state)     │       │
│  │     - Report volume over time                                    │       │
│  │     - Drug prescription patterns                                 │       │
│  │   • TanStack Table:                                              │       │
│  │     - Paginated, sortable, filterable report list                │       │
│  │     - Export to CSV/Excel                                        │       │
│  │   • Alert feed:                                                  │       │
│  │     - Real-time WebSocket updates                                │       │
│  │     - Severity levels, acknowledge/dismiss actions               │       │
│  │                                                                  │       │
│  │ Role-based access:                                               │       │
│  │   • Facility Admin: sees own facility only                       │       │
│  │   • State Admin: sees all facilities in state                    │       │
│  │   • National Admin: sees all facilities, can export GLASS data   │       │
│  └──────────────────────────────────────────────────────────────────┘       │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

### Data Flow Diagram

```
  ┌──────┐    ┌──────┐    ┌──────┐    ┌──────┐
  │ CHW  │    │ CHW  │    │ CHW  │    │ CHW  │
  │USSD  │    │WA Msg│    │TG Msg│    │PWA   │
  └──┬───┘    └──┬───┘    └──┬───┘    └──┬───┘
     │           │           │           │
     └───────────┴─────┬─────┴───────────┘
                       │
              ┌────────┴────────┐
              │  Africa's Talk  │◀──── Africa's Talking API (USSD/SMS)
              │  Gateway        │
              └────────┬────────┘
                       │
     ┌─────────────────┼──────────────────┐
     │                 │                  │
┌────┴────┐    ┌───────┴──────┐    ┌──────┴──────┐
│WhatsApp │    │   Telegram   │    │  PWA Direct │
│Webhook  │    │   Webhook    │    │  HTTPS      │
└────┬────┘    └───────┬──────┘    └──────┬──────┘
     │                 │                  │
     └─────────────────┼──────────────────┘
                       │
              ┌────────┴────────┐
              │ MESSAGE ROUTER  │
              │ (Edge FastAPI)  │
              └────────┬────────┘
                       │
          ┌────────────┼────────────┐
          │            │            │
     ┌────┴────┐ ┌────┴────┐ ┌────┴────┐
     │ LID     │ │ NER     │ │ Report  │
     │fastText │ │AfroBERT │ │ Builder │
     └─────────┘ └────┬────┘ └────┬────┘
                      │            │
                      └──────┬─────┘
                             │
                    ┌────────┴────────┐
                    │   SQLite WAL    │
                    │   + ChromaDB    │
                    └────────┬────────┘
                             │
                    ┌────────┴────────┐
                    │  Bayesian       │
                    │  Engine (PyMC)  │
                    └────────┬────────┘
                             │
                    ┌────────┴────────┐
                    │  Delta Sync     │──── AES-256-GCM ────▶ Cloud
                    │  (gzip+CRDT)    │
                    └─────────────────┘
```

---

## Sync Architecture

The sync system is the most critical and complex part of UDARA AI. It must handle
intermittent connectivity, data conflicts, bandwidth constraints, and security
requirements — all while being invisible to the end user.

### Delta-Only JSON

We never send full database dumps. Instead, we track row-level changes using a
`sync_queue` table and send only the deltas.

```sql
-- sync_queue schema (runs on both edge and cloud)
CREATE TABLE sync_queue (
    id              INTEGER PRIMARY KEY AUTOINCREMENT,
    table_name      TEXT NOT NULL,
    row_id          TEXT NOT NULL,
    operation       TEXT NOT NULL CHECK (operation IN ('INSERT', 'UPDATE', 'DELETE')),
    data            JSON NOT NULL,            -- Full row as JSON (for INSERT/UPDATE)
    previous_data   JSON,                     -- Previous row state (for conflict resolution)
    edge_node_id    TEXT NOT NULL,
    created_at      TEXT NOT NULL DEFAULT (datetime('now')),
    synced_at       TEXT,                      -- NULL = pending, timestamp = synced
    sync_attempt    INTEGER DEFAULT 0,
    last_error      TEXT,
    
    -- Composite index for efficient pending query
    CONSTRAINT uq_table_row UNIQUE (table_name, row_id)
);

CREATE INDEX idx_sync_pending ON sync_queue(synced_at) WHERE synced_at IS NULL;
CREATE INDEX idx_sync_node ON sync_queue(edge_node_id);
```

```python
# edge/app/sync/delta.py
"""Delta-only change tracking for edge→cloud synchronization."""

from __future__ import annotations

import json
import logging
from datetime import datetime, timezone
from enum import Enum
from typing import Any

from sqlalchemy import text
from sqlalchemy.ext.asyncio import AsyncSession

logger = logging.getLogger(__name__)


class SyncOperation(str, Enum):
    INSERT = "INSERT"
    UPDATE = "UPDATE"
    DELETE = "DELETE"


class DeltaTracker:
    """
    Tracks row-level changes in SQLite tables and queues them for sync.
    
    Usage:
        tracker = DeltaTracker(db_session, edge_node_id="edge_lagos_01")
        await tracker.track_insert("reports", report_id, report_data)
        await tracker.track_update("reports", report_id, old_data, new_data)
        await tracker.get_pending_deltas(limit=100)
        await tracker.mark_synced(delta_ids=[1, 2, 3])
    """
    
    SYNC_TABLES = [
        "reports",
        "report_entities",
        "facilities",
        "chw_profiles",
        "resistance_estimates",
        "local_alerts",
    ]
    
    def __init__(self, session: AsyncSession, edge_node_id: str) -> None:
        self.session = session
        self.edge_node_id = edge_node_id
    
    async def track_insert(
        self,
        table_name: str,
        row_id: str,
        data: dict[str, Any],
    ) -> int:
        """Queue a new row insertion for sync."""
        assert table_name in self.SYNC_TABLES, f"Unknown table: {table_name}"
        
        result = await self.session.execute(
            text("""
                INSERT INTO sync_queue (table_name, row_id, operation, data, edge_node_id)
                VALUES (:table_name, :row_id, :operation, :data, :edge_node_id)
                ON CONFLICT (table_name, row_id) 
                DO UPDATE SET 
                    operation = :operation,
                    data = :data,
                    created_at = datetime('now'),
                    synced_at = NULL,
                    sync_attempt = 0
            """),
            {
                "table_name": table_name,
                "row_id": row_id,
                "operation": SyncOperation.INSERT.value,
                "data": json.dumps(data),
                "edge_node_id": self.edge_node_id,
            },
        )
        await self.session.commit()
        logger.info("Queued INSERT for %s:%s", table_name, row_id)
        return result.lastrowid
    
    async def track_update(
        self,
        table_name: str,
        row_id: str,
        previous_data: dict[str, Any],
        new_data: dict[str, Any],
    ) -> int:
        """Queue a row update for sync, preserving previous state for CRDT."""
        assert table_name in self.SYNC_TABLES
        
        result = await self.session.execute(
            text("""
                INSERT INTO sync_queue (table_name, row_id, operation, data, 
                                        previous_data, edge_node_id)
                VALUES (:table_name, :row_id, :operation, :data, 
                        :previous_data, :edge_node_id)
                ON CONFLICT (table_name, row_id)
                DO UPDATE SET
                    operation = :operation,
                    data = :data,
                    previous_data = :previous_data,
                    created_at = datetime('now'),
                    synced_at = NULL,
                    sync_attempt = 0
            """),
            {
                "table_name": table_name,
                "row_id": row_id,
                "operation": SyncOperation.UPDATE.value,
                "data": json.dumps(new_data),
                "previous_data": json.dumps(previous_data),
                "edge_node_id": self.edge_node_id,
            },
        )
        await self.session.commit()
        return result.lastrowid
    
    async def get_pending_deltas(self, limit: int = 100) -> list[dict]:
        """Fetch pending (un-synced) deltas, ordered by creation time."""
        result = await self.session.execute(
            text("""
                SELECT id, table_name, row_id, operation, data, previous_data,
                       edge_node_id, created_at
                FROM sync_queue
                WHERE synced_at IS NULL
                ORDER BY created_at ASC
                LIMIT :limit
            """),
            {"limit": limit},
        )
        
        rows = result.mappings().all()
        deltas = []
        for row in rows:
            deltas.append({
                "id": row["id"],
                "table_name": row["table_name"],
                "row_id": row["row_id"],
                "operation": row["operation"],
                "data": json.loads(row["data"]),
                "previous_data": json.loads(row["previous_data"]) if row["previous_data"] else None,
                "edge_node_id": row["edge_node_id"],
                "created_at": row["created_at"],
            })
        
        logger.info("Fetched %d pending deltas (limit=%d)", len(deltas), limit)
        return deltas
    
    async def mark_synced(self, delta_ids: list[int]) -> None:
        """Mark deltas as successfully synced."""
        if not delta_ids:
            return
        
        placeholders = ",".join(f":id_{i}" for i in range(len(delta_ids)))
        params = {f"id_{i}": did for i, did in enumerate(delta_ids)}
        
        await self.session.execute(
            text(f"""
                UPDATE sync_queue 
                SET synced_at = datetime('now')
                WHERE id IN ({placeholders})
            """),
            params,
        )
        await self.session.commit()
        logger.info("Marked %d deltas as synced", len(delta_ids))
    
    async def mark_failed(self, delta_ids: list[int], error: str) -> None:
        """Increment sync_attempt and record error for failed deltas."""
        if not delta_ids:
            return
        
        placeholders = ",".join(f":id_{i}" for i in range(len(delta_ids)))
        params = {f"id_{i}": did for i, did in enumerate(delta_ids)}
        params["error"] = error
        
        await self.session.execute(
            text(f"""
                UPDATE sync_queue
                SET sync_attempt = sync_attempt + 1,
                    last_error = :error
                WHERE id IN ({placeholders})
            """),
            params,
        )
        await self.session.commit()
    
    async def get_sync_stats(self) -> dict[str, int]:
        """Return counts of pending, synced, and failed deltas."""
        result = await self.session.execute(text("""
            SELECT 
                COUNT(*) FILTER (WHERE synced_at IS NULL) as pending,
                COUNT(*) FILTER (WHERE synced_at IS NOT NULL) as synced,
                COUNT(*) FILTER (WHERE last_error IS NOT NULL AND synced_at IS NULL) as failed
            FROM sync_queue
        """))
        row = result.mappings().one()
        return dict(row)
```

### Compression

All sync payloads are gzip-compressed before encryption. Typical compression ratios:

| Data Type | Raw Size | Compressed | Ratio |
|-----------|----------|------------|-------|
| 100 report deltas | ~85 KB | ~18 KB | 79% |
| 1000 report deltas | ~850 KB | ~150 KB | 82% |
| 50 image metadata (no images) | ~12 KB | ~3 KB | 75% |
| Resistance estimate batch | ~5 KB | ~1.2 KB | 76% |

```python
import gzip
import io
import json


def compress_payload(data: dict | list) -> bytes:
    """Compress JSON payload with gzip (level 6 = balance of speed/ratio)."""
    json_bytes = json.dumps(data, default=str).encode("utf-8")
    buf = io.BytesIO()
    with gzip.GzipFile(fileobj=buf, mode="wb", compresslevel=6) as f:
        f.write(json_bytes)
    return buf.getvalue()


def decompress_payload(compressed: bytes) -> dict | list:
    """Decompress gzip payload back to JSON."""
    buf = io.BytesIO(compressed)
    with gzip.GzipFile(fileobj=buf, mode="rb") as f:
        json_bytes = f.read()
    return json.loads(json_bytes)
```

### Encryption

All sync payloads are encrypted with AES-256-GCM before transmission.

```python
# shared/python/udara_shared/crypto.py
"""AES-256-GCM encryption for sync payloads."""

from __future__ import annotations

import os
import base64
import json
import gzip
import io
from dataclasses import dataclass

from cryptography.hazmat.primitives.ciphers.aead import AESGCM


@dataclass
class EncryptedPayload:
    """Container for an encrypted sync payload."""
    ciphertext: bytes
    nonce: bytes
    associated_data: bytes  # AAD: edge_node_id + timestamp


class SyncEncryptor:
    """
    AES-256-GCM encryptor/decryptor for edge↔cloud sync.
    
    Key rotation:
    - Keys are 256-bit (32 bytes)
    - New key generated every 90 days
    - Old keys kept for 30 days (decryption only) for graceful rotation
    - Keys stored in HashiCorp Vault (cloud) or encrypted env vars (edge)
    """
    
    NONCE_SIZE = 12  # 96 bits, standard for GCM
    KEY_SIZE = 32    # 256 bits
    
    def __init__(self, key: bytes) -> None:
        if len(key) != self.KEY_SIZE:
            raise ValueError(f"Key must be {self.KEY_SIZE} bytes, got {len(key)}")
        self._aesgcm = AESGCM(key)
    
    def encrypt(
        self,
        data: bytes,
        edge_node_id: str,
        timestamp: str,
    ) -> EncryptedPayload:
        """
        Encrypt data with AES-256-GCM.
        
        Args:
            data: Raw bytes to encrypt (typically gzip-compressed JSON)
            edge_node_id: Included as AAD for authentication
            timestamp: ISO 8601 timestamp included as AAD
        
        Returns:
            EncryptedPayload with ciphertext, nonce, and AAD
        """
        nonce = os.urandom(self.NONCE_SIZE)
        associated_data = f"{edge_node_id}:{timestamp}".encode("utf-8")
        
        ciphertext = self._aesgcm.encrypt(nonce, data, associated_data)
        
        return EncryptedPayload(
            ciphertext=ciphertext,
            nonce=nonce,
            associated_data=associated_data,
        )
    
    def decrypt(
        self,
        payload: EncryptedPayload,
        expected_node_id: str | None = None,
    ) -> bytes:
        """
        Decrypt and verify AES-256-GCM payload.
        
        Raises cryptography.exceptions.InvalidTag if:
            - Key is wrong
            - Data was tampered with
            - AAD doesn't match
        """
        plaintext = self._aesgcm.decrypt(
            payload.nonce,
            payload.ciphertext,
            payload.associated_data,
        )
        
        # Optional: verify edge_node_id in AAD
        if expected_node_id:
            aad_str = payload.associated_data.decode("utf-8")
            node_id_from_aad = aad_str.split(":")[0]
            if node_id_from_aad != expected_node_id:
                raise ValueError(
                    f"AAD node ID mismatch: expected {expected_node_id}, "
                    f"got {node_id_from_aad}"
                )
        
        return plaintext
    
    @classmethod
    def generate_key(cls) -> bytes:
        """Generate a new random 256-bit key."""
        return os.urandom(cls.KEY_SIZE)
    
    @classmethod
    def from_base64(cls, key_b64: str) -> SyncEncryptor:
        """Create encryptor from base64-encoded key."""
        key = base64.b64decode(key_b64)
        return cls(key)
    
    def to_base64(self) -> str:
        """Export the key as base64 (for storage in Vault)."""
        return base64.b64encode(
            self._aesgcm._key  # type: ignore[attr-defined]
        ).decode("utf-8")
```

### CRDT via Yjs

Conflict resolution uses a CRDT (Conflict-free Replicated Data Type) approach
via the Yjs library. Each edge node maintains a local Yjs document, and the
cloud merges documents from all nodes.

```python
# edge/app/sync/crdt.py
"""CRDT-based conflict resolution using Yjs for sync conflicts."""

from __future__ import annotations

import logging
from typing import Any

logger = logging.getLogger(__name__)


class CRDTMergeStrategy:
    """
    Merge strategy for conflicting deltas using CRDT principles.
    
    For most AMR data, we use:
    - Reports: Last-writer-wins (LWW) based on created_at timestamp
    - Resistance estimates: Mathematical merge (combine Beta distributions)
    - Facility data: Manual resolution + audit log
    
    Priority order for timestamps (when equal):
    1. Higher edge_node_id (lexicographic) wins
    2. Later created_at wins
    3. Cloud version wins (serves as tiebreaker)
    """
    
    @staticmethod
    def merge_reports(
        edge_data: dict[str, Any],
        cloud_data: dict[str, Any] | None,
    ) -> dict[str, Any]:
        """
        Merge a report from edge with existing cloud data.
        
        Strategy: Last-writer-wins on created_at.
        If edge report is newer, it wins. If same timestamp,
        compare edge_node_id.
        """
        if cloud_data is None:
            return edge_data  # No conflict, edge wins
        
        edge_ts = edge_data.get("created_at", "")
        cloud_ts = cloud_data.get("created_at", "")
        
        if edge_ts > cloud_ts:
            logger.debug("Edge report wins (newer): %s", edge_data.get("id"))
            return edge_data
        elif cloud_ts > edge_ts:
            logger.debug("Cloud report wins (newer): %s", cloud_data.get("id"))
            return cloud_data
        else:
            # Same timestamp: use edge_node_id as tiebreaker
            edge_node = edge_data.get("edge_node_id", "")
            cloud_node = cloud_data.get("edge_node_id", "")
            if edge_node > cloud_node:
                return edge_data
            else:
                return cloud_data
    
    @staticmethod
    def merge_resistance_estimates(
        edge_data: dict[str, Any],
        cloud_data: dict[str, Any] | None,
    ) -> dict[str, Any]:
        """
        Merge resistance estimates using Beta distribution combination.
        
        Beta(a1,b1) + Beta(a2,b2) = Beta(a1+a2, b1+b2)
        This is mathematically correct for combining independent observations.
        """
        if cloud_data is None:
            return edge_data
        
        # Extract Beta parameters
        edge_alpha = edge_data.get("beta_alpha", 1.0)
        edge_beta = edge_data.get("beta_beta", 1.0)
        cloud_alpha = cloud_data.get("beta_alpha", 1.0)
        cloud_beta = cloud_data.get("beta_beta", 1.0)
        
        merged = {
            **edge_data,
            "beta_alpha": edge_alpha + cloud_alpha,
            "beta_beta": edge_beta + cloud_beta,
            "observation_count": (
                edge_data.get("observation_count", 0) 
                + cloud_data.get("observation_count", 0)
            ),
            "merged_from": [
                edge_data.get("edge_node_id", "unknown"),
                cloud_data.get("edge_node_id", "cloud"),
            ],
        }
        
        # Recompute point estimate and CI
        total = merged["beta_alpha"] + merged["beta_beta"]
        merged["resistance_rate"] = round(merged["beta_alpha"] / total, 4)
        
        logger.info(
            "Merged resistance estimates: %s (α=%.1f, β=%.1f, P=%.2f)",
            merged.get("drug", "unknown"),
            merged["beta_alpha"],
            merged["beta_beta"],
            merged["resistance_rate"],
        )
        
        return merged
```

### Exponential Backoff

```python
# edge/app/sync/transport.py
"""HTTP sync transport with exponential backoff and jitter."""

from __future__ import annotations

import asyncio
import logging
import random
import time
from dataclasses import dataclass, field
from enum import Enum
from typing import Callable

import httpx

logger = logging.getLogger(__name__)


class SyncStatus(str, Enum):
    ONLINE = "online"
    OFFLINE = "offline"
    DEGRADED = "degraded"


@dataclass
class BackoffConfig:
    """Exponential backoff configuration for sync retries."""
    base_delay: float = 1.0           # Initial delay in seconds
    max_delay: float = 300.0          # Maximum delay (5 minutes)
    multiplier: float = 2.0           # Backoff multiplier
    jitter_range: float = 0.1         # ±10% jitter to prevent thundering herd
    max_retries: int = 10             # Max retries before giving up
    health_check_interval: float = 30.0  # Connectivity check every 30s


@dataclass
class SyncStats:
    """Statistics for the sync transport."""
    total_syncs: int = 0
    successful_syncs: int = 0
    failed_syncs: int = 0
    bytes_uploaded: int = 0
    bytes_downloaded: int = 0
    last_sync_at: float | None = None
    last_error: str | None = None
    current_backoff: float = 1.0
    consecutive_failures: int = 0
    status: SyncStatus = SyncStatus.OFFLINE


class SyncTransport:
    """
    HTTP transport for edge→cloud sync with exponential backoff.
    
    Features:
    - Automatic connectivity detection
    - Exponential backoff with jitter
    - Bandwidth-aware batching
    - Timeout handling
    - USB fallback trigger
    """
    
    def __init__(
        self,
        cloud_url: str,
        edge_node_id: str,
        encryptor: Any,  # SyncEncryptor
        config: BackoffConfig | None = None,
    ) -> None:
        self.cloud_url = cloud_url.rstrip("/")
        self.edge_node_id = edge_node_id
        self.encryptor = encryptor
        self.config = config or BackoffConfig()
        self.stats = SyncStats()
        self._client = httpx.AsyncClient(
            timeout=httpx.Timeout(30.0, connect=10.0),
            limits=httpx.Limits(max_connections=2, max_keepalive_connections=1),
        )
    
    def _calculate_backoff(self) -> float:
        """Calculate next backoff delay with jitter."""
        delay = min(
            self.config.base_delay * (self.config.multiplier ** self.stats.consecutive_failures),
            self.config.max_delay,
        )
        # Add jitter: ±jitter_range%
        jitter = delay * self.config.jitter_range * (random.random() * 2 - 1)
        return max(0.1, delay + jitter)
    
    async def check_connectivity(self) -> bool:
        """Check if cloud is reachable."""
        try:
            resp = await self._client.get(
                f"{self.cloud_url}/api/v1/health",
                timeout=5.0,
            )
            return resp.status_code == 200
        except (httpx.ConnectError, httpx.TimeoutException):
            return False
    
    async def sync(self, payload: bytes) -> bool:
        """
        Attempt to sync payload to cloud with backoff.
        
        Returns True if sync succeeded, False otherwise.
        """
        timestamp = time.strftime("%Y-%m-%dT%H:%M:%SZ", time.gmtime())
        
        encrypted = self.encryptor.encrypt(payload, self.edge_node_id, timestamp)
        
        # Encode for HTTP transmission
        body = {
            "edge_node_id": self.edge_node_id,
            "timestamp": timestamp,
            "nonce": encrypted.nonce.hex(),
            "ciphertext": encrypted.ciphertext.hex(),
            "aad": encrypted.associated_data.hex(),
        }
        
        for attempt in range(self.config.max_retries):
            try:
                resp = await self._client.post(
                    f"{self.cloud_url}/api/v1/sync/receive",
                    json=body,
                    timeout=30.0,
                )
                
                if resp.status_code == 200:
                    self.stats.successful_syncs += 1
                    self.stats.total_syncs += 1
                    self.stats.bytes_uploaded += len(payload)
                    self.stats.last_sync_at = time.time()
                    self.stats.consecutive_failures = 0
                    self.stats.current_backoff = self.config.base_delay
                    self.stats.status = SyncStatus.ONLINE
                    logger.info("Sync succeeded (attempt %d)", attempt + 1)
                    return True
                
                elif resp.status_code == 409:
                    # Conflict: cloud has newer data, need CRDT merge
                    logger.warning("Sync conflict detected, CRDT merge required")
                    self.stats.last_error = "CONFLICT"
                    return False  # Caller should handle merge
                
                else:
                    self.stats.last_error = f"HTTP {resp.status_code}"
                    logger.warning(
                        "Sync failed: HTTP %d (attempt %d/%d)",
                        resp.status_code, attempt + 1, self.config.max_retries,
                    )
                    
            except (httpx.ConnectError, httpx.TimeoutException) as exc:
                self.stats.last_error = str(exc)
                logger.warning(
                    "Sync connection error: %s (attempt %d/%d)",
                    exc, attempt + 1, self.config.max_retries,
                )
            
            self.stats.consecutive_failures += 1
            self.stats.current_backoff = self._calculate_backoff()
            
            if self.stats.consecutive_failures >= 3:
                self.stats.status = SyncStatus.OFFLINE
            
            # Wait before retry
            logger.info(
                "Retrying in %.1fs (backoff) — consecutive failures: %d",
                self.stats.current_backoff,
                self.stats.consecutive_failures,
            )
            await asyncio.sleep(self.stats.current_backoff)
        
        self.stats.failed_syncs += 1
        self.stats.total_syncs += 1
        self.stats.status = SyncStatus.OFFLINE
        logger.error(
            "Sync failed after %d attempts. Last error: %s",
            self.config.max_retries, self.stats.last_error,
        )
        return False
    
    async def close(self) -> None:
        await self._client.aclose()
```

### USB Fallback

When connectivity is lost for an extended period (>24 hours), the system provides
a USB-based fallback for data transfer.

```python
# edge/scripts/usb_sync.py
"""
USB-based sync fallback for extended connectivity outages.

Workflow:
1. Admin inserts USB drive into Pi
2. Script auto-detects USB (udev rule triggers)
3. Exports pending deltas as encrypted archive to USB
4. Admin physically transports USB to location with connectivity
5. Upload script sends archive to cloud
6. Returns ACK file on USB
7. Admin inserts USB back into Pi
8. ACK is processed, deltas marked as synced

File structure on USB:
  /udara-sync/
    outgoing/
      {edge_node_id}_{timestamp}.udara.gz.aes   # Encrypted sync archive
    incoming/
      {cloud_timestamp}.ack.udara                  # Sync acknowledgment
    logs/
      usb_sync.log
"""

import os
import json
import gzip
import logging
import shutil
from datetime import datetime, timezone
from pathlib import Path

logger = logging.getLogger(__name__)

USB_LABEL = "UDARA-SYNC"
USB_MOUNT_POINT = Path("/media/udara-sync")
SYNC_DIR = USB_MOUNT_POINT / "udara-sync"
OUTGOING_DIR = SYNC_DIR / "outgoing"
INCOMING_DIR = SYNC_DIR / "incoming"


def export_to_usb(
    edge_node_id: str,
    pending_deltas: list[dict],
    encryptor: Any,
) -> Path:
    """
    Export pending deltas to USB drive as encrypted archive.
    
    Args:
        edge_node_id: Unique identifier for this edge node
        pending_deltas: List of pending delta records
        encryptor: SyncEncryptor instance
    
    Returns:
        Path to the created archive file
    """
    OUTGOING_DIR.mkdir(parents=True, exist_ok=True)
    
    timestamp = datetime.now(timezone.utc).strftime("%Y%m%dT%H%M%SZ")
    filename = f"{edge_node_id}_{timestamp}.udara.gz.aes"
    filepath = OUTGOING_DIR / filename
    
    # Step 1: Serialize deltas to JSON
    payload = {
        "edge_node_id": edge_node_id,
        "exported_at": timestamp,
        "delta_count": len(pending_deltas),
        "deltas": pending_deltas,
    }
    json_bytes = json.dumps(payload, default=str).encode("utf-8")
    
    # Step 2: Gzip compress
    buf = io.BytesIO()
    with gzip.GzipFile(fileobj=buf, mode="wb", compresslevel=6) as f:
        f.write(json_bytes)
    compressed = buf.getvalue()
    
    # Step 3: AES-256-GCM encrypt
    encrypted = encryptor.encrypt(compressed, edge_node_id, timestamp)
    
    # Step 4: Write to USB
    archive = {
        "nonce": encrypted.nonce.hex(),
        "aad": encrypted.associated_data.hex(),
        "ciphertext": encrypted.ciphertext.hex(),
    }
    
    filepath.write_text(json.dumps(archive))
    logger.info(
        "Exported %d deltas to USB: %s (%d bytes)",
        len(pending_deltas), filename, len(compressed),
    )
    
    return filepath


def process_usb_ack() -> list[int]:
    """
    Process acknowledgment files from cloud on USB drive.
    
    Returns list of delta IDs that were confirmed as synced.
    """
    acked_ids = []
    
    if not INCOMING_DIR.exists():
        logger.warning("USB incoming directory does not exist")
        return acked_ids
    
    for ack_file in INCOMING_DIR.glob("*.ack.udara"):
        try:
            data = json.loads(ack_file.read_text())
            acked_ids.extend(data.get("synced_delta_ids", []))
            ack_file.unlink()  # Remove processed ACK
            logger.info("Processed ACK: %d deltas confirmed", len(data.get("synced_delta_ids", [])))
        except Exception as exc:
            logger.error("Failed to process ACK file %s: %s", ack_file, exc)
    
    return acked_ids
```

---

## Edge Node Specifications

### Hardware

| Component | Specification | Notes |
|-----------|--------------|-------|
| **Board** | Raspberry Pi 5 Model B Rev 1.0 | Broadcom BCM2712, 4× Cortex-A76 @ 2.4GHz |
| **RAM** | 8 GB LPDDR4X-4267 | Minimum 8GB required for AI models |
| **Storage** | 128 GB SanDisk High Endurance microSD | A2 rated, high TBW for write-heavy SQLite |
| **Backup Storage** | 256 GB USB SSD (Samsung T7) | USB 3.0, for model storage + USB sync |
| **Power** | Official Raspberry Pi 27W USB-C PSU | With UPS HAT for brownout protection |
| **UPS** | Waveshare Pi UPS Plus (18650) | ~30 min backup, clean shutdown |
| **Networking** | Onboard Wi-Fi 5 GHz + Ethernet | Ethernet preferred when available |
| **Cellular** | Huawei E3372 USB LTE dongle | MTN/Airtel/Glo SIM with data plan |
| **Case** | Argon ONE M.2 Pi 5 Case | Passive cooling, aluminum heatsink |
| **Display** | Optional: 7" Official Touch Screen | For local dashboard (Door 3 lite) |
| **Thermal** | Argon ONE active cooling fan | Thermostatic, kicks in at 60°C |

### Software Stack

| Layer | Component | Version | Purpose |
|-------|-----------|---------|---------|
| **OS** | Ubuntu Core 22.04 | 22.04 | Immutable, OTA updates, snap-based |
| **Fleet Mgmt** | Balena Cloud | Latest | Remote fleet management, OTA deploys |
| **Runtime** | Python 3.11 | 3.11.x | Application runtime (uv-managed) |
| **ASGI** | Uvicorn | Latest | ASGI server for FastAPI |
| **Framework** | FastAPI | 0.115+ | API framework |
| **Database** | SQLite 3 (WAL mode) | 3.44+ | Local persistent storage |
| **Vector DB** | ChromaDB | 0.5+ | Local embeddings for RAG |
| **Cache** | Redis | 7.x | Session store, rate limiting, queues |
| **ML Runtime** | llama.cpp | Latest | LLM inference (Llama 3.2 3B) |
| **ML Runtime** | ONNX Runtime | 1.18+ | NER, OCR, LID model inference |
| **ML Runtime** | PyMC | 5.x | Bayesian resistance estimation |
| **NLP** | spaCy | 3.7+ | Rule-based NER (stage 1) |
| **NLP** | HuggingFace Transformers | 4.40+ | Transformer model loading |
| **NLP** | fastText | 0.9.2 | Language identification |
| **NLP** | Presidio | 2.2+ | PII detection and redaction |
| **OCR** | PaddleOCR | 2.7+ | Text extraction from images |
| **ASR** | MMS-ASR | Latest | Speech-to-text (1100+ languages) |
| **CRDT** | Yjs (via y-py) | Latest | Conflict resolution |
| **Bot** | aiogram | 3.x | Telegram bot framework |
| **Comms** | africa-talking-python | Latest | USSD + SMS integration |
| **Crypto** | cryptography | 42+ | AES-256-GCM encryption |

### RAM Budget

The Pi 5 has 8 GB total RAM. The OS + system services consume ~1.5 GB. Python
runtime + FastAPI takes ~300 MB. This leaves ~6.2 GB for AI models, but we set
a hard budget of 3.5 GB for models (leaving headroom for data processing).

| Component | Resident RAM | Peak RAM | Notes |
|-----------|-------------|----------|-------|
| **Ubuntu Core + system** | 450 MB | 600 MB | Kernel, systemd, snapd |
| **Balena agent** | 80 MB | 120 MB | Fleet management |
| **Python 3.11 runtime** | 120 MB | 200 MB | Interpreter + stdlib |
| **FastAPI + Uvicorn** | 80 MB | 150 MB | 4 worker processes |
| **Redis 7** | 50 MB | 100 MB | Session + cache |
| **SQLite (WAL)** | 30 MB | 80 MB | Depends on dataset size |
| **ChromaDB** | 100 MB | 200 MB | Vector store, depends on docs |
| **Celery (edge tasks)** | 40 MB | 80 MB | Local async task queue |
| **--- TOTAL SYSTEM ---** | **~950 MB** | **~1,530 MB** | |
| | | | |
| **AfroBERT NER (INT8)** | 280 MB | 320 MB | XLM-RoBERTa base, always loaded |
| **PaddleOCR (INT8)** | 150 MB | 200 MB | Loaded on demand (OCR request) |
| **MMS-ASR (INT8)** | 350 MB | 400 MB | Loaded on demand (audio request) |
| **Llama 3.2 3B INT4** | 1,800 MB | 2,000 MB | llama.cpp, loaded on demand |
| **fastText LID** | 30 MB | 50 MB | Always loaded (tiny) |
| **PyMC Bayesian** | 50 MB | 80 MB | Loaded on demand |
| **multilingual-e5 (embeddings)** | 400 MB | 500 MB | Always loaded for RAG |
| **Whoosh BM25** | 20 MB | 40 MB | Always loaded for RAG |
| **spaCy NLP (stage 1)** | 80 MB | 100 MB | Always loaded |
| **--- TOTAL AI ---** | **~3,160 MB** | **~3,690 MB** | |
| | | | |
| **GRAND TOTAL** | **~4,110 MB** | **~5,220 MB** | Within 8 GB with headroom |

> **Note:** Not all models are loaded simultaneously. The `ModelRegistry` enforces a
> 3.5 GB budget for AI models. On-demand models (OCR, ASR, LLM, PyMC) are loaded
> when needed and evicted when budget is exceeded. Always-loaded models (NER, LID,
> embeddings, spaCy, Whoosh) total ~810 MB.

---

## Failure Modes

### Connectivity Lost

```
┌──────────────────────────────────────────────────────────────────┐
│  SCENARIO: Internet connectivity lost for >5 minutes             │
├──────────────────────────────────────────────────────────────────┤
│                                                                  │
│  DETECTION:                                                      │
│  • Sync transport detects 3 consecutive failed attempts          │
│  • Health check task (every 30s) confirms offline status         │
│  • Status changes: ONLINE → DEGRADED → OFFLINE                   │
│                                                                  │
│  IMPACT:                                                         │
│  ✅ Door 1 (USSD): UNAFFECTED — runs entirely on edge            │
│  ✅ Door 2 (WhatsApp): PARTIAL — messages queue locally,         │
│     responses sent when connectivity returns (may time out)      │
│  ✅ Door 2 (Telegram): PARTIAL — same as WhatsApp                │
│  ✅ Door 2 (PWA): UNAFFECTED — offline mode with service worker  │
│  ✅ Data collection: UNAFFECTED — SQLite continues locally       │
│  ✅ NER/OCR/ASR: UNAFFECTED — models run on Pi                   │
│  ✅ Bayesian estimates: UNAFFECTED — PyMC runs locally           │
│  ❌ Door 3 (Web Dashboard): CANNOT LOAD — cloud-dependent         │
│  ❌ Cross-facility sync: BLOCKED — deltas queue in sync_queue     │
│  ❌ Model updates: BLOCKED — cannot download new models           │
│  ❌ National alerts: BLOCKED — cloud dispatch fails               │
│                                                                  │
│  RECOVERY:                                                       │
│  1. Connectivity returns                                         │
│  2. Backoff timer resets                                         │
│  3. Sync transport flushes queue (oldest first)                  │
│  4. If queue > 1000 deltas: batch into groups of 100             │
│  5. Backfill any missed model updates                            │
│                                                                  │
│  DURING OUTAGE (>24h):                                           │
│  • Admin is notified via Pi LED pattern (blinking red)           │
│  • USB fallback is available for critical data                   │
│  • Local resistance dashboard still works (PWA on Pi)            │
│                                                                  │
└──────────────────────────────────────────────────────────────────┘
```

### Pi Reboots Unexpectedly

```
┌──────────────────────────────────────────────────────────────────┐
│  SCENARIO: Power loss or kernel panic causes Pi reboot           │
├──────────────────────────────────────────────────────────────────┤
│                                                                  │
│  PREVENTION:                                                     │
│  • UPS HAT provides ~30 min backup                               │
│  • systemd listens for power events, triggers clean shutdown     │
│  • High-endurance microSD card rated for sustained writes        │
│  • SQLite WAL mode provides crash recovery                       │
│  • Regular fsck on boot (systemd-fsck@.service)                  │
│                                                                  │
│  ON REBOOT:                                                      │
│  1. Balena agent auto-starts all services                        │
│  2. FastAPI health check: /health returns 503 until models loaded│
│  3. Model registry: loads always-on models (NER, LID, embed)     │
│  4. SQLite: WAL replay recovers any uncommitted transactions     │
│  5. sync_queue: check for pending deltas, resume sync            │
│  6. USSD sessions: expired (>120s TTL), clean from Redis         │
│                                                                  │
│  DATA SAFETY:                                                    │
│  ✅ Reports: Safe — SQLite WAL ensures atomic commits            │
│  ✅ NER results: Safe — stored in SQLite alongside reports       │
│  ✅ Sync queue: Safe — pending deltas survive reboot             │
│  ⚠️ In-flight NER: Lost — report is saved, NER runs on retry     │
│  ⚠️ In-flight OCR: Lost — image is saved, OCR runs on retry      │
│  ❌ Redis cache: Cleared — sessions expired, non-critical         │
│                                                                  │
│  RECOVERY TIME: ~45 seconds (Balena auto-restart)                │
│                                                                  │
└──────────────────────────────────────────────────────────────────┘
```

### Model Out of Memory

```
┌──────────────────────────────────────────────────────────────────┐
│  SCENARIO: Request requires loading model that exceeds budget    │
├──────────────────────────────────────────────────────────────────┤
│                                                                  │
│  EXAMPLE: LLM (1.8 GB) requested while OCR (200 MB) and          │
│          ASR (400 MB) are already loaded. Total would be         │
│          2.4 GB on-demand + 810 MB always = 3.21 GB.             │
│          Budget is 3.5 GB, so this FITS. But if we also had      │
│          PyMC (80 MB) loaded, we'd be at 3.29 GB, leaving        │
│          only 210 MB — not enough for LLM's 2 GB peak.           │
│                                                                  │
│  RESOLUTION (ModelRegistry._evict()):                            │
│  1. Calculate required memory for requested model                │
│  2. If current_usage + required > budget (3.5 GB):               │
│     a. Find least-recently-used on-demand model                  │
│     b. Unload it (free memory)                                   │
│     c. Repeat until enough space available                       │
│  3. Load requested model                                         │
│  4. If no models can be evicted (all recently used):             │
│     a. Return 503 Service Unavailable                            │
│     b. Log "MODEL_OOM" alert to Sentry                           │
│     c. Retry after 30 seconds (if async)                         │
│                                                                  │
│  EVICTION PRIORITY (first evicted):                              │
│  1. PyMC (50 MB) — rarely needed, quick to reload                │
│  2. PaddleOCR (150 MB) — moderate reload time                    │
│  3. MMS-ASR (350 MB) — moderate reload time                      │
│  4. Llama 3.2 3B (1.8 GB) — NEVER auto-evicted if actively       │
│     generating; evicted only between requests                    │
│                                                                  │
│  NEVER EVICTED (always loaded):                                  │
│  • AfroBERT NER (280 MB) — core NLP pipeline                     │
│  • fastText LID (30 MB) — tiny, every request needs it           │
│  • multilingual-e5 (400 MB) — RAG pipeline needs it              │
│  • Whoosh BM25 (20 MB) — tiny, RAG pipeline needs it             │
│  • spaCy (80 MB) — stage 1 NER, every request needs it           │
│                                                                  │
└──────────────────────────────────────────────────────────────────┘
```

### SD Card Corruption

```
┌──────────────────────────────────────────────────────────────────┐
│  SCENARIO: microSD card develops bad sectors or becomes          │
│           read-only due to write exhaustion                      │
├──────────────────────────────────────────────────────────────────┤
│                                                                  │
│  PREVENTION:                                                     │
│  • SanDisk High Endurance (100 TBW) — rated for 24/7 writes      │
│  • SQLite WAL reduces write amplification                        │
│  • (weekly) fstrim to maintain flash performance              │
│  • log2ram: keep logs in RAM, flush to disk hourly               │
│  • /tmp and /var/tmp mounted as tmpfs (RAM)                      │
│  • No swap on SD card (swap on USB SSD if needed)                │
│                                                                  │
│  DETECTION:                                                      │
│  • dmesg shows I/O errors, CRC failures                          │
│  • fsck reports bad blocks on boot                               │
│  • Balena health check detects read-only filesystem              │
│  • SMART data (if supported) shows wear level >90%               │
│                                                                  │
│  MITIGATION:                                                     │
│  1. USB SSD backup: nightly rsync of SQLite DB + ChromaDB        │
│  2. If SD becomes read-only:                                     │
│     a. System continues in read-only mode (degraded)             │
│     b. All writes go to USB SSD (failover mount)                 │
│     c. Admin notified (LED + Balena dashboard)                   │
│  3. If SD fully fails:                                           │
│     a. Boot from USB SSD (pre-imaged)                            │
│     b. Restore latest backup from cloud or USB backup            │
│     c. Replace SD card                                           │
│                                                                  │
│  RECOVERY TIME:                                                  │
│  • Failover to USB SSD: ~2 minutes (auto)                        │
│  • Full SD replacement + restore: ~30 minutes (manual)           │
│                                                                  │
│  BACKUP STRATEGY:                                                │
│  ┌────────────────────┬──────────────┬──────────────────┐        │
│  │ What               │ Frequency    │ Destination      │        │
│  ├────────────────────┼──────────────┼──────────────────┤        │
│  │ SQLite DB (.wal)   │ Every 6h     │ USB SSD          │        │
│  │ ChromaDB           │ Every 6h     │ USB SSD          │        │
│  │ Model files        │ On download  │ USB SSD (cache)  │        │
│  │ Config + env       │ On change    │ USB SSD          │        │
│  │ Full backup        │ Daily        │ Cloud (S3)       │        │
│  └────────────────────┴──────────────┴──────────────────┘        │
│                                                                  │
└──────────────────────────────────────────────────────────────────┘
```

### Database Lock Contention

```
┌──────────────────────────────────────────────────────────────────┐
│  SCENARIO: Multiple concurrent requests contend for SQLite lock  │
├──────────────────────────────────────────────────────────────────┤
│                                                                  │
│  CAUSE:                                                          │
│  • SQLite allows one writer at a time                            │
│  • USSD session update + report insert + sync queue write        │
│    may happen simultaneously                                     │
│  • WAL mode allows concurrent reads but serialises writes        │
│                                                                  │
│  MITIGATION:                                                     │
│  1. WAL mode (already enabled): readers don't block writers      │
│  2. busy_timeout = 5000ms: SQLite retries for up to 5 seconds    │
│  3. Write batching: accumulate writes, flush every 500ms         │
│  4. Separate databases:                                          │
│     - reports.db (high write volume)                             │
│     - models.db (low write volume, mostly reads)                 │
│     - sync.db (moderate write volume)                            │
│  5. Connection pooling: max 10 connections, write queue          │
│                                                                  │
│  MONITORING:                                                     │
│  • Track "database is locked" errors via Sentry                  │
│  • Grafana panel: SQLite write latency P50/P95/P99               │
│  • Alert if lock wait time > 1 second average                    │
│                                                                  │
└──────────────────────────────────────────────────────────────────┘
```

### Clock Drift

```
┌──────────────────────────────────────────────────────────────────┐
│  SCENARIO: Pi's system clock drifts, causing timestamp issues    │
├──────────────────────────────────────────────────────────────────┤
│                                                                  │
│  CAUSE: Pi has no RTC (real-time clock) battery                  │
│  • On boot, clock resets to epoch (1970) or last shutdown time   │
│  • NTP sync may fail if internet is unavailable                  │
│                                                                  │
│  MITIGATION:                                                     │
│  1. fake-hwclock: persist last known time to disk on shutdown    │
│  2. systemd-timesyncd: NTP sync on boot + periodic               │
│  3. GPS-based time (if LTE dongle provides it): fallback         │
│  4. Cloud server timestamp: sync_queue uses server_ts from       │
│     cloud ACK, not local clock, for ordering                     │
│  5. CRDT uses logical timestamps (Lamport clocks) for ordering,  │
│     not wall-clock time                                          │
│                                                                  │
│  IMPACT:                                                         │
│  • Reports get correct timestamps from server (cloud) or         │
│    NTP-synced local clock                                        │
│  • Sync ordering uses edge_node_id + sequence number, not        │
│    wall-clock timestamps                                         │
│  • CRDT merge is timestamp-agnostic (uses logical ordering)      │
│                                                                  │
└──────────────────────────────────────────────────────────────────┘
```

---

## Key Architectural Decisions

| # | Decision | Alternatives Considered | Rationale |
|---|----------|----------------------|-----------|
| 1 | **RPi 5 over Jetson Nano** | Jetson Nano, Intel NUC, Cloud-only | RPi 5: 8GB RAM, $80 cost, massive community, aarch64 support. Jetson: GPU overkill for INT4/INT8 models. NUC: too expensive ($300+). |
| 2 | **SQLite WAL over PostgreSQL at edge** | PostgreSQL, DuckDB, Firebird | SQLite: zero-config, single-file backup, WAL mode allows concurrent reads. PostgreSQL: too heavy for Pi (200MB+ RAM just for DB). DuckDB: no WAL, no concurrent access. |
| 3 | **FastAPI over Flask/Django** | Flask, Django, aiohttp | FastAPI: async native, automatic OpenAPI docs, Pydantic validation, type hints. Flask: sync, no built-in validation. Django: too heavy for edge. |
| 4 | **llama.cpp over vLLM/Ollama** | vLLM, Ollama, TensorFlow Lite | llama.cpp: pure C++, minimal dependencies, INT4 quantization, thread-safe, 1.8GB for 3B model. vLLM: requires CUDA. Ollama: additional overhead. TFLite: limited model support. |
| 5 | **ChromaDB over FAISS at edge** | FAISS, Annoy, ScaNN | ChromaDB: embedded, no server, built-in metadata filtering, Python-native. FAISS: C++ library, no metadata support, needs wrapper. Annoy: read-only index. |
| 6 | **Delta sync over full replication** | Full dump, CDC (Debezium), event sourcing | Delta: minimal bandwidth, simple implementation, works over USB. Full dump: too much data. CDC: requires PostgreSQL. Event sourcing: complex for edge. |
| 7 | **Balena over raw systemd** | systemd, Kubernetes (K3s), Docker Swarm | Balena: purpose-built for edge fleets, OTA updates, health monitoring, multi-arch builds. K3s: too heavy for Pi. Docker Swarm: no fleet management. |
| 8 | **Next.js 14 over React SPA** | Vite+React, Nuxt, Angular | Next.js: SSR for SEO, App Router, API routes for proxying, built-in image optimisation, great DX. Vite: no SSR. Nuxt: Vue ecosystem. Angular: larger bundle. |
| 9 | **MapLibre GL over Google Maps** | Google Maps, Leaflet, Deck.gl | MapLibre: open-source, self-hosted tiles, no API key, GL JS rendering, WHO can self-host. Google Maps: requires API key, costs, proprietary. Leaflet: no WebGL. |
| 10 | **Yjs CRDT over custom merge** | Custom LWW, Operational Transform, eventual consistency | Yjs: battle-tested, supports complex data types, Python bindings (y-py), efficient binary format. Custom: error-prone, hard to test. OT: requires central server. |

---

> **Next:** [02 — Three-Door Interface](02-three-door-interface.md)
> **Prev:** [00 — README](README.md)
