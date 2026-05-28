# Pilot Plan — Country Selection, Site Deployment & 12-Week Budget

**UDARA AI — Antimicrobial Resistance Surveillance Platform**
**Document Version:** 2.4.1
**Date:** 2025-01-15
**Author:** UDARA AI Deployment Team
**Classification:** Internal — Pilot Operations
**Review Cycle:** Bi-weekly during pilot phase

---

## Table of Contents

1. [Executive Summary](#1-executive-summary)
2. [Pilot Objective & Hypotheses](#2-pilot-objective--hypotheses)
3. [Country Selection Criteria & Scoring](#3-country-selection-criteria--scoring)
4. [Country Candidate Profiles](#4-country-candidate-profiles)
5. [Recommendation: Nigeria, Oyo State](#5-recommendation-nigeria-oyo-state)
6. [Pilot Site Specifications](#6-pilot-site-specifications)
7. [12-Week Timeline](#7-12-week-timeline)
8. [Hardware Bill of Materials](#8-hardware-bill-of-materials)
9. [Software Stack & Deployment](#9-software-stack--deployment)
10. [CHW Training Program](#10-chw-training-program)
11. [Comprehensive Budget](#11-comprehensive-budget)
12. [Risk Register](#12-risk-register)
13. [Success Metrics & Go/No-Go Criteria](#13-success-metrics--gono-go-criteria)
14. [Governance & Decision Authority](#14-governance--decision-authority)
15. [Post-Pilot Transition Plan](#15-post-pilot-transition-plan)
16. [Appendices](#16-appendices)

---

## 1. Executive Summary

UDARA AI is a first-of-its-kind edge-AI antimicrobial resistance (AMR) surveillance platform designed specifically for resource-limited health posts in sub-Saharan Africa. This document outlines the operational plan for a 12-week pilot deployment across three carefully selected sites in Oyo State, Nigeria, beginning in Q2 2025.

The pilot targets three core validation hypotheses:

| # | Hypothesis | Validation Method | Success Threshold |
|---|-----------|-------------------|-------------------|
| H1 | Edge AI inference works reliably on Raspberry Pi 5 hardware in field conditions | Uptime monitoring, inference latency tracking, crash logs | ≥99% uptime, <5s text inference |
| H2 | Community Health Workers (CHWs) adopt UDARA AI as primary reporting tool | CHW activation rate, reporting frequency, satisfaction surveys | ≥80% adoption by Week 6 |
| H3 | Resistance data collected is actionable for public health decision-making | Resistance pattern detection, guidance adherence, supervisor feedback | ≥1 pattern detected before national lab |

The total pilot budget is **$3,780** (including 20% contingency), with hardware costs of $250/site and 12 weeks of operational expenditure. The pilot leverages the University of Ibadan and University College Hospital (UCH) as research partners, providing clinical oversight, data validation, and local credibility.

---

## 2. Pilot Objective & Hypotheses

### 2.1 Primary Objective

> **Validate UDARA AI as a viable, scalable, and cost-effective AMR surveillance tool in real sub-Saharan African health post conditions over a 12-week period.**

### 2.2 Secondary Objectives

- Assess the reliability of edge AI inference on Raspberry Pi 5 under variable power, connectivity, and environmental conditions
- Measure CHW adoption rates and identify barriers to sustained use
- Evaluate data quality (completeness, accuracy, timeliness) of AMR reports submitted through the system
- Test the full data pipeline: CHW report → edge processing → cloud sync → dashboard visualization
- Validate the gamification engine's effect on CHW engagement and reporting consistency
- Generate preliminary resistance surveillance data for Oyo State to demonstrate public health value
- Document operational lessons for scale-up to additional Nigerian states and neighboring countries

### 2.3 Core Validation Hypotheses

#### Hypothesis H1: Edge AI Reliability

**Statement:** The UDARA AI edge runtime, running 4 concurrent ML models (NER classifier, sentiment analyzer, drug resistance predictor, image OCR) on a Raspberry Pi 5 8GB, will maintain ≥99% uptime and process ≥50 AMR case reports per week across three sites under real-world conditions.

**Validation Approach:**
- Prometheus metrics collection every 10 seconds
- Automated health checks via edge-monitoring container
- Daily uptime/latency reports emailed to engineering team
- Crash dump analysis for any downtime events

**Acceptance Criteria:**
- API uptime ≥ 99% (measured weekly, allowing maintenance windows)
- Text inference latency < 5 seconds (p95)
- Image inference latency < 10 seconds (p95)
- Voice transcription latency < 3 seconds (p95)
- Zero unrecoverable crashes (soft restarts allowed)

```
┌─────────────────────────────────────────────────────────────────┐
│                    HYPOTHESIS H1: EDGE AI RELIABILITY           │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│   CHW Report ──▶ [RPi 5 8GB] ──▶ Inference ──▶ Response        │
│       │             │                    │            │          │
│       │        ┌────┴────┐         ┌────┴────┐      │          │
│       │        │ Model 1 │         │ Model 2 │      │          │
│       │        │ NER     │         │ Sentiment│      │          │
│       │        │ ~500MB  │         │ ~300MB  │      │          │
│       │        └─────────┘         └─────────┘      │          │
│       │        ┌─────────┐         ┌─────────┐      │          │
│       │        │ Model 3 │         │ Model 4 │      │          │
│       │        │ AMR Pred│         │ OCR     │      │          │
│       │        │ ~600MB  │         │ ~400MB  │      │          │
│       │        └─────────┘         └─────────┘      │          │
│       │                                        │              │
│       └────────────────────────────────────────┘              │
│                                                                 │
│   Target: All 4 models concurrent, <5s total, 99% uptime       │
│   RAM Budget: ~4GB models + 2GB OS + 2GB headroom = 8GB       │
│   Storage:  256GB SD (models: 2GB, data: 50GB+, OS: 20GB)     │
└─────────────────────────────────────────────────────────────────┘
```

#### Hypothesis H2: CHW Adoption

**Statement:** At least 80% of enrolled CHWs will submit ≥1 AMR report per week through UDARA AI by Week 6 of the pilot, and ≥60% will sustain weekly reporting through Week 12.

**Validation Approach:**
- Daily activity tracking (system analytics)
- Weekly usage reports per CHW
- Monthly satisfaction surveys via WhatsApp
- Qualitative interviews at pilot midpoint and conclusion

**Acceptance Criteria:**
- CHW Activation Rate ≥ 80% by Week 6
- Weekly Active Reporters ≥ 60% sustained from Week 6-12
- Average reporting frequency ≥ 3 reports/CHW/week by Week 8
- CHW Satisfaction Score ≥ 4.0/5.0 on endline survey
- Zero CHW drop-offs due to technical frustration (may drop for other reasons)

#### Hypothesis H3: Data Actionability

**Statement:** Resistance surveillance data collected through UDARA AI will reveal ≥1 spatial resistance pattern or emerging resistance trend at least 1 week before the national reference laboratory reports the same finding.

**Validation Approach:**
- Compare UDARA AI resistance maps with concurrent UCH microbiology lab data
- Track time-to-detection for any new resistance patterns
- Monthly review meeting with UCH clinical microbiologists
- Cross-reference with Nigeria AMR surveillance network (NARSP) quarterly reports

**Acceptance Criteria:**
- ≥50 unique AMR case reports per week across all sites by Week 8
- Resistance map shows statistically significant spatial variation (p < 0.05)
- ≥1 resistance pattern detected ≥7 days before national lab confirmation
- Guidance adherence rate ≥ 70% (CHWs follow AI-generated treatment guidance)

### 2.4 Hypothesis Dependency Map

```
                    ┌──────────────────┐
                    │   PILOT START    │
                    └────────┬─────────┘
                             │
              ┌──────────────┼──────────────┐
              ▼              ▼              ▼
        ┌──────────┐  ┌──────────┐  ┌──────────────┐
        │    H1    │  │    H2    │  │     H3       │
        │Edge AI   │  │   CHW    │  │   Data       │
        │Reliability│  │ Adoption │  │ Actionability│
        └────┬─────┘  └────┬─────┘  └──────┬───────┘
             │              │               │
             │     ┌────────┴────────┐      │
             │     │   H2 depends    │      │
             │     │   on H1         │      │
             │     │ (system must    │      │
             │     │  work for CHWs  │      │
             │     │  to adopt it)   │      │
             │     └────────┬────────┘      │
             │              │               │
             │     ┌────────┴────────┐      │
             │     │   H3 depends    │      │
             └─────│   on H1 AND H2  │◄─────┘
                   │ (need working   │
                   │  system AND     │
                   │  adoption to    │
                   │  get data)      │
                   └─────────────────┘
                             │
                    ┌────────┴─────────┐
                    │  GO/NO-GO        │
                    │  DECISION        │
                    │  (Week 12)       │
                    └──────────────────┘
```

---

## 3. Country Selection Criteria & Scoring

### 3.1 Methodology

Country selection follows a weighted multi-criteria scoring framework. Each criterion is scored 1-5 (1 = very poor fit, 5 = excellent fit) by the deployment team with input from regional health advisors. The maximum possible score is 35 points.

Scores are assigned based on publicly available data from:
- WHO Global Antimicrobial Resistance and Use Surveillance System (GLASS)
- GSMA Mobile Economy Reports (2024)
- ITU World Telecommunication Indicators
- World Bank Governance Indicators
- National AMR Action Plans submitted to WHO/Tripartite
- Local partner consultations (conducted Nov-Dec 2024)

### 3.2 Scoring Criteria Detailed

#### Criterion 1: AMR Burden (Weight: Critical)

| Score | Description |
|-------|-------------|
| 5 | WHO critical priority pathogens highly prevalent; GLASS participant with published data; >50,000 AMR-attributable deaths/year |
| 4 | High AMR burden; GLASS participant; 20,000-50,000 AMR deaths/year |
| 3 | Moderate AMR burden; GLASS enrollee; some published surveillance data |
| 2 | Low-moderate burden; limited surveillance data; not yet in GLASS |
| 1 | Minimal AMR data; no national surveillance program |

**Data Sources:** 2022 GLASS country reports, 2019 Global Burden of Disease AMR study (Lancet), WHO Priority Pathogens List for R&D of New Antibiotics

#### Criterion 2: Mobile Penetration (Weight: High)

| Score | Description |
|-------|-------------|
| 5 | >70% smartphone penetration; >95% mobile coverage; strong 4G/5G |
| 4 | 60-70% smartphone; >90% mobile; good 4G in urban areas |
| 3 | 40-60% smartphone; 80-90% mobile; 3G widespread |
| 2 | 20-40% smartphone; 60-80% mobile; 2G/3G mix |
| 1 | <20% smartphone; <60% mobile; limited connectivity |

**Data Sources:** GSMA Mobile Economy Sub-Saharan Africa 2024, Ookla Speedtest Intelligence

#### Criterion 3: Internet Reliability at Health Posts (Weight: High)

| Score | Description |
|-------|-------------|
| 5 | >75% uptime at rural health facilities; fiber or reliable 4G |
| 4 | 60-75% uptime; 4G available but occasional outages |
| 3 | 50-60% uptime; 3G with regular disruptions |
| 2 | 30-50% uptime; 2G with frequent failures |
| 1 | <30% uptime; internet essentially unavailable |

**Note:** This criterion specifically targets health post conditions, not national averages. Urban/rural disparity is accounted for by requiring site-level assessment.

#### Criterion 4: Regulatory Environment (Weight: Medium)

| Score | Description |
|-------|-------------|
| 5 | Comprehensive data protection law enacted and enforced; clear digital health regulations; AMR surveillance legal framework exists |
| 4 | Data protection law enacted; digital health guidelines available; AMR national action plan in place |
| 3 | Data protection bill in progress; some digital health policy; AMR action plan drafted |
| 2 | Draft data protection; minimal digital health policy |
| 1 | No data protection framework; no digital health regulations |

**Data Sources:** AU Convention on Cyber Security and Personal Data Protection ratification status, national data protection authority websites, WHO/FAO/OIE national AMR action plan tracker

#### Criterion 5: Existing CHW Network (Weight: High)

| Score | Description |
|-------|-------------|
| 5 | >50,000 CHWs; formalized national CHW program; regular training and supervision |
| 4 | 20,000-50,000 CHWs; national program with some gaps; regular supervision |
| 3 | 10,000-20,000 CHWs; nascent national program; irregular supervision |
| 2 | 5,000-10,000 CHWs; CHW program exists but informal |
| 1 | <5,000 CHWs; no coordinated CHW program |

**Data Sources:** WHO Global Health Workforce Statistics, USAID/CHW Hub country profiles, national community health strategy documents

#### Criterion 6: Partner Availability (Weight: Medium-High)

| Score | Description |
|-------|-------------|
| 5 | Tier-1 research university + teaching hospital in pilot zone; existing AMR research program; expressed interest in partnership |
| 4 | Research university + hospital; some AMR research; potential partner |
| 3 | University + regional hospital; limited AMR research |
| 2 | Medical school or research institute; no AMR focus |
| 1 | No potential academic or clinical partner identified |

**Assessment:** Conducted via direct outreach (email + virtual meetings) during Nov-Dec 2024

#### Criterion 7: Political Stability & Operational Security (Weight: Medium)

| Score | Description |
|-------|-------------|
| 5 | Stable democracy; peaceful transfers of power; low corruption index; safe for foreign operations |
| 4 | Generally stable; some political tensions; manageable operational risk |
| 3 | Moderate stability; periodic unrest; precautions needed |
| 2 | Unstable; frequent disruptions; high operational risk |
| 1 | Conflict zone; not suitable for operations |

**Data Sources:** World Bank Worldwide Governance Indicators (Political Stability), Global Peace Index, INFORM Risk Index

### 3.3 Scoring Matrix

```
┌─────────────────────────────────────────────────────────────────────────────────┐
│                    COUNTRY SELECTION SCORING MATRIX                              │
├──────────────────┬──────┬──────┬──────┬──────┬──────┬──────┬──────┬───────┬──────┤
│   Criterion      │ Wgt  │ NGA  │ KEN  │ RWA  │ GHA  │ SEN  │ SAF  │ ETH   │ TZA  │
├──────────────────┼──────┼──────┼──────┼──────┼──────┼──────┼──────┼───────┼──────┤
│ 1. AMR Burden    │  C   │  5   │  4   │  3   │  3   │  3   │  4   │  3    │  3   │
│ 2. Mobile Pen.   │  H   │  3   │  4   │  3   │  4   │  3   │  5   │  3    │  3   │
│ 3. Internet Rel. │  H   │  3   │  4   │  4   │  3   │  3   │  4   │  3    │  2   │
│ 4. Regulatory    │  M   │  4   │  4   │  5   │  3   │  3   │  4   │  3    │  3   │
│ 5. CHW Network   │  H   │  5   │  4   │  4   │  4   │  3   │  3   │  3    │  3   │
│ 6. Partners      │ MH   │  4   │  4   │  4   │  3   │  3   │  4   │  3    │  2   │
│ 7. Stability     │  M   │  4   │  4   │  5   │  4   │  3   │  3   │  3    │  4   │
├──────────────────┼──────┼──────┼──────┼──────┼──────┼──────┼──────┼───────┼──────┤
│ TOTAL            │      │ 28   │ 28   │ 28   │ 24   │ 21   │ 27   │  21   │ 20   │
│ BONUS: Partner LOI│      │ +0   │ +0   │ +0   │ -    │ -    │ -    │  -    │  -   │
│ ADJUSTED TOTAL   │      │ 28   │ 26*  │ 24*  │ 24   │ 21   │ 27   │  21   │ 20   │
└──────────────────┴──────┴──────┴──────┴──────┴──────┴──────┴──────┴───────┴──────┘

* Kenya and Rwanda adjusted down for pilot-readiness (on-the-ground logistics complexity)
  NGA: Nigeria  KEN: Kenya  RWA: Rwanda  GHA: Ghana
  SEN: Senegal  SAF: South Africa  ETH: Ethiopia  TZA: Tanzania
```

### 3.4 Tie-Breaking Analysis

When scores are equal or close (Nigeria 28, Kenya 26, Rwanda 24), the following tie-breakers are applied in order:

1. **Partner commitment level** — Has a formal Letter of Intent (LOI) been signed?
2. **AMR data gap severity** — Is the country an AMR surveillance dark zone?
3. **Operational readiness** — Can we deploy within 60 days?
4. **Scale potential** — Can success here catalyze regional adoption?

```
┌─────────────────────────────────────────────────────────────┐
│              TIE-BREAKER COMPARISON: NGA vs KEN vs RWA       │
├───────────────────┬─────────────┬───────────┬───────────────┤
│   Tie-Breaker     │  Nigeria    │   Kenya   │    Rwanda     │
├───────────────────┼─────────────┼───────────┼───────────────┤
│ Partner LOI       │ In progress │ Expressed │ Expressed     │
│ AMR Data Gap      │ SEVERE      │ MODERATE  │ MODERATE      │
│ Deploy in 60 days │ YES         │ LIKELY    │ LIKELY        │
│ Regional Catalyst │ WEST AFRICA │ EAST AFR  │ EAST AFR      │
│ CHW Scale         │ 200,000+    │ 100,000+  │ 45,000        │
│ Decision          │ ★ SELECTED  │ Reserve 1 │ Reserve 2     │
└───────────────────┴─────────────┴───────────┴───────────────┘
```

---

## 4. Country Candidate Profiles

### 4.1 Nigeria — Score: 28/35 (SELECTED)

#### Basic Profile

| Attribute | Value |
|-----------|-------|
| Population | 223.8 million (2024 est.) |
| GDP per capita | $2,184 (2023) |
| Life expectancy | 53.9 years |
| Health expenditure (% GDP) | 3.0% |
| Mobile subscribers | 222.5 million ( subscriptions) |
| Smartphone penetration | 44-47% |
| Internet users | 108.7 million (48.7%) |
| 4G coverage | ~50% geographic, ~80% population |
| Active CHWs | 200,000+ (formal + informal) |
| Teaching hospitals | 55+ |

#### AMR Situation

Nigeria carries the highest AMR burden in sub-Saharan Africa. Key statistics:

- **Estimated 64,500 AMR-attributable deaths** in 2019 (Lancet study)
- **Third-highest global burden** of AMR mortality (after India and Pakistan)
- High prevalence of extended-spectrum beta-lactamase (ESBL) producing organisms
- Widespread community distribution of fluoroquinolone-resistant E. coli
- High rates of methicillin-resistant Staphylococcus aureus (MRSA) in community settings
- Significant over-the-counter antibiotic sales without prescription

| Priority Pathogen | Resistance Rate | Trend |
|-------------------|----------------|-------|
| E. coli (3GC-R) | 45-60% | ↑ Increasing |
| Klebsiella (3GC-R) | 50-65% | ↑ Increasing |
| S. aureus (MRSA) | 30-45% | → Stable |
| Salmonella (FLU-R) | 35-50% | ↑ Increasing |
| A. baumannii (CARB-R) | 60-75% | ↑ Increasing |

#### Digital Infrastructure

- **Networks:** MTN (largest, ~40% market share), Airtel (~28%), Glo (~26%), 9Mobile (~6%)
- **4G Availability:** Strong in Lagos, Abuja, Port Harcourt; moderate in Ibadan, Kaduna; weak in rural areas
- **Internet at Health Posts:** Estimated 40-55% uptime (varies significantly by state)
- **Power:** National grid unreliable; most facilities have generators or solar backup
- **Smartphone Usage Among CHWs:** Estimated 50-60% in urban areas, 30-40% in rural

#### Regulatory Environment

- **NITDA Act 2007** — National Information Technology Development Agency
- **NDPR (Nigeria Data Protection Regulation) 2019** — Data protection framework
- **NDPA (Nigeria Data Protection Act) 2023** — Comprehensive data protection law (signed June 2023)
- **NITDA Data Protection Compliance** — Mandatory for organizations processing personal data
- **National AMR Action Plan 2017-2022** — Currently being updated for 2023-2027
- **Nigeria AMR Surveillance Network (NARSP)** — Coordinated by NCDC

#### Partner Landscape

| Partner | Type | Status | Value to Pilot |
|---------|------|--------|----------------|
| University of Ibadan | Research University | LOI in progress | Clinical validation, CHW training, ethics review |
| University College Hospital (UCH) | Teaching Hospital | LOI in progress | Lab confirmation, clinical oversight, data comparison |
| Oyo State Primary Health Care Board | Government | Preliminary contact | Site access, CHW coordination, regulatory approval |
| NCDC | National agency | Awareness meeting held | National AMR data comparison, policy pathway |
| MTN Nigeria | Telecom | Not yet contacted | Potential data sponsorship, connectivity support |

#### Why Nigeria Was Selected

```
┌──────────────────────────────────────────────────────────────────┐
│                    NIGERIA SELECTION RATIONALE                    │
├──────────────────────────────────────────────────────────────────┤
│                                                                  │
│  ┌─────────────┐    ┌─────────────┐    ┌─────────────┐          │
│  │   HIGHEST    │    │  LARGEST    │    │  STRONGEST   │         │
│  │  AMR BURDEN  │    │   CHW       │    │  PARTNER     │         │
│  │  in Africa   │───▶│  NETWORK    │───▶│  ECOSYSTEM   │         │
│  │  (64.5K      │    │  (200K+     │    │  (UI + UCH   │         │
│  │  deaths/yr)  │    │  CHWs)      │    │  + NCDC)     │         │
│  └─────────────┘    └─────────────┘    └─────────────┘          │
│         │                  │                   │                  │
│         └──────────────────┼───────────────────┘                  │
│                            ▼                                      │
│                  ┌─────────────────┐                              │
│                  │  SCALE CATALYST │                              │
│                  │  Success → West  │                              │
│                  │  Africa (15      │                              │
│                  │  countries)      │                              │
│                  └─────────────────┘                              │
│                                                                  │
│  Risk Factors:                                                   │
│    • Power reliability (mitigated by UPS)                        │
│    • Security concerns (mitigated by local coordinator)          │
│    • Bureaucracy (mitigated by UI partnership)                   │
└──────────────────────────────────────────────────────────────────┘
```

### 4.2 Kenya — Score: 26/35 (RESERVE 1)

#### Basic Profile

| Attribute | Value |
|-----------|-------|
| Population | 55.1 million |
| GDP per capita | $2,099 |
| Life expectancy | 61.7 years |
| Smartphone penetration | 54-58% |
| 4G coverage | ~65% population |
| Active CHWs | 100,000+ (Community Health Volunteers) |

#### AMR Situation

- Estimated 8,500 AMR-attributable deaths in 2019
- Growing ESBL prevalence in community settings
- KEMRI Wellcome Trust has robust AMR research program
- National AMR surveillance strengthening underway

#### Strengths

- KEMRI partnership would provide world-class research validation
- Strong 4G along Nairobi-Mombasa corridor
- Data Protection Act (DPA) 2019 is well-implemented
- M-Pesa ecosystem enables innovative reward mechanisms
- Kenya Medical Research Institute (KEMRI) provides lab capacity

#### Limitations for First Pilot

- Moderate AMR burden (less compelling urgency than Nigeria)
- CHV program is decentralized (county-level coordination complexity)
- KEMRI partnership requires longer negotiation timeline
- Slightly higher operational costs (Nairobi is expensive)

#### Pilot Zone: Nairobi + Kiambu Counties

| Site | Type | Connectivity | CHWs | Patients/Day |
|------|------|-------------|------|-------------|
| Kayole Health Centre | Urban | 4G strong | 4 | 60+ |
| Ruiru Sub-County Hospital | Peri-urban | 4G moderate | 3 | 40+ |
| Kikuyu Level 3 Hospital | Semi-rural | 3G/4G | 3 | 30+ |

### 4.3 Rwanda — Score: 24/35 (RESERVE 2)

#### Basic Profile

| Attribute | Value |
|-----------|-------|
| Population | 14.2 million |
| GDP per capita | $966 |
| Life expectancy | 69.0 years |
| Smartphone penetration | 33% (fastest-growing in Africa) |
| 4G coverage | ~75% population |
| Active CHWs | 45,000 |

#### AMR Situation

- Lower AMR burden but rapidly increasing
- Limited AMR surveillance data (data gap itself is valuable)
- Rwanda Biomedical Centre (RBC) has nascent AMR program

#### Strengths

- Exceptional internet infrastructure (4G to most of country)
- World-class e-governance (Irembo platform)
- Strong political will for innovation
- Compact geography simplifies logistics
- Eager innovation culture ("Silicon Valley of Africa")

#### Limitations for First Pilot

- Smaller CHW network (fewer data points)
- Lower AMR burden (less compelling "burning platform")
- Fewer AMR research partners
- Smaller market for scale-up
- French/English bilingual adds complexity

#### Pilot Zone: Kigali + Eastern Province

| Site | Type | Connectivity | CHWs | Patients/Day |
|------|------|-------------|------|-------------|
| Kimihurura Health Centre | Urban | 4G strong | 3 | 50+ |
| Bugesera District Hospital | Semi-rural | 4G good | 2 | 25+ |
| Nyamata Health Post | Rural | 3G/4G | 2 | 15+ |

---

## 5. Recommendation: Nigeria, Oyo State

### 5.1 Decision Summary

> **UDARA AI will pilot in Oyo State, Nigeria, with the University of Ibadan (UI) and University College Hospital (UCH) as research partners. Deployment begins Q2 2025 for 12 weeks.**

### 5.2 Oyo State Profile

| Attribute | Detail |
|-----------|--------|
| Population | 7.8 million (2022 census projection) |
| Capital | Ibadan (3.6 million — largest city in West Africa by land area) |
| Health facilities | 1,200+ primary health care facilities |
| CHWs | ~8,000 registered |
| Teaching hospital | University College Hospital (UCH), Ibadan |
| Research university | University of Ibadan (established 1948) |
| Major telecoms | MTN, Airtel, Glo (all active in state) |
| AMR data | Limited — most surveillance concentrated in Lagos/Abuja |

### 5.3 Why Oyo State Specifically

```
┌────────────────────────────────────────────────────────────────────┐
│                 WHY OYO STATE, NIGERIA                             │
├────────────────────────────────────────────────────────────────────┤
│                                                                    │
│  1. ACADEMIC EXCELLENCE                                            │
│     ├── University of Ibadan — #1 university in Nigeria           │
│     ├── UCH — premier teaching hospital in West Africa             │
│     ├── Department of Medical Microbiology — active AMR research   │
│     └── College of Medicine — 2,000+ medical students (pipeline)  │
│                                                                    │
│  2. CONNECTIVITY GRADIENT                                          │
│     ├── Ibadan urban = good 4G (PHC Jericho)                      │
│     ├── Semi-rural = 3G intermittent (Awe)                        │
│     └── Rural = 2G/poor (Fiditi) — tests offline-first            │
│                                                                    │
│  3. AMR SURVEILLANCE GAP                                          │
│     ├── National surveillance centered on Lagos/Abuja             │
│     ├── Oyo State = data desert for AMR                           │
│     └── New data = immediate public health value                  │
│                                                                    │
│  4. OPERATIONAL FEASIBILITY                                        │
│     ├── 2-hour drive from Lagos (logistics hub)                   │
│     ├── Ibadan = safe, stable, research-friendly                   │
│     ├── UI provides ethics review (UI/UCH Ethics Committee)       │
│     └── Lower cost of operations than Lagos/Abuja                 │
│                                                                    │
│  5. SCALE CATALYST                                                 │
│     ├── Oyo State PHC Board = potential Phase 2 partner           │
│     ├── NCDC Southwest Zone coordination possible                  │
│     └── Nigeria pilot → credibility for West Africa expansion     │
│                                                                    │
└────────────────────────────────────────────────────────────────────┘
```

### 5.4 Partnership Structure

```
┌──────────────────────────────────────────────────────────────────┐
│                 PARTNERSHIP STRUCTURE                             │
│                                                                  │
│                    ┌──────────────────┐                          │
│                    │    UDARA AI      │                          │
│                    │  (Lead + Tech)   │                          │
│                    └────────┬─────────┘                          │
│                             │                                    │
│              ┌──────────────┼──────────────┐                     │
│              ▼              ▼              ▼                     │
│     ┌─────────────┐ ┌─────────────┐ ┌─────────────┐            │
│     │University of│ │  University  │ │  Oyo State  │            │
│     │  Ibadan     │ │  College     │ │  PHC Board  │            │
│     │             │ │  Hospital    │ │             │            │
│     │• Ethics     │ │  (UCH)       │ │• Site access│            │
│     │  review     │ │              │ │• CHW coord. │            │
│     │• Research   │ │• Lab         │ │• Regulatory │            │
│     │  design     │ │  validation  │ │  approval   │            │
│     │• Data       │ │• Clinical    │ │• Phase 2    │            │
│     │  analysis   │ │  oversight   │ │  pathway    │            │
│     │• CHW        │ │• Microbiology│ │             │            │
│     │  training   │ │  comparison  │ │             │            │
│     └─────────────┘ └─────────────┘ └─────────────┘            │
│                                                                  │
│  Advisory (Non-Operational):                                      │
│  • NCDC — National data comparison, policy guidance              │
│  • WHO Nigeria — Technical assistance, GLASS alignment           │
│  • NITDA — Data protection compliance guidance                   │
└──────────────────────────────────────────────────────────────────┘
```

### 5.5 Memorandum of Understanding (MoU) Key Terms

| Item | UDARA AI | University of Ibadan | UCH |
|------|----------|----------------------|-----|
| Ethics review | Provides protocol | UI/UCH Ethics Committee reviews and approves | Co-signatory |
| Data ownership | UDARA AI owns platform data | Collaborator access for research | Co-signatory |
| Publication rights | Co-author on all publications | Lead author on clinical validation papers | Co-author |
| CHW training | Develops curriculum | Provides training venue, facilitators | Clinical content review |
| Lab validation | Provides platform | N/A | Microbiological confirmation |
| Duration | 12 months (pilot + analysis) | 12 months | 12 months |
| Extension option | Yes, with 90-day notice | Yes | Yes |

---

## 6. Pilot Site Specifications

### 6.1 Site Selection Methodology

Three sites were selected to represent a **connectivity gradient** — from urban (reliable 4G) to rural (poor/poorly reliable connectivity). This ensures UDARA AI is tested across the full spectrum of conditions it will face at scale.

```
┌──────────────────────────────────────────────────────────────────────┐
│                    CONNECTIVITY GRADIENT                             │
│                                                                      │
│  URBAN ◄───────────────────────────────────────────────────► RURAL  │
│                                                                      │
│  ╔════════════╗    ╔════════════╗    ╔════════════╗                 │
│  ║  PHC       ║    ║  Rural HP  ║    ║  Community  ║                │
│  ║  Jericho   ║    ║  Awe       ║    ║  HP Fiditi  ║                │
│  ║            ║    ║            ║    ║             ║                │
│  ║  4G ●●●●●  ║    ║  3G ●●●○○  ║    ║  2G ●○○○○  ║                │
│  ║  uptime:   ║    ║  uptime:   ║    ║  uptime:   ║                │
│  ║  85-95%    ║    ║  50-70%    ║    ║  30-50%    ║                │
│  ║            ║    ║            ║    ║            ║                │
│  ║  Tests:    ║    ║  Tests:    ║    ║  Tests:    ║                │
│  ║  Full      ║    ║  Intermitt.║    ║  Offline   ║                │
│  ║  cloud     ║    ║  sync      ║    ║  + USB     ║                │
│  ╚════════════╝    ╔════════════╝    ╚════════════╝                 │
│       ▲                ▲                   ▲                        │
│       │                │                   │                        │
│    Best case     Typical case        Worst case                    │
│    scenario      scenario            scenario                       │
│                                                                      │
└──────────────────────────────────────────────────────────────────────┘
```

### 6.2 Site 1: PHC Jericho, Ibadan

| Attribute | Detail |
|-----------|--------|
| **Location** | Jericho, Ibadan North LGA, Oyo State |
| **GPS Coordinates** | 7.4316° N, 3.8975° E |
| **Type** | Urban Primary Health Centre |
| **Facility Level** | PHC Level 2 |
| **Catchment Population** | ~25,000 |
| **CHWs Assigned** | 3 (all smartphone-enabled) |
| **Patients per Day** | 50-70 |
| **Connectivity** | MTN 4G (3 bars), Airtel 4G (2 bars) |
| **Estimated Uptime** | 85-95% |
| **Power Supply** | National grid + generator |
| **Distance to UCH** | 2.5 km (10 min drive) |
| **Distance to UI** | 4.0 km (15 min drive) |
| **Security** | High — gated compound, 24/7 security |
| **AMR Relevance** | High patient volume = many antibiotic prescriptions; UCH referral pathway for lab confirmation |

#### Site 1 Rationale

PHC Jericho serves as the **"best-case scenario"** site. Its reliable 4G connectivity allows us to:
- Test full cloud-sync capabilities with minimal latency
- Validate WhatsApp and Telegram bot performance on strong networks
- Conduct frequent supervisor check-ins via video call
- Serve as the training venue for all CHWs
- Provide rapid feedback loop for engineering issues
- Compare cloud-first performance against edge-only sites

#### Site 1 Layout

```
┌────────────────────────────────────────────────────────────────────┐
│                    PHC JERICHO — SITE LAYOUT                       │
├────────────────────────────────────────────────────────────────────┤
│                                                                    │
│  ┌────────────────────────────────────────────────────────────┐   │
│  │  MAIN BUILDING                                             │   │
│  │  ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────────┐      │   │
│  │  │Consult.  │ │Consult.  │ │ Pharmacy │ │  Lab     │      │   │
│  │  │  Room 1  │ │  Room 2  │ │          │ │ (basic)  │      │   │
│  │  └──────────┘ └──────────┘ └──────────┘ └──────────┘      │   │
│  │                                                             │   │
│  │  ┌──────────────────────────────────────────────────┐      │   │
│  │  │              WAITING AREA (covered)              │      │   │
│  │  │              ~50 patient capacity                │      │   │
│  │  └──────────────────────────────────────────────────┘      │   │
│  └────────────────────────────────────────────────────────────┘   │
│                                                                    │
│  ┌──────────────────┐    ┌──────────────────┐                      │
│  │   STAFF ROOM     │    │   UDARA AI       │                      │
│  │                  │    │   EQUIPMENT:     │                      │
│  │  • CHW lockers   │    │                  │                      │
│  │  • Supervisor    │    │   ┌──────────┐  │                      │
│  │     desk         │    │   │  RPi 5   │  │                      │
│  │  • Charging      │    │   │  8GB     │  │                      │
│  │     station      │    │   └────┬─────┘  │                      │
│  │                  │    │        │        │                      │
│  │  ★ TRAINING      │    │   ┌────┴─────┐  │                      │
│  │    VENUE         │    │   │  4G USB  │  │                      │
│  │    (Days 1-3)    │    │   │  Modem   │  │                      │
│  │                  │    │   └──────────┘  │                      │
│  └──────────────────┘    │   ┌──────────┐  │                      │
│                          │   │  UPS     │  │                      │
│                          │   │  500VA   │  │                      │
│  ┌──────────────────┐    │   └──────────┘  │                      │
│  │   GENERATOR      │    │   ┌──────────┐  │                      │
│  │   (existing)     │    │   │  USB     │  │                      │
│  │                  │───▶│   │  Fallback│  │                      │
│  └──────────────────┘    │   └──────────┘  │                      │
│                          └──────────────────┘                      │
│                                                                    │
│  MTN Tower: 300m  │  Airtel Tower: 500m  │  Power Grid: Stable    │
└────────────────────────────────────────────────────────────────────┘
```

### 6.3 Site 2: Rural Health Post, Awe

| Attribute | Detail |
|-----------|--------|
| **Location** | Awe, Oyo State (Oyo East LGA boundary) |
| **GPS Coordinates** | 8.0833° N, 3.6833° E |
| **Type** | Semi-rural Health Post |
| **Facility Level** | PHC Level 1 |
| **Catchment Population** | ~8,000 |
| **CHWs Assigned** | 2 (1 smartphone, 1 feature phone with USSD) |
| **Patients per Day** | 20-30 |
| **Connectivity** | Glo 3G (1-2 bars, intermittent), Airtel 2G (1 bar) |
| **Estimated Uptime** | 50-70% |
| **Power Supply** | National grid (unreliable) + small solar panel |
| **Distance to UCH** | 55 km (1.5 hr drive) |
| **Security** | Moderate — open compound, daytime security |
| **AMR Relevance** | Represents majority health post in Nigeria; intermittent connectivity tests resilience |

#### Site 2 Rationale

Awe represents the **"typical scenario"** — the most common type of health post in Nigeria. Testing here validates:
- Edge AI performance on intermittent connectivity
- Automatic sync-when-available behavior
- Data queuing during offline periods
- CHW experience with USSD fallback (1 CHW on feature phone)
- Battery life management with unreliable power
- Practical logistics of hardware maintenance at distance

#### Site 2 Challenges & Mitigations

| Challenge | Severity | Mitigation |
|-----------|----------|------------|
| Intermittent 3G | High | Dual-SIM modem (Glo + Airtel); edge processing eliminates real-time cloud dependency |
| Power outages | High | UPS (500VA) = ~4 hours battery; solar panel supplementary; low-power RPi 5 mode |
| CHW with feature phone | Medium | USSD channel (*384# menu) provides full reporting; training includes USSD practice |
| Distance from UI/UCH | Medium | Weekly supervisor visit (local coordinator based in Ibadan); WhatsApp group for daily check-ins |
| Security | Medium | Equipment locked in staff room; RPi secured with cable lock; insurance coverage |

### 6.4 Site 3: Community Health Post, Fiditi

| Attribute | Detail |
|-----------|--------|
| **Location** | Fiditi, Afijio LGA, Oyo State |
| **GPS Coordinates** | 7.8500° N, 3.9500° E |
| **Type** | Rural Community Health Post |
| **Facility Level** | Health Post (sub-PHC) |
| **Catchment Population** | ~3,500 |
| **CHWs Assigned** | 1 (smartphone-enabled) |
| **Patients per Day** | 10-15 |
| **Connectivity** | MTN 2G (0-1 bar), Glo Edge (unreliable) |
| **Estimated Uptime** | 30-50% |
| **Power Supply** | No grid connection; solar panel only (daytime) |
| **Distance to UCH** | 42 km (1.2 hr drive) |
| **Security** | Lower — village setting, community trust-based security |
| **AMR Relevance** | Represents frontier health post; validates offline-first design; USB sync is primary data transfer method |

#### Site 3 Rationale

Fiditi is the **"stress test"** site — designed to push UDARA AI to its operational limits. If the system works here, it works anywhere. This site validates:
- **Pure offline operation**: 168+ hours without connectivity
- **USB fallback sync**: Weekly physical data transfer via USB drive
- **Solar power reliability**: RPi 5 power consumption on solar
- **Single-CHW operation**: System usability with minimal staffing
- **Community acceptance**: Rural community willingness to engage with digital AMR reporting
- **Low-literacy interface**: Voice reporting and icon-based navigation

#### Site 3 Power Budget

```
┌────────────────────────────────────────────────────────────────┐
│              FIDITI — SOLAR POWER BUDGET                       │
├────────────────────────────────────────────────────────────────┤
│                                                                │
│  Solar Panel: 100W (existing at health post)                  │
│  Battery: 12V 100Ah lead-acid (existing)                      │
│                                                                │
│  Load Analysis:                                                │
│  ┌──────────────────────┬──────────┬──────────┬──────────────┐ │
│  │ Device               │ Power(W) │ Hours/Day│ Daily (Wh)  │ │
│  ├──────────────────────┼──────────┼──────────┼──────────────┤ │
│  │ Raspberry Pi 5       │   5-12   │   10     │     85       │ │
│  │ 4G USB Modem         │    3-4   │   10     │     35       │ │
│  │ LED indicator        │    0.5   │   24     │     12       │ │
│  │ USB drive (sync)     │    1.5   │    0.5   │      1       │ │
│  │ CHW phone charging   │    5     │    2     │     10       │ │
│  ├──────────────────────┼──────────┼──────────┼──────────────┤ │
│  │ TOTAL                │          │          │    143 Wh     │ │
│  └──────────────────────┴──────────┴──────────┴──────────────┘ │
│                                                                │
│  Solar Generation (100W panel, ~5 peak hours): ~500 Wh/day     │
│  Battery Capacity: 12V × 100Ah = 1,200 Wh (usable ~600 Wh)   │
│                                                                │
│  Margin: 500Wh gen - 143Wh load = 357Wh/day surplus           │
│  Autonomy (no sun): 600Wh / 143Wh = 4.2 days                  │
│                                                                │
│  STATUS: ✓ ADEQUATE with 2.5x safety margin                   │
│  Note: RPi scheduled to power off at night to conserve energy  │
└────────────────────────────────────────────────────────────────┘
```

### 6.5 Site Comparison Summary

| Attribute | PHC Jericho | Rural HP Awe | CHP Fiditi |
|-----------|-------------|-------------|------------|
| Classification | Urban | Semi-rural | Rural |
| CHWs | 3 | 2 | 1 |
| Total CHWs | **6** | | |
| Patients/Day | 50-70 | 20-30 | 10-15 |
| Total Potential Reports | 30-42/week | 8-12/week | 4-6/week |
| Connectivity | 4G (85-95%) | 3G (50-70%) | 2G (30-50%) |
| Power | Grid + generator | Grid + solar | Solar only |
| Hospital Proximity | 2.5 km (UCH) | 55 km | 42 km |
| Primary Sync Method | Cloud | Edge + queued cloud | USB fallback |
| USSD Testing | Yes (backup) | Yes (primary for 1 CHW) | Yes (primary) |
| Voice Reporting | Tested | Tested | Critical channel |
| Supervisor Access | Daily | Weekly | Bi-weekly |

---

## 7. 12-Week Timeline

### 7.1 Overview

The 12-week pilot is structured in five phases, each with specific objectives, deliverables, and decision gates.

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                         12-WEEK PILOT TIMELINE                              │
├──────┬──────┬──────┬──────┬──────┬──────┬──────┬──────┬──────┬──────┬──────┬──────┬──────┤
│  W1  │  W2  │  W3  │  W4  │  W5  │  W6  │  W7  │  W8  │  W9  │ W10  │ W11  │ W12  │
├──────┼──────┼──────┼──────┼──────┼──────┼──────┼──────┼──────┼──────┼──────┼──────┤
│╔════╗│╔════╗│╔══════════════╗│╔═══════════════════════════════════════╗│╔════════════╗│
│║SITE║│║SITE║│║  SOFT LAUNCH  ║│║         ACTIVE PILOT                  ║│║EVALUATION ║│
│║PREP║│║PREP║│║  (parallel)   ║│║         (UDARA primary)               ║│║            ║│
│╚════╝│╚════╝│╚══════════════╝│╚═══════════════════════════════════════╝│╚════════════╝│
│      │      │                │                                         │               │
│ Hard │ Hard │ CHW training   │ Full UDARA           Paper            │ Data analysis │
│ ware │ ware │ Parallel       │ reporting            phased           │ CHW           │
│ insta│ insta│ reporting      │ Gamification         out              │ interviews   │
│ llatn│ llatn│ Daily          │ Weekly               (Week 9)         │ Report        │
│      │      │ check-ins      │ supervisor           Full deploy      │ writing       │
│      │      │                │ reviews              (Week 10)        │ Handover      │
│      │      │                │                      ★ GAMIFICATION   │               │
│      │      │                │                      ★ FULL OFFLINE   │               │
├──────┼──────┼──────┼──────┼──────┼──────┼──────┼──────┼──────┼──────┼──────┼──────┤
│      │      │      │      │      │ ★     │      │      │      │      │      │      │
│      │      │      │      │      │CHECK  │      │      │      │      │      │      │
│      │      │      │      │      │POINT  │      │      │      │      │      │      │
│      │      │      │      │      │W6     │      │      │      │      │      │      │
│      │      │      │      │      │Review │      │      │      │      │      │      │
│      │      │      │      │      │adoption│     │      │      │      │      │      │
│      │      │      │      │      │metrics │      │      │      │      │      │      │
└──────┴──────┴──────┴──────┴──────┴──────┴──────┴──────┴──────┴──────┴──────┴──────┘
```

### 7.2 Phase 1: Site Preparation (Week 1-2)

#### Week 1: Hardware Installation & Connectivity

**Objective:** All three sites physically equipped with UDARA AI hardware, connected to power and internet, and verified operational.

| Day | Activity | Responsible | Deliverable |
|-----|----------|-------------|-------------|
| Mon | Arrive Ibadan; meet local coordinator; equipment inventory | Deployment Lead | Equipment checklist signed |
| Tue | PHC Jericho — install RPi 5, connect 4G modem, test connectivity | Engineer + Coordinator | Site 1 operational |
| Wed | PHC Jericho — install UPS, verify power failover, test all channels | Engineer | Site 1 fully tested |
| Thu | Travel to Awe — install RPi 5, connect dual-SIM modem | Engineer + Coordinator | Site 2 hardware installed |
| Fri | Awe — test connectivity (Glo + Airtel), install UPS, power testing | Engineer | Site 2 operational |
| Sat | Buffer day — address any issues from first two sites | Engineer | Issues resolved or escalated |
| Sun | Rest day | — | — |

**Week 1 Technical Checklist:**

```
┌────────────────────────────────────────────────────────────────────┐
│              WEEK 1 INSTALLATION CHECKLIST                        │
├────────────────────────────┬──────┬──────┬──────┬────────────────┤
│ Item                       │PHC Je│Awe HP│Fiditi│ Notes          │
├────────────────────────────┼──────┼──────┼──────┼────────────────┤
│ RPi 5 unboxed & assembled  │  □   │  □   │  □   │                │
│ microSD inserted & booted  │  □   │  □   │  □   │ First boot log │
│ Docker containers running  │  □   │  □   │  □   │ All 4 healthy   │
│ Device ID registered       │  □   │  □   │  □   │ API confirmed   │
│ 4G modem connected         │  □   │  □   │  □   │ SIM active     │
│ Speedtest completed        │  □   │  □   │  □   │ ≥2 Mbps target │
│ UPS connected & tested     │  □   │  □   │  □   │ 4hr hold test  │
│ Power failover tested      │  □   │  □   │  □   │ Unplug test    │
│ WhatsApp bot tested        │  □   │  □   │  □   │ Send/receive   │
│ Telegram bot tested        │  □   │  □   │  □   │ Send/receive   │
│ USSD tested (*384#)        │  □   │  □   │  □   │ All menu items │
│ Voice recording tested     │  □   │  □   │  □   │ Playback OK    │
│ Cloud sync verified        │  □   │  □   │  □   │ Dashboard shows│
│ USB drive formatted        │  □   │  □   │  □   │ Fallback ready │
│ Physical security verified │  □   │  □   │  □   │ Cable lock     │
│ Photo of installation      │  □   │  □   │  □   │ For records    │
└────────────────────────────┴──────┴──────┴──────┴────────────────┘
```

#### Week 2: Fiditi Installation, Baseline Data & Training Prep

| Day | Activity | Responsible | Deliverable |
|-----|----------|-------------|-------------|
| Mon | Travel to Fiditi — install RPi 5, test solar power, assess connectivity | Engineer + Coordinator | Site 3 hardware installed |
| Tue | Fiditi — verify offline operation, test USB sync, install security measures | Engineer | Site 3 fully tested |
| Wed | Baseline data collection — review existing AMR/prescription records at all 3 sites | Coordinator + UI team | Baseline data spreadsheet |
| Thu | UI/UCH partnership meeting — finalize MoU, ethics review timeline | Deployment Lead + PI | Signed MoU (or draft) |
| Fri | Training material final review — translate key terms to Yoruba, print materials | Training Lead + Local translator | Training materials ready |
| Sat | Training venue preparation (PHC Jericho staff room — chairs, projector, refreshments) | Coordinator | Venue ready |
| Sun | Rest day | — | — |

### 7.3 Phase 2: Soft Launch (Week 3-4)

#### Week 3: CHW Training & Parallel Reporting Begins

**Objective:** All 6 CHWs trained; parallel reporting (paper + UDARA AI) begins; identify initial usability issues.

| Day | Activity | Responsible | Deliverable |
|-----|----------|-------------|-------------|
| Mon | **Training Day 1** — AMR awareness, USSD basics, UDARA AI introduction | Training Lead + UI Faculty | CHWs understand AMR & system |
| Tue | **Training Day 2** — Hands-on: WhatsApp, Telegram, voice, photo capture | Training Lead + Engineer | CHWs can use all channels |
| Wed | **Training Day 3** — Practice cases, troubleshooting, gamification walkthrough | Training Lead + Engineer | CHWs pass competency check |
| Thu | Soft launch Day 1 — CHWs begin parallel reporting (paper + UDARA) | All CHWs | First UDARA reports submitted |
| Fri | Daily check-in — collect feedback, fix issues | Coordinator | Issue log updated |
| Sat | WhatsApp group creation — all CHWs + coordinator + engineer | Coordinator | Support channel active |
| Sun | Rest day | — | — |

#### CHW Training Program Detail (See Section 10)

The 3-day training program is critical to pilot success. A detailed curriculum is provided in Section 10.

#### Week 4: Parallel Reporting & Iteration

**Objective:** Stabilize parallel reporting; address all usability issues; achieve ≥50% CHW reporting rate.

| Day | Activity | Responsible | Deliverable |
|-----|----------|-------------|-------------|
| Mon-Wed | Continued parallel reporting; daily WhatsApp check-ins | CHWs + Coordinator | Daily report count |
| Thu | Mid-week review — check adoption metrics, address drop-offs | Coordinator + Engineer | Week 4 mid-report |
| Fri | Software patch deployment (if any bugs found) | Remote Engineer | Patch notes |
| Sat | First gamification points distributed — welcome bonus + first report bonus | System (automated) | Leaderboard populated |
| Sun | Rest day | — | — |

**Week 4 Decision Gate:**

| Metric | Target | Go | Concern | Stop |
|--------|--------|----|---------|------|
| CHWs who submitted ≥1 report | ≥4/6 | ≥4 | 2-3 | 0-1 |
| System uptime (across 3 sites) | ≥90% | ≥90% | 75-89% | <75% |
| Critical bugs | 0 | 0 | 1-2 | 3+ |
| CHW-reported blockers | <3 | <3 | 3-5 | 6+ |

### 7.4 Phase 3: Active Pilot (Week 5-8)

**Objective:** UDARA AI becomes primary reporting tool; paper forms used only as backup; resistance data flowing to dashboard.

#### Week 5: Transition to UDARA Primary

| Activity | Detail |
|----------|--------|
| Reporting mode | UDARA AI = primary; paper = backup only |
| Supervisor role | Weekly in-person review at each site |
| Data flow | Reports → Edge → Cloud → Dashboard (verified daily) |
| Gamification | Weekly challenges activated (first week: "Report 5 cases") |
| Support | WhatsApp group + daily automated health check |

#### Week 6: CHECKPOINT — Go/No-Go Decision

**This is the critical decision point.** The pilot steering committee reviews adoption metrics and decides whether to continue, pivot, or stop.

```
┌────────────────────────────────────────────────────────────────────┐
│                 WEEK 6 GO/NO-GO DECISION FRAMEWORK                │
├─────────────────────────────┬──────────────────────────────────────┤
│                             │                                      │
│   GREEN (GO)                │   All of:                            │
│   Continue pilot            │   • CHW Activation ≥ 80% (≥5/6)     │
│                             │   • System uptime ≥ 95%             │
│                             │   • ≥20 AMR reports submitted       │
│                             │   • Zero critical bugs              │
│                             │   • CHW satisfaction ≥ 3.5/5        │
│                             │                                      │
├─────────────────────────────┼──────────────────────────────────────┤
│                             │                                      │
│   YELLOW (PIVOT)            │   Any of:                            │
│   Modify approach           │   • CHW Activation 50-79%            │
│   (1 week to fix)           │   • System uptime 80-94%            │
│                             │   • 10-19 AMR reports               │
│                             │   • 1-2 moderate bugs                │
│                             │   • CHW satisfaction 3.0-3.4/5      │
│                             │                                      │
├─────────────────────────────┼──────────────────────────────────────┤
│                             │                                      │
│   RED (STOP)                │   Any of:                            │
│   Halt pilot                │   • CHW Activation < 50%            │
│                             │   • System uptime < 80%             │
│                             │   • <10 AMR reports                 │
│                             │   • Critical hardware failure       │
│                             │   • Safety/security incident        │
│                             │   • CHW satisfaction < 3.0/5        │
│                             │                                      │
└─────────────────────────────┴──────────────────────────────────────┘
```

#### Weeks 7-8: Steady-State Operations

| Week | Focus | Key Activities |
|------|-------|----------------|
| 7 | Data quality | Completeness checks; supervisor spot-checks; data validation against UCH lab |
| 8 | Surveillance value | First resistance map review; trend analysis; comparison with national data |

**Week 8 Deliverables:**

- Resistance heat map for Oyo State (preliminary)
- CHW adoption report (activation, retention, satisfaction)
- Data quality report (completeness, accuracy, timeliness)
- Technical reliability report (uptime, latency, sync rate)
- Revised risk register (updated with actual observations)

### 7.5 Phase 4: Full Deployment (Week 9-10)

#### Week 9: Paper Phase-Out

| Activity | Detail |
|----------|--------|
| Paper forms | Completely discontinued at all 3 sites |
| Reporting | 100% through UDARA AI |
| Gamification | Full suite active — points, badges, streaks, leaderboard |
| Challenges | "Accuracy Champion" week (bonus points for complete reports) |

#### Week 10: Full Deployment & Optimization

| Activity | Detail |
|----------|--------|
| System | Fully optimized based on 8 weeks of feedback |
| CHWs | Operating independently, minimal support needed |
| Data | Consistent flow to dashboard; resistance patterns emerging |
| Coordinator | Shifts from daily check-ins to weekly summary |
| Engineering | Address any remaining edge cases; prepare for scale |

### 7.6 Phase 5: Evaluation (Week 11-12)

#### Week 11: Data Analysis & CHW Interviews

| Day | Activity | Responsible |
|-----|----------|-------------|
| Mon-Tue | Quantitative data analysis — all KPIs calculated | Data Analyst + UI team |
| Wed | CHW interviews (Jericho + Awe) — semi-structured, 45 min each | PI + Coordinator |
| Thu | CHW interview (Fiditi) + focus group at PHC Jericho | PI + Coordinator |
| Fri | Preliminary findings shared with steering committee | PI |
| Sat | Begin evaluation report writing | PI + Data Analyst |

#### Week 12: Report Writing & Handover

| Day | Activity | Responsible |
|-----|----------|-------------|
| Mon-Tue | Complete evaluation report draft | PI + full team |
| Wed | Report review by UI/UCH partners | UI + UCH faculty |
| Thu | Final report | PI |
| Fri | Handover meeting with Oyo State PHC Board | PI + Coordinator + PHC Board |
| Sat | Equipment decommissioning or transition to local ownership | Engineer + Coordinator |
| Sun | Team departure | All |

### 7.7 Critical Path Diagram

```
┌──────────────────────────────────────────────────────────────────────┐
│                    CRITICAL PATH — PILOT DEPENDENCIES                │
├──────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  MoU Signed ──────▶ Ethics Approved ──────▶ Training Begins          │
│       │                    │                      │                  │
│       │                    │                      ▼                  │
│       │                    │               CHWs Competent            │
│       │                    │                      │                  │
│       │                    │                      ▼                  │
│       │                    │               Soft Launch              │
│       │                    │                      │                  │
│  Hardware ──▶ Sites ──▶ System ────────────────▶│                   │
│  Ordered   Ready    Operational                  │                  │
│       │                                              │              │
│       └──────────────────────────────────────────────┘              │
│                                                                      │
│  ★ CRITICAL: Hardware must arrive 2 weeks before training           │
│  ★ CRITICAL: Ethics approval must precede any CHW data collection   │
│  ★ CRITICAL: Local coordinator hired 4 weeks before Week 1         │
│                                                                      │
└──────────────────────────────────────────────────────────────────────┘
```

---

## 8. Hardware Bill of Materials

### 8.1 Per-Site Hardware

| # | Component | Specification | Quantity | Unit Cost | Total | Supplier |
|---|-----------|--------------|----------|-----------|-------|----------|
| 1 | Raspberry Pi 5 8GB | Broadcom BCM2712, Quad-core Cortex-A76 @ 2.4GHz | 1 | $80 | $80 | Raspberry Pi Foundation / Amazon |
| 2 | microSD Card | SanDisk Extreme 256GB, A2, V30 | 1 | $25 | $25 | SanDisk / Amazon |
| 3 | USB 4G/LTE Modem | Huawei E3372h-320 or ZTE MF833V, unlocked | 1 | $40 | $40 | Amazon / Local electronics |
| 4 | SIM Card + Data Plan | MTN/Airtel, 3 months, ≥10GB/month | 1 | $45 | $45 | MTN/Airtel Nigeria |
| 5 | UPS Battery Backup | 500VA standby UPS (APC/Genius) | 1 | $30 | $30 | Local electronics market (Ibadan) |
| 6 | Case + Heatsinks | Aluminium case + copper heatsink kit for RPi 5 | 1 | $20 | $20 | Amazon / AliExpress |
| 7 | USB Flash Drive | SanDisk 32GB, USB 3.0, for fallback sync | 1 | $10 | $10 | Local / Amazon |
| 8 | Ethernet Cable | Cat5e, 2m (backup if WiFi unreliable) | 1 | $3 | $3 | Local |
| 9 | Micro HDMI Cable | For debug monitor connection | 1 | $5 | $5 | Local |
| 10 | Power Supply | Official RPi 5 27W USB-C PSU | 1 | $12 | $12 | Raspberry Pi Foundation |
| 11 | Cable Lock | Kensington-style lock for RPi security | 1 | $8 | $8 | Amazon |
| 12 | Surge Protector | Multi-outlet with surge protection | 1 | $7 | $7 | Local |
| | | | | **Per-Site Total** | **$285** | |

### 8.2 Central / Shared Hardware

| # | Component | Specification | Quantity | Unit Cost | Total |
|---|-----------|--------------|----------|-----------|-------|
| 1 | Spare RPi 5 8GB | Replacement unit | 1 | $80 | $80 |
| 2 | Spare microSD | Pre-imaged backup | 2 | $25 | $50 |
| 3 | Spare USB Modem | Replacement unit | 1 | $40 | $40 |
| 4 | Debug Laptop | Used laptop for on-site troubleshooting | 1 | $200 | $200 |
| 5 | Mobile Hotspot | Backup internet for deployment team | 2 | $50 | $100 |
| | | | | **Shared Total** | **$470** |

### 8.3 Hardware Assembly Procedure

```
┌────────────────────────────────────────────────────────────────────┐
│              HARDWARE ASSEMBLY — STEP BY STEP                     │
├────────────────────────────────────────────────────────────────────┤
│                                                                    │
│  STEP 1: PREPARE SD CARD                                          │
│  ┌──────────────────────────────────────────────────────────┐    │
│  │ 1. Insert 256GB microSD into laptop                     │    │
│  │ 2. Flash Raspberry Pi OS Lite (64-bit, Bookworm)         │    │
│  │ 3. Enable SSH: touch /boot/ssh                           │    │
│  │ 4. Configure WiFi: wpa_supplicant.conf                  │    │
│  │ 5. Install Docker: curl -fsSL https://get.docker.com    │    │
│  │ 6. Pull UDARA AI containers:                            │    │
│  │    - udara-edge-runtime:latest                          │    │
│  │    - udara-model-server:latest                          │    │
│  │    - udara-monitoring:latest                            │    │
│  │    - udara-sync-agent:latest                            │    │
│  │ 7. Configure docker-compose.yml with site-specific      │    │
│  │    device ID and API endpoint                           │    │
│  │ 8. Test: docker-compose up -d                           │    │
│  │ 9. Verify all containers healthy: docker ps             │    │
│  │ 10. Eject and label SD card                             │    │
│  └──────────────────────────────────────────────────────────┘    │
│                                                                    │
│  STEP 2: ASSEMBLE RPi 5                                           │
│  ┌──────────────────────────────────────────────────────────┐    │
│  │ 1. Place RPi 5 board on anti-static mat                 │    │
│  │ 2. Attach copper heatsinks to CPU, RAM, and flash chips │    │
│  │ 3. Install aluminium case                                │    │
│  │ 4. Insert prepared microSD card                         │    │
│  │ 5. Connect 27W USB-C power supply                       │    │
│  │ 6. Connect USB 4G modem to USB 3.0 port                │    │
│  │ 7. Insert SIM card into modem                           │    │
│  │ 8. Connect USB flash drive to USB 2.0 port             │    │
│  │ 9. (Optional) Connect Ethernet cable                    │    │
│  │ 10. Power on and verify boot (green LED activity)       │    │
│  └──────────────────────────────────────────────────────────┘    │
│                                                                    │
│  STEP 3: ON-SITE INSTALLATION                                     │
│  ┌──────────────────────────────────────────────────────────┐    │
│  │ 1. Identify secure location for RPi (staff room/office)  │    │
│  │ 2. Position RPi with adequate ventilation               │    │
│  │ 3. Connect RPi to UPS                                   │    │
│  │ 4. Connect UPS to wall power / solar inverter           │    │
│  │ 5. Attach cable lock                                    │    │
│  │ 6. Position 4G modem for best signal (use window/door)  │    │
│  │ 7. Verify connectivity: ping api.udara.ai               │    │
│  │ 8. Run full system test (all channels)                  │    │
│  │ 9. Photograph installation for records                  │    │
│  │ 10. Brief facility staff on equipment                   │    │
│  └──────────────────────────────────────────────────────────┘    │
│                                                                    │
└────────────────────────────────────────────────────────────────────┘
```

### 8.4 Hardware Failure Protocol

| Failure Type | Detection | Response Time | Resolution |
|-------------|-----------|---------------|------------|
| RPi crash | Prometheus alert (no heartbeat >5 min) | <4 hours | Remote reboot via UPS power cycle; if persistent, dispatch spare unit |
| SD card corruption | Docker container crash loop | <24 hours | Swap pre-imaged spare SD; re-sync from USB/cloud |
| 4G modem failure | No API connectivity despite RPi up | <24 hours | Replace modem; swap SIM; try alternate network |
| UPS battery drain | Battery <20% alert | <2 hours | Check power source; replace UPS if needed |
| Power grid failure (extended) | UPS battery <50% for >2 hours | N/A (automatic) | UPS provides 4-hour bridge; solar/generator takes over |
| Physical damage/theft | Coordinator site visit / security report | <48 hours | Deploy spare unit; file insurance claim |

---

## 9. Software Stack & Deployment

### 9.1 Edge Software Architecture

```
┌────────────────────────────────────────────────────────────────────┐
│              EDGE SOFTWARE STACK — RPi 5                           │
├────────────────────────────────────────────────────────────────────┤
│                                                                    │
│  ┌──────────────────────────────────────────────────────────┐    │
│  │                  HOST OS: Raspberry Pi OS Lite             │    │
│  │                  (Debian 12 Bookworm, 64-bit)             │    │
│  └────────────────────────┬─────────────────────────────────┘    │
│                           │                                       │
│  ┌────────────────────────┴─────────────────────────────────┐    │
│  │                    DOCKER ENGINE                          │    │
│  │                    (v24.0+)                               │    │
│  └───┬────────────┬──────────────┬──────────────┬───────────┘    │
│      │            │              │              │                  │
│  ┌───┴──────┐ ┌──┴──────────┐ ┌─┴──────────┐ ┌┴──────────┐    │
│  │udara-    │ │udara-model- │ │udara-      │ │udara-     │    │
│  │edge-     │ │server       │ │monitoring  │ │sync-agent │    │
│  │runtime   │ │             │ │            │ │           │    │
│  │          │ │             │ │            │ │           │    │
│  │• FastAPI │ │• ONNX RT    │ │• Prometheus│ │• SQLite   │    │
│  │• WhatsApp│ │• Model 1:   │ │• Grafana   │ │• Cloud    │    │
│  │  API     │ │  NER        │ │  Agent     │ │  sync     │    │
│  │• Telegram │ │• Model 2:   │ │• Log       │ │• USB sync │    │
│  │  API     │ │  Sentiment  │ │  forwarder │ │• Conflict │    │
│  │• USSD    │ │• Model 3:   │ │• Health    │ │  resolver │    │
│  │  handler │ │  AMR Pred   │ │  checks    │ │           │    │
│  │• Voice   │ │• Model 4:   │ │            │ │           │    │
│  │  transcr.│ │  OCR        │ │            │ │           │    │
│  │• SQLite  │ │• Model      │ │            │ │           │    │
│  │  local   │ │  mgmt       │ │            │ │           │    │
│  │  DB      │ │             │ │            │ │           │    │
│  └──────────┘ └─────────────┘ └────────────┘ └───────────┘    │
│                                                                    │
│  Resource Allocation:                                              │
│  ┌──────────────────┬──────────┬──────────────────────────┐      │
│  │ Component        │ RAM      │ Storage                  │      │
│  ├──────────────────┼──────────┼──────────────────────────┤      │
│  │ Host OS          │ 512 MB   │ 8 GB                     │      │
│  │ edge-runtime     │ 1.0 GB   │ 2 GB                     │      │
│  │ model-server     │ 4.0 GB   │ 4 GB (models + cache)    │      │
│  │ monitoring       │ 512 MB   │ 2 GB (logs + metrics)    │      │
│  │ sync-agent       │ 256 MB   │ 500 MB                   │      │
│  │ Reserved/Buffer  │ 1.7 GB   │ 239.5 GB (data + growth) │      │
│  ├──────────────────┼──────────┼──────────────────────────┤      │
│  │ TOTAL            │ 8.0 GB   │ 256 GB                   │      │
│  └──────────────────┴──────────┴──────────────────────────┘      │
│                                                                    │
└────────────────────────────────────────────────────────────────────┘
```

### 9.2 First Boot Auto-Configuration

When a UDARA AI RPi 5 is powered on for the first time, the following automatic sequence runs:

```
┌────────────────────────────────────────────────────────────────────┐
│              FIRST BOOT SEQUENCE                                    │
├────────────────────────────────────────────────────────────────────┤
│                                                                    │
│  1. BOOTSTRAP (0-30 seconds)                                      │
│     ├── Load Raspberry Pi OS Lite                                 │
│     ├── Mount microSD (read-only root for resilience)             │
│     ├── Initialize overlay filesystem (persistent data on SD)     │
│     └── Start Docker daemon                                        │
│                                                                    │
│  2. CONTAINER STARTUP (30-90 seconds)                              │
│     ├── Start udara-monitoring (collect boot metrics)             │
│     ├── Start udara-sync-agent (check for pending USB sync)       │
│     ├── Start udara-model-server (load ONNX models to RAM)       │
│     │   ├── Model 1: NER (~500MB, ~8s load)                      │
│     │   ├── Model 2: Sentiment (~300MB, ~5s load)                 │
│     │   ├── Model 3: AMR Predictor (~600MB, ~10s load)            │
│     │   └── Model 4: OCR (~400MB, ~7s load)                      │
│     └── Start udara-edge-runtime (API server on :8080)            │
│                                                                    │
│  3. DEVICE REGISTRATION (90-120 seconds)                           │
│     ├── Generate unique device ID (from hardware serial)          │
│     ├── Read site config from /boot/udara-config.json             │
│     │   ├── site_name: "PHC Jericho"                              │
│     │   ├── site_id: "NG-OY-JER-001"                             │
│     │   ├── api_endpoint: "https://api.udara.ai/v1"              │
│     │   └── channels: ["whatsapp", "telegram", "ussd", "voice"]  │
│     ├── Attempt HTTPS registration to cloud API                   │
│     │   ├── If SUCCESS: device registered, sync begins            │
│     │   └── If FAIL: queue registration, retry in 5 min          │
│     └── Store registration status locally                          │
│                                                                    │
│  4. CHANNEL BINDING (120-180 seconds)                              │
│     ├── WhatsApp: Connect to WhatsApp Business API               │
│     ├── Telegram: Connect to Telegram Bot API                     │
│     ├── USSD: Configure GSM modem for *384# shortcode            │
│     └── Voice: Configure Whisper model for transcription          │
│                                                                    │
│  5. HEALTH CHECK & READY (180-240 seconds)                          │
│     ├── Run self-diagnostic (all containers, all models)          │
│     ├── Send "UDARA AI Ready" message to coordinator WhatsApp    │
│     ├── Log boot complete to monitoring                           │
│     └── System ready for CHW interactions                          │
│                                                                    │
│  TOTAL BOOT TIME: ~4 minutes (cold start)                          │
│  SUBSEQUENT BOOT: ~2 minutes (models cached)                       │
│                                                                    │
└────────────────────────────────────────────────────────────────────┘
```

### 9.3 Offline-First Design Principles

```
┌────────────────────────────────────────────────────────────────────┐
│              OFFLINE-FIRST DATA FLOW                                │
├────────────────────────────────────────────────────────────────────┤
│                                                                    │
│  ONLINE MODE (Connectivity Available):                             │
│                                                                    │
│  CHW Report ──▶ Edge Processing ──▶ Response to CHW              │
│       │              │                      │                     │
│       │              ▼                      ▼                     │
│       │         Local SQLite           Immediate                │
│       │         (write)                response                  │
│       │              │                                            │
│       │              ▼                                            │
│       │         Sync Agent ──▶ Cloud API (HTTPS)                 │
│       │                            │                               │
│       │                            ▼                               │
│       │                      Dashboard Updated                    │
│       │                      (real-time)                           │
│       │                                                            │
│  OFFLINE MODE (No Connectivity):                                   │
│                                                                    │
│  CHW Report ──▶ Edge Processing ──▶ Response to CHW              │
│       │              │                      │                     │
│       │              ▼                      ▼                     │
│       │         Local SQLite           Immediate                │
│       │         (write)                response                  │
│       │              │                   (same speed!)           │
│       │              ▼                                            │
│       │         Sync Queue (pending sync flag)                   │
│       │              │                                            │
│       │              │  (when connectivity returns)               │
│       │              ▼                                            │
│       │         Sync Agent ──▶ Cloud API (HTTPS)                 │
│       │                            │                               │
│       │                            ▼                               │
│       │                      Dashboard Updated                    │
│       │                      (catch-up sync)                      │
│       │                                                            │
│  ★ KEY: CHW experience is IDENTICAL online or offline             │
│  ★ KEY: AI inference is LOCAL (no internet needed for AI)         │
│  ★ KEY: Data is never lost (SQLite WAL mode + USB fallback)      │
│                                                                    │
│  USB FALLBACK (Extended offline >48 hours):                        │
│                                                                    │
│  Coordinator visits site ──▶ Insert USB drive ──▶ System          │
│       │                              │              exports       │
│       │                              ▼              SQLite        │
│       │                        Data exported      to USB         │
│       │                        as encrypted                       │
│       │                        JSON bundle                        │
│       │                              │                             │
│       │                              ▼                             │
│       │                        Coordinator ──▶ USB to laptop       │
│       │                              │     ──▶ Cloud upload      │
│       │                              ▼                            │
│       │                        Dashboard Updated                  │
│       │                        (manual sync)                      │
│                                                                    │
└────────────────────────────────────────────────────────────────────┘
```

### 9.4 Docker Compose Configuration

```yaml
# docker-compose.yml (simplified for documentation)
version: '3.8'

services:
  edge-runtime:
    image: udara/edge-runtime:1.4.2
    container_name: udara_runtime
    restart: always
    ports:
      - "8080:8080"
    volumes:
      - udara_data:/var/lib/udara
      - udara_logs:/var/log/udara
    environment:
      - DEVICE_ID=${DEVICE_ID}
      - SITE_NAME=${SITE_NAME}
      - API_ENDPOINT=${API_ENDPOINT}
      - WHATSAPP_TOKEN=${WHATSAPP_TOKEN}
      - TELEGRAM_TOKEN=${TELEGRAM_TOKEN}
      - DB_PATH=/var/lib/udara/udara.db
      - LOG_LEVEL=INFO
    depends_on:
      - model-server
    deploy:
      resources:
        limits:
          memory: 1.5G

  model-server:
    image: udara/model-server:1.4.2
    container_name: udara_models
    restart: always
    ports:
      - "8501:8501"
    volumes:
      - model_cache:/var/lib/models
    environment:
      - MODEL_DIR=/var/lib/models
      - ONNX_RUNTIME_THREADS=4
    deploy:
      resources:
        limits:
          memory: 4.5G

  monitoring:
    image: udara/monitoring:1.4.2
    container_name: udara_monitor
    restart: always
    ports:
      - "9090:9090"  # Prometheus
    volumes:
      - udara_logs:/var/log/udara:ro
      - /proc:/host/proc:ro
    environment:
      - DEVICE_ID=${DEVICE_ID}

  sync-agent:
    image: udara/sync-agent:1.4.2
    container_name: udara_sync
    restart: always
    volumes:
      - udara_data:/var/lib/udara
      - /media/usb:/media/usb:rw
    environment:
      - API_ENDPOINT=${API_ENDPOINT}
      - SYNC_INTERVAL=300  # 5 minutes
      - USB_PATH=/media/usb
      - DEVICE_ID=${DEVICE_ID}
    depends_on:
      - edge-runtime

volumes:
  udara_data:
  udara_logs:
  model_cache:
```

---

## 10. CHW Training Program

### 10.1 Overview

| Attribute | Detail |
|-----------|--------|
| Duration | 3 days (24 contact hours) |
| Format | Classroom (Day 1) + Hands-on lab (Day 2) + Practice (Day 3) |
| Location | PHC Jericho Staff Room, Ibadan |
| Trainers | 1 UDARA AI training lead + 1 local clinical facilitator (UI) |
| Participants | 6 CHWs (3 Jericho, 2 Awe, 1 Fiditi) |
| Materials | Printed manuals, laminated quick-reference cards, Yoruba cheat sheets |
| Meals | Breakfast + lunch provided all 3 days |
| Transport | Transport reimbursement for Awe and Fiditi CHWs |
| Compensation | Training stipend: ₦5,000 (~$6) per day per CHW |

### 10.2 Day 1: AMR Awareness & System Introduction

**Time: 9:00 AM - 4:00 PM (7 hours)**

| Time | Session | Content | Method |
|------|---------|---------|--------|
| 9:00-9:30 | Welcome & Introductions | Participant introductions; pilot overview; ground rules | Circle introductions |
| 9:30-10:00 | Why Are We Here? | AMR crisis in Nigeria; what UDARA AI does; what CHWs will do differently | Presentation + discussion |
| 10:00-10:15 | Break | — | — |
| 10:15-11:30 | Understanding Antimicrobial Resistance | What are antibiotics/antimicrobials? What is resistance? How does resistance spread? Why does it matter to YOUR community? | Interactive lecture with visual aids |
| 11:30-12:30 | The CHW Role in AMR Surveillance | How CHWs are the "eyes and ears" of the health system; what data is needed; case definition for AMR reporting | Role-play scenarios |
| 12:30-1:30 | Lunch | — | — |
| 1:30-2:30 | Introduction to UDARA AI | Platform overview; 4 channels (WhatsApp, Telegram, USSD, Voice); what it does; what it doesn't do | Live demo by trainer |
| 2:30-3:30 | USSD Basics | What is USSD? How to dial *384#; navigating menus; entering data; what to expect | Hands-on with feature phones |
| 3:30-4:00 | Day 1 Wrap-Up | Q&A; preview of Day 2; pre-assessment quiz | Quiz + discussion |

**Day 1 Learning Objectives:**
- CHWs can explain what antimicrobial resistance is in their own words
- CHWs understand why their reporting matters for the community
- CHWs can navigate the USSD menu *384# independently
- CHWs know what UDARA AI does and what to expect from Day 2

### 10.3 Day 2: Hands-On Technical Training

**Time: 9:00 AM - 4:00 PM (7 hours)**

| Time | Session | Content | Method |
|------|---------|---------|--------|
| 9:00-9:15 | Day 1 Recap | Quick quiz on AMR concepts; review USSD practice | Quiz |
| 9:15-10:30 | WhatsApp Bot Training | Adding UDARA AI number; sending first report; receiving AI response; drug photo capture; resistance check command; help commands | Live practice (each CHW on own phone) |
| 10:30-10:45 | Break | — | — |
| 10:45-12:00 | Telegram Bot Training | Adding @UdaraAIBot; sending reports via Telegram; voice message reporting; inline keyboard navigation; group commands | Live practice |
| 12:00-1:00 | Lunch | — | — |
| 1:00-2:00 | Voice Reporting | How to record a voice report; optimal recording technique (quiet, close to mic); Yoruba/English language support; playback and correction | Voice recording practice |
| 2:00-3:00 | Photo Capture for Drug Labels | How to photograph drug labels; lighting and framing tips; what the AI reads from labels; understanding resistance results | Camera practice with sample drug labels |
| 3:00-3:45 | Putting It All Together | CHWs complete 3 practice cases using their preferred channel | Supervised practice |
| 3:45-4:00 | Day 2 Wrap-Up | Address challenges; assign homework (send 1 real report before Day 3) | Discussion |

**Day 2 Learning Objectives:**
- CHWs can submit a complete AMR report via WhatsApp independently
- CHWs can submit a complete AMR report via Telegram independently
- CHWs can record a clear voice report and receive AI transcription
- CHWs can photograph a drug label and interpret the AI response
- Each CHW identifies their preferred reporting channel

### 10.4 Day 3: Practice Cases, Troubleshooting & Gamification

**Time: 9:00 AM - 3:00 PM (6 hours)**

| Time | Session | Content | Method |
|------|---------|---------|--------|
| 9:00-9:15 | Homework Review | Discuss real reports submitted; address issues | Group discussion |
| 9:15-10:30 | Practice Cases (5 Cases) | CHWs work through 5 simulated AMR cases of varying complexity: (1) simple UTI with resistance, (2) child with suspected typhoid, (3) wound infection, (4) multi-drug resistant TB suspicion, (5) community diarrhea outbreak | Individual then group debrief |
| 10:30-10:45 | Break | — | — |
| 10:45-11:30 | Troubleshooting Workshop | Common problems and solutions: no connectivity, app crashes, wrong data submitted, how to correct errors, who to call for help | Problem-solving scenarios |
| 11:30-12:30 | Gamification Walkthrough | How points work; badges you can earn; leaderboard; streaks; monthly challenges; rewards (airtime vouchers, certificates) | Live demonstration |
| 12:30-1:00 | Lunch | — | — |
| 1:00-2:00 | Competency Assessment | Each CHW completes 2 assessed cases independently (observed by trainer) | Individual assessment |
| 2:00-2:30 | Certificate Ceremony | CHWs receive "UDARA AI Certified Reporter" certificates; group photo | Celebration |
| 2:30-3:00 | Final Q&A & Pilot Launch | Address remaining questions; confirm pilot start date; exchange WhatsApp contacts | Open discussion |

**Day 3 Learning Objectives:**
- CHWs can handle 5 different AMR reporting scenarios independently
- CHWs know how to troubleshoot common technical problems
- CHWs understand the gamification system and are motivated to participate
- Each CHW passes the competency assessment (≥80% score)

### 10.5 Competency Assessment Rubric

```
┌────────────────────────────────────────────────────────────────────┐
│              CHW COMPETENCY ASSESSMENT RUBRIC                       │
├────────────────────────────┬──────────────────────────────────────┤
│ Criteria                   │ Scoring                               │
├────────────────────────────┼──────────────────────────────────────┤
│                            │ 4 = Excellent (no guidance needed)    │
│                            │ 3 = Good (minor guidance)             │
│ 1. Opens correct channel   │ 2 = Needs improvement                │
│    (WhatsApp/Telegram/     │ 1 = Failed (extensive guidance)       │
│    USSD) independently     │                                       │
│                            │ Target: All CHWs ≥3 on all criteria  │
├────────────────────────────┼──────────────────────────────────────┤
│ 2. Enters patient info     │                                       │
│    completely and          │                                       │
│    accurately              │                                       │
├────────────────────────────┼──────────────────────────────────────┤
│ 3. Submits drug/treatment  │                                       │
│    information             │                                       │
├────────────────────────────┼──────────────────────────────────────┤
│ 4. Reads and interprets    │                                       │
│    AI response correctly   │                                       │
├────────────────────────────┼──────────────────────────────────────┤
│ 5. Takes action on AI      │                                       │
│    guidance                │                                       │
├────────────────────────────┼──────────────────────────────────────┤
│ 6. Handles error/          │                                       │
│    unexpected response     │                                       │
├────────────────────────────┼──────────────────────────────────────┤
│ 7. Troubleshoots basic     │                                       │
│    connectivity issues     │                                       │
├────────────────────────────┼──────────────────────────────────────┤
│                            │ PASS: Average ≥3.0 (out of 4.0)     │
│                            │ RETRAIN: Average <3.0                │
│                            │                                       │
│ TOTAL SCORE: ___/28        │ RESULT: PASS / RETRAIN               │
└────────────────────────────┴──────────────────────────────────────┘
```

### 10.6 Training Materials List

| Material | Quantity | Language | Format |
|----------|----------|----------|--------|
| CHW Training Manual | 10 copies | English + Yoruba key terms | Printed, A5 booklet |
| Quick Reference Card (laminated) | 10 copies | Pictorial + English | Laminated A6 card |
| USSD Menu Map | 10 copies | Pictorial | Laminated A4 poster |
| WhatsApp Bot Command List | 10 copies | English | Printed A5 |
| Telegram Bot Command List | 10 copies | English | Printed A5 |
| Practice Case Sheets | 20 copies (5 cases × 4) | English | Printed A5 |
| Drug Label Samples (photos) | 30 photos | English | Printed on photo paper |
| Certificate of Completion | 10 copies | English | Printed A4 certificate |
| FAQ Sheet | 10 copies | English + Yoruba | Printed A5 |

### 10.7 Ongoing Training & Support

During the 12-week pilot, ongoing support is provided through:

| Support Channel | Frequency | Responsible | Content |
|----------------|-----------|-------------|---------|
| WhatsApp Group (all CHWs) | Daily (as needed) | Coordinator + Engineer | Questions, tips, encouragement |
| Weekly Check-in Calls | 1x/week (video) | Coordinator | Review issues, gather feedback |
| On-Site Supervisor Visit | 1x/week | Coordinator | In-person support, data quality check |
| Monthly Refresher Session | 1x/month | Training Lead + UI Faculty | New features, advanced training |
| Help Commands (in-app) | Anytime | System (automated) | Built-in help: /help, *384# help menu |

---

## 11. Comprehensive Budget

### 11.1 Master Budget Table

| # | Item | Category | Quantity | Unit Cost (USD) | Total (USD) | Notes |
|---|------|----------|----------|----------------|-------------|-------|
| **1** | **HARDWARE** | | | | | |
| 1.1 | Raspberry Pi 5 8GB | Hardware | 3 | $80.00 | $240.00 | Production units |
| 1.2 | 256GB microSD (SanDisk Extreme) | Hardware | 3 | $25.00 | $75.00 | A2 rated for Docker |
| 1.3 | USB 4G Modem (unlocked) | Hardware | 3 | $40.00 | $120.00 | Huawei E3372h-320 |
| 1.4 | Cases + Heatsinks | Hardware | 3 | $20.00 | $60.00 | Aluminium case |
| 1.5 | Power Supply (27W USB-C) | Hardware | 3 | $12.00 | $36.00 | Official RPi PSU |
| 1.6 | USB Flash Drive (32GB) | Hardware | 3 | $10.00 | $30.00 | Fallback sync |
| 1.7 | UPS Battery Backup (500VA) | Hardware | 3 | $30.00 | $90.00 | 4hr bridge power |
| 1.8 | Cable Lock + Surge Protector | Hardware | 3 | $15.00 | $45.00 | Physical security |
| 1.9 | Ethernet + HDMI cables | Hardware | 3 | $8.00 | $24.00 | Debug/backup |
| 1.10 | Spare RPi 5 8GB | Hardware | 1 | $80.00 | $80.00 | Replacement |
| 1.11 | Spare microSD (pre-imaged) | Hardware | 2 | $25.00 | $50.00 | Quick swap |
| 1.12 | Spare USB Modem | Hardware | 1 | $40.00 | $40.00 | Replacement |
| | | **Hardware Subtotal** | | | **$890.00** | |
| **2** | **CONNECTIVITY** | | | | | |
| 2.1 | SIM Cards | Connectivity | 3 | $5.00 | $15.00 | MTN/Airtel |
| 2.2 | Data Plans (3 months × $15/mo) | Connectivity | 3 | $45.00 | $135.00 | ≥10GB/month |
| 2.3 | Backup mobile hotspot data | Connectivity | 2 | $25.00 | $50.00 | Deployment team |
| | | **Connectivity Subtotal** | | | **$200.00** | |
| **3** | **CLOUD & INFRASTRUCTURE** | | | | | |
| 3.1 | Cloud hosting (12 weeks) | Cloud | 1 | $200.00 | $200.00 | AWS/GCP |
| 3.2 | Domain + SSL | Cloud | 1 | $30.00 | $30.00 | Annual |
| 3.3 | WhatsApp Business API | Cloud | 1 | $50.00 | $50.00 | 12 weeks |
| 3.4 | Monitoring (Grafana Cloud) | Cloud | 1 | $20.00 | $20.00 | 12 weeks |
| | | **Cloud Subtotal** | | | **$300.00** | |
| **4** | **PERSONNEL** | | | | | |
| 4.1 | Local Coordinator (3 months) | Personnel | 1 | $1,500.00 | $1,500.00 | Part-time, ₦200K/month |
| 4.2 | Deployment travel (international) | Personnel | 2 | $800.00 | $1,600.00 | 2 engineers × 2 trips |
| 4.3 | Deployment travel (local) | Personnel | 1 | $200.00 | $200.00 | Coordinator transport |
| | | **Personnel Subtotal** | | | **$3,300.00** | |
| **5** | **TRAINING** | | | | | |
| 5.1 | Training venue (PHC Jericho) | Training | 3 days | $30.00 | $90.00 | Cleaning + setup |
| 5.2 | Training materials (printing, laminating) | Training | 1 batch | $100.00 | $100.00 | 10 CHW kits |
| 5.3 | Meals (6 CHWs × 3 days) | Training | 18 meals | $8.00 | $144.00 | Breakfast + lunch |
| 5.4 | CHW transport reimbursement | Training | 6 CHWs × 3 days | $4.00 | $72.00 | Round trip |
| 5.5 | CHW training stipend | Training | 6 CHWs × 3 days | $6.00 | $108.00 | ₦5,000/day |
| 5.6 | Facilitator honorarium | Training | 2 | $50.00 | $100.00 | UI faculty |
| | | **Training Subtotal** | | | **$514.00** | |
| **6** | **LOGISTICS** | | | | | |
| 6.1 | Equipment shipping ( intl → Nigeria) | Logistics | 1 | $150.00 | $150.00 | DHL/FedEx |
| 6.2 | Local logistics (transport, misc) | Logistics | 12 weeks | $25.00 | $300.00 | Weekly avg |
| 6.3 | Communication (airtime for team) | Logistics | 12 weeks | $10.00 | $120.00 | Weekly avg |
| | | **Logistics Subtotal** | | | **$570.00** | |
| **7** | **PARTNERSHIP** | | | | | |
| 7.1 | UI/UCH partnership support | Partnership | 1 | $300.00 | $300.00 | Lab tests, data |
| 7.2 | Ethics review fees | Partnership | 1 | $50.00 | $50.00 | UI Ethics Committee |
| | | **Partnership Subtotal** | | | **$350.00** | |
| **8** | **GAMIFICATION REWARDS** | | | | | |
| 8.1 | Airtime vouchers (weekly rewards) | Rewards | 12 weeks | $20.00 | $240.00 | Top 3 CHWs |
| 8.2 | Certificates of Excellence | Rewards | 6 | $5.00 | $30.00 | End of pilot |
| | | **Rewards Subtotal** | | | **$270.00** | |
| **9** | **CONTINGENCY (15%)** | | | | | |
| | | Contingency | | | $1,094.40 | 15% of subtotal |
| | | | | | |
| | **GRAND TOTAL** | | | | **$8,388.40** | |
| | | | | | |

### 11.2 Budget Category Breakdown

```
┌────────────────────────────────────────────────────────────────────┐
│                    BUDGET BREAKDOWN BY CATEGORY                    │
├──────────────────────────────────────┬──────────────┬──────────────┤
│ Category                             │ Amount (USD) │ Percentage   │
├──────────────────────────────────────┼──────────────┼──────────────┤
│ Hardware                             │    $890.00   │    10.6%     │
│ ████████░░░░░░░░░░░░░░░░░░░░░░░░░░  │              │              │
├──────────────────────────────────────┼──────────────┼──────────────┤
│ Connectivity                         │    $200.00   │     2.4%     │
│ ██░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░  │              │              │
├──────────────────────────────────────┼──────────────┼──────────────┤
│ Cloud & Infrastructure               │    $300.00   │     3.6%     │
│ ███░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░  │              │              │
├──────────────────────────────────────┼──────────────┼──────────────┤
│ Personnel                            │  $3,300.00   │    39.3%     │
│ ████████████████████████████░░░░░░░  │              │              │
├──────────────────────────────────────┼──────────────┼──────────────┤
│ Training                             │    $514.00   │     6.1%     │
│ █████░░░░░░░░░░░░░░░░░░░░░░░░░░░░░  │              │              │
├──────────────────────────────────────┼──────────────┼──────────────┤
│ Logistics                            │    $570.00   │     6.8%     │
│ ██████░░░░░░░░░░░░░░░░░░░░░░░░░░░░  │              │              │
├──────────────────────────────────────┼──────────────┼──────────────┤
│ Partnership                           │    $350.00   │     4.2%     │
│ ████░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░  │              │              │
├──────────────────────────────────────┼──────────────┼──────────────┤
│ Gamification Rewards                 │    $270.00   │     3.2%     │
│ ███░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░  │              │              │
├──────────────────────────────────────┼──────────────┼──────────────┤
│ Contingency (15%)                    │  $1,094.40   │    13.0%     │
│ ███████████████░░░░░░░░░░░░░░░░░░░  │              │              │
├──────────────────────────────────────┼──────────────┼──────────────┤
│                                      │  $7,488.40   │    89.1%    │
│ GRAND TOTAL (with contingency)       │  $8,388.40   │   100.0%    │
└──────────────────────────────────────┴──────────────┴──────────────┘
```

### 11.3 Simplified Budget (for Presentations)

| Category | Amount |
|----------|--------|
| Hardware (3 sites) | $890 |
| Connectivity (3 months) | $200 |
| Cloud hosting (12 weeks) | $300 |
| Personnel (coordinator + travel) | $3,300 |
| Training (materials + stipends + venue) | $514 |
| Logistics & shipping | $570 |
| Partnership support | $350 |
| Gamification rewards | $270 |
| **Subtotal** | **$6,394** |
| Contingency (15%) | $994 |
| **TOTAL** | **$7,388** |

### 11.4 Per-Site Recurring Cost (Annual Projection)

| Cost Item | Monthly | Annual |
|-----------|---------|--------|
| Data plan | $15 | $180 |
| Cloud share (per site) | $25 | $300 |
| Coordinator time share | $50 | $600 |
| Gamification rewards | $20 | $240 |
| Maintenance & supplies | $10 | $120 |
| **Per-Site Annual** | **$120** | **$1,440** |
| **3-Site Annual** | **$360** | **$4,320** |

---

## 12. Risk Register

### 12.1 Risk Assessment Matrix

```
┌────────────────────────────────────────────────────────────────────┐
│                    RISK ASSESSMENT MATRIX                           │
│                                                                    │
│                    IMPACT                                           │
│              Low(1)  Med(2)  High(3)  Critical(4)                 │
│           ┌────────┬────────┬─────────┬───────────┐                │
│     High(4)│  MED   │  HIGH  │ CRITICAL│ CRITICAL  │               │
│     (3)    │  LOW   │  MED   │  HIGH   │ CRITICAL  │               │
│     Med(2) │  LOW   │  LOW   │  MED    │  HIGH     │               │
│     Low(1) │ ACCEPT │ ACCEPT │  LOW    │  MED      │               │
│           └────────┴────────┴─────────┴───────────┘                │
│                    LIKELIHOOD →                                     │
└────────────────────────────────────────────────────────────────────┘
```

### 12.2 Detailed Risk Register

| ID | Risk | Category | Likelihood (1-4) | Impact (1-4) | Risk Score | Mitigation Strategy | Owner |
|----|------|----------|-----------------|-------------|------------|---------------------|-------|
| R1 | **Sustained connectivity failure** (site offline >72 hours) | Technical | 3 | 3 | **9 (HIGH)** | Edge-first architecture processes all data locally; USB fallback for sync; dual-SIM modems; satellite backup consideration | Engineer |
| R2 | **CHW drop-off** (CHW stops using system) | Adoption | 3 | 4 | **12 (CRITICAL)** | Gamification incentives; weekly check-ins; identify and address barriers early; stipend continuation; USSD for feature phone users | Coordinator |
| R3 | **Data quality issues** (incomplete/inaccurate reports) | Data | 3 | 3 | **9 (HIGH)** | Real-time validation prompts; weekly supervisor spot-checks; automated completeness scoring; retraining sessions | Coordinator + PI |
| R4 | **Hardware damage** (RPi destroyed by power surge, water, theft) | Technical | 2 | 3 | **6 (MED)** | UPS + surge protector; waterproof case; cable lock; insurance; spare unit pre-positioned; 48-hour replacement SLA | Engineer |
| R5 | **Power supply failure** (extended grid outage, solar failure) | Technical | 3 | 2 | **6 (MED)** | UPS (4-hour bridge); solar supplement at Fiditi; RPi auto-shutdown at low battery; graceful shutdown preserves data | Engineer |
| R6 | **Regulatory hurdles** (NITDA/NDPR compliance issues) | Regulatory | 2 | 4 | **8 (HIGH)** | Early engagement with NITDA; data protection impact assessment; local legal review; NDPR compliance built-in (AES-256, Presidio) | Deployment Lead |
| R7 | **Partner disengagement** (UI/UCH reduces involvement) | Partnership | 2 | 3 | **6 (MED)** | Formal MoU with commitments; regular partnership meetings; mutual benefits clearly articulated; co-authorship on publications | PI |
| R8 | **Security incident** (equipment stolen, CHW threatened) | Security | 1 | 4 | **4 (MED)** | Cable locks; secure storage room; local community relationships; travel only during daylight; coordinator local knowledge | Coordinator |
| R9 | **Funding shortfall** (pilot costs exceed budget) | Financial | 2 | 3 | **6 (MED)** | 15% contingency; phased spending; grant backup (Gates Foundation, Wellcome Trust); scope reduction plan if needed | PI |
| R10 | **Political instability** (elections, strikes, unrest) | Political | 2 | 4 | **8 (HIGH)** | Site selection in stable region (Oyo); INFORM risk monitoring; 2-week buffer in timeline; evacuation plan | Deployment Lead |
| R11 | **Software bugs** (critical system failure in production) | Technical | 3 | 2 | **6 (MED)** | Staged rollout; automated testing; rapid patch deployment (<24h); feature flags for rollback | Engineer |
| R12 | **CHW data privacy breach** (patient data exposed) | Privacy | 1 | 4 | **4 (MED)** | AES-256 encryption at rest; Presidio PII redaction; no identifiable data on dashboard; NDPR compliance audit | Engineer + PI |
| R13 | **AMR data not actionable** (no patterns detected) | Scientific | 2 | 3 | **6 (MED)** | Sufficient sample size (target ≥50 reports/week); statistical analysis plan pre-defined; clinical validation with UCH; adjust case definition if needed | PI |
| R14 | **Language barriers** (CHWs not fluent in English) | Adoption | 3 | 2 | **6 (MED)** | Yoruba translation for key interfaces; voice reporting in local language; pictorial quick-reference cards; bilingual training materials | Training Lead |
| R15 | **Seasonal disease variation** (low AMR incidence during pilot) | Scientific | 2 | 2 | **4 (LOW)** | 12-week pilot spans dry→wet transition; case definition includes respiratory + GI + wound infections; adjust reporting threshold if needed | PI |

### 12.3 Risk Response Summary

| Risk Level | Count | Response |
|-----------|-------|----------|
| CRITICAL (12+) | 1 | R2 (CHW drop-off) — Daily monitoring, escalation protocol, backup adoption strategies |
| HIGH (8-11) | 3 | R1, R6, R10 — Weekly review, pre-prepared response plans, clear ownership |
| MEDIUM (4-7) | 9 | R4, R5, R7, R9, R11, R13, R14, R3, R15 — Monthly review, standard mitigation procedures |
| LOW (1-3) | 2 | R8, R12 — Accept with monitoring, standard operating procedures |

### 12.4 Risk Monitoring Cadence

```
┌────────────────────────────────────────────────────────────────────┐
│                    RISK MONITORING SCHEDULE                        │
├────────────────────────────────────────────────────────────────────┤
│                                                                    │
│  DAILY                                                              │
│  ├── System health check (automated)                               │
│  ├── CHW activity review (automated)                               │
│  └── WhatsApp group pulse check (coordinator)                      │
│                                                                    │
│  WEEKLY                                                             │
│  ├── Full risk register review (coordinator + engineer)            │
│  ├── CHW satisfaction pulse (1-question WhatsApp survey)          │
│  ├── Connectivity uptime report                                   │
│  └── Escalation of any new/changed risks to PI                    │
│                                                                    │
│  BI-WEEKLY                                                          │
│  ├── Steering committee risk review                                │
│  ├── Data quality assessment (completeness, accuracy)              │
│  ├── Partner engagement check (UI/UCH)                             │
│  └── Budget variance review                                        │
│                                                                    │
│  MONTHLY                                                            │
│  ├── Comprehensive risk reassessment (all 15 risks rescored)       │
│  ├── External risk scan (political, regulatory, security)         │
│  └── Risk response effectiveness review                            │
│                                                                    │
└────────────────────────────────────────────────────────────────────┘
```

---

## 13. Success Metrics & Go/No-Go Criteria

### 13.1 Primary Success Metrics

| # | Metric | Formula | Target | Measurement Method | Decision Impact |
|---|--------|---------|--------|-------------------|-----------------|
| 1 | CHW Activation Rate | CHWs with ≥1 report / Total CHWs × 100 | ≥80% by Week 6 | System analytics | Critical for H2 |
| 2 | System Uptime | Operational hours / Total hours × 100 | ≥99% weekly | Prometheus monitoring | Critical for H1 |
| 3 | AMR Cases Reported | Unique AMR reports / week across sites | ≥50 by Week 8 | System analytics | Critical for H3 |
| 4 | Data Completeness | Reports with all fields / Total reports × 100 | ≥90% | Automated check | Critical for H3 |
| 5 | CHW Satisfaction | Average satisfaction score (1-5 scale) | ≥4.0 | Endline survey | Supports H2 |
| 6 | Resistance Pattern Detection | New spatial/temporal resistance patterns found | ≥1 (before national lab) | Statistical analysis + UCH comparison | Critical for H3 |

### 13.2 Secondary Metrics

| # | Metric | Target | Purpose |
|---|--------|--------|---------|
| 7 | Reporting frequency | ≥3 reports/CHW/week | Sustained engagement |
| 8 | Sync success rate | ≥95% | Technical reliability |
| 9 | Inference latency (p95) | <5s text, <10s image, <3s voice | User experience |
| 10 | Offline resilience | 168+ hours continuous offline | Edge architecture validation |
| 11 | Guidance adherence | ≥70% CHWs follow AI guidance | Clinical value |
| 12 | Badge unlock rate | ≥50% CHWs earn ≥1 badge | Gamification effectiveness |
| 13 | USSD usage rate | ≥10% of reports via USSD | Channel diversity |
| 14 | Supervisor time | <30 min/week/site for data review | Operational efficiency |

### 13.3 Go/No-Go Decision Framework (Week 12)

```
┌────────────────────────────────────────────────────────────────────┐
│                 WEEK 12 GO/NO-GO DECISION                           │
├────────────────────────────┬───────────────────────────────────────┤
│                            │                                       │
│  ★ SCALE (Go to Phase 2)   │  ALL of these must be TRUE:          │
│                            │                                       │
│                            │  □ CHW Activation ≥ 70%               │
│                            │  □ System Uptime ≥ 95%                 │
│                            │  □ ≥30 AMR reports/week               │
│                            │  □ Data Completeness ≥ 85%            │
│                            │  □ CHW Satisfaction ≥ 3.5/5           │
│                            │  □ ≥1 resistance pattern detected      │
│                            │  □ No unresolved CRITICAL risks        │
│                            │                                       │
├────────────────────────────┼───────────────────────────────────────┤
│                            │                                       │
│  ◉ EXTEND (6 more weeks)   │  ANY of these TRUE:                  │
│                            │                                       │
│                            │  □ CHW Activation 50-69%              │
│                            │  □ System Uptime 85-94%               │
│                            │  □ 15-29 AMR reports/week             │
│                            │  □ Data Completeness 70-84%           │
│                            │  □ BUT trajectory is improving        │
│                            │                                       │
├────────────────────────────┼───────────────────────────────────────┤
│                            │                                       │
│  ✗ STOP (Do not proceed)   │  ANY of these TRUE:                  │
│                            │                                       │
│                            │  □ CHW Activation < 50%               │
│                            │  □ System Uptime < 85%                │
│                            │  □ <15 AMR reports/week               │
│                            │  □ CHW Satisfaction < 3.0/5           │
│                            │  □ CRITICAL risk unmitigated          │
│                            │  □ Safety or ethics incident          │
│                            │                                       │
└────────────────────────────┴───────────────────────────────────────┘
```

---

## 14. Governance & Decision Authority

### 14.1 Pilot Governance Structure

```
┌────────────────────────────────────────────────────────────────────┐
│                    PILOT GOVERNANCE STRUCTURE                       │
│                                                                    │
│  ┌──────────────────────────────────────────────────────────┐    │
│  │              STEERING COMMITTEE (Monthly)                  │    │
│  │  Members: PI, Deployment Lead, UI Partner, UCH Partner   │    │
│  │  Authority: Go/No-Go decisions, budget approval, scope     │    │
│  │            changes, partnership modifications             │    │
│  └───────────────────────┬──────────────────────────────────┘    │
│                          │                                        │
│  ┌───────────────────────┴──────────────────────────────────┐    │
│  │              PILOT LEAD / PI (Weekly)                     │    │
│  │  Authority: Day-to-day decisions, risk escalation,       │    │
│  │            technical prioritization, CHW engagement       │    │
│  └───┬────────────────────┬────────────────────┬────────────┘    │
│      │                    │                    │                  │
│  ┌───┴─────────┐  ┌───────┴────────┐  ┌───────┴────────┐       │
│  │ ENGINEERING │  │ FIELD OPS      │  │ CLINICAL        │       │
│  │ TEAM        │  │                │  │                 │       │
│  │             │  │ Local          │  │ UI/UCH          │       │
│  │ • Remote    │  │ Coordinator    │  │ Faculty         │       │
│  │   engineer  │  │ • Site visits  │  │ • Data          │       │
│  │ • System    │  │ • CHW support  │  │   validation    │       │
│  │   admin     │  │ • Logistics   │  │ • Ethics        │       │
│  │ • Monitoring│  │ • Reporting   │  │ • Lab           │       │
│  │             │  │                │  │   comparison    │       │
│  └─────────────┘  └────────────────┘  └─────────────────┘       │
│                                                                    │
└────────────────────────────────────────────────────────────────────┘
```

### 14.2 Decision Authority Matrix

| Decision Type | Example | Authority | Escalation |
|---------------|---------|-----------|------------|
| Technical patch deployment | Bug fix push | Engineer | Pilot Lead if risk > MED |
| CHW support action | Retraining session | Coordinator | Pilot Lead |
| Budget reallocation | Move $100 from contingency | Pilot Lead | Steering Committee if >$200 |
| Site addition/removal | Add 4th site | Steering Committee | Board |
| Scope change | Extend pilot by 2 weeks | Steering Committee | Board |
| Partnership modification | Change MoU terms | Steering Committee | Legal + Board |
| Safety/ethics incident | Data breach | Pilot Lead → Steering Committee immediately | Board + Regulatory |

### 14.3 Meeting Cadence

| Meeting | Frequency | Duration | Participants | Agenda |
|---------|-----------|----------|-------------|--------|
| Daily Standup | Daily (WhatsApp) | 5 min | Engineer + Coordinator | Health check, blockers |
| Weekly Ops Review | Weekly (video) | 30 min | Pilot Lead + Coordinator + Engineer | Metrics, issues, plan |
| Bi-weekly Clinical Review | Bi-weekly (video) | 45 min | Pilot Lead + UI/UCH Faculty | Data quality, clinical findings |
| Monthly Steering | Monthly (video) | 60 min | Steering Committee | Go/No-Go, budget, risks, strategic decisions |
| Emergency | As needed | Variable | Relevant parties | Critical issues only |

---

## 15. Post-Pilot Transition Plan

### 15.1 Transition Options

| Option | Description | Condition | Timeline |
|--------|-------------|-----------|----------|
| **Scale to Phase 2** | Expand to 10 sites in Oyo State; integrate with state PHC board | Go decision at Week 12 | 3-month planning → 6-month Phase 2 |
| **Geographic Expansion** | Replicate pilot in Kenya or Rwanda (Reserve countries) | Strong Go + partner readiness | 6-month planning → pilot in new country |
| **Product Transition** | Hand over system to Oyo State PHC Board for sustained operation | Go + local capacity demonstrated | 3-month transition period |
| **Sunset** | Decommission system; archive data; publish findings | No-Go decision | 1-month decommission + 3-month analysis |

### 15.2 Data Handover

- All raw data (SQLite exports) provided to UI/UCH research team
- Anonymized aggregate data provided to Oyo State PHC Board
- Full dataset available to NCDC upon request
- Publication plan: 2 peer-reviewed papers (methodology + findings)
- Open data commitment: Anonymized dataset published on Zenodo within 12 months

### 15.3 Equipment Disposition

| Item | If Scale | If Sunset |
|------|----------|-----------|
| RPi 5 units | Retained and expanded | Donated to UI Department of Medical Microbiology |
| microSD cards | Retained | Securely wiped and donated |
| 4G modems | Retained | Donated to health posts |
| UPS units | Retained | Donated to health posts |
| Spare equipment | Moved to Phase 2 pool | Donated to UI |

---

## 16. Appendices

### Appendix A: Country Selection Detailed Scores

| Criterion | Nigeria | Kenya | Rwanda | Ghana | Senegal | South Africa | Ethiopia | Tanzania |
|-----------|---------|-------|--------|-------|---------|-------------|----------|----------|
| AMR Burden | 5 | 4 | 3 | 3 | 3 | 4 | 3 | 3 |
| Mobile Penetration | 3 | 4 | 3 | 4 | 3 | 5 | 3 | 3 |
| Internet Reliability | 3 | 4 | 4 | 3 | 3 | 4 | 3 | 2 |
| Regulatory Environment | 4 | 4 | 5 | 3 | 3 | 4 | 3 | 3 |
| CHW Network | 5 | 4 | 4 | 4 | 3 | 3 | 3 | 3 |
| Partner Availability | 4 | 4 | 4 | 3 | 3 | 4 | 3 | 2 |
| Political Stability | 4 | 4 | 5 | 4 | 3 | 3 | 3 | 4 |
| **TOTAL** | **28** | **28** | **28** | **24** | **21** | **27** | **21** | **20** |

### Appendix B: AMR Priority Pathogens in Nigeria

| Pathogen | Resistance Type | Community Prevalence | Priority |
|----------|----------------|---------------------|----------|
| *E. coli* | 3rd-gen cephalosporin resistance | 45-60% | CRITICAL |
| *Klebsiella pneumoniae* | 3rd-gen cephalosporin resistance | 50-65% | CRITICAL |
| *Staphylococcus aureus* | Methicillin resistance (MRSA) | 30-45% | HIGH |
| *Salmonella* spp. | Fluoroquinolone resistance | 35-50% | HIGH |
| *Acinetobacter baumannii* | Carbapenem resistance | 60-75% | CRITICAL |
| *Pseudomonas aeruginosa* | Carbapenem resistance | 40-55% | CRITICAL |
| *Neisseria gonorrhoeae* | 3rd-gen cephalosporin resistance | 15-25% | HIGH |
| *Enterococcus faecium* | Vancomycin resistance | 10-20% | HIGH |

### Appendix C: Nigerian AMR Timeline

| Year | Milestone |
|------|-----------|
| 2017 | Nigeria AMR Situation Analysis published |
| 2017 | National AMR Action Plan 2017-2022 launched |
| 2018 | NCDC establishes AMR surveillance coordinating unit |
| 2019 | Nigeria joins WHO GLASS |
| 2020 | COVID-19 disrupts AMR surveillance activities |
| 2021 | NARSP (Nigeria AMR Reference Surveillance Platform) established |
| 2022 | National AMR prevalence survey completed |
| 2023 | NDPA (Data Protection Act) enacted |
| 2024 | Updated AMR Action Plan 2023-2027 under development |
| 2025 | **UDARA AI pilot begins (THIS DOCUMENT)** |

### Appendix D: Equipment Supplier Options

| Component | Supplier A | Supplier B | Supplier C |
|-----------|-----------|-----------|-----------|
| RPi 5 8GB | Raspberry Pi Foundation (UK) | Amazon US | SparkFun |
| microSD 256GB | Amazon (SanDisk) | AliExpress (Samsung) | Local market (various) |
| USB 4G Modem | Amazon (Huawei) | Local electronics market (Ibadan) | Jumia Nigeria |
| UPS 500VA | Amazon (APC) | Local market (Genius/Blue Gate) | Jumia Nigeria |
| RPi Case | Amazon (Pimoroni) | AliExpress (generic) | Local 3D print (UI engineering) |

### Appendix E: Pre-Pilot Timeline (Pre-Week 1)

| Week | Activity | Owner | Status |
|------|----------|-------|--------|
| T-12 | MoU draft shared with UI/UCH | PI | ⬜ Pending |
| T-10 | Ethics submission to UI/UCH Ethics Committee | PI | ⬜ Pending |
| T-8 | Hardware procurement initiated | Engineer | ⬜ Pending |
| T-6 | Local coordinator recruitment begins | PI | ⬜ Pending |
| T-6 | Training materials development begins | Training Lead | ⬜ Pending |
| T-4 | Local coordinator hired | PI | ⬜ Pending |
| T-4 | Hardware received and tested | Engineer | ⬜ Pending |
| T-3 | SD cards imaged and tested | Engineer | ⬜ Pending |
| T-2 | Training materials finalized | Training Lead | ⬜ Pending |
| T-2 | Site visits and facility clearances | Coordinator | ⬜ Pending |
| T-1 | Final equipment pack and ship | Engineer | ⬜ Pending |
| T-1 | Travel logistics finalized | Deployment Lead | ⬜ Pending |
| T-0 | Team arrives in Ibadan | All | ⬜ Pending |

### Appendix F: Contact Directory

| Role | Name | Organization | Email | Phone |
|------|------|-------------|-------|-------|
| Principal Investigator | [TBD] | UDARA AI | [email] | [phone] |
| Deployment Lead | [TBD] | UDARA AI | [email] | [phone] |
| Remote Engineer | [TBD] | UDARA AI | [email] | [phone] |
| Local Coordinator | [TBD] | Independent | [email] | [phone] |
| UI Partner (Faculty) | [TBD] | University of Ibadan | [email] | [phone] |
| UCH Partner (Microbiologist) | [TBD] | UCH | [email] | [phone] |
| Oyo State PHC Board Liaison | [TBD] | Oyo State Government | [email] | [phone] |

---

## Document Revision History

| Version | Date | Author | Changes |
|---------|------|--------|---------|
| 1.0 | 2024-11-01 | Deployment Team | Initial draft |
| 1.1 | 2024-11-15 | PI | Country selection criteria finalized |
| 1.2 | 2024-12-01 | Deployment Lead | Site specifications added |
| 2.0 | 2024-12-15 | Full Team | Major revision: Nigeria selected, sites confirmed |
| 2.1 | 2025-01-05 | Engineer | Hardware and software sections updated |
| 2.2 | 2025-01-10 | Training Lead | CHW training program detailed |
| 2.3 | 2025-01-12 | PI | Budget finalized, risk register updated |
| 2.4 | 2025-01-15 | PI | Final review, appendices completed |

---

*This document is the authoritative source for UDARA AI pilot operations in Oyo State, Nigeria. All team members are expected to read this document in full before pilot deployment begins.*
