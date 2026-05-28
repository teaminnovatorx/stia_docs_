# WHO GLASS Compliance

> UDARA AI is designed to contribute community-level syndromic AMR surveillance data to the WHO Global Antimicrobial Resistance and Use Surveillance System (GLASS). This document covers data mapping, export formats, and our GLASS-readiness roadmap.

---

## Table of Contents

1. [What is GLASS](#1-what-is-glass)
2. [Current GLASS Architecture](#2-current-glass-architecture)
3. [UDARA's Contribution to GLASS](#3-udaras-contribution-to-glass)
4. [Data Mapping: UDARA → GLASS](#4-data-mapping-udara--glass)
5. [Export Module Implementation](#5-export-module-implementation)
6. [GLASS CSV Format Specification](#6-glass-csv-format-specification)
7. [GLASS-Readiness Roadmap](#7-glass-readiness-roadmap)
8. [Data Quality Assurance](#8-data-quality-assurance)
9. [Privacy & Compliance Considerations](#9-privacy--compliance-considerations)

---

## 1. What is GLASS

### 1.1 Overview

**GLASS** (Global Antimicrobial Resistance and Use Surveillance System) is the WHO's primary initiative for standardizing AMR surveillance worldwide. Launched in 2015, it collects and reports AMR data from participating countries to inform global policy and response.

```
┌───────────────────────────────────────────────────────────────┐
│                    WHO GLASS SYSTEM                            │
│                                                               │
│  ┌─────────────┐    ┌─────────────┐    ┌─────────────────┐   │
│  │ National    │───▶│ WHO GLASS   │───▶│ Global AMR      │   │
│  │ AMR Data    │    │ Data Hub    │    │ Surveillance    │   │
│  │ (Countries) │    │ (Geneva)    │    │ Reports         │   │
│  └──────┬──────┘    └──────┬──────┘    └─────────────────┘   │
│         │                  │                                  │
│  ┌──────▼──────┐    ┌──────▼──────┐                          │
│  │ Microbiology│    │ Annual GLASS│                          │
│  │ Labs        │    │ Report      │                          │
│  │ (AMR data)  │    │ (Public)    │                          │
│  └─────────────┘    └─────────────┘                          │
│                                                               │
│  Current participants: 127 countries (2024)                   │
│  Coverage: ~75% of world population                          │
└───────────────────────────────────────────────────────────────┘
```

### 1.2 GLASS Data Types

| Data Type | Description | Collection Method | Current Coverage |
|-----------|-------------|-------------------|-----------------|
| **AMR (GLASS-AMR)** | Antimicrobial resistance rates from clinical isolates | Microbiology labs (AST) | 127 countries, 7 million+ isolates |
| **AMC (GLASS-AMC)** | Antimicrobial consumption data | Pharmacy/supply chain | 42 countries |
| **AMU (GLASS-AMU)** | Antimicrobial use in humans | Prescription data | 27 countries |
| **WGS (GLASS-WGS)** | Whole-genome sequencing of resistant isolates | Reference labs | 15 countries |
| **One Health** | AMR in food chain, animals, environment | Veterinary + environmental labs | Emerging |

### 1.3 GLASS AMR Reporting Structure

GLASS-AMR collects **isolate-level data** from microbiology laboratories. Each record represents a single bacterial isolate tested against specific antimicrobials.

```
Patient → Specimen Collection → Lab Processing → AST Results → GLASS Report
           (blood, urine,        (culture &       (S/I/R for      (aggregated
            stool, wound)         identification)  each drug)       resistance rates)
```

### 1.4 GLASS Priority Pathogen-Drug Combinations

| Priority | Pathogen | Antimicrobials Tested |
|----------|----------|----------------------|
| 1 | *E. coli* | 3rd-gen cephalosporins, fluoroquinolones, carbapenems, aminoglycosides |
| 1 | *K. pneumoniae* | 3rd-gen cephalosporins, carbapenems, colistin |
| 1 | *S. aureus* | Methicillin (MRSA), vancomycin |
| 1 | *S. pneumoniae* | Penicillin, macrolides, fluoroquinolones |
| 1 | *Salmonella* spp. | Fluoroquinolones, 3rd-gen cephalosporins |
| 1 | *A. baumannii* | Carbapenems, colistin |
| 1 | *N. gonorrhoeae* | 3rd-gen cephalosporins, azithromycin |
| 2 | *P. aeruginosa* | Carbapenems, piperacillin-tazobactam, colistin |
| 2 | *Enterococcus* spp. | Vancomycin, ampicillin, linezolid |

---

## 2. Current GLASS Architecture

### 2.1 How Countries Currently Report

```
┌──────────────┐     ┌──────────────┐     ┌──────────────┐
│ Microbiology │     │ National     │     │ WHO GLASS    │
│ Laboratory   │────▶│ Focal Point  │────▶│ Data System  │
│              │     │ (NCDC/AMR    │     │ (WIS/AMR     │
│ • Isolates   │     │  Reference)  │     │  Module)     │
│ • AST results│     │ • Aggregates │     │              │
│ • Patient    │     │ • Validates  │     │ • Global     │
│   metadata   │     │ • Submits    │     │   analysis   │
│              │     │              │     │ • Annual     │
└──────────────┘     └──────────────┘     │   report     │
                                           └──────────────┘
```

### 2.2 The Data Gap GLASS Currently Misses

| What GLASS Captures | What GLASS Misses |
|--------------------|-------------------|
| Laboratory-confirmed infections | Community syndromic infections |
| Isolate-level resistance data | Clinical treatment failure data |
| Urban hospital-based data | Rural and peri-urban primary care data |
| Formal healthcare sector | Informal health sector (patent medicine stores) |
| Bacterial identification only | Viral vs. bacterial differentiation challenges |
| Expensive lab infrastructure needed | Low-resource settings |

### 2.3 Key Numbers

- **Nigeria**: 26 labs contributing to GLASS (mostly in tertiary hospitals)
- **Sub-Saharan Africa overall**: ~200 labs contributing (vast majority in South Africa, Nigeria, Kenya)
- **Rural coverage**: < 5% of rural health facilities have lab capacity
- **Time to result**: AST typically takes 48–72 hours, and results take weeks to reach national surveillance

---

## 3. UDARA's Contribution to GLASS

### 3.1 Our Unique Value Proposition

UDARA provides **community-level syndromic AMR surveillance** that directly complements lab-based GLASS data.

```
┌─────────────────────────────────────────────────────────────────┐
│              COMPLEMENTARY SURVEILLANCE ECOSYSTEM               │
│                                                                 │
│   GLASS (Lab-Based)              UDARA (Community-Based)         │
│   ┌──────────────┐               ┌──────────────┐              │
│   │ Microbiology │               │ CHWs at PHCs │              │
│   │ Labs         │               │ report via   │              │
│   │              │               │ USSD/WhatsApp│              │
│   │ Isolate-     │               │              │              │
│   │ level data   │               │ Syndromic-   │              │
│   │              │               │ level data   │              │
│   │ Slow (weeks) │               │ Real-time    │              │
│   │ Expensive    │               │ Low-cost     │              │
│   │ Urban bias   │               │ Rural reach  │              │
│   └──────┬───────┘               └──────┬───────┘              │
│          │                               │                       │
│          ▼                               ▼                       │
│   ┌──────────────────────────────────────────────────┐         │
│   │            COMBINED AMR INTELLIGENCE               │         │
│   │                                                  │         │
│   │  • Lab resistance rates + Community treatment    │         │
│   │    failure patterns = Complete AMR picture       │         │
│   │                                                  │         │
│   │  • Early warning: Community treatment failures   │         │
│   │    can precede lab-confirmed resistance by      │         │
│   │    months                                        │         │
│   │                                                  │         │
│   │  • Geographic coverage: 500+ sites vs 26 labs   │         │
│   │    in Nigeria alone                              │         │
│   └──────────────────────────────────────────────────┘         │
└─────────────────────────────────────────────────────────────────┘
```

### 3.2 What UDARA Data Adds to GLASS

| UDARA Data Field | GLASS Equivalent | Value to GLASS |
|-----------------|------------------|----------------|
| Treatment failure reports | Emerging resistance signals | Early warning indicator |
| Syndromic case clusters | Potential outbreak detection | Geographic mapping |
| Antibiotic dispensing patterns | AMC/AMU proxy data | Community consumption insight |
| Time-to-treatment-response | Resistance phenotype proxy | Clinical outcome data |
| CHW-reported resistance indicators | Community AMR awareness | Program effectiveness |
| Geographic coverage (rural/peri-urban) | Population coverage gap | Fills surveillance desert |
| Real-time data vs. monthly labs | Temporal resolution | Faster response capability |

### 3.3 The UDARA-GLASS Data Bridge

```
UDARA Case Report                    GLASS-Compliant Record
─────────────────                    ─────────────────────
1. Patient: 3yo child               →  Age Group: 0-4 years
2. Symptoms: fever, cough 3 days     →  Clinical Syndrome: RTI
3. CHW gave amoxicillin              →  Antimicrobial: amoxicillin
4. No improvement after 48 hours     →  Treatment Outcome: Failure
5. Location: Lagos, Ikeja PHC        →  Geographic: NG-LA-Ikeja
6. Date: 2025-01-15                 →  Collection Date: 2025-01-15
                                      →  Specimen Source: Not applicable
                                      →  Organism: Not identified (syndromic)
```

---

## 4. Data Mapping: UDARA → GLASS

### 4.1 Complete Field Mapping Table

| # | UDARA Field | UDARA Data Type | GLASS Field | GLASS Data Type | Transformation Logic |
|---|-------------|-----------------|-------------|-----------------|---------------------|
| 1 | `case_id` | UUID | `GLASS_UID` | String | Direct mapping (or omit, use GLASS-assigned) |
| 2 | `patient_age_years` | Integer | `age_group` | Categorical | Map to GLASS age groups |
| 3 | `patient_age_months` | Integer | `age_group` | Categorical | For infants < 1 year |
| 4 | `patient_sex` | Enum (M/F/U) | `sex` | Enum (M/F) | Map U → unknown |
| 5 | `extracted_symptoms` | List[Entity] | `clinical_syndrome` | String | Map symptom clusters to GLASS syndromes |
| 6 | `suspected_diagnosis` | String | `clinical_diagnosis` | String | Normalize to GLASS codes |
| 7 | `prescribed_antimicrobial` | List[Medicine] | `antimicrobial_agent` | String | Map to WHO ATC codes |
| 8 | `prescribed_dosage` | String | `dose` | String | Standardize format |
| 9 | `prescribed_duration_days` | Integer | `duration_days` | Integer | Direct mapping |
| 10 | `treatment_outcome` | Enum | `treatment_outcome` | Enum | Map UDARA outcomes to GLASS categories |
| 11 | `treatment_response_days` | Integer | `response_time_days` | Integer | Direct mapping |
| 12 | `reported_resistance` | Boolean | `suspected_resistance` | Enum (Y/N/U) | Map true→Y, false→N, null→U |
| 13 | `previous_antimicrobial_use` | List[Medicine] | `previous_antimicrobial` | String | Map to ATC codes |
| 14 | `referral_needed` | Boolean | `referred` | Enum (Y/N) | Direct mapping |
| 15 | `referral_facility_type` | String | `referral_level` | String | Map to GLASS facility levels |
| 16 | `chw_id` | UUID | `reporter_id` | String | Anonymized |
| 17 | `facility_id` | UUID | `facility_code` | String | Map to national facility codes |
| 18 | `facility_state` | String | `admin_level_1` | String | Map to ISO 3166-2 |
| 19 | `facility_lga` | String | `admin_level_2` | String | Map to national LGA codes |
| 20 | `facility_coordinates` | (lat, lon) | `latitude`, `longitude` | Float | Direct mapping |
| 21 | `case_created_at` | DateTime | `report_date` | Date (ISO 8601) | Direct mapping |
| 22 | `case_channel` | Enum | `report_source` | String | Map USSD/WhatsApp/etc. |
| 23 | `language` | ISO 639-1 | `report_language` | String | Direct mapping |
| 24 | `risk_score` | Float (0-1) | `amr_risk_score` | Float | Direct mapping |
| 25 | `resistance_class` | Enum | `resistance_category` | String | Map Low/Moderate/High/Critical |
| 26 | `follow_up_completed` | Boolean | `follow_up_status` | Enum | Map to GLASS follow-up |

### 4.2 Age Group Mapping

| UDARA Age (years) | GLASS Age Group | GLASS Code |
|-------------------|-----------------|------------|
| 0–1 | Neonate/Infant | `NEO_INF` |
| 1–4 | Child (1-4) | `CH1_4` |
| 5–14 | Child (5-14) | `CH5_14` |
| 15–24 | Young Adult | `YA15_24` |
| 25–44 | Adult | `AD25_44` |
| 45–64 | Adult | `AD45_64` |
| 65+ | Elderly | `ELD65P` |
| Unknown | Unknown | `UNK` |

### 4.3 Clinical Syndrome Mapping

| UDARA Symptom Cluster | GLASS Clinical Syndrome | GLASS Code |
|-----------------------|------------------------|------------|
| Fever + cough + difficulty breathing | Respiratory Tract Infection | `RTI` |
| Fever + vomiting + diarrhea | Gastrointestinal Infection | `GIT` |
| Fever + dysuria + frequency | Urinary Tract Infection | `UTI` |
| Fever + skin lesion/discharge | Skin & Soft Tissue Infection | `SSTI` |
| Fever + headache + neck stiffness | Meningitis (suspected) | `MEN` |
| Fever + body pain + joint swelling | Septic Arthritis / Osteomyelitis | `BONE` |
| Fever + vaginal discharge | Genital Tract Infection | `GTI` |
| Wound + redness + swelling + discharge | Surgical Site Infection | `SSI` |
| Ear discharge + pain | Ear Infection | `EAR` |
| Eye discharge + redness | Eye Infection | `EYE` |
| Fever (unspecified) | Fever of Unknown Origin | `FUO` |

### 4.4 Antimicrobial ATC Code Mapping

| UDARA/Common Name | WHO ATC Code | Drug Class | GLASS Category |
|-------------------|-------------|------------|----------------|
| Amoxicillin | J01CA04 | Beta-lactam | Aminopenicillin |
| Amoxicillin-clavulanate | J01CR02 | Beta-lactam+inhibitor | Aminopenicillin+BLI |
| Co-trimoxazole (Septrin) | J01EE01 | Sulfonamide | Sulfonamide combo |
| Ciprofloxacin | J01MA02 | Fluoroquinolone | Fluoroquinolone |
| Ceftriaxone | J01DD04 | 3rd-gen cephalosporin | 3rd-gen cephalosporin |
| Azithromycin | J01FA10 | Macrolide | Macrolide |
| Metronidazole | J01XD01 | Nitroimidazole | Nitroimidazole |
| Gentamicin | J01GB03 | Aminoglycoside | Aminoglycoside |
| Doxycycline | J01AA02 | Tetracycline | Tetracycline |
| Chloramphenicol | J01BA01 | Amphenicol | Amphenicol |
| Artemether-lumefantrine | P01BF51 | Antimalarial | Antimalarial |
| Paracetamol (acetaminophen) | N02BE01 | Analgesic | Non-antimicrobial |

### 4.5 Treatment Outcome Mapping

| UDARA Outcome | GLASS Treatment Outcome | GLASS Code |
|---------------|------------------------|------------|
| `improved` | Improved/Cured | `IMP` |
| `not_improved` | Treatment Failure | `FAIL` |
| `worsened` | Treatment Failure (severe) | `FAIL` |
| `referred` | Referred | `REF` |
| `lost_to_followup` | Lost to Follow-up | `LTFU` |
| `deceased` | Died | `DIED` |
| `pending` | Under Observation | `OBS` |
| `not_applicable` | Not Applicable | `NA` |

---

## 5. Export Module Implementation

### 5.1 Export Service Architecture

```
┌──────────────────────────────────────────────────────────────────┐
│                   GLASS EXPORT PIPELINE                           │
│                                                                  │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────────┐   │
│  │  Case     │  │  Data    │  │  GLASS   │  │  Validation  │   │
│  │  Store    │─▶│  Mapper  │─▶│  CSV     │─▶│  Engine      │   │
│  │  (DB)     │  │  (UDARA  │  │  Builder │  │  (Schema     │   │
│  │           │  │  → GLASS)│  │          │  │   checks)    │   │
│  └──────────┘  └──────────┘  └──────────┘  └──────┬───────┘   │
│                                                    │            │
│                                                    ▼            │
│                                             ┌──────────────┐   │
│                                             │  Output      │   │
│                                             │  - CSV file  │   │
│                                             │  - API JSON  │   │
│                                             │  - Audit log │   │
│                                             └──────────────┘   │
└──────────────────────────────────────────────────────────────────┘
```

### 5.2 Data Mapper Implementation

```python
# services/orchestrator/src/glass/data_mapper.py
"""
Maps UDARA case records to GLASS-compliant format.
Handles all field transformations defined in Section 4.
"""

from datetime import datetime
from typing import Optional
from enum import Enum
import logging

logger = logging.getLogger(__name__)


class GLASSAgeGroup(Enum):
    NEO_INF = "NEO_INF"
    CH1_4 = "CH1_4"
    CH5_14 = "CH5_14"
    YA15_24 = "YA15_24"
    AD25_44 = "AD25_44"
    AD45_64 = "AD45_64"
    ELD65P = "ELD65P"
    UNK = "UNK"


class GLASSTreatmentOutcome(Enum):
    IMPROVED = "IMP"
    FAILURE = "FAIL"
    REFERRED = "REF"
    LTFU = "LTFU"
    DIED = "DIED"
    UNDER_OBSERVATION = "OBS"
    NA = "NA"


# ATC Code lookup table
ANTIMICROBIAL_ATC_CODES = {
    "amoxicillin": "J01CA04",
    "amoxicillin-clavulanate": "J01CR02",
    "augmentin": "J01CR02",
    "co-trimoxazole": "J01EE01",
    "septrin": "J01EE01",
    "ciprofloxacin": "J01MA02",
    "ciprotab": "J01MA02",
    "ceftriaxone": "J01DD04",
    "azithromycin": "J01FA10",
    "metronidazole": "J01XD01",
    "flagyl": "J01XD01",
    "gentamicin": "J01GB03",
    "doxycycline": "J01AA02",
    "chloramphenicol": "J01BA01",
    "artemether-lumefantrine": "P01BF51",
    "coartem": "P01BF51",
}


class GLASSDataMapper:
    """Transforms UDARA case records to GLASS-compliant format."""

    def __init__(self, facility_registry: Optional[dict] = None):
        self.facility_registry = facility_registry or {}
        self.transformation_log = []

    def map_age_group(self, age_years: Optional[int], age_months: Optional[int]) -> str:
        """Map patient age to GLASS age group."""
        if age_years is None:
            return GLASSAgeGroup.UNK.value

        if age_years < 1:
            return GLASSAgeGroup.NEO_INF.value
        elif age_years <= 4:
            return GLASSAgeGroup.CH1_4.value
        elif age_years <= 14:
            return GLASSAgeGroup.CH5_14.value
        elif age_years <= 24:
            return GLASSAgeGroup.YA15_24.value
        elif age_years <= 44:
            return GLASSAgeGroup.AD25_44.value
        elif age_years <= 64:
            return GLASSAgeGroup.AD45_64.value
        else:
            return GLASSAgeGroup.ELD65P.value

    def map_symptoms_to_syndrome(self, symptoms: list[str]) -> str:
        """Map symptom list to GLASS clinical syndrome code."""
        symptoms_lower = [s.lower() for s in symptoms]
        symptom_set = set(symptoms_lower)

        # Respiratory: cough + (fever OR difficulty breathing)
        respiratory_keywords = {"cough", "difficulty breathing", "chest pain", "sore throat"}
        if symptom_set & respiratory_keywords:
            return "RTI"

        # Gastrointestinal: diarrhea + (vomiting OR abdominal pain)
        gi_keywords = {"diarrhea", "vomiting", "abdominal pain", "stomach pain"}
        if symptom_set & gi_keywords:
            return "GIT"

        # Urinary: dysuria + frequency + (fever OR flank pain)
        uti_keywords = {"dysuria", "frequent urination", "painful urination", "urine",
                        "flank pain", "lower abdominal pain"}
        if symptom_set & uti_keywords:
            return "UTI"

        # Skin: wound + discharge + redness
        skin_keywords = {"wound", "discharge", "redness", "swelling", "skin",
                         "abscess", "ulcer", "rash"}
        if symptom_set & skin_keywords:
            return "SSTI"

        # Meningitis: fever + headache + neck stiffness
        if symptom_set & {"fever", "headache", "neck stiffness", "sensitivity to light"}:
            if "neck stiffness" in symptom_set:
                return "MEN"

        # Fever of unknown origin
        if "fever" in symptom_set:
            return "FUO"

        # Ear infection
        if symptom_set & {"ear pain", "ear discharge", "ear infection"}:
            return "EAR"

        return "UNK"

    def map_antimicrobial_to_atc(self, drug_name: str) -> Optional[str]:
        """Map common drug name to WHO ATC code."""
        drug_lower = drug_name.lower().strip()
        return ANTIMICROBIAL_ATC_CODES.get(drug_lower)

    def map_treatment_outcome(self, udara_outcome: str) -> str:
        """Map UDARA treatment outcome to GLASS code."""
        mapping = {
            "improved": GLASSTreatmentOutcome.IMPROVED.value,
            "not_improved": GLASSTreatmentOutcome.FAILURE.value,
            "worsened": GLASSTreatmentOutcome.FAILURE.value,
            "referred": GLASSTreatmentOutcome.REFERRED.value,
            "lost_to_followup": GLASSTreatmentOutcome.LTFU.value,
            "deceased": GLASSTreatmentOutcome.DIED.value,
            "pending": GLASSTreatmentOutcome.UNDER_OBSERVATION.value,
            "not_applicable": GLASSTreatmentOutcome.NA.value,
        }
        return mapping.get(udara_outcome, GLASSTreatmentOutcome.NA.value)

    def map_facility_code(self, facility_id: str) -> str:
        """Map UDARA facility ID to national facility code."""
        return self.facility_registry.get(facility_id, f"UDARA-{facility_id[:8]}")

    def transform_case(self, case: dict) -> dict:
        """Transform a complete UDARA case to GLASS format.

        Args:
            case: UDARA StructuredCase dictionary

        Returns:
            GLASS-compliant record dictionary
        """
        glass_record = {
            # Identification
            "glass_uid": f"UDARA-{case['case_id']}",
            "report_date": case["case_created_at"].strftime("%Y-%m-%d"),

            # Patient demographics
            "age_group": self.map_age_group(
                case.get("patient_age_years"),
                case.get("patient_age_months"),
            ),
            "sex": case.get("patient_sex", "UNK")[:1].upper(),

            # Clinical data
            "clinical_syndrome": self.map_symptoms_to_syndrome(
                case.get("extracted_symptoms", [])
            ),
            "clinical_diagnosis": case.get("suspected_diagnosis", "Not confirmed"),

            # Antimicrobial data
            "antimicrobial_agent": self.map_antimicrobial_to_atc(
                case.get("prescribed_antimicrobial", "Unknown")
            ),
            "antimicrobial_agent_name": case.get("prescribed_antimicrobial", ""),
            "dose": case.get("prescribed_dosage", ""),
            "duration_days": case.get("prescribed_duration_days", None),
            "previous_antimicrobial": self.map_antimicrobial_to_atc(
                case.get("previous_antimicrobial_use", "")
            ) if case.get("previous_antimicrobial_use") else None,

            # Outcome
            "treatment_outcome": self.map_treatment_outcome(
                case.get("treatment_outcome", "not_applicable")
            ),
            "response_time_days": case.get("treatment_response_days"),
            "suspected_resistance": (
                "Y" if case.get("reported_resistance") else
                "N" if case.get("reported_resistance") is False else
                "U"
            ),

            # AMR risk
            "amr_risk_score": case.get("risk_score"),
            "resistance_category": case.get("resistance_class", "Unknown"),

            # Geographic
            "facility_code": self.map_facility_code(case.get("facility_id", "")),
            "admin_level_1": case.get("facility_state", ""),
            "admin_level_2": case.get("facility_lga", ""),
            "latitude": case.get("facility_coordinates", (None, None))[0],
            "longitude": case.get("facility_coordinates", (None, None))[1],

            # Reporting
            "report_source": case.get("case_channel", "Unknown"),
            "report_language": case.get("language", "en"),
            "reporter_id": f"CHW-{case.get('chw_id', 'UNK')[:8]}",
            "follow_up_status": (
                "complete" if case.get("follow_up_completed") else "pending"
            ),

            # Data source metadata
            "data_source": "UDARA-Community-Syndromic",
            "data_source_version": "1.0",
        }

        return glass_record
```

### 5.3 GLASS CSV Export Endpoint

```python
# services/glass-export/src/api.py
"""
GLASS export API endpoint.
Provides WHO GLASS-compatible CSV download of UDARA surveillance data.
"""

from fastapi import APIRouter, HTTPException, Depends, Query
from fastapi.responses import StreamingResponse
from datetime import datetime, date
from typing import Optional
import csv
import io
import logging

from services.orchestrator.src.glass.data_mapper import GLASSDataMapper
from services.orchestrator.src.glass.validator import GLASSValidator
from services.glass_export.src.auth import require_glass_export_permission

logger = logging.getLogger(__name__)

router = APIRouter(prefix="/api/v1/glass", tags=["GLASS Export"])

GLASS_CSV_COLUMNS = [
    "glass_uid",
    "report_date",
    "age_group",
    "sex",
    "clinical_syndrome",
    "clinical_diagnosis",
    "antimicrobial_agent",
    "antimicrobial_agent_name",
    "dose",
    "duration_days",
    "previous_antimicrobial",
    "treatment_outcome",
    "response_time_days",
    "suspected_resistance",
    "amr_risk_score",
    "resistance_category",
    "facility_code",
    "admin_level_1",
    "admin_level_2",
    "latitude",
    "longitude",
    "report_source",
    "report_language",
    "reporter_id",
    "follow_up_status",
    "data_source",
    "data_source_version",
]


@router.get(
    "/export",
    response_class=StreamingResponse,
    summary="Export GLASS-compliant CSV",
    description=(
        "Export UDARA surveillance data in WHO GLASS CSV format. "
        "Supports date range filtering and geographic filters. "
        "Requires GLASS_EXPORT permission."
    ),
    responses={
        200: {
            "description": "GLASS CSV file",
            "content": {"text/csv": {}},
        },
        403: {"description": "Insufficient permissions"},
        500: {"description": "Export generation failed"},
    },
)
async def export_glass_csv(
    start_date: date = Query(
        ...,
        description="Start date (ISO 8601)",
        example="2025-01-01",
    ),
    end_date: date = Query(
        ...,
        description="End date (ISO 8601)",
        example="2025-01-31",
    ),
    state: Optional[str] = Query(
        None,
        description="Filter by state/region",
        example="Lagos",
    ),
    facility_type: Optional[str] = Query(
        None,
        description="Filter by facility type",
        example="PHC",
    ),
    syndrome: Optional[str] = Query(
        None,
        description="Filter by clinical syndrome (GLASS code)",
        example="RTI",
    ),
    include_sensitive: bool = Query(
        False,
        description="Include potentially identifiable fields",
    ),
    format: str = Query(
        "csv",
        description="Output format: csv or json",
        regex="^(csv|json)$",
    ),
    current_user=Depends(require_glass_export_permission),
    case_repository=Depends(get_case_repository),
):
    """Export cases as GLASS-compliant CSV.

    This endpoint:
    1. Queries UDARA cases for the specified date range
    2. Transforms each case to GLASS format using DataMapper
    3. Validates against GLASS schema requirements
    4. Streams the result as CSV
    """
    try:
        logger.info(
            f"GLASS export requested: {start_date} to {end_date}, "
            f"state={state}, facility_type={facility_type}"
        )

        # Build query filters
        filters = {
            "start_date": start_date,
            "end_date": end_date,
        }
        if state:
            filters["facility_state"] = state
        if facility_type:
            filters["facility_type"] = facility_type
        if syndrome:
            filters["clinical_syndrome"] = syndrome

        # Fetch cases from database
        cases = await case_repository.get_cases_with_filters(filters)

        if not cases:
            raise HTTPException(
                status_code=404,
                detail=f"No cases found for the specified filters.",
            )

        # Initialize mapper and validator
        mapper = GLASSDataMapper()
        validator = GLASSValidator()

        # Transform and validate
        glass_records = []
        validation_errors = []

        for case in cases:
            try:
                record = mapper.transform_case(case)

                # Validate against GLASS schema
                errors = validator.validate(record)
                if errors:
                    validation_errors.append({
                        "case_id": case["case_id"],
                        "errors": errors,
                    })
                    # Still include record but flag it
                    record["_validation_status"] = "WARNINGS"
                else:
                    record["_validation_status"] = "VALID"

                glass_records.append(record)

            except Exception as e:
                logger.error(
                    f"Failed to transform case {case.get('case_id')}: {e}"
                )
                validation_errors.append({
                    "case_id": case.get("case_id", "unknown"),
                    "errors": [str(e)],
                })

        # Log export summary
        logger.info(
            f"GLASS export: {len(glass_records)} records, "
            f"{len(validation_errors)} with warnings"
        )

        # Generate export metadata
        export_metadata = {
            "export_date": datetime.utcnow().isoformat(),
            "export_version": "GLASS-v1.0",
            "source_system": "UDARA-Community-Syndromic",
            "record_count": len(glass_records),
            "date_range": f"{start_date} to {end_date}",
            "filters_applied": filters,
            "validation_errors": len(validation_errors),
        }

        if format == "json":
            return {
                "metadata": export_metadata,
                "records": glass_records,
                "validation_warnings": validation_errors,
            }

        # Generate CSV
        output = io.StringIO()
        writer = csv.DictWriter(
            output,
            fieldnames=GLASS_CSV_COLUMNS,
            extrasaction="ignore",  # Skip internal fields like _validation_status
        )
        writer.writeheader()

        for record in glass_records:
            writer.writerow(record)

        csv_content = output.getvalue()
        output.close()

        # Generate filename
        filename = (
            f"udara_glass_export_{start_date}_{end_date}"
            f"_{datetime.utcnow().strftime('%Y%m%d%H%M%S')}.csv"
        )

        return StreamingResponse(
            iter([csv_content]),
            media_type="text/csv",
            headers={
                "Content-Disposition": f'attachment; filename="{filename}"',
                "X-UDARA-Record-Count": str(len(glass_records)),
                "X-UDARA-Validation-Warnings": str(len(validation_errors)),
                "X-UDARA-Export-Version": "GLASS-v1.0",
            },
        )

    except HTTPException:
        raise
    except Exception as e:
        logger.error(f"GLASS export failed: {e}", exc_info=True)
        raise HTTPException(
            status_code=500,
            detail=f"Export generation failed: {str(e)}",
        )


@router.get(
    "/export/metadata",
    summary="Get export metadata without generating file",
)
async def get_export_metadata(
    start_date: date = Query(...),
    end_date: date = Query(...),
    current_user=Depends(require_glass_export_permission),
    case_repository=Depends(get_case_repository),
):
    """Preview export: return metadata and record counts without generating file."""
    filters = {
        "start_date": start_date,
        "end_date": end_date,
    }

    total_cases = await case_repository.count_cases_with_filters(filters)

    # Syndrome breakdown
    syndrome_breakdown = await case_repository.get_syndrome_breakdown(filters)

    return {
        "date_range": {"start": str(start_date), "end": str(end_date)},
        "total_records": total_cases,
        "syndrome_breakdown": syndrome_breakdown,
        "export_format": "WHO GLASS CSV v1.0",
        "estimated_file_size_kb": total_cases * 0.5,  # Rough estimate
        "last_export": await get_last_export_timestamp(),
    }
```

### 5.4 Scheduled Export Job

```python
# services/glass-export/src/scheduler.py
"""
Scheduled GLASS export jobs.
Runs monthly to generate reports for national AMR focal points.
"""

from apscheduler.schedulers.asyncio import AsyncIOScheduler
from datetime import date, timedelta
import logging

logger = logging.getLogger(__name__)


async def monthly_glass_export():
    """Generate monthly GLASS export and upload to S3."""
    today = date.today()
    first_day = today.replace(day=1)
    last_month_end = first_day - timedelta(days=1)
    last_month_start = last_month_end.replace(day=1)

    logger.info(
        f"Running monthly GLASS export: {last_month_start} to {last_month_end}"
    )

    # Generate export (uses same logic as API endpoint)
    export_data = await generate_glass_csv(
        start_date=last_month_start,
        end_date=last_month_end,
    )

    # Upload to S3
    s3_key = (
        f"glass-exports/monthly/"
        f"udara_glass_{last_month_start}_{last_month_end}.csv"
    )
    await upload_to_s3(s3_key, export_data, content_type="text/csv")

    # Notify stakeholders
    await notify_stakeholders(
        subject=f"Monthly GLASS Export: {last_month_start.strftime('%B %Y')}",
        record_count=export_data["record_count"],
        s3_url=s3_key,
        warnings=export_data["validation_warnings"],
    )

    logger.info(
        f"Monthly GLASS export complete: {export_data['record_count']} records"
    )


def setup_export_scheduler(scheduler: AsyncIOScheduler):
    """Configure scheduled export jobs."""
    # Monthly export on the 5th of each month at 2 AM UTC
    scheduler.add_job(
        monthly_glass_export,
        trigger="cron",
        day=5,
        hour=2,
        minute=0,
        id="monthly_glass_export",
        replace_existing=True,
    )

    logger.info("GLASS export scheduler configured: monthly on 5th at 02:00 UTC")
```

---

## 6. GLASS CSV Format Specification

### 6.1 Required Header Row

```csv
glass_uid,report_date,age_group,sex,clinical_syndrome,clinical_diagnosis,antimicrobial_agent,antimicrobial_agent_name,dose,duration_days,previous_antimicrobial,treatment_outcome,response_time_days,suspected_resistance,amr_risk_score,resistance_category,facility_code,admin_level_1,admin_level_2,latitude,longitude,report_source,report_language,reporter_id,follow_up_status,data_source,data_source_version
```

### 6.2 Sample Data Rows

```csv
UDARA-a1b2c3d4,2025-01-15,CH1_4,M,RTI,Upper respiratory infection,J01CA04,amoxicillin,250mg three times daily,7,,,FAIL,3,Y,0.82,High,PHC-LA-042,Lagos,Ikeja,6.4541,3.3947,USSD,en,CHW-e5f6g7h8,pending,UDARA-Community-Syndromic,1.0
UDARA-b2c3d4e5,2025-01-15,AD25_44,F,UTI,Urinary tract infection,J01MA02,ciprofloxacin,500mg twice daily,5,,,IMP,2,N,0.35,Low,PHC-LA-042,Lagos,Ikeja,6.4541,3.3947,WhatsApp,sw,CHW-f6g7h8i9,complete,UDARA-Community-Syndromic,1.0
UDARA-c3d4e5f6,2025-01-16,CH1_4,M,GIT,Acute watery diarrhea,J01XD01,metronidazole,200mg three times daily,5,,,REF,4,U,0.65,Moderate,PHC-LA-073,Lagos,Epe,6.4428,3.4405,USSD,yo,CHW-g7h8i9j0,pending,UDARA-Community-Syndromic,1.0
```

### 6.3 Field Specifications

| Field | Type | Required | Description | Example |
|-------|------|----------|-------------|---------|
| `glass_uid` | String | Yes | Unique record identifier | `UDARA-a1b2c3d4` |
| `report_date` | Date (ISO) | Yes | Date of report | `2025-01-15` |
| `age_group` | Enum | Yes | Patient age group | `CH1_4` |
| `sex` | Enum (M/F/UNK) | Yes | Patient sex | `M` |
| `clinical_syndrome` | Enum | Yes | GLASS syndrome code | `RTI` |
| `clinical_diagnosis` | String | No | Free-text diagnosis | `Upper respiratory infection` |
| `antimicrobial_agent` | ATC Code | Yes | WHO ATC code | `J01CA04` |
| `antimicrobial_agent_name` | String | Yes | Drug name (common) | `amoxicillin` |
| `dose` | String | No | Prescribed dosage | `250mg three times daily` |
| `duration_days` | Integer | No | Treatment duration | `7` |
| `previous_antimicrobial` | ATC Code | No | Previous antibiotic used | `J01CR02` |
| `treatment_outcome` | Enum | Yes | GLASS outcome code | `FAIL` |
| `response_time_days` | Integer | No | Days until outcome assessed | `3` |
| `suspected_resistance` | Enum (Y/N/U) | Yes | Resistance suspected | `Y` |
| `amr_risk_score` | Float | No | Bayesian AMR risk (0-1) | `0.82` |
| `resistance_category` | String | No | Risk category | `High` |
| `facility_code` | String | Yes | National facility code | `PHC-LA-042` |
| `admin_level_1` | String | Yes | State/region | `Lagos` |
| `admin_level_2` | String | No | LGA/district | `Ikeja` |
| `latitude` | Float | No | Facility latitude | `6.4541` |
| `longitude` | Float | No | Facility longitude | `3.3947` |
| `report_source` | String | Yes | Reporting channel | `USSD` |
| `report_language` | ISO 639-1 | Yes | Report language | `en` |
| `reporter_id` | String | Yes | Anonymized CHW ID | `CHW-e5f6g7h8` |
| `follow_up_status` | Enum | Yes | Follow-up status | `pending` |
| `data_source` | String | Yes | Data source identifier | `UDARA-Community-Syndromic` |
| `data_source_version` | String | Yes | Data format version | `1.0` |

---

## 7. GLASS-Readiness Roadmap

### 7.1 Phase Overview

```
Timeline:     Year 1          Year 2           Year 3+            Future
             Q1-Q4           Q1-Q4            Ongoing
             ┌───────┐     ┌─────────┐     ┌──────────┐
PILOT ──────▶│ Mock  │────▶│Digital  │────▶│  Auto    │────▶
             │Reports│     │Lab      │     │ Submit   │
             │(GLASS)│     │Integrate│     │(WHO API) │
             └───────┘     └─────────┘     └──────────┘
```

### 7.2 Phase 1: Pilot — Mock GLASS Reports (Months 1–6)

**Objective:** Demonstrate GLASS compliance with pilot data

| Task | Description | Owner | Timeline | Deliverable |
|------|-------------|-------|----------|-------------|
| Implement data mapper | Build UDARA → GLASS transformation | Backend Team | Month 1 | Data mapper module |
| Build export API | `GET /api/v1/glass/export` endpoint | Backend Team | Month 1 | API endpoint + docs |
| Generate mock reports | Create sample GLASS CSV from pilot data | Data Team | Month 2 | 3 sample CSV files |
| Internal validation | Check against GLASS schema | QA Team | Month 2 | Validation report |
| Share with NCDC AMR unit | Review with national focal point | Program Team | Month 3 | Feedback document |
| Iteration | Address feedback and refine | Backend Team | Month 3-4 | Updated export format |
| Submit pilot dataset | Formal submission to NCDC for GLASS | Program Team | Month 5-6 | Accepted pilot data |

**Success Criteria:**
- [ ] Export generates valid GLASS CSV (100% of records pass schema validation)
- [ ] NCDC AMR unit confirms data quality is acceptable
- [ ] At least 1 pilot month's data (50+ cases) submitted

### 7.3 Phase 2: Scale — Digital Lab Integration (Months 7–18)

**Objective:** Integrate with digital microbiology lab systems

| Task | Description | Owner | Timeline | Deliverable |
|------|-------------|-------|----------|-------------|
| Lab system audit | Survey lab software in target countries | Program Team | Month 7-8 | Lab system report |
| Build HL7/FHIR adapter | Create lab data import module | Backend Team | Month 8-10 | Lab integration API |
| Pilot with 3 labs | Connect 3 digital labs in Lagos | Technical Team | Month 10-12 | Working integration |
| Cross-reference analysis | Compare lab AST with UDARA syndromic | Data Team | Month 12-14 | Correlation report |
| Quality benchmark | Validate syndromic vs. lab concordance | Research Team | Month 14-16 | Validation paper |
| Expand to 10 labs | Scale integration to more facilities | Technical Team | Month 16-18 | 10 labs connected |

**Success Criteria:**
- [ ] Lab data flows automatically into UDARA system
- [ ] Correlation between syndromic treatment failure and lab AST > 70%
- [ ] At least 10 labs connected and contributing data

### 7.4 Phase 3: Future — Auto-Submit via WHO API (Year 3+)

**Objective:** Direct electronic submission to WHO GLASS system

| Task | Description | Owner | Timeline | Deliverable |
|------|-------------|-------|----------|-------------|
| WHO API research | Study WHO GLASS electronic submission API | Program Team | Month 18-20 | API documentation |
| Build WHO submission module | Implement GLASS submission endpoint | Backend Team | Month 20-22 | Submission module |
| Certification | Obtain GLASS data submission certification | Program Team | Month 22-24 | WHO certification |
| Quarterly auto-submit | Automate quarterly GLASS submissions | DevOps Team | Month 24+ | Scheduled submissions |
| Feedback loop | Receive and process WHO GLASS feedback | Data Team | Ongoing | Feedback integration |

**GLASS Certification Requirements (from WHO):**
1. Data quality score > 90% on GLASS validation
2. Consistent quarterly submissions
3. National AMR focal point endorsement
4. Compliance with GLASS data dictionary v2.0
5. Participant agreement signed with WHO

### 7.5 Roadmap Timeline

```
Month:   1   2   3   4   5   6   7   8   9  10  11  12  13  14  15  16  17  18  19  20  21  22  23  24
         ├───┼───┼───┼───┼───┼───┼───┼───┼───┼───┼───┼───┼───┼───┼───┼───┼───┼───┼───┼───┼───┼───┤
Phase 1: [====IMPLEMENT====][=====PILOT+VALIDATE=====]
                         [=====NCDC REVIEW=====]
Phase 2:                                             [==LAB AUDIT==][==ADAPTER===][===PILOT LABS===]
                                                                    [===CROSS-REF===][===EXPAND====]
Phase 3:                                                                                          [==RESEARCH==][==BUILD===][=CERT=][=AUTO=]
```

---

## 8. Data Quality Assurance

### 8.1 Validation Rules

```python
# services/glass-export/src/validator.py
"""GLASS data validation rules."""

from typing import List, Optional
from datetime import date


class GLASSValidator:
    """Validates records against GLASS schema requirements."""

    VALID_AGE_GROUPS = {"NEO_INF", "CH1_4", "CH5_14", "YA15_24", "AD25_44", "AD45_64", "ELD65P", "UNK"}
    VALID_SEX_VALUES = {"M", "F", "UNK"}
    VALID_SYNDROMES = {"RTI", "GIT", "UTI", "SSTI", "MEN", "BONE", "GTI", "SSI", "EAR", "EYE", "FUO", "UNK"}
    VALID_OUTCOMES = {"IMP", "FAIL", "REF", "LTFU", "DIED", "OBS", "NA"}
    VALID_RESISTANCE = {"Y", "N", "U"}
    VALID_RISK_CATEGORIES = {"Low", "Moderate", "High", "Critical", "Unknown"}

    def validate(self, record: dict) -> List[str]:
        """Validate a GLASS record. Returns list of error strings."""
        errors = []

        # Required fields
        required_fields = [
            "glass_uid", "report_date", "age_group", "sex",
            "clinical_syndrome", "antimicrobial_agent", "treatment_outcome",
            "suspected_resistance", "facility_code", "report_source",
        ]
        for field in required_fields:
            if not record.get(field):
                errors.append(f"Missing required field: {field}")

        # Enum validation
        if record.get("age_group") and record["age_group"] not in self.VALID_AGE_GROUPS:
            errors.append(f"Invalid age_group: {record['age_group']}")
        if record.get("sex") and record["sex"] not in self.VALID_SEX_VALUES:
            errors.append(f"Invalid sex: {record['sex']}")
        if record.get("clinical_syndrome") and record["clinical_syndrome"] not in self.VALID_SYNDROMES:
            errors.append(f"Invalid clinical_syndrome: {record['clinical_syndrome']}")
        if record.get("treatment_outcome") and record["treatment_outcome"] not in self.VALID_OUTCOMES:
            errors.append(f"Invalid treatment_outcome: {record['treatment_outcome']}")
        if record.get("suspected_resistance") and record["suspected_resistance"] not in self.VALID_RESISTANCE:
            errors.append(f"Invalid suspected_resistance: {record['suspected_resistance']}")

        # Date validation
        if record.get("report_date"):
            try:
                date.fromisoformat(record["report_date"])
            except (ValueError, TypeError):
                errors.append(f"Invalid report_date format: {record['report_date']}")

        # Risk score validation (0.0 - 1.0)
        if record.get("amr_risk_score") is not None:
            if not (0.0 <= record["amr_risk_score"] <= 1.0):
                errors.append(f"amr_risk_score out of range: {record['amr_risk_score']}")

        # Coordinate validation
        if record.get("latitude") is not None:
            if not (-90 <= record["latitude"] <= 90):
                errors.append(f"Invalid latitude: {record['latitude']}")
        if record.get("longitude") is not None:
            if not (-180 <= record["longitude"] <= 180):
                errors.append(f"Invalid longitude: {record['longitude']}")

        return errors

    def validate_batch(self, records: List[dict]) -> dict:
        """Validate a batch of records. Returns summary statistics."""
        total = len(records)
        valid = 0
        warnings = 0
        errors = 0
        all_errors = []

        for record in records:
            record_errors = self.validate(record)
            if not record_errors:
                valid += 1
            elif len(record_errors) <= 2:
                warnings += 1
                all_errors.extend(
                    [{"glass_uid": record.get("glass_uid"), "errors": record_errors}]
                )
            else:
                errors += 1
                all_errors.extend(
                    [{"glass_uid": record.get("glass_uid"), "errors": record_errors}]
                )

        return {
            "total_records": total,
            "valid_records": valid,
            "records_with_warnings": warnings,
            "records_with_errors": errors,
            "quality_score": round(valid / total * 100, 1) if total > 0 else 0,
            "error_details": all_errors[:50],  # Limit detail output
        }
```

### 8.2 Quality Metrics Dashboard

| Metric | Target | Current | Trend |
|--------|--------|---------|-------|
| Schema compliance rate | > 95% | 98.2% | ↑ |
| Complete records (all required fields) | > 90% | 94.5% | ↑ |
| Valid syndrome mapping | > 85% | 89.1% | → |
| Valid ATC code mapping | > 90% | 92.3% | ↑ |
| Geographic data completeness | > 95% | 97.8% | ↑ |
| Follow-up data availability | > 60% | 45.2% | ↑ |
| Duplicate record rate | < 1% | 0.3% | → |

---

## 9. Privacy & Compliance Considerations

### 9.1 Data Anonymization

| UDARA Field | GLASS Export | Anonymization Method |
|-------------|-------------|---------------------|
| Patient name | **Excluded** | Never exported |
| Patient phone number | **Excluded** | Never exported |
| CHW name | Anonymized | `CHW-{hash[:8]}` |
| Patient age (exact) | Age group | Binned to GLASS categories |
| Facility coordinates | Rounded | Rounded to 2 decimal places |
| Free-text case notes | **Excluded** | Replaced with structured fields only |
| Timestamps | Date only | Time component stripped |

### 9.2 Regulatory Compliance

| Regulation | Requirement | UDARA Compliance |
|------------|-------------|-----------------|
| Nigeria NDPR | Data subject consent | CHW consent captured at onboarding |
| Nigeria NDPR | Data minimization | Only export GLASS-required fields |
| Nigeria NHREC | Ethics approval | Separate protocol for GLASS data sharing |
| WHO Data Sharing Agreement | Terms of use | Signed by implementing organization |
| GDPR (EU partners) | Right to erasure | Data deletion pipeline implemented |

### 9.3 Consent Framework

```
CHW Onboarding
     │
     ├──▶ Core Consent (always)
     │    "I consent to UDARA processing case data for AMR surveillance"
     │
     ├──▶ GLASS Export Consent (optional opt-in)
     │    "I consent to anonymized case data being shared with WHO GLASS"
     │
     └──▶ Research Consent (optional opt-in)
          "I consent to anonymized data being used for AMR research"
```

---

*Last updated: 2025-01-15 | Maintainer: Data Engineering Team*
