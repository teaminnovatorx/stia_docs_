# UDARA AI — Team Overview & Sprint Plan

> **Document ID:** 04-TEAM-PLAN  
> **Version:** 1.0.0  
> **Last Updated:** 2026-05-27  
> **Status:** Active  
> **Classification:** Internal — Team Reference

---

## Table of Contents

1. [Project Overview](#1-project-overview)
2. [Team Composition & Role Definitions](#2-team-composition--role-definitions)
3. [Five-Week Sprint Plan](#3-five-week-sprint-plan)
4. [Gantt-Style Sprint Chart](#4-gantt-style-sprint-chart)
5. [Daily Standup Cadence](#5-daily-standup-cadence)
6. [Git Branching Strategy](#6-git-branching-strategy)
7. [Communication Tools & Workflows](#7-communication-tools--workflows)
8. [RACI Responsibility Matrix](#8-raci-responsibility-matrix)
9. [Risk Register & Mitigation](#9-risk-register--mitigation)
10. [Quality Gates & Definition of Done](#10-quality-gates--definition-of-done)
11. [Appendices](#11-appendices)

---

## 1. Project Overview

### 1.1 What is UDARA AI?

**UDARA AI** is an **Antimicrobial Resistance (AMR) surveillance platform** purpose-built for sub-Saharan Africa. The name "UDARA" derives from the Hausa word for "goodness" — reflecting the platform's mission to deliver better health outcomes through intelligent, accessible disease surveillance.

```
┌─────────────────────────────────────────────────────────────────────┐
│                         UDARA AI PLATFORM                           │
│    Antimicrobial Resistance Surveillance for Sub-Saharan Africa     │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│   ┌──────────┐   ┌──────────────┐   ┌──────────────────────────┐   │
│   │  DOOR 1  │   │   DOOR 2     │   │       DOOR 3             │   │
│   │  USSD    │   │  WhatsApp /  │   │  Web Dashboard           │   │
│   │  Feature │   │  Telegram    │   │  (Health Workers /       │   │
│   │  Phones  │   │  Smartphones │   │   Epidemiologists)       │   │
│   └────┬─────┘   └──────┬───────┘   └───────────┬──────────────┘   │
│        │                │                        │                   │
│   ┌────▼────────────────▼────────────────────────▼────────────┐    │
│   │                    CLOUD LAYER                             │    │
│   │  FastAPI Gateway │ PostgreSQL │ Qdrant │ Neo4j │ Next.js  │    │
│   └──────────────────────────┬──────────────────────────────────┘    │
│                              │ Sync (Delta JSON + CRDT)             │
│   ┌──────────────────────────▼──────────────────────────────────┐    │
│   │                    EDGE LAYER (RPi 5)                        │    │
│   │  Llama 3.2 3B │ AfroBERT │ PaddleOCR │ MMS-ASR │ SQLite   │    │
│   │  Bayesian Resistance Engine │ RAG Pipeline │ Agent System   │    │
│   └─────────────────────────────────────────────────────────────┘    │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

### 1.2 The Problem We Solve

| Dimension | Current State | UDARA AI Solution |
|-----------|--------------|-------------------|
| **Data Collection** | Paper-based, weeks/months of delay | Real-time via USSD, WhatsApp, Telegram, Voice |
| **Language Barriers** | English/French only, misses local context | 12+ African languages via MMS-ASR + AfroBERT NER |
| **Connectivity** | Rural clinics often offline | Edge AI on RPi 5 — works fully offline |
| **Resistance Prediction** | Reactive — only reports confirmed cases | Proactive — Bayesian forecasting with spatial correlation |
| **Accessibility** | Web-only tools exclude 60% of CHWs | Three-Door interface reaches every device type |
| **Drug Quality** | Counterfeit drugs undetected | Snap-Detect OCR validates drug labels instantly |

### 1.3 Architecture Summary

```
                    ┌─────────────────┐
                    │   Cloud (AWS)   │
                    │                 │
    ┌───────────────┼─────────────────┼───────────────┐
    │               │                 │               │
┌───▼───┐    ┌─────▼──────┐   ┌─────▼──────┐  ┌────▼─────┐
│ Next.js│    │  FastAPI   │   │ PostgreSQL │  │  Qdrant  │
│ 14 App │    │  Gateway   │   │ + Timescale│  │  Vector  │
│ PWA    │    │  + Auth    │   │ + PostGIS  │  │   DB     │
└───┬───┘    └─────┬──────┘   └────────────┘  └──────────┘
    │              │
    │       ┌──────▼──────┐
    │       │   Africa's  │
    │       │   Talking   │
    │       │  USSD/SMS   │
    │       └──────┬──────┘
    │              │
┌───▼──────────────▼─────────────────────────────────────────┐
│               EDGE LAYER — Raspberry Pi 5 (8GB)            │
│                                                            │
│  ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌───────────┐    │
│  │ Llama    │ │AfroBERT  │ │PaddleOCR │ │ MMS-ASR   │    │
│  │ 3.2 3B   │ │  NER     │ │  INT8    │ │  INT8     │    │
│  │ INT4     │ │  INT8    │ │          │ │           │    │
│  └──────────┘ └──────────┘ └──────────┘ └───────────┘    │
│                                                            │
│  ┌──────────────┐ ┌────────────┐ ┌───────────────────────┐│
│  │  Bayesian    │ │ChromaDB    │ │  SQLite + WAL Mode   ││
│  │  Resistance  │ │Edge Vector │ │  Local Persistence   ││
│  │  Engine      │ │Store       │ │                       ││
│  └──────────────┘ └────────────┘ └───────────────────────┘│
│                                                            │
│  ┌──────────────────────────────────────────────────────┐  │
│  │  Sync Protocol: Delta JSON + CRDT (Yjs) + gzip      │  │
│  │  + USB Fallback (encrypted tarball)                  │  │
│  └──────────────────────────────────────────────────────┘  │
└────────────────────────────────────────────────────────────┘
```

### 1.4 Key Technical Decisions

| Decision | Choice | Rationale |
|----------|--------|-----------|
| **Edge Hardware** | Raspberry Pi 5 8GB | Cost ($80/unit), community support, GPIO for sensors, sufficient RAM for INT4 LLM |
| **LLM** | Llama 3.2 3B INT4 via llama.cpp | Best quality/size ratio for edge; ~1.7GB in memory; <2s inference |
| **NER** | AfroBERT INT8 | Best multilingual African language coverage; fine-tunable |
| **OCR** | PaddleOCR INT8 | Handles curved labels, multilingual text, low-light photos |
| **ASR** | MMS-ASR INT8 | 1100+ languages including 12 target African languages |
| **Vector DB (Edge)** | ChromaDB | Embedded, lightweight, Python-native, no server process |
| **Vector DB (Cloud)** | Qdrant | Horizontal scaling, gRPC, filtering, hybrid search |
| **Backend** | FastAPI | Async, auto-docs, type hints, WebSocket support |
| **Frontend** | Next.js 14 + shadcn/ui | SSR, App Router, accessible components, PWA support |
| **Maps** | MapLibre GL | Open-source, no API key required, custom styling |
| **Sync** | Delta JSON + Yjs CRDT | Bandwidth-efficient, conflict-free, offline-first |

### 1.5 Target Users

```
┌────────────────────────────────────────────────────────────────────┐
│                        USER PERSONAS                               │
├──────────────┬─────────────────┬───────────┬───────────────────────┤
│    Persona   │  Primary Door   │ Device    │  Key Needs            │
├──────────────┼─────────────────┼───────────┼───────────────────────┤
│ CHW (Rural)  │ Door 1 (USSD)   │ Feature   │ Simple, offline, local │
│              │                  │ Phone     │ language               │
├──────────────┼─────────────────┼───────────┼───────────────────────┤
│ CHW (Urban)  │ Door 2 (WA/TG)  │ Android   │ Photo capture, voice  │
│              │                  │ Smartphone│ reporting, quick ref   │
├──────────────┼─────────────────┼───────────┼───────────────────────┤
│ Clinician    │ Door 2 (WA/TG)  │ iOS/      │ Resistance lookup,     │
│              │ + Door 3 (Web)  │ Android   │ clinical guidance      │
├──────────────┼─────────────────┼───────────┼───────────────────────┤
│ Lab Tech     │ Door 3 (Web)    │ Desktop   │ Data entry, quality    │
│              │                  │ Browser   │ control, reports       │
├──────────────┼─────────────────┼───────────┼───────────────────────┤
│ Epidemiologist│ Door 3 (Web)   │ Desktop   │ Dashboards, maps,      │
│              │                  │ Browser   │ outbreak simulation    │
├──────────────┼─────────────────┼───────────┼───────────────────────┤
│ MOH Official │ Door 3 (Web)    │ Desktop   │ Policy dashboards,     │
│              │                  │ Browser   │ AMR trend reports      │
├──────────────┼─────────────────┼───────────┼───────────────────────┤
│ Pharmacy     │ Door 2 (WA/TG)  │ Android   │ Drug validation,       │
│ Staff        │                  │ Smartphone│ counterfeit detection  │
└──────────────┴─────────────────┴───────────┴───────────────────────┘
```

---

## 2. Team Composition & Role Definitions

### 2.1 Team Structure Overview

```
                    ┌─────────────────────┐
                    │   Project Lead      │
                    │   (Rotating weekly) │
                    └──────────┬──────────┘
                               │
         ┌─────────────────────┼─────────────────────┐
         │                     │                     │
    ┌────▼─────┐         ┌────▼─────┐         ┌────▼─────┐
    │ Member 01│         │ Member 02│         │ Member 03│
    │ ML/AI    │◄───────►│ Backend  │◄───────►│ Frontend │
    │ Engineer │         │ + Edge   │         │ Engineer │
    └──────────┘         └────┬─────┘         └────┬─────┘
                               │                     │
                          ┌────▼─────────────────────▼────┐
                          │        Member 04              │
                          │   Integration + Bot Engineer   │
                          └───────────────────────────────┘
```

### 2.2 Member 01 — ML/AI Engineer

**Role:** Senior ML/AI Engineer  
**Focus:** All AI/ML models, RAG pipeline, agent ingestion system, Bayesian resistance engine  
**Reports to:** Project Lead (rotating)

```
┌─────────────────────────────────────────────────────────────────────┐
│                    MEMBER 01 — ML/AI ENGINEER                       │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  PRIMARY OWNERSHIP:                                                 │
│                                                                     │
│  ┌────────────────────────────────────────────────────────────┐    │
│  │  AI/ML MODELS (Edge-Optimized)                             │    │
│  │                                                            │    │
│  │  • Llama 3.2 3B INT4 via llama.cpp                        │    │
│  │    → AMR clinical guidance generation                     │    │
│  │    → ~1.7GB memory, <2s inference on RPi 5                │    │
│  │                                                            │    │
│  │  • AfroBERT-base INT8 for Clinical NER                    │    │
│  │    → Entities: drug_name, bacteria, resistance_pattern,    │    │
│  │      dosage, patient_age, specimen                        │    │
│  │    → Fine-tuned on AMR clinical corpus                    │    │
│  │                                                            │    │
│  │  • PaddleOCR INT8 for Drug Label Recognition              │    │
│  │    → Drug name, batch number, expiry date, dosage         │    │
│  │    → Handles curved labels, low-light, multilingual       │    │
│  │                                                            │    │
│  │  • MMS-ASR INT8 for Voice Reporting                       │    │
│  │    → 1100+ languages, 12 target African languages         │    │
│  │    → Streaming transcription with confidence scoring      │    │
│  └────────────────────────────────────────────────────────────┘    │
│                                                                     │
│  ┌────────────────────────────────────────────────────────────┐    │
│  │  BAYESIAN RESISTANCE ENGINE                                 │    │
│  │                                                            │    │
│  │  • PyMC Beta(2,2) priors per drug/district                │    │
│  │  • Likelihood from observed resistance counts              │    │
│  │  • Spatial correlation via PySAL (spatial autocorrelation) │    │
│  │  • Posterior sampling → P(resistance) per drug/pathogen    │    │
│  │    per district                                            │    │
│  └────────────────────────────────────────────────────────────┘    │
│                                                                     │
│  ┌────────────────────────────────────────────────────────────┐    │
│  │  RAG PIPELINE                                              │    │
│  │                                                            │    │
│  │  • Hybrid retrieval: Dense (multilingual-e5) + BM25        │    │
│  │  • ChromaDB at edge / Qdrant at cloud                     │    │
│  │  • Reciprocal Rank Fusion for ranking                     │    │
│  │  • Context injection into Llama for guidance               │    │
│  │  • Document sources: WHO AMR guidelines, national AMR     │    │
│  │    action plans, published AMR research papers             │    │
│  └────────────────────────────────────────────────────────────┘    │
│                                                                     │
│  ┌────────────────────────────────────────────────────────────┐    │
│  │  AGENT INGESTION SYSTEM                                    │    │
│  │                                                            │    │
│  │  • Raw input (text/image/voice) → Structured AMR case     │    │
│  │  • Ingestion router: classify modality, dispatch          │    │
│  │  • Case builder: extract entities, validate, persist      │    │
│  │  • Guidance generator: resistance check + evidence-based   │    │
│  │    treatment recommendation                               │    │
│  └────────────────────────────────────────────────────────────┘    │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

#### Key Skills Required

| Skill | Proficiency | Why Needed |
|-------|-------------|------------|
| **PyTorch / llama.cpp** | Expert | Model quantization, INT4/INT8 optimization for edge |
| **Hugging Face Transformers** | Expert | AfroBERT fine-tuning, inference pipelines |
| **PyMC / Bayesian Statistics** | Advanced | Resistance probability estimation |
| **PySAL / Spatial Analysis** | Intermediate | Spatial correlation of resistance patterns |
| **Vector Databases** | Advanced | ChromaDB, Qdrant, hybrid retrieval |
| **NLP / NER** | Expert | Clinical entity extraction, multilingual NER |
| **Python** | Expert | Primary development language for ML stack |
| **Docker** | Intermediate | Model containerization for edge deployment |

#### Performance Targets

| Metric | Target | Measurement Method |
|--------|--------|-------------------|
| Llama 3.2 3B inference latency | < 2.0s (first token) | llama.cpp benchmark on RPi 5 |
| AfroBERT NER F1 score | > 0.85 on AMR corpus | CoNLL-style evaluation |
| PaddleOCR accuracy | > 0.90 on drug labels | Custom eval dataset (500 labels) |
| MMS-ASR WER | < 0.25 (12 target langs) | Common Voice + custom AMR corpus |
| RAG retrieval recall@10 | > 0.80 | AMR guideline test queries |
| Bayesian prediction MAE | < 0.05 | Cross-validated on historical data |

#### Week-by-Week Ownership

| Week | Primary Deliverables |
|------|---------------------|
| Week 1 | Docker environment for ML, llama.cpp build, AfroBERT baseline |
| Week 2 | Model quantization (INT4/INT8), first inference benchmarks on RPi 5 |
| Week 3 | MMS-ASR integration, voice pipeline, NER on real CHW inputs |
| Week 4 | RAG pipeline (ingestion → retrieval → generation), Bayesian engine v1 |
| Week 5 | RAG with resistance maps, model eval benchmarks, pilot validation |

---

### 2.3 Member 02 — Backend + Edge Engineer

**Role:** Senior Backend & Edge Engineer  
**Focus:** FastAPI gateway, USSD session management, Edge runtime on RPi 5, sync protocol, database schemas  
**Reports to:** Project Lead (rotating)

```
┌─────────────────────────────────────────────────────────────────────┐
│                 MEMBER 02 — BACKEND + EDGE ENGINEER                 │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  PRIMARY OWNERSHIP:                                                 │
│                                                                     │
│  ┌────────────────────────────────────────────────────────────┐    │
│  │  FASTAPI GATEWAY                                           │    │
│  │                                                            │    │
│  │  • REST API for case CRUD, resistance queries, reports     │    │
│  │  • WebSocket support for real-time notifications           │    │
│  │  • JWT authentication + Keycloak SSO integration           │    │
│  │  • Rate limiting, request logging, CORS middleware         │    │
│  │  • OpenAPI 3.1 auto-generated documentation                │    │
│  │  • Dependency injection for DB sessions, auth, services    │    │
│  └────────────────────────────────────────────────────────────┘    │
│                                                                     │
│  ┌────────────────────────────────────────────────────────────┐    │
│  │  EDGE RUNTIME (RPi 5)                                      │    │
│  │                                                            │    │
│  │  • Model lifecycle management (load/unload/swap)           │    │
│  │  • Memory monitoring — graceful degradation strategy       │    │
│  │  • SQLite WAL mode for concurrent read/write              │    │
│  │  • Hardware monitoring (CPU temp, RAM, storage, network)  │    │
│  │  • Balena Fleet integration for OTA updates                │    │
│  └────────────────────────────────────────────────────────────┘    │
│                                                                     │
│  ┌────────────────────────────────────────────────────────────┐    │
│  │  SYNC PROTOCOL                                             │    │
│  │                                                            │    │
│  │  • Delta JSON generation (local SQLite vs last sync point) │    │
│  │  • CRDT conflict resolution via Yjs                       │    │
│  │  • gzip compression (level 6) for bandwidth efficiency     │    │
│  │  • Retry with exponential backoff (1s, 2s, 4s, 8s, 16s)   │    │
│  │  • USB fallback: encrypted tarball for physical transport  │    │
│  │  • State machine: IDLE → GENERATING → COMPRESSING →       │    │
│  │    UPLOADING → CONFIRMING → IDLE                          │    │
│  └────────────────────────────────────────────────────────────┘    │
│                                                                     │
│  ┌────────────────────────────────────────────────────────────┐    │
│  │  DATABASES                                                 │    │
│  │                                                            │    │
│  │  EDGE:  SQLite (4 tables: cases, resistance_data,          │    │
│  │         sync_meta, local_users)                            │    │
│  │  CLOUD: PostgreSQL + TimescaleDB + PostGIS + Neo4j          │    │
│  │         (12+ tables, spatial queries, time-series, KG)     │    │
│  └────────────────────────────────────────────────────────────┘    │
│                                                                     │
│  ┌────────────────────────────────────────────────────────────┐    │
│  │  USSD SESSION MANAGER                                      │    │
│  │                                                            │    │
│  │  • Redis-backed sessions with 120s TTL                    │    │
│  │  • State machine for multi-step menu navigation            │    │
│  │  • Input validation and sanitization for feature phones    │    │
│  │  • Africa's Talking API integration (USSD + SMS)           │    │
│  │  • Session timeout handling and recovery flows             │    │
│  └────────────────────────────────────────────────────────────┘    │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

#### Key Skills Required

| Skill | Proficiency | Why Needed |
|-------|-------------|------------|
| **FastAPI** | Expert | Primary backend framework |
| **PostgreSQL / TimescaleDB / PostGIS** | Expert | Cloud database with spatial and time-series capabilities |
| **Neo4j / Cypher** | Intermediate | Knowledge graph for drug-pathogen relationships |
| **Redis** | Advanced | USSD session management, caching |
| **SQLite** | Expert | Edge persistence with WAL mode |
| **Docker / Docker Compose** | Expert | Container orchestration for edge and cloud |
| **CRDT / Yjs** | Intermediate | Conflict-free data synchronization |
| **Python** | Expert | Primary development language |
| **Raspberry Pi / Linux** | Advanced | Edge hardware management |

#### Week-by-Week Ownership

| Week | Primary Deliverables |
|------|---------------------|
| Week 1 | Docker Compose for all services, repo structure, API scaffold |
| Week 2 | Edge container bootstrap, SQLite schema, first sync prototype |
| Week 3 | USSD session manager, Africa's Talking integration, menu flows |
| Week 4 | Full sync protocol (CRDT, retry, USB fallback), DB schemas |
| Week 5 | E2E testing, edge deployment to 3 RPi units, monitoring |

---

### 2.4 Member 03 — Frontend Engineer

**Role:** Frontend Engineer  
**Focus:** Next.js 14 web application, MapLibre GL resistance maps, ECharts dashboards, PWA for Door 2, design system  
**Reports to:** Project Lead (rotating)

```
┌─────────────────────────────────────────────────────────────────────┐
│                   MEMBER 03 — FRONTEND ENGINEER                     │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  PRIMARY OWNERSHIP:                                                 │
│                                                                     │
│  ┌────────────────────────────────────────────────────────────┐    │
│  │  NEXT.JS 14 APPLICATION                                    │    │
│  │                                                            │    │
│  │  • App Router with server components                      │    │
│  │  • shadcn/ui component library                            │    │
│  │  • Tailwind CSS 4 for styling                             │    │
│  │  • Internationalization (i18n) for 12+ languages           │    │
│  │  • Responsive design (mobile-first for Door 2 PWA)        │    │
│  │  • Accessibility (WCAG 2.1 AA compliance)                 │    │
│  └────────────────────────────────────────────────────────────┘    │
│                                                                     │
│  ┌────────────────────────────────────────────────────────────┐    │
│  │  MAPLIBRE GL RESISTANCE MAPS                              │    │
│  │                                                            │    │
│  │  • Choropleth layers: resistance % by district            │    │
│  │  • Spatial interpolation (IDW) for unsampled areas        │    │
│  │  • Heatmap overlay for outbreak visualization             │    │
│  │  • Time-series animation (play/pause) for trend viewing   │    │
│  │  • Custom vector tiles from PostGIS                      │    │
│  │  • Offline map tile caching via service worker            │    │
│  └────────────────────────────────────────────────────────────┘    │
│                                                                     │
│  ┌────────────────────────────────────────────────────────────┐    │
│  │  ECHARTS DASHBOARDS                                        │    │
│  │                                                            │    │
│  │  • AMR trend charts (line, area) by drug/pathogen         │    │
│  │  • Resistance distribution (box plots, violin plots)       │    │
│  │  • Case reporting volume (bar charts, sparklines)         │    │
│  │  • District comparison (radar, grouped bar)               │    │
│  │  • Outbreak simulation visualization                     │    │
│  │  • Export to PNG/PDF for offline reporting                │    │
│  └────────────────────────────────────────────────────────────┘    │
│                                                                     │
│  ┌────────────────────────────────────────────────────────────┐    │
│  │  PWA (DOOR 2 — WEB APP)                                   │    │
│  │                                                            │    │
│  │  • Service worker for offline access                      │    │
│  │  • Background sync for deferred case submission           │    │
│  │  • Push notifications for outbreak alerts                 │    │
│  │  • App manifest with icons and theme                     │    │
│  │  • IndexedDB for local data caching                       │    │
│  └────────────────────────────────────────────────────────────┘    │
│                                                                     │
│  ┌────────────────────────────────────────────────────────────┐    │
│  │  DESIGN SYSTEM                                             │    │
│  │                                                            │    │
│  │  • shadcn/ui theme customization (dark/light)             │    │
│  │  • Design tokens for consistent spacing, color, typography │    │
│  │  • Component storybook for development reference          │    │
│  │  • Low-bandwidth mode (reduce images, defer loading)      │    │
│  └────────────────────────────────────────────────────────────┘    │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

#### Key Skills Required

| Skill | Proficiency | Why Needed |
|-------|-------------|------------|
| **Next.js 14** | Expert | App Router, SSR, server components |
| **React / TypeScript** | Expert | UI development |
| **MapLibre GL JS** | Advanced | Resistance choropleth maps |
| **ECharts** | Advanced | Data visualization dashboards |
| **Tailwind CSS 4** | Expert | Utility-first styling |
| **shadcn/ui** | Advanced | Accessible component library |
| **PWA / Service Workers** | Advanced | Offline-first Door 2 experience |
| **i18n (next-intl)** | Intermediate | Multilingual support |

#### Week-by-Week Ownership

| Week | Primary Deliverables |
|------|---------------------|
| Week 1 | Next.js project setup, shadcn/ui integration, design tokens |
| Week 2 | Dashboard layout, ECharts baseline charts, API integration |
| Week 3 | MapLibre resistance map skeleton, PWA service worker setup |
| Week 4 | Resistance maps with real data, ECharts dashboards complete |
| Week 5 | Polish, accessibility audit, pilot site deployment, i18n |

---

### 2.5 Member 04 — Integration + Bot Engineer

**Role:** Integration & Bot Engineer  
**Focus:** aiogram 3.x Telegram bot, WhatsApp Cloud API, unified Bot Core, CHW onboarding flows, gamification, E2E testing  
**Reports to:** Project Lead (rotating)

```
┌─────────────────────────────────────────────────────────────────────┐
│              MEMBER 04 — INTEGRATION + BOT ENGINEER                 │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  PRIMARY OWNERSHIP:                                                 │
│                                                                     │
│  ┌────────────────────────────────────────────────────────────┐    │
│  │  UNIFIED BOT CORE                                          │    │
│  │                                                            │    │
│  │  • Channel-agnostic message handling                      │    │
│  │  • Adapter pattern: WhatsApp, Telegram, SMS                │    │
│  │  • Conversation state machine                             │    │
│  │  • Media handling (photos, voice, documents)              │    │
│  │  • Quick reply buttons, inline keyboards                   │    │
│  └────────────────────────────────────────────────────────────┘    │
│                                                                     │
│  ┌────────────────────────────────────────────────────────────┐    │
│  │  TELEGRAM BOT (aiogram 3.x)                                │    │
│  │                                                            │    │
│  │  • aiogram 3.x async framework                           │    │
│  │  • Command handlers: /start, /report, /check, /guide      │    │
│  │  • Callback query handling for inline keyboards            │    │
│  │  • Photo handler → OCR pipeline integration               │    │
│  │  • Voice handler → ASR pipeline integration               │    │
│  │  • BotFather sandbox for development & testing            │    │
│  └────────────────────────────────────────────────────────────┘    │
│                                                                     │
│  ┌────────────────────────────────────────────────────────────┐    │
│  │  WHATSAPP CLOUD API                                        │    │
│  │                                                            │    │
│  │  • WhatsApp Business Cloud API integration                │    │
│  │  • Webhook handler for incoming messages                  │    │
│  │  • Template messages for notifications                    │    │
│  │  • Interactive messages (list, button, reply)             │    │
│  │  • Media message handling (photos, voice notes)            │    │
│  │  • Rate limit management (WhatsApp API quotas)            │    │
│  └────────────────────────────────────────────────────────────┘    │
│                                                                     │
│  ┌────────────────────────────────────────────────────────────┐    │
│  │  CHW ONBOARDING FLOWS                                      │    │
│  │                                                            │    │
│  │  • Registration wizard (name, location, facility, role)    │    │
│  │  • Training modules via conversational UI                 │    │
│  │  • Practice case reporting with feedback                  │    │
│  │  • Certification quiz (AMR knowledge)                      │    │
│  │  • Facility assignment and access provisioning            │    │
│  └────────────────────────────────────────────────────────────┘    │
│                                                                     │
│  ┌────────────────────────────────────────────────────────────┐    │
│  │  GAMIFICATION SYSTEM                                       │    │
│  │                                                            │    │
│  │  • Points system: report case (+10), validate drug (+5)   │    │
│  │  • Badges: First Report, 10 Cases, 50 Cases, Zero Error   │    │
│  │  • Leaderboard: district-level and national rankings      │    │
│  │  • Rewards: airtime credits, certificates, recognition    │    │
│  │  • Streak bonuses for consistent reporting                │    │
│  └────────────────────────────────────────────────────────────┘    │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

#### Key Skills Required

| Skill | Proficiency | Why Needed |
|-------|-------------|------------|
| **aiogram 3.x** | Expert | Async Telegram bot framework |
| **WhatsApp Cloud API** | Advanced | Business messaging platform |
| **Python** | Expert | Bot development language |
| **Async/Await** | Expert | Concurrent bot handling |
| **Redis** | Intermediate | Conversation state persistence |
| **Testing (pytest-asyncio)** | Advanced | E2E bot testing |
| **BotFather** | Intermediate | Bot configuration and management |
| **Webhook Management** | Advanced | Cloud API webhook handling |

#### Week-by-Week Ownership

| Week | Primary Deliverables |
|------|---------------------|
| Week 1 | Bot project setup, aiogram 3.x skeleton, BotFather config |
| Week 2 | Telegram /report and /check commands, media handlers |
| Week 3 | WhatsApp Cloud API integration, unified Bot Core, dual messenger |
| Week 4 | CHW onboarding flows, gamification system (points + badges) |
| Week 5 | E2E testing, leaderboard, pilot deployment, documentation |

---

## 3. Five-Week Sprint Plan

### 3.1 Sprint Overview

| Aspect | Details |
|--------|---------|
| **Sprint Duration** | 5 weeks (35 working days) |
| **Working Hours** | 09:00 — 18:00 UTC (adjusted for team time zones) |
| **Sprint Goal** | Deploy MVP to 3 pilot sites with all Three Doors functional |
| **Velocity Target** | ~80 story points total (~20 points per engineer) |
| **Definition of Done** | Code reviewed, tested (unit + integration), documented, deployed |

### 3.2 Week 1: Foundation

**Theme:** "Get everything talking to everything"  
**Goal:** Development environment, repo structure, basic scaffolds, edge container bootstrap

```
WEEK 1 — FOUNDATION
═══════════════════════════════════════════════════════════════

Day 1 (Mon): Project Bootstrap
├── Create monorepo structure
├── Initialize Git (main, develop branches)
├── Set up Docker Compose for cloud services
│   ├── PostgreSQL 16 + TimescaleDB + PostGIS
│   ├── Redis 7
│   ├── Qdrant
│   ├── Neo4j 5
│   └── FastAPI dev server
├── Set up Notion workspace + Slack channels
└── Team kickoff meeting

Day 2 (Tue): Backend Scaffold
├── FastAPI project structure (see Member 02 doc)
├── Basic CRUD endpoints (cases, health)
├── JWT auth middleware (Keycloak integration start)
├── Alembic migrations setup
└── API documentation (OpenAPI auto-gen)

Day 3 (Wed): Frontend Scaffold
├── Next.js 14 project with App Router
├── shadcn/ui component installation
├── Design token configuration
├── API client setup (React Query / SWR)
└── Basic layout: sidebar, header, dashboard placeholder

Day 4 (Thu): ML Environment
├── Docker image for ML inference (llama.cpp, transformers)
├── llama.cpp build with RPi 5 ARM64 cross-compilation
├── AfroBERT baseline inference test
├── PaddleOCR + MMS-ASR baseline tests
└── ChromaDB setup

Day 5 (Fri): Edge Bootstrap
├── Balena CLI setup
├── Edge Docker image (slim Debian + Python 3.11)
├── RPi 5 flashing with Balena OS
├── SQLite schema (4 tables)
├── First model load test on physical RPi
└── Week 1 demo + retrospective
```

**Week 1 Deliverables:**

| Deliverable | Owner | Acceptance Criteria |
|-------------|-------|-------------------|
| Monorepo with Docker Compose | Member 02 | `docker compose up` starts all cloud services |
| FastAPI scaffold with 3 endpoints | Member 02 | GET /health, POST /cases, GET /cases return 200 |
| Next.js scaffold with design tokens | Member 03 | App renders with sidebar + header |
| ML Docker image | Member 01 | llama.cpp + PaddleOCR + MMS-ASR run in container |
| Edge container on RPi 5 | Member 02 | SQLite reads/writes, one model loads |

### 3.3 Week 2: Core ML + Backend

**Theme:** "Make the edge intelligent"  
**Goal:** Quantized models running on RPi 5, first real API endpoints, SQLite at edge, basic sync

```
WEEK 2 — CORE ML + BACKEND
═══════════════════════════════════════════════════════════════

Day 6 (Mon): Model Quantization
├── Llama 3.2 3B → INT4 via llama.cpp (4.5GB → 1.7GB)
├── AfroBERT → INT8 via ONNX Runtime (670MB → 340MB)
├── PaddleOCR → INT8 via PaddleSlim
├── MMS-ASR → INT8 via CTranslate2
└── Memory footprint validation on RPi 5

Day 7 (Tue): Edge Runtime
├── Model loading strategy (lazy load, LRU cache)
├── Memory monitoring daemon
├── Graceful degradation (drop OCR if RAM > 6.5GB)
├── SQLite WAL mode enablement
└── Local API server on RPi (FastAPI + Uvicorn)

Day 8 (Wed): API Endpoints
├── POST /api/v1/cases — create AMR case
├── GET /api/v1/cases/{id} — retrieve case
├── GET /api/v1/resistance — query resistance probability
├── POST /api/v1/ocr — drug label OCR
└── POST /api/v1/transcribe — voice transcription

Day 9 (Thu): Sync Protocol v1
├── Delta JSON generation from SQLite
├── Basic upload to cloud endpoint
├── Compression (gzip level 6)
├── Sync metadata tracking (last_sync, sync_id)
└── Retry logic (exponential backoff)

Day 10 (Fri): Frontend Integration
├── Case submission form (Door 3)
├── Resistance query interface
├── ECharts baseline charts (mock data)
├── API integration with React Query
└── Week 2 demo + retrospective
```

**Week 2 Deliverables:**

| Deliverable | Owner | Acceptance Criteria |
|-------------|-------|-------------------|
| All 4 models quantized for RPi 5 | Member 01 | Total RAM < 6GB, each model functional |
| Edge runtime on RPi 5 | Member 02 | Models load/unload, memory monitoring works |
| 5 API endpoints functional | Member 02 | All pass integration tests |
| Sync protocol v1 | Member 02 | Delta generated, compressed, uploaded |
| Case form + resistance query UI | Member 03 | Form submits, chart renders with mock data |

### 3.4 Week 3: Interfaces

**Theme:** "Open all three doors"  
**Goal:** USSD flows working, Telegram bot functional, web map skeleton, voice ASR integration

```
WEEK 3 — INTERFACES
═══════════════════════════════════════════════════════════════

Day 11 (Mon): USSD Development
├── Africa's Talking sandbox setup
├── Redis session manager
├── USSD menu tree design (report case, check resistance, help)
├── State machine implementation
└── Africa's Talking callback endpoint

Day 12 (Tue): USSD Flows
├── Report Case flow (6-step wizard)
│   1. Select pathogen category
│   2. Enter patient age group
│   3. Select specimen type
│   4. Enter drug(s) prescribed
│   5. Enter observed outcome
│   6. Confirm & submit
├── Check Resistance flow (search by drug/district)
├── Input validation (phone keypad constraints)
└── Timeout handling (120s session TTL)

Day 13 (Wed): Telegram Bot
├── aiogram 3.x project setup
├── /start onboarding message
├── /report command → case reporting conversation
├── /check command → resistance lookup
├── Photo handler → PaddleOCR integration
└── Voice handler → MMS-ASR integration

Day 14 (Thu): Web Map Skeleton
├── MapLibre GL initialization
├── GeoJSON layer for district boundaries
├── Mock choropleth layer (resistance %)
├── Popup on click (district details)
├── Legend component
└── Map controls (zoom, filter by drug)

Day 15 (Fri): Integration & Testing
├── End-to-end: USSD → API → SQLite → Sync → Cloud
├── End-to-end: Telegram → Photo → OCR → Case → DB
├── Voice ASR pipeline test (Swahili + English)
├── Member 04: WhatsApp Cloud API sandbox setup
└── Week 3 demo + retrospective
```

**Week 3 Deliverables:**

| Deliverable | Owner | Acceptance Criteria |
|-------------|-------|-------------------|
| USSD report case flow | Member 02 | Complete 6-step wizard via Africa's Talking sandbox |
| Telegram /report command | Member 04 | Sends case via conversation, stores in DB |
| Photo → OCR pipeline | Member 01 + 04 | Photo in Telegram → extracted drug info |
| Voice → ASR pipeline | Member 01 + 04 | Voice note → transcription in target language |
| MapLibre skeleton | Member 03 | District boundaries render, mock choropleth |

### 3.5 Week 4: Intelligence Layer

**Theme:** "Add the brain"  
**Goal:** RAG pipeline operational, resistance maps with real data, outbreak sim v1, dual messenger

```
WEEK 4 — INTELLIGENCE LAYER
═══════════════════════════════════════════════════════════════

Day 16 (Mon): RAG Pipeline
├── Document ingestion (WHO AMR guidelines PDF)
├── Semantic chunking (512 tokens, 50 overlap)
├── Hybrid embedding (multilingual-e5 dense + BM25 sparse)
├── ChromaDB indexing at edge
├── Qdrant indexing at cloud
└── Retrieval evaluation on 50 test queries

Day 17 (Tue): RAG + LLM Integration
├── Retriever with reciprocal rank fusion
├── Context injection into Llama 3.2 prompt
├── AMR guidance generation from retrieved context
├── Prompt template design for clinical guidance
├── Guidance quality evaluation (expert review)
└── RAG endpoint: POST /api/v1/guidance

Day 18 (Wed): Resistance Maps
├── Bayesian engine v1 (PyMC Beta priors)
├── Resistance data aggregation by district
├── MapLibre choropleth with real data
├── Spatial interpolation (IDW for unsampled areas)
├── Color scale: green (0-25%) → yellow → orange → red (>75%)
└── Time slider for temporal filtering

Day 19 (Thu): Dual Messenger + Outbreak Sim
├── WhatsApp Cloud API webhook handler
├── Unified Bot Core (shared message handling)
├── Country-configurable messenger (env var)
├── Outbreak simulator v1 (SIR model)
│   ├── Basic SIR parameters (beta, gamma, population)
│   ├── Agent-based spatial spread
│   └── Web visualization
├── Gamification: points + badges system
└── ECharts dashboard integration

Day 20 (Fri): Integration Sprint
├── RAG-guided Telegram responses
├── Resistance maps with Bayesian data
├── Dual messenger (switch between WA/TG per country)
├── End-to-end: voice report → NER → case → RAG → guidance
├── Performance benchmarks (latency, memory)
└── Week 4 demo + retrospective
```

**Week 4 Deliverables:**

| Deliverable | Owner | Acceptance Criteria |
|-------------|-------|-------------------|
| RAG pipeline end-to-end | Member 01 | Ingest → retrieve → generate guidance in < 5s |
| Bayesian resistance engine | Member 01 | P(resistance) output per drug/pathogen/district |
| Resistance maps with real data | Member 03 | Choropleth renders from Bayesian output |
| Dual messenger (WA + TG) | Member 04 | Same functionality on both platforms |
| Outbreak sim v1 | Member 01 | SIR simulation runs, results visualized |

### 3.6 Week 5: Polish + Pilot Prep

**Theme:** "Ship it"  
**Goal:** End-to-end testing, gamification, pilot deployment to 3 sites, documentation

```
WEEK 5 — POLISH + PILOT PREP
═══════════════════════════════════════════════════════════════

Day 21 (Mon): E2E Testing
├── Full E2E test suite (pytest + Playwright)
│   ├── Door 1: USSD → API → DB → Sync → Cloud
│   ├── Door 2: Telegram → Voice → ASR → NER → Case → DB
│   ├── Door 2: WhatsApp → Photo → OCR → Case → DB
│   ├── Door 3: Web → Case form → Map → Dashboard
│   └── Edge: Offline case → sync on reconnect
├── Load testing (Locust): 100 concurrent users
├── Security audit: OWASP Top 10 check
└── Bug triage and fixes

Day 22 (Tue): Gamification + Polish
├── Points system live (report +10, validate +5)
├── Badges system (5 badges: First, 10, 50, 100, Zero Error)
├── Leaderboard (district + national)
├── UI polish: loading states, error states, empty states
├── Accessibility audit (WCAG 2.1 AA)
└── Mobile responsive fixes

Day 23 (Wed): Pilot Deployment Prep
├── 3 RPi 5 units: flash Balena OS, install edge container
├── Configuration per site (district, facility, language)
├── Network testing (offline → online sync)
├── USB sync fallback test (encrypted tarball)
├── Monitoring setup (Balena dashboard + Grafana)
└── Deployment runbook creation

Day 24 (Thu): Documentation & Training
├── API documentation review (OpenAPI)
├── User guides (CHW quick-start, USSD guide, bot guide)
├── Operations runbook (deployment, monitoring, incident response)
├── Training materials for pilot site staff
├── Video walkthroughs (Loom)
└── Code documentation (docstrings, README)

Day 25 (Fri): Pilot Launch
├── Deploy to Site 1 (urban clinic — Kampala)
├── Deploy to Site 2 (peri-urban — Accra)
├── Deploy to Site 3 (rural — Kigali outskirts)
├── Go-live verification checklist
├── Sprint retrospective + lessons learned
└── Sprint review presentation to stakeholders
```

**Week 5 Deliverables:**

| Deliverable | Owner | Acceptance Criteria |
|-------------|-------|-------------------|
| E2E test suite (15+ tests) | All | All tests pass, CI/CD green |
| Gamification system | Member 04 | Points, badges, leaderboard functional |
| Pilot deployment (3 sites) | Member 02 | All 3 RPi units deployed and reporting |
| Documentation package | All | API docs, user guides, ops runbook complete |
| Sprint review | All | Stakeholder sign-off on MVP |

---

## 4. Gantt-Style Sprint Chart

```
UDARA AI — 5-WEEK SPRINT GANTT CHART
═══════════════════════════════════════════════════════════════════════════

MEMBER              W1          W2          W3          W4          W5
                   D1-D5       D6-D10      D11-D15     D16-D20     D21-D25
                   ██████      ██████      ██████      ██████      ██████

MEMBER 01
  ML Environment   [████]
  Model Quant.                  [████]
  NER/ASR Integ.                            [████████]
  RAG Pipeline                                           [████████]
  Bayesian Eng.                                          [████████]
  Model Eval.                                                        [████]
  Pilot Deploy                                                       [████]

MEMBER 02
  Docker/Repo      [████]
  API Scaffold     [████]
  Edge Runtime                  [████]
  Sync Protocol v1              [████]
  USSD Sessions                             [████]
  USSD Flows                                [████]
  Sync Protocol v2                                      [████]
  DB Schemas                                            [████]
  E2E Testing                                                     [████]
  Pilot Deploy                                                     [████]

MEMBER 03
  Next.js Setup     [████]
  Dashboard v1                  [████]
  MapLibre Skeleton                           [████]
  Resistance Maps                                          [████]
  ECharts Dashboards                                        [████████]
  Polish/Access.                                                        [████]
  Pilot Deploy                                                          [████]

MEMBER 04
  Bot Setup         [████]
  TG /report                      [████]
  TG Photo/Voice                               [████]
  WA Cloud API                                         [████]
  Bot Core Unif.                                       [████]
  CHW Onboarding                                        [████████]
  Gamification                                          [████████]
  E2E Testing                                                     [████]
  Pilot Deploy                                                     [████]

═══════════════════════════════════════════════════════════════════════════

KEY MILESTONES:
  ◆ D5  (Fri W1): Environment ready, all scaffolds running
  ◆ D10 (Fri W2): Edge AI working, sync v1 functional
  ◆ D15 (Fri W3): All 3 Doors open (USSD, TG, Web)
  ◆ D20 (Fri W4): Intelligence layer live (RAG, Bayesian, Maps)
  ◆ D25 (Fri W5): Pilot launch at 3 sites

DEPENDENCIES (critical path):
  W1 → W2: Docker/Repo → Model Quantization + API endpoints
  W2 → W3: Edge Runtime → USSD + Telegram media handlers
  W3 → W4: Interfaces → RAG + Bayesian + Dual Messenger
  W4 → W5: Intelligence → E2E testing + Pilot deploy
```

---

## 5. Daily Standup Cadence

### 5.1 Schedule

| Aspect | Details |
|--------|---------|
| **Frequency** | Daily, Monday — Friday |
| **Time** | 09:15 — 09:30 UTC (15 minutes max) |
| **Format** | Video call (Google Meet / Zoom) |
| **Facilitator** | Rotating daily (alphabetical order) |
| **Notes** | Recorded in Notion standup page |

### 5.2 Standup Format

Each member answers 3 questions in < 2 minutes:

```
┌──────────────────────────────────────────────────────────┐
│                   DAILY STANDUP TEMPLATE                  │
├──────────────────────────────────────────────────────────┤
│                                                          │
│  Member: ____________                                    │
│  Date: ____________                                      │
│                                                          │
│  1. WHAT I DID YESTERDAY:                                │
│     • Completed [task]                                   │
│     • Started [task]                                     │
│     • Reviewed [PR #]                                    │
│                                                          │
│  2. WHAT I'M DOING TODAY:                                │
│     • Continue [task]                                    │
│     • Start [task]                                       │
│     • Pair with [member] on [topic]                      │
│                                                          │
│  3. BLOCKERS / HELP NEEDED:                              │
│     • Blocked by [dependency]                            │
│     • Need [member]'s input on [topic]                   │
│     • None                                              │
│                                                          │
└──────────────────────────────────────────────────────────┘
```

### 5.3 Weekly Ceremonies

| Ceremony | Day | Time | Duration | Participants |
|----------|-----|------|----------|--------------|
| Daily Standup | Mon–Fri | 09:15 UTC | 15 min | All 4 members |
| Sprint Planning | Monday | 10:00 UTC | 60 min | All 4 members |
| Pair Programming | Tue & Thu | 14:00 UTC | 90 min | 2 members rotate |
| Demo Day | Friday | 15:00 UTC | 30 min | All + stakeholders |
| Retrospective | Friday | 16:00 UTC | 30 min | All 4 members |

### 5.4 Async Communication Guidelines

```
ASYNC COMMUNICATION RULES:
═══════════════════════════════════════════════════════════════

1. SLACK CHANNELS:
   #udara-general       — Announcements, general discussion
   #udara-ml            — ML/AI model discussions (Member 01)
   #udara-backend       — Backend/Edge discussions (Member 02)
   #udara-frontend      — Frontend/Map discussions (Member 03)
   #udara-bots          — Bot/Messenger discussions (Member 04)
   #udara-standup       — Async standup updates (if missed sync)
   #udara-alerts        — CI/CD failures, monitoring alerts

2. RESPONSE TIME EXPECTATIONS:
   • Normal messages: within 4 hours (working hours)
   • Urgent (blocks): within 1 hour
   • Critical (production down): immediately

3. NOTION WORKSPACE:
   • Sprint Board — Kanban with To Do / In Progress / Review / Done
   • Technical Docs — Architecture decisions, ADRs
   • Meeting Notes — Standups, planning, retros
   • Pilot Tracker — Site deployment status

4. CODE REVIEW:
   • PRs should be reviewed within 24 hours
   • Minimum 1 approval required (2 for DB schema changes)
   • Auto-assign reviewer based on file ownership
```

---

## 6. Git Branching Strategy

### 6.1 Branching Model

```
                    main (production)
                      │
                      │ merge (PR + 2 approvals)
                      ▼
                    develop (staging)
                   ╱    │    ╲
                  ╱     │     ╲
         feature/     hotfix/   release/
         ml-model     sync-fix  v0.1.0
         ussd-flow    api-auth
         map-view     ...

BRANCHING RULES:
═══════════════════════════════════════════════════════════════

1. main branch:
   • Protected, requires PR + 2 approvals + CI green
   • Only released features and hotfixes
   • Tagged with semantic version (v0.1.0, v0.2.0, etc.)

2. develop branch:
   • Integration branch for current sprint
   • All feature branches merge here via PR
   • Auto-deployed to staging environment
   • Requires 1 approval + CI green

3. feature/* branches:
   • Created from develop
   • Named: feature/<ticket-id>-<short-description>
   • Example: feature/UD-42-bayesian-engine
   • Squash-merged into develop

4. hotfix/* branches:
   • Created from main for urgent fixes
   • Named: hotfix/<ticket-id>-<description>
   • Merged into both main AND develop

5. release/* branches:
   • Created from develop for release stabilization
   • Named: release/v<version>
   • Merged into main after QA sign-off
```

### 6.2 Commit Convention

```
COMMIT MESSAGE FORMAT:
═══════════════════════════════════════════════════════════════

<type>(<scope>): <short description> [UD-<ticket-id>]

<body: detailed explanation if needed>

<footer: breaking changes, refs>

TYPES:
  feat:     New feature
  fix:      Bug fix
  docs:     Documentation only
  style:    Formatting, no code change
  refactor: Code restructuring, no behavior change
  perf:     Performance improvement
  test:     Test additions/changes
  chore:    Build process, dependencies
  ci:       CI/CD changes

EXAMPLES:
  feat(ml): add Bayesian resistance engine with PyMC [UD-42]
  fix(sync): handle CRDT conflict on concurrent case edits [UD-38]
  feat(ussd): implement report case 6-step flow [UD-27]
  perf(edge): lazy-load models with LRU cache [UD-31]
  test(rag): add retrieval recall@10 benchmark [UD-45]
  docs(api): update OpenAPI schema for /resistance endpoint [UD-19]
```

### 6.3 Pull Request Template

```markdown
## Description
[What does this PR do?]

## Type of Change
- [ ] Feature
- [ ] Bug Fix
- [ ] Refactor
- [ ] Performance
- [ ] Documentation

## Related Ticket
UD-XX

## Changes Made
- [List key changes]

## Testing
- [ ] Unit tests added/updated
- [ ] Integration tests pass
- [ ] Manual testing performed
- [ ] Edge cases covered

## Checklist
- [ ] Code follows project style
- [ ] Self-review completed
- [ ] No new warnings
- [ ] Documentation updated

## Screenshots (if applicable)
[Attach screenshots]
```

### 6.4 CI/CD Pipeline

```
PUSH TO feature/*          PUSH TO develop           PUSH TO main
       │                         │                        │
       ▼                         ▼                        ▼
  ┌─────────┐             ┌──────────┐            ┌──────────┐
  │  Lint   │             │   Lint   │            │   Lint   │
  │ (ruff,  │             │ (ruff,   │            │ (ruff,   │
  │ eslint) │             │ eslint)  │            │ eslint)  │
  └────┬────┘             └────┬─────┘            └────┬─────┘
       │                       │                       │
       ▼                       ▼                       ▼
  ┌─────────┐             ┌──────────┐            ┌──────────┐
  │  Test   │             │   Test   │            │   Test   │
  │ (pytest,│             │ (pytest, │            │ (pytest, │
  │  jest)  │             │  jest)   │            │  jest)   │
  └────┬────┘             └────┬─────┘            └────┬─────┘
       │                       │                       │
       ▼                       ▼                       ▼
  ┌─────────┐             ┌──────────┐            ┌──────────┐
  │ Build   │             │  Build   │            │  Build   │
  │ Docker  │             │  Docker  │            │  Docker  │
  └────┬────┘             └────┬─────┘            └────┬─────┘
       │                       │                       │
       │ (PR required)         ▼                       ▼
       │                 ┌──────────┐            ┌──────────┐
       │                 │  Deploy  │            │  Deploy  │
       │                 │ Staging  │            │Production│
       │                 │ (auto)   │            │ (manual) │
       │                 └──────────┘            └──────────┘
       │                       │                       │
       ▼                       ▼                       ▼
  [Wait for         [Staging tests        [Production
   PR review]         run automatically]    monitoring]
```

---

## 7. Communication Tools & Workflows

### 7.1 Tool Stack

| Tool | Purpose | Configuration |
|------|---------|---------------|
| **Slack** | Real-time messaging | Free plan, 8 channels |
| **Notion** | Project management, docs | Team workspace, shared DB |
| **GitHub** | Code repository, PRs, CI/CD | Private repo, Actions |
| **Google Meet** | Video calls | Free tier, calendar integration |
| **Figma** | UI/UX design (shared) | Free plan, prototype links |
| **Loom** | Video walkthroughs | Free plan, async updates |
| **Excalidraw** | Architecture diagrams | Collaborative, async |

### 7.2 Slack Channel Guide

```
# udara-general
  ├── Announcements (sprint start/end, releases)
  ├── General questions
  └── Watercooler / team bonding

# udara-ml (Member 01 primary)
  ├── Model performance discussions
  ├── Quantization experiments
  ├── RAG pipeline tuning
  └── Bayesian engine results

# udara-backend (Member 02 primary)
  ├── API design discussions
  ├── Edge runtime issues
  ├── Sync protocol debugging
  └── Database schema changes

# udara-frontend (Member 03 primary)
  ├── UI component discussions
  ├── Map visualization questions
  ├── ECharts dashboard configs
  └── Design system updates

# udara-bots (Member 04 primary)
  ├── Telegram bot development
  ├── WhatsApp API integration
  ├── Gamification features
  └── CHW onboarding flow design

# udara-standup
  ├── Async standup posts (if missed sync call)
  └── Daily status updates

# udara-alerts (automated)
  ├── CI/CD failure notifications
  ├── Monitoring alerts (Grafana)
  └── Error tracking (Sentry)
```

### 7.3 Notion Workspace Structure

```
UDARA AI — Notion Workspace
├── 📋 Sprint Board (Kanban)
│   ├── To Do
│   ├── In Progress
│   ├── In Review
│   └── Done
├── 📝 Meeting Notes
│   ├── Daily Standups
│   ├── Sprint Planning
│   ├── Demos
│   └── Retrospectives
├── 🏗️ Technical Documentation
│   ├── Architecture Decision Records (ADR)
│   ├── API Design Notes
│   ├── Model Performance Logs
│   └── Infrastructure Diagrams
├── 🚀 Deployment
│   ├── Pilot Site Tracker
│   ├── Deployment Runbook
│   └── Incident Response Playbook
└── 📊 Sprint Metrics
    ├── Velocity Chart
    ├── Burndown Chart
    └── Quality Metrics
```

### 7.4 Escalation Path

```
ESCALATION MATRIX:
═══════════════════════════════════════════════════════════════

LEVEL 1: Peer Discussion (0-1 hour)
├── Post in relevant #udara-* channel
├── Tag relevant team member(s)
└── If unresolved in 1 hour → Level 2

LEVEL 2: Pair Session (1-4 hours)
├── Schedule ad-hoc pair programming session
├── Use Excalidraw for visual problem-solving
├── Document findings in Notion
└── If unresolved in 4 hours → Level 3

LEVEL 3: Team Discussion (4-24 hours)
├── Add to next standup agenda
├── Schedule dedicated team call
├── Evaluate options, make team decision
├── Document ADR if architecture-level decision
└── If unresolved → Escalate to stakeholder

BLOCKING ISSUES:
├── If any member is blocked for > 24 hours
├── Automatic flag in standup
├── Project Lead facilitates resolution
└── Consider scope adjustment if needed
```

---

## 8. RACI Responsibility Matrix

### 8.1 RACI Legend

| Code | Meaning |
|------|---------|
| **R** | **Responsible** — Does the work |
| **A** | **Accountable** — Owns the outcome, final decision |
| **C** | **Consulted** — Provides input before decision |
| **I** | **Informed** — Notified after decision |

### 8.2 Full RACI Matrix

```
TASK / DELIVERABLE                          M01   M02   M03   M04
                                           ML    BE    FE    Bot
────────────────────────────────────────────────────────────────────
ARCHITECTURE & DESIGN
  Overall system architecture               C     R     C     C
  Edge-Cloud architecture                   R     A     I     I
  Three-Door interface design               C     C     R     A
  Database schema design                    C     R     I     I
  Security architecture                     C     R     C     I
  API contract design                       C     R     C     C

ML / AI
  Model selection & quantization            R/A   I     I     I
  Bayesian resistance engine               R/A   C     I     I
  RAG pipeline design & implementation      R/A   C     I     I
  Agent ingestion system                   R/A   C     C     I
  NER pipeline (AfroBERT)                  R/A   I     I     C
  OCR pipeline (PaddleOCR)                 R/A   I     I     C
  ASR pipeline (MMS-ASR)                   R/A   I     I     C
  Model performance benchmarking            R     I     I     I

BACKEND / EDGE
  FastAPI gateway implementation            I     R/A   I     I
  Edge runtime (RPi 5)                     C     R/A   I     I
  Sync protocol (Delta + CRDT)             C     R/A   I     I
  USSD session manager                     I     R/A   I     C
  PostgreSQL / Neo4j schemas               C     R/A   I     I
  SQLite edge schema                       C     R/A   I     I
  Africa's Talking integration             I     R/A   I     C
  Authentication (Keycloak)                I     R/A   I     I

FRONTEND
  Next.js application setup                I     C     R/A   I
  MapLibre resistance maps                 C     I     R/A   I
  ECharts dashboards                       I     C     R/A   I
  PWA (service worker, offline)            I     C     R/A   I
  Design system (shadcn/ui)                I     I     R/A   I
  Internationalization (i18n)              C     I     R/A   I

BOTS / INTEGRATION
  Telegram bot (aiogram 3.x)              C     I     I     R/A
  WhatsApp Cloud API                      C     I     I     R/A
  Unified Bot Core                        C     I     I     R/A
  CHW onboarding flows                    C     I     I     R/A
  Gamification system                     I     C     C     R/A
  E2E bot testing                         C     I     I     R/A

CROSS-CUTTING
  Docker / DevOps                          C     R     C     C
  CI/CD pipeline                           C     R     C     C
  Documentation                            C     C     C     C
  E2E testing (full system)               R     R     R     R
  Pilot deployment                         C     R     C     C
  Sprint planning / review                R     R     R     R
  Security audit                           C     R     I     I
────────────────────────────────────────────────────────────────────
```

### 8.3 RACI Summary Statistics

```
RESPONSIBILITY DISTRIBUTION:
═══════════════════════════════════════════════════════════════

Member    | R+A Count | R Count | C Count | I Count | Total Touches
──────────┼───────────┼─────────┼─────────┼─────────┼───────────────
M01 (ML)  |     9     |    2    |    8    |    15   |      34
M02 (BE)  |    11     |    2    |    7    |    14   |      34
M03 (FE)  |     7     |    0    |    6    |    21   |      34
M04 (Bot) |     7     |    0    |    4    |    23   |      34
──────────┴───────────┴─────────┴─────────┴─────────┴───────────────

KEY INSIGHTS:
• M02 (Backend/Edge) has the most accountable items (11) — they own
  the infrastructure backbone
• M01 (ML/AI) is second (9) — critical AI/ML decisions
• All members participate in cross-cutting concerns (E2E, planning)
• No single point of failure — every component has secondary ownership
```

---

## 9. Risk Register & Mitigation

### 9.1 Risk Assessment Matrix

```
LIKELIHOOD vs IMPACT:
═══════════════════════════════════════════════════════════════

         │ Low Impact │ Med Impact │ High Impact │ Critical
─────────┼────────────┼────────────┼─────────────┼──────────
High     │            │ R3         │ R1, R4      │ R2
Likely   │            │            │             │
─────────┼────────────┼────────────┼─────────────┼──────────
Medium   │            │ R5         │ R6          │ R7
Likely   │            │            │             │
─────────┼────────────┼────────────┼─────────────┼──────────
Low      │ R8, R9     │            │ R10         │
Likely   │            │            │             │
─────────┴────────────┴────────────┴─────────────┴──────────
```

### 9.2 Risk Register

| ID | Risk | Likelihood | Impact | Mitigation | Owner |
|----|------|-----------|--------|------------|-------|
| **R1** | RPi 5 insufficient RAM for all models | High | High | Lazy loading, graceful degradation, memory monitoring | M01, M02 |
| **R2** | Internet connectivity too unreliable for sync | High | Critical | USB fallback, long offline support, delta compression | M02 |
| **R3** | Model accuracy too low for clinical use | Medium | Medium | Fine-tuning on local AMR data, expert review, confidence thresholds | M01 |
| **R4** | Africa's Talking API changes/breaks | High | High | Abstraction layer, sandbox testing, fallback SMS gateway | M02, M04 |
| **R5** | WhatsApp Cloud API approval delays | Medium | Medium | Start approval early, develop TG bot in parallel | M04 |
| **R6** | AMR data quality from CHWs is poor | Medium | High | Input validation, NER extraction, data quality scoring | M01, M04 |
| **R7** | Security breach / patient data leak | Low | Critical | AES-256, Presidio PII scrubbing, Keycloak SSO, security audit | M02 |
| **R8** | Team member unavailability | Low | Low | Cross-training, documentation, pair programming | All |
| **R9** | Open-source dependency breaks | Low | Low | Pin versions, lock files, Docker images | M02 |
| **R10** | Pilot site infrastructure issues (power, hardware) | Low | High | Pre-deployment visit, UPS battery backup, spare RPi unit | M02 |

### 9.3 Risk Mitigation Deep Dive: RPi 5 Memory Management

```
MEMORY MANAGEMENT STRATEGY:
═══════════════════════════════════════════════════════════════

RPi 5 8GB RAM Budget:
┌──────────────────────────────────────────────────────┐
│  TOTAL AVAILABLE: ~7.5GB (after OS + services)       │
├──────────────────────────────────────────────────────┤
│  Llama 3.2 3B INT4:      ~1.7GB   [ALWAYS LOADED]  │
│  AfroBERT INT8:           ~0.35GB  [LAZY LOAD]      │
│  PaddleOCR INT8:          ~0.5GB   [LAZY LOAD]      │
│  MMS-ASR INT8:            ~0.6GB   [LAZY LOAD]      │
│  ChromaDB:                ~0.3GB   [ALWAYS LOADED]  │
│  FastAPI + SQLite:        ~0.2GB   [ALWAYS LOADED]  │
│  Bayesian Engine (PyMC):  ~0.4GB   [LAZY LOAD]      │
├──────────────────────────────────────────────────────┤
│  PEAK USAGE: ~4.05GB     HEADROOM: ~3.45GB          │
│  SAFE THRESHOLD: 6.5GB   → DROP LAZY MODELS         │
└──────────────────────────────────────────────────────┘

GRACEFUL DEGRADATION ORDER:
  1. If RAM > 6.5GB: All models available
  2. If RAM > 5.5GB: Drop Bayesian Engine (compute-heavy)
  3. If RAM > 4.5GB: Drop OCR (least critical for basic flow)
  4. If RAM > 3.5GB: Drop ASR (text input still works)
  5. ALWAYS KEEP: Llama + AfroBERT + ChromaDB (core functionality)
```

---

## 10. Quality Gates & Definition of Done

### 10.1 Definition of Done (DoD)

Every task/story must meet ALL criteria before moving to "Done":

```
DEFINITION OF DONE CHECKLIST:
═══════════════════════════════════════════════════════════════

☐ Code complete
  ☐ Implements acceptance criteria from ticket
  ☐ Follows project code style (ruff/black for Python, eslint for TS)
  ☐ No TODO comments left without tracking ticket
  ☐ Error handling for all edge cases

☐ Testing
  ☐ Unit tests pass (coverage > 80% for new code)
  ☐ Integration tests pass (if applicable)
  ☐ Manual testing performed
  ☐ Edge cases tested

☐ Code Review
  ☐ PR created from feature branch
  ☐ At least 1 team member approved
  ☐ All review comments addressed

☐ Documentation
  ☐ Docstrings for all public functions/classes
  ☐ API changes reflected in OpenAPI spec
  ☐ README updated if user-facing change

☐ CI/CD
  ☐ All CI checks pass (lint, test, build)
  ☐ Docker image builds successfully
  ☐ Deployed to staging environment

☐ Acceptance
  ☐ Meets acceptance criteria
  ☐ Demo recorded (if significant feature)
  ☐ Product owner / stakeholder approved (if applicable)
```

### 10.2 Quality Gates Per Week

| Week | Gate | Criteria | Decision |
|------|------|----------|----------|
| Week 1 | Environment Gate | All 4 Docker containers start, all scaffolds load | Proceed if GREEN |
| Week 2 | Edge AI Gate | Models quantized, < 6GB RAM on RPi, inference < 3s | Proceed if GREEN |
| Week 3 | Interface Gate | USSD + Telegram + Web all functional end-to-end | Proceed if GREEN |
| Week 4 | Intelligence Gate | RAG recall@10 > 0.7, Bayesian output valid, maps render | Proceed if GREEN |
| Week 5 | Pilot Gate | All E2E tests pass, 3 RPi deployed, monitoring active | LAUNCH if GREEN |

---

## 11. Appendices

### Appendix A: Technology Radar

```
TECHNOLOGY RADAR — UDARA AI (as of 2025-01):
═══════════════════════════════════════════════════════════════

ADOPT (use now):
  • llama.cpp (LLM inference)
  • FastAPI (backend)
  • Next.js 14 (frontend)
  • MapLibre GL (maps)
  • Redis (sessions)
  • Docker / Balena (deployment)

TRIAL (use in pilot, evaluate):
  • Yjs (CRDT sync)
  • Qdrant (cloud vector DB)
  • aiogram 3.x (Telegram)
  • TimescaleDB (time-series)
  • Keycloak (auth)

ASSESS (research for post-MVP):
  • Apache Kafka (event streaming)
  • TensorFlow Lite (alternative model runtime)
  • Matrix protocol (alternative messenger)
  • ODK Collect (offline form alternative)
  • DHIS2 integration (national HIS)

HOLD (do not use):
  • OpenAI API (requires internet, not edge-compatible)
  • Firebase (vendor lock-in)
  • GraphQL (over-engineering for our API surface)
```

### Appendix B: Acronym Glossary

| Acronym | Full Form |
|---------|-----------|
| AMR | Antimicrobial Resistance |
| CHW | Community Health Worker |
| CRDT | Conflict-free Replicated Data Type |
| IDW | Inverse Distance Weighting |
| INT4/INT8 | 4-bit / 8-bit Integer Quantization |
| LLM | Large Language Model |
| MOH | Ministry of Health |
| NER | Named Entity Recognition |
| OTA | Over-The-Air (updates) |
| PWA | Progressive Web App |
| RACI | Responsible, Accountable, Consulted, Informed |
| RAG | Retrieval-Augmented Generation |
| RPi | Raspberry Pi |
| SIR | Susceptible-Infected-Recovered (model) |
| USSD | Unstructured Supplementary Service Data |
| WAL | Write-Ahead Logging |
| WER | Word Error Rate |

### Appendix C: Contact Information Template

```
TEAM CONTACTS:
═══════════════════════════════════════════════════════════════

Member 01 (ML/AI):
  Email: [member01@udara.ai]
  Slack: @member01
  GitHub: @member01
  Timezone: [TBD]
  Strengths: PyTorch, NLP, Bayesian stats

Member 02 (Backend/Edge):
  Email: [member02@udara.ai]
  Slack: @member02
  GitHub: @member02
  Timezone: [TBD]
  Strengths: FastAPI, databases, embedded systems

Member 03 (Frontend):
  Email: [member03@udara.ai]
  Slack: @member03
  GitHub: @member03
  Timezone: [TBD]
  Strengths: React, data visualization, maps

Member 04 (Integration/Bot):
  Email: [member04@udara.ai]
  Slack: @member04
  GitHub: @member04
  Timezone: [TBD]
  Strengths: Bot development, integration, testing
```

---

> **Document End**  
> Next: [05-edge-cloud-sync.md](./05-edge-cloud-sync.md) | Prev: [03-tech-stack.md](./03-tech-stack.md)
