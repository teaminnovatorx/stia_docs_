# UDARA AI — Dual Messenger Strategy: WhatsApp + Telegram

> **Document ID:** UDARA-ARCH-010
> **Version:** 2.1.0
> **Last Updated:** 2026-05-27
> **Owner:** Platform Engineering — Bot Integrations Team
> **Status:** Approved
> **Review Cycle:** Per Sprint

---

## Table of Contents

1. [Executive Summary](#1-executive-summary)
2. [Why Dual Messenger: The Strategic Rationale](#2-why-dual-messenger-the-strategic-rationale)
3. [Country-by-Country Channel Strategy](#3-country-by-country-channel-strategy)
4. [Unified Bot Intelligence Core Architecture](#4-unified-bot-intelligence-core-architecture)
5. [Telegram Bot Implementation](#5-telegram-bot-implementation)
6. [WhatsApp Bot Implementation](#6-whatsapp-bot-implementation)
7. [Onboarding Flow — 5-Step FSM](#7-onboarding-flow--5-step-fsm)
8. [Resistance Calculator Conversation](#8-resistance-calculator-conversation)
9. [Daily Digest Scheduler](#9-daily-digest-scheduler)
10. [Peer Alert Network](#10-peer-alert-network)
11. [Country Configuration Templates](#11-country-configuration-templates)
12. [Multilingual Support](#12-multilingual-support)
13. [Testing Strategy](#13-testing-strategy)
14. [Operational Runbooks](#14-operational-runbooks)
15. [Appendix: Full Source Code Index](#15-appendix-full-source-code-index)

---

## 1. Executive Summary

UDARA AI provides AMR (Antimicrobial Resistance) surveillance intelligence
to Community Health Workers (CHWs) across sub-Saharan Africa. CHWs interact
with the platform through **two messenger channels**: **WhatsApp** and **Telegram**.

This document defines the architecture, implementation, and operational
details of the dual-messenger strategy.

### Key Metrics at a Glance

| Metric | WhatsApp | Telegram |
|--------|----------|----------|
| Monthly Active Users (target) | 5,000+ | 3,000+ |
| Cost per message (outbound) | $0.005–$0.08 | $0.000 |
| Cost per message (media) | $0.02–$0.10 | $0.000 |
| Estimated monthly cost @ 10K users | ~$7,500 | ~$0 |
| Avg latency (text) | 200–500 ms | 50–150 ms |
| Max media size | 16 MB | 50 MB |
| Bot API rate limit | 80 msg/sec (Business) | 30 msg/sec |
| Rich UI support | Interactive buttons (max 3) | Inline keyboards, custom keyboards |
| Voice message handling | Via Cloud API download | Native bot API |
| End-to-end encryption | ✅ Yes (Signal protocol) | ❌ No (server-side) |
| Offline reach | ✅ SMS fallback | ❌ Requires internet |
| Required infrastructure | Meta Business verification | Self-hosted bot server |

### Total Platform Cost Projection

```
Scenario A: WhatsApp-only @ 10K users
  Messages/month ≈ 300,000 (30/user × 10,000)
  Cost ≈ $7,500/month = $90,000/year

Scenario B: Telegram-only @ 10K users
  Messages/month ≈ 300,000
  Cost ≈ $0/month = $0/year
  (Hosting: ~$20/month on a small VPS)

Scenario C: Dual messenger (recommended)
  WhatsApp: 5,000 users × 30 msg × $0.005 = $750/month
  Telegram: 5,000 users × 30 msg × $0.000 = $0/month
  Hosting:                              $20/month
  Total:                                ~$770/month = $9,240/year

  💰 SAVINGS vs WhatsApp-only: $80,760/year (89.7% reduction)
```

---

## 2. Why Dual Messenger: The Strategic Rationale

### 2.1 WhatsApp Dominance in West Africa

WhatsApp is the dominant messaging platform in Nigeria and Ghana:

- **Nigeria**: 50M+ active users (65% of internet users)
- **Ghana**: 5M+ active users (48% of internet users)
- CHWs in these countries already use WhatsApp for work coordination
- Building on existing behavior = higher adoption, lower training cost

### 2.2 Telegram's Cost Advantage and Feature Supremacy

Telegram is FREE for bot developers — no per-message charges whatsoever.
This is critical for a non-profit AMR surveillance system:

```
Cost Comparison at Scale (messages per month):

  100K msgs:   WhatsApp $750  |  Telegram $0
  500K msgs:   WhatsApp $3,750 |  Telegram $0
  1M msgs:     WhatsApp $7,500 |  Telegram $0
  5M msgs:     WhatsApp $37,500|  Telegram $0

  The savings fund 2 full-time engineers in Africa.
```

Telegram also offers superior bot developer features:

| Feature | WhatsApp | Telegram |
|---------|----------|----------|
| Inline keyboards | ❌ No (max 3 buttons via interactive) | ✅ Unlimited rows × columns |
| Custom keyboards | ❌ No | ✅ Persistent keypads |
| Callback queries | ✅ Limited | ✅ Full support with data payload |
| File uploads from bot | ❌ No (media URL only) | ✅ Direct upload |
| Voice transcription | ❌ No | ✅ Via bot API |
| Bot descriptions | ❌ No | ✅ `set_my_description` |
| Commands menu | ❌ No | ✅ `set_my_commands` |
| Web app (mini app) | ✅ Limited | ✅ Full Web App support |
| Payment integration | ❌ No | ✅ Native |
| Topic threads | ❌ No | ✅ Forum groups |
| Stories | ❌ No | ✅ Bot stories |

### 2.3 Risk Mitigation: Platform Diversification

Relying on a single platform creates vendor lock-in risk:

```
  Single Platform Risk:
  ┌─────────────────────────────────────────────────┐
  │  API changes → downtime                          │
  │  Pricing changes → budget crisis                 │
  │  Account suspension → total service loss         │
  │  Government blocking → regional blackout         │
  │  Terms of service change → feature removal       │
  └─────────────────────────────────────────────────┘

  Dual Platform Mitigation:
  ┌──────────┐   ┌──────────┐
  │WhatsApp  │   │Telegram  │
  │(Meta)    │   │(Durov)   │
  └────┬─────┘   └────┬─────┘
       │              │
       └──────┬───────┘
              │
     ┌────────▼────────┐
     │  Bot Intelligence│
     │  Core (platform- │
     │  agnostic)       │
     └─────────────────┘
       If one fails, the
       other continues.
```

---

## 3. Country-by-Country Channel Strategy

### 3.1 Primary Channel Assignment Table

| Country | ISO Code | Population | Internet Penetration | Primary Channel | Secondary Channel | Rationale |
|---------|----------|------------|---------------------|-----------------|-------------------|-----------|
| **Nigeria** | NG | 223M | 55.4% | **WhatsApp** | Telegram | Largest WhatsApp market in Africa (50M+ users). CHWs familiar with the platform. |
| **Kenya** | KE | 55M | 85.0% | **Telegram** | WhatsApp | High internet penetration. Strong developer community on Telegram. Cost savings priority. |
| **Tanzania** | TZ | 65M | 35.0% | **Telegram** | USSD | Lower smartphone penetration. Telegram lightweight. USSD as fallback. |
| **Ghana** | GH | 34M | 58.0% | **WhatsApp** | Telegram | Similar to Nigeria. WhatsApp-first culture. |
| **Uganda** | UG | 48M | 46.0% | **Telegram** | USSD | Growing Telegram adoption. Cost-sensitive market. |
| **Ethiopia** | ET | 126M | 25.0% | **Telegram** | USSD | Government restrictions on some platforms. Telegram more accessible. |

### 3.2 Channel Selection Decision Matrix

```
                        │ WhatsApp Primary? │ Telegram Primary?
  ┌────────────────────┼───────────────────┼───────────────────┐
  │ >50M WA users?     │ YES → WA primary  │ NO                │
  │ Cost sensitivity?  │ Low priority      │ HIGH → TG primary │
  │ Internet > 50%?    │ Any               │ Any               │
  │ CHW familiarity?   │ High → WA         │ Medium → TG       │
  │ USSD fallback?     │ Recommended       │ Recommended       │
  └────────────────────┴───────────────────┴───────────────────┘
```

### 3.3 Channel Migration Path

When onboarding a new country, the default is **Telegram** (free) with a
planned migration to **WhatsApp** once the user base exceeds 1,000 active
CHWs and funding is secured:

```
Phase 1 (0–1K users):  Telegram only         → $0/month
Phase 2 (1K–5K users): Telegram + evaluate    → ~$0/month
Phase 3 (5K+ users):    Dual (TG + WA)        → ~$400/month
Phase 4 (10K+ users):   Dual, WA primary      → ~$770/month
```

---

## 4. Unified Bot Intelligence Core Architecture

### 4.1 High-Level Architecture Diagram

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                          UDARA BOT ARCHITECTURE                              │
│                                                                             │
│  ┌──────────────────────┐       ┌──────────────────────┐                   │
│  │   WHATSAPP ADAPTER   │       │   TELEGRAM ADAPTER    │                   │
│  │                      │       │                      │                   │
│  │  ┌────────────────┐  │       │  ┌────────────────┐  │                   │
│  │  │ Meta Cloud API │  │       │  │ aiogram 3.x    │  │                   │
│  │  │ Webhook Handler│  │       │  │ Polling/Webhook│  │                   │
│  │  └───────┬────────┘  │       │  └───────┬────────┘  │                   │
│  │          │           │       │          │           │                   │
│  │  ┌───────▼────────┐  │       │  ┌───────▼────────┐  │                   │
│  │  │ WhatsAppBot    │  │       │  │ TelegramBot    │  │                   │
│  │  │ - send_text()  │  │       │  │ - handlers/    │  │                   │
│  │  │ - send_btns()  │  │       │  │ - keyboards/   │  │                   │
│  │  │ - download()   │  │       │  │ - formatters/  │  │                   │
│  │  └───────┬────────┘  │       │  └───────┬────────┘  │                   │
│  └──────────┼───────────┘       └──────────┼───────────┘                   │
│             │                              │                               │
│             │    Unified Message Schema     │                               │
│             │    ┌──────────────────┐      │                               │
│             └───►│ IncomingMessage  │◄─────┘                               │
│                  │ - user_id        │                                      │
│                  │ - channel        │                                      │
│                  │ - text           │                                      │
│                  │ - media_url      │                                      │
│                  │ - media_type     │                                      │
│                  │ - timestamp      │                                      │
│                  │ - session_id     │                                      │
│                  └────────┬─────────┘                                      │
│                           │                                                 │
│  ┌────────────────────────▼─────────────────────────────────────┐          │
│  │                    BOT INTELLIGENCE CORE                       │          │
│  │                                                              │          │
│  │  ┌─────────────┐  ┌──────────────┐  ┌──────────────────┐    │          │
│  │  │  Onboarding  │  │  Resistance  │  │   Daily Digest   │    │          │
│  │  │  FSM Engine  │  │  Calculator  │  │   Scheduler      │    │          │
│  │  │  (5 steps)   │  │  Pipeline    │  │  (cron/tz)       │    │          │
│  │  └──────┬──────┘  └──────┬───────┘  └──────┬───────────┘    │          │
│  │         │                │                  │               │          │
│  │  ┌──────▼────────────────▼──────────────────▼───────────┐    │          │
│  │  │              Conversation Router                      │    │          │
│  │  │  /start → Onboarding FSM                              │    │          │
│  │  │  /resist → Resistance Calculator                      │    │          │
│  │  │  /digest → Manual digest trigger                      │    │          │
│  │  │  /alert → Peer alert subscribe                        │    │          │
│  │  │  text → Intent classification (LLM)                   │    │          │
│  │  └────────────────────────┬──────────────────────────────┘    │          │
│  │                           │                                   │          │
│  │  ┌────────────────────────▼──────────────────────────────┐    │          │
│  │  │              Response Formatter                        │    │          │
│  │  │  per-channel: USSD 160ch | SMS 160ch | TG MD | WA ★   │    │          │
│  │  └────────────────────────┬──────────────────────────────┘    │          │
│  └───────────────────────────┼───────────────────────────────────┘          │
│                              │                                               │
│             ┌────────────────▼────────────────┐                              │
│             │        AGENT CLIENT             │                              │
│             │  (gRPC to ML Pipeline)          │                              │
│             └────────┬──────────┬────────────┘                              │
│                      │          │                                           │
│         ┌────────────▼──┐  ┌───▼──────────┐  ┌──────────────┐              │
│         │   Agent A     │  │   Agent B    │  │   Agent C    │              │
│         │  Extraction   │  │  Resistance  │  │   LLM RAG    │              │
│         │  (NER/OCR/ASR)│  │  (Bayesian)  │  │  (Guidance)  │              │
│         └───────────────┘  └──────────────┘  └──────────────┘              │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 4.2 Unified Message Schema

```python
# bots/core/schemas.py

from dataclasses import dataclass, field
from datetime import datetime
from enum import Enum
from typing import Optional


class Channel(str, Enum):
    WHATSAPP = "whatsapp"
    TELEGRAM = "telegram"
    USSD = "ussd"
    SMS = "sms"


class MediaType(str, Enum):
    TEXT = "text"
    PHOTO = "photo"
    VOICE = "voice"
    VIDEO = "video"
    DOCUMENT = "document"


@dataclass
class IncomingMessage:
    """Normalized incoming message from any channel."""

    user_id: str
    channel: Channel
    text: Optional[str] = None
    media_url: Optional[str] = None
    media_type: Optional[MediaType] = None
    media_file_id: Optional[str] = None          # Telegram file_id
    caption: Optional[str] = None                # Media caption
    timestamp: datetime = field(default_factory=datetime.utcnow)
    session_id: Optional[str] = None
    phone_number: Optional[str] = None
    locale: Optional[str] = None                 # "en", "sw", "am", "ha", "yo", "ig"
    latitude: Optional[float] = None
    longitude: Optional[float] = None
    callback_data: Optional[str] = None          # Telegram callback query
    contact: Optional[dict] = None               # Shared contact info

    @property
    def has_media(self) -> bool:
        return self.media_type is not None and self.media_type != MediaType.TEXT

    @property
    def content(self) -> str:
        """Get the primary text content from message."""
        if self.text:
            return self.text.strip()
        if self.caption:
            return self.caption.strip()
        if self.callback_data:
            return self.callback_data.strip()
        return ""


@dataclass
class OutgoingMessage:
    """Normalized outgoing message to any channel."""

    user_id: str
    channel: Channel
    text: str
    media_url: Optional[str] = None
    media_type: Optional[MediaType] = None
    buttons: Optional[list[dict[str, str]]] = None  # [{"id": "btn_1", "label": "Option A"}]
    reply_to_message_id: Optional[int] = None
    parse_mode: Optional[str] = None   # "Markdown", "MarkdownV2", "HTML"
    disable_notification: bool = False
```

### 4.3 Adapter Interface (Abstract Base)

```python
# bots/core/adapter_base.py

from abc import ABC, abstractmethod
from bots.core.schemas import IncomingMessage, OutgoingMessage


class ChannelAdapter(ABC):
    """Abstract base class for all channel adapters."""

    channel: Channel

    @abstractmethod
    async def send_message(self, message: OutgoingMessage) -> bool:
        """Send a message to the user."""
        ...

    @abstractmethod
    async def download_media(self, message: IncomingMessage) -> Optional[bytes]:
        """Download media from an incoming message."""
        ...

    @abstractmethod
    async def get_user_profile(self, user_id: str) -> dict:
        """Get user profile information."""
        ...

    @abstractmethod
    async def register_webhook(self, webhook_url: str) -> bool:
        """Register webhook for receiving messages."""
        ...
```

---

## 5. Telegram Bot Implementation

### 5.1 Technology Stack

| Component | Technology | Version | Notes |
|-----------|-----------|---------|-------|
| Bot Framework | aiogram | 3.x | Async, modern Python bot framework |
| FSM | aiogram.fsm | 3.x | Memory + Redis storage |
| Webhook | aiohttp / uvicorn | — | Production webhook server |
| Keyboard Builder | aiogram.utils.keyboard | 3.x | Inline + Reply keyboards |
| Media Handling | aiogram.client | 3.x | File download/upload |
| i18n | gettext / custom | — | Per-language template files |

### 5.2 File Structure

```
bots/telegram/
├── __init__.py
├── bot.py                  # Main bot initialization, dispatcher, startup/shutdown
├── handlers/
│   ├── __init__.py
│   ├── start.py            # /start, /help commands
│   ├── text.py             # Text message handler with intent routing
│   ├── photo.py            # Photo (lab report) handler
│   ├── voice.py            # Voice message handler → ASR
│   ├── callback.py         # Inline keyboard callback queries
│   └── error.py            # Global error handler
├── keyboards/
│   ├── __init__.py
│   ├── main_menu.py        # Main menu keyboard
│   ├── onboarding.py       # Onboarding step keyboards
│   └── resistance.py       # Resistance calculator keyboards
├── formatters/
│   ├── __init__.py
│   ├── resistance.py       # Resistance probability bar formatter
│   ├── digest.py           # Daily digest message formatter
│   └── alert.py            # Peer alert message formatter
├── middlewares/
│   ├── __init__.py
│   ├── auth.py             # User identification middleware
│   ├── locale.py           # Language detection middleware
│   └── rate_limit.py       # Rate limiting middleware
├── filters/
│   ├── __init__.py
│   └── onboarding.py       # FSM state filters for onboarding
└── config.py               # Telegram bot configuration
```

### 5.3 Main Bot — `bot.py`

```python
# bots/telegram/bot.py

"""
UDARA AI Telegram Bot — Main Entry Point

This module initializes the Telegram bot using aiogram 3.x,
sets up the dispatcher with all handlers, middleware, and
starts the polling or webhook server.
"""

import asyncio
import logging
from typing import Optional

from aiogram import Bot, Dispatcher, Router
from aiogram.client.default import DefaultBotProperties
from aiogram.enums import ParseMode
from aiogram.fsm.storage.memory import MemoryStorage
from aiogram.fsm.storage.redis import RedisStorage
from aiogram.types import BotCommand, BotCommandScopeDefault
from redis.asyncio import Redis

from bots.telegram.config import TelegramConfig
from bots.telegram.handlers import (
    start,
    text,
    photo,
    voice,
    callback,
    error,
)
from bots.telegram.middlewares.auth import AuthMiddleware
from bots.telegram.middlewares.locale import LocaleMiddleware
from bots.telegram.middlewares.rate_limit import RateLimitMiddleware

logger = logging.getLogger(__name__)


class UDARATelegramBot:
    """Telegram bot for UDARA AI AMR surveillance."""

    def __init__(self, config: TelegramConfig):
        self.config = config
        self.bot: Optional[Bot] = None
        self.dp: Optional[Dispatcher] = None
        self.redis: Optional[Redis] = None

    async def initialize(self) -> None:
        """Initialize bot, dispatcher, and all components."""
        logger.info("Initializing UDARA Telegram bot...")

        # --- Storage setup ---
        if self.config.redis_url:
            self.redis = Redis.from_url(
                self.config.redis_url,
                decode_responses=True,
            )
            storage = RedisStorage(redis=self.redis)
            logger.info("Using Redis FSM storage")
        else:
            storage = MemoryStorage()
            logger.info("Using in-memory FSM storage (development mode)")

        # --- Bot instance ---
        self.bot = Bot(
            token=self.config.bot_token,
            default=DefaultBotProperties(
                parse_mode=ParseMode.HTML,
                allow_sending_without_reply=True,
            ),
        )

        # --- Dispatcher ---
        self.dp = Dispatcher(storage=storage)
        self.dp.include_router(start.router)
        self.dp.include_router(text.router)
        self.dp.include_router(photo.router)
        self.dp.include_router(voice.router)
        self.dp.include_router(callback.router)

        # --- Middleware chain (order matters: outer processes first) ---
        # Rate limit → Auth → Locale → Handler
        self.dp.update.outer_middleware(RateLimitMiddleware(
            max_requests=self.config.rate_limit_max,
            window_seconds=self.config.rate_limit_window,
        ))
        self.dp.update.middleware(AuthMiddleware())
        self.dp.update.middleware(LocaleMiddleware())

        # --- Error handler ---
        self.dp.errors.register(error.handle_global_error)

        # --- Bot commands ---
        await self._set_bot_commands()

        logger.info("UDARA Telegram bot initialized successfully")

    async def _set_bot_commands(self) -> None:
        """Register bot commands in the Telegram client UI."""
        commands = [
            BotCommand(command="start", description="🚀 Start / Register"),
            BotCommand(command="help", description="❓ Show help menu"),
            BotCommand(command="resist", description="🔬 Check drug resistance"),
            BotCommand(command="digest", description="📊 Get daily digest"),
            BotCommand(command="alert", description="🚨 Peer alert settings"),
            BotCommand(command="report", description="📋 Report a case"),
            BotCommand(command="language", description="🌐 Change language"),
            BotCommand(command="settings", description="⚙️ My settings"),
        ]
        await self.bot.set_my_commands(
            commands=commands,
            scope=BotCommandScopeDefault(),
        )
        # Also set bot description
        await self.bot.set_my_description(
            "UDARA AI — Antimicrobial Resistance Surveillance\n\n"
            "Report cases, check drug resistance, and receive\n"
            "daily AMR intelligence tailored to your district.\n\n"
            "🔒 Your data is encrypted and stays in Africa."
        )
        await self.bot.set_my_short_description(
            "AMR surveillance bot for Community Health Workers"
        )

    async def start_polling(self) -> None:
        """Start long-polling (development / single-instance)."""
        assert self.bot and self.dp
        logger.info("Starting long-polling...")
        await self.dp.start_polling(
            self.bot,
            allowed_updates=self.dp.resolve_used_update_types(),
            close_bot_session=False,
        )

    async def start_webhook(
        self,
        webhook_url: str,
        host: str = "0.0.0.0",
        port: int = 8080,
    ) -> None:
        """Start webhook server (production / multi-instance)."""
        assert self.bot and self.dp
        logger.info("Setting webhook: %s", webhook_url)
        await self.bot.set_webhook(
            url=webhook_url,
            drop_pending_updates=True,
            allowed_updates=self.dp.resolve_used_update_types(),
        )
        logger.info("Starting webhook server on %s:%d", host, port)
        from aiogram.webhook.aiohttp_server import (
            SimpleRequestHandler,
            setup_application,
        )
        from aiohttp import web

        app = web.Application()
        webhook_requests_handler = SimpleRequestHandler(
            dispatcher=self.dp,
            bot=self.bot,
        )
        webhook_requests_handler.register(app, path="/webhook/telegram")
        setup_application(app, self.dp, bot=self.bot)
        runner = web.AppRunner(app)
        await runner.setup()
        site = web.TCPSite(runner, host, port)
        await site.start()
        logger.info("Webhook server running on %s:%d", host, port)

    async def shutdown(self) -> None:
        """Graceful shutdown."""
        logger.info("Shutting down Telegram bot...")
        if self.bot:
            await self.bot.session.close()
        if self.redis:
            await self.redis.close()
        logger.info("Telegram bot shutdown complete")


async def main():
    """Entry point for running the bot."""
    import sys

    config = TelegramConfig(
        bot_token=sys.argv[1] if len(sys.argv) > 1 else "",
        redis_url="redis://localhost:6379/1",
        rate_limit_max=30,
        rate_limit_window=60,
    )
    bot = UDARATelegramBot(config)
    await bot.initialize()

    if config.webhook_url:
        await bot.start_webhook(config.webhook_url)
    else:
        await bot.start_polling()


if __name__ == "__main__":
    asyncio.run(main())
```

### 5.4 Text Handler — `handlers/text.py`

```python
# bots/telegram/handlers/text.py

"""
Text message handler — routes incoming text to the appropriate
conversation pipeline via intent classification.
"""

import logging
from typing import Optional

from aiogram import Router, F
from aiogram.types import Message
from aiogram.fsm.context import FSMContext

from bots.core.intents import IntentClassifier
from bots.core.intelligence_core import IntelligenceCore
from bots.telegram.formatters.resistance import format_resistance_result
from bots.telegram.keyboards.main_menu import get_main_menu_keyboard
from bots.telegram.config import TelegramConfig

logger = logging.getLogger(__name__)
router = Router()


class TextHandlerRouter:
    """Routes text messages to the correct conversation pipeline."""

    def __init__(self, intelligence_core: IntelligenceCore):
        self.core = intelligence_core
        self.intent_classifier = IntentClassifier()

    async def handle_text_message(
        self,
        message: Message,
        state: FSMContext,
    ) -> None:
        """Main entry point for all text messages."""
        user_text = message.text or ""
        user_id = str(message.from_user.id)

        logger.info(
            "Text from %s (%s): %.80s",
            message.from_user.full_name,
            user_id,
            user_text,
        )

        # 1. Check if user is in onboarding FSM
        current_state = await state.get_state()
        if current_state and "onboarding" in current_state:
            # Delegate to onboarding handler
            from bots.telegram.handlers.start import handle_onboarding_step
            await handle_onboarding_step(message, state)
            return

        # 2. Check if user is in resistance calculator FSM
        if current_state and "resistance" in current_state:
            from bots.resistance.resistance_calc import (
                handle_resistance_step,
            )
            await handle_resistance_step(message, state)
            return

        # 3. Command-like shortcuts
        if user_text.lower().strip() in ("menu", "/menu", "main"):
            keyboard = get_main_menu_keyboard()
            await message.answer(
                "📋 *Main Menu*\n\n"
                "What would you like to do?",
                reply_markup=keyboard,
            )
            return

        # 4. Intent classification
        intent, confidence = await self.intent_classifier.classify(user_text)

        logger.info(
            "Intent: %s (confidence: %.2f) for user %s",
            intent,
            confidence,
            user_id,
        )

        # 5. Route to appropriate pipeline
        if intent == "report_case" and confidence > 0.7:
            await self._handle_case_report(message, state, user_text)
        elif intent == "check_resistance" and confidence > 0.7:
            await self._handle_resistance_query(message, state, user_text)
        elif intent == "get_digest" and confidence > 0.6:
            await self._handle_digest_request(message, user_id)
        elif intent == "help" or confidence < 0.4:
            await self._handle_unknown_or_help(message)
        else:
            await self._handle_generic(message, user_text)

    async def _handle_case_report(
        self,
        message: Message,
        state: FSMContext,
        text: str,
    ) -> None:
        """Handle a case report from free text."""
        await message.answer(
            "📝 *Processing your report...*\n\n"
            "I'm extracting symptoms, drugs, and patient info.\n"
            "This usually takes 3–5 seconds.",
            parse_mode="Markdown",
        )

        # Send to Agent A for extraction
        result = await self.core.process_case_report(
            user_id=str(message.from_user.id),
            channel="telegram",
            text=text,
        )

        if result.success:
            await message.answer(
                "✅ *Case Reported Successfully!*\n\n"
                f"**Case ID:** `{result.case_id}`\n"
                f"**Symptoms:** {', '.join(result.symptoms)}\n"
                f"**Drugs mentioned:** {', '.join(result.drugs)}\n"
                f"**Resistance risk:** {result.risk_level}\n\n"
                f"*{result.recommendation}*",
                parse_mode="Markdown",
            )
        else:
            await message.answer(
                "⚠️ I couldn't fully process your report. "
                "Please try again with more detail, or use "
                "/report for a guided format.",
            )

    async def _handle_resistance_query(
        self,
        message: Message,
        state: FSMContext,
        text: str,
    ) -> None:
        """Handle drug resistance query."""
        # Extract drug name from text
        from bots.core.intents import extract_drug_names
        drugs = await extract_drug_names(text)

        if drugs:
            await state.set_state("resistance:awaiting_confirmation")
            await state.update_data(drugs=drugs, context=text)
            await message.answer(
                f"🔬 I found **{len(drugs)} drug(s)** in your message:\n\n"
                + "\n".join(f"• `{d}`" for d in drugs)
                + "\n\n_Should I check resistance for these drugs?_",
                parse_mode="Markdown",
            )
        else:
            await message.answer(
                "🔬 *Drug Resistance Check*\n\n"
                "Please tell me which drug(s) you'd like to check.\n\n"
                "Examples:\n"
                "• `Amoxicillin resistance in Lagos`\n"
                "• `Ciprofloxacin efficacy in Nairobi`\n"
                "• `Ceftriaxone for typhoid in Ghana`",
                parse_mode="Markdown",
            )

    async def _handle_digest_request(
        self,
        message: Message,
        user_id: str,
    ) -> None:
        """Send the daily digest to the user."""
        digest = await self.core.get_personalized_digest(user_id)

        if not digest:
            await message.answer(
                "📊 No digest available yet. "
                "Digests are sent daily at 7:00 AM local time.\n"
                "You can also use /resist for on-demand checks.",
            )
            return

        await message.answer(
            digest.formatted_text,
            parse_mode="HTML",
        )

    async def _handle_unknown_or_help(self, message: Message) -> None:
        """Handle unrecognized input or help request."""
        keyboard = get_main_menu_keyboard()
        await message.answer(
            "🤔 I'm not sure what you mean. Here's what I can do:\n\n"
            "📋 **/report** — Report a new AMR case\n"
            "🔬 **/resist** — Check drug resistance\n"
            "📊 **/digest** — Get daily AMR digest\n"
            "🚨 **/alert** — Peer alert settings\n"
            "🌐 **/language** — Change language\n"
            "❓ **/help** — Full help guide",
            parse_mode="Markdown",
            reply_markup=keyboard,
        )

    async def _handle_generic(self, message: Message, text: str) -> None:
        """Handle low-confidence or generic messages."""
        # Try to provide a helpful response anyway
        response = await self.core.generate_helpful_response(text)
        await message.answer(response, parse_mode="Markdown")


# Instantiate the router (lazy-loaded handler)
_text_handler: Optional[TextHandlerRouter] = None


def get_text_handler(core: IntelligenceCore) -> TextHandlerRouter:
    global _text_handler
    if _text_handler is None:
        _text_handler = TextHandlerRouter(core)
    return _text_handler


# --- aiogram route registration ---

@router.message(F.text)
async def handle_text(message: Message, state: FSMContext):
    """Catch-all text message handler."""
    from bots.core.intelligence_core import get_intelligence_core
    handler = get_text_handler(get_intelligence_core())
    await handler.handle_text_message(message, state)
```

### 5.5 Main Menu Keyboard — `keyboards/main_menu.py`

```python
# bots/telegram/keyboards/main_menu.py

"""
Main menu keyboard for the UDARA Telegram bot.
Provides persistent reply keyboard and inline keyboard variants.
"""

from aiogram.types import (
    InlineKeyboardMarkup,
    InlineKeyboardButton,
    ReplyKeyboardMarkup,
    KeyboardButton,
    ReplyKeyboardRemove,
)
from aiogram.utils.keyboard import InlineKeyboardBuilder, ReplyKeyboardBuilder


def get_main_menu_keyboard() -> ReplyKeyboardMarkup:
    """Persistent reply keyboard — always visible below chat input."""

    builder = ReplyKeyboardBuilder()
    builder.row(
        KeyboardButton(text="📋 Report Case"),
        KeyboardButton(text="🔬 Check Resistance"),
    )
    builder.row(
        KeyboardButton(text="📊 Daily Digest"),
        KeyboardButton(text="🚨 Peer Alerts"),
    )
    builder.row(
        KeyboardButton(text="🌐 Language"),
        KeyboardButton(text="⚙️ Settings"),
    )
    builder.row(
        KeyboardButton(text="❓ Help"),
    )
    # Adjust to 2 columns
    builder.adjust(2, 2, 2, 1)

    return builder.as_markup(resize_keyboard=True, input_field_placeholder="Type a message or tap a button...")


def get_report_type_inline_keyboard() -> InlineKeyboardMarkup:
    """Inline keyboard for selecting report type."""

    builder = InlineKeyboardBuilder()
    builder.button(text="🦠 Clinical Case", callback_data="report:clinical")
    builder.button(text="🧪 Lab Report", callback_data="report:lab")
    builder.button(text="💊 Drug Stock", callback_data="report:drugstock")
    builder.button(text="❌ Cancel", callback_data="report:cancel")
    builder.adjust(2, 1, 1)

    return builder.as_markup()


def get_resistance_drug_inline_keyboard(
    drugs: list[str],
) -> InlineKeyboardMarkup:
    """Dynamic inline keyboard for drug selection."""

    builder = InlineKeyboardBuilder()
    for drug in drugs[:10]:  # Max 10 drugs per page
        builder.button(text=f"💊 {drug}", callback_data=f"resist:drug:{drug}")

    builder.button(text="🔍 Custom drug", callback_data="resist:custom")
    builder.button(text="❌ Cancel", callback_data="resist:cancel")
    builder.adjust(2, 2, 2, 2, 1, 1)

    return builder.as_markup()


def get_language_inline_keyboard() -> InlineKeyboardMarkup:
    """Language selection inline keyboard."""

    builder = InlineKeyboardBuilder()
    languages = [
        ("🇬🇧 English", "lang:en"),
        ("🇳🇬 Hausa", "lang:ha"),
        ("🇳🇬 Yoruba", "lang:yo"),
        ("🇳🇬 Igbo", "lang:ig"),
        ("🇰🇪 Swahili", "lang:sw"),
        ("🇪🇹 Amharic", "lang:am"),
    ]
    for label, callback in languages:
        builder.button(text=label, callback_data=callback)
    builder.adjust(2)

    return builder.as_markup()


def get_onboarding_step_keyboard(step: int) -> InlineKeyboardMarkup:
    """Contextual keyboard for onboarding steps."""

    keyboards = {
        1: InlineKeyboardMarkup(inline_keyboard=[
            [InlineKeyboardButton(text="🇳🇬 Nigeria", callback_data="onboard:country:NG"),
             InlineKeyboardButton(text="🇰🇪 Kenya", callback_data="onboard:country:KE")],
            [InlineKeyboardButton(text="🇬🇭 Ghana", callback_data="onboard:country:GH"),
             InlineKeyboardButton(text="🇹🇿 Tanzania", callback_data="onboard:country:TZ")],
            [InlineKeyboardButton(text="🇺🇬 Uganda", callback_data="onboard:country:UG"),
             InlineKeyboardButton(text="🇪🇹 Ethiopia", callback_data="onboard:country:ET")],
        ]),
        2: None,  # Free text input (name)
        3: None,  # Free text input (district)
        4: InlineKeyboardMarkup(inline_keyboard=[
            [InlineKeyboardButton(text="👩‍⚕️ CHW", callback_data="onboard:role:chw"),
             InlineKeyboardButton(text="👨‍⚕️ Nurse", callback_data="onboard:role:nurse")],
            [InlineKeyboardButton(text="🧑‍⚕️ Clinician", callback_data="onboard:role:clinician"),
             InlineKeyboardButton(text="🔬 Lab Tech", callback_data="onboard:role:labtech")],
            [InlineKeyboardButton(text="👨‍⚕️ Pharmacist", callback_data="onboard:role:pharmacist")],
        ]),
        5: InlineKeyboardMarkup(inline_keyboard=[
            [InlineKeyboardButton(text="🇬🇧 English", callback_data="onboard:lang:en"),
             InlineKeyboardButton(text="🇳🇬 Hausa", callback_data="onboard:lang:ha")],
            [InlineKeyboardButton(text="🇳🇬 Yoruba", callback_data="onboard:lang:yo"),
             InlineKeyboardButton(text="🇰🇪 Swahili", callback_data="onboard:lang:sw")],
            [InlineKeyboardButton(text="🇪🇹 Amharic", callback_data="onboard:lang:am")],
        ]),
    }
    return keyboards.get(step)


def get_alert_settings_keyboard(
    is_subscribed: bool,
    district: str,
) -> InlineKeyboardMarkup:
    """Peer alert subscription toggle keyboard."""

    builder = InlineKeyboardBuilder()
    if is_subscribed:
        builder.button(
            text="🔕 Mute alerts",
            callback_data="alert:unsubscribe",
        )
    else:
        builder.button(
            text="🔔 Subscribe to alerts",
            callback_data="alert:subscribe",
        )
    builder.button(text="📍 Change district", callback_data="alert:district")
    builder.button(text="🔙 Back to menu", callback_data="menu:main")
    builder.adjust(1, 1, 1)

    return builder.as_markup()
```

---

## 6. WhatsApp Bot Implementation

### 6.1 Technology Stack

| Component | Technology | Notes |
|-----------|-----------|-------|
| API | Meta Cloud API (v18.0+) | Official WhatsApp Business API |
| Authentication | Bearer token (permanent) + app secret | |
| Webhook | Meta Graph API webhook | POST to `/webhook/whatsapp` |
| Media | Cloud API media endpoints | Download/upload via `media/v1/` |
| Templates | Pre-approved message templates | For proactive messages |
| Rate Limit | 80 messages/second (Business tier) | |

### 6.2 WhatsAppBot Class

```python
# bots/whatsapp/whatsapp_bot.py

"""
UDARA AI WhatsApp Bot — Meta Cloud API Integration

Handles all WhatsApp Business API interactions including
message sending, media download, interactive buttons,
and webhook verification.
"""

import hashlib
import hmac
import json
import logging
from datetime import datetime
from typing import Any, Optional

import httpx

logger = logging.getLogger(__name__)


class WhatsAppAPIError(Exception):
    """Raised when WhatsApp Cloud API returns an error."""

    def __init__(self, status_code: int, message: str, error_data: dict = None):
        self.status_code = status_code
        self.message = message
        self.error_data = error_data or {}
        super().__init__(f"WhatsApp API {status_code}: {message}")


class WhatsAppBot:
    """
    WhatsApp Business Cloud API client for UDARA AI.

    Supports:
    - Text messages
    - Interactive buttons (max 3 per message)
    - Interactive list messages
    - Media messages (image, document)
    - Media download
    - Template messages
    - Location requests
    - Contact requests
    - Reaction messages
    """

    # Meta Cloud API base URL
    BASE_URL = "https://graph.facebook.com/v18.0"

    def __init__(
        self,
        phone_number_id: str,
        access_token: str,
        verify_token: str,
        app_secret: str,
        api_version: str = "v18.0",
        timeout: float = 30.0,
    ):
        """
        Initialize WhatsApp bot.

        Args:
            phone_number_id: WhatsApp Business phone number ID from Meta dashboard.
            access_token: Permanent access token from Meta App settings.
            verify_token: Custom verification token for webhook setup.
            app_secret: App secret for webhook signature verification.
            api_version: Meta Graph API version.
            timeout: HTTP request timeout in seconds.
        """
        self.phone_number_id = phone_number_id
        self.access_token = access_token
        self.verify_token = verify_token
        self.app_secret = app_secret
        self.api_version = api_version
        self.base_url = f"https://graph.facebook.com/{api_version}"
        self.timeout = timeout

        # HTTP client with connection pooling
        self._client = httpx.AsyncClient(
            base_url=self.base_url,
            headers={
                "Authorization": f"Bearer {access_token}",
                "Content-Type": "application/json",
            },
            timeout=timeout,
        )

    # ────────────────────────────────────────────────────────────
    # WEBHOOK VERIFICATION
    # ────────────────────────────────────────────────────────────

    def verify_webhook(
        self,
        mode: str,
        token: str,
        challenge: str,
    ) -> Optional[str]:
        """
        Verify webhook subscription with Meta.

        Called when Meta sends the initial GET request to your
        webhook URL during setup.

        Returns:
            The challenge string if verification succeeds, None otherwise.
        """
        if mode == "subscribe" and token == self.verify_token:
            logger.info("Webhook verified successfully")
            return challenge
        logger.warning(
            "Webhook verification failed: mode=%s, token=%s",
            mode,
            token[:4] + "..." if token else None,
        )
        return None

    def verify_signature(self, payload: bytes, signature: str) -> bool:
        """
        Verify webhook payload signature (HMAC-SHA256).

        Meta includes an X-Hub-Signature-256 header with every webhook POST.
        This MUST be verified to prevent spoofed payloads.

        Args:
            payload: Raw request body bytes.
            signature: Value of X-Hub-Signature-256 header.

        Returns:
            True if signature is valid.
        """
        expected = "sha256=" + hmac.new(
            self.app_secret.encode("utf-8"),
            payload,
            hashlib.sha256,
        ).hexdigest()
        return hmac.compare_digest(expected, signature)

    # ────────────────────────────────────────────────────────────
    # MESSAGE SENDING
    # ────────────────────────────────────────────────────────────

    async def send_text(
        self,
        to: str,
        text: str,
        preview_url: bool = False,
        message_id: Optional[str] = None,
    ) -> dict:
        """
        Send a text message.

        Args:
            to: Recipient phone number in international format (e.g., "2348012345678").
            text: Message text. Supports bold (*...*), italic (_..._), strikethrough (~...~), code (```...```).
            preview_url: Whether to generate a link preview for URLs in text.
            message_id: Optional message ID to reply to (context).

        Returns:
            API response dict with message_id.
        """
        payload: dict[str, Any] = {
            "messaging_product": "whatsapp",
            "to": to,
            "type": "text",
            "text": {
                "body": text,
                "preview_url": preview_url,
            },
        }
        if message_id:
            payload["context"] = {"message_id": message_id}

        response = await self._send(payload)
        logger.info("Sent text to %s: %.60s...", to, text)
        return response

    async def send_interactive_buttons(
        self,
        to: str,
        body: str,
        buttons: list[dict[str, str]],
        header: Optional[str] = None,
        footer: Optional[str] = None,
        message_id: Optional[str] = None,
    ) -> dict:
        """
        Send an interactive message with reply buttons.

        CRITICAL: WhatsApp allows MAXIMUM 3 buttons per message.
        If you pass more than 3, only the first 3 will be sent.

        Args:
            to: Recipient phone number.
            body: Body text (max 1024 characters).
            buttons: List of button dicts: [{"id": "unique_id", "title": "Button Label"}]
                     Max 3 buttons. Title max 20 chars each.
            header: Optional header text (text type only, max 60 chars).
            footer: Optional footer text (max 60 chars).
            message_id: Optional message ID to reply to.

        Returns:
            API response dict.

        Raises:
            WhatsAppAPIError: If API returns an error.
        """
        if len(buttons) > 3:
            logger.warning(
                "Truncating buttons from %d to 3 (WhatsApp limit)",
                len(buttons),
            )
            buttons = buttons[:3]

        # Build interactive object
        interactive: dict[str, Any] = {
            "type": "button",
            "body": {"text": body[:1024]},
        }

        if header:
            interactive["header"] = {
                "type": "text",
                "text": header[:60],
            }
        if footer:
            interactive["footer"] = {"text": footer[:60]}

        # Build button objects
        interactive["action"] = {
            "buttons": [
                {
                    "type": "reply",
                    "reply": {
                        "id": btn["id"],
                        "title": btn["title"][:20],
                    },
                }
                for btn in buttons
            ]
        }

        payload: dict[str, Any] = {
            "messaging_product": "whatsapp",
            "to": to,
            "type": "interactive",
            "interactive": interactive,
        }
        if message_id:
            payload["context"] = {"message_id": message_id}

        response = await self._send(payload)
        logger.info(
            "Sent interactive buttons to %s: %d buttons",
            to,
            len(buttons),
        )
        return response

    async def send_interactive_list(
        self,
        to: str,
        body: str,
        button_text: str,
        sections: list[dict[str, Any]],
        header: Optional[str] = None,
        footer: Optional[str] = None,
    ) -> dict:
        """
        Send an interactive list message.

        Lists support up to 10 items per section and up to 10 sections.
        Each item has a title (max 24 chars) and optional description.

        Args:
            to: Recipient phone number.
            body: Body text (max 1024 chars).
            button_text: The button label that opens the list (max 20 chars).
            sections: List of section dicts with "title" and "rows".
            header: Optional header text.
            footer: Optional footer text.
        """
        interactive: dict[str, Any] = {
            "type": "list",
            "body": {"text": body[:1024]},
            "action": {
                "button": button_text[:20],
                "sections": sections[:10],
            },
        }
        if header:
            interactive["header"] = {"type": "text", "text": header[:60]}
        if footer:
            interactive["footer"] = {"text": footer[:60]}

        payload = {
            "messaging_product": "whatsapp",
            "to": to,
            "type": "interactive",
            "interactive": interactive,
        }
        return await self._send(payload)

    async def send_image(
        self,
        to: str,
        image_url: str,
        caption: Optional[str] = None,
        message_id: Optional[str] = None,
    ) -> dict:
        """Send an image message."""
        payload: dict[str, Any] = {
            "messaging_product": "whatsapp",
            "to": to,
            "type": "image",
            "image": {"link": image_url},
        }
        if caption:
            payload["image"]["caption"] = caption[:1024]
        if message_id:
            payload["context"] = {"message_id": message_id}
        return await self._send(payload)

    async def send_document(
        self,
        to: str,
        document_url: str,
        filename: str,
        caption: Optional[str] = None,
    ) -> dict:
        """Send a document message (PDF, etc)."""
        payload: dict[str, Any] = {
            "messaging_product": "whatsapp",
            "to": to,
            "type": "document",
            "document": {
                "link": document_url,
                "filename": filename,
            },
        }
        if caption:
            payload["document"]["caption"] = caption[:1024]
        return await self._send(payload)

    async def send_location_request(self, to: str, body: str) -> dict:
        """Request the user's location."""
        payload = {
            "messaging_product": "whatsapp",
            "to": to,
            "type": "interactive",
            "interactive": {
                "type": "location_request_message",
                "body": {"text": body},
                "action": {
                    "name": "send_location",
                },
            },
        }
        return await self._send(payload)

    async def send_reaction(
        self,
        to: str,
        message_id: str,
        emoji: str,
    ) -> dict:
        """React to a message with an emoji."""
        payload = {
            "messaging_product": "whatsapp",
            "to": to,
            "type": "reaction",
            "reaction": {
                "message_id": message_id,
                "emoji": emoji,
            },
        }
        return await self._send(payload)

    async def send_template(
        self,
        to: str,
        template_name: str,
        language_code: str = "en",
        components: Optional[list[dict]] = None,
    ) -> dict:
        """
        Send a pre-approved template message.

        Template messages are the ONLY way to initiate a conversation
        with a user who hasn't messaged you in the last 24 hours.

        Templates must be pre-approved by Meta. Typical approval time: 24-48h.
        """
        payload = {
            "messaging_product": "whatsapp",
            "to": to,
            "type": "template",
            "template": {
                "name": template_name,
                "language": {"code": language_code},
            },
        }
        if components:
            payload["template"]["components"] = components
        return await self._send(payload)

    # ────────────────────────────────────────────────────────────
    # MEDIA DOWNLOAD
    # ────────────────────────────────────────────────────────────

    async def download_media(self, media_id: str) -> tuple[bytes, str]:
        """
        Download media from WhatsApp Cloud API.

        Two-step process:
        1. GET /{media_id} → get the media URL
        2. GET the media URL → download the actual file

        Args:
            media_id: Media ID from the incoming webhook payload.

        Returns:
            Tuple of (file_bytes, mime_type).

        Raises:
            WhatsAppAPIError: If download fails.
        """
        # Step 1: Get media URL
        resp = await self._client.get(f"/{media_id}")
        if resp.status_code != 200:
            raise WhatsAppAPIError(
                resp.status_code,
                f"Failed to get media URL for {media_id}",
                resp.json() if resp.content else {},
            )
        media_data = resp.json()
        media_url = media_data["url"]
        mime_type = media_data.get("mime_type", "application/octet-stream")
        file_size = media_data.get("file_size", 0)

        logger.info(
            "Media metadata: id=%s, mime=%s, size=%d",
            media_id,
            mime_type,
            file_size,
        )

        # Step 2: Download actual media
        # Must use a fresh client WITHOUT auth header for the CDN URL
        async with httpx.AsyncClient(timeout=60.0) as cdn_client:
            media_resp = await cdn_client.get(media_url)
            if media_resp.status_code != 200:
                raise WhatsAppAPIError(
                    media_resp.status_code,
                    f"Failed to download media from {media_url}",
                )

        logger.info(
            "Downloaded media: %d bytes, type=%s",
            len(media_resp.content),
            mime_type,
        )
        return media_resp.content, mime_type

    # ────────────────────────────────────────────────────────────
    # WEBHOOK PAYLOAD PARSING
    # ────────────────────────────────────────────────────────────

    def parse_webhook_payload(self, payload: dict) -> list[dict]:
        """
        Parse a WhatsApp webhook payload into a list of normalized messages.

        Meta sends payloads in this structure:
        {
            "entry": [{
                "changes": [{
                    "value": {
                        "messages": [...],
                        "statuses": [...],
                        "contacts": [...]
                    }
                }]
            }]
        }

        Returns:
            List of normalized message dicts.
        """
        messages = []

        for entry in payload.get("entry", []):
            for change in entry.get("changes", []):
                value = change.get("value", [])

                # Extract contacts info (name, phone)
                contacts = value.get("contacts", [])
                contact_map = {}
                for contact in contacts:
                    wa_id = contact.get("wa_id", "")
                    name = contact.get("profile", {}).get("name", "Unknown")
                    contact_map[wa_id] = name

                # Parse messages
                for msg in value.get("messages", []):
                    normalized = self._normalize_message(msg, contact_map)
                    if normalized:
                        messages.append(normalized)

        return messages

    def _normalize_message(
        self,
        msg: dict,
        contact_map: dict[str, str],
    ) -> Optional[dict]:
        """Normalize a single WhatsApp message into platform-agnostic format."""
        msg_type = msg.get("type", "")
        from_number = msg.get("from", "")
        msg_id = msg.get("id", "")
        timestamp = msg.get("timestamp", "")

        if not from_number:
            return None

        base = {
            "message_id": msg_id,
            "user_id": f"wa_{from_number}",
            "phone_number": from_number,
            "channel": "whatsapp",
            "timestamp": datetime.utcfromtimestamp(int(timestamp)),
            "sender_name": contact_map.get(from_number, "Unknown"),
        }

        if msg_type == "text":
            base["text"] = msg["text"]["body"]
            base["media_type"] = "text"
        elif msg_type == "image":
            base["media_type"] = "photo"
            base["media_id"] = msg["image"]["id"]
            base["media_url"] = msg["image"].get("caption", "")
        elif msg_type == "document":
            base["media_type"] = "document"
            base["media_id"] = msg["document"]["id"]
            base["media_filename"] = msg["document"].get("filename", "")
            base["caption"] = msg["document"].get("caption", "")
        elif msg_type == "audio":
            base["media_type"] = "voice"
            base["media_id"] = msg["audio"]["id"]
        elif msg_type == "video":
            base["media_type"] = "video"
            base["media_id"] = msg["video"]["id"]
        elif msg_type == "location":
            base["media_type"] = "location"
            base["latitude"] = msg["location"]["latitude"]
            base["longitude"] = msg["location"]["longitude"]
            base["location_name"] = msg["location"].get("name", "")
        elif msg_type == "interactive":
            # Button reply or list reply
            interactive = msg.get("interactive", {})
            if "button_reply" in interactive:
                base["text"] = interactive["button_reply"]["title"]
                base["callback_data"] = interactive["button_reply"]["id"]
            elif "list_reply" in interactive:
                base["text"] = interactive["list_reply"]["title"]
                base["callback_data"] = interactive["list_reply"]["id"]

        return base

    # ────────────────────────────────────────────────────────────
    # INTERNAL HELPERS
    # ────────────────────────────────────────────────────────────

    async def _send(self, payload: dict) -> dict:
        """Send a payload to the WhatsApp API."""
        url = f"/{self.phone_number_id}/messages"
        resp = await self._client.post(url, json=payload)

        if resp.status_code not in (200, 201):
            error_data = resp.json() if resp.content else {}
            logger.error(
                "WhatsApp API error %d: %s",
                resp.status_code,
                error_data,
            )
            raise WhatsAppAPIError(resp.status_code, resp.text, error_data)

        return resp.json()

    async def close(self) -> None:
        """Close the HTTP client."""
        await self._client.aclose()
```

---

## 7. Onboarding Flow — 5-Step FSM

### 7.1 FSM State Diagram

```
┌─────────────────────────────────────────────────────────┐
│                 ONBOARDING FSM STATE MACHINE            │
│                                                         │
│  ┌──────────┐                                           │
│  │  START   │◄─────────────────────────────────────┐    │
│  │  /start  │                                      │    │
│  └────┬─────┘                                      │    │
│       │                                            │    │
│       ▼                                            │    │
│  ┌──────────┐  Invalid    ┌──────────┐            │    │
│  │ STEP 1   │───────────►│  ERROR   │            │    │
│  │ Country  │             │  State   │            │    │
│  └────┬─────┘             └────┬─────┘            │    │
│       │ Select                  │ /restart         │    │
│       ▼                         └──────────────────┘    │
│  ┌──────────┐                                           │
│  │ STEP 2   │                                           │
│  │ Name     │                                           │
│  └────┬─────┘                                           │
│       │ Free text                                        │
│       ▼                                                  │
│  ┌──────────┐                                           │
│  │ STEP 3   │                                           │
│  │ District │                                           │
│  └────┬─────┘                                           │
│       │ Free text (validated against known districts)    │
│       ▼                                                  │
│  ┌──────────┐                                           │
│  │ STEP 4   │                                           │
│  │ Role     │                                           │
│  └────┬─────┘                                           │
│       │ Button select                                    │
│       ▼                                                  │
│  ┌──────────┐                                           │
│  │ STEP 5   │                                           │
│  │ Language │                                           │
│  └────┬─────┘                                           │
│       │ Button select                                    │
│       ▼                                                  │
│  ┌──────────┐                                           │
│  │ COMPLETE │                                           │
│  │ Save DB  │──────────► Show main menu                 │
│  │ Send msg │                                           │
│  └──────────┘                                           │
│                                                         │
└─────────────────────────────────────────────────────────┘
```

### 7.2 Full Onboarding Code

```python
# bots/core/onboarding.py

"""
5-step onboarding FSM for UDARA AI.

Works identically on WhatsApp and Telegram via the
platform-agnostic Bot Intelligence Core.

Steps:
  1. Select country (button menu)
  2. Enter name (free text)
  3. Enter district (free text, validated)
  4. Select role (button menu)
  5. Select language (button menu)
"""

import json
import logging
from dataclasses import dataclass, field, asdict
from datetime import datetime
from enum import Enum
from typing import Any, Optional

from bots.core.schemas import IncomingMessage, OutgoingMessage, Channel

logger = logging.getLogger(__name__)


class OnboardingStep(int, Enum):
    COUNTRY = 1
    NAME = 2
    DISTRICT = 3
    ROLE = 4
    LANGUAGE = 5
    COMPLETE = 99


# Known districts per country (simplified)
COUNTRY_DISTRICTS: dict[str, list[str]] = {
    "NG": ["Lagos Mainland", "Lagos Island", "Abuja Municipal", "Kano Municipal",
           "Ibadan North", "Port Harcourt", "Benin City", "Kaduna North",
           "Maiduguri", "Enugu Urban"],
    "KE": ["Nairobi Central", "Mombasa", "Kisumu", "Nakuru", "Eldoret",
           "Thika", "Malindi", "Kitale", "Nyeri", "Meru"],
    "TZ": ["Dar es Salaam", "Dodoma", "Arusha", "Mwanza", "Mbeya",
           "Morogoro", "Tanga", "Zanzibar", "Kilimanjaro", "Iringa"],
    "GH": ["Accra", "Kumasi", "Tamale", "Takoradi", "Sunyani",
           "Cape Coast", "Ho", "Koforidua", "Bolgatanga", "Wa"],
    "UG": ["Kampala", "Entebbe", "Jinja", "Gulu", "Mbarara",
           "Fort Portal", "Mbale", "Arua", "Lira", "Soroti"],
    "ET": ["Addis Ababa", "Dire Dawa", "Hawassa", "Bahir Dar", "Adama",
           "Mekelle", "Jimma", "Dessie", "Harar", "Shashamane"],
}

VALID_ROLES = [
    ("chw", "Community Health Worker", "👩‍⚕️"),
    ("nurse", "Nurse", "👨‍⚕️"),
    ("clinician", "Clinician / Doctor", "🧑‍⚕️"),
    ("labtech", "Laboratory Technician", "🔬"),
    ("pharmacist", "Pharmacist", "💊"),
    ("admin", "Health Facility Admin", "📋"),
]

VALID_LANGUAGES = [
    ("en", "English", "🇬🇧"),
    ("ha", "Hausa", "🇳🇬"),
    ("yo", "Yoruba", "🇳🇬"),
    ("ig", "Igbo", "🇳🇬"),
    ("sw", "Swahili", "🇰🇪"),
    ("am", "Amharic", "🇪🇹"),
]


@dataclass
class OnboardingProfile:
    """Data collected during onboarding."""
    user_id: str
    channel: Channel
    country: Optional[str] = None
    country_name: Optional[str] = None
    name: Optional[str] = None
    district: Optional[str] = None
    role: Optional[str] = None
    role_name: Optional[str] = None
    language: Optional[str] = None
    language_name: Optional[str] = None
    phone_number: Optional[str] = None
    started_at: datetime = field(default_factory=datetime.utcnow)
    completed_at: Optional[datetime] = None

    @property
    def is_complete(self) -> bool:
        return all([self.country, self.name, self.district,
                     self.role, self.language])

    def to_dict(self) -> dict:
        return {
            **asdict(self),
            "started_at": self.started_at.isoformat(),
            "completed_at": self.completed_at.isoformat() if self.completed_at else None,
        }


class OnboardingEngine:
    """
    Platform-agnostic onboarding engine.

    Usage:
        engine = OnboardingEngine(state_store, user_store)
        response = await engine.process_message(incoming_message, fsm_state)
    """

    def __init__(
        self,
        state_store: Any,   # Redis/async dict for FSM state
        user_store: Any,     # Database for saving completed profiles
    ):
        self.state_store = state_store
        self.user_store = user_store

    async def start_onboarding(
        self,
        user_id: str,
        channel: Channel,
        phone_number: Optional[str] = None,
    ) -> OutgoingMessage:
        """Initiate the onboarding flow."""
        # Create profile
        profile = OnboardingProfile(
            user_id=user_id,
            channel=channel,
            phone_number=phone_number,
        )
        await self._save_state(user_id, profile)
        await self._set_step(user_id, OnboardingStep.COUNTRY)

        return self._build_country_prompt(channel)

    async def process_message(
        self,
        message: IncomingMessage,
        fsm_state: dict,
    ) -> list[OutgoingMessage]:
        """
        Process an incoming message during onboarding.

        Returns a list of outgoing messages (may be multiple for
        validation errors + re-prompt).
        """
        user_id = message.user_id
        step = fsm_state.get("step", OnboardingStep.COUNTRY)
        profile = await self._load_profile(user_id)

        responses: list[OutgoingMessage] = []

        if step == OnboardingStep.COUNTRY:
            result = await self._handle_country(message, profile)
            responses.extend(result)
        elif step == OnboardingStep.NAME:
            result = await self._handle_name(message, profile)
            responses.extend(result)
        elif step == OnboardingStep.DISTRICT:
            result = await self._handle_district(message, profile)
            responses.extend(result)
        elif step == OnboardingStep.ROLE:
            result = await self._handle_role(message, profile)
            responses.extend(result)
        elif step == OnboardingStep.LANGUAGE:
            result = await self._handle_language(message, profile)
            responses.extend(result)

        return responses

    # ─── Step handlers ───────────────────────────────────────

    async def _handle_country(
        self,
        message: IncomingMessage,
        profile: OnboardingProfile,
    ) -> list[OutgoingMessage]:
        """Step 1: Country selection."""
        callback = message.callback_data or message.text or ""

        # Map callback to country code
        country_map = {
            "onboard:country:NG": ("NG", "🇳🇬 Nigeria"),
            "onboard:country:KE": ("KE", "🇰🇪 Kenya"),
            "onboard:country:TZ": ("TZ", "🇹🇿 Tanzania"),
            "onboard:country:GH": ("GH", "🇬🇭 Ghana"),
            "onboard:country:UG": ("UG", "🇺🇬 Uganda"),
            "onboard:country:ET": ("ET", "🇪🇹 Ethiopia"),
        }

        if callback not in country_map:
            # Re-prompt
            return [
                OutgoingMessage(
                    user_id=message.user_id,
                    channel=message.channel,
                    text="Please select your country:",
                    buttons=[
                        {"id": "onboard:country:NG", "label": "🇳🇬 Nigeria"},
                        {"id": "onboard:country:KE", "label": "🇰🇪 Kenya"},
                        {"id": "onboard:country:GH", "label": "🇬🇭 Ghana"},
                        # WhatsApp: max 3 buttons, so we'll paginate
                    ],
                )
            ]

        code, name = country_map[callback]
        profile.country = code
        profile.country_name = name
        await self._save_state(message.user_id, profile)
        await self._set_step(message.user_id, OnboardingStep.NAME)

        return [
            OutgoingMessage(
                user_id=message.user_id,
                channel=message.channel,
                text=f"✅ {name}\n\n📝 What is your full name?\n\n"
                     "This helps us personalize your experience.",
            )
        ]

    async def _handle_name(
        self,
        message: IncomingMessage,
        profile: OnboardingProfile,
    ) -> list[OutgoingMessage]:
        """Step 2: Name entry."""
        name = (message.text or "").strip()

        if len(name) < 2:
            return [
                OutgoingMessage(
                    user_id=message.user_id,
                    channel=message.channel,
                    text="❌ Name too short. Please enter your full name (at least 2 characters).",
                )
            ]

        profile.name = name
        await self._save_state(message.user_id, profile)
        await self._set_step(message.user_id, OnboardingStep.DISTRICT)

        districts = COUNTRY_DISTRICTS.get(profile.country, [])
        districts_text = "\n".join(f"  • {d}" for d in districts[:8])

        return [
            OutgoingMessage(
                user_id=message.user_id,
                channel=message.channel,
                text=f"✅ Nice to meet you, {name}!\n\n"
                     f"📍 **Step 3: Select your district**\n\n"
                     f"Available districts for {profile.country_name}:\n"
                     f"{districts_text}\n\n"
                     f"Type your district name:",
            )
        ]

    async def _handle_district(
        self,
        message: IncomingMessage,
        profile: OnboardingProfile,
    ) -> list[OutgoingMessage]:
        """Step 3: District entry with fuzzy matching."""
        district_input = (message.text or "").strip().lower()

        if len(district_input) < 2:
            return [
                OutgoingMessage(
                    user_id=message.user_id,
                    channel=message.channel,
                    text="❌ Please enter a valid district name.",
                )
            ]

        # Fuzzy match against known districts
        known = COUNTRY_DISTRICTS.get(profile.country, [])
        match = self._fuzzy_match(district_input, known)

        if not match:
            return [
                OutgoingMessage(
                    user_id=message.user_id,
                    channel=message.channel,
                    text=f"⚠️ District '{district_input}' not found for {profile.country_name}.\n\n"
                         "Please check the spelling and try again, or type the "
                         "nearest district from the list.",
                )
            ]

        profile.district = match
        await self._save_state(message.user_id, profile)
        await self._set_step(message.user_id, OnboardingStep.ROLE)

        # Build role buttons (max 3 for WhatsApp, unlimited for Telegram)
        buttons = [
            {"id": f"onboard:role:{role[0]}", "label": f"{role[2]} {role[1]}"}
            for role in VALID_ROLES[:3]
        ]

        return [
            OutgoingMessage(
                user_id=message.user_id,
                channel=message.channel,
                text=f"✅ District: {match}\n\n"
                     f"👨‍⚕️ **Step 4: Select your role**\n\n"
                     f"What is your primary role?",
                buttons=buttons,
            )
        ]

    async def _handle_role(
        self,
        message: IncomingMessage,
        profile: OnboardingProfile,
    ) -> list[OutgoingMessage]:
        """Step 4: Role selection."""
        callback = message.callback_data or ""

        role_map = {f"onboard:role:{r[0]}": (r[0], r[1]) for r in VALID_ROLES}

        if callback not in role_map:
            buttons = [
                {"id": f"onboard:role:{role[0]}", "label": f"{role[2]} {role[1]}"}
                for role in VALID_ROLES[:3]
            ]
            return [
                OutgoingMessage(
                    user_id=message.user_id,
                    channel=message.channel,
                    text="Please select your role:",
                    buttons=buttons,
                )
            ]

        role_code, role_name = role_map[callback]
        profile.role = role_code
        profile.role_name = role_name
        await self._save_state(message.user_id, profile)
        await self._set_step(message.user_id, OnboardingStep.LANGUAGE)

        buttons = [
            {"id": f"onboard:lang:{lang[0]}", "label": f"{lang[2]} {lang[1]}"}
            for lang in VALID_LANGUAGES[:3]
        ]

        return [
            OutgoingMessage(
                user_id=message.user_id,
                channel=message.channel,
                text=f"✅ Role: {role_name}\n\n"
                     f"🌐 **Step 5: Select your language**\n\n"
                     f"Choose your preferred language for digests and alerts:",
                buttons=buttons,
            )
        ]

    async def _handle_language(
        self,
        message: IncomingMessage,
        profile: OnboardingProfile,
    ) -> list[OutgoingMessage]:
        """Step 5: Language selection → Complete."""
        callback = message.callback_data or ""

        lang_map = {
            f"onboard:lang:{l[0]}": (l[0], l[1]) for l in VALID_LANGUAGES
        }

        if callback not in lang_map:
            buttons = [
                {"id": f"onboard:lang:{lang[0]}", "label": f"{lang[2]} {lang[1]}"}
                for lang in VALID_LANGUAGES[:3]
            ]
            return [
                OutgoingMessage(
                    user_id=message.user_id,
                    channel=message.channel,
                    text="Please select your language:",
                    buttons=buttons,
                )
            ]

        lang_code, lang_name = lang_map[callback]
        profile.language = lang_code
        profile.language_name = lang_name
        profile.completed_at = datetime.utcnow()

        # Save to database
        await self.user_store.save_user_profile(profile)

        # Clear FSM state
        await self._clear_state(message.user_id)

        return [
            OutgoingMessage(
                user_id=message.user_id,
                channel=message.channel,
                text=(
                    f"🎉 **Welcome to UDARA AI, {profile.name}!**\n\n"
                    f"Here's your profile:\n"
                    f"  📍 {profile.country_name} — {profile.district}\n"
                    f"  👨‍⚕️ {profile.role_name}\n"
                    f"  🌐 {profile.language_name}\n\n"
                    f"You're all set! Here's what you can do:\n\n"
                    f"  📋 **Report** a suspected AMR case\n"
                    f"  🔬 **Check** drug resistance in your area\n"
                    f"  📊 **Daily digest** at 7:00 AM local time\n"
                    f"  🚨 **Peer alerts** when resistance spikes nearby\n\n"
                    f"Type /help for the full command list, or use "
                    f"the buttons below to get started."
                ),
            )
        ]

    # ─── Helpers ─────────────────────────────────────────────

    def _build_country_prompt(self, channel: Channel) -> OutgoingMessage:
        """Build the initial country selection prompt."""
        buttons = [
            {"id": "onboard:country:NG", "label": "🇳🇬 Nigeria"},
            {"id": "onboard:country:KE", "label": "🇰🇪 Kenya"},
            {"id": "onboard:country:GH", "label": "🇬🇭 Ghana"},
        ]
        return OutgoingMessage(
            user_id="",  # Set by caller
            channel=channel,
            text=(
                "🚀 **Welcome to UDARA AI!**\n\n"
                "Antimicrobial Resistance Surveillance\n"
                "for Community Health Workers\n\n"
                "📍 **Step 1: Select your country**"
            ),
            buttons=buttons,
        )

    @staticmethod
    def _fuzzy_match(input_str: str, candidates: list[str]) -> Optional[str]:
        """Simple fuzzy matching: case-insensitive substring match."""
        input_lower = input_str.lower()
        for candidate in candidates:
            if input_lower in candidate.lower() or candidate.lower() in input_lower:
                return candidate
        return None

    async def _save_state(self, user_id: str, profile: OnboardingProfile) -> None:
        """Save onboarding profile to state store."""
        key = f"onboarding:{user_id}"
        await self.state_store.set(key, json.dumps(profile.to_dict()), ex=3600)

    async def _load_profile(self, user_id: str) -> OnboardingProfile:
        """Load onboarding profile from state store."""
        key = f"onboarding:{user_id}"
        data = await self.state_store.get(key)
        if data:
            d = json.loads(data)
            d.pop("started_at", None)
            d.pop("completed_at", None)
            return OnboardingProfile(**d)
        return OnboardingProfile(user_id=user_id, channel=Channel.TELEGRAM)

    async def _set_step(self, user_id: str, step: OnboardingStep) -> None:
        """Update the current FSM step."""
        key = f"onboarding_step:{user_id}"
        await self.state_store.set(key, str(step.value), ex=3600)

    async def _clear_state(self, user_id: str) -> None:
        """Clear all onboarding state."""
        await self.state_store.delete(f"onboarding:{user_id}")
        await self.state_store.delete(f"onboarding_step:{user_id}")
```

---

## 8. Resistance Calculator Conversation

### 8.1 Conversation Flow

```
User: /resist
Bot:  🔬 Drug Resistance Checker
      Enter the antibiotic name:

User: amoxicillin
Bot:  🔍 Found: Amoxicillin
      Select condition:
      [1] Urinary Tract Infection
      [2] Respiratory Infection
      [3] Skin/Soft Tissue
      [4] Typhoid
      [5] General

User: 1
Bot:  📍 Select region:
      [1] Lagos
      [2] Abuja
      [3] Nationwide

User: 1
Bot:  ⚠️ Amoxicillin Resistance — Lagos, Nigeria
      🦠 UTI Treatment

      Resistance Probability:
      ████████████████░░░░  78%

      📊 Confidence: High (n=342 cases)

      ✅ Recommended Alternatives:
      1. Nitrofurantoin — 12% resistance █████░░░░░░░░░░
      2. Fosfomycin   —  8% resistance ███░░░░░░░░░░░░░
      3. Ciprofloxacin — 42% resistance ████████░░░░░░░░

      ⚠️ Avoid: Co-trimoxazole (89% resistance)

      📅 Last updated: 2025-01-14
      📖 Source: 342 cases from 23 facilities
```

### 8.2 Resistance Calculator Code

```python
# bots/resistance/resistance_calc.py

"""
Resistance Calculator — conversational interface for checking
drug resistance probability.

Features:
- Drug name extraction from free text
- Unicode probability bar visualization
- Alternative drug recommendations
- Confidence scoring based on sample size
"""

import logging
import re
from dataclasses import dataclass, field
from datetime import datetime, date
from enum import Enum
from typing import Optional

logger = logging.getLogger(__name__)


class ResistanceLevel(str, Enum):
    LOW = "low"          # < 30%
    MODERATE = "moderate" # 30–60%
    HIGH = "high"        # 60–80%
    CRITICAL = "critical" # > 80%


@dataclass
class DrugResistanceResult:
    """Result of a drug resistance query."""
    drug_name: str
    condition: str
    region: str
    country: str
    probability: float           # 0.0 to 1.0
    confidence: str             # "low", "medium", "high"
    sample_size: int
    alternatives: list[dict]    # [{"name": str, "probability": float}]
    warnings: list[str]
    last_updated: date
    source_facilities: int
    trend: Optional[str] = None  # "increasing", "stable", "decreasing"

    @property
    def resistance_level(self) -> ResistanceLevel:
        if self.probability < 0.3:
            return ResistanceLevel.LOW
        elif self.probability < 0.6:
            return ResistanceLevel.MODERATE
        elif self.probability < 0.8:
            return ResistanceLevel.HIGH
        else:
            return ResistanceLevel.CRITICAL

    @property
    def percentage(self) -> int:
        return int(self.probability * 100)


@dataclass
class ResistanceCalcSession:
    """State for an active resistance calculator conversation."""
    user_id: str
    step: int = 1  # 1=drug, 2=condition, 3=region
    drug_name: Optional[str] = None
    condition: Optional[str] = None
    region: Optional[str] = None


# Known conditions per drug (simplified mapping)
CONDITION_MAP = {
    "amoxicillin": ["Urinary Tract Infection", "Respiratory Infection",
                     "Skin/Soft Tissue", "Typhoid", "Otitis Media"],
    "ciprofloxacin": ["Urinary Tract Infection", "Respiratory Infection",
                       "Gastroenteritis", "Typhoid", "Bone Infection"],
    "ceftriaxone": ["Typhoid", "Meningitis", "Pneumonia",
                     "Pelvic Inflammatory Disease", "Sepsis"],
    "co_trimoxazole": ["Urinary Tract Infection", "Respiratory Infection",
                        "Pneumocystis", "Traveler's Diarrhea"],
    "metronidazole": ["Bacterial Vaginosis", "Amebiasis",
                       "Giardiasis", "Anaerobic Infections"],
    "azithromycin": ["Respiratory Infection", "Sexually Transmitted Infections",
                      "Traveler's Diarrhea", "Skin/Soft Tissue"],
    "gentamicin": ["Sepsis", "Urinary Tract Infection",
                    "Endocarditis", "Bone Infection"],
    "nitrofurantoin": ["Urinary Tract Infection"],
    "fosfomycin": ["Urinary Tract Infection"],
    "doxycycline": ["Respiratory Infection", "Sexually Transmitted Infections",
                     "Malaria prophylaxis", "Skin/Soft Tissue"],
}


class ResistanceFormatter:
    """Formats resistance results with Unicode bar charts."""

    FULL_BLOCK = "█"
    EMPTY_BLOCK = "░"
    BAR_WIDTH = 20

    @classmethod
    def probability_bar(cls, probability: float, width: int = 20) -> str:
        """Generate a Unicode probability bar.

        Example:
            0.78 → ████████████████░░░░
            0.12 → ███░░░░░░░░░░░░░░░░
        """
        filled = int(probability * width)
        empty = width - filled
        return cls.FULL_BLOCK * filled + cls.EMPTY_BLOCK * empty

    @classmethod
    def format_result(cls, result: DrugResistanceResult) -> str:
        """Format a DrugResistanceResult as a rich text message."""

        # Header with resistance level emoji
        level_emoji = {
            ResistanceLevel.LOW: "✅",
            ResistanceLevel.MODERATE: "⚠️",
            ResistanceLevel.HIGH: "🚨",
            ResistanceLevel.CRITICAL: "⛔",
        }
        emoji = level_emoji.get(result.resistance_level, "❓")

        # Main probability bar
        bar = cls.probability_bar(result.probability)
        pct = result.percentage

        # Confidence indicator
        conf_emoji = {
            "high": "🟢",
            "medium": "🟡",
            "low": "🔴",
        }
        conf_emoji_icon = conf_emoji.get(result.confidence, "⚪")

        lines = [
            f"{emoji} *{result.drug_name} Resistance* — "
            f"{result.region}, {result.country}",
            f"🦠 {result.condition}",
            "",
            f"Resistance Probability:",
            f"`{bar}  {pct}%`",
            "",
            f"{conf_emoji_icon} Confidence: {result.confidence.title()} "
            f"(n={result.sample_size:,} cases)",
        ]

        # Trend
        if result.trend:
            trend_emoji = {
                "increasing": "📈",
                "stable": "➡️",
                "decreasing": "📉",
            }
            lines.append(
                f"{trend_emoji.get(result.trend, '➡️')} Trend: "
                f"{result.trend.title()} over last 90 days"
            )

        lines.append("")

        # Alternatives
        if result.alternatives:
            lines.append("✅ *Recommended Alternatives:*")
            for i, alt in enumerate(result.alternatives, 1):
                alt_bar = cls.probability_bar(alt["probability"])
                alt_pct = int(alt["probability"] * 100)
                lines.append(
                    f"{i}. {alt['name']} — {alt_pct}% resistance `{alt_bar}`"
                )
            lines.append("")

        # Warnings
        if result.warnings:
            lines.append("⛔ *Warnings:*")
            for warning in result.warnings:
                lines.append(f"  • {warning}")
            lines.append("")

        # Metadata
        lines.extend([
            f"📅 Last updated: {result.last_updated.isoformat()}",
            f"📖 Source: {result.sample_size:,} cases from "
            f"{result.source_facilities} facilities",
        ])

        return "\n".join(lines)


class ResistanceCalculator:
    """
    Interactive resistance calculator.

    Manages the conversational flow for querying drug resistance data.
    """

    def __init__(self, resistance_service, state_store):
        self.service = resistance_service  # Connects to Agent B / Bayesian engine
        self.state_store = state_store
        self.formatter = ResistanceFormatter()

    async def start(self, user_id: str, channel: str) -> str:
        """Begin a new resistance calculation session."""
        session = ResistanceCalcSession(user_id=user_id, step=1)
        await self._save_session(user_id, session)

        return (
            "🔬 *Drug Resistance Checker*\n\n"
            "Enter the antibiotic name:\n\n"
            "Examples:\n"
            "• Amoxicillin\n"
            "• Ciprofloxacin\n"
            "• Ceftriaxone\n"
            "• Co-trimoxazole\n\n"
            "Type /cancel to exit."
        )

    async def process_message(
        self,
        user_id: str,
        text: str,
        channel: str,
    ) -> str:
        """Process a message in the resistance calculator conversation."""
        session = await self._load_session(user_id)
        if not session:
            return await self.start(user_id, channel)

        text = text.strip()

        if text.lower() in ("/cancel", "cancel", "exit"):
            await self._clear_session(user_id)
            return "❌ Resistance check cancelled."

        if session.step == 1:
            return await self._handle_drug(session, text)
        elif session.step == 2:
            return await self._handle_condition(session, text)
        elif session.step == 3:
            return await self._handle_region(session, text)

        return "⚠️ Unexpected state. Type /cancel and try again."

    async def _handle_drug(
        self,
        session: ResistanceCalcSession,
        text: str,
    ) -> str:
        """Step 1: Extract and validate drug name."""
        # Normalize drug name
        drug_normalized = self._normalize_drug_name(text)

        if drug_normalized not in CONDITION_MAP:
            similar = self._find_similar_drugs(text)
            hint = ""
            if similar:
                hint = "\n\nDid you mean:\n" + "\n".join(
                    f"  • {d.title()}" for d in similar[:5]
                )
            return (
                f"❌ Unknown drug: *{text}*\n"
                f"Please check the spelling and try again.{hint}\n\n"
                f"Type /cancel to exit."
            )

        session.drug_name = drug_normalized
        conditions = CONDITION_MAP[drug_normalized]
        session.step = 2
        await self._save_session(session.user_id, session)

        # Build condition buttons (1-indexed for callback)
        condition_list = "\n".join(
            f"  [{i+1}] {c}" for i, c in enumerate(conditions)
        )
        return (
            f"✅ Drug: *{drug_normalized.title()}*\n\n"
            f"Select condition:\n{condition_list}\n\n"
            f"Reply with the number or type the condition name."
        )

    async def _handle_condition(
        self,
        session: ResistanceCalcSession,
        text: str,
    ) -> str:
        """Step 2: Select condition."""
        conditions = CONDITION_MAP.get(session.drug_name, [])

        # Try numeric selection
        try:
            idx = int(text.strip()) - 1
            if 0 <= idx < len(conditions):
                session.condition = conditions[idx]
            else:
                raise ValueError("Out of range")
        except ValueError:
            # Try text match
            text_lower = text.lower()
            match = None
            for c in conditions:
                if text_lower in c.lower() or c.lower() in text_lower:
                    match = c
                    break
            if match:
                session.condition = match
            else:
                condition_list = "\n".join(
                    f"  [{i+1}] {c}" for i, c in enumerate(conditions)
                )
                return (
                    f"❌ Invalid condition. Please select:\n\n"
                    f"{condition_list}"
                )

        session.step = 3
        await self._save_session(session.user_id, session)

        return (
            f"✅ Drug: *{session.drug_name.title()}*\n"
            f"✅ Condition: *{session.condition}*\n\n"
            f"📍 Enter your district or region:\n\n"
            f"(Type 'national' for country-wide data)"
        )

    async def _handle_region(
        self,
        session: ResistanceCalcSession,
        text: str,
    ) -> str:
        """Step 3: Select region → compute and display results."""
        session.region = text.strip()
        await self._clear_session(session.user_id)

        # Query the resistance engine (Agent B)
        result = await self.service.get_resistance(
            drug=session.drug_name,
            condition=session.condition,
            region=session.region,
        )

        if not result:
            return (
                "❌ No data available for this query.\n\n"
                "This could mean:\n"
                "  • Not enough cases reported in this region yet\n"
                "  • Drug-condition combination not tracked\n\n"
                "Try 'national' for broader data, or report more cases "
                "to improve coverage. 🙏"
            )

        # Format the result
        return self.formatter.format_result(result)

    # ─── Helpers ─────────────────────────────────────────────

    @staticmethod
    def _normalize_drug_name(text: str) -> str:
        """Normalize drug name to lookup key."""
        return (
            text.lower()
            .replace("-", "")
            .replace("_", "")
            .replace(" ", "")
        )

    @staticmethod
    def _find_similar_drugs(text: str, threshold: int = 3) -> list[str]:
        """Find drugs with similar spelling using Levenshtein distance."""
        from difflib import get_close_matches
        normalized = text.lower().replace("-", "").replace(" ", "")
        matches = get_close_matches(normalized, CONDITION_MAP.keys(), n=5, cutoff=0.5)
        return matches

    async def _save_session(self, user_id: str, session: ResistanceCalcSession) -> None:
        import json
        key = f"resist_session:{user_id}"
        await self.state_store.set(
            key,
            json.dumps({
                "user_id": session.user_id,
                "step": session.step,
                "drug_name": session.drug_name,
                "condition": session.condition,
                "region": session.region,
            }),
            ex=1800,  # 30 min TTL
        )

    async def _load_session(self, user_id: str) -> Optional[ResistanceCalcSession]:
        import json
        key = f"resist_session:{user_id}"
        data = await self.state_store.get(key)
        if data:
            d = json.loads(data)
            return ResistanceCalcSession(**d)
        return None

    async def _clear_session(self, user_id: str) -> None:
        await self.state_store.delete(f"resist_session:{user_id}")
```

---

## 9. Daily Digest Scheduler

### 9.1 Digest Architecture

```
┌─────────────────────────────────────────────────────┐
│              DAILY DIGEST PIPELINE                   │
│                                                     │
│  ┌───────────┐    ┌──────────────┐    ┌──────────┐ │
│  │  Cron Job  │───►│  Digest      │───►│  Channel │ │
│  │  (per TZ)  │    │  Generator   │    │  Router  │ │
│  └───────────┘    └──────┬───────┘    └─────┬────┘ │
│                          │                   │       │
│  Timezones:              ▼                   │       │
│  ┌───────────────────────────────┐            │       │
│  │ WAT +1:00  NG, GH → 06:00 UTC│            │       │
│  │ EAT +3:00  KE, TZ, UG, ET   │            │       │
│  │           → 04:00 UTC         │            │       │
│  └───────────────────────────────┘            │       │
│                                               │       │
│                    Per-user data:             │       │
│                    ├─ district                │       │
│                    ├─ role                    │       │
│                    ├─ language                │       │
│                    ├─ last digest sent        │       │
│                    └─ active channels         │       │
│                                               │       │
│                    ┌──────────┐               │       │
│                    │  Digest  │◄──────────────┘       │
│                    │  Content │                       │
│                    ├──────────┤                       │
│                    │ 📊 New cases in district       │
│                    │ 🚨 Resistance spike alerts      │
│                    │ 📈 Trend changes               │
│                    │ 💊 Drug recommendations        │
│                    │ 🌍 National AMR news           │
│                    │ 📚 Educational tips            │
│                    └──────────┘                       │
└─────────────────────────────────────────────────────┘
```

### 9.2 Daily Digest Code

```python
# bots/scheduler/daily_digest.py

"""
Daily Digest Scheduler

Sends personalized AMR intelligence digests to all users
at their local 7:00 AM based on timezone.

Uses APScheduler for cron-based scheduling with timezone support.
"""

import asyncio
import logging
from dataclasses import dataclass, field
from datetime import datetime, date, timedelta
from typing import Any, Optional

from apscheduler.schedulers.asyncio import AsyncIOScheduler
from apscheduler.triggers.cron import CronTrigger
import pytz

from bots.core.schemas import OutgoingMessage, Channel

logger = logging.getLogger(__name__)


# Timezone → cron schedule mapping
TIMEZONE_SCHEDULES = {
    "Africa/Lagos": {"hour": 7, "minute": 0},      # WAT — Nigeria, Ghana
    "Africa/Nairobi": {"hour": 7, "minute": 0},    # EAT — Kenya, Tanzania, Uganda, Ethiopia
    "Africa/Accra": {"hour": 7, "minute": 0},      # GMT — Ghana (UTC)
}


@dataclass
class DigestContent:
    """Personalized digest content for a single user."""
    user_id: str
    district: str
    country: str
    language: str

    # Content sections
    new_cases_count: int = 0
    new_cases_detail: list[dict] = field(default_factory=list)
    resistance_alerts: list[dict] = field(default_factory=list)
    trend_changes: list[dict] = field(default_factory=list)
    drug_recommendations: list[dict] = field(default_factory=list)
    national_news: list[dict] = field(default_factory=list)
    educational_tip: Optional[str] = None

    @property
    def has_content(self) -> bool:
        return (
            self.new_cases_count > 0
            or len(self.resistance_alerts) > 0
            or len(self.trend_changes) > 0
        )

    def format_telegram(self) -> str:
        """Format digest for Telegram (Markdown)."""
        lines = [
            f"📊 *Daily AMR Digest* — {self.district}",
            f"📅 {date.today().strftime('%B %d, %Y')}",
            "",
        ]

        if self.new_cases_count > 0:
            lines.append(f"📋 *New Cases: {self.new_cases_count}*")
            for case in self.new_cases_detail[:5]:
                lines.append(f"  • {case['summary']}")
            if self.new_cases_count > 5:
                lines.append(f"  ... and {self.new_cases_count - 5} more")
            lines.append("")

        if self.resistance_alerts:
            lines.append("🚨 *Resistance Alerts*")
            for alert in self.resistance_alerts[:3]:
                emoji = "⛔" if alert["level"] == "critical" else "⚠️"
                lines.append(f"  {emoji} {alert['drug']} ({alert['probability']}%) — {alert['condition']}")
            lines.append("")

        if self.trend_changes:
            lines.append("📈 *Trend Changes*")
            for trend in self.trend_changes[:3]:
                emoji = {"increasing": "📈", "decreasing": "📉", "stable": "➡️"}[trend["direction"]]
                lines.append(f"  {emoji} {trend['drug']}: {trend['direction']} ({trend['change']}%)")
            lines.append("")

        if self.drug_recommendations:
            lines.append("💊 *Drug Recommendations*")
            for rec in self.drug_recommendations[:2]:
                lines.append(f"  ✅ {rec['drug']} for {rec['condition']} ({rec['efficacy']}%)")
            lines.append("")

        if self.educational_tip:
            lines.extend([
                "📚 *Did You Know?*",
                f"  {self.educational_tip}",
                "",
            ])

        if not self.has_content:
            lines.extend([
                "📭 No new updates in your area today.",
                "",
                "Keep reporting cases to improve AMR surveillance!",
                "",
                "Use /resist to check any drug's resistance level.",
            ])

        lines.extend([
            "─" * 20,
            "🔒 Your data is encrypted and stays in Africa.",
            "Type /help for all commands.",
        ])

        return "\n".join(lines)

    def format_whatsapp(self) -> str:
        """Format digest for WhatsApp (bold with asterisks)."""
        # Same content, WhatsApp uses *bold* and _italic_
        text = self.format_telegram()
        # Telegram uses `` for code, WhatsApp doesn't support it
        text = text.replace("`", "")
        return text


@dataclass
class DigestUser:
    """A user who should receive a daily digest."""
    user_id: str
    phone_number: Optional[str]
    telegram_id: Optional[str]
    channel: Channel
    country: str
    district: str
    language: str
    timezone: str
    is_active: bool = True
    digest_enabled: bool = True
    last_digest_sent: Optional[date] = None


class DailyDigestScheduler:
    """
    Scheduler for daily personalized AMR digests.

    Usage:
        scheduler = DailyDigestScheduler(
            user_repo=user_repository,
            data_service=amr_data_service,
            bot_core=bot_intelligence_core,
        )
        scheduler.start()
    """

    def __init__(
        self,
        user_repo: Any,
        data_service: Any,
        bot_core: Any,
        dry_run: bool = False,
    ):
        self.user_repo = user_repo
        self.data_service = data_service
        self.bot_core = bot_core
        self.dry_run = dry_run
        self.scheduler = AsyncIOScheduler(timezone="UTC")
        self._running = False

    def start(self) -> None:
        """Start the scheduler."""
        if self._running:
            return

        # Register cron jobs per timezone
        for tz_name, schedule in TIMEZONE_SCHEDULES.items():
            self.scheduler.add_job(
                self._run_digest_for_timezone,
                CronTrigger(hour=schedule["hour"], minute=schedule["minute"],
                            timezone=tz_name),
                id=f"digest_{tz_name}",
                name=f"Daily digest for {tz_name}",
                replace_existing=True,
                max_instances=1,
            )
            logger.info(
                "Scheduled digest for %s at %02d:%02d local",
                tz_name,
                schedule["hour"],
                schedule["minute"],
            )

        self.scheduler.start()
        self._running = True
        logger.info("Daily digest scheduler started")

    def stop(self) -> None:
        """Stop the scheduler."""
        if self._running:
            self.scheduler.shutdown(wait=False)
            self._running = False
            logger.info("Daily digest scheduler stopped")

    async def _run_digest_for_timezone(self, timezone: str) -> None:
        """
        Generate and send digests for all users in a timezone.

        This is the main entry point called by the cron scheduler.
        """
        logger.info("Starting digest run for timezone: %s", timezone)

        # Get all active users in this timezone who have digest enabled
        users = await self.user_repo.get_users_for_digest(timezone)

        if not users:
            logger.info("No users found for timezone: %s", timezone)
            return

        logger.info(
            "Processing digests for %d users in %s",
            len(users),
            timezone,
        )

        stats = {"sent": 0, "skipped": 0, "failed": 0}

        for user in users:
            try:
                # Check if already sent today
                today = date.today()
                if user.last_digest_sent == today:
                    stats["skipped"] += 1
                    continue

                # Generate personalized content
                content = await self._generate_digest_content(user)

                # Format per channel
                if user.channel == Channel.TELEGRAM:
                    formatted = content.format_telegram()
                else:
                    formatted = content.format_whatsapp()

                # Send
                if not self.dry_run:
                    message = OutgoingMessage(
                        user_id=user.user_id,
                        channel=user.channel,
                        text=formatted,
                    )
                    await self.bot_core.send_message(message)

                # Update last sent timestamp
                await self.user_repo.mark_digest_sent(
                    user.user_id, today
                )

                stats["sent"] += 1

                # Rate limit: avoid hitting API limits
                await asyncio.sleep(0.1)

            except Exception as e:
                logger.error(
                    "Failed to send digest to %s: %s",
                    user.user_id,
                    e,
                    exc_info=True,
                )
                stats["failed"] += 1

        logger.info(
            "Digest run complete for %s: sent=%d, skipped=%d, failed=%d",
            timezone,
            stats["sent"],
            stats["skipped"],
            stats["failed"],
        )

    async def _generate_digest_content(
        self,
        user: DigestUser,
    ) -> DigestContent:
        """Generate personalized digest content for a user."""
        since = datetime.utcnow() - timedelta(hours=24)

        # Fetch data from the AMR service
        new_cases = await self.data_service.get_cases_since(
            district=user.district,
            country=user.country,
            since=since,
        )

        alerts = await self.data_service.get_resistance_alerts(
            district=user.district,
            country=user.country,
        )

        trends = await self.data_service.get_trend_changes(
            district=user.district,
            country=user.country,
        )

        recommendations = await self.data_service.get_recommendations(
            district=user.district,
            country=user.country,
            role=user.role if hasattr(user, 'role') else None,
        )

        return DigestContent(
            user_id=user.user_id,
            district=user.district,
            country=user.country,
            language=user.language,
            new_cases_count=len(new_cases),
            new_cases_detail=new_cases[:10],
            resistance_alerts=alerts,
            trend_changes=trends,
            drug_recommendations=recommendations,
            educational_tip=await self._get_educational_tip(user.language),
        )

    async def _get_educational_tip(self, language: str) -> str:
        """Get a rotating educational tip in the user's language."""
        tips = {
            "en": [
                "Antibiotics do NOT treat viral infections like colds or flu. "
                "Using them for viruses contributes to resistance.",
                "Always complete the full antibiotic course, even if you feel better. "
                "Stopping early can lead to resistant bacteria.",
                "Resistance can spread between bacteria of different species "
                "through horizontal gene transfer (plasmids).",
            ],
            "sw": [
                "Antibiotiki hazitibu maambukizi ya virusi kama mafua. "
                "Kuzitumia kwa virusi husababisha upinzani.",
                "Kamaliza dozi kamili ya antibaotiki, hata ukiwa umepata nafuu.",
            ],
        }
        # Rotate based on day of year
        tips_list = tips.get(language, tips["en"])
        day_of_year = date.today().timetuple().tm_yday
        return tips_list[day_of_year % len(tips_list)]

    async def trigger_manual_digest(
        self,
        user_id: str,
    ) -> Optional[str]:
        """Manually trigger a digest for a specific user (via /digest command)."""
        user = await self.user_repo.get_user(user_id)
        if not user:
            return None

        content = await self._generate_digest_content(user)
        if user.channel == Channel.TELEGRAM:
            return content.format_telegram()
        return content.format_whatsapp()
```

---

## 10. Peer Alert Network

### 10.1 Alert Propagation Flow

```
Resistance Spike Detected
        │
        ▼
┌───────────────────────┐
│  Agent B (Bayesian)   │
│  detects spike:       │
│  - Drug: Ciprofloxacin│
│  - District: Lagos    │
│  - Spike: 12% → 45%  │
│  - Window: 30 days    │
└───────────┬───────────┘
            │
            ▼
┌───────────────────────────────────┐
│  Alert Engine                      │
│                                    │
│  1. Validate spike (min n=20)     │
│  2. Check cooldown (max 1/district│
│     every 24h)                    │
│  3. Query all CHWs in district    │
│  4. Batch send (max 30/sec)       │
│  5. Log alert for audit           │
└───────────┬───────────────────────┘
            │
            ▼
┌──────────────────────────────────────┐
│  Per-CHW Notification                 │
│                                       │
│  🚨 Resistance Spike Alert           │
│  📍 Lagos Mainland, Nigeria          │
│                                       │
│  Ciprofloxacin resistance for UTI    │
│  jumped from 12% to 45% in 30 days.  │
│                                       │
│  ✅ Consider switching to:            │
│  • Nitrofurantoin (12% resist.)       │
│  • Fosfomycin (8% resist.)           │
│                                       │
│  [View Details] [Mute for 24h]       │
└──────────────────────────────────────┘
```

### 10.2 Peer Alert Implementation

```python
# bots/alerts/peer_alert.py

"""
Peer Alert Network — notifies all CHWs in a district
when a significant resistance spike is detected.

Features:
- Rate-limited: max 1 alert per district per drug per 24h
- Batched sending to respect API rate limits
- Per-user mute/unsubscribe
- Escalation: critical alerts bypass mute
"""

import asyncio
import logging
from dataclasses import dataclass
from datetime import datetime, timedelta
from typing import Optional

logger = logging.getLogger(__name__)


@dataclass
class ResistanceAlert:
    """A resistance spike alert to be distributed."""
    drug: str
    condition: str
    district: str
    country: str
    old_probability: float
    new_probability: float
    change_percentage: float
    sample_size: int
    severity: str  # "warning", "critical"
    detected_at: datetime
    recommendations: list[dict]


class PeerAlertEngine:
    """
    Manages the peer alert network.

    Monitors for resistance spikes and notifies all active
    CHWs in the affected district.
    """

    def __init__(
        self,
        user_repo,
        alert_repo,
        bot_core,
        cooldown_hours: int = 24,
        min_sample_size: int = 20,
        min_change_threshold: float = 15.0,  # 15 percentage points
        critical_threshold: float = 30.0,      # 30 percentage points
        batch_size: int = 25,
        batch_delay_seconds: float = 1.0,
    ):
        self.user_repo = user_repo
        self.alert_repo = alert_repo
        self.bot_core = bot_core
        self.cooldown_hours = cooldown_hours
        self.min_sample_size = min_sample_size
        self.min_change_threshold = min_change_threshold
        self.critical_threshold = critical_threshold
        self.batch_size = batch_size
        self.batch_delay = batch_delay_seconds

    async def check_and_alert(self, alert: ResistanceAlert) -> dict:
        """
        Check if an alert should be sent and distribute it.

        Returns:
            Dict with alert stats: {"sent": int, "skipped": int, "muted": int}
        """
        # 1. Validate sample size
        if alert.sample_size < self.min_sample_size:
            logger.info(
                "Skipping alert: sample size %d < minimum %d",
                alert.sample_size,
                self.min_sample_size,
            )
            return {"sent": 0, "skipped": 0, "muted": 0, "reason": "insufficient_sample"}

        # 2. Validate change threshold
        if alert.change_percentage < self.min_change_threshold:
            logger.info(
                "Skipping alert: change %.1f%% < threshold %.1f%%",
                alert.change_percentage,
                self.min_change_threshold,
            )
            return {"sent": 0, "skipped": 0, "muted": 0, "reason": "below_threshold"}

        # 3. Determine severity
        if alert.change_percentage >= self.critical_threshold:
            alert.severity = "critical"
        else:
            alert.severity = "warning"

        # 4. Check cooldown
        is_on_cooldown = await self.alert_repo.is_on_cooldown(
            drug=alert.drug,
            district=alert.district,
            hours=self.cooldown_hours,
        )
        if is_on_cooldown and alert.severity != "critical":
            logger.info(
                "Skipping non-critical alert: %s in %s is on cooldown",
                alert.drug,
                alert.district,
            )
            return {"sent": 0, "skipped": 0, "muted": 0, "reason": "cooldown"}

        # 5. Get all CHWs in the district
        chw_users = await self.user_repo.get_active_users_by_district(
            district=alert.district,
            country=alert.country,
        )

        if not chw_users:
            logger.info(
                "No active CHWs in %s, %s",
                alert.district,
                alert.country,
            )
            return {"sent": 0, "skipped": 0, "muted": 0, "reason": "no_users"}

        # 6. Filter out muted users (except critical alerts)
        stats = {"sent": 0, "skipped": 0, "muted": 0}

        recipients = []
        for user in chw_users:
            if alert.severity == "critical":
                recipients.append(user)
            elif await self.alert_repo.is_user_muted(user.user_id):
                stats["muted"] += 1
            else:
                recipients.append(user)

        # 7. Batch send
        for i in range(0, len(recipients), self.batch_size):
            batch = recipients[i : i + self.batch_size]
            tasks = [
                self._send_alert(user, alert) for user in batch
            ]
            results = await asyncio.gather(*tasks, return_exceptions=True)

            for result in results:
                if isinstance(result, Exception):
                    logger.error("Alert send failed: %s", result)
                    stats["skipped"] += 1
                elif result:
                    stats["sent"] += 1

            # Rate limit delay between batches
            if i + self.batch_size < len(recipients):
                await asyncio.sleep(self.batch_delay)

        # 8. Set cooldown
        await self.alert_repo.set_cooldown(
            drug=alert.drug,
            district=alert.district,
            hours=self.cooldown_hours,
        )

        # 9. Log the alert
        await self.alert_repo.log_alert(alert, stats)

        logger.info(
            "Alert sent: drug=%s, district=%s, sent=%d, muted=%d",
            alert.drug,
            alert.district,
            stats["sent"],
            stats["muted"],
        )

        return stats

    async def _send_alert(self, user, alert: ResistanceAlert) -> bool:
        """Send a formatted alert to a single user."""
        severity_emoji = "🚨" if alert.severity == "critical" else "⚠️"
        direction = "increased" if alert.new_probability > alert.old_probability else "decreased"

        text = (
            f"{severity_emoji} *Resistance Spike Alert*\n\n"
            f"📍 {alert.district}, {alert.country}\n\n"
            f"*{alert.drug}* resistance for *{alert.condition}*\n"
            f"{direction} from {int(alert.old_probability * 100)}% "
            f"to {int(alert.new_probability * 100)}%\n\n"
        )

        if alert.recommendations:
            text += "✅ *Consider switching to:*\n"
            for rec in alert.recommendations[:3]:
                text += f"  • {rec['drug']} ({int(rec['probability'] * 100)}% resistance)\n"
            text += "\n"

        text += (
            f"📊 Based on {alert.sample_size:,} cases\n"
            f"📅 Detected: {alert.detected_at.strftime('%Y-%m-%d %H:%M UTC')}\n\n"
            f"_Type /resist for detailed analysis_"
        )

        message = OutgoingMessage(
            user_id=user.user_id,
            channel=user.channel,
            text=text,
        )
        await self.bot_core.send_message(message)
        return True
```

---

## 11. Country Configuration Templates

### 11.1 JSON Template

```json
{
  "country_code": "NG",
  "country_name": "Nigeria",
  "channels": {
    "primary": "whatsapp",
    "secondary": "telegram",
    "ussd_enabled": true,
    "sms_enabled": false
  },
  "languages": {
    "primary": "en",
    "supported": ["en", "ha", "yo", "ig"],
    "default_fallback": "en",
    "lid_model": "fasttext_lid_175.bin"
  },
  "ussd_code": "*347*2#",
  "telco": "MTN",
  "currency": "NGN",
  "timezone": "Africa/Lagos",
  "utc_offset": "+01:00",
  "digest_schedule": {
    "hour": 7,
    "minute": 0,
    "timezone": "Africa/Lagos"
  },
  "districts": [
    {"code": "LG01", "name": "Lagos Mainland", "region": "Lagos", "population_estimate": 3200000},
    {"code": "LG02", "name": "Lagos Island", "region": "Lagos", "population_estimate": 2100000},
    {"code": "AB01", "name": "Abuja Municipal", "region": "FCT", "population_estimate": 3600000},
    {"code": "KN01", "name": "Kano Municipal", "region": "Kano", "population_estimate": 4100000},
    {"code": "IB01", "name": "Ibadan North", "region": "Oyo", "population_estimate": 3100000},
    {"code": "PH01", "name": "Port Harcourt", "region": "Rivers", "population_estimate": 2300000},
    {"code": "BN01", "name": "Benin City", "region": "Edo", "population_estimate": 1800000},
    {"code": "KD01", "name": "Kaduna North", "region": "Kaduna", "population_estimate": 2100000}
  ],
  "common_antibiotics": [
    {"name": "amoxicillin", "generic_available": true, "otc_available": false},
    {"name": "ciprofloxacin", "generic_available": true, "otc_available": true},
    {"name": "ceftriaxone", "generic_available": true, "otc_available": false},
    {"name": "co_trimoxazole", "generic_available": true, "otc_available": true},
    {"name": "metronidazole", "generic_available": true, "otc_available": true},
    {"name": "azithromycin", "generic_available": true, "otc_available": false},
    {"name": "gentamicin", "generic_available": true, "otc_available": false}
  ],
  "health_facilities_count_estimate": 40000,
  "chw_count_target": 5000,
  "data_retention_days": 365,
  "privacy": {
    "gdpr_applicable": false,
    "ndpr_applicable": true,
    "data_residency": "AWS af-south-1"
  }
}
```

### 11.2 Full Country Configurations Table

| Field | Nigeria (NG) | Kenya (KE) | Tanzania (TZ) | Ghana (GH) | Uganda (UG) | Ethiopia (ET) |
|-------|-------------|------------|---------------|-----------|------------|---------------|
| Primary Channel | WhatsApp | Telegram | Telegram | WhatsApp | Telegram | Telegram |
| Secondary Channel | Telegram | USSD | USSD | Telegram | USSD | USSD |
| Languages | en, ha, yo, ig | en, sw | en, sw | en, ak, tw, ee | en, sw, lg | en, am, om, ti |
| Timezone | Africa/Lagos | Africa/Nairobi | Africa/Dar_es_Salaam | Africa/Accra | Africa/Kampala | Africa/Addis_Ababa |
| USSD Code | *347*2# | *384*123# | *149*11# | *714# | *165*3# | *948# |
| Telco Partner | MTN | Safaricom | Vodacom | MTN | MTN | Ethio Telecom |
| Currency | NGN | KES | TZS | GHS | UGX | ETB |
| Privacy Law | NDPR | DPA 2019 | POCA | DPA 2012 | DPA 2019 | TBD |
| Data Residency | af-south-1 | af-south-1 | af-south-1 | af-south-1 | af-south-1 | af-south-1 |

---

## 12. Multilingual Support

### 12.1 Language Map

```python
# bots/i18n/languages.py

SUPPORTED_LANGUAGES = {
    "en": {
        "name": "English",
        "flag": "🇬🇧",
        "countries": ["NG", "KE", "TZ", "GH", "UG", "ET"],
        "direction": "ltr",
        "fasttext_code": "eng_Latn",
    },
    "ha": {
        "name": "Hausa",
        "flag": "🇳🇬",
        "countries": ["NG"],
        "direction": "ltr",
        "fasttext_code": "hau_Latn",
    },
    "yo": {
        "name": "Yoruba",
        "flag": "🇳🇬",
        "countries": ["NG"],
        "direction": "ltr",
        "fasttext_code": "yor_Latn",
    },
    "ig": {
        "name": "Igbo",
        "flag": "🇳🇬",
        "countries": ["NG"],
        "direction": "ltr",
        "fasttext_code": "ibo_Latn",
    },
    "sw": {
        "name": "Swahili",
        "flag": "🇰🇪",
        "countries": ["KE", "TZ", "UG"],
        "direction": "ltr",
        "fasttext_code": "swa_Latn",
    },
    "am": {
        "name": "Amharic",
        "flag": "🇪🇹",
        "countries": ["ET"],
        "direction": "ltr",
        "fasttext_code": "amh_Ethi",
    },
}

# Translation templates
TEMPLATES = {
    "welcome": {
        "en": "🚀 Welcome to UDARA AI!\n\nAntimicrobial Resistance Surveillance\nfor Community Health Workers",
        "ha": "🚀 Bari da zuwa UDARA AI!\n\nTsaro tsarin ƙwayoyin cuta\nMa'aikatan Lafiya na Al'umma",
        "yo": "🚀 Ku abura UDARA AI!\n\nIṣẹ́ ìkànìyàn fún àwọn ẹlẹ́kọ́ọ́sàní",
        "ig": "🚀 Nnọọ UDARA AI!\n\nNlekọta ọgụgụ nje mmadụ\nMaka ndị ọrụ ahụike obodo",
        "sw": "🚀 Karibu UDARA AI!\n\nUfuatiliaji wa Upinzani wa Dawa",
        "am": "🚀 እንኳን ወደ UDARA AI በደህና መጡ!\n\nየአንቲባዮቲክ ተቃውሞ ክትትል",
    },
    "resistance_high": {
        "en": "🚨 HIGH RESISTANCE: {drug} shows {percentage}% resistance in {district}",
        "ha": "🚨 GABA TAYA: {drug} yana nuna {percentage}% juriya a {district}",
        "sw": "🚨 UPINZANI MKUBWA: {drug} inaonyesha {percentage}% upinzani katika {district}",
    },
    "report_success": {
        "en": "✅ Case reported successfully! Case ID: {case_id}",
        "ha": "✅ An bayar da rahoton nasara! Lambar shari'a: {case_id}",
        "sw": "✅ Ripoti imewasilishwa kwa mafanikio! Nambari ya kesi: {case_id}",
    },
}
```

### 12.2 Language Detection with fastText

```python
# bots/i18n/language_detector.py

"""
Language detection using fastText.

Uses the lid.176.bin model which supports 176 languages
including all African languages used by UDARA AI.
"""

import logging
from pathlib import Path
from typing import Optional

import fasttext

logger = logging.getLogger(__name__)


class LanguageDetector:
    """
    Detects the language of input text using fastText.

    Features:
    - Lazy model loading (loaded on first use)
    - Minimum text length threshold
    - Confidence threshold
    - Force-set language override from user profile
    - Fallback to English if detection fails
    """

    MODEL_PATH = Path("models/lid.176.bin")
    MIN_TEXT_LENGTH = 3
    CONFIDENCE_THRESHOLD = 0.5
    FASTTEXT_TO_UDARA = {
        "eng_Latn": "en",
        "hau_Latn": "ha",
        "yor_Latn": "yo",
        "ibo_Latn": "ig",
        "swa_Latn": "sw",
        "amh_Ethi": "am",
        "afr_Latn": "en",   # Afrikaans → fallback to English
        "fra_Latn": "en",   # French → fallback to English
        "por_Latn": "en",   # Portuguese → fallback to English
        "ara_Arab": "en",   # Arabic → fallback to English
    }

    def __init__(self):
        self._model: Optional[fasttext.FastText._FastText] = None

    def _load_model(self) -> None:
        """Lazily load the fastText model."""
        if self._model is not None:
            return

        logger.info("Loading fastText language detection model...")
        self._model = fasttext.load_model(str(self.MODEL_PATH))
        logger.info("fastText model loaded successfully")

    def detect(self, text: str) -> dict:
        """
        Detect the language of the given text.

        Returns:
            {
                "language": "en",     # UDARA language code
                "confidence": 0.95,   # 0.0 to 1.0
                "raw_label": "eng_Latn",  # fastText label
            }
        """
        if len(text.strip()) < self.MIN_TEXT_LENGTH:
            return {
                "language": "en",
                "confidence": 0.0,
                "raw_label": "too_short",
            }

        self._load_model()

        # fastText predict returns labels and probabilities
        predictions = self._model.predict(text.replace("\n", " "), k=1)
        raw_label = predictions[0][0].replace("__label__", "")
        confidence = float(predictions[1][0])

        # Map to UDARA language code
        language = self.FASTTEXT_TO_UDARA.get(raw_label, "en")

        # If confidence is too low, fall back to English
        if confidence < self.CONFIDENCE_THRESHOLD:
            logger.debug(
                "Low confidence detection: %s (%.2f) for text: %.40s",
                raw_label,
                confidence,
                text,
            )
            return {
                "language": "en",
                "confidence": confidence,
                "raw_label": raw_label,
            }

        return {
            "language": language,
            "confidence": confidence,
            "raw_label": raw_label,
        }
```

---

## 13. Testing Strategy

### 13.1 Test Categories

| Category | Tool | Coverage Target | Run Frequency |
|----------|------|----------------|---------------|
| Unit Tests | pytest + pytest-asyncio | 90%+ | Every commit |
| Integration Tests | pytest + httpx test client | Key flows | Every PR |
| E2E Tests | Bot testing framework | Critical paths | Daily |
| Load Tests | locust | 1000 concurrent users | Weekly |
| Security Tests | bandit + safety | 0 vulnerabilities | Every PR |

### 13.2 Test File Structure

```
tests/bots/
├── conftest.py                    # Shared fixtures
├── test_onboarding.py             # FSM state machine tests
├── test_resistance_calc.py        # Resistance calculator tests
├── test_daily_digest.py           # Digest scheduler tests
├── test_peer_alert.py             # Alert rate limiting tests
├── test_telegram/
│   ├── test_bot.py                # Bot initialization
│   ├── test_handlers_text.py      # Text handler routing
│   ├── test_handlers_photo.py     # Photo processing
│   ├── test_handlers_voice.py     # Voice → ASR pipeline
│   ├── test_keyboards.py          # Keyboard generation
│   └── test_webhook_parsing.py    # Webhook payload parsing
├── test_whatsapp/
│   ├── test_whatsapp_bot.py       # WhatsAppBot class
│   ├── test_send_text.py          # Text sending
│   ├── test_send_buttons.py       # Interactive buttons
│   ├── test_download_media.py     # Media download
│   └── test_signature_verify.py   # HMAC verification
└── test_i18n/
    ├── test_language_detector.py  # fastText LID
    └── test_translations.py       # Template rendering
```

---

## 14. Operational Runbooks

### 14.1 Bot Down Runbook

```
┌─────────────────────────────────────────────────────┐
│              BOT DOWN — RESPONSE RUNBOOK             │
├─────────────────────────────────────────────────────┤
│                                                     │
│  1. DETECT                                          │
│     - Health check fails (every 60s)                │
│     - Grafana alert fires                           │
│     - Error rate > 5% for 5 min                    │
│                                                     │
│  2. DIAGNOSE                                       │
│     $ ssh edge-node-01                              │
│     $ docker compose ps                             │
│     $ docker compose logs telegram-bot --tail=100   │
│     $ docker compose logs whatsapp-bot --tail=100   │
│     $ redis-cli ping                                │
│                                                     │
│  3. RESTART (if needed)                             │
│     $ docker compose restart telegram-bot           │
│     $ docker compose restart whatsapp-bot           │
│                                                     │
│  4. VERIFY                                          │
│     $ curl http://localhost:8080/health             │
│     - Send test message to bot                     │
│     - Check webhook logs on Meta dashboard          │
│                                                     │
│  5. COMMUNICATE                                     │
│     - Post in #incidents Slack channel             │
│     - If > 30 min downtime, send WhatsApp broadcast │
│       to affected country coordinators             │
│                                                     │
│  6. POST-MORTEM                                     │
│     - Create incident ticket within 24h             │
│     - Root cause analysis within 72h               │
│     - Prevention action item within 1 week         │
│                                                     │
└─────────────────────────────────────────────────────┘
```

### 14.2 Webhook Verification Runbook (WhatsApp)

```
1. Meta sends GET to /webhook/whatsapp with:
   - hub.mode = "subscribe"
   - hub.verify_token = your_verify_token
   - hub.challenge = random_string

2. Your server must:
   - Verify hub.mode == "subscribe"
   - Verify hub.verify_token matches your stored token
   - Return hub.challenge as plain text (200 OK)

3. If webhook setup fails:
   - Check server is reachable from internet (ngrok for dev)
   - Check SSL certificate is valid
   - Check verify_token matches between Meta dashboard and code
   - Check firewall allows inbound HTTPS (443)
   - Check response is plain text, not JSON
```

---

## 15. Appendix: Full Source Code Index

```
bots/
├── core/
│   ├── __init__.py
│   ├── schemas.py              # IncomingMessage, OutgoingMessage
│   ├── adapter_base.py         # ChannelAdapter ABC
│   ├── intelligence_core.py    # Bot Intelligence Core
│   ├── intents.py              # IntentClassifier
│   └── onboarding.py           # 5-step FSM engine
├── telegram/
│   ├── __init__.py
│   ├── bot.py                  # UDARATelegramBot class
│   ├── config.py               # TelegramConfig
│   ├── handlers/
│   │   ├── __init__.py
│   │   ├── start.py            # /start, /help
│   │   ├── text.py             # Text routing
│   │   ├── photo.py            # Photo handler
│   │   ├── voice.py            # Voice → ASR
│   │   ├── callback.py         # Inline buttons
│   │   └── error.py            # Error handler
│   ├── keyboards/
│   │   ├── __init__.py
│   │   ├── main_menu.py        # Main menu
│   │   ├── onboarding.py       # Onboarding keyboards
│   │   └── resistance.py       # Resistance keyboards
│   ├── formatters/
│   │   ├── __init__.py
│   │   ├── resistance.py       # Unicode bars
│   │   ├── digest.py           # Digest formatting
│   │   └── alert.py            # Alert formatting
│   ├── middlewares/
│   │   ├── __init__.py
│   │   ├── auth.py             # User auth
│   │   ├── locale.py           # Language detection
│   │   └── rate_limit.py       # Rate limiting
│   └── filters/
│       └── onboarding.py       # FSM filters
├── whatsapp/
│   ├── __init__.py
│   ├── whatsapp_bot.py         # WhatsAppBot class
│   ├── webhook.py              # FastAPI webhook handler
│   └── config.py               # WhatsAppConfig
├── resistance/
│   └── resistance_calc.py      # Resistance calculator
├── scheduler/
│   └── daily_digest.py         # Daily digest scheduler
├── alerts/
│   └── peer_alert.py           # Peer alert engine
└── i18n/
    ├── __init__.py
    ├── languages.py            # Language map + templates
    └── language_detector.py    # fastText LID
```

---

*End of Document UDARA-ARCH-010*
