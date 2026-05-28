# Monitoring & Evaluation — KPIs, Success Metrics & M&E Framework

**UDARA AI — Antimicrobial Resistance Surveillance Platform**
**Document Version:** 2.3.0
**Date:** 2025-01-15
**Author:** UDARA AI M&E Team
**Classification:** Internal — M&E Operations
**Review Cycle:** Monthly during pilot; quarterly post-pilot

---

## Table of Contents

1. [Executive Summary](#1-executive-summary)
2. [M&E Framework — Logical Model](#2-me-framework--logical-model)
3. [Theory of Change](#3-theory-of-change)
4. [KPI Category 1: Adoption KPIs](#4-kpi-category-1-adoption-kpis)
5. [KPI Category 2: Data Quality KPIs](#5-kpi-category-2-data-quality-kpis)
6. [KPI Category 3: Technical KPIs](#6-kpi-category-3-technical-kpis)
7. [KPI Category 4: Public Health KPIs](#7-kpi-category-4-public-health-kpis)
8. [KPI Category 5: Gamification KPIs](#8-kpi-category-5-gamification-kpis)
9. [KPI Category 6: Financial & Efficiency KPIs](#9-kpi-category-6-financial--efficiency-kpis)
10. [Composite Scorecards](#10-composite-scorecards)
11. [Data Collection Methods](#11-data-collection-methods)
12. [Reporting Cadence & Templates](#12-reporting-cadence--templates)
13. [Grafana Dashboard Layout](#13-grafana-dashboard-layout)
14. [Evaluation Design](#14-evaluation-design)
15. [Data Quality Assurance](#15-data-quality-assurance)
16. [Ethical Considerations in M&E](#16-ethical-considerations-in-me)
17. [Appendices](#17-appendices)

---

## 1. Executive Summary

This Monitoring & Evaluation (M&E) framework establishes the measurement architecture for the UDARA AI pilot deployment in Oyo State, Nigeria. The framework is designed to answer three fundamental questions:

1. **Is the technology working?** (Technical performance)
2. **Are people using it?** (Adoption and engagement)
3. **Is it making a difference?** (Public health impact)

The framework spans **6 KPI categories** with **42 individual KPIs**, each defined by formula, data source, collection frequency, target, and threshold. KPIs are organized from proximal (adoption) to distal (public health impact), aligned with a rigorous logical model that traces inputs through activities, outputs, outcomes, and impact.

```
┌────────────────────────────────────────────────────────────────────┐
│                    M&E FRAMEWORK AT A GLANCE                       │
├────────────────────────────────────────────────────────────────────┤
│                                                                    │
│  INPUTS            ACTIVITIES         OUTPUTS                      │
│  ───────           ──────────         ───────                      │
│  • $8,388 budget   • Deploy RPi 5    • 3 sites operational        │
│  • 6 CHWs          • Train CHWs       • 6 CHWs trained             │
│  • 3 RPi 5 units   • Run pilot       • N reports submitted        │
│  • Partners        • Collect data     • N resistance patterns      │
│                                                                    │
│  OUTCOMES           IMPACT                                          │
│  ────────           ──────                                         │
│  • 80% CHW         • Earlier AMR                                   │
│    adoption         detection                                      │
│  • Improved data    • Better treatment                              │
│    quality          decisions                                      │
│  • Clinically       • Reduced AMR                                  │
│    actionable       mortality in                                   │
│    surveillance     catchment area                                │
│                                                                    │
│  ┌──────────────────────────────────────────────────────────┐    │
│  │ 42 KPIs | 6 Categories | Daily→Monthly→Quarterly        │    │
│  │ Grafana Dashboards | Prometheus Metrics | CHW Surveys   │    │
│  │ Automated Reporting | Monthly M&E Reports                │    │
│  └──────────────────────────────────────────────────────────┘    │
│                                                                    │
└────────────────────────────────────────────────────────────────────┘
```

---

## 2. M&E Framework — Logical Model

### 2.1 Inputs → Activities → Outputs → Outcomes → Impact

The UDARA AI M&E framework follows the standard logical model used in global health program evaluation. Each level is defined with specific, measurable indicators.

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    LOGICAL MODEL — UDARA AI                                  │
├─────────────┬───────────────────┬──────────────┬──────────────┬─────────────┤
│   INPUTS    │    ACTIVITIES     │   OUTPUTS    │  OUTCOMES    │   IMPACT    │
│             │                   │              │  (Short/Med) │  (Long-term)│
├─────────────┼───────────────────┼──────────────┼──────────────┼─────────────┤
│             │                   │              │              │             │
│ • Hardware  │ • Site preparation│ • 3 RPi 5    │ • ≥80% CHW   │ • Earlier   │
│   (RPi 5,   │   & installation │   units      │   adoption   │   AMR       │
│   modem,    │ • CHW training    │   deployed   │   rate       │   pattern   │
│   UPS)      │   (3-day)         │ • 6 CHWs     │ • ≥90% data  │   detection │
│             │ • Pilot           │   certified  │   completeness│             │
│ • Software  │   execution       │ • ≥500 total │ • ≥50 AMR    │ • Improved  │
│   (edge     │   (12 weeks)      │   AMR        │   reports/   │   treatment │
│   runtime,  │ • Daily support   │   reports    │   week       │   outcomes  │
│   models,   │   & monitoring    │ • ≥1         │ • ≥70%       │             │
│   monitoring│ • Weekly          │   resistance │   guidance   │ • Reduced   │
│   stack)    │   supervisor      │   pattern    │   adherence  │   AMR       │
│             │   reviews         │   detected   │ • ≥95% sync  │   mortality  │
│ • Personnel │ • Bi-weekly       │ • Resistance │   success    │             │
│   (engineer,│   data quality    │   map for    │   rate       │ • Strengthen│
│   coord,    │   checks          │   Oyo State  │ • ≥99% API   │   surveillance│
│   PI)       │ • Monthly M&E     │ • Dashboard  │   uptime     │   systems   │
│             │   reporting       │   operational│ • ≥4.0/5.0   │             │
│ • Partners  │ • Cloud sync &    │ • Gamification│   CHW       │ • Policy    │
│   (UI, UCH, │   dashboard       │   system     │   satisfaction│  influence │
│   PHC Board)│   management      │   active     │ • Gamification│             │
│             │ • Weekly partner  │ • Training   │   drives     │             │
│ • Budget    │   meetings        │   materials  │   engagement│             │
│   ($8,388)  │ • Endline         │   delivered  │ • Supervisors│             │
│             │   evaluation      │ • M&E        │   find data  │             │
│             │                   │   reports    │   useful     │             │
├─────────────┴───────────────────┴──────────────┴──────────────┴─────────────┤
│                                                                             │
│  ASSUMPTIONS:                                                               │
│  1. CHWs have access to smartphones or feature phones                       │
│  2. Internet connectivity is available ≥30% of the time                     │
│  3. CHWs are willing to learn and adopt new technology                      │
│  4. Health authorities are receptive to digital AMR data                    │
│  5. Edge AI models perform adequately on RPi 5 hardware                     │
│                                                                             │
│  EXTERNAL FACTORS:                                                          │
│  1. Political stability in Oyo State during pilot period                    │
│  2. No major disease outbreaks that overwhelm health posts                  │
│  3. Telecom infrastructure remains functional                                │
│  4. Power supply remains available (grid/solar)                             │
│  5. National AMR surveillance continues (for comparison)                   │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 2.2 Indicator Mapping Table

| Level | Indicator | Type | Measurement | Frequency |
|-------|-----------|------|-------------|-----------|
| Input | Budget disbursed on schedule | Process | Financial tracking | Monthly |
| Input | Hardware delivered to sites | Process | Inventory check | Once (Week 1) |
| Input | CHWs enrolled | Process | Registration system | Once (Week 3) |
| Activity | Training sessions completed | Process | Training logs | Once (Week 3) |
| Activity | CHWs passing competency test | Process | Assessment scores | Once (Week 3) |
| Activity | Supervisory visits conducted | Process | Visit logs | Weekly |
| Activity | System uptime maintained | Process | Prometheus metrics | Daily |
| Output | AMR reports submitted | Output | System analytics | Daily |
| Output | Unique pathogens reported | Output | System analytics | Weekly |
| Output | Resistance patterns detected | Output | Statistical analysis | Monthly |
| Output | Gamification points distributed | Output | System analytics | Daily |
| Outcome | CHW adoption rate | Outcome | Composite metric | Weekly |
| Outcome | Data quality score | Outcome | Automated + manual | Weekly |
| Outcome | Guidance adherence rate | Outcome | System + supervisor | Monthly |
| Outcome | CHW satisfaction | Outcome | Survey | Monthly + Endline |
| Impact | Earlier resistance detection | Impact | Comparison with national lab | Monthly |
| Impact | Treatment decision improvement | Impact | Supervisor assessment | Quarterly |
| Impact | AMR surveillance data used by health authority | Impact | Policy tracking | Post-pilot |

### 2.3 Data Flow for M&E

```
┌────────────────────────────────────────────────────────────────────┐
│                    M&E DATA FLOW                                     │
├────────────────────────────────────────────────────────────────────┤
│                                                                    │
│  DATA SOURCES                                                      │
│  ────────────                                                      │
│                                                                    │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐        │
│  │Prometheus│  │  CHW     │  │Supervisor│  │  Clinical│        │
│  │ Metrics  │  │ Surveys  │  │ Spot     │  │ Lab Data │        │
│  │(system)  │  │(monthly) │  │ Checks   │  │(UCH)     │        │
│  └────┬─────┘  └────┬─────┘  └────┬─────┘  └────┬─────┘        │
│       │              │              │              │              │
│       ▼              ▼              ▼              ▼              │
│  ┌──────────────────────────────────────────────────────────┐    │
│  │                    M&E DATABASE                           │    │
│  │              (PostgreSQL + TimescaleDB)                   │    │
│  │                                                           │    │
│  │  ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────────┐   │    │
│  │  │ Technical│ │ Adoption │ │   Data   │ │ Public   │   │    │
│  │  │ Metrics  │ │ Metrics  │ │ Quality  │ │ Health   │   │    │
│  │  │ Table    │ │ Table    │ │ Table    │ │ Table    │   │    │
│  │  └──────────┘ └──────────┘ └──────────┘ └──────────┘   │    │
│  │                                                           │    │
│  │  ┌──────────┐ ┌──────────┐ ┌──────────┐                 │    │
│  │  │Gamificat.│ │Financial │ │ Survey   │                 │    │
│  │  │ Metrics  │ │ Metrics  │ │ Responses│                 │    │
│  │  │ Table    │ │ Table    │ │ Table    │                 │    │
│  │  └──────────┘ └──────────┘ └──────────┘                 │    │
│  └──────────────────────┬───────────────────────────────────┘    │
│                          │                                        │
│       ┌──────────────────┼──────────────────┐                    │
│       ▼                  ▼                  ▼                    │
│  ┌──────────┐     ┌──────────┐     ┌──────────────┐           │
│  │ Grafana  │     │ Monthly  │     │ Quarterly    │           │
│  │ Dashboard│     │ M&E      │     │ Funder       │           │
│  │ (real-   │     │ Report   │     │ Impact       │           │
│  │  time)   │     │ (PDF)    │     │ Report       │           │
│  └──────────┘     └──────────┘     └──────────────┘           │
│                                                                    │
└────────────────────────────────────────────────────────────────────┘
```

---

## 3. Theory of Change

### 3.1 Impact Pathway

```
┌────────────────────────────────────────────────────────────────────┐
│              THEORY OF CHANGE — UDARA AI                           │
├────────────────────────────────────────────────────────────────────┤
│                                                                    │
│  IF we deploy edge AI AMR surveillance at health posts             │
│  AND train CHWs to report through accessible channels             │
│  AND provide gamification incentives for consistent reporting     │
│                                                                    │
│  THEN CHWs will adopt the system and submit high-quality AMR data │
│  AND the edge AI will process reports locally (even offline)      │
│  AND resistance patterns will be detected in near-real-time        │
│                                                                    │
│  AND IF health authorities and clinicians access the dashboard    │
│                                                                    │
│  THEN treatment decisions will improve based on local resistance  │
│  AND outbreaks of resistant infections will be detected earlier   │
│  AND inappropriate antibiotic prescriptions will decrease          │
│                                                                    │
│  ULTIMATELY reducing AMR-attributable morbidity and mortality     │
│  in the catchment population                                      │
│                                                                    │
│  ─────────────────────────────────────────────────────────        │
│  ASSUMPTIONS UNDERLYING THIS THEORY:                              │
│                                                                    │
│  1. CHWs are present and willing at health posts                   │
│  2. RPi 5 hardware is reliable in field conditions                │
│  3. AI models accurately identify resistance-related information  │
│  4. Health authorities will act on the data                       │
│  5. The reporting channels (WhatsApp, Telegram, USSD, voice)      │
│     are accessible to the target CHW population                    │
│  6. Gamification provides sufficient incentive for sustained use  │
│  7. The political and regulatory environment remains stable        │
│                                                                    │
└────────────────────────────────────────────────────────────────────┘
```

### 3.2 Outcome Chain with Timeframes

| Outcome | Level | Timeframe | Measurement Point |
|---------|-------|-----------|-----------------|
| CHWs trained and competent | Output | Week 3 | Training assessment |
| CHWs actively reporting | Short-term outcome | Week 4-6 | Adoption rate ≥80% |
| Consistent reporting habits established | Short-term outcome | Week 6-12 | Retention rate ≥85% |
| High-quality AMR data collected | Short-term outcome | Week 4-12 | Completeness ≥90% |
| Resistance patterns identified | Medium-term outcome | Week 8-12 | ≥1 pattern detected |
| Clinicians using dashboard | Medium-term outcome | Week 10-12 | Dashboard logins tracked |
| Treatment guidance followed | Medium-term outcome | Week 8-12 | Guidance adherence ≥70% |
| Earlier AMR detection vs. status quo | Medium-term impact | Week 12+ | Comparison with national lab |
| Health policy influenced | Long-term impact | Post-pilot (6-12 months) | Policy tracking |
| AMR mortality reduced | Long-term impact | Post-pilot (12-24 months) | Epidemiological study |

---

## 4. KPI Category 1: Adoption KPIs

### 4.1 Overview

Adoption KPIs measure whether CHWs are actually using UDARA AI, how frequently, through which channels, and whether they continue using it over time. These are the most proximal indicators of pilot success.

```
┌────────────────────────────────────────────────────────────────────┐
│                    ADOPTION KPIs — DASHBOARD VIEW                   │
├────────────────────────────────────────────────────────────────────┤
│                                                                    │
│  CHW Activation Rate ████████████████████░░░░  80%     ▲ +5% WoW  │
│  Weekly Active Rate  ██████████████████████░░  85%     ▲ +2% WoW  │
│  Reporting Freq.    ██████░░░░░░░░░░░░░░░░░░░  3.2/wk  ▲ +0.3 WoW │
│  Channel: WhatsApp  ████████████████████░░░░░░  65%     ───       │
│  Channel: Telegram  ████░░░░░░░░░░░░░░░░░░░░░░  20%     ▲ +2%     │
│  Channel: USSD      ███░░░░░░░░░░░░░░░░░░░░░░░░  10%     ───       │
│  Channel: Voice     █░░░░░░░░░░░░░░░░░░░░░░░░░░   5%     ▼ -1%     │
│  Retention (MoM)    ████████████████████████░░  90%     ▲ +3%     │
│                                                                    │
└────────────────────────────────────────────────────────────────────┘
```

### 4.2 ADOPT-1: CHW Activation Rate

| Attribute | Detail |
|-----------|--------|
| **Full Name** | CHW Activation Rate |
| **Code** | ADOPT-1 |
| **Definition** | Percentage of registered CHWs who have submitted at least one AMR report through UDARA AI |
| **Formula** | `CHWs with ≥1 report / Total registered CHWs × 100` |
| **Target** | ≥80% by Week 6 |
| **Stretch Target** | ≥90% by Week 10 |
| **Data Source** | UDARA AI system analytics (user_activity table) |
| **Collection Frequency** | Daily (automated), reported weekly |
| **Responsible** | Product Team |
| **Disaggregation** | By site, by channel, by CHW age/gender |
| **Threshold (Red)** | <50% — urgent intervention needed |
| **Threshold (Yellow)** | 50-79% — improvement plan needed |
| **Threshold (Green)** | ≥80% — on track |

**Tracking Table:**

| Week | Registered CHWs | CHWs with ≥1 Report | Activation Rate | Status |
|------|----------------|--------------------|-----------------| -------|
| 3 | 6 | 6 | 100%* | 🟢 *Training effect |
| 4 | 6 | 5 | 83% | 🟢 |
| 5 | 6 | 5 | 83% | 🟢 |
| 6 | 6 | 5 | 83% | 🟢 |
| 7 | 6 | 6 | 100% | 🟢 |
| 8 | 6 | 6 | 100% | 🟢 |
| 9 | 6 | 6 | 100% | 🟢 |
| 10 | 6 | 6 | 100% | 🟢 |
| 11 | 6 | 6 | 100% | 🟢 |
| 12 | 6 | 6 | 100% | 🟢 |

*Note: 100% in Week 3 reflects mandatory first report during training. True adoption measured from Week 4 onwards.

### 4.3 ADOPT-2: Weekly Active Reporters

| Attribute | Detail |
|-----------|--------|
| **Full Name** | Weekly Active Reporters Rate |
| **Code** | ADOPT-2 |
| **Definition** | Percentage of registered CHWs who submitted ≥1 report in the current week |
| **Formula** | `CHWs with ≥1 report this week / Total registered CHWs × 100` |
| **Target** | ≥60% sustained from Week 6 to Week 12 |
| **Stretch Target** | ≥75% sustained |
| **Data Source** | System analytics (weekly aggregation) |
| **Collection Frequency** | Weekly |
| **Responsible** | Product Team |
| **Disaggregation** | By site, by channel |
| **Threshold (Red)** | <40% |
| **Threshold (Yellow)** | 40-59% |
| **Threshold (Green)** | ≥60% |

**Trend Visualization:**

```
Weekly Active Reporters (%)
100% ┤
 90% ┤                                                 ●
 80% ┤                                           ●
 70% ┤
 60% ┤          ●●●  TARGET LINE ──────────────────────
 50% ┤     ●
 40% ┤
 30% ┤
 20% ┤
 10% ┤
  0% ┼──┬──┬──┬──┬──┬──┬──┬──┬──┬──┬──┬──
     W3 W4 W5 W6 W7 W8 W9 W10 W11 W12
```

### 4.4 ADOPT-3: Channel Distribution

| Attribute | Detail |
|-----------|--------|
| **Full Name** | Reporting Channel Distribution |
| **Code** | ADOPT-3 |
| **Definition** | Percentage of reports submitted through each of the 4 channels (WhatsApp, Telegram, USSD, Voice) |
| **Formula** | `Reports via channel X / Total reports × 100` for each channel |
| **Target** | No single channel >80% (diversity indicates accessibility) |
| **Expected Distribution** | WhatsApp 50-60%, Telegram 15-25%, USSD 10-20%, Voice 5-10% |
| **Data Source** | System analytics (report_channel column) |
| **Collection Frequency** | Weekly |
| **Responsible** | Product Team |
| **Disaggregation** | By site (urban vs rural expected to differ) |
| **Alert** | Any single channel <5% may indicate access barrier |

**Expected Channel Distribution by Site:**

| Channel | PHC Jericho (Urban) | Awe (Semi-rural) | Fiditi (Rural) | Overall |
|---------|--------------------|--------------------|-----------------|---------|
| WhatsApp | 55% | 50% | 40% | 50% |
| Telegram | 25% | 20% | 15% | 20% |
| USSD | 10% | 20% | 35% | 18% |
| Voice | 10% | 10% | 10% | 10% |

### 4.5 ADOPT-4: Retention Rate (Month-over-Month)

| Attribute | Detail |
|-----------|--------|
| **Full Name** | CHW Retention Rate (MoM) |
| **Code** | ADOPT-4 |
| **Definition** | Percentage of CHWs who were active in the previous month AND are active in the current month |
| **Formula** | `CHWs active this month AND last month / CHWs active last month × 100` |
| **Target** | ≥85% |
| **Data Source** | System analytics (monthly user activity aggregation) |
| **Collection Frequency** | Monthly |
| **Responsible** | Product Team |
| **Threshold (Red)** | <70% |
| **Threshold (Yellow)** | 70-84% |
| **Threshold (Green)** | ≥85% |

### 4.6 ADOPT-5: Reporting Frequency

| Attribute | Detail |
|-----------|--------|
| **Full Name** | Average Reports per CHW per Week |
| **Code** | ADOPT-5 |
| **Definition** | Average number of AMR reports submitted per active CHW per week |
| **Formula** | `Total reports this week / Active CHWs this week` |
| **Target** | ≥3 reports/CHW/week by Week 8 |
| **Stretch Target** | ≥5 reports/CHW/week |
| **Data Source** | System analytics |
| **Collection Frequency** | Weekly |
| **Responsible** | Product Team |
| **Disaggregation** | By site, by CHW |
| **Note** | Expect variation by patient volume: Jericho CHWs report more than Fiditi |

### 4.7 ADOPT-6: Time to First Report

| Attribute | Detail |
|-----------|--------|
| **Full Name** | Time to First Report (Post-Training) |
| **Code** | ADOPT-6 |
| **Definition** | Average time between training completion and first unsupervised report submission |
| **Formula** | `Date of first real report - Date of training completion` (in days) |
| **Target** | ≤2 days |
| **Data Source** | System analytics + training records |
| **Collection Frequency** | Once (after training) |
| **Responsible** | Training Lead |
| **Note** | Shorter time = higher confidence from training |

### 4.8 ADOPT-7: Channel Switching Rate

| Attribute | Detail |
|-----------|--------|
| **Full Name** | Multi-Channel Usage Rate |
| **Code** | ADOPT-7 |
| **Definition** | Percentage of CHWs who have used ≥2 different channels to submit reports |
| **Formula** | `CHWs using ≥2 channels / Total active CHWs × 100` |
| **Target** | ≥30% |
| **Data Source** | System analytics (unique channel count per CHW) |
| **Collection Frequency** | Monthly |
| **Responsible** | Product Team |
| **Interpretation** | High multi-channel usage indicates CHWs understand all options and switch based on context |

### 4.9 ADOPT-8: Net Promoter Score (CHW)

| Attribute | Detail |
|-----------|--------|
| **Full Name** | CHW Net Promoter Score (NPS) |
| **Code** | ADOPT-8 |
| **Definition** | Likelihood of CHW recommending UDARA AI to a colleague CHW (0-10 scale) |
| **Formula** | `% Promoters (9-10) - % Detractors (0-6)` |
| **Target** | NPS ≥30 |
| **Data Source** | Monthly WhatsApp survey (1 question) |
| **Collection Frequency** | Monthly |
| **Responsible** | M&E Lead |

### 4.10 Adoption KPI Summary Table

| Code | KPI | Formula | Target | Frequency | Data Source |
|------|-----|---------|--------|-----------|-------------|
| ADOPT-1 | CHW Activation Rate | ≥1 report CHWs / Total × 100 | ≥80% by W6 | Weekly | System |
| ADOPT-2 | Weekly Active Reporters | Active this week / Total × 100 | ≥60% | Weekly | System |
| ADOPT-3 | Channel Distribution | Reports per channel / Total × 100 | No channel >80% | Weekly | System |
| ADOPT-4 | Retention Rate (MoM) | Active both months / Active prev × 100 | ≥85% | Monthly | System |
| ADOPT-5 | Reporting Frequency | Total reports / Active CHWs | ≥3/week | Weekly | System |
| ADOPT-6 | Time to First Report | First report date - Training date | ≤2 days | Once | System + Training |
| ADOPT-7 | Multi-Channel Usage | ≥2 channels CHWs / Total × 100 | ≥30% | Monthly | System |
| ADOPT-8 | CHW NPS | % Promoters - % Detractors | ≥30 | Monthly | Survey |

---

## 5. KPI Category 2: Data Quality KPIs

### 5.1 Overview

Data quality KPIs measure whether the AMR reports submitted by CHWs are complete, accurate, and timely. Poor data quality undermines the public health value of the entire system.

```
┌────────────────────────────────────────────────────────────────────┐
│              DATA QUALITY DIMENSIONS                                 │
├────────────────────────────────────────────────────────────────────┤
│                                                                    │
│              ┌──────────────┐                                       │
│              │  DATA        │                                       │
│              │  QUALITY     │                                       │
│              └──────┬───────┘                                       │
│                     │                                               │
│        ┌────────────┼────────────┬──────────┐                      │
│        ▼            ▼            ▼          ▼                      │
│  ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────────┐            │
│  │COMPLETE- │ │ ACCURATE │ │ TIMELY   │ │CONSIST-  │            │
│  │NESS      │ │          │ │          │ │ENT       │            │
│  │          │ │          │ │          │ │          │            │
│  │All fields│ │Correct   │ │Submitted │ │Same case │            │
│  │populated │ │values    │ │within    │ │not       │            │
│  │          │ │verified  │ │4 hours   │ │duplicated│            │
│  └──────────┘ └──────────┘ └──────────┘ └──────────┘            │
│                                                                    │
│  Target: ≥90% completeness, ≥85% accuracy, <4h timeliness         │
└────────────────────────────────────────────────────────────────────┘
```

### 5.2 DQ-1: Report Completeness

| Attribute | Detail |
|-----------|--------|
| **Full Name** | Report Completeness Rate |
| **Code** | DQ-1 |
| **Definition** | Percentage of submitted reports that contain all required fields without missing data |
| **Formula** | `Reports with all required fields / Total reports × 100` |
| **Required Fields** | (1) Patient age group, (2) Symptoms, (3) Suspected infection, (4) Antibiotic prescribed/taken, (5) Outcome |
| **Target** | ≥90% |
| **Stretch Target** | ≥95% |
| **Data Source** | Automated field validation in edge-runtime |
| **Collection Frequency** | Daily (automated), reported weekly |
| **Responsible** | Product Team |
| **Disaggregation** | By site, by channel, by CHW, by field |
| **Threshold (Red)** | <75% |
| **Threshold (Yellow)** | 75-89% |
| **Threshold (Green)** | ≥90% |

**Field-Level Completeness Tracking:**

| Required Field | Completeness Target | Urban (Jericho) | Semi-rural (Awe) | Rural (Fiditi) |
|---------------|-------------------|-----------------|-------------------|----------------|
| Patient age group | 95% | 98% | 92% | 88% |
| Symptoms | 95% | 96% | 94% | 90% |
| Suspected infection type | 90% | 95% | 88% | 82% |
| Antibiotic prescribed | 85% | 90% | 85% | 78% |
| Treatment outcome | 80% | 85% | 78% | 70% |
| **Overall Completeness** | **90%** | **93%** | **87%** | **82%** |

### 5.3 DQ-2: Report Accuracy

| Attribute | Detail |
|-----------|--------|
| **Full Name** | Report Accuracy Rate |
| **Code** | DQ-2 |
| **Definition** | Percentage of spot-checked reports confirmed as accurate by supervisor |
| **Formula** | `Supervisor-verified correct reports / Total spot-checked × 100` |
| **Target** | ≥85% |
| **Data Source** | Supervisor spot-checks (5 random reports per site per week = 15/week) |
| **Collection Frequency** | Weekly |
| **Responsible** | Local Coordinator (supervisor) |
| **Accuracy Criteria** | (1) Patient age matches records, (2) Symptoms accurately described, (3) Antibiotic correctly identified, (4) Classification appropriate |
| **Threshold (Red)** | <70% |
| **Threshold (Yellow)** | 70-84% |
| **Threshold (Green)** | ≥85% |

**Spot-Check Protocol:**

```
┌────────────────────────────────────────────────────────────────────┐
│              WEEKLY SPOT-CHECK PROTOCOL                             │
├────────────────────────────────────────────────────────────────────┤
│                                                                    │
│  1. Every MONDAY, system randomly selects 5 reports per site       │
│     from the previous week (15 total)                              │
│                                                                    │
│  2. Coordinator visits each site during the week                   │
│     Tuesday: Jericho (5 reports)                                   │
│     Wednesday: Awe (5 reports)                                     │
│     Thursday: Fiditi (5 reports)                                   │
│                                                                    │
│  3. For each report, coordinator verifies:                        │
│     □ Does the patient exist in facility records?                  │
│     □ Is the age group correct?                                   │
│     □ Are the symptoms consistent with the diagnosis?             │
│     □ Is the antibiotic correctly identified?                      │
│     □ Is the outcome recorded correctly?                           │
│                                                                    │
│  4. Scoring per report:                                            │
│     • All 5 correct = ACCURATE                                    │
│     • 3-4 correct = PARTIALLY ACCURATE                            │
│     • 0-2 correct = INACCURATE                                    │
│                                                                    │
│  5. Results logged in supervisor dashboard                         │
│  6. If accuracy <70% at any site: schedule retraining              │
│  7. Monthly accuracy aggregate reported to steering committee      │
│                                                                    │
└────────────────────────────────────────────────────────────────────┘
```

### 5.4 DQ-3: Reporting Timeliness

| Attribute | Detail |
|-----------|--------|
| **Full Name** | Average Reporting Timeliness |
| **Code** | DQ-3 |
| **Definition** | Average time between patient encounter and report submission |
| **Formula** | `AVG(Report submission timestamp - Patient encounter timestamp)` |
| **Target** | <4 hours average |
| **Stretch Target** | <2 hours average |
| **Data Source** | System analytics (report timestamps) |
| **Collection Frequency** | Daily (automated), reported weekly |
| **Responsible** | Product Team |
| **Disaggregation** | By site, by channel, by time of day |
| **Threshold (Red)** | >12 hours |
| **Threshold (Yellow)** | 4-12 hours |
| **Threshold (Green)** | <4 hours |

**Timeliness Distribution (Expected):**

```
Reports by Timeliness Bucket (%)
50% ┤  ██
    │  ██ ██
40% ┤  ██ ██ ██
    │  ██ ██ ██
30% ┤  ██ ██ ██ ██
    │  ██ ██ ██ ██
20% ┤  ██ ██ ██ ██
    │  ██ ██ ██ ██ ██
10% ┤  ██ ██ ██ ██ ██ ██
    │  ██ ██ ██ ██ ██ ██ ██
 0% ┼──┴──┴──┴──┴──┴──┴──┴──
    <1h 1-2h 2-4h 4-8h 8-12h >12h

    ██████ = Immediate (same visit)
```

### 5.5 DQ-4: Duplicate Report Rate

| Attribute | Detail |
|-----------|--------|
| **Full Name** | Duplicate Report Rate |
| **Code** | DQ-4 |
| **Definition** | Percentage of reports identified as duplicates (same patient, same visit, submitted twice) |
| **Formula** | `Duplicate reports / Total reports × 100` |
| **Target** | <5% |
| **Data Source** | Automated deduplication in edge-runtime (based on patient ID + date + site) |
| **Collection Frequency** | Weekly |
| **Responsible** | Product Team |
| **Note** | Duplicates are flagged but NOT deleted (retained for analysis of why duplicates occur) |

### 5.6 DQ-5: Classification Accuracy (AI Model)

| Attribute | Detail |
|-----------|--------|
| **Full Name** | AI Classification Accuracy |
| **Code** | DQ-5 |
| **Definition** | Percentage of AI-generated case classifications that match expert clinical review |
| **Formula** | `AI classifications matching expert review / Total reviewed × 100` |
| **Target** | ≥80% |
| **Data Source** | Monthly expert review of 30 random AI-classified reports |
| **Collection Frequency** | Monthly |
| **Responsible** | UCH Clinical Microbiologist |
| **Note** | This measures AI quality separately from CHW data entry quality |

### 5.7 Data Quality KPI Summary Table

| Code | KPI | Formula | Target | Frequency | Data Source |
|------|-----|---------|--------|-----------|-------------|
| DQ-1 | Completeness Rate | Complete reports / Total × 100 | ≥90% | Weekly | System (automated) |
| DQ-2 | Accuracy Rate | Verified correct / Spot-checked × 100 | ≥85% | Weekly | Supervisor |
| DQ-3 | Timeliness | AVG(submission - encounter) | <4 hours | Weekly | System (automated) |
| DQ-4 | Duplicate Rate | Duplicates / Total × 100 | <5% | Weekly | System (automated) |
| DQ-5 | AI Classification Accuracy | AI matches expert / Reviewed × 100 | ≥80% | Monthly | Expert review |

---

## 6. KPI Category 3: Technical KPIs

### 6.1 Overview

Technical KPIs measure the reliability, performance, and resilience of the UDARA AI system infrastructure. These are critical for validating Hypothesis H1 (Edge AI works reliably).

```
┌────────────────────────────────────────────────────────────────────┐
│              TECHNICAL KPIs — SYSTEM HEALTH                         │
├────────────────────────────────────────────────────────────────────┤
│                                                                    │
│  API Uptime         ██████████████████████░░░  99.7%   ▲ Target   │
│  Edge Inference     ████░░░░░░░░░░░░░░░░░░░░░░  2.3s    ▲ Target   │
│  Sync Success       ████████████████████████░░  97.2%   ▲ Target   │
│  Offline Resilience ██████████████████████████  192h    ▲ Target   │
│  Memory Usage       ██████████████░░░░░░░░░░░░  62%     ▲ Target   │
│  USB Fallback       █░░░░░░░░░░░░░░░░░░░░░░░░░  0.3/mo  ▲ Target   │
│  Error Rate         ██░░░░░░░░░░░░░░░░░░░░░░░░░  0.1%    ▲ Target   │
│                                                                    │
└────────────────────────────────────────────────────────────────────┘
```

### 6.2 TECH-1: API Uptime Percentage

| Attribute | Detail |
|-----------|--------|
| **Full Name** | API Uptime Percentage |
| **Code** | TECH-1 |
| **Definition** | Percentage of time the UDARA AI edge API is available and responding to health checks |
| **Formula** | `(Total minutes - Downtime minutes) / Total minutes × 100` |
| **Target** | ≥99.5% (allows ~21 minutes/week downtime for maintenance) |
| **Data Source** | Prometheus health check probes (every 10 seconds) |
| **Collection Frequency** | Continuous (10s intervals) |
| **Responsible** | Remote Engineer |
| **Disaggregation** | Per site, per container |
| **Downtime Definition** | Any period where health check fails for ≥60 seconds consecutively |
| **Exclusions** | Planned maintenance windows (pre-approved) |
| **Threshold (Red)** | <95% |
| **Threshold (Yellow)** | 95-99% |
| **Threshold (Green)** | ≥99.5% |

**Uptime Monitoring Configuration:**

```yaml
# Prometheus scrape config
scrape_configs:
  - job_name: 'udara-edge-health'
    scrape_interval: 10s
    static_configs:
      - targets:
          - 'jericho.udara.local:8080/health'
          - 'awe.udara.local:8080/health'
          - 'fiditi.udara.local:8080/health'

# Alert rules
groups:
  - name: udara-alerts
    rules:
      - alert: EdgeAPIDown
        expr: up{job="udara-edge-health"} == 0
        for: 1m
        labels:
          severity: critical
        annotations:
          summary: "UDARA Edge API down at {{ $labels.instance }}"
          
      - alert: HighLatency
        expr: histogram_quantile(0.95, rate(http_request_duration_seconds_bucket[5m])) > 5
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "P95 latency > 5s at {{ $labels.instance }}"
```

### 6.3 TECH-2: Edge Inference Latency

| Attribute | Detail |
|-----------|--------|
| **Full Name** | Edge Inference Latency (P95) |
| **Code** | TECH-2 |
| **Definition** | 95th percentile time for AI model inference from request to response |
| **Formula** | `P95(request_complete - request_start)` measured per inference type |
| **Targets by Inference Type:** | |
| - Text (NER + Sentiment) | <5 seconds |
| - Image (OCR drug label) | <10 seconds |
| - Voice (Whisper transcription) | <3 seconds |
| - Full AMR prediction | <8 seconds |
| **Data Source** | Prometheus histogram metrics (model_server_inference_duration_seconds) |
| **Collection Frequency** | Continuous (per request) |
| **Responsible** | Remote Engineer |
| **Disaggregation** | Per site, per model, per input type |
| **Threshold (Red)** | >15s (any type) |
| **Threshold (Yellow)** | 8-15s |
| **Threshold (Green)** | <8s |

**Latency Benchmark Matrix:**

| Input Type | Model(s) Used | Target P50 | Target P95 | Target P99 |
|-----------|--------------|-----------|-----------|-----------|
| Text report (English) | NER + Sentiment + AMR Pred | 2s | 4s | 7s |
| Text report (Yoruba) | NER + Sentiment + AMR Pred | 3s | 5s | 9s |
| Voice (English, 30s clip) | Whisper + NER + AMR Pred | 4s | 6s | 10s |
| Voice (Yoruba, 30s clip) | Whisper + NER + AMR Pred | 5s | 8s | 12s |
| Drug label photo | OCR + NER + AMR Pred | 3s | 7s | 12s |
| Multi-photo report | OCR + NER + AMR Pred (×2) | 5s | 10s | 15s |

### 6.4 TECH-3: Sync Success Rate

| Attribute | Detail |
|-----------|--------|
| **Full Name** | Cloud Sync Success Rate |
| **Code** | TECH-3 |
| **Definition** | Percentage of data sync attempts that complete successfully |
| **Formula** | `Successful syncs / Total sync attempts × 100` |
| **Target** | ≥95% |
| **Data Source** | Sync agent logs (sync_agent_success_total, sync_agent_failure_total) |
| **Collection Frequency** | Per sync attempt (~every 5 minutes when online) |
| **Responsible** | Remote Engineer |
| **Disaggregation** | Per site (expect Fiditi to have lower rate due to connectivity) |
| **Note** | Failed syncs are retried with exponential backoff (5s → 10s → 30s → 60s → 300s) |

**Sync Performance by Site (Expected):**

| Site | Connectivity | Expected Sync Rate | Expected Sync Method |
|------|-------------|-------------------|---------------------|
| PHC Jericho | 4G (85-95% uptime) | 98% | Cloud (automatic) |
| Awe | 3G (50-70% uptime) | 92% | Cloud (queued) |
| Fiditi | 2G (30-50% uptime) | 75% | Cloud + USB fallback |

### 6.5 TECH-4: Offline Resilience

| Attribute | Detail |
|-----------|--------|
| **Full Name** | Maximum Continuous Offline Operation |
| **Code** | TECH-4 |
| **Definition** | Maximum duration the system has operated without any cloud connectivity while maintaining full functionality |
| **Formula** | `MAX(duration of continuous offline operation)` in hours |
| **Target** | ≥168 hours (7 days) |
| **Stretch Target** | ≥336 hours (14 days) |
| **Data Source** | Sync agent logs (offline_duration_seconds) |
| **Collection Frequency** | Event-based (when connectivity returns) |
| **Responsible** | Remote Engineer |
| **Disaggregation** | Per site |
| **Note** | This is a PASSIVE metric — we observe offline periods, not create them |

### 6.6 TECH-5: RPi Resource Utilization

| Attribute | Detail |
|-----------|--------|
| **Full Name** | Raspberry Pi Resource Utilization |
| **Code** | TECH-5 |
| **Definition** | Average CPU, RAM, storage, and temperature utilization across pilot period |
| **Targets:** | |
| - CPU Usage (average) | <60% |
| - RAM Usage (average) | <80% (of 8GB) |
| - Storage Used | <50% (of 256GB) |
| - CPU Temperature | <75°C |
| - Swap Usage | <100MB |
| **Data Source** | Prometheus node_exporter metrics |
| **Collection Frequency** | Every 10 seconds |
| **Responsible** | Remote Engineer |

**Resource Utilization by Container (Expected):**

| Container | CPU (avg) | RAM (avg) | Storage |
|-----------|-----------|-----------|---------|
| Host OS | 2% | 512 MB | 8 GB |
| edge-runtime | 8% | 800 MB | 2 GB |
| model-server | 15% | 3.5 GB | 4 GB |
| monitoring | 3% | 400 MB | 2 GB |
| sync-agent | 2% | 200 MB | 500 MB |
| **Total** | **30%** | **5.4 GB (68%)** | **16.5 GB (6%)** |
| **Headroom** | **70%** | **2.6 GB (32%)** | **239.5 GB (94%)** |

### 6.7 TECH-6: USB Fallback Events

| Attribute | Detail |
|-----------|--------|
| **Full Name** | USB Fallback Sync Events |
| **Code** | TECH-6 |
| **Definition** | Number of times USB physical sync was required due to extended offline period |
| **Target** | <2 events/month per device |
| **Data Source** | Sync agent USB event logs |
| **Collection Frequency** | Event-based |
| **Responsible** | Coordinator |
| **Note** | USB fallback is expected at Fiditi but should be rare at other sites |

### 6.8 TECH-7: Error Rate

| Attribute | Detail |
|-----------|--------|
| **Full Name** | System Error Rate |
| **Code** | TECH-7 |
| **Definition** | Percentage of API requests that result in an error (HTTP 5xx) |
| **Formula** | `HTTP 5xx responses / Total API requests × 100` |
| **Target** | <0.5% |
| **Data Source** | Edge runtime access logs |
| **Collection Frequency** | Per request (aggregated hourly) |
| **Responsible** | Remote Engineer |
| **Threshold (Red)** | >5% |
| **Threshold (Yellow)** | 0.5-5% |
| **Threshold (Green)** | <0.5% |

### 6.9 TECH-8: Model Loading Time

| Attribute | Detail |
|-----------|--------|
| **Full Name** | Cold Start Model Loading Time |
| **Code** | TECH-8 |
| **Definition** | Time from RPi boot to all 4 AI models fully loaded and ready for inference |
| **Target** | <180 seconds |
| **Data Source** | Model server startup logs |
| **Collection Frequency** | Per boot event |
| **Responsible** | Remote Engineer |

### 6.10 Technical KPI Summary Table

| Code | KPI | Formula | Target | Frequency | Data Source |
|------|-----|---------|--------|-----------|-------------|
| TECH-1 | API Uptime % | Uptime / Total × 100 | ≥99.5% | Continuous | Prometheus |
| TECH-2 | Inference Latency (P95) | P95(request time) | <5s text, <10s image | Per request | Prometheus |
| TECH-3 | Sync Success Rate | Successful / Attempts × 100 | ≥95% | Per sync | Sync agent logs |
| TECH-4 | Offline Resilience | MAX(offline duration) | ≥168 hours | Event-based | Sync agent logs |
| TECH-5 | Resource Utilization | CPU/RAM/Storage/Temperature | <80% each | Continuous | node_exporter |
| TECH-6 | USB Fallback Events | COUNT(USB sync events) | <2/month/device | Event-based | Sync agent logs |
| TECH-7 | Error Rate | 5xx errors / Total × 100 | <0.5% | Per request | Access logs |
| TECH-8 | Model Load Time | Boot to models ready | <180s | Per boot | Startup logs |

---

## 7. KPI Category 4: Public Health KPIs

### 7.1 Overview

Public Health KPIs measure the ultimate value of UDARA AI — whether the data collected leads to actionable insights that can improve AMR surveillance and treatment outcomes.

```
┌────────────────────────────────────────────────────────────────────┐
│              PUBLIC HEALTH KPIs — IMPACT INDICATORS                 │
├────────────────────────────────────────────────────────────────────┤
│                                                                    │
│  New Resistance Patterns    ████████████████░░░░░░░░  3 detected   │
│  Time to Detection          ██████████████████████░░  5 days      │
│  Districts Flagged          ████████████████████░░░░  2 districts  │
│  Guidance Adherence         ██████████████████████░░  72%          │
│  Reports Actionable         ████████████████████████  88%          │
│                                                                    │
│  ★ KEY: "Detected Ciprofloxacin resistance cluster in             │
│          Awe 3 weeks before national lab"                          │
│                                                                    │
└────────────────────────────────────────────────────────────────────┘
```

### 7.2 PH-1: New Resistance Patterns Detected

| Attribute | Detail |
|-----------|--------|
| **Full Name** | New Resistance Patterns Detected |
| **Code** | PH-1 |
| **Definition** | Cumulative count of statistically significant new AMR resistance patterns identified through UDARA AI that were not previously documented in the area |
| **Formula** | `COUNT(resistance patterns passing significance threshold)` where significance = spatial clustering (p < 0.05) or temporal trend (p < 0.05) |
| **Target** | ≥1 pattern detected by Week 12 |
| **Stretch Target** | ≥3 patterns |
| **Data Source** | Statistical analysis engine (Bayesian spatial model) + UCH lab comparison |
| **Collection Frequency** | Weekly analysis, monthly formal review |
| **Responsible** | PI + UCH Clinical Microbiologist |
| **Validation** | All patterns must be validated against UCH microbiology lab data before counting |

### 7.3 PH-2: Time to Detection

| Attribute | Detail |
|-----------|--------|
| **Full Name** | Time to Resistance Pattern Detection |
| **Code** | PH-2 |
| **Definition** | Number of days from the first report indicating a new resistance pattern to the date the pattern is formally flagged by the system |
| **Formula** | `Date flagged - Date of first contributing report` (in days) |
| **Target** | <7 days |
| **Stretch Target** | <3 days |
| **Comparison Metric** | `Time to detection (UDARA AI) vs. Time to detection (national lab)` |
| **Target Advantage** | UDARA AI detects ≥7 days before national lab |
| **Data Source** | UDARA AI detection timestamps + NCDC/NARSP report dates |
| **Collection Frequency** | Per detection event |
| **Responsible** | PI |
| **Note** | This is the "killer metric" — demonstrating that UDARA AI provides EARLIER detection than conventional surveillance |

### 7.4 PH-3: Districts Flagged for Elevated Resistance

| Attribute | Detail |
|-----------|--------|
| **Full Name** | Districts Flagged for Elevated Resistance |
| **Code** | PH-3 |
| **Definition** | Number of districts/LGAs where resistance rates exceed regional baselines by ≥2 standard deviations |
| **Formula** | `COUNT(districts WHERE resistance_rate > regional_mean + 2*std)` |
| **Target** | ≥1 district flagged (validates spatial analysis capability) |
| **Data Source** | Resistance dashboard + spatial analysis engine |
| **Collection Frequency** | Monthly |
| **Responsible** | PI |

### 7.5 PH-4: Guidance Adherence Rate

| Attribute | Detail |
|-----------|--------|
| **Full Name** | AI Guidance Adherence Rate |
| **Code** | PH-4 |
| **Definition** | Percentage of times CHWs followed the AI-generated treatment guidance when guidance was provided |
| **Formula** | `Reports where CHW action matches AI guidance / Reports with AI guidance × 100` |
| **Target** | ≥70% |
| **Data Source** | System analytics (guidance_given flag + CHW_response field) |
| **Collection Frequency** | Weekly |
| **Responsible** | M&E Lead |
| **Note** | Adherence is not mandatory — CHWs may have valid clinical reasons for deviating. Low adherence triggers investigation, not punitive action. |

### 7.6 PH-5: Reports Classified as Actionable

| Attribute | Detail |
|-----------|--------|
| **Full Name** | Reports with Actionable AMR Data |
| **Code** | PH-5 |
| **Definition** | Percentage of submitted reports that contain sufficient data for AMR surveillance analysis (i.e., include antibiotic information and outcome) |
| **Formula** | `Reports with antibiotic + outcome data / Total reports × 100` |
| **Target** | ≥85% |
| **Data Source** | System analytics (automated field check) |
| **Collection Frequency** | Weekly |
| **Responsible** | Product Team |

### 7.7 PH-6: Pathogen Coverage

| Attribute | Detail |
|-----------|--------|
| **Full Name** | Pathogen/Condition Coverage Breadth |
| **Code** | PH-6 |
| **Definition** | Number of distinct infection types represented in the AMR report database |
| **Formula** | `COUNT(DISTINCT infection_type) from reports` |
| **Target** | ≥10 distinct types (UTI, RTI, wound, GI, typhoid, SSTI, sepsis, otitis, meningitis, other) |
| **Data Source** | System analytics |
| **Collection Frequency** | Monthly |
| **Responsible** | M&E Lead |
| **Interpretation** | Breadth indicates comprehensive surveillance, not just single-condition focus |

**Expected Infection Type Distribution:**

| Infection Type | Expected Frequency | AMR Relevance |
|---------------|-------------------|---------------|
| Urinary Tract Infection (UTI) | 25% | HIGH — E. coli fluoroquinolone resistance |
| Respiratory Tract Infection (RTI) | 20% | HIGH — S. pneumoniae macrolide resistance |
| Gastrointestinal (GI) | 15% | MEDIUM — Shigella, Salmonella resistance |
| Wound/Skin & Soft Tissue (SSTI) | 15% | HIGH — S. aureus (MRSA) |
| Typhoid/Enteric Fever | 8% | HIGH — S. Typhi fluoroquinolone resistance |
| Other/Unspecified | 17% | VARIABLE |
| **Total** | **100%** | |

### 7.8 PH-7: Antibiotic Resistance Rates (Specific Pathogens)

| Attribute | Detail |
|-----------|--------|
| **Full Name** | Measured Antibiotic Resistance Rates |
| **Code** | PH-7a through PH-7f |
| **Definition** | Resistance rate for specific pathogen-antibiotic combinations as reported through UDARA AI |
| **Formula** | `Reports indicating resistance to antibiotic X for pathogen Y / Total reports for pathogen Y × 100` |
| **Target Pathogens:** | E. coli (3GC-R), S. aureus (MRSA), Salmonella (FLU-R), K. pneumoniae (3GC-R) |
| **Data Source** | UDARA AI reports + AI classification |
| **Collection Frequency** | Monthly |
| **Validation** | Comparison with UCH lab data for concordance |

**Resistance Rate Tracking Table:**

| Pathogen-Antibiotic | Month 1 | Month 2 | Month 3 | National Average | UDARA vs. National |
|--------------------|---------|---------|---------|-----------------|-------------------|
| E. coli — Ciprofloxacin | — | — | — | 45% | TBD |
| E. coli — Ceftriaxone | — | — | — | 52% | TBD |
| S. aureus — Methicillin | — | — | — | 35% | TBD |
| Salmonella — Ciprofloxacin | — | — | — | 38% | TBD |
| K. pneumoniae — Ceftriaxone | — | — | — | 58% | TBD |

### 7.9 Public Health KPI Summary Table

| Code | KPI | Formula | Target | Frequency | Data Source |
|------|-----|---------|--------|-----------|-------------|
| PH-1 | New Resistance Patterns | COUNT(significant patterns) | ≥1 | Monthly | Statistical engine + UCH |
| PH-2 | Time to Detection | Date flagged - First report date | <7 days | Per event | System + NCDC |
| PH-3 | Districts Flagged | COUNT(high-resistance districts) | ≥1 | Monthly | Spatial analysis |
| PH-4 | Guidance Adherence | CHW followed guidance / Guidance given × 100 | ≥70% | Weekly | System |
| PH-5 | Actionable Reports | Reports with antibiotic + outcome / Total × 100 | ≥85% | Weekly | System |
| PH-6 | Pathogen Coverage | COUNT(DISTINCT infection types) | ≥10 | Monthly | System |
| PH-7 | Resistance Rates | Resistant / Total per pathogen-antibiotic | Track | Monthly | System + UCH lab |

---

## 8. KPI Category 5: Gamification KPIs

### 8.1 Overview

Gamification KPIs measure the effectiveness of the incentive system in driving CHW engagement and consistent reporting.

```
┌────────────────────────────────────────────────────────────────────┐
│              GAMIFICATION SYSTEM — REWARD STRUCTURE                  │
├────────────────────────────────────────────────────────────────────┤
│                                                                    │
│  POINTS                                                            │
│  ──────                                                            │
│  • First Report:           50 points                               │
│  • Each AMR Report:        10 points                               │
│  • Complete Report:        +5 bonus points                          │
│  • Photo Included:         +5 bonus points                          │
│  • Voice Report:           +3 bonus points                          │
│  • Weekly Challenge Win:   100 points                              │
│  • Streak Bonus (7 days):  50 points                               │
│  • Streak Bonus (30 days): 200 points                              │
│                                                                    │
│  BADGES                                                            │
│  ─────                                                            │
│  🏥 "First Step"      — Submit first report                        │
│  📸 "Shutterbug"       — Include photo in report                   │
│  🎤 "Voice Champion"   — Submit 10 voice reports                   │
│  📅 "Consistent"       — 7-day reporting streak                    │
│  🔥 "On Fire"          — 30-day reporting streak                   │
│  🏆 "Top Reporter"     — Most reports in a month                   │
│  🎯 "Accuracy Ace"     — 100% completeness for 20 reports          │
│  🌍 "Community Hero"   — Refer another CHW to the platform          │
│                                                                    │
│  WEEKLY CHALLENGES                                                   │
│  ─────────────────                                                   │
│  • "Report 5 cases this week" — All participants get 10 pts each  │
│  • "Photo week" — Bonus for reports with photos                    │
│  • "Try a new channel" — Bonus for using a different channel       │
│                                                                    │
│  REWARDS                                                            │
│  ───────                                                           │
│  • 500 points:  ₦500 airtime voucher                              │
│  • 1000 points: ₦1,000 airtime voucher                             │
│  • 2000 points: Certificate of Excellence + ₦2,000 voucher        │
│  • Top 3 leaderboard: Bonus airtime each month                     │
│                                                                    │
└────────────────────────────────────────────────────────────────────┘
```

### 8.2 GAM-1: Total Points Distributed

| Attribute | Detail |
|-----------|--------|
| **Full Name** | Total Gamification Points Distributed |
| **Code** | GAM-1 |
| **Definition** | Cumulative points earned by all CHWs through the gamification system |
| **Formula** | `SUM(points_earned) across all CHWs` |
| **Target** | ≥2,000 points by Week 12 (reflects sustained engagement) |
| **Data Source** | Gamification engine database |
| **Collection Frequency** | Daily (automated) |
| **Responsible** | Product Team |

### 8.3 GAM-2: Badge Unlock Rate

| Attribute | Detail |
|-----------|--------|
| **Full Name** | Badge Unlock Rate |
| **Code** | GAM-2 |
| **Definition** | Percentage of active CHWs who have earned at least 1 badge |
| **Formula** | `CHWs with ≥1 badge / Active CHWs × 100` |
| **Target** | ≥80% by Week 8 |
| **Data Source** | Gamification engine |
| **Collection Frequency** | Weekly |
| **Responsible** | Product Team |
| **Expected Badge Distribution:** | |

| Badge | Expected % Unlock (Week 12) |
|-------|---------------------------|
| 🏥 First Step | 100% (all active CHWs) |
| 📸 Shutterbug | 60% |
| 🎤 Voice Champion | 30% |
| 📅 Consistent (7-day) | 70% |
| 🔥 On Fire (30-day) | 20% |
| 🏆 Top Reporter | 17% (1 per month) |
| 🎯 Accuracy Ace | 40% |
| 🌍 Community Hero | 10% |

### 8.4 GAM-3: Leaderboard Check Rate

| Attribute | Detail |
|-----------|--------|
| **Full Name** | Weekly Leaderboard Check Rate |
| **Code** | GAM-3 |
| **Definition** | Percentage of active CHWs who check the leaderboard at least once per week |
| **Formula** | `CHWs checking leaderboard / Active CHWs × 100` |
| **Target** | ≥50% weekly |
| **Data Source** | Gamification engine (leaderboard_view events) |
| **Collection Frequency** | Weekly |
| **Responsible** | Product Team |

### 8.5 GAM-4: Streak Statistics

| Attribute | Detail |
|-----------|--------|
| **Full Name** | CHW Reporting Streak Statistics |
| **Code** | GAM-4 |
| **Definition** | Distribution of maximum reporting streaks (consecutive days with ≥1 report) across all CHWs |
| **Metrics:** | |
| - Max streak (any CHW) | Target: ≥30 days |
| - Average max streak | Target: ≥14 days |
| - CHWs with ≥7-day streak | Target: ≥50% |
| **Data Source** | Gamification engine |
| **Collection Frequency** | Weekly |
| **Responsible** | Product Team |

### 8.6 GAM-5: Reward Redemption Rate

| Attribute | Detail |
|-----------|--------|
| **Full Name** | Reward Redemption Rate |
| **Code** | GAM-5 |
| **Definition** | Percentage of earned rewards (airtime vouchers) that are redeemed |
| **Formula** | `Redeemed rewards / Total earned rewards × 100` |
| **Target** | ≥90% |
| **Data Source** | Gamification engine (redemption log) |
| **Collection Frequency** | Weekly |
| **Responsible** | Coordinator (processes redemptions) |
| **Note** | Low redemption rate may indicate rewards are not valued or process is too complex |

### 8.7 GAM-6: Gamification Impact on Reporting

| Attribute | Detail |
|-----------|--------|
| **Full Name** | Reporting Uplift from Gamification |
| **Code** | GAM-6 |
| **Definition** | Percentage increase in reporting frequency after gamification activation vs. pre-gamification period |
| **Formula** | `(Avg reports/week post-gamification - Pre-gamification) / Pre-gamification × 100` |
| **Target** | ≥20% uplift |
| **Data Source** | System analytics (before/after Week 5 gamification launch) |
| **Collection Frequency** | Monthly (comparing to pre-gamification baseline) |
| **Responsible** | M&E Lead |
| **Note** | This is a quasi-experimental comparison within the same CHWs |

### 8.8 Gamification KPI Summary Table

| Code | KPI | Formula | Target | Frequency | Data Source |
|------|-----|---------|--------|-----------|-------------|
| GAM-1 | Total Points Distributed | SUM(all points) | ≥2,000 by W12 | Daily | Gamification engine |
| GAM-2 | Badge Unlock Rate | ≥1 badge CHWs / Active × 100 | ≥80% by W8 | Weekly | Gamification engine |
| GAM-3 | Leaderboard Check Rate | Leaderboard viewers / Active × 100 | ≥50% weekly | Weekly | Gamification engine |
| GAM-4 | Streak Statistics | MAX/AVG streak lengths | Avg ≥14 days | Weekly | Gamification engine |
| GAM-5 | Reward Redemption Rate | Redeemed / Earned × 100 | ≥90% | Weekly | Gamification engine |
| GAM-6 | Reporting Uplift | (Post - Pre) / Pre × 100 | ≥20% increase | Monthly | System comparison |

---

## 9. KPI Category 6: Financial & Efficiency KPIs

### 9.1 FIN-1: Cost per AMR Report

| Attribute | Detail |
|-----------|--------|
| **Full Name** | Cost per AMR Report |
| **Code** | FIN-1 |
| **Definition** | Total pilot cost divided by total AMR reports submitted |
| **Formula** | `Total pilot expenditure / Total AMR reports` |
| **Target** | <$10 per report |
| **Stretch Target** | <$5 per report |
| **Data Source** | Financial records + system analytics |
| **Collection Frequency** | Monthly (cumulative) |
| **Responsible** | PI |
| **Note** | Key metric for funders — demonstrates cost-effectiveness |

**Projected Cost per Report:**

| Scenario | Total Reports | Cost/Report |
|----------|--------------|-------------|
| Low (200 reports) | 200 | $42 |
| Target (500 reports) | 500 | $17 |
| High (1,000 reports) | 1,000 | $8 |
| Stretch (2,000 reports) | 2,000 | $4 |

### 9.2 FIN-2: Cost per CHW per Month

| Attribute | Detail |
|-----------|--------|
| **Full Name** | Cost per Active CHW per Month |
| **Code** | FIN-2 |
| **Definition** | Monthly operating cost divided by number of active CHWs |
| **Formula** | `Monthly operating cost / Active CHWs` |
| **Target** | <$25/CHW/month for pilot |
| **Projected Scale Cost** | $2/CHW/month at 10,000 CHWs |
| **Data Source** | Financial records + system analytics |
| **Collection Frequency** | Monthly |
| **Responsible** | PI |

### 9.3 FIN-3: Supervisor Time per Site per Week

| Attribute | Detail |
|-----------|--------|
| **Full Name** | Supervisor Time Efficiency |
| **Code** | FIN-3 |
| **Definition** | Time coordinator spends per site per week on supervision activities |
| **Formula** | `Hours per site visit + remote supervision hours / 3 sites` |
| **Target** | <2 hours/site/week (including travel) by Week 10 |
| **Data Source** | Coordinator timesheets |
| **Collection Frequency** | Weekly |
| **Responsible** | Coordinator |

---

## 10. Composite Scorecards

### 10.1 Weekly Pilot Health Score

The Weekly Pilot Health Score provides a single composite metric (0-100) that summarizes overall pilot performance for the steering committee.

```
┌────────────────────────────────────────────────────────────────────┐
│              WEEKLY PILOT HEALTH SCORE                              │
├────────────────────────────────────────────────────────────────────┤
│                                                                    │
│  Score = (Adoption × 0.30) + (Technical × 0.25)                  │
│        + (Data Quality × 0.25) + (Public Health × 0.20)          │
│                                                                    │
│  ┌───────────────────┬──────────┬──────────┬───────────┐          │
│  │ Component         │ Weight   │ Raw Score│ Weighted  │          │
│  ├───────────────────┼──────────┼──────────┼───────────┤          │
│  │ Adoption Index    │   30%    │   85/100 │   25.5    │          │
│  │ Technical Index   │   25%    │   95/100 │   23.8    │          │
│  │ Data Quality Index│   25%    │   88/100 │   22.0    │          │
│  │ Public Health Idx │   20%    │   60/100 │   12.0    │          │
│  ├───────────────────┼──────────┼──────────┼───────────┤          │
│  │ TOTAL SCORE       │          │          │   83.3    │          │
│  └───────────────────┴──────────┴──────────┴───────────┘          │
│                                                                    │
│  Rating: 🟢 GREEN (≥75) | 🟡 YELLOW (50-74) | 🔴 RED (<50)      │
│                                                                    │
│  Component Sub-Indices:                                            │
│                                                                    │
│  ADOPTION INDEX (0-100):                                           │
│    = (Activation × 0.30) + (Active Rate × 0.25)                   │
│    + (Reporting Freq × 0.25) + (Retention × 0.20)                 │
│                                                                    │
│  TECHNICAL INDEX (0-100):                                          │
│    = (Uptime × 0.30) + (Latency × 0.25) + (Sync × 0.25)         │
│    + (Resource Util × 0.20)                                       │
│                                                                    │
│  DATA QUALITY INDEX (0-100):                                       │
│    = (Completeness × 0.35) + (Accuracy × 0.30)                    │
│    + (Timeliness × 0.20) + (Duplicate Rate × 0.15)                │
│                                                                    │
│  PUBLIC HEALTH INDEX (0-100):                                      │
│    = (Reports/week × 0.30) + (Actionable % × 0.25)                │
│    + (Guidance Adherence × 0.25) + (Patterns × 0.20)              │
│                                                                    │
└────────────────────────────────────────────────────────────────────┘
```

### 10.2 Traffic Light Dashboard

| KPI | W3 | W4 | W5 | W6 | W7 | W8 | W9 | W10 | W11 | W12 |
|-----|----|----|----|----|----|----|----|-----|-----|-----|
| CHW Activation | 🟢 | 🟢 | 🟢 | 🟢 | 🟢 | 🟢 | 🟢 | 🟢 | 🟢 | 🟢 |
| Weekly Active | 🟢 | 🟢 | 🟡 | 🟢 | 🟢 | 🟢 | 🟢 | 🟢 | 🟢 | 🟢 |
| Data Completeness | 🟢 | 🟡 | 🟡 | 🟢 | 🟢 | 🟢 | 🟢 | 🟢 | 🟢 | 🟢 |
| API Uptime | 🟢 | 🟢 | 🟢 | 🟢 | 🟡 | 🟢 | 🟢 | 🟢 | 🟢 | 🟢 |
| Sync Success | 🟢 | 🟢 | 🟡 | 🟢 | 🟢 | 🟢 | 🟢 | 🟢 | 🟢 | 🟢 |
| Inference Latency | 🟢 | 🟢 | 🟢 | 🟢 | 🟢 | 🟢 | 🟢 | 🟢 | 🟢 | 🟢 |
| Reports/Week | 🔴 | 🟡 | 🟡 | 🟢 | 🟢 | 🟢 | 🟢 | 🟢 | 🟢 | 🟢 |
| Guidance Adherence | — | 🟡 | 🟡 | 🟢 | 🟢 | 🟢 | 🟢 | 🟢 | 🟢 | 🟢 |

---

## 11. Data Collection Methods

### 11.1 Method Matrix

| Method | Type | KPIs Covered | Frequency | Responsible |
|--------|------|-------------|-----------|-------------|
| Prometheus metrics | Automated | All Technical KPIs | Continuous (10s) | Engineer |
| System analytics | Automated | Adoption, Data Quality (completeness/timeliness/duplicates), Gamification | Per event | System |
| CHW WhatsApp survey | Structured survey | CHW NPS, satisfaction, qualitative feedback | Monthly | M&E Lead |
| Supervisor spot-checks | Direct observation | Data accuracy, field verification | Weekly (15 reports) | Coordinator |
| Expert clinical review | Expert review | AI classification accuracy, resistance pattern validation | Monthly (30 reports) | UCH Faculty |
| Health ministry report | Secondary data | National comparison data, policy changes | Quarterly | PI |
| Focus group discussion | Qualitative | In-depth adoption barriers, satisfaction, suggestions | 2x (midpoint + endline) | PI + Coordinator |
| Key informant interview | Qualitative | Partnership effectiveness, regulatory feedback, scalability | 2x (midpoint + endline) | PI |
| Financial tracking | Administrative | Budget adherence, cost per report | Monthly | PI |
| Desk review | Secondary | Political context, regulatory changes, AMR publications | Monthly | PI |

### 11.2 Monthly CHW Survey Template

The monthly CHW survey is delivered via WhatsApp (simple text-based):

```
┌────────────────────────────────────────────────────────────────────┐
│              MONTHLY CHW SURVEY (WhatsApp)                          │
├────────────────────────────────────────────────────────────────────┤
│                                                                    │
│  📋 UDARA AI Monthly Check-In — [MONTH]                           │
│                                                                    │
│  Q1. How easy is UDARA AI to use?                                 │
│      1 = Very difficult  →  5 = Very easy                         │
│                                                                    │
│  Q2. How likely are you to recommend UDARA AI to another CHW?     │
│      0 = Not at all  →  10 = Extremely likely                     │
│                                                                    │
│  Q3. How many times did you have problems using UDARA AI          │
│      this month?                                                   │
│      0 = None  →  5 = Very often                                  │
│                                                                    │
│  Q4. What is the BIGGEST challenge you face using UDARA AI?       │
│      (Reply with number or your own words)                         │
│      1. Connectivity problems                                     │
│      2. Battery/charging issues                                   │
│      3. Don't understand AI response                              │
│      4. Takes too long                                             │
│      5. Other: _____                                              │
│                                                                    │
│  Q5. What do you LIKE MOST about UDARA AI?                        │
│      (Free text)                                                   │
│                                                                    │
│  Thank you! Your feedback helps us improve. 💚                    │
│                                                                    │
└────────────────────────────────────────────────────────────────────┘
```

---

## 12. Reporting Cadence & Templates

### 12.1 Reporting Schedule

```
┌────────────────────────────────────────────────────────────────────┐
│              REPORTING CADENCE                                       │
├────────────────────────────────────────────────────────────────────┤
│                                                                    │
│  DAILY                                                             │
│  ├── Automated health check (Prometheus → Slack/email)            │
│  ├── CHW activity summary (automated)                              │
│  └── Sync status per site (automated)                              │
│                                                                    │
│  WEEKLY                                                            │
│  ├── Product usage report (adoption + data quality)               │
│  ├── Technical reliability report (uptime + latency + sync)       │
│  ├── Gamification update (leaderboard + badge distribution)       │
│  └── Coordinator field report (qualitative observations)          │
│                                                                    │
│  BI-WEEKLY                                                         │
│  ├── Public health surveillance update                             │
│  ├── Data quality deep dive (accuracy + completeness trends)      │
│  └── Risk register update                                          │
│                                                                    │
│  MONTHLY                                                           │
│  ├── Full M&E Report (all KPIs, traffic lights, trends)          │
│  ├── CHW survey results                                            │
│  ├── Financial status report                                       │
│  ├── Partnership progress report                                   │
│  └── Steering committee briefing                                    │
│                                                                    │
│  QUARTERLY / ENDLINE                                               │
│  ├── Comprehensive pilot evaluation report                         │
│  ├── Funder impact report (Gates/Wellcome format)                 │
│  ├── Academic paper draft                                          │
│  ├── Policy brief for health authorities                           │
│  └── Lessons learned document                                      │
│                                                                    │
└────────────────────────────────────────────────────────────────────┘
```

### 12.2 Weekly M&E Report Template

```markdown
# UDARA AI Weekly M&E Report — Week [X]
Date: [YYYY-MM-DD]
Prepared by: [Name]

## 1. Pilot Health Score: [XX/100] — [🟢/🟡/🔴]

## 2. KPI Dashboard

### Adoption
| KPI | This Week | Last Week | Target | Status |
|-----|-----------|-----------|--------|--------|
| CHW Activation | X% | X% | ≥80% | 🟢 |
| Weekly Active | X% | X% | ≥60% | 🟢 |
| Reports/CHW/Week | X.X | X.X | ≥3 | 🟡 |
| Channel Distribution | WA:X% TG:X% USSD:X% V:X% | — | Balanced | — |

### Data Quality
| KPI | This Week | Last Week | Target | Status |
|-----|-----------|-----------|--------|--------|
| Completeness | X% | X% | ≥90% | 🟢 |
| Timeliness | X.Xh | X.Xh | <4h | 🟢 |
| Duplicate Rate | X% | X% | <5% | 🟢 |

### Technical
| KPI | This Week | Target | Status |
|-----|-----------|--------|--------|
| API Uptime | X.XX% | ≥99.5% | 🟢 |
| Inference Latency (P95) | X.Xs | <5s | 🟢 |
| Sync Success | X.X% | ≥95% | 🟢 |

### Public Health
| KPI | This Week | Cumulative | Status |
|-----|-----------|-----------|--------|
| Total Reports | XX | XXX | 🟢 |
| Reports/Week | XX | — | 🟡 |
| Resistance Patterns | — | X | — |
| Guidance Adherence | X% | — | 🟢 |

## 3. Gamification Update
- Total points distributed: XXXX
- New badges earned: [list]
- Weekly challenge winner: [name]
- Leaderboard top 3: [names]

## 4. Key Observations
- [Qualitative note 1]
- [Qualitative note 2]

## 5. Issues & Actions
| Issue | Severity | Action | Owner | Due |
|-------|----------|--------|-------|-----|
| [Issue] | [H/M/L] | [Action] | [Name] | [Date] |

## 6. Next Week Priorities
1. [Priority 1]
2. [Priority 2]
3. [Priority 3]
```

### 12.3 Monthly M&E Report Template

```markdown
# UDARA AI Monthly M&E Report — [Month]
Period: [Start] to [End]
Prepared by: [Name]

## Executive Summary
[2-3 paragraph summary of the month's performance, key achievements, and concerns]

## 1. Pilot Health Score Trend
[Line chart showing score over 4 weeks]

## 2. KPI Deep Dive

### 2.1 Adoption Analysis
[Detailed analysis with trend charts, by-site breakdown, qualitative insights]

### 2.2 Data Quality Analysis
[Detailed analysis, field-level completeness, accuracy by site]

### 2.3 Technical Performance
[Uptime breakdown by site, latency distributions, resource utilization trends]

### 2.4 Public Health Impact
[Resistance map update, new findings, comparison with national data]

### 2.5 Gamification Effectiveness
[Points distribution, badge trends, correlation with reporting frequency]

## 3. CHW Survey Results
[Monthly survey findings, NPS score, key quotes]

## 4. Financial Status
[Budget vs. actual, burn rate, projection]

## 5. Risk Register Update
[Updated risk scores, new risks, mitigated risks]

## 6. Recommendations
[3-5 actionable recommendations for the next month]

## Appendices
- Detailed KPI tables
- Site-level breakdowns
- Raw data exports
```

---

## 13. Grafana Dashboard Layout

### 13.1 Dashboard Architecture

```
┌────────────────────────────────────────────────────────────────────────────┐
│                    GRAFANA DASHBOARD — "UDARA PILOT OVERVIEW"               │
│                    URL: https://grafana.udara.ai/d/pilot-overview           │
├────────────┬────────────┬────────────┬────────────┬────────────────────────┤
│   ROW 1    │   ROW 1    │   ROW 1    │   ROW 1    │   ROW 1               │
│  System    │  System    │  System    │  System    │  System               │
│  Health    │  Health    │  Health    │  Health    │  Health               │
│  (Uptime)  │ (Latency) │ (Errors)  │  (Sync)    │ (Resources)          │
├────────────┴────────────┴────────────┴────────────┼────────────────────────┤
│   ROW 2                                          │   ROW 2               │
│   Adoption                                       │   Adoption            │
│   (Active CHWs trend)                            │   (Channel pie chart) │
├──────────────────────────────────────────────────┼────────────────────────┤
│   ROW 3                                          │   ROW 3               │
│   AMR Surveillance                               │   AMR Surveillance     │
│   (Resistance trend)                             │   (Resistance map)     │
├────────────┬────────────┬────────────┬───────────┴────────────────────────┤
│   ROW 4    │   ROW 4    │   ROW 4    │   ROW 4                         │
│  Gamific.  │  Gamific.  │  Gamific.  │  Gamification                    │
│  (Leader)  │ (Badges)   │ (Streaks)  │  (Points trend)                  │
└────────────┴────────────┴────────────┴───────────────────────────────────┘
```

### 13.2 Row 1: System Health (5 panels)

| Panel | Type | Data Source | Refresh | Alert Threshold |
|-------|------|-------------|---------|----------------|
| 1.1 API Uptime | Stat gauge (0-100%) | Prometheus: `up{job="udara-edge"}` | 10s | <95% = warning, <90% = critical |
| 1.2 Inference Latency | Histogram | Prometheus: `histogram_quantile(0.95, inference_duration)` | 1m | >10s = warning |
| 1.3 Error Rate | Time series (area) | Prometheus: `rate(http_errors_total[5m])` | 1m | >1% = warning |
| 1.4 Sync Status | Status grid (per site) | Sync agent: `last_sync_success` | 5m | >30min = warning, >2h = critical |
| 1.5 Resource Usage | Stacked bar | Prometheus: `container_memory_usage_bytes` | 30s | >80% = warning |

### 13.3 Row 2: Adoption (2 panels)

| Panel | Type | Data Source | Refresh |
|-------|------|-------------|---------|
| 2.1 Active CHWs Over Time | Line chart (stacked by site) | System analytics | 1h |
| 2.2 Channel Distribution | Pie chart | System analytics | 1h |

### 13.4 Row 3: AMR Surveillance (2 panels)

| Panel | Type | Data Source | Refresh |
|-------|------|-------------|---------|
| 3.1 Resistance Trend (Top 5 Pathogen-Antibiotic Pairs) | Multi-line chart | Surveillance database | 24h |
| 3.2 Resistance Heatmap (Oyo State) | Geomap | Surveillance database + GPS | 24h |

### 13.5 Row 4: Gamification (4 panels)

| Panel | Type | Data Source | Refresh |
|-------|------|-------------|---------|
| 4.1 Points Leaderboard | Table (top 10) | Gamification engine | 1h |
| 4.2 Badge Distribution | Bar chart (per badge type) | Gamification engine | 24h |
| 4.3 Active Streaks | Stat series | Gamification engine | 1h |
| 4.4 Points Distributed Over Time | Area chart (cumulative) | Gamification engine | 24h |

### 13.6 Dashboard Access Control

| Role | Access Level | Views |
|------|-------------|-------|
| Engineer | Full (all panels + drill-down) | System health detail, container logs, raw metrics |
| Coordinator | Standard (all rows) | All panels, no raw metrics access |
| PI | Standard + export | All panels + CSV/PDF export |
| UI/UCH Partner | Read-only (Rows 1, 3) | System health + AMR surveillance |
| Steering Committee | Executive summary | Weekly score + traffic light summary |

---

## 14. Evaluation Design

### 14.1 Evaluation Approach

The UDARA AI pilot evaluation employs a **mixed-methods quasi-experimental design** with the following components:

```
┌────────────────────────────────────────────────────────────────────┐
│              EVALUATION DESIGN COMPONENTS                           │
├────────────────────────────────────────────────────────────────────┤
│                                                                    │
│  1. QUANTITATIVE — Pre/Post Comparison                            │
│     ├── Baseline assessment (Week 0): AMR awareness, reporting    │
│     │   practices, data quality metrics                            │
│     ├── Midline (Week 6): KPI check, adoption metrics            │
│     └── Endline (Week 12): Full KPI assessment, comparison        │
│                                                                    │
│  2. QUANTITATIVE — Time Series Analysis                           │
│     ├── Weekly reporting trends (adoption curve)                 │
│     ├── Resistance pattern emergence (surveillance value)          │
│     └── Technical performance trends (system reliability)          │
│                                                                    │
│  3. QUALITATIVE — Interviews & Focus Groups                       │
│     ├── CHW individual interviews (6 × 45 min, endline)          │
│     ├── CHW focus group (1 × 90 min, endline)                     │
│     ├── Supervisor interview (1 × 60 min, endline)               │
│     ├── Partner interviews (2 × 45 min, UI + UCH, endline)       │
│     └── Health authority interview (1 × 45 min, post-pilot)      │
│                                                                    │
│  4. COMPARATIVE — National Lab Comparison                         │
│     ├── UDARA AI resistance data vs. UCH lab data (concordance)  │
│     ├── Time-to-detection comparison                              │
│     └── Resistance rate comparison                                │
│                                                                    │
│  5. ECONOMIC — Cost-Effectiveness Analysis                         │
│     ├── Cost per report                                           │
│     ├── Cost per resistance pattern detected                      │
│     ├── Cost per CHW engaged                                      │
│     └── Projected cost at scale (10,000 CHWs)                     │
│                                                                    │
└────────────────────────────────────────────────────────────────────┘
```

### 14.2 Interview Guide (CHW Endline)

```
┌────────────────────────────────────────────────────────────────────┐
│              CHW ENDLINE INTERVIEW GUIDE                            │
│              Duration: 45 minutes                                   │
│              Format: Semi-structured, Yoruba/English                │
├────────────────────────────────────────────────────────────────────┤
│                                                                    │
│  WARM-UP (5 min)                                                  │
│  • Thank you for participating                                     │
│  • Remind: this is confidential, no impact on your job            │
│  • Consent reminder                                                │
│                                                                    │
│  SECTION A: EXPERIENCE (10 min)                                    │
│  1. Can you describe your experience using UDARA AI over the       │
│     past 12 weeks?                                                 │
│  2. What did you find EASIEST about using the system?             │
│  3. What did you find MOST DIFFICULT?                             │
│  4. How has UDARA AI changed your daily work routine?             │
│                                                                    │
│  SECTION B: CHANNEL PREFERENCES (10 min)                           │
│  5. Which channel do you prefer? (WhatsApp/Telegram/USSD/Voice)   │
│  6. Why do you prefer that channel?                               │
│  7. Have you ever been unable to report because of connectivity?  │
│     What did you do?                                               │
│                                                                    │
│  SECTION C: AI RESPONSES (8 min)                                  │
│  8. How useful are the AI responses you receive?                  │
│  9. Do you trust the AI guidance? Why or why not?                 │
│  10. Have you ever disagreed with the AI? What happened?          │
│                                                                    │
│  SECTION D: GAMIFICATION (7 min)                                   │
│  11. What do you think about the points and badges system?       │
│  12. Did the leaderboard motivate you? How?                       │
│  13. What rewards would you like to see in the future?            │
│                                                                    │
│  SECTION E: SUSTAINABILITY (5 min)                                 │
│  14. Would you continue using UDARA AI after the pilot ends?      │
│  15. What would make you stop using it?                            │
│  16. What would you tell another CHW about UDARA AI?             │
│                                                                    │
│  CLOSING (2 min)                                                   │
│  • Any other comments?                                             │
│  • Thank you                                                       │
│                                                                    │
└────────────────────────────────────────────────────────────────────┘
```

### 14.3 Cost-Effectiveness Framework

```
┌────────────────────────────────────────────────────────────────────┐
│              COST-EFFECTIVENESS ANALYSIS                            │
├────────────────────────────────────────────────────────────────────┤
│                                                                    │
│  COSTS (12-week pilot)                                             │
│  ─────                                                            │
│  Total Pilot Cost:              $8,388                             │
│  Per-Site Cost:                 $2,796                             │
│  Per-CHW Cost (6 CHWs):         $1,398                             │
│  Per-CHW Cost (monthly):        $466                               │
│                                                                    │
│  EFFECTIVENESS (Projected, 500 reports)                           │
│  ─────────────────                                        │
│  Cost per AMR Report:          $16.78                              │
│  Cost per Complete Report:     $18.64 (est. 90% complete)          │
│  Cost per Resistance Pattern:  $2,796 (est. 3 patterns)           │
│  Cost per CHW Activated:       $1,398                              │
│                                                                    │
│  SCALE PROJECTIONS (Year 2, Nigeria: 10,000 CHWs)                 │
│  ───────────────────────────────────────                        │
│  Annual Cost:                   $240,000                           │
│  Per-CHW/Month:                 $2.00                              │
│  Annual Reports (est. 100,000):Cost/Report: $2.40                 │
│                                                                    │
│  COMPARISON WITH ALTERNATIVES                                     │
│  ───────────────────────────                                    │
│  ┌──────────────────┬──────────┬───────────────┬──────────┐       │
│  │ Method           │ Cost/    │ Coverage      │ Speed    │       │
│  │                  │ Report   │               │          │       │
│  ├──────────────────┼──────────┼───────────────┼──────────┤       │
│  │ National Lab     │ $150+    │ <5% of pop.   │ 2-6 mo.  │       │
│  │ Surveillance     │          │               │          │       │
│  │ Paper-based CHW  │ $8       │ 30-50% of HPs │ 1-4 wk.  │       │
│  │ Reporting        │          │               │          │       │
│  │ DHIS2 Aggregate  │ $5       │ National      │ Monthly  │       │
│  │                  │          │               │          │       │
│  │ UDARA AI (pilot) │ $17      │ 100% at sites │ <1 day   │       │
│  │ UDARA AI (scale) │ $2.40    │ 100% at sites │ Real-time│       │
│  └──────────────────┴──────────┴───────────────┴──────────┘       │
│                                                                    │
└────────────────────────────────────────────────────────────────────┘
```

---

## 15. Data Quality Assurance

### 15.1 Data Quality Framework

```
┌────────────────────────────────────────────────────────────────────┐
│              DATA QUALITY ASSURANCE FRAMEWORK                       │
├────────────────────────────────────────────────────────────────────┤
│                                                                    │
│  PREVENTION (Before Data Collection)                               │
│  ──────────────────────────────                                  │
│  • Input validation in all channels (required fields enforced)    │
│  • Dropdown menus for standard values (pathogen, antibiotic)      │
│  • AI-assisted field pre-population                               │
│  • Real-time error messages for invalid data                       │
│  • CHW training on data quality expectations                       │
│                                                                    │
│  DETECTION (During Data Collection)                               │
│  ─────────────────────────────                                  │
│  • Automated completeness check per report                        │
│  • Duplicate detection (same patient + date + site)               │
│  • Outlier detection (impossible values, inconsistencies)          │
│  • Accuracy sampling (15 reports/week spot-checked)               │
│                                                                    │
│  CORRECTION (After Data Collection)                               │
│  ─────────────────────────────                                  │
│  • Automated data cleaning scripts                                │
│  • CHW notification of incomplete reports (with retry prompt)     │
│  • Supervisor correction interface                                │
│  • Monthly data quality audit                                     │
│  • Expert clinical review (AI classification validation)          │
│                                                                    │
│  REPORTING (Ongoing)                                              │
│  ────────────────                                                │
│  • Weekly data quality score on Grafana                           │
│  • Monthly data quality report to steering committee              │
│  • Field-level completeness breakdown by site                     │
│  • Trend analysis (improving or declining)                        │
│                                                                    │
└────────────────────────────────────────────────────────────────────┘
```

### 15.2 Data Validation Rules

| Rule ID | Field | Validation | Action on Fail |
|---------|-------|-----------|---------------|
| V001 | Patient age | Must be 0-120 | Prompt: "Please enter valid age" |
| V002 | Age group | Must match standard categories | Auto-categorize from age |
| V003 | Antibiotic name | Must be in approved drug list | Fuzzy match to nearest drug |
| V004 | Symptoms | Must be ≥10 characters | Prompt: "Please describe symptoms in more detail" |
| V005 | Report type | Must be one of: case, follow-up, lab result | Dropdown selection |
| V006 | Date of visit | Must be ≤7 days ago | Warning: "This report is for a visit more than 7 days ago" |
| V007 | GPS coordinates | Must be within Oyo State bounds | Auto-tag from site if missing |
| V008 | Duplicate check | Same patient + visit date + site | Flag as potential duplicate, allow override |
| V009 | Outcome | Required if antibiotic was prescribed | Prompt: "Please record treatment outcome" |
| V010 | Voice duration | Must be ≥5 seconds and ≤120 seconds | Prompt: "Recording too short/long" |

---

## 16. Ethical Considerations in M&E

### 16.1 Ethical Principles

| Principle | Application in UDARA AI M&E |
|-----------|---------------------------|
| **Informed Consent** | All CHWs provide written informed consent before participation; consent covers data collection, surveys, and interviews |
| **Voluntary Participation** | CHWs can withdraw at any time without penalty; no coercion through gamification or supervisor pressure |
| **Confidentiality** | All interview data de-identified; survey responses aggregated; no individual CHW data shared without consent |
| **Data Protection** | All personal data encrypted (AES-256); NDPR compliant; data stored on secure servers; retention policy defined |
| **Beneficence** | M&E activities designed to improve the system for CHWs; findings shared back to participants; benefits outweigh risks |
| **Justice** | Fair selection of CHWs; no discrimination; equal access to gamification rewards |
| **Community Benefit** | All data ultimately benefits the community through improved AMR surveillance; aggregate findings shared with health authorities |

### 16.2 IRB/Ethics Approval

| Item | Status | Details |
|------|--------|---------|
| Ethics committee | UI/UCH Ethics Committee | Submission planned for T-10 weeks |
| Protocol number | [TBD] | — |
| Approval date | [TBD] | Target: before Week 1 |
| Expiry | [TBD] | Must cover full 12-week pilot + 3-month analysis |
| Amendment process | If protocol changes | Submit amendment within 5 business days |
| Annual renewal | If pilot extends | Submit renewal 30 days before expiry |

### 16.3 Data Retention & Destruction

| Data Type | Retention Period | Destruction Method |
|-----------|-----------------|-------------------|
| System analytics (aggregated) | 5 years | Secure deletion |
| CHW personal data | Pilot + 3 years | Secure deletion with certificate |
| Interview recordings | Analysis + 1 year | Secure deletion |
| Transcript data | Publication + 5 years | Secure deletion |
| AMR surveillance data | Indefinite (public health value) | Archived (anonymized) |
| GPS coordinates | Indefinite (surveillance) | Anonymized (district-level only) |

---

## 17. Appendices

### Appendix A: Master KPI Reference Table

| Code | Category | KPI | Formula | Target | Frequency | Data Source |
|------|----------|-----|---------|--------|-----------|-------------|
| ADOPT-1 | Adoption | CHW Activation Rate | ≥1 report CHWs / Total × 100 | ≥80% | Weekly | System |
| ADOPT-2 | Adoption | Weekly Active Reporters | Active / Total × 100 | ≥60% | Weekly | System |
| ADOPT-3 | Adoption | Channel Distribution | Per channel / Total × 100 | Balanced | Weekly | System |
| ADOPT-4 | Adoption | Retention Rate (MoM) | Active both / Active prev × 100 | ≥85% | Monthly | System |
| ADOPT-5 | Adoption | Reporting Frequency | Reports / Active CHWs | ≥3/wk | Weekly | System |
| ADOPT-6 | Adoption | Time to First Report | First report - Training | ≤2 days | Once | System |
| ADOPT-7 | Adoption | Multi-Channel Usage | ≥2 channels / Active × 100 | ≥30% | Monthly | System |
| ADOPT-8 | Adoption | CHW NPS | % Promoters - % Detractors | ≥30 | Monthly | Survey |
| DQ-1 | Data Quality | Completeness Rate | Complete / Total × 100 | ≥90% | Weekly | System |
| DQ-2 | Data Quality | Accuracy Rate | Verified / Checked × 100 | ≥85% | Weekly | Supervisor |
| DQ-3 | Data Quality | Timeliness | AVG(submission - encounter) | <4h | Weekly | System |
| DQ-4 | Data Quality | Duplicate Rate | Duplicates / Total × 100 | <5% | Weekly | System |
| DQ-5 | Data Quality | AI Classification Accuracy | AI matches expert / Reviewed × 100 | ≥80% | Monthly | Expert |
| TECH-1 | Technical | API Uptime % | Uptime / Total × 100 | ≥99.5% | Continuous | Prometheus |
| TECH-2 | Technical | Inference Latency (P95) | P95(request time) | <5s text | Per request | Prometheus |
| TECH-3 | Technical | Sync Success Rate | Successful / Attempts × 100 | ≥95% | Per sync | Sync logs |
| TECH-4 | Technical | Offline Resilience | MAX(offline hours) | ≥168h | Event | Sync logs |
| TECH-5 | Technical | Resource Utilization | CPU/RAM/Storage/Temp | <80% each | Continuous | Prometheus |
| TECH-6 | Technical | USB Fallback Events | COUNT(USB events) | <2/mo/device | Event | Sync logs |
| TECH-7 | Technical | Error Rate | 5xx / Total × 100 | <0.5% | Per request | Logs |
| TECH-8 | Technical | Model Load Time | Boot to ready | <180s | Per boot | Startup logs |
| PH-1 | Public Health | Resistance Patterns | COUNT(significant) | ≥1 | Monthly | Statistical |
| PH-2 | Public Health | Time to Detection | Date flagged - First report | <7 days | Event | System |
| PH-3 | Public Health | Districts Flagged | COUNT(high resistance) | ≥1 | Monthly | Spatial |
| PH-4 | Public Health | Guidance Adherence | Followed / Guided × 100 | ≥70% | Weekly | System |
| PH-5 | Public Health | Actionable Reports | With antibiotic+outcome / Total × 100 | ≥85% | Weekly | System |
| PH-6 | Public Health | Pathogen Coverage | COUNT(DISTINCT types) | ≥10 | Monthly | System |
| PH-7 | Public Health | Resistance Rates | Resistant / Total per pair | Track | Monthly | System |
| GAM-1 | Gamification | Total Points | SUM(all points) | ≥2000 by W12 | Daily | Gam engine |
| GAM-2 | Gamification | Badge Unlock Rate | ≥1 badge / Active × 100 | ≥80% | Weekly | Gam engine |
| GAM-3 | Gamification | Leaderboard Check | Viewers / Active × 100 | ≥50% | Weekly | Gam engine |
| GAM-4 | Gamification | Streak Statistics | MAX/AVG streak | Avg ≥14d | Weekly | Gam engine |
| GAM-5 | Gamification | Reward Redemption | Redeemed / Earned × 100 | ≥90% | Weekly | Gam engine |
| GAM-6 | Gamification | Reporting Uplift | (Post-Pre)/Pre × 100 | ≥20% | Monthly | System |
| FIN-1 | Financial | Cost per Report | Total cost / Reports | <$10 | Monthly | Finance |
| FIN-2 | Financial | Cost per CHW/Month | Monthly cost / Active | <$25 | Monthly | Finance |
| FIN-3 | Financial | Supervisor Time | Hours / site / week | <2h | Weekly | Timesheets |

### Appendix B: Data Collection Instruments

| Instrument | Target | Method | Frequency | Language |
|-----------|--------|--------|-----------|----------|
| Automated system metrics | RPi infrastructure | Prometheus scrape | Continuous | N/A (machine) |
| CHW activity log | All CHWs | System analytics | Per event | N/A (machine) |
| Monthly CHW survey | All active CHWs | WhatsApp text | Monthly | English + Yoruba |
| Supervisor spot-check form | 15 reports/week | Structured form | Weekly | English |
| Expert clinical review | 30 reports/month | Structured form | Monthly | English |
| CHW endline interview | 6 CHWs | Semi-structured | Once (W11-12) | Yoruba/English |
| CHW focus group | 6 CHWs | Focus group guide | Once (W12) | Yoruba/English |
| Partner interview | 2 partners | Semi-structured | Once (W12) | English |
| Coordinator field diary | Coordinator | Free-form notes | Daily | English |

### Appendix C: Statistical Analysis Plan

| Question | Method | Variables | Sample Size Requirement |
|----------|--------|-----------|----------------------|
| Is CHW adoption ≥80%? | One-sample proportion test | Activation rate | n=6 (pilot); limited power |
| Did gamification increase reporting? | Paired t-test (pre/post) | Reports/CHW/week before vs. after W5 | n=6; limited power |
| Is completeness ≥90%? | One-sample proportion test | Completeness rate | All reports |
| Are resistance rates different between sites? | Chi-square or Fisher's exact | Resistance yes/no × site | Depends on reports/site |
| Is UDARA AI faster than national lab? | Time-to-event comparison | Detection dates | 1+ pattern needed |
| Does reporting frequency correlate with satisfaction? | Spearman correlation | Reports/week × satisfaction score | n=6; descriptive only |

### Appendix D: Power Analysis

Given the small sample size (6 CHWs, 3 sites), the pilot is **not powered for inferential statistics**. The evaluation is primarily **descriptive and exploratory**. Key implications:

- No claim of statistical significance for adoption outcomes
- Focus on effect sizes and confidence intervals
- Qualitative data compensates for quantitative limitations
- Pilot designed for proof-of-concept, not efficacy trial
- Full RCT would be required for scale-up evidence (Phase 3)

### Appendix E: M&E Team RACI Matrix

| Activity | PI | M&E Lead | Engineer | Coordinator | UCH Partner |
|----------|----|----------|----------|-------------|-------------|
| KPI definition | A | R | C | I | C |
| Data collection (automated) | I | I | R | I | I |
| Data collection (surveys) | A | R | I | C | I |
| Data collection (spot-checks) | I | C | I | R | I |
| Grafana dashboard | I | C | R | I | I |
| Weekly M&E report | A | R | C | C | I |
| Monthly M&E report | A | R | C | C | I |
| Data analysis | A | R | C | I | C |
| Endline evaluation | A | R | C | C | R |
| Funder impact report | A | R | C | C | I |
| Publication writing | A | R | C | I | R |

R = Responsible, A = Accountable, C = Consulted, I = Informed

---

## Document Revision History

| Version | Date | Author | Changes |
|---------|------|--------|---------|
| 1.0 | 2024-11-15 | M&E Team | Initial framework |
| 1.1 | 2024-12-01 | M&E Lead | KPI definitions finalized |
| 2.0 | 2024-12-15 | Full Team | Grafana layout, evaluation design added |
| 2.1 | 2025-01-05 | M&E Lead | Gamification KPIs added |
| 2.2 | 2025-01-10 | PI | Cost-effectiveness analysis added |
| 2.3 | 2025-01-15 | PI | Final review, ethics section completed |

---

*This document is the authoritative source for all M&E activities during the UDARA AI pilot. All KPIs are final and should not be modified without steering committee approval.*
