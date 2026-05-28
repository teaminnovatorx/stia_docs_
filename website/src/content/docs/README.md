# UDARA AI — AI-Powered AMR Surveillance for Sub-Saharan Africa

> **"Udara"** means *life* in several African languages. UDARA AI gives life to antimicrobial
> resistance (AMR) data by turning raw community observations into actionable, real-time
> surveillance intelligence.

---

## What This Repo Is

A design-first spec library for a hybrid edge-cloud AMR surveillance platform built for sub-Saharan Africa's real constraints — feature phones, offline clinics, 12+ languages, counterfeit drugs, and fragmented paper-based reporting.

**Zero implementation code exists yet.** This repo holds 29 numbered design documents, a problem-space anchor doc, and a license. The actual platform (edge runtime, cloud backend, web dashboard, bots) is yet to be built.

---

## The Problem

| Metric | Sub-Saharan Africa | Global Average |
|--------|-------------------|----------------|
| AMR-attributable mortality | **24 per 100,000** | 16 per 100,000 |
| Annual AMR deaths | ~1.27 million | 4.95 million (total) |
| Access to lab-based AMR data | <5% of facilities | ~60% |
| Median AMR surveillance coverage | <2% of population | ~45% |

AMR surveillance in sub-Saharan Africa is fragmented: paper-based records, months-long delays, no CHW data capture, and cloud-only solutions that don't work where connectivity is intermittent.

---

## The Solution

| Dimension | Current State | UDARA AI Approach |
|-----------|--------------|-------------------|
| **Data Collection** | Paper-based, weeks/months delay | Real-time via USSD, WhatsApp, Telegram, Voice |
| **Language** | English/French only | 12+ African languages via MMS-ASR + AfroBERT NER |
| **Connectivity** | Rural clinics often offline | Edge AI on RPi 5 — works fully disconnected |
| **Resistance** | Reactive lab reports | Proactive Bayesian forecasting with spatial correlation |
| **Access** | Web-only, excludes 60% of CHWs | Three-Door interface (USSD → Chat → Web) |
| **Drug Quality** | Undetected counterfeits | Snap-Detect OCR validates drug labels instantly |

Architecture detail: [`01-architecture-overview.md`](01-architecture-overview.md)

---

## Files

| # | File | Topic |
|---|------|-------|
| 01 | [`01-architecture-overview.md`](01-architecture-overview.md) | System architecture, data flows, failure modes |
| 02 | [`02-three-door-interface.md`](02-three-door-interface.md) | USSD, WhatsApp, Telegram, PWA, Web dashboard |
| 03 | [`03-tech-stack.md`](03-tech-stack.md) | Complete technology stack for all layers |
| 04 | [`04-team-overview-sprint-plan.md`](04-team-overview-sprint-plan.md) | Team composition, 5-week sprint plan |
| 05 | [`05-edge-cloud-sync.md`](05-edge-cloud-sync.md) | Edge-cloud sync protocol design |
| 06 | [`06-docker-dev-environment.md`](06-docker-dev-environment.md) | Docker development environment setup |
| 07 | [`07-database-schemas.md`](07-database-schemas.md) | Database schemas, ER diagrams |
| 08 | [`08-api-contracts.md`](08-api-contracts.md) | API contracts and endpoint design |
| 09 | [`09-ussd-strategy.md`](09-ussd-strategy.md) | USSD session management, menu trees |
| 10 | [`10-dual-messenger-strategy.md`](10-dual-messenger-strategy.md) | WhatsApp + Telegram messenger strategy |
| 11 | [`11-security-privacy.md`](11-security-privacy.md) | Encryption, PII handling, threat model |
| 12 | [`12-member-01-ml-ai.md`](12-member-01-ml-ai.md) | ML/AI Engineer role, model ownership |
| 13 | [`13-member-02-backend-edge.md`](13-member-02-backend-edge.md) | Backend + Edge Engineer role |
| 14 | [`14-usps-features.md`](14-usps-features.md) | USPs, features, competitive moat |
| 15 | [`15-member-03-frontend-engineer.md`](15-member-03-frontend-engineer.md) | Frontend Engineer role |
| 16 | [`16-member-04-integration-bot-engineer.md`](16-member-04-integration-bot-engineer.md) | Integration + Bot Engineer role |
| 17 | [`17-predictive-resistance-forecasting.md`](17-predictive-resistance-forecasting.md) | Bayesian resistance prediction engine |
| 18 | [`18-outbreak-simulator.md`](18-outbreak-simulator.md) | Outbreak simulation and visualization |
| 19 | [`19-voice-first-interface.md`](19-voice-first-interface.md) | Voice-first reporting in local languages |
| 20 | [`20-deployment-devops.md`](20-deployment-devops.md) | Deployment, DevOps, infrastructure |
| 21 | [`21-multilingual-support.md`](21-multilingual-support.md) | Multi-language support strategy |
| 22 | [`22-glass-compliance.md`](22-glass-compliance.md) | WHO GLASS compliance and regulatory |
| 23 | [`23-snap-detect-drug-ocr.md`](23-snap-detect-drug-ocr.md) | Drug label OCR via Snap-Detect |
| 24 | [`24-chw-gamification.md`](24-chw-gamification.md) | CHW gamification and incentives |
| 25 | [`25-sync-protocol.md`](25-sync-protocol.md) | Sync protocol deep dive |
| 26 | [`26-ci-cd-pipeline.md`](26-ci-cd-pipeline.md) | CI/CD pipeline design |
| 27 | [`27-pilot-plan.md`](27-pilot-plan.md) | Pilot deployment plan |
| 28 | [`28-monitoring-evaluation.md`](28-monitoring-evaluation.md) | Monitoring, evaluation, KPIs |
| 29 | [`29-investor-demo-guide.md`](29-investor-demo-guide.md) | Investor demo guide |
| — | [`PS.md`](PS.md) | Problem space document (standalone) |
| — | [`LICENSE`](LICENSE) | AGPL-3.0 |

---

## Phase

The project is in **design phase**. Next step: start implementation of the first slice. See `04-team-overview-sprint-plan.md` for the planned build sequence.

---

## License

CC BY 4.0 — see [LICENSE](LICENSE).
