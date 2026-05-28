# 09 — USSD Strategy

> **UDARA AI** — AMR Surveillance Platform for Sub-Saharan Africa  
> Document Version: 2.1.0 | Last Updated: 2026-05-27 | Status: Production

---

## Table of Contents

1. [Why USSD?](#1-why-ussd)
2. [USSD in Africa: Market Context](#2-ussd-in-africa-market-context)
3. [Free USSD Strategies](#3-free-ussd-strategies)
4. [USSD Menu State Machine](#4-ussd-menu-state-machine)
5. [Response Formatting & Constraints](#5-response-formatting--constraints)
6. [Session Manager Implementation](#6-session-manager-implementation)
7. [USSD Webhook Handler](#7-ussd-webhook-handler)
8. [Integration with UDARA Agents](#8-integration-with-udara-agents)
9. [Cost Analysis](#9-cost-analysis)
10. [Testing & Simulation](#10-testing--simulation)
11. [Monitoring & Analytics](#11-monitoring--analytics)

---

## 1. Why USSD?

### 1.1 The Feature Phone Gap

UDARA's CHWs (Community Health Workers) operate in environments where smartphones
are not universal. USSD (Unstructured Supplementary Service Data) is the only
channel that reaches every mobile phone user, regardless of device type.

```
┌─────────────────────────────────────────────────────────────────────┐
│                CHW DEVICE DISTRIBUTION (ESTIMATED)                  │
│                                                                      │
│  ┌────────────────────────┐  ┌────────────────────────┐             │
│  │   FEATURE PHONES       │  │   SMARTPHONES           │             │
│  │   30-40% of CHWs       │  │   60-70% of CHWs        │             │
│  │                        │  │                         │             │
│  │  • Nokia 105/110       │  │  • Android 8-14         │             │
│  │  • Tecno T301          │  │  • Samsung Galaxy A     │             │
│  │  • Itel keypad phones  │  │  • Budget Chinese brands │             │
│  │  • No data plan        │  │  • Variable data plans  │             │
│  │  • No app install      │  │  • WhatsApp available   │             │
│  │  • No internet         │  │  • Some have 3G/4G      │             │
│  │  • USSD ONLY channel   │  │  • USSD as backup       │             │
│  │                        │  │                         │             │
│  │  CHANNELS: USSD        │  │  CHANNELS: App, USSD,   │             │
│  │            (mandatory) │  │            WhatsApp,     │             │
│  │                        │  │            Telegram      │             │
│  └────────────────────────┘  └────────────────────────┘             │
│                                                                      │
│  ═════════════════════════════════════════════════════════════════  │
│  USSD is the ONLY channel that reaches 100% of CHWs.               │
│  Smartphones may lack data. Feature phones cannot install apps.    │
│  USSD works on every phone, on every network, with zero data.      │
└─────────────────────────────────────────────────────────────────────┘
```

### 1.2 USSD Advantages for UDARA

| Advantage | Why It Matters for AMR Surveillance |
|-----------|-------------------------------------|
| **Universal reach** | Every phone with a SIM card supports USSD. No downloads, no installs. |
| **No data plan required** | USSD runs on the signaling channel, not the data channel. Works even with zero data balance. |
| **No app install** | Eliminates onboarding friction. CHW can start reporting immediately. |
| **No storage required** | Phone memory is not a constraint (feature phones have ~4KB app storage). |
| **Familiar interface** | CHWs already use USSD for mobile money (M-Pesa), airtime, banking. Low learning curve. |
| **Works on 2G networks** | USSD uses the GSM signaling channel, functional even on EDGE (Tier 3 connectivity). |
| **Persistent session** | USSD sessions are telco-managed, no client-side state management. |
| **Low cognitive load** | Menu-driven interface with numbered choices. No typing required for most flows. |

### 1.3 USSD Limitations

| Limitation | Mitigation Strategy |
|------------|-------------------|
| **160 char response limit** | Aggressive abbreviation, multi-screen flows, CON/END management |
| **No media (photos/audio)** | USSD is text-only. For images/audio, CHW must use smartphone app or feature phone camera workaround |
| **Session timeout** | Telcos timeout sessions after 30-180 seconds. Design flows to complete within 60 seconds |
| **No push notifications** | CHW must initiate. We cannot proactively alert via USSD (use SMS fallback) |
| **No offline mode** | USSD requires network (even if no data). For fully offline, use paper forms |
| **Screen navigation only** | No back button (on most phones). Use numbered options to navigate |
| **Telco-dependent** | Each telco has slightly different USSD behavior. Must test per-network |
| **Latency** | 1-3 seconds per request/response round-trip. Keep flow steps minimal |

---

## 2. USSD in Africa: Market Context

### 2.1 USSD Adoption by Country

| Country | Mobile Penetration | Feature Phone % | Primary USSD Provider | USSD Session Cost |
|---------|-------------------|-----------------|----------------------|-------------------|
| Kenya | 95% | 25-30% | Safaricom (*384#) | KES 0.50/session |
| Nigeria | 88% | 35-40% | MTN/Airtel/*384# | NGN 20/session |
| Ghana | 85% | 30-35% | MTN/Vodafone/*384# | GHS 0.30/session |
| Tanzania | 75% | 40-45% | Vodacom/Airtel/*384# | TZS 100/session |
| Uganda | 70% | 45-50% | MTN/Airtel/*384# | UGX 100/session |
| Ethiopia | 55% | 60-70% | Ethio Telecom/*384# | ETB 1.50/session |
| Rwanda | 75% | 35-40% | MTN/Airtel/*384# | RWF 20/session |
| Mozambique | 50% | 55-65% | Vodacom/Movitel/*384# | MZN 2.00/session |

### 2.2 Telco USSD Technical Specs

```
┌─────────────────────────────────────────────────────────────────────┐
│               TELCO USSD TECHNICAL CONSTRAINTS                     │
├────────────────────────┬───────────────────────────────────────────┤
│ Parameter             │ Typical Values                             │
├────────────────────────┼───────────────────────────────────────────┤
│ Max response length   │ 160 characters (GSM 7-bit encoding)        │
│ Max input length      │ 182 characters (varies by telco)          │
│ Session timeout       │ 30-180 seconds (Safaricom: 120s default)  │
│ Concurrent sessions   │ 1 per MSISDN (new session kills old)     │
│ Encoding              │ GSM 7-bit (128 chars) or UCS-2 (70 chars)│
│ Special characters    │ Limited: avoid é, ñ,中文 (use ASCII only)  │
│ Newline               │ \n works (renders as line break on phone) │
│ Session initiation     │ User dials *code# (e.g., *384*12345#)    │
│ Session continuation  │ User sends next input (1-9 digit choice) │
│ Session termination    │ Server sends "END" prefix                │
│ Connection            │ Telco → webhook (HTTP POST, form-encoded) │
└────────────────────────┴───────────────────────────────────────────┘
```

---

## 3. Free USSD Strategies

### 3.1 Strategy Overview

USSD sessions typically cost the service provider (UDARA) money per session. This
is a significant cost concern for a surveillance platform operating across thousands
of CHWs making multiple reports daily. UDARA implements 4 complementary strategies
to minimize or eliminate USSD costs.

```
┌─────────────────────────────────────────────────────────────────────┐
│                    UDARA FREE USSD STRATEGIES                        │
│                                                                      │
│  STRATEGY 1:            STRATEGY 2:           STRATEGY 3:           │
│  ┌──────────────┐      ┌──────────────┐     ┌──────────────┐       │
│  │ Africa's      │      │ AT Sandbox   │     │ Telco CSR    │       │
│  │ Talking       │      │ (Staging)    │     │ Partnership  │       │
│  │ Simulator     │      │              │     │ (Production) │       │
│  │ (Development) │      │              │     │              │       │
│  │ Free, local   │      │ Free quota,  │     │ Zero-rated,  │       │
│  │ testing only  │      │ real telco   │     │ telco bears  │       │
│  │ No real phone │      │ integration  │     │ cost         │       │
│  └──────────────┘      └──────────────┘     └──────────────┘       │
│       │                      │                      │               │
│       ▼                      ▼                      ▼               │
│  STRATEGY 4:                                                         │
│  ┌──────────────────────────────────────────────────────┐            │
│  │ Government/Donor Sponsorship                         │            │
│  │ (Long-term sustainability)                          │            │
│  │ Ministry of Health budget or donor funding covers    │            │
│  │ USSD costs as a public health surveillance cost      │            │
│  └──────────────────────────────────────────────────────┘            │
│                                                                      │
│  Cost per session by strategy:                                       │
│    Simulator: $0.00    Sandbox: $0.00    CSR: $0.00    Donor: funded │
└─────────────────────────────────────────────────────────────────────┘
```

### 3.2 Strategy 1: Africa's Talking Simulator (Development)

```bash
# Install Africa's Talking Python SDK
pip install africastalking

# Run the USSD simulator locally (no telco, no phone needed)
# Africa's Talking provides a web-based simulator at:
# https://simulator.africastalking.com:16000

# Configure for simulator
# .env (development)
AT_USERNAME=sandbox                    # Use "sandbox" for testing
AT_API_KEY=test_key_from_at_dashboard   # Free sandbox API key
UDARA_USSD_SHORTCODE=12345             # Simulated shortcode
UDARA_USSD_CALLBACK_URL=https://your-ngrok-url/webhooks/ussd
```

**How it works:**
1. Go to https://simulator.africastalking.com:16000
2. Enter your phone number (any format)
3. Dial your USSD code (*384*12345#)
4. The simulator sends HTTP POST to your webhook
5. Your response text appears in the simulator
6. No real phone needed, no cost, unlimited testing

### 3.3 Strategy 2: AT Sandbox (Staging)

The Africa's Talking Sandbox provides real telco integration without charges:

```bash
# .env (staging)
AT_USERNAME=sandbox
AT_API_KEY=<sandbox-api-key>
UDARA_USSD_SHORTCODE=12345
# Expose local webhook via ngrok for telco callbacks
# ngrok http 8000 → https://abc123.ngrok-free.app
UDARA_USSD_CALLBACK_URL=https://abc123.ngrok-free.app/v2/webhooks/ussd
```

**Sandbox limits:**
| Resource | Limit | Notes |
|----------|-------|-------|
| USSD sessions/day | Unlimited (testing) | Not production traffic |
| Test numbers | 5 registered numbers | Register at AT dashboard |
| SMS (if used) | 100/day free | For USSD backup alerts |
| Real phone required | Yes (registered test numbers) | Can test actual phone UX |
| Cost | $0.00 | Free for development/staging |

### 3.4 Strategy 3: Telco CSR Partnerships (Production)

**Negotiate toll-free (zero-rated) USSD with mobile network operators.** This is
the primary strategy for production deployment. Telcos in East Africa have
established CSR (Corporate Social Responsibility) programs that zero-rate
health-related USSD services.

```
┌─────────────────────────────────────────────────────────────────────┐
│              TELCO CSR PARTNERSHIP MODEL                            │
│                                                                      │
│  UDARA Foundation          Telco (Safaricom, MTN, etc.)            │
│  ───────────────           ──────────────────────────────           │
│                                                                      │
│  Provides:                 Provides:                                 │
│  • Health impact data     • Zero-rated USSD shortcode              │
│  • Community trust        • Technical integration support           │
│  • Regulatory goodwill    • Marketing/awareness campaigns           │
│  • Reporting compliance   • Priority support channel                │
│                                                                      │
│  Negotiation points:                                                  │
│  ┌────────────────────────────────────────────────────┐             │
│  │ 1. Frame as PUBLIC HEALTH, not commercial          │             │
│  │ 2. Cite WHO AMR action plan (government priority)  │             │
│  │ 3. Highlight brand value: "Telco fighting AMR"     │             │
│  │ 4. Start with pilot (1 district, 3 months)         │             │
│  │ 5. Government MoH letter of support                │             │
│  │ 6. Data sharing agreement (aggregate only)          │             │
│  │ 7. Joint press release on launch                    │             │
│  └────────────────────────────────────────────────────┘             │
│                                                                      │
│  Template MOU clauses:                                               │
│  • USSD shortcode *384*UDARA# zero-rated for health workers        │
│  • No per-session charges to UDARA or CHW                           │
│  • Minimum 3-year agreement with annual review                      │
│  • Telco provides SLA for USSD uptime (99.5%)                       │
│  • UDARA provides quarterly impact reports                          │
│  • Aggregate, anonymized data shared with telco for CSR reporting   │
└─────────────────────────────────────────────────────────────────────┘
```

### 3.5 Strategy 4: Government Sponsorship

As a last resort or supplement, government Ministry of Health budgets or
donor funding (USAID, Gates Foundation, Wellcome Trust) can cover USSD costs:

```
Cost projection for government sponsorship:
  Assume: 2,000 CHWs × 5 reports/day × 22 working days/month × 12 months
  = 2,640,000 USSD sessions/year

  At KES 0.50/session (Safaricom):
    Annual cost: KES 1,320,000 ≈ USD 10,000/year

  At NGN 20/session (MTN Nigeria):
    Annual cost: NGN 52,800,000 ≈ USD 35,000/year

  Per-CHW cost: ~$5-17/year — extremely affordable for public health
```

---

## 4. USSD Menu State Machine

### 4.1 Complete State Diagram

```
┌─────────────────────────────────────────────────────────────────────┐
│                    UDARA USSD STATE MACHINE                         │
│                    Session Flow for Case Reporting                   │
│                                                                      │
│                        ┌─────────┐                                  │
│                        │  MAIN   │                                  │
│                        │  MENU   │                                  │
│                        └────┬────┘                                  │
│              ┌──────────────┼──────────────┐                        │
│              │              │              │                        │
│         [1]  │         [2]  │         [3]  │                        │
│              ▼              │              ▼                        │
│       ┌────────────┐        │       ┌────────────┐                 │
│       │  SYMPTOM   │        │       │  RESISTANCE│                 │
│       │  SELECT    │        │       │  CHECK     │                 │
│       └──────┬─────┘        │       └──────┬─────┘                 │
│              │              │              │                        │
│         [select]            │         [input]                       │
│              │              │              │                        │
│              ▼              │              ▼                        │
│       ┌────────────┐        │       ┌────────────┐                 │
│       │   AGE      │        │       │ RESISTANCE │                 │
│       │  PROMPT    │        │       │   RESULT   │                 │
│       └──────┬─────┘        │       └──────┬─────┘                 │
│              │              │              │                        │
│         [input]             │         [any] → END                   │
│              │              │                                      │
│              ▼              │                                      │
│       ┌────────────┐        │                                      │
│       │ DURATION   │        │                                      │
│       │  PROMPT    │        │                                      │
│       └──────┬─────┘        │                                      │
│              │              │                                      │
│         [input]             │                                      │
│              │              │                                      │
│              ▼              │                                      │
│       ┌────────────┐        │                                      │
│       │   DRUG     │        │                                      │
│       │  PROMPT    │        │                                      │
│       └──────┬─────┘        │                                      │
│              │              │                                      │
│         [input]             │                                      │
│              │              │                                      │
│              ▼              │                                      │
│       ┌────────────┐        │                                      │
│       │  OUTCOME   │        │                                      │
│       │  PROMPT    │        │                                      │
│       └──────┬─────┘        │                                      │
│              │              │                                      │
│         [input]             │                                      │
│              │              │                                      │
│              ▼              │                                      │
│       ┌────────────┐        │                                      │
│       │  CONFIRM   │        │                                      │
│       │  REVIEW    │        │                                      │
│       └──────┬─────┘        │                                      │
│              │              │                                      │
│        ┌─────┴─────┐        │                                      │
│     [1]│           │[2]      │                                      │
│  SUBMIT│           │CANCEL   │                                      │
│        ▼           ▼         │                                      │
│   ┌─────────┐ ┌─────────┐   │                                      │
│   │  THANK  │ │  CANCEL │   │                                      │
│   │  (END)  │ │  (END)  │   │                                      │
│   └─────────┘ └─────────┘   │                                      │
│                              │                                      │
│  From MAIN: [3] → RES_CHECK │                                      │
│              [4] → HELP     │                                      │
│              [0] → END     │                                      │
│                                                                      │
│  From any state:                                                     │
│    [0] → Return to MAIN                                             │
│    [99] → HELP                                                      │
│    Timeout → END (auto-save partial)                                 │
└─────────────────────────────────────────────────────────────────────┘
```

### 4.2 State Definitions

| State | Description | Input Type | Timeout Risk |
|-------|-------------|------------|-------------|
| `MAIN` | Top-level menu. Choose Report Case, Check Resistance, or Help | Single digit (1-4) | Low |
| `SYMPTOM` | Select primary symptom from list (paginated) | Single digit (1-9, 0=more) | Medium |
| `AGE` | Enter patient age group | Single digit (1-7) | Low |
| `DURATION` | Enter symptom duration in days | Single digit (1-9) or "0" for unknown | Low |
| `DRUG` | Enter drug given (select from list) | Single digit (1-9, 0=none) | Low |
| `OUTCOME` | Select treatment outcome after follow-up | Single digit (1-4) | Low |
| `CONFIRM` | Review all entered data. Confirm or cancel. | Single digit (1=submit, 2=cancel) | Medium |
| `HELP` | Display help text and return to MAIN | Any input → MAIN | Low |
| `RESISTANCE_CHECK` | Input drug name + district, get resistance rate | Text input (drug code) + digit (district) | High |
| `RESISTANCE_RESULT` | Display resistance rate and trend. END. | — (auto-END) | None |
| `THANK` | Submission confirmed. Thank you message. END. | — (auto-END) | None |
| `CANCEL` | Submission cancelled. Return to MAIN. | — (auto-redirect) | None |

### 4.3 Full Menu Text (160-char limit enforced)

```
STATE: MAIN
Response: CON Welcome to UDARA AMR Surveillance
1.Report case
2.Check resistance
3.Help
0.Exit

STATE: SYMPTOM (Page 1)
Response: CON Select symptom:
1.Fever
2.Cough
3.Diarrhea
4.Dysuria(pain pee)
5.Skin wound
6.Sore throat
0.More(2/3)

STATE: SYMPTOM (Page 2)
Response: CON Select symptom:
1.Headache
2.Body pain
3.Vomiting
4.Ear pain
5.Rash
6.Eye infection
0.More(3/3)

STATE: SYMPTOM (Page 3)
Response: CON Select symptom:
1.Stomach pain
2.Chest pain
3.Weakness
4.None of these
#.Back to menu
0.Main menu

STATE: AGE
Response: CON Patient age group:
1.<1yr 2.1-4 3.5-14
4.15-24 5.25-34 6.35-54
7.55+
0.Back

STATE: DURATION
Response: CON Symptom duration(days):
1.1day 2.2-3 3.4-7
4.1-2wks 5.2-4wks 6.1mth+
7.Unknown
0.Back

STATE: DRUG (Page 1)
Response: CON Drug given?
1.None
2.Amoxicillin
3.Ciprofloxacin
4.Cotrimoxazole
5.Metronidazole
6.Azithromycin
0.More(2/2)

STATE: DRUG (Page 2)
Response: CON Drug given?
1.Amox-clavulanate
2.Doxycycline
3.Nitrofurantoin
4.Fosfomycin
5.Other(specify)
#.Back to menu
0.Main menu

STATE: OUTCOME
Response: CON Outcome:
1.Improved
2.No change
3.Worse
4.Unknown
0.Back

STATE: CONFIRM
Response: CON Confirm report:
Sym:Fvr Age:25-34
Dur:4-7 Drug:Cipro
Out:Improved
1.Submit
2.Cancel
0.Main menu

STATE: HELP
Response: CON UDARA helps track drug resistance.Send us clinical reports.Choose what matches patient symptoms.Drug given=treatment prescribed. # to continue

STATE: RESISTANCE_CHECK
Response: CON Check resistance.
Enter drug code:
1.Amx 2.Cip 3.Cot
4.Met 5.Azi 6.Nit
0.Back

STATE: RESISTANCE_RESULT
Response: END Cipro in Nrb:
Rate:68%(HIGH)
Trend:Up
Alt:Nitro(12%)
Consult supervisor.
Tnx!*384*12345# to report

STATE: THANK
Response: END Report saved.
Ref:UD20250001
Tnx!*384*12345#
to report again.

STATE: CANCEL
Response: CON Report cancelled.
Start new?1.Yes 0.Exit
```

---

## 5. Response Formatting & Constraints

### 5.1 The 160-Character Constraint

Every USSD response must be ≤ 160 characters using GSM 7-bit encoding. This is
a HARD limit enforced by the telco. Responses exceeding this limit are truncated.

```python
# USSD Response Length Validator

GSM_7BIT_ALPHABET = set(
    "@£$¥èéùìòÇ\nØø\rÅåΔ_ΦΓΛΩΠΨΣΘΞÆæßÉ "
    " !\"#¤%&'()*+,-./0123456789:;<=>?"
    "ABCDEFGHIJKLMNOPQRSTUVWXYZÄÖÑÜ§"
    "abcdefghijklmnopqrstuvwxyzäöñüà"
)

GSM_ESCAPE_CHARS = {
    '^': '\x14', '{': '\x1b', '}': '\x1d', '\\': '\x00',
    '[': '\x1b\x3c', '~': '\x1b\x3d', ']': '\x1b\x3e',
    '|': '\x1b\x40', '€': '\x1b\x65',
}


def gsm7_char_count(text: str) -> int:
    """Count characters using GSM 7-bit encoding rules."""
    count = 0
    for ch in text:
        if ch in GSM_ESCAPE_CHARS:
            count += 2  # Escape sequences use 2 characters
        else:
            count += 1
    return count


def validate_ussd_response(text: str, max_chars: int = 160) -> dict:
    """
    Validate a USSD response message.

    Returns:
        {
            "valid": bool,
            "char_count": int,
            "max_chars": int,
            "over_by": int (0 if valid),
            "has_non_gsm7": bool,
            "problematic_chars": list[str],
            "suggestions": list[str]
        }
    """
    problematic = []
    has_non_gsm7 = False

    for ch in text:
        if ch not in GSM_7BIT_ALPHABET and ch not in GSM_ESCAPE_CHARS:
            has_non_gsm7 = True
            problematic.append(ch)

    count = gsm7_char_count(text)
    over_by = max(0, count - max_chars)

    suggestions = []
    if over_by > 0:
        suggestions.append(f"Over by {over_by} chars. Shorten text or split into multiple screens.")
    if has_non_gsm7:
        suggestions.append(
            f"Non-GSM7 chars: {problematic}. "
            f"Replace: é→e, ñ→n, —→-, "→', •→*"
        )

    return {
        "valid": over_by == 0 and not has_non_gsm7,
        "char_count": count,
        "max_chars": max_chars,
        "over_by": over_by,
        "has_non_gsm7": has_non_gsm7,
        "problematic_chars": problematic,
        "suggestions": suggestions,
    }


# Example validation
result = validate_ussd_response(
    "CON Welcome to UDARA AMR Surveillance\n"
    "1.Report case\n"
    "2.Check resistance\n"
    "3.Help\n"
    "0.Exit"
)
# {"valid": True, "char_count": 82, "max_chars": 160, "over_by": 0, ...}
```

### 5.2 Abbreviation Strategy

```
┌─────────────────────────────────────────────────────────────────────┐
│                    USSD ABBREVIATION TABLE                          │
├────────────────────────────┬───────────────────────────────────────┤
│ Full Term                  │ Abbreviation (USSD)                  │
├────────────────────────────┼───────────────────────────────────────┤
│ Fever                      │ Fvr / Fev                            │
│ Cough                      │ Cough (short enough)                  │
│ Diarrhea                   │ Diarrh                               │
│ Dysuria (painful urination)│ Dysuria(pain pee)                    │
│ Skin wound/infection       │ Skin wound                           │
│ Sore throat                │ Sore throat                          │
│ Headache                   │ Headache (short enough)               │
│ Vomiting                   │ Vomit                                │
│ Amoxicillin                │ Amx                                  │
│ Ciprofloxacin              │ Cip                                  │
│ Cotrimoxazole              │ Cot                                  │
│ Metronidazole              │ Met                                  │
│ Azithromycin               │ Azi                                  │
│ Nitrofurantoin             │ Nit                                  │
│ Amoxicillin-clavulanate    │ Amox-clav                            │
│ Doxycycline                │ Doxy                                 │
│ Fosfomycin                 │ Fosf                                 │
│ Improved                   │ Improved                             │
│ No change                  │ No change                            │
│ Worsened                   │ Worse                                │
│ Unknown                    │ Unknown                              │
│ Nairobi                    │ Nrb                                  │
│ Resistance                 │ Resist                               │
│ District                   │ Dist / Dst                           │
│ Report                     │ Report (short enough)                 │
│ Thank you                  │ Tnx                                  │
│ Continue                   │ #                                    │
│ Back                       │ 0                                    │
│ Exit                       │ 0                                    │
└────────────────────────────┴───────────────────────────────────────┘
```

### 5.3 CON/END Flow Control

```
┌─────────────────────────────────────────────────────────────────────┐
│                    CON / END FLOW CONTROL                            │
│                                                                      │
│  CON = Continue Session                                              │
│  ─── = Keep the USSD session open.                                  │
│        The user's phone shows the response and waits for input.     │
│        Use CON for ALL intermediate screens (menus, prompts).       │
│                                                                      │
│  END = Terminate Session                                             │
│  ─── = Close the USSD session.                                      │
│        The user's phone shows the response and returns to idle.      │
│        Use END ONLY for final screens (confirmation, error, timeout). │
│                                                                      │
│  RULES:                                                              │
│  1. Every response MUST start with CON or END (uppercase).          │
│  2. CON responses should end with a prompt or instruction.          │
│  3. END responses should include a call-to-action (redial number).  │
│  4. NEVER send END in the middle of a data collection flow.         │
│  5. If processing fails, send END with error + call-to-action.       │
│                                                                      │
│  EXAMPLES:                                                           │
│                                                                      │
│  ✅ CON Select symptom:                                             │
│     1.Fever 2.Cough 3.Diarrhea                                     │
│     0.More                                                          │
│                                                                      │
│  ✅ END Report saved.Ref:UD20250001                                 │
│     Tnx!*384*12345# to report again.                                │
│                                                                      │
│  ✅ END Error.Please try again.                                     │
│     *384*12345#                                                     │
│                                                                      │
│  ❌ CON (no text after — confusing to user)                         │
│  ❌ END Select symptom: (session ends, user can't respond)          │
│  ❌ Here's your result (missing CON/END prefix — telco rejects)      │
└─────────────────────────────────────────────────────────────────────┘
```

### 5.4 Example Screens on Phone

```
┌──────────────────────┐     ┌──────────────────────┐
│ *384*12345#          │     │ UDARA AMR            │
│ Connecting...        │     │ Surveillance         │
│                      │     │                      │
│ Sending request...   │     │ Select:              │
│                      │     │ 1.Report case        │
│ ┌──────────────────┐ │     │ 2.Check resistance   │
│ │UDARA AMR         │ │     │ 3.Help               │
│ │Surveillance      │ │     │ 0.Exit               │
│ │                  │ │     │                      │
│ │1.Report case     │ │     │ Reply: _             │
│ │2.Check resist.   │ │     └──────────────────────┘
│ │3.Help            │ │
│ │0.Exit            │ │     ┌──────────────────────┐
│ │                  │ │     │ UDARA AMR            │
│ │Reply: _          │ │     │                      │
│ └──────────────────┘ │     │ Select symptom:      │
│                      │     │ 1.Fever              │
│ User types: 1        │     │ 2.Cough              │
│                      │     │ 3.Diarrhea           │
│ ┌──────────────────┐ │     │ 4.Dysuria(pain pee)  │
│ │Select symptom:   │ │     │ 0.More               │
│ │1.Fever           │ │     │                      │
│ │2.Cough           │ │     │ Reply: _             │
│ │3.Diarrhea        │ │     └──────────────────────┘
│ │4.Dysuria(pain    │ │
│ │  pee)            │ │
│ │0.More            │ │
│ │Reply: _          │ │
│ └──────────────────┘ │
└──────────────────────┘
```

---

## 6. Session Manager Implementation

### 6.1 Full Redis-Based Session Manager

```python
# shared/ussd/session_manager.py

import json
import logging
import time
from dataclasses import dataclass, field
from datetime import datetime, timezone
from typing import Any, Dict, List, Optional

import redis

logger = logging.getLogger("udara.ussd.sessions")


# ─── Constants ──────────────────────────────────────────────────────────

USSD_SESSION_PREFIX = "udara:ussd:session:"
USSD_SESSION_TTL = 180          # 3 minutes (most telcos timeout at 120-180s)
USSD_ACTIVE_SET_KEY = "udara:ussd:active_sessions"
USSD_COUNTER_KEY = "udara:ussd:session_counter"


# ─── Session Data Model ────────────────────────────────────────────────

@dataclass
class USSDSession:
    """Represents a USSD session state."""
    session_id: str                              # Telco session ID (from Africa's Talking)
    phone_number_hash: str                       # SHA-256 of MSISDN (no PII)
    network_operator: Optional[str] = None      # "safaricom", "airtel", "mtn"
    country_code: str = "254"                    # International dialing code

    current_state: str = "MAIN"                 # State machine state
    session_data: Dict[str, Any] = field(default_factory=dict)  # Accumulated form data

    started_at: str = ""                        # ISO 8601
    last_interaction_at: str = ""               # ISO 8601
    steps_completed: int = 0                    # Number of screens visited

    # Derived/computed fields
    completed: bool = False
    case_id: Optional[str] = None               # Linked case if submitted
    final_submission: Optional[Dict] = None     # Final data if submitted

    def to_dict(self) -> Dict[str, Any]:
        return {
            "session_id": self.session_id,
            "phone_number_hash": self.phone_number_hash,
            "network_operator": self.network_operator,
            "country_code": self.country_code,
            "current_state": self.current_state,
            "session_data": self.session_data,
            "started_at": self.started_at,
            "last_interaction_at": self.last_interaction_at,
            "steps_completed": self.steps_completed,
            "completed": self.completed,
            "case_id": self.case_id,
            "final_submission": self.final_submission,
        }

    @classmethod
    def from_dict(cls, data: Dict[str, Any]) -> "USSDSession":
        return cls(**{
            k: v for k, v in data.items()
            if k in cls.__dataclass_fields__
        })


# ─── Session Manager ───────────────────────────────────────────────────

class SessionManager:
    """
    Redis-backed USSD session manager.

    Stores session state in Redis with TTL auto-expiry.
    Tracks active sessions in a Redis Set for monitoring.

    Expected Redis operations:
    - SETEX: Create/update session (O(1))
    - GET: Read session (O(1))
    - SADD/SREM: Track active sessions (O(1))
    - SCARD: Count active sessions (O(1))
    - DEL: Delete session (O(1))
    """

    def __init__(self, redis_url: str = "redis://localhost:6379/0", ttl: int = USSD_SESSION_TTL):
        self.redis = redis.from_url(redis_url, decode_responses=True)
        self.ttl = ttl
        self._validate_connection()

    def _validate_connection(self):
        """Verify Redis connection on initialization."""
        try:
            self.redis.ping()
            logger.info("SessionManager connected to Redis (TTL=%ds)", self.ttl)
        except redis.ConnectionError as e:
            raise RuntimeError(f"Cannot connect to Redis at {self.redis}: {e}") from e

    def _session_key(self, session_id: str) -> str:
        """Generate the Redis key for a session."""
        return f"{USSD_SESSION_PREFIX}{session_id}"

    # ── Create Session ─────────────────────────────────────────────

    def create_session(
        self,
        session_id: str,
        phone_number: str,
        network_operator: Optional[str] = None,
        country_code: str = "254",
    ) -> USSDSession:
        """
        Create a new USSD session.

        Args:
            session_id: Telco-assigned session ID (unique per interaction)
            phone_number: MSISDN (e.g., "+254712345678")
            network_operator: Network name (e.g., "safaricom")
            country_code: International dialing code

        Returns:
            The created USSDSession object.
        """
        now = datetime.now(timezone.utc).isoformat()

        # Hash phone number for privacy (no PII in Redis)
        phone_hash = self._hash_phone(phone_number)

        session = USSDSession(
            session_id=session_id,
            phone_number_hash=phone_hash,
            network_operator=network_operator,
            country_code=country_code,
            current_state="MAIN",
            session_data={},
            started_at=now,
            last_interaction_at=now,
            steps_completed=1,
        )

        # Store in Redis with TTL
        key = self._session_key(session_id)
        self.redis.setex(key, self.ttl, json.dumps(session.to_dict()))

        # Track in active sessions set
        self.redis.sadd(USSD_ACTIVE_SET_KEY, session_id)

        # Increment session counter
        self.redis.incr(USSD_COUNTER_KEY)

        logger.info(
            "USSD session created: %s (phone_hash=%s..., operator=%s)",
            session_id, phone_hash[:12], network_operator,
        )
        return session

    # ── Update Session ─────────────────────────────────────────────

    def update_session(
        self,
        session_id: str,
        state: Optional[str] = None,
        data: Optional[Dict[str, Any]] = None,
        merge_data: Optional[Dict[str, Any]] = None,
        completed: bool = False,
        case_id: Optional[str] = None,
        final_submission: Optional[Dict] = None,
    ) -> Optional[USSDSession]:
        """
        Update an existing USSD session.

        Args:
            session_id: Telco session ID
            state: New state (if changing state)
            data: Replace entire session_data dict
            merge_data: Merge into existing session_data (keys overwritten)
            completed: Mark session as completed
            case_id: Link to a created case
            final_submission: Set final submitted data

        Returns:
            Updated USSDSession, or None if session expired/not found.
        """
        key = self._session_key(session_id)

        # Fetch current session
        raw = self.redis.get(key)
        if raw is None:
            logger.warning("USSD session not found (expired?): %s", session_id)
            return None

        session = USSDSession.from_dict(json.loads(raw))

        # Apply updates
        if state is not None:
            session.current_state = state

        if data is not None:
            session.session_data = data

        if merge_data is not None:
            session.session_data.update(merge_data)

        if completed:
            session.completed = completed
            session.current_state = "COMPLETED"

        if case_id is not None:
            session.case_id = case_id

        if final_submission is not None:
            session.final_submission = final_submission

        session.last_interaction_at = datetime.now(timezone.utc).isoformat()
        session.steps_completed += 1

        # Store updated session (reset TTL on each interaction)
        self.redis.setex(key, self.ttl, json.dumps(session.to_dict()))

        # If completed, remove from active set
        if completed:
            self.redis.srem(USSD_ACTIVE_SET_KEY, session_id)

        logger.debug(
            "USSD session updated: %s → state=%s, steps=%d",
            session_id, session.current_state, session.steps_completed,
        )
        return session

    # ── Get Session ─────────────────────────────────────────────────

    def get_session(self, session_id: str) -> Optional[USSDSession]:
        """
        Retrieve a USSD session by session ID.

        Returns None if the session has expired or doesn't exist.
        """
        key = self._session_key(session_id)
        raw = self.redis.get(key)

        if raw is None:
            return None

        session = USSDSession.from_dict(json.loads(raw))

        # Check if session is too old (even if Redis hasn't expired it yet)
        try:
            last = datetime.fromisoformat(session.last_interaction_at)
            age = (datetime.now(timezone.utc) - last).total_seconds()
            if age > self.ttl + 30:  # 30s grace period
                logger.info("USSD session stale: %s (age=%ds)", session_id, age)
                self.end_session(session_id)
                return None
        except (ValueError, TypeError):
            pass

        return session

    # ── End Session ─────────────────────────────────────────────────

    def end_session(
        self,
        session_id: str,
        completed: bool = False,
        case_id: Optional[str] = None,
        final_submission: Optional[Dict] = None,
    ) -> bool:
        """
        End a USSD session. Removes from active tracking.

        Returns True if session existed and was ended, False if not found.
        """
        key = self._session_key(session_id)

        # Try to update the session before deleting
        raw = self.redis.get(key)
        if raw is not None:
            session = USSDSession.from_dict(json.loads(raw))
            session.completed = completed
            session.case_id = case_id
            session.final_submission = final_submission
            session.last_interaction_at = datetime.now(timezone.utc).isoformat()

            # Store final state (with shorter TTL for analytics)
            self.redis.setex(key, 3600, json.dumps(session.to_dict()))  # Keep 1h for analytics
        else:
            return False

        # Remove from active set
        self.redis.srem(USSD_ACTIVE_SET_KEY, session_id)

        logger.info(
            "USSD session ended: %s (completed=%s, case=%s)",
            session_id, completed, case_id,
        )
        return True

    # ── Get Active Count ────────────────────────────────────────────

    def get_active_count(self) -> int:
        """
        Count currently active (in-progress) USSD sessions.

        Uses Redis SCARD which is O(1).
        Note: This may include sessions that have TTL-expired but not yet
        cleaned from the set. Call cleanup_expired() periodically.
        """
        return self.redis.scard(USSD_ACTIVE_SET_KEY)

    def get_total_count(self) -> int:
        """Total sessions since counter was last reset."""
        return int(self.redis.get(USSD_COUNTER_KEY) or 0)

    # ── Cleanup ─────────────────────────────────────────────────────

    def cleanup_expired(self) -> int:
        """
        Remove expired sessions from the active set.

        Iterates through active sessions and removes any whose Redis key
        no longer exists (TTL expired). Run periodically (every 5 minutes).

        Returns:
            Number of expired sessions cleaned up.
        """
        active_ids = self.redis.smembers(USSD_ACTIVE_SET_KEY)
        cleaned = 0

        for session_id in active_ids:
            key = self._session_key(session_id)
            if not self.redis.exists(key):
                self.redis.srem(USSD_ACTIVE_SET_KEY, session_id)
                cleaned += 1

        if cleaned > 0:
            logger.info("Cleaned up %d expired USSD sessions", cleaned)

        return cleaned

    # ── Get Sessions by Phone Hash ──────────────────────────────────

    def get_recent_sessions_by_phone(
        self, phone_number: str, limit: int = 10
    ) -> List[USSDSession]:
        """
        Get recent sessions for a phone number (by hash).
        Useful for debugging and analytics.

        Note: This requires scanning keys, which is O(N).
        Only use for admin/debugging, not in hot path.
        """
        phone_hash = self._hash_phone(phone_number)
        pattern = f"{USSD_SESSION_PREFIX}*"
        sessions = []

        for key in self.redis.scan_iter(match=pattern, count=100):
            raw = self.redis.get(key)
            if raw:
                session = USSDSession.from_dict(json.loads(raw))
                if session.phone_number_hash == phone_hash:
                    sessions.append(session)
                    if len(sessions) >= limit:
                        break

        return sorted(sessions, key=lambda s: s.last_interaction_at, reverse=True)

    # ── Utilities ───────────────────────────────────────────────────

    @staticmethod
    def _hash_phone(phone_number: str) -> str:
        """Hash a phone number for privacy-safe storage."""
        import hashlib
        # Normalize phone number
        normalized = phone_number.replace("+", "").replace(" ", "").replace("-", "")
        return hashlib.sha256(normalized.encode("utf-8")).hexdigest()

    def get_session_analytics(self) -> Dict[str, Any]:
        """
        Get session analytics for monitoring dashboard.
        """
        return {
            "active_sessions": self.get_active_count(),
            "total_sessions": self.get_total_count(),
            "redis_connected": self.redis.ping(),
        }
```

---

## 7. USSD Webhook Handler

### 7.1 FastAPI Webhook Implementation

```python
# gateway/webhooks/ussd_handler.py

import hashlib
import logging
from datetime import datetime, timezone
from typing import Any, Dict, Optional

from fastapi import APIRouter, Form, HTTPException, Request, Response
from pydantic import BaseModel, Field

from shared.ussd.session_manager import SessionManager, USSDSession

logger = logging.getLogger("udara.webhooks.ussd")

router = APIRouter(prefix="/v2/webhooks", tags=["USSD"])

# State menu definitions
MENUS = {
    "MAIN": {
        "response": "CON Welcome to UDARA AMR\n1.Report case\n2.Check resistance\n3.Help\n0.Exit",
        "transitions": {"1": "SYMPTOM_P1", "2": "RESISTANCE_INPUT", "3": "HELP"},
    },
    "SYMPTOM_P1": {
        "response": "CON Select symptom:\n1.Fever\n2.Cough\n3.Diarrhea\n4.Dysuria(pain pee)\n5.Skin wound\n6.Sore throat\n0.More(2/3)",
        "transitions": {
            "1": ("SYMPTOM_AGE", {"symptom": "fever"}),
            "2": ("SYMPTOM_AGE", {"symptom": "cough"}),
            "3": ("SYMPTOM_AGE", {"symptom": "diarrhea"}),
            "4": ("SYMPTOM_AGE", {"symptom": "dysuria"}),
            "5": ("SYMPTOM_AGE", {"symptom": "skin_wound"}),
            "6": ("SYMPTOM_AGE", {"symptom": "sore_throat"}),
            "0": "SYMPTOM_P2",
        },
    },
    "SYMPTOM_P2": {
        "response": "CON Select symptom:\n1.Headache\n2.Body pain\n3.Vomiting\n4.Ear pain\n5.Rash\n6.Eye infection\n0.More(3/3)",
        "transitions": {
            "1": ("SYMPTOM_AGE", {"symptom": "headache"}),
            "2": ("SYMPTOM_AGE", {"symptom": "body_pain"}),
            "3": ("SYMPTOM_AGE", {"symptom": "vomiting"}),
            "4": ("SYMPTOM_AGE", {"symptom": "ear_pain"}),
            "5": ("SYMPTOM_AGE", {"symptom": "rash"}),
            "6": ("SYMPTOM_AGE", {"symptom": "eye_infection"}),
            "0": "SYMPTOM_P3",
        },
    },
    "SYMPTOM_P3": {
        "response": "CON Select symptom:\n1.Stomach pain\n2.Chest pain\n3.Weakness\n4.None\n#.Back\n0.Main menu",
        "transitions": {
            "1": ("SYMPTOM_AGE", {"symptom": "stomach_pain"}),
            "2": ("SYMPTOM_AGE", {"symptom": "chest_pain"}),
            "3": ("SYMPTOM_AGE", {"symptom": "weakness"}),
            "4": ("SYMPTOM_AGE", {"symptom": "none"}),
            "0": "MAIN",
        },
    },
    "SYMPTOM_AGE": {
        "response": "CON Patient age:\n1.<1yr 2.1-4 3.5-14\n4.15-24 5.25-34 6.35-54\n7.55+\n0.Back",
        "transitions": {
            "1": ("SYMPTOM_DURATION", {"age_group": "<1"}),
            "2": ("SYMPTOM_DURATION", {"age_group": "1-4"}),
            "3": ("SYMPTOM_DURATION", {"age_group": "5-14"}),
            "4": ("SYMPTOM_DURATION", {"age_group": "15-24"}),
            "5": ("SYMPTOM_DURATION", {"age_group": "25-34"}),
            "6": ("SYMPTOM_DURATION", {"age_group": "35-54"}),
            "7": ("SYMPTOM_DURATION", {"age_group": "55+"}),
            "0": "SYMPTOM_P1",
        },
    },
    "SYMPTOM_DURATION": {
        "response": "CON Duration(days):\n1.1day 2.2-3 3.4-7\n4.1-2wks 5.2-4wks 6.1mth+\n7.Unknown\n0.Back",
        "transitions": {
            "1": ("DRUG_P1", {"duration": "1"}),
            "2": ("DRUG_P1", {"duration": "2-3"}),
            "3": ("DRUG_P1", {"duration": "4-7"}),
            "4": ("DRUG_P1", {"duration": "1-2wks"}),
            "5": ("DRUG_P1", {"duration": "2-4wks"}),
            "6": ("DRUG_P1", {"duration": "1mth+"}),
            "7": ("DRUG_P1", {"duration": "unknown"}),
            "0": "SYMPTOM_AGE",
        },
    },
    "DRUG_P1": {
        "response": "CON Drug given?\n1.None\n2.Amoxicillin\n3.Ciprofloxacin\n4.Cotrimoxazole\n5.Metronidazole\n6.Azithromycin\n0.More",
        "transitions": {
            "1": ("OUTCOME", {"drug": "none"}),
            "2": ("OUTCOME", {"drug": "amoxicillin"}),
            "3": ("OUTCOME", {"drug": "ciprofloxacin"}),
            "4": ("OUTCOME", {"drug": "cotrimoxazole"}),
            "5": ("OUTCOME", {"drug": "metronidazole"}),
            "6": ("OUTCOME", {"drug": "azithromycin"}),
            "0": "DRUG_P2",
        },
    },
    "DRUG_P2": {
        "response": "CON Drug given?\n1.Amox-clavulanate\n2.Doxycycline\n3.Nitrofurantoin\n4.Fosfomycin\n5.Other(specify)\n#.Back\n0.Main menu",
        "transitions": {
            "1": ("OUTCOME", {"drug": "amoxicillin_clavulanate"}),
            "2": ("OUTCOME", {"drug": "doxycycline"}),
            "3": ("OUTCOME", {"drug": "nitrofurantoin"}),
            "4": ("OUTCOME", {"drug": "fosfomycin"}),
            "0": "MAIN",
        },
    },
    "OUTCOME": {
        "response": "CON Outcome:\n1.Improved\n2.No change\n3.Worse\n4.Unknown\n0.Back",
        "transitions": {
            "1": ("CONFIRM", {"outcome": "improved"}),
            "2": ("CONFIRM", {"outcome": "no_change"}),
            "3": ("CONFIRM", {"outcome": "worse"}),
            "4": ("CONFIRM", {"outcome": "unknown"}),
            "0": "DRUG_P1",
        },
    },
    "CONFIRM": {
        "response_template": "CON Confirm:\nSym:{symptom} Age:{age}\nDur:{duration} Drug:{drug}\nOut:{outcome}\n1.Submit 2.Cancel\n0.Main",
        "transitions": {
            "1": "SUBMIT",
            "2": "CANCEL",
            "0": "MAIN",
        },
    },
    "HELP": {
        "response": "CON UDARA tracks drug resistance.Report symptoms & drugs given.Choose what matches patient.# continue",
        "transitions": {"*": "MAIN"},  # Any input returns to MAIN
    },
    "RESISTANCE_INPUT": {
        "response": "CON Check resistance.\nEnter drug code:\n1.Amx 2.Cip 3.Cot\n4.Met 5.Azi 6.Nit\n0.Back",
        "transitions": {
            "1": ("RESISTANCE_RESULT", {"query_drug": "amoxicillin"}),
            "2": ("RESISTANCE_RESULT", {"query_drug": "ciprofloxacin"}),
            "3": ("RESISTANCE_RESULT", {"query_drug": "cotrimoxazole"}),
            "4": ("RESISTANCE_RESULT", {"query_drug": "metronidazole"}),
            "5": ("RESISTANCE_RESULT", {"query_drug": "azithromycin"}),
            "6": ("RESISTANCE_RESULT", {"query_drug": "nitrofurantoin"}),
            "0": "MAIN",
        },
    },
}

# Global session manager (initialized in app startup)
session_manager: Optional[SessionManager] = None


def init_ussd_handler(redis_url: str) -> None:
    """Initialize the USSD handler with a Redis connection."""
    global session_manager
    session_manager = SessionManager(redis_url=redis_url)
    logger.info("USSD webhook handler initialized")


# ─── Webhook Endpoint ──────────────────────────────────────────────────

@router.post("/ussd")
async def handle_ussd_webhook(
    request: Request,
    sessionId: str = Form(...),
    serviceCode: str = Form(...),
    phoneNumber: str = Form(...),
    text: str = Form(""),
    networkCode: str = Form(None),
):
    """
    Handle incoming USSD webhook from Africa's Talking.

    This endpoint receives form-encoded data from the telco gateway.
    The response body (plain text) is sent directly to the user's phone.

    Response format:
        CON <text>  → Continue session (wait for user input)
        END <text>  → End session (return to idle)
    """
    if session_manager is None:
        return Response(
            content="END Service unavailable. Try again later.",
            media_type="text/plain",
        )

    # Determine if this is a new session or continuation
    user_input = text.strip()

    if not user_input:
        # New session: user just dialed *384*12345#
        session = session_manager.create_session(
            session_id=sessionId,
            phone_number=phoneNumber,
            network_operator=networkCode,
        )
        response_text = MENUS["MAIN"]["response"]
        logger.info(
            "New USSD session: %s (phone=%s, operator=%s)",
            sessionId, phoneNumber[-4:], networkCode,
        )
    else:
        # Continuation: user responded to previous prompt
        session = session_manager.get_session(sessionId)

        if session is None:
            # Session expired — start fresh
            session = session_manager.create_session(
                session_id=sessionId,
                phone_number=phoneNumber,
                network_operator=networkCode,
            )
            response_text = "END Session expired.Please try again.\n*384*12345#"
            return Response(content=response_text, media_type="text/plain")

        # Process the user's input based on current state
        response_text = process_state_transition(session, user_input)

    return Response(content=response_text, media_type="text/plain")


def process_state_transition(session: USSDSession, user_input: str) -> str:
    """
    Process user input and determine the next state/response.

    Args:
        session: Current USSD session
        user_input: The digit(s) the user entered

    Returns:
        Response text (starting with CON or END)
    """
    current_state = session.current_state

    # Global commands
    if user_input == "0" and current_state not in ("SYMPTOM_AGE", "SYMPTOM_DURATION", "OUTCOME"):
        session_manager.update_session(session.session_id, state="MAIN")
        return MENUS["MAIN"]["response"]

    if user_input == "99":
        session_manager.update_session(session.session_id, state="HELP")
        return MENUS["HELP"]["response"]

    # Get current menu definition
    menu = MENUS.get(current_state)

    if menu is None:
        logger.error("Unknown USSD state: %s", current_state)
        return "END Error.Please try again.\n*384*12345#"

    # Handle special states
    if current_state == "SUBMIT":
        return handle_submission(session)

    if current_state == "CANCEL":
        session_manager.end_session(session.session_id)
        session_manager.update_session(session.session_id, state="MAIN")
        return "CON Report cancelled.\nStart new?1.Yes 0.Exit"

    if current_state == "CONFIRM":
        return handle_confirm(session, user_input)

    if current_state == "RESISTANCE_RESULT":
        return handle_resistance_result(session)

    # Standard state transition
    transitions = menu.get("transitions", {})
    transition = transitions.get(user_input)

    if transition is None:
        # Invalid input — re-display current menu
        return menu.get("response", f"CON Invalid choice.\n{menu.get('response', '')}")

    if isinstance(transition, tuple):
        # Transition with data merge: ("NEXT_STATE", {"key": "value"})
        next_state, merge_data = transition
        session_manager.update_session(
            session.session_id,
            state=next_state,
            merge_data=merge_data,
        )
    else:
        # Simple state transition: "NEXT_STATE"
        session_manager.update_session(session.session_id, state=transition)

    # Handle wildcard transitions (any input)
    if isinstance(transitions.get("*"), str):
        next_state = transitions["*"]
        session_manager.update_session(session.session_id, state=next_state)

    # Get response for the new state
    next_menu = MENUS.get(
        session_manager.get_session(session.session_id).current_state
    )
    if next_menu is None:
        return "END Error.Please try again.\n*384*12345#"

    # Handle template-based responses (e.g., CONFIRM with dynamic data)
    if "response_template" in next_menu:
        return render_template(next_menu["response_template"], session.session_data)

    return next_menu.get("response", "END Service error.")


def render_template(template: str, data: Dict[str, Any]) -> str:
    """Render a response template with session data."""
    abbreviations = {
        "fever": "Fvr", "cough": "Cough", "diarrhea": "Diarrh",
        "dysuria": "Dysur", "skin_wound": "Wound", "sore_throat": "Thr",
        "headache": "Head", "body_pain": "Pain", "vomiting": "Vomit",
        "ear_pain": "Ear", "rash": "Rash", "eye_infection": "Eye",
        "stomach_pain": "Stom", "chest_pain": "Chest", "weakness": "Weak",
        "none": "None",
        "amoxicillin": "Amx", "ciprofloxacin": "Cip",
        "cotrimoxazole": "Cot", "metronidazole": "Met",
        "azithromycin": "Azi", "nitrofurantoin": "Nit",
        "amoxicillin_clavulanate": "Amx-C",
        "doxycycline": "Doxy", "fosfomycin": "Fosf",
        "improved": "Improved", "no_change": "No chg", "worse": "Worse",
        "unknown": "Unkn",
    }

    def abbrev(val):
        if val is None:
            return "?"
        return abbreviations.get(val, str(val)[:5])

    try:
        return template.format(
            symptom=abbrev(data.get("symptom")),
            age=data.get("age_group", "?"),
            duration=data.get("duration", "?"),
            drug=abbrev(data.get("drug")),
            outcome=abbrev(data.get("outcome")),
        )
    except KeyError:
        return template  # Fallback to raw template


def handle_confirm(session: USSDSession, user_input: str) -> str:
    """Handle confirm/cancel at the CONFIRM state."""
    if user_input == "1":
        session_manager.update_session(session.session_id, state="SUBMIT")
        return handle_submission(session)
    elif user_input == "2":
        session_manager.update_session(session.session_id, state="CANCEL")
        return "CON Report cancelled.\nStart new?1.Yes 0.Exit"
    else:
        confirm_menu = MENUS.get("CONFIRM", {})
        return confirm_menu.get("response", "CON Invalid.")


def handle_submission(session: USSDSession) -> str:
    """
    Submit the collected case data to Agent A.

    Converts USSD session data to a StructuredCase and sends to the
    gateway's ingest endpoint.
    """
    import httpx
    import uuid

    data = session.session_data

    try:
        # Call Agent A via gateway (internal HTTP call)
        case_response = httpx.post(
            "http://agent-a:8001/ingest/ussd",
            json={
                "session_id": session.session_id,
                "phone_hash": session.phone_number_hash,
                "symptom": data.get("symptom"),
                "age_group": data.get("age_group"),
                "duration": data.get("duration"),
                "drug": data.get("drug"),
                "outcome": data.get("outcome"),
                "network_operator": session.network_operator,
                "source": "ussd",
            },
            timeout=10.0,
        ).json()

        case_id = case_response.get("case_id", str(uuid.uuid4()))

        # End session with success
        session_manager.end_session(
            session.session_id,
            completed=True,
            case_id=case_id,
            final_submission=data,
        )

        return f"END Report saved.\nRef:{case_id[:8]}\nTnx!*384*12345#"

    except Exception as e:
        logger.error("USSD submission failed: %s", e)
        return "END Error saving.Try again.\n*384*12345#"


def handle_resistance_result(session: USSDSession) -> str:
    """
    Query resistance data for the selected drug and return result.
    """
    import httpx

    data = session.session_data
    query_drug = data.get("query_drug", "ciprofloxacin")

    try:
        response = httpx.get(
            f"http://agent-b:8002/resistance/{query_drug}/default",
            timeout=5.0,
        ).json()

        rate_pct = response.get("rate_percentage", "?")
        trend = response.get("trend", "?")
        threshold = response.get("threshold_exceeded", False)
        severity = "HIGH" if threshold else "OK"

        result_text = (
            f"END {query_drug[:3].title()} resistance:\n"
            f"Rate:{rate_pct}%({severity})\n"
            f"Trend:{trend}\n"
            f"Consult supervisor.\n"
            f"*384*12345# to report"
        )

        session_manager.end_session(session.session_id, completed=True)
        return result_text

    except Exception as e:
        logger.error("Resistance query failed: %s", e)
        session_manager.end_session(session.session_id, completed=False)
        return "END Data unavailable.\nTry again later.\n*384*12345#"
```

---

## 8. Integration with UDARA Agents

### 8.1 USSD → Agent A Data Mapping

| USSD Field | Session Key | Agent A Field | Notes |
|-----------|-------------|---------------|-------|
| Symptom selection | `symptom` | `symptoms[0].name` | Single symptom (USSD limitation) |
| Age group | `age_group` | `patient.age_group` | Categorical, not exact age |
| Duration | `duration` | `symptoms[0].duration_days` | Range or exact |
| Drug given | `drug` | `drugs_prescribed[0].name` | Single drug selection |
| Outcome | `outcome` | `resolution_outcome` | Follow-up data |
| Phone hash | `phone_number_hash` | `metadata.phone_hash` | For deduplication |
| Network | `network_operator` | `metadata.network` | Safaricom, Airtel, etc. |

### 8.2 USSD Data Quality Considerations

```
┌─────────────────────────────────────────────────────────────────────┐
│              USSD vs APP DATA QUALITY COMPARISON                    │
├──────────────────────┬──────────────────┬──────────────────────────┤
│ Aspect               │ USSD             │ Mobile App               │
├──────────────────────┼──────────────────┼──────────────────────────┤
│ Symptoms captured    │ 1 (primary only) │ Multiple (full list)     │
│ Symptom severity     │ No               │ Yes (mild/moderate/severe)│
│ Symptom duration     │ Range (4-7 days) │ Exact (5 days)           │
│ Drug dose            │ No               │ Yes (500mg)              │
│ Drug frequency       │ No               │ Yes (bid)                │
│ Drug duration        │ No               │ Yes (7 days)             │
│ Drug source          │ No               │ Yes (prescription/OTC)   │
│ Diagnosis            │ No               │ Yes (AI-suggested)       │
│ Patient age          │ Group (25-34)     │ Exact (28) + group       │
│ Multiple drugs       │ No               │ Yes                      │
│ Photo evidence       │ No               │ Yes                      │
│ Audio notes          │ No               │ Yes                      │
│ Free-text notes      │ No               │ Yes                      │
│ GPS location         │ No               │ Yes                      │
├──────────────────────┼──────────────────┼──────────────────────────┤
│ Data completeness    │ ~30%             │ ~85%                      │
│ AMR analysis utility │ Low-medium       │ High                      │
│ Training requirement │ Minimal          │ Moderate                  │
└──────────────────────┴──────────────────┴──────────────────────────┘
```

**Key insight:** USSD data is lower fidelity than app data, but it captures
cases that would otherwise be completely invisible to the surveillance system.
Even partial data (symptom + drug) is valuable for resistance rate estimation.

---

## 9. Cost Analysis

### 9.1 Per-Provider Cost Breakdown

| Provider | Country | Cost/Session | Cost/Month (2K CHWs, 5/day) | Cost/Year | Zero-Rate Possible |
|----------|---------|-------------|---------------------------|----------|-------------------|
| Safaricom | Kenya | KES 0.50 ($0.004) | $400 | $4,800 | Yes (CSR program) |
| Airtel | Kenya | KES 1.00 ($0.008) | $800 | $9,600 | Yes |
| MTN | Nigeria | NGN 20 ($0.013) | $1,300 | $15,600 | Yes (CSR program) |
| MTN | Ghana | GHS 0.30 ($0.025) | $2,500 | $30,000 | Yes |
| Vodacom | Tanzania | TZS 100 ($0.038) | $3,800 | $45,600 | Negotiable |
| Airtel | Uganda | UGX 100 ($0.026) | $2,600 | $31,200 | Negotiable |
| Ethio Telecom | Ethiopia | ETB 1.50 ($0.013) | $1,300 | $15,600 | Government |
| MTN | Rwanda | RWF 20 ($0.015) | $1,500 | $18,000 | Yes |
| Vodacom | Mozambique | MZN 2.00 ($0.031) | $3,100 | $37,200 | Negotiable |

### 9.2 Cost Optimization Strategies

```
┌─────────────────────────────────────────────────────────────────────┐
│                    USSD COST OPTIMIZATION                            │
│                                                                      │
│  Strategy                  Impact          Implementation           │
│  ───────────────────────── ─────────────── ───────────────────────  │
│  Telco zero-rating         90-100% saving  CSR partnership (primary) │
│  Reduce menu steps         10-20% saving   Fewer screens = fewer    │
│                                             session round-trips      │
│  Pre-fill known data       5-10% saving    Remember CHW's facility,  │
│                                             district from last visit │
│  Batch reports             30-50% saving   Multiple cases per       │
│                                             session (future feature) │
│  SMS for confirmations     20-30% saving   Use cheap SMS instead of  │
│                                             USSD for non-interactive │
│  Offline → sync later      100% saving     Paper forms + batch entry │
│  Switch to app users       100% saving     Migrate CHWs to Android   │
│                                             app where possible       │
│                                                                      │
│  PROJECTED COST WITH OPTIMIZATIONS:                                  │
│                                                                      │
│  Without optimization:  $4,800 - $45,600/year per country             │
│  With telco zero-rate:  $0/year (best case)                          │
│  With 50% migration:  $2,400 - $22,800/year per country               │
│  Hybrid (zero-rate +   :  $500 - $5,000/year per country            │
│    30% app migration)                                               │
└─────────────────────────────────────────────────────────────────────┘
```

---

## 10. Testing & Simulation

### 10.1 Africa's Talking Simulator Commands

```bash
# Test the full USSD flow using AT Simulator

# 1. Start local server
docker compose up -d gateway
ngrok http 8000  # Get public URL for AT webhook

# 2. Configure AT Sandbox callback URL
# Go to: https://dashboard.africastalking.com/sandbox
# Set Callback URL: https://your-ngrok-url/v2/webhooks/ussd

# 3. Open Simulator: https://simulator.africastalking.com:16000

# 4. Test Flow:
# Dial: *384*12345#
# Response: "Welcome to UDARA AMR..."
# Reply: 1 → "Select symptom:"
# Reply: 1 → "Patient age:"
# Reply: 5 → "Duration:"
# Reply: 3 → "Drug given?"
# Reply: 3 → "Outcome:"
# Reply: 1 → "Confirm:"
# Reply: 1 → "Report saved. Ref:..."
```

### 10.2 Automated Integration Test

```python
# tests/integration/test_ussd_flow.py

import pytest
from fastapi.testclient import TestClient
from shared.ussd.session_manager import SessionManager


class TestUSSDFlow:
    """Integration tests for the USSD reporting flow."""

    def test_full_case_report_flow(self, client: TestClient, redis_url: str):
        """Test complete case reporting: MAIN → SYMPTOM → AGE → DURATION → DRUG → OUTCOME → CONFIRM → SUBMIT"""

        session_id = "test-session-001"
        phone = "+254712345678"

        # Step 1: Start session (dial *384*12345#)
        resp = client.post("/v2/webhooks/ussd", data={
            "sessionId": session_id,
            "serviceCode": "*384*12345#",
            "phoneNumber": phone,
            "text": "",
            "networkCode": "63902",
        })
        assert resp.status_code == 200
        assert resp.text.startswith("CON")
        assert "Report case" in resp.text

        # Step 2: Select "Report case" (1)
        resp = client.post("/v2/webhooks/ussd", data={
            "sessionId": session_id,
            "serviceCode": "*384*12345#",
            "phoneNumber": phone,
            "text": "1",
            "networkCode": "63902",
        })
        assert "Select symptom" in resp.text
        assert "Fever" in resp.text

        # Step 3: Select "Fever" (1)
        resp = client.post("/v2/webhooks/ussd", data={
            "sessionId": session_id,
            "serviceCode": "*384*12345#",
            "phoneNumber": phone,
            "text": "1",
            "networkCode": "63902",
        })
        assert "Patient age" in resp.text

        # Step 4: Select age 25-34 (5)
        resp = client.post("/v2/webhooks/ussd", data={
            "sessionId": session_id,
            "serviceCode": "*384*12345#",
            "phoneNumber": phone,
            "text": "5",
            "networkCode": "63902",
        })
        assert "Duration" in resp.text

        # Step 5: Select duration 4-7 days (3)
        resp = client.post("/v2/webhooks/ussd", data={
            "sessionId": session_id,
            "serviceCode": "*384*12345#",
            "phoneNumber": phone,
            "text": "3",
            "networkCode": "63902",
        })
        assert "Drug given" in resp.text

        # Step 6: Select Ciprofloxacin (3)
        resp = client.post("/v2/webhooks/ussd", data={
            "sessionId": session_id,
            "serviceCode": "*384*12345#",
            "phoneNumber": phone,
            "text": "3",
            "networkCode": "63902",
        })
        assert "Outcome" in resp.text

        # Step 7: Select Improved (1)
        resp = client.post("/v2/webhooks/ussd", data={
            "sessionId": session_id,
            "serviceCode": "*384*12345#",
            "phoneNumber": phone,
            "text": "1",
            "networkCode": "63902",
        })
        assert "Confirm" in resp.text
        assert "Fvr" in resp.text
        assert "Cip" in resp.text

        # Step 8: Submit (1)
        resp = client.post("/v2/webhooks/ussd", data={
            "sessionId": session_id,
            "serviceCode": "*384*12345#",
            "phoneNumber": phone,
            "text": "1",
            "networkCode": "63902",
        })
        assert resp.text.startswith("END")
        assert "Report saved" in resp.text

    def test_session_expiry(self, client: TestClient):
        """Test that expired sessions are handled gracefully."""
        resp = client.post("/v2/webhooks/ussd", data={
            "sessionId": "expired-session-xyz",
            "serviceCode": "*384*12345#",
            "phoneNumber": "+254712345678",
            "text": "1",  # Trying to continue an expired session
        })
        assert "expired" in resp.text.lower() or "try again" in resp.text.lower()

    def test_resistance_check_flow(self, client: TestClient):
        """Test the resistance check flow: MAIN → RESISTANCE_INPUT → RESISTANCE_RESULT"""
        session_id = "test-resistance-001"

        # Start
        resp = client.post("/v2/webhooks/ussd", data={
            "sessionId": session_id,
            "serviceCode": "*384*12345#",
            "phoneNumber": "+254712345678",
            "text": "",
        })

        # Select Check resistance (2)
        resp = client.post("/v2/webhooks/ussd", data={
            "sessionId": session_id,
            "serviceCode": "*384*12345#",
            "phoneNumber": "+254712345678",
            "text": "2",
        })
        assert "drug code" in resp.text.lower()

    def test_all_responses_under_160_chars(self):
        """Verify every menu response is within GSM 7-bit 160 char limit."""
        from shared.ussd.session_manager import validate_ussd_response

        for state_name, menu in MENUS.items():
            response = menu.get("response", "")
            if not response:
                continue

            # Strip CON/END prefix for measurement
            body = response[4:] if response.startswith(("CON ", "END ")) else response

            result = validate_ussd_response(body, max_chars=160)
            assert result["valid"], (
                f"State {state_name} response exceeds 160 chars or has "
                f"non-GSM7 chars: {result}"
            )
```

---

## 11. Monitoring & Analytics

### 11.1 USSD Metrics

```
# Prometheus metrics for USSD monitoring
#
# HELP udara_ussd_sessions_total Total USSD sessions initiated
# TYPE udara_ussd_sessions_total counter
udara_ussd_sessions_total{network="safaricom",country="KEN"} 15420
udara_ussd_sessions_total{network="airtel",country="KEN"} 3210

# HELP udara_ussd_sessions_active Active USSD sessions right now
# TYPE udara_ussd_sessions_active gauge
udara_ussd_sessions_active 7

# HELP udara_ussd_sessions_completed_total Completed (submitted) USSD sessions
# TYPE udara_ussd_sessions_completed_total counter
udara_ussd_sessions_completed_total 11200

# HELP udara_ussd_sessions_abandoned_total Sessions that timed out without completion
# TYPE udara_ussd_sessions_abandoned_total counter
udara_ussd_sessions_abandoned_total 2310

# HELP udara_ussd_session_duration_seconds Duration of USSD sessions
# TYPE udara_ussd_session_duration_seconds histogram
udara_ussd_session_duration_seconds_bucket{le="30"} 8900
udara_ussd_session_duration_seconds_bucket{le="60"} 14100
udara_ussd_session_duration_seconds_bucket{le="120"} 15300
udara_ussd_session_duration_seconds_bucket{le="+Inf"} 15420

# HELP udara_ussd_steps_per_session Number of screens visited per session
# TYPE udara_ussd_steps_per_session histogram
udara_ussd_steps_per_session_bucket{le="3"} 2100
udara_ussd_steps_per_session_bucket{le="5"} 8900
udara_ussd_steps_per_session_bucket{le="8"} 14500
udara_ussd_steps_per_session_bucket{le="+Inf"} 15420

# HELP udara_ussd_symptom_reported_total Symptoms reported via USSD
# TYPE udara_ussd_symptom_reported_total counter
udara_ussd_symptom_reported_total{symptom="fever"} 5200
udara_ussd_symptom_reported_total{symptom="cough"} 3100
udara_ussd_symptom_reported_total{symptom="diarrhea"} 2400

# HELP udara_ussd_drug_reported_total Drugs reported via USSD
# TYPE udara_ussd_drug_reported_total counter
udara_ussd_drug_reported_total{drug="ciprofloxacin"} 4100
udara_ussd_drug_reported_total{drug="amoxicillin"} 3200
udara_ussd_drug_reported_total{drug="none"} 2100
```

### 11.2 Abandonment Funnel

```
SESSIONS           COUNT    % OF START   CUMULATIVE DROP
────────────────   ──────   ──────────   ───────────────
Start (MAIN)       15,420   100%         0%
→ Symptom          14,100    91.4%       8.6%     ← User exits here most
→ Age              13,200    85.6%       14.4%
→ Duration         12,800    83.0%       17.0%
→ Drug             12,300    79.8%       20.2%
→ Outcome          11,900    77.2%       22.8%
→ Confirm          11,500    74.6%       25.4%
→ Submit (END)     11,200    72.6%       27.4%    ← 72.6% completion rate
  ┌────────────────────────────────────────────────────────┐
  │ Completion rate: 72.6% (target: >70%)                 │
  │ Biggest drop: MAIN → Symptom (8.6%)                    │
  │ Action: Simplify MAIN menu text, add "2.Check resist" │
  └────────────────────────────────────────────────────────┘
```
