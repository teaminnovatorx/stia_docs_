# 02 — Three-Door Interface

> UDARA AI provides three distinct interface channels ("doors") to ensure every
> community health worker — regardless of device capability — can contribute AMR
> surveillance data.

---

## Table of Contents

- [Design Rationale](#design-rationale)
- [Door 1: USSD + SMS (Feature Phones)](#door-1-ussd--sms-feature-phones)
  - [Technology Stack](#technology-stack)
  - [USSD Menu State Machine](#ussd-menu-state-machine)
  - [Session Management](#session-management)
  - [SMS Notifications](#sms-notifications)
  - [Africa's Talking Integration](#africas-talking-integration)
  - [Code Examples](#code-examples)
- [Door 2: WhatsApp + Telegram + PWA (Smartphones)](#door-2-whatsapp--telegram--pwa-smartphones)
  - [WhatsApp Business API](#whatsapp-business-api)
  - [Telegram Bot](#telegram-bot)
  - [Progressive Web App (PWA)](#progressive-web-app-pwa)
  - [Features Unique to Door 2](#features-unique-to-door-2)
  - [Code Examples](#code-examples-1)
- [Door 3: Web Dashboard (Officials)](#door-3-web-dashboard-officials)
  - [Technology Stack](#technology-stack-1)
  - [Access Levels](#access-levels)
  - [Dashboard Components](#dashboard-components)
  - [ASCII Wireframes](#ascii-wireframes)
- [Channel Comparison Table](#channel-comparison-table)
- [ASCII Wireframes per Channel](#ascii-wireframes-per-channel)

---

## Design Rationale

### The Digital Divide in sub-Saharan Africa

```
┌─────────────────────────────────────────────────────────────────────────────┐
│              DEVICE LANDSCAPE — TARGET DEMOGRAPHIC                          │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  Feature Phones (Nokia 105, Itel, Tecno):      ~35% of CHWs                 │
│  ┌────────────────────────────────────────────────────────────────┐         │
│  │ ✗ No internet browser     ✗ No app store                       │         │
│  │ ✗ No touchscreen          ✗ No camera (usually)                │         │
│  │ ✓ USSD support (*code#)   ✓ SMS send/receive                   │         │
│  │ ✓ Phone calls             ✓ T9 text input                      │         │
│  └────────────────────────────────────────────────────────────────┘         │
│                          │                                                  │
│                          ▼                                                  │
│  Smartphones (Android, low-end):                 ~45% of CHWs               │
│  ┌────────────────────────────────────────────────────────────────┐         │
│  │ ✓ WhatsApp installed       ✓ Telegram installed                │         │
│  │ ✓ Camera (2-13 MP)        ✓ Microphone                         │         │
│  │ ✓ Intermittent 3G/4G      ✗ Limited storage (8-32 GB)          │         │
│  │ ✗ Browser may be slow      ✗ May not install new apps          │         │
│  └────────────────────────────────────────────────────────────────┘         │
│                          │                                                  │
│                          ▼                                                  │
│  Smartphones + Tablets (mid-range):               ~15% of CHWs              │
│  ┌────────────────────────────────────────────────────────────────┐         │
│  │ ✓ All messaging apps       ✓ PWA support (Chrome)              │         │
│  │ ✓ Decent camera            ✓ Wi-Fi at facility                 │         │
│  │ ✓ Chrome browser           ✓ Moderate storage (32-64 GB)       │         │
│  └────────────────────────────────────────────────────────────────┘         │
│                          │                                                  │
│                          ▼                                                  │
│  Desktop/Laptop (facility computers):            ~5% of users               │
│  ┌────────────────────────────────────────────────────────────────┐         │
│  │ ✓ Full web browser          ✓ Large screen                     │         │
│  │ ✓ Reliable internet (usually) ✓ Keyboard + mouse               │         │
│  │ ✓ Used by facility admin, state officials                      │         │
│  └────────────────────────────────────────────────────────────────┘         │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

### The Three-Door Strategy

Each "door" targets a specific device capability tier, ensuring **zero-exclusion**
— every CHW can participate regardless of their device.

```
  ┌──────────────────┐     ┌──────────────────┐     ┌──────────────────┐
  │   DOOR 1         │     │   DOOR 2         │     │   DOOR 3         │
  │   USSD + SMS     │     │   WhatsApp +     │     │   Web            │
  │                  │     │   Telegram + PWA │     │   Dashboard      │
  │   Feature phones │     │   Smartphones    │     │   Desktop        │
  └────────┬─────────┘     └────────┬─────────┘     └────────┬─────────┘
           │                        │                        │
           │                        │                        │
           ▼                        ▼                        ▼
  ┌─────────────────────────────────────────────────────────────────┐
  │                     SAME DATA MODEL                             │
  │                                                                 │
  │   All doors produce the same Report structure:                  │
  │   • CHW ID, Facility ID, Timestamp                              │
  │   • Symptoms, Duration, Drug, Outcome                           │
  │   • Source channel (ussd/whatsapp/telegram/pwa/web)             │
  │   • NER confidence, Language                                    │
  │                                                                 │
  │   All doors go through the same AI pipeline:                    │
  │   LID → NER → PII Redaction → Report Builder → Bayesian         │
  └─────────────────────────────────────────────────────────────────┘
```

---

## Door 1: USSD + SMS (Feature Phones)

### Technology Stack

| Component | Technology | Purpose |
|-----------|-----------|---------|
| USSD Gateway | Africa's Talking API | Receive/send USSD sessions |
| SMS Gateway | Africa's Talking API | Send SMS notifications |
| Session Store | Redis | USSD session state (120s TTL) |
| Backend | Edge FastAPI | Process USSD callbacks |
| Response Format | Plain text, max 160 chars | USSD screen limit |

### USSD Menu State Machine

USSD sessions are structured as a **state machine** where each menu option
transitions to the next state. The entire flow is designed to be completable
in under 90 seconds (within the 120s session timeout).

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                        USSD MENU STATE MACHINE                              │
│                     (*384# → Africa's Talking → Edge)                       │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  ┌─────────────────────────────────────┐                                    │
│  │         STATE: MAIN_MENU            │                                    │
│  │                                     │                                    │
│  │  UDARA AMR Surveillance             │                                    │
│  │  1. Report treatment outcome        │                                    │
│  │  2. Check resistance info           │                                    │
│  │  3. My reports                      │                                    │
│  │  4. Help                            │                                    │
│  │                                     │                                    │
│  └──────┬──────────┬──────────┬──────-─┘                                    │
│         │1         │2         │3                                            │
│         ▼          ▼          ▼                                             │
│  ┌──────────────┐ ┌──────────────┐ ┌──────────────┐                         │
│  │SYMPTOM_MENU  │ │RESISTANCE_   │ │MY_REPORTS    │                         │
│  │              │ │QUERY         │ │              │                         │
│  │Enter symptom:│ │              │ │Last 5 reports│                         │
│  │1.Fever       │ │Enter drug:   │ │displayed     │                         │
│  │2.Cough       │ │1.Amoxicillin │ │              │                         │
│  │3.Diarrhoea   │ │2.Cipro       │ │              │                         │
│  │4.Rash        │ │3.Metronidazole││              │                         │
│  │5.Other       │ │4.Co-trimoxazole││             │                         │
│  │              │ │              │ │              │                         │
│  └──────┬───────┘ └──────┬───────┘ └──────────────┘                         │
│         │                │                                                  │
│         ▼                ▼                                                  │
│  ┌──────────────┐ ┌──────────────┐                                          │
│  │AGE_MENU      │ │RESISTANCE_   │                                          │
│  │              │ │RESULT        │                                          │
│  │Patient age:  │ │              │                                          │
│  │1.<2 yrs      │ │Amoxicillin   │                                          │
│  │2.2-5 yrs     │ │resistance in │                                          │
│  │3.5-15 yrs    │ │your area:    │                                          │
│  │4.15-65 yrs   │ │35% (medium)  │                                          │
│  │5.>65 yrs     │ │              │                                          │
│  │              │ │Consider alt: │                                          │
│  └──────┬───────┘ │Ciprofloxacin │                                          │
│         │         └──────────────┘                                          │
│         ▼                                                                   │
│  ┌──────────────┐                                                           │
│  │DURATION_MENU │                                                           │
│  │              │                                                           │
│  │Illness       │                                                           │
│  │duration:     │                                                           │
│  │1.<3 days     │                                                           │
│  │2.3-7 days    │                                                           │
│  │3.7-14 days   │                                                           │
│  │4.>14 days    │                                                           │
│  │              │                                                           │
│  └──────┬───────┘                                                           │
│         │                                                                   │
│         ▼                                                                   │
│  ┌──────────────┐                                                           │
│  │DRUG_MENU     │                                                           │
│  │              │                                                           │
│  │Drug given:   │                                                           │
│  │1.Amoxicillin │                                                           │
│  │2.Ciprofloxacin│                                                          │
│  │3.Metronidazole│                                                          │
│  │4.Co-trimoxazole│                                                         │
│  │5.Azithromycin│                                                           │
│  │6.Other       │                                                           │
│  │7.None        │                                                           │
│  │              │                                                           │
│  └──────┬───────┘                                                           │
│         │                                                                   │
│         ▼                                                                   │
│  ┌──────────────┐                                                           │
│  │OUTCOME_MENU  │                                                           │
│  │              │                                                           │
│  │After drug:   │                                                           │
│  │1.Improved    │                                                           │
│  │2.No change   │                                                           │
│  │3.Worse       │                                                           │
│  │4.Side effects│                                                           │
│  │              │                                                           │
│  └──────┬───────┘                                                           │
│         │                                                                   │
│         ▼                                                                   │
│  ┌──────────────┐                                                           │
│  │CONFIRM_MENU  │                                                           │
│  │              │                                                           │
│  │Report:       │                                                           │
│  │Symptom:Fever │                                                           │
│  │Age:Adult     │                                                           │
│  │Dur:5 days    │                                                           │
│  │Drug:Amox     │                                                           │
│  │Out:No change │                                                           │
│  │              │                                                           │
│  │1.Confirm     │                                                           │
│  │2.Edit        │                                                           │
│  │3.Cancel      │                                                           │
│  └──────┬───────┘                                                           │
│         │1                                                                  │
│         ▼                                                                   │
│  ┌──────────────┐                                                           │
│  │  END         │                                                           │
│  │              │                                                           │
│  │ Thank you!   │                                                           │
│  │ Report saved.│                                                           │
│  │ Ref:UDR-1234 │                                                           │
│  │              │                                                           │
│  └──────────────┘                                                           │
│                                                                             │
│  CHARACTER COUNT CONSTRAINTS:                                               │
│  ┌────────────────────────────┬──────────┬──────────┐                       │
│  │ Menu State                 │ Max chars│ Used     │                       │
│  ├────────────────────────────┼──────────┼──────────┤                       │
│  │ MAIN_MENU                  │    160   │    98    │                       │
│  │ SYMPTOM_MENU               │    160   │    85    │                       │
│  │ AGE_MENU                   │    160   │    72    │                       │
│  │ DURATION_MENU              │    160   │    63    │                       │
│  │ DRUG_MENU                  │    160   │    90    │                       │
│  │ OUTCOME_MENU               │    160   │    62    │                       │
│  │ CONFIRM_MENU               │    160   │   132    │                       │
│  │ END                        │    160   │    42    │                       │
│  └────────────────────────────┴──────────┴──────────┘                       │
│                                                                             │
│  NOTE: If text exceeds 160 chars, it is truncated with "..." at end         │
│  and user is sent follow-up SMS with remaining content.                     │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

### Session Management

USSD sessions are stored in Redis with a 120-second TTL. When a session expires,
the CHW is returned to the main menu.

```python
# edge/app/services/session_service.py
"""USSD session management with Redis backend."""

from __future__ import annotations

import json
import logging
from dataclasses import dataclass, field
from enum import Enum
from typing import Any

import redis

logger = logging.getLogger(__name__)


class USSDState(str, Enum):
    """All possible states in the USSD menu state machine."""
    MAIN_MENU = "main_menu"
    SYMPTOM_MENU = "symptom_menu"
    AGE_MENU = "age_menu"
    DURATION_MENU = "duration_menu"
    DRUG_MENU = "drug_menu"
    OUTCOME_MENU = "outcome_menu"
    CONFIRM_MENU = "confirm_menu"
    RESISTANCE_QUERY = "resistance_query"
    RESISTANCE_RESULT = "resistance_result"
    MY_REPORTS = "my_reports"
    HELP = "help"
    END = "end"


@dataclass
class USSDSession:
    """Represents a single USSD session state."""
    session_id: str                 # Africa's Talking sessionId
    phone_number: str               # CHW's phone number
    state: USSDState = USSDState.MAIN_MENU
    data: dict[str, Any] = field(default_factory=dict)
    created_at: float = 0.0         # Unix timestamp
    last_activity: float = 0.0      # Unix timestamp
    menu_history: list[str] = field(default_factory=list)  # For "back" navigation

    @property
    def is_expired(self) -> bool:
        """Check if session has exceeded 120s TTL."""
        import time
        return (time.time() - self.last_activity) > 120.0

    def to_json(self) -> str:
        return json.dumps({
            "session_id": self.session_id,
            "phone_number": self.phone_number,
            "state": self.state.value,
            "data": self.data,
            "created_at": self.created_at,
            "last_activity": self.last_activity,
            "menu_history": self.menu_history,
        })

    @classmethod
    def from_json(cls, json_str: str) -> USSDSession:
        d = json.loads(json_str)
        d["state"] = USSDState(d["state"])
        return cls(**d)


class USSDSessionManager:
    """
    Manages USSD sessions with Redis as the backing store.
    
    Features:
    - 120s TTL (auto-expiry via Redis EX)
    - Concurrent session support (same phone, different sessions)
    - Session recovery on timeout
    - Memory-efficient serialization (msgpack in production)
    """
    
    SESSION_TTL = 120  # seconds (USSD session timeout)
    REDIS_PREFIX = "udara:ussd:session:"

    def __init__(self, redis_client: redis.Redis) -> None:
        self.redis = redis_client

    def create_session(
        self,
        session_id: str,
        phone_number: str,
    ) -> USSDSession:
        """Create a new USSD session."""
        import time
        session = USSDSession(
            session_id=session_id,
            phone_number=phone_number,
            created_at=time.time(),
            last_activity=time.time(),
        )
        self._save(session)
        logger.info("Created USSD session %s for %s", session_id, phone_number)
        return session

    def get_session(self, session_id: str) -> USSDSession | None:
        """Retrieve an existing session, or None if expired/missing."""
        key = f"{self.REDIS_PREFIX}{session_id}"
        data = self.redis.get(key)
        if data is None:
            logger.debug("USSD session %s not found (expired or missing)", session_id)
            return None
        
        session = USSDSession.from_json(data)
        
        # Double-check expiry (in case Redis TTL was extended)
        if session.is_expired:
            self.delete_session(session_id)
            logger.info("USSD session %s expired (TTL exceeded)", session_id)
            return None
        
        return session

    def update_session(
        self,
        session: USSDSession,
        new_state: USSDState | None = None,
        **extra_data: Any,
    ) -> USSDSession:
        """Update session state and/or data. Refreshes TTL."""
        import time
        
        if new_state is not None:
            session.menu_history.append(session.state.value)
            session.state = new_state
        
        session.data.update(extra_data)
        session.last_activity = time.time()
        
        self._save(session)
        return session

    def delete_session(self, session_id: str) -> None:
        """Manually delete a session."""
        key = f"{self.REDIS_PREFIX}{session_id}"
        self.redis.delete(key)

    def _save(self, session: USSDSession) -> None:
        """Persist session to Redis with TTL."""
        key = f"{self.REDIS_PREFIX}{session.session_id}"
        self.redis.setex(
            key,
            self.SESSION_TTL,
            session.to_json(),
        )

    def get_active_sessions_count(self) -> int:
        """Count currently active USSD sessions."""
        pattern = f"{self.REDIS_PREFIX}*"
        return len(self.redis.keys(pattern))
```

### SMS Notifications

SMS is used for confirmations, alerts, and follow-up when USSD text exceeds
160 characters.

| Trigger | SMS Template | Char Count |
|---------|-------------|------------|
| Report submitted | `"UDARA: Report UDR-{ref} saved. Thank you, {chw_name}!"` | ~55 |
| Treatment failure alert | `"UDARA ALERT: High resistance to {drug} in {area}. Consider {alt}."` | ~72 |
| Follow-up needed | `"UDARA: Please check on patient {initials}. Report outcome within 48h."` | ~72 |
| New model available | `"UDARA: System updated. New resistance data available. Dial *384#"` | ~68 |
| Session expired | `"UDARA: Your session timed out. Dial *384# to start a new report."` | ~64 |

### Africa's Talking Integration

```python
# edge/app/api/routes/ussd.py
"""USSD callback handler for Africa's Talking API."""

from __future__ import annotations

import logging
from fastapi import APIRouter, Request, Response
from pydantic import BaseModel, Field

from app.services.session_service import (
    USSDSessionManager,
    USSDState,
)

logger = logging.getLogger(__name__)
router = APIRouter(prefix="/api/v1/ussd", tags=["ussd"])


class USSDCallback(BaseModel):
    """Payload from Africa's Talking USSD callback."""
    sessionId: str
    serviceCode: str
    phoneNumber: str
    text: str              # User's cumulative input (all previous + current)
    networkCode: str       # Mobile network (e.g., "MTN_NG")


class USSDResponse(BaseModel):
    """Response format expected by Africa's Talking."""
    response: str


# Maps user input to next state and response text
MENU_RESPONSES: dict[USSDState, dict[str, tuple[USSDState, str]]] = {
    USSDState.MAIN_MENU: {
        "1": (USSDState.SYMPTOM_MENU, "CON Enter symptom:\n1.Fever\n2.Cough\n3.Diarrhoea\n4.Rash\n5.Other"),
        "2": (USSDState.RESISTANCE_QUERY, "CON Enter drug name:\n1.Amoxicillin\n2.Ciprofloxacin\n3.Metronidazole\n4.Co-trimoxazole"),
        "3": (USSDState.MY_REPORTS, "CON Loading recent reports..."),
        "4": (USSDState.HELP, "CON UDARA helps track drug resistance.\nReport treatment outcomes.\nDial *384# anytime.\nCall 0800-UDARA for help."),
    },
    USSDState.SYMPTOM_MENU: {
        "1": (USSDState.AGE_MENU, "CON Patient age:\n1.<2yrs\n2.2-5yrs\n3.5-15yrs\n4.15-65yrs\n5.>65yrs"),
        "2": (USSDState.AGE_MENU, "CON Patient age:\n1.<2yrs\n2.2-5yrs\n3.5-15yrs\n4.15-65yrs\n5.>65yrs"),
        "3": (USSDState.AGE_MENU, "CON Patient age:\n1.<2yrs\n2.2-5yrs\n3.5-15yrs\n4.15-65yrs\n5.>65yrs"),
        "4": (USSDState.AGE_MENU, "CON Patient age:\n1.<2yrs\n2.2-5yrs\n3.5-15yrs\n4.15-65yrs\n5.>65yrs"),
    },
    USSDState.AGE_MENU: {
        "1": (USSDState.DURATION_MENU, "CON Illness duration:\n1.<3 days\n2.3-7 days\n3.7-14 days\n4.>14 days"),
        "2": (USSDState.DURATION_MENU, "CON Illness duration:\n1.<3 days\n2.3-7 days\n3.7-14 days\n4.>14 days"),
        "3": (USSDState.DURATION_MENU, "CON Illness duration:\n1.<3 days\n2.3-7 days\n3.7-14 days\n4.>14 days"),
        "4": (USSDState.DURATION_MENU, "CON Illness duration:\n1.<3 days\n2.3-7 days\n3.7-14 days\n4.>14 days"),
        "5": (USSDState.DURATION_MENU, "CON Illness duration:\n1.<3 days\n2.3-7 days\n3.7-14 days\n4.>14 days"),
    },
    USSDState.DURATION_MENU: {
        "1": (USSDState.DRUG_MENU, "CON Drug given:\n1.Amoxicillin\n2.Ciprofloxacin\n3.Metronidazole\n4.Co-trimoxazole\n5.Azithro\n6.Other\n7.None"),
        "2": (USSDState.DRUG_MENU, "CON Drug given:\n1.Amoxicillin\n2.Ciprofloxacin\n3.Metronidazole\n4.Co-trimoxazole\n5.Azithro\n6.Other\n7.None"),
        "3": (USSDState.DRUG_MENU, "CON Drug given:\n1.Amoxicillin\n2.Ciprofloxacin\n3.Metronidazole\n4.Co-trimoxazole\n5.Azithro\n6.Other\n7.None"),
        "4": (USSDState.DRUG_MENU, "CON Drug given:\n1.Amoxicillin\n2.Ciprofloxacin\n3.Metronidazole\n4.Co-trimoxazole\n5.Azithro\n6.Other\n7.None"),
    },
    USSDState.DRUG_MENU: {
        "1": (USSDState.OUTCOME_MENU, "CON After drug:\n1.Improved\n2.No change\n3.Worse\n4.Side effects"),
        "2": (USSDState.OUTCOME_MENU, "CON After drug:\n1.Improved\n2.No change\n3.Worse\n4.Side effects"),
        "3": (USSDState.OUTCOME_MENU, "CON After drug:\n1.Improved\n2.No change\n3.Worse\n4.Side effects"),
        "4": (USSDState.OUTCOME_MENU, "CON After drug:\n1.Improved\n2.No change\n3.Worse\n4.Side effects"),
        "5": (USSDState.OUTCOME_MENU, "CON After drug:\n1.Improved\n2.No change\n3.Worse\n4.Side effects"),
        "6": (USSDState.DRUG_MENU, "CON Type drug name:"),  # Free text input
        "7": (USSDState.OUTCOME_MENU, "CON No drug given. After illness:\n1.Improved\n2.No change\n3.Worse"),
    },
    USSDState.OUTCOME_MENU: {
        "1": (USSDState.CONFIRM_MENU, "CON Review report:\n{summary}\n1.Confirm\n2.Edit\n3.Cancel"),
        "2": (USSDState.CONFIRM_MENU, "CON Review report:\n{summary}\n1.Confirm\n2.Edit\n3.Cancel"),
        "3": (USSDState.CONFIRM_MENU, "CON Review report:\n{summary}\n1.Confirm\n2.Edit\n3.Cancel"),
        "4": (USSDState.CONFIRM_MENU, "CON Review report:\n{summary}\n1.Confirm\n2.Edit\n3.Cancel"),
    },
    USSDState.CONFIRM_MENU: {
        "1": (USSDState.END, "END Thank you! Report UDR-{ref} saved."),
        "2": (USSDState.SYMPTOM_MENU, "CON Enter symptom:\n1.Fever\n2.Cough\n3.Diarrhoea\n4.Rash"),
        "3": (USSDState.END, "END Report cancelled. Dial *384# to start again."),
    },
}

SYMPTOM_MAP = {"1": "fever", "2": "cough", "3": "diarrhoea", "4": "rash", "5": "other"}
AGE_MAP = {"1": "infant", "2": "child", "3": "adolescent", "4": "adult", "5": "elderly"}
DURATION_MAP = {"1": "<3d", "2": "3-7d", "3": "7-14d", "4": ">14d"}
DRUG_MAP = {
    "1": "amoxicillin", "2": "ciprofloxacin", "3": "metronidazole",
    "4": "co_trimoxazole", "5": "azithromycin", "7": "none",
}
OUTCOME_MAP = {
    "1": "improved", "2": "no_change", "3": "worse", "4": "side_effects",
}


@router.post("/callback")
async def ussd_callback(request: Request) -> Response:
    """
    Handle USSD callback from Africa's Talking.
    
    Africa's Talking sends a form-encoded POST with:
    - sessionId: unique session identifier
    - serviceCode: the USSD code (*384#)
    - phoneNumber: user's phone number
    - text: cumulative user input (joined by *)
    """
    form = await request.form()
    callback = USSDCallback(
        sessionId=str(form["sessionId"]),
        serviceCode=str(form["serviceCode"]),
        phoneNumber=str(form["phoneNumber"]),
        text=str(form["text"]),
    )
    
    # Get session manager (injected via FastAPI dependency)
    session_mgr: USSDSessionManager = request.app.state.ussd_session_manager
    
    # Parse user input: Africa's Talking sends cumulative input joined by *
    # e.g., "1*2*3*1*2" means user selected 1, then 2, then 3, then 1, then 2
    inputs = callback.text.split("*") if callback.text else []
    current_input = inputs[-1] if inputs else ""
    
    # Get or create session
    session = session_mgr.get_session(callback.sessionId)
    if session is None:
        session = session_mgr.create_session(callback.sessionId, callback.phoneNumber)
    
    # Route based on current state
    if session.state == USSDState.END:
        session_mgr.delete_session(callback.sessionId)
        return Response(content=session.data.get("end_text", "END Goodbye."), media_type="text/plain")
    
    # Look up menu response for current state + user input
    state_menu = MENU_RESPONSES.get(session.state, {})
    action = state_menu.get(current_input)
    
    if action is None:
        # Invalid input — redisplay current menu
        response_text = f"CON Invalid option. {state_menu.get('1', ('', ''))[1]}"
        return Response(content=response_text, media_type="text/plain")
    
    next_state, response_text = action
    
    # Store user's selection in session data
    if session.state == USSDState.SYMPTOM_MENU:
        session.data["symptom"] = SYMPTOM_MAP.get(current_input, "unknown")
    elif session.state == USSDState.AGE_MENU:
        session.data["age_group"] = AGE_MAP.get(current_input, "unknown")
    elif session.state == USSDState.DURATION_MENU:
        session.data["duration"] = DURATION_MAP.get(current_input, "unknown")
    elif session.state == USSDState.DRUG_MENU:
        session.data["drug"] = DRUG_MAP.get(current_input, "unknown")
    elif session.state == USSDState.OUTCOME_MENU:
        session.data["outcome"] = OUTCOME_MAP.get(current_input, "unknown")
    
    # Build confirm summary
    if next_state == USSDState.CONFIRM_MENU:
        summary = (
            f"Symptom:{session.data.get('symptom', '?')} "
            f"Age:{session.data.get('age_group', '?')} "
            f"Dur:{session.data.get('duration', '?')} "
            f"Drug:{session.data.get('drug', '?')} "
            f"Out:{session.data.get('outcome', '?')}"
        )
        response_text = f"CON {summary}\n1.Confirm\n2.Edit\n3.Cancel"
    
    # Handle confirmation — create report
    if next_state == USSDState.END and current_input == "1":
        # TODO: Create report via report_service
        import uuid
        ref = uuid.uuid4().hex[:8].upper()
        response_text = f"END Thank you! Report UDR-{ref} saved."
        session.data["end_text"] = response_text
    
    # Update session state
    session_mgr.update_session(session, new_state=next_state)
    
    return Response(content=response_text, media_type="text/plain")
```

---

## Door 2: WhatsApp + Telegram + PWA (Smartphones)

### WhatsApp Business API

| Feature | Implementation | Notes |
|---------|---------------|-------|
| Messages | Cloud API (v18.0+) | No on-premise connector needed |
| Media | Upload to WhatsApp servers | Photos, audio, documents |
| Interactive | Interactive messages, list messages, reply buttons | Replaces USSD menus |
| Templates | Pre-approved message templates | Required for proactive messages |
| Webhook | POST to `/api/v1/whatsapp/webhook` | Verify token for setup |
| Rate Limit | WhatsApp: 80 msgs/sec per number | Queued via Celery |

### Telegram Bot

| Feature | Implementation | Notes |
|---------|---------------|-------|
| Framework | aiogram 3.x | Async-native, full Bot API 7.x support |
| Commands | `/report`, `/resistance`, `/help`, `/start` | Quick access to common actions |
| Inline Keyboards | `InlineKeyboardMarkup` | Structured input like USSD but richer |
| Photos | `@message.photo` handler | Receives prescription photos for OCR |
| Voice | `@message.voice` handler | Receives voice notes for ASR |
| Callbacks | `CallbackQueryHandler` | Button press handling |
| Groups | Support via bot permissions | Facility-level group reporting |

### Progressive Web App (PWA)

| Feature | Implementation | Notes |
|---------|---------------|-------|
| Framework | Next.js 14 (static export) | Build as PWA with `next-pwa` |
| Service Worker | Workbox | Offline caching of reports + forms |
| Manifest | `manifest.json` | Installable on Android |
| Camera | `<input type="file" capture="environment">` | Direct photo capture |
| Storage | IndexedDB | Local report cache when offline |
| Sync | Background Sync API | Auto-sync when connectivity returns |

### Features Unique to Door 2

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    DOOR 2 EXCLUSIVE FEATURES                                 │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  ┌──────────────────────────────────────────────────────────────────┐      │
│  │ 1. FREE TEXT REPORTING                                           │      │
│  │                                                                  │      │
│  │ CHW sends natural language message:                              │      │
│  │ "I have a 7-year-old patient with fever and cough for 5 days.   │      │
│  │  I gave amoxicillin but no improvement after 3 days."            │      │
│  │                                                                  │      │
│  │ System extracts all entities via NER (no structured menu needed) │      │
│  │ Response: "Extracted: Fever, Cough | Drug: Amoxicillin |        │      │
│  │  Outcome: Treatment failure. Is this correct? ✅/❌"            │      │
│  └──────────────────────────────────────────────────────────────────┘      │
│                                                                             │
│  ┌──────────────────────────────────────────────────────────────────┐      │
│  │ 2. PHOTO OCR (Prescription/Image Analysis)                       │      │
│  │                                                                  │      │
│  │ CHW sends photo of:                                               │      │
│  │  • Prescription paper → OCR extracts drug name, dosage          │      │
│  │  • Medicine label → OCR extracts drug name, batch number        │      │
│  │  • Handwritten notes → TrOCR for handwritten text                │      │
│  │                                                                  │      │
│  │ Pipeline: Grayscale → CLAHE → Threshold → Deskew →              │      │
│  │           PaddleOCR → Confidence filter (>0.7) → NER merge      │      │
│  └──────────────────────────────────────────────────────────────────┘      │
│                                                                             │
│  ┌──────────────────────────────────────────────────────────────────┐      │
│  │ 3. VOICE ASR (Voice Notes)                                       │      │
│  │                                                                  │      │
│  │ CHW sends voice note (up to 60 seconds):                         │      │
│  │ "Patient with severe diarrhoea, given metronidazole, no         │      │
│  │  improvement, now on second day"                                 │      │
│  │                                                                  │      │
│  │ Pipeline: MMS-ASR (16kHz resample) → Transcript →               │      │
│  │           NER extraction → Structured report                    │      │
│  │                                                                  │      │
│  │ Languages: English, Yoruba, Hausa, Igbo (1100+ total)          │      │
│  └──────────────────────────────────────────────────────────────────┘      │
│                                                                             │
│  ┌──────────────────────────────────────────────────────────────────┐      │
│  │ 4. INLINE KEYBOARDS (Structured Input)                           │      │
│  │                                                                  │      │
│  │ WhatsApp/Telegram show button menus:                              │      │
│  │                                                                  │      │
│  │ ┌─────────────────────────┐                                      │      │
│  │ │ Select Symptom          │                                      │      │
│  │ ├────────────┬────────────┤                                      │      │
│  │ │ 🤒 Fever  │ 💊 Drug    │                                      │      │
│  │ ├────────────┼────────────┤                                      │      │
│  │ │ 😷 Cough  │ ⏱ Duration │                                      │      │
│  │ ├────────────┼────────────┤                                      │      │
│  │ │ 🤢 Diarrhoea│ 👤 Age   │                                      │      │
│  │ └────────────┴────────────┘                                      │      │
│  └──────────────────────────────────────────────────────────────────┘      │
│                                                                             │
│  ┌──────────────────────────────────────────────────────────────────┐      │
│  │ 5. RESISTANCE CALCULATOR                                         │      │
│  │                                                                  │      │
│  │ CHW queries: "/resistance amoxicillin lagos"                      │      │
│  │                                                                  │      │
│  │ Response:                                                        │      │
│  │ "📊 Amoxicillin Resistance in Lagos:                            │      │
│  │  • Overall: 35% [95% CI: 22-48%]                                │      │
│  │  • Last 30 days: 42% ⬆️ (trending up)                          │      │
│  │  • Recommended alternative: Ciprofloxacin (18% resistance)     │      │
│  │  • Data: 156 reports from 12 facilities"                        │      │
│  └──────────────────────────────────────────────────────────────────┘      │
│                                                                             │
│  ┌──────────────────────────────────────────────────────────────────┐      │
│  │ 6. DAILY DIGEST                                                  │      │
│  │                                                                  │      │
│  │ Every day at 07:00 WAT, active CHWs receive:                    │      │
│  │                                                                  │      │
│  │ "📋 UDARA Daily Digest — {date}                                │      │
│  │  📊 Your facility: 5 reports yesterday                         │      │
│  │  📈 Resistance alert: Co-trimoxazole up to 48% in your area    │      │
│  │  🏆 Top reporter: {name} (12 reports this month)              │      │
│  │  💡 Tip: Always complete the full antibiotic course            │      │
│  │  📖 Microlearning: What is ESBL?"                              │      │
│  └──────────────────────────────────────────────────────────────────┘      │
│                                                                             │
│  ┌──────────────────────────────────────────────────────────────────┐      │
│  │ 7. MICROLEARNING PUSH                                            │      │
│  │                                                                  │      │
│  │ Daily AMR education snippets (auto-generated by LLM):          │      │
│  │                                                                  │      │
│  │ "📖 Did you know? ESBL (Extended-Spectrum Beta-Lactamase) is   │      │
│  │  an enzyme that makes bacteria resistant to penicillins and     │      │
│  │  cephalosporins. If a patient doesn't improve after amoxicillin,│      │
│  │  they may have an ESBL-producing infection. Report this! 📲"    │      │
│  └──────────────────────────────────────────────────────────────────┘      │
│                                                                             │
│  ┌──────────────────────────────────────────────────────────────────┐      │
│  │ 8. PEER ALERTS                                                   │      │
│  │                                                                  │      │
│  │ When a CHW reports treatment failure, nearby CHWs are notified: │      │
│  │                                                                  │      │
│  │ "⚠️ Peer Alert: A CHW in your area just reported treatment     │      │
│  │  failure with Amoxicillin for a respiratory infection.          │      │
│  │  Be alert for similar cases. Dial *384# to report."            │      │
│  └──────────────────────────────────────────────────────────────────┘      │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

### Code Examples

```python
# edge/app/api/routes/chatbot.py
"""WhatsApp and Telegram webhook handlers."""

from __future__ import annotations

import logging
from fastapi import APIRouter, Request, Response, HTTPException
from pydantic import BaseModel, Field

logger = logging.getLogger(__name__)
router = APIRouter(prefix="/api/v1/chat", tags=["chatbot"])


# ─── WhatsApp Webhook ───────────────────────────────────────────────

class WhatsAppVerification(BaseModel):
    """WhatsApp Business API verification challenge."""
    hub_mode: str
    hub_verify_token: str
    hub_challenge: str


class WhatsAppMessage(BaseModel):
    """Incoming WhatsApp message."""
    from_: str = Field(..., alias="from")
    id: str
    type: str  # "text", "image", "audio", "interactive"
    timestamp: str
    text: dict | None = None
    image: dict | None = None
    audio: dict | None = None
    interactive: dict | None = None


class WhatsAppPayload(BaseModel):
    """WhatsApp webhook payload."""
    object: str
    entry: list[dict]


@router.get("/whatsapp/webhook")
async def whatsapp_verify(
    hub_mode: str,
    hub_verify_token: str,
    hub_challenge: str,
) -> Response:
    """Verify WhatsApp webhook (called once during setup)."""
    expected_token = "your-verify-token"  # From env var in production
    if hub_mode == "subscribe" and hub_verify_token == expected_token:
        return Response(content=hub_challenge, media_type="text/plain")
    raise HTTPException(status_code=403, detail="Invalid verification token")


@router.post("/whatsapp/webhook")
async def whatsapp_webhook(request: Request) -> Response:
    """Handle incoming WhatsApp messages."""
    payload = await request.json()
    logger.info("WhatsApp webhook received: %s", payload)

    # Extract message from payload
    try:
        entry = payload["entry"][0]
        changes = entry["changes"][0]
        value = changes["value"]
        messages = value.get("messages", [])
        contacts = value.get("contacts", [])

        if not messages:
            return Response(content="OK", media_type="text/plain")

        msg = messages[0]
        phone_number = msg["from"]
        msg_type = msg["type"]
        msg_id = msg["id"]

        # Route based on message type
        if msg_type == "text":
            text = msg["text"]["body"]
            await _handle_text_message(phone_number, text, source="whatsapp")
        elif msg_type == "image":
            image_id = msg["image"]["id"]
            caption = msg["image"].get("caption", "")
            await _handle_image_message(phone_number, image_id, caption, source="whatsapp")
        elif msg_type == "audio":
            audio_id = msg["audio"]["id"]
            await _handle_audio_message(phone_number, audio_id, source="whatsapp")
        elif msg_type == "interactive":
            # Handle button/list responses
            interactive = msg["interactive"]
            await _handle_interactive_response(phone_number, interactive, source="whatsapp")

    except (KeyError, IndexError) as exc:
        logger.error("Failed to parse WhatsApp webhook: %s", exc)
    
    # Always return 200 quickly (WhatsApp retries on 5xx)
    return Response(content="OK", media_type="text/plain")


# ─── Telegram Bot ───────────────────────────────────────────────────

class TelegramUpdate(BaseModel):
    """Incoming Telegram Update."""
    update_id: int
    message: dict | None = None
    callback_query: dict | None = None


@router.post("/telegram/webhook")
async def telegram_webhook(request: Request) -> Response:
    """Handle incoming Telegram updates."""
    payload = await request.json()
    update = TelegramUpdate(**payload)
    logger.info("Telegram update received: update_id=%d", update.update_id)

    if update.message:
        chat_id = update.message["chat"]["id"]
        text = update.message.get("text", "")
        
        # Handle commands
        if text.startswith("/"):
            command = text.split()[0].lower()
            await _handle_telegram_command(chat_id, command, text)
        else:
            await _handle_text_message(str(chat_id), text, source="telegram")
    
    elif update.callback_query:
        callback_id = update.callback_query["id"]
        chat_id = update.callback_query["message"]["chat"]["id"]
        data = update.callback_query["data"]
        await _handle_callback_query(callback_id, chat_id, data, source="telegram")
    
    return Response(content="OK", media_type="text/plain")


# ─── Shared Handlers ────────────────────────────────────────────────

async def _handle_text_message(
    phone_number: str,
    text: str,
    source: str,
) -> None:
    """Process free-text message from any channel."""
    from app.nlp.extractor import extract_entities
    from app.services.report_service import create_report_from_text

    # Step 1: Language identification
    # Step 2: NER extraction
    entities = await extract_entities(text)
    
    # Step 3: Present to user for confirmation
    summary = _format_entities_summary(entities)
    
    if source == "whatsapp":
        await _send_whatsapp_interactive(
            phone_number,
            text=f"📋 I extracted the following:\n\n{summary}\n\nIs this correct?",
            buttons=[{"id": "confirm", "title": "✅ Yes, submit"}, {"id": "edit", "title": "✏️ Edit"}],
        )
    elif source == "telegram":
        await _send_telegram_buttons(
            phone_number,
            text=f"📋 I extracted the following:\n\n{summary}\n\nIs this correct?",
            buttons=[["✅ Yes, submit", "✏️ Edit"]],
        )


async def _handle_image_message(
    phone_number: str,
    image_id: str,
    caption: str,
    source: str,
) -> None:
    """Process image (photo) for OCR."""
    from app.nlp.ocr_pipeline import process_image

    # Download image (channel-specific)
    image_bytes = await _download_media(image_id, source)
    
    # Run OCR pipeline
    ocr_result = await process_image(image_bytes, caption=caption)
    
    # Send OCR result
    text = f"📸 OCR Result (confidence: {ocr_result['avg_confidence']:.0%}):\n\n"
    text += ocr_result["text"]
    
    if ocr_result["entities"]:
        text += "\n\n📋 Extracted:\n"
        for entity in ocr_result["entities"]:
            text += f"  • {entity['type']}: {entity['value']}\n"
    
    await _send_message(phone_number, text, source)


async def _handle_audio_message(
    phone_number: str,
    audio_id: str,
    source: str,
) -> None:
    """Process audio (voice note) for ASR."""
    from app.nlp.transcription import transcribe_audio

    # Download audio
    audio_bytes = await _download_media(audio_id, source)
    
    # Run ASR
    transcript = await transcribe_audio(audio_bytes)
    
    # Send transcript
    await _send_message(
        phone_number,
        f"🎙️ Transcript:\n\n{transcript['text']}\n\n"
        f"(Language: {transcript['language']}, confidence: {transcript['confidence']:.0%})\n\n"
        f"Processing report...",
        source,
    )
    
    # Process as text message
    await _handle_text_message(phone_number, transcript["text"], source)


def _format_entities_summary(entities: list[dict]) -> str:
    """Format extracted entities for user confirmation."""
    type_icons = {
        "SYMPTOM": "🤒", "DRUG": "💊", "DOSAGE": "📏",
        "DURATION": "⏱", "AGE": "👤", "OUTCOME": "📊",
        "DIAGNOSIS": "🏥", "BODY_PART": "🫀",
    }
    lines = []
    for e in entities:
        icon = type_icons.get(e["type"], "📌")
        lines.append(f"{icon} {e['type'].title()}: {e['value']}")
    return "\n".join(lines) if lines else "(No entities detected)"


async def _send_whatsapp_interactive(
    to: str,
    text: str,
    buttons: list[dict],
) -> None:
    """Send interactive buttons via WhatsApp Business API."""
    import httpx
    # Implementation: POST to WhatsApp Cloud API
    # See 16-member-04-integration-bot-engineer.md for full implementation
    pass


async def _send_telegram_buttons(
    chat_id: str,
    text: str,
    buttons: list[list[str]],
) -> None:
    """Send inline keyboard via Telegram Bot API."""
    from aiogram import Bot
    from aiogram.types import InlineKeyboardMarkup, InlineKeyboardButton
    
    bot = Bot(token="from-env")
    keyboard = InlineKeyboardMarkup(
        inline_keyboard=[
            [InlineKeyboardButton(text=btn, callback_data=btn) for btn in row]
            for row in buttons
        ]
    )
    await bot.send_message(chat_id=int(chat_id), text=text, reply_markup=keyboard)


async def _send_message(to: str, text: str, source: str) -> None:
    """Send text message via appropriate channel."""
    pass


async def _download_media(media_id: str, source: str) -> bytes:
    """Download media file from WhatsApp or Telegram."""
    pass
```

---

## Door 3: Web Dashboard (Officials)

### Technology Stack

| Component | Technology | Version | Purpose |
|-----------|-----------|---------|---------|
| Framework | Next.js 14 (App Router) | 14.x | React framework with SSR |
| Language | TypeScript | 5.x | Type safety |
| UI Library | shadcn/ui | Latest | Accessible component library |
| Styling | Tailwind CSS | 4.x | Utility-first CSS |
| Maps | MapLibre GL JS | 4.x | Interactive web maps |
| Charts | Apache ECharts | 5.x | Data visualisation |
| Tables | TanStack Table | 8.x | Headless data tables |
| Data Fetching | TanStack Query | 5.x | Server state management |
| State | Zustand | 4.x | Client state management |
| Forms | React Hook Form + Zod | 7.x / 3.x | Form handling + validation |
| Icons | Lucide React | Latest | Icon library |
| Auth | Keycloak OIDC | 23.x | Authentication + SSO |

### Access Levels

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    ROLE-BASED ACCESS CONTROL (RBAC)                           │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  ┌─────────────────────────────────────────────────────────────────┐        │
│  │  LEVEL 1: FACILITY ADMIN                                        │        │
│  │  Role: facility_admin                                           │        │
│  │  Scope: Single facility                                         │        │
│  │                                                                  │        │
│  │  ✅ View reports from own facility                              │        │
│  │  ✅ View resistance index for own facility                      │        │
│  │  ✅ View local alert feed                                       │        │
│  │  ✅ Export facility data (CSV)                                  │        │
│  │  ✅ Manage CHW profiles (own facility)                          │        │
│  │  ✅ View edge node health (own node)                            │        │
│  │  ❌ View other facilities' data                                │        │
│  │  ❌ Access state/national analytics                            │        │
│  │  ❌ Manage users                                               │        │
│  │  ❌ Export WHO GLASS data                                      │        │
│  └─────────────────────────────────────────────────────────────────┘        │
│                                                                             │
│  ┌─────────────────────────────────────────────────────────────────┐        │
│  │  LEVEL 2: STATE ADMIN                                           │        │
│  │  Role: state_admin                                              │        │
│  │  Scope: All facilities in assigned state                        │        │
│  │                                                                  │        │
│  │  ✅ Everything Level 1 can do (for all facilities in state)    │        │
│  │  ✅ View state-wide resistance trends                          │        │
│  │  ✅ View state-wide map with all facilities                    │        │
│  │  ✅ Run state-level outbreak detection                         │        │
│  │  ✅ Export state data (CSV, PDF, GeoJSON)                       │        │
│  │  ✅ Manage facility admins in state                            │        │
│  │  ✅ View state-level analytics dashboard                        │        │
│  │  ❌ Access other states' data                                  │        │
│  │  ❌ Access national dashboard                                  │        │
│  │  ❌ Export WHO GLASS data                                      │        │
│  └─────────────────────────────────────────────────────────────────┘        │
│                                                                             │
│  ┌─────────────────────────────────────────────────────────────────┐        │
│  │  LEVEL 3: NATIONAL ADMIN                                        │        │
│  │  Role: national_admin                                           │        │
│  │  Scope: All facilities, all states                              │        │
│  │                                                                  │        │
│  │  ✅ Everything Level 2 can do (for all states)                 │        │
│  │  ✅ View national resistance index dashboard                    │        │
│  │  ✅ National interactive map (all facilities)                   │        │
│  │  ✅ Cross-state comparison analytics                            │        │
│  │  ✅ National outbreak alerts + dispatch                        │        │
│  │  ✅ Export WHO GLASS-compatible data                           │        │
│  │  ✅ Manage all users and roles                                 │        │
│  │  ✅ Configure alert thresholds                                 │        │
│  │  ✅ Manage edge node fleet                                     │        │
│  │  ✅ Access ML model performance metrics                        │        │
│  │  ✅ Configure data retention policies                           │        │
│  └─────────────────────────────────────────────────────────────────┘        │
│                                                                             │
│  PERMISSION MATRIX:                                                         │
│  ┌─────────────────────────┬────────┬────────┬──────────┐                   │
│  │ Resource                │ FACILITY│ STATE │ NATIONAL │                   │
│  ├─────────────────────────┼────────┼────────┼──────────┤                   │
│  │ View own reports        │   ✅    │   ✅    │    ✅     │                   │
│  │ View other facility     │   ❌    │   ✅    │    ✅     │                   │
│  │ View state reports      │   ❌    │   ✅    │    ✅     │                   │
│  │ View national reports   │   ❌    │   ❌    │    ✅     │                   │
│  │ Export CSV              │   ✅    │   ✅    │    ✅     │                   │
│  │ Export GeoJSON          │   ❌    │   ✅    │    ✅     │                   │
│  │ Export WHO GLASS        │   ❌    │   ❌    │    ✅     │                   │
│  │ Manage users            │   ❌    │   ✅*   │    ✅     │                   │
│  │ Configure alerts        │   ❌    │   ✅*   │    ✅     │                   │
│  │ Edge node management    │   ❌    │   ❌    │    ✅     │                   │
│  │ Model management        │   ❌    │   ❌    │    ✅     │                   │
│  └─────────────────────────┴────────┴────────┴──────────┘                   │
│  * State admin can only manage facility admins within their state           │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

### Dashboard Components

| Component | Library | Description |
|-----------|---------|-------------|
| **MapView** | MapLibre GL JS | Interactive map with facility markers, choropleth layers, outbreak polygons |
| **ResistanceIndexChart** | ECharts | Line chart: resistance index over time per drug/district |
| **TrendLineChart** | ECharts | Multi-series time series for report volume, treatment outcomes |
| **DrugResistanceBar** | ECharts | Stacked bar: drug resistance rates by region |
| **OutbreakHeatmap** | ECharts | Calendar heatmap: daily report density |
| **ReportsTable** | TanStack Table | Paginated, sortable, filterable report browser |
| **AlertsTable** | TanStack Table | Alert feed with severity, status, acknowledge/dismiss |
| **FacilitiesTable** | TanStack Table | Facility list with health indicators |
| **FiltersPanel** | shadcn/ui | Date range, drug, region, outcome filters |
| **KPI Cards** | shadcn/ui Card | Key metrics: total reports, resistance index, active alerts |

### ASCII Wireframes

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                         DOOR 3: WEB DASHBOARD                              │
│                        ASCII WIREFRAME (DESKTOP)                            │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  ┌──────┐ ┌────────────────────────────────────────────────────────────┐   │
│  │      │ │  🔍 Search reports...              🔔 3 alerts  👤 Admin  │   │
│  │  SID │ ├────────────────────────────────────────────────────────────┤   │
│  │      │ │                                                            │   │
│  │ 📊   │ │  ┌────────────┐ ┌────────────┐ ┌────────────┐ ┌────────┐ │   │
│  │Overview│ │  │  Reports   │ │ Resistance │ │   Active   │  Edge   │ │   │
│  │      │ │  │  This Week │ │   Index    │ │   Alerts   │ │ Nodes  │ │   │
│  │ 📋   │ │  │    847     │ │   32.4%   │ │     12     │ │  89%   │ │   │
│  │Reports│ │  │  ▲ +12%   │ │  ▲ +2.1%  │ │  ▲ +3     │ │ online │ │   │
│  │      │ │  └────────────┘ └────────────┘ └────────────┘ └────────┘ │   │
│  │ 🗺️   │ │                                                            │   │
│  │ Map  │ │  ┌──────────────────────────┐ ┌──────────────────────────┐ │   │
│  │      │ │  │                          │ │  Resistance Index Trend   │ │   │
│  │ 📈   │ │  │     🗺️ MAP VIEW          │ │                          │ │   │
│  │Trends│ │  │                          │ │  45%│     *               │ │   │
│  │      │ │  │   🔴 Lagos      42%     │ │  40%│   *   *             │ │   │
│  │ ⚠️   │ │  │   🟡 Abuja     28%     │ │  35%│ *                   │ │   │
│  │Alerts│ │  │   🟢 Ibadan     15%     │ │  30%│                     │ │   │
│  │      │ │  │   🟢 Kano       12%     │ │  25%│                     │ │   │
│  │ 🏥   │ │  │                          │ │  20%│                     │ │   │
│  │Facil.│ │  │   ──────────────────      │ │     └──────────────────  │ │   │
│  │      │ │  │   Legend: Resistance %    │ │  Jan Feb Mar Apr May    │ │   │
│  │ 🤖   │ │  │   🟢 <20% 🟡 20-40% 🔴 >40% │                          │ │   │
│  │Edge  │ │  └──────────────────────────┘ └──────────────────────────┘ │   │
│  │      │ │                                                            │   │
│  │ ⚙️   │ │  ┌──────────────────────────────────────────────────────┐ │   │
│  │Sett. │ │  │  Recent Reports                          📥 Export   │ │   │
│  │      │ │  ├──────────────────────────────────────────────────────┤ │   │
│  └──────┘ │  │ ID │ Facility │ Drug │ Symptom │ Outcome │ Date     │ │   │
│           │  │────┼──────────┼──────┼─────────┼─────────┼──────────│ │   │
│           │  │UDR│ PHC_Lagos│ Amox │ Fever   │ Tx Fail │ Jan 15   │ │   │
│           │  │UDR│ PHC_Abuja│ Cipro│ Cough   │ Improved│ Jan 15   │ │   │
│           │  │UDR│ PHC_Kano │ CTMX │ Diarrh. │ No Chg  │ Jan 14   │ │   │
│           │  │UDR│ PHC_Ibadan│ Metro│ Rash    │ Worse   │ Jan 14   │ │   │
│           │  │                                                      │ │   │
│           │  │  ← 1  2  3  4  5  ...  42 →    Showing 1-10 of 847  │ │   │
│           │  └──────────────────────────────────────────────────────┘ │   │
│           └────────────────────────────────────────────────────────────┘   │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## Channel Comparison Table

| Feature | USSD | WhatsApp | Telegram | PWA | Web Dashboard |
|---------|------|----------|----------|-----|---------------|
| **Device** | Feature phone | Smartphone | Smartphone | Smartphone | Desktop/Laptop |
| **Internet Required** | No | Yes | Yes | No (offline) | Yes |
| **Data Cost** | ~0 (USSD free) | ~1-5 KB/msg | ~1-3 KB/msg | ~50-200 KB | ~500 KB/page |
| **Input Method** | Numeric menu | Free text + buttons | Free text + buttons | Touch + type | Keyboard + mouse |
| **Photo Support** | ❌ | ✅ | ✅ | ✅ | N/A |
| **Voice Support** | ❌ | ✅ | ✅ | ✅ | N/A |
| **Language** | English (menu) | Any (free text) | Any (free text) | Any | UI: EN, YO, HA, IG |
| **Structured Input** | ✅ (menu tree) | ✅ (buttons) | ✅ (buttons) | ✅ (forms) | ✅ (full forms) |
| **Free Text** | ⚠️ (limited) | ✅ | ✅ | ✅ | ✅ |
| **OCR** | ❌ | ✅ | ✅ | ✅ | N/A |
| **ASR** | ❌ | ✅ | ✅ | ✅ | N/A |
| **Maps** | ❌ | ❌ | ❌ | ⚠️ (basic) | ✅ (MapLibre GL) |
| **Charts** | ❌ | ❌ | ❌ | ⚠️ (basic) | ✅ (ECharts) |
| **Resistance Calc** | ✅ (basic) | ✅ | ✅ | ✅ | ✅ (full) |
| **Alerts** | ✅ (SMS) | ✅ | ✅ | ✅ | ✅ (full) |
| **Daily Digest** | ✅ (SMS) | ✅ | ✅ | ✅ | N/A |
| **Microlearning** | ✅ (SMS) | ✅ | ✅ | ✅ | N/A |
| **Peer Alerts** | ✅ (SMS) | ✅ | ✅ | ✅ | N/A |
| **Data Export** | ❌ | ❌ | ❌ | ⚠️ (CSV) | ✅ (CSV, PDF, GeoJSON) |
| **Offline Support** | ✅ (inherent) | ❌ | ❌ | ✅ (service worker) | ❌ |
| **Session Timeout** | 120s | 24h | Unlimited | Unlimited | 30 min idle |
| **Max Message Length** | 160 chars | 4096 chars | 4096 chars | N/A | N/A |
| **Target Users** | CHWs (35%) | CHWs (30%) | CHWs (10%) | CHWs (15%) | Officials (5%) |

---

## ASCII Wireframes per Channel

### Door 1: USSD Wireframe

```
┌─────────────────────┐
│   NOKIA 105 LCD     │
│  (84×48 pixels)     │
│  ┌───────────────┐  │
│  │UDARA AMR Surv. │  │
│  │1.Report outcm  │  │
│  │2.Resist info   │  │
│  │3.My reports    │  │
│  │4.Help          │  │
│  │               │  │
│  └───────────────┘  │
│  ┌─┐ ┌─┐ ┌─┐     │
│  │1│ │2│ │3│      │  (Numeric keypad)
│  ├─┤ ├─┤ ├─┤     │
│  │4│ │5│ │6│      │
│  ├─┤ ├─┤ ├─┤     │
│  │7│ │8│ │9│      │
│  ├─┤ ├─┤ ├─┤     │
│  │*│ │0│ │#│      │
│  └─┘ └─┘ └─┘     │
└─────────────────────┘

 Flow: User dials *384#
       ↓
 ┌───────────────────┐
 │UDARA AMR Surv.    │
 │1.Report outcm     │
 │2.Resist info      │
 │3.My reports       │
 │4.Help             │
 └───────────────────┘
       │ User presses 1
       ↓
 ┌───────────────────┐
 │Enter symptom:     │
 │1.Fever            │
 │2.Cough            │
 │3.Diarrhoea        │
 │4.Rash             │
 │5.Other            │
 └───────────────────┘
       │ User presses 1
       ↓
 ┌───────────────────┐
 │Patient age:       │
 │1.<2yrs 2.2-5yrs   │
 │3.5-15  4.15-65    │
 │5.>65              │
 └───────────────────┘
       │ ... continues through all menus
       ↓
 ┌───────────────────┐
 │Confirm:           │
 │Fever|Adult|5d     │
 │Amox|No change     │
 │1.OK 2.Edit 3.No   │
 └───────────────────┘
       │ User presses 1
       ↓
 ┌───────────────────┐
 │Thank you!         │
 │Ref:UDR-A1B2C3D4   │
 └───────────────────┘
```

### Door 2: WhatsApp Wireframe

```
┌──────────────────────────────────┐
│  WhatsApp Chat Screen            │
├──────────────────────────────────┤
│                                  │
│  ┌────────────────────────────┐  │
│  │ 🟢 UDARA AI Bot     10:30 │  │  ← Bot message
│  │                            │  │
│  │ Welcome to UDARA AI! 🏥    │  │
│  │ I can help you report AMR  │  │
│  │ observations.              │  │
│  │                            │  │
│  │ Choose an action:          │  │
│  │ ┌──────────┐ ┌──────────┐ │  │
│  │ │ 📋 Report│ │ 📊 Resist│ │  │  ← Quick reply buttons
│  │ └──────────┘ └──────────┘ │  │
│  │ ┌──────────┐ ┌──────────┐ │  │
│  │ │ 📖 Learn │ │ ❓ Help   │ │  │
│  │ └──────────┘ └──────────┘ │  │
│  └────────────────────────────┘  │
│                                  │
│  ┌────────────────────────────┐  │
│  │                    CHW 👤  │  │  ← User taps 📋 Report
│  │ Patient has fever and     │  │
│  │ cough for 5 days. Given   │  │
│  │ amoxicillin, no change.   │  │
│  └────────────────────────────┘  │
│                                  │
│  ┌────────────────────────────┐  │
│  │ 🟢 UDARA AI Bot     10:31 │  │
│  │                            │  │
│  │ 📋 I extracted:            │  │
│  │                            │  │
│  │ 🤒 Symptom: Fever, Cough  │  │
│  │ ⏱ Duration: 5 days        │  │
│  │ 💊 Drug: Amoxicillin       │  │
│  │ 📊 Outcome: No change      │  │
│  │                            │  │
│  │ Is this correct?           │  │
│  │ ┌──────────┐ ┌──────────┐ │  │
│  │ │ ✅ Yes   │ │ ✏️ Edit  │ │  │
│  │ └──────────┘ └──────────┘ │  │
│  └────────────────────────────┘  │
│                                  │
│  ┌────────────────────────────┐  │
│  │                    CHW 👤  │  │  ← User taps ✅ Yes
│  │ ✅                      │  │
│  └────────────────────────────┘  │
│                                  │
│  ┌────────────────────────────┐  │
│  │ 🟢 UDARA AI Bot     10:31 │  │
│  │                            │  │
│  │ ✅ Report saved! Ref:      │  │
│  │ UDR-A1B2C3D4               │  │
│  │                            │  │
│  │ ⚠️ Alert: Amoxicillin      │  │
│  │ resistance in your area is │  │
│  │ 42%. Consider alternative. │  │
│  │                            │  │
│  │ ┌────────────────────────┐│  │
│  │ │📸 Send photo of Rx     ││  │
│  │ └────────────────────────┘│  │
│  └────────────────────────────┘  │
│                                  │
│  ┌────────────────────────────┐  │
│  │                    CHW 👤  │  │  ← User sends photo
│  │         📷 [Photo]        │  │
│  │  Prescription for patient │  │
│  └────────────────────────────┘  │
│                                  │
│  ┌────────────────────────────┐  │
│  │ 🟢 UDARA AI Bot     10:32 │  │
│  │                            │  │
│  │ 📸 OCR Result (92%):       │  │
│  │                            │  │
│  │ Drug: METRONIDAZOLE 400mg  │  │
│  │ Take 3x daily for 7 days   │  │
│  │                            │  │
│  │ Added to report UDR-A1B2.. │  │
│  └────────────────────────────┘  │
│                                  │
│  ┌──────────────────────────────┐
│  │ Type a message...      📎 🎤│  │  ← Input bar
│  └──────────────────────────────┘
└──────────────────────────────────┘
```

### Door 2: Telegram Wireframe

```
┌──────────────────────────────────┐
│  Telegram Chat Screen            │
├──────────────────────────────────┤
│                                  │
│  /report  — Submit AMR report    │  ← Bot command menu
│  /resistance — Check resistance │
│  /learn   — AMR microlearning   │
│  /help    — Get help            │
│                                  │
│  ─────────────────────────────── │
│                                  │
│  ┌────────────────────────────┐  │
│  │ UDARA AI Bot         10:30│  │
│  │                            │  │
│  │ 👋 Welcome to UDARA AI!    │  │
│  │                            │  │
│  │ /report — Submit report    │  │
│  │ /resistance — Drug info    │  │
│  │ /learn — AMR education     │  │
│  │ /help — Get help           │  │
│  └────────────────────────────┘  │
│                                  │
│  ┌────────────────────────────┐  │
│  │                    You 10:31│  │
│  │ /report                    │  │
│  └────────────────────────────┘  │
│                                  │
│  ┌────────────────────────────┐  │
│  │ UDARA AI Bot         10:31│  │
│  │                            │  │
│  │ 📋 Report AMR Observation  │  │
│  │                            │  │
│  │ Choose input method:       │  │
│  │                            │  │
│  │ ┌──────────────────────┐   │  │
│  │ │ ✍️ Type symptoms     │   │  │  ← Inline keyboard
│  │ ├──────────────────────┤   │  │
│  │ │ 📸 Send photo        │   │  │
│  │ ├──────────────────────┤   │  │
│  │ │ 🎤 Send voice note   │   │  │
│  │ ├──────────────────────┤   │  │
│  │ │ 📋 Structured menu   │   │  │
│  │ └──────────────────────┘   │  │
│  └────────────────────────────┘  │
│                                  │
│  ┌────────────────────────────┐  │
│  │ UDARA AI Bot         10:32│  │  ← After voice note
│  │                            │  │
│  │ 🎙️ Transcript (94%):      │  │
│  │ "Patient with severe       │  │
│  │  diarrhoea, given metro-   │  │
│  │  nidazole, no improvement,  │  │
│  │  now on second day"        │  │
│  │                            │  │
│  │ 📋 Extracted:              │  │
│  │ 🤒 Symptom: Diarrhoea      │  │
│  │ 💊 Drug: Metronidazole     │  │
│  │ ⏱ Duration: 2 days        │  │
│  │ 📊 Outcome: No change      │  │
│  │                            │  │
│  │ ┌──────────┐ ┌──────────┐  │  │
│  │ │ ✅ Send  │ │ ✏️ Edit  │  │  │
│  │ └──────────┘ └──────────┘  │  │
│  └────────────────────────────┘  │
│                                  │
│  ┌──────────────────────────────┐
│  │ Message...               🎤 📎│
│  └──────────────────────────────┘
└──────────────────────────────────┘
```

### Door 2: PWA Wireframe

```
┌──────────────────────────────────────┐
│  UDARA AI — PWA (Mobile Chrome)      │
│  ┌────────────────────────────────┐  │
│  │ ≡  UDARA AI        🔔  👤     │  │  ← Header
│  ├────────────────────────────────┤  │
│  │                                │  │
│  │  ┌────────┐ ┌────────┐ ┌────┐ │  │
│  │  │ Reports│ │ Resis- │ │Alert│ │  │  ← Bottom nav
│  │  │   12   │ │ tance  │ │  3  │ │  │
│  │  │  today │ │  32%   │ │ new │ │  │
│  │  └────────┘ └────────┘ └────┘ │  │
│  │                                │  │
│  │  ── Quick Actions ──           │  │
│  │  ┌──────────────────────────┐  │  │
│  │  │ 📋 New Report            │  │  │  ← Action cards
│  │  └──────────────────────────┘  │  │
│  │  ┌──────────┐ ┌──────────┐   │  │
│  │  │ 📸 Photo │ │ 🎤 Voice │   │  │
│  │  │  Report  │ │  Report  │   │  │
│  │  └──────────┘ └──────────┘   │  │
│  │                                │  │
│  │  ── Resistance in Your Area ── │  │
│  │                                │  │
│  │  Amoxicillin    ████████░ 42% │  │  ← Mini chart
│  │  Ciprofloxacin  ███░░░░░░ 18% │  │
│  │  Co-trimoxazole ██████░░░ 35% │  │
│  │  Metronidazole  ██░░░░░░░ 12% │  │
│  │                                │  │
│  │  ── Recent Reports ──         │  │
│  │  ┌──────────────────────────┐  │  │
│  │  │ 🤒 Fever + Cough        │  │  │
│  │  │ 💊 Amoxicillin | ❌ Fail │  │  │
│  │  │ 📅 Jan 15, 10:30       │  │  │
│  │  ├──────────────────────────┤  │  │
│  │  │ 🤢 Diarrhoea            │  │  │
│  │  │ 💊 Metronidazole | ⚠️ NC │  │  │
│  │  │ 📅 Jan 15, 09:15       │  │  │
│  │  ├──────────────────────────┤  │  │
│  │  │ 😷 Cough                │  │  │
│  │  │ 💊 Azithromycin | ✅ Imp │  │  │
│  │  │ 📅 Jan 14, 16:45       │  │  │
│  │  └──────────────────────────┘  │  │
│  │                                │  │
│  ├────────────────────────────────┤  │
│  │ 📋      🗺️      📊      ⚙️   │  │  ← Tab bar
│  │Reports   Map   Analytics Settings│
│  └────────────────────────────────┘  │
│                                      │
│  ── New Report Form (expand) ──      │
│  ┌────────────────────────────────┐  │
│  │  ← New Report                 │  │
│  │                                │  │
│  │  Symptoms *                    │  │
│  │  ┌──────────────────────────┐  │  │
│  │  │ Type or tap to select...│  │  │
│  │  └──────────────────────────┘  │  │
│  │  🤒 Fever  😷 Cough  🤢 Diar  │  │  ← Suggestion chips
│  │                                │  │
│  │  Drug Prescribed *             │  │
│  │  ┌──────────────────────────┐  │  │
│  │  │ Select drug...          │  │  │
│  │  └──────────────────────────┘  │  │
│  │  💊 Amox  💊 Cipro  💊 Metro  │  │
│  │                                │  │
│  │  Duration                      │  │
│  │  ┌──────────────────────────┐  │  │
│  │  │ [3] days                │  │  │
│  │  └──────────────────────────┘  │  │
│  │                                │  │
│  │  Outcome *                     │  │
│  │  (•) ✅ Improved               │  │
│  │  ( ) ⚠️ No change              │  │
│  │  ( ) ❌ Worse                  │  │
│  │  ( ) 💊 Side effects           │  │
│  │                                │  │
│  │  📸 Attach Photo               │  │
│  │  ┌──────────────────────────┐  │  │
│  │  │    📷 Tap to capture     │  │  │
│  │  └──────────────────────────┘  │  │
│  │                                │  │
│  │  ┌──────────────────────────┐  │  │
│  │  │     ✅ Submit Report      │  │  │
│  │  └──────────────────────────┘  │  │
│  │                                │  │
│  │  ☐ Offline mode — will sync    │  │
│  │    when connected              │  │
│  └────────────────────────────────┘  │
└──────────────────────────────────────┘
```

### Door 3: Web Dashboard — Mobile Wireframe

```
┌──────────────────────────────────┐
│  UDARA AI — Dashboard (Mobile)  │
│  ┌────────────────────────────┐  │
│  │ ☰  UDARA AI     🔔  👤    │  │
│  ├────────────────────────────┤  │
│  │                            │  │
│  │  ┌────────┐ ┌────────┐    │  │
│  │  │  847   │ │ 32.4%  │    │  │
│  │  │Reports │ │ Resis- │    │  │  ← KPI cards
│  │  │ +12% ↑ │ │ +2.1% ↑│    │  │
│  │  └────────┘ └────────┘    │  │
│  │  ┌────────┐ ┌────────┐    │  │
│  │  │   12   │ │  89%   │    │  │
│  │  │ Alerts │ │ Nodes  │    │  │
│  │  │ +3 ↑   │ │ Online │    │  │
│  │  └────────┘ └────────┘    │  │
│  │                            │  │
│  │  ── Resistance Trend ──    │  │
│  │  ┌────────────────────┐   │  │
│  │  │   *                │   │  │  ← Mini chart
│  │  │  * *   *           │   │  │
│  │  │ *     *            │   │  │
│  │  │____________________│   │  │
│  │  │ Jan Feb Mar Apr May│   │  │
│  │  └────────────────────┘   │  │
│  │                            │  │
│  │  ── Facility Map ──        │  │
│  │  ┌────────────────────┐   │  │
│  │  │                    │   │  │
│  │  │  🟢  🟡  🔴        │   │  │
│  │  │    🟢     🟢       │   │  │  ← Map preview
│  │  │  🟡           🟡   │   │  │
│  │  │                    │   │  │
│  │  │  [View Full Map →] │   │  │
│  │  └────────────────────┘   │  │
│  │                            │  │
│  │  ── Recent Alerts ──       │  │
│  │  ┌────────────────────┐   │  │
│  │  │ 🔴 HIGH Amoxicillin│   │  │
│  │  │    resistance >40% │   │  │
│  │  │    Lagos | 2h ago  │   │  │
│  │  ├────────────────────┤   │  │
│  │  │ 🟡 MED Co-trimox   │   │  │
│  │  │    upward trend    │   │  │
│  │  │    Abuja | 5h ago  │   │  │
│  │  └────────────────────┘   │  │
│  │                            │  │
│  ├────────────────────────────┤  │
│  │ 📊  🗺️  ⚠️  🏥  ⚙️        │  │  ← Tab bar
│  └────────────────────────────┘  │
└──────────────────────────────────┘
```

---

> **Next:** [03 — Tech Stack](03-tech-stack.md)
> **Prev:** [01 — Architecture Overview](01-architecture-overview.md)
