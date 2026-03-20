# Argus — WhatsApp Proactive Memory Assistant

> AI-powered proactive memory assistant that learns from your WhatsApp conversations and reminds you about relevant events while browsing — built with Gemini 3 Flash Preview.

[![Version](https://img.shields.io/badge/version-2.7.1-blue.svg)](argus/CHANGELOG.md)
[![Node](https://img.shields.io/badge/node-22%2B-brightgreen.svg)](https://nodejs.org)
[![TypeScript](https://img.shields.io/badge/TypeScript-5.7-blue.svg)](https://www.typescriptlang.org/)
[![Gemini](https://img.shields.io/badge/Gemini_3-Flash_Preview-4285F4.svg)](https://ai.google.dev/)
[![Docker](https://img.shields.io/badge/Docker-4_containers-2496ED.svg)](https://docker.com)
[![License](https://img.shields.io/badge/license-Private-red.svg)](LICENSE)

---

## 🎯 What is Argus?

Argus monitors your WhatsApp conversations via [Evolution API](https://github.com/EvolutionAPI/evolution-api) webhooks, extracts events and intent using Google Gemini AI, and delivers real-time popup overlays in your Chrome browser — at the right time and on the right page.

### How It Works

```
WhatsApp Message
      │
      ▼
Evolution API (webhook: messages.upsert)
      │
      ▼
Argus Server (Express + WebSocket)
  ├── Gemini AI extracts events / detects actions
  ├── SQLite + FTS5 stores & indexes events
  ├── QuickSave compresses context (~40-55% fewer tokens)
  └── Scheduler manages time-based reminders
      │
      ▼
Chrome Extension (Manifest V3)
  ├── WebSocket receives real-time events
  ├── Content script renders popup overlays
  ├── URL watcher triggers context reminders
  └── DOM watcher detects form mismatches
```

### 5 Demo Scenarios

| # | Scenario | What Happens |
|---|----------|-------------|
| 1 | **Goa Cashew** | Friend texts "try cashews at Zantye's in Goa" → later you browse a Goa travel site → Argus popup: "Rahul recommended cashews at Zantye's" |
| 2 | **Insurance Accuracy** | You type "Honda Civic 2022" on ACKO → but your WhatsApp says you own a 2018 model → popup: "You might be overpaying!" + ✏️ Fix It button |
| 3 | **Gift Intent** | Chat says "need makeup for sis birthday" → you visit Nykaa → popup: "Your sister's birthday gift — makeup is on sale!" |
| 4 | **Netflix Subscription** | You said "cancel Netflix after this show" → you visit netflix.com → popup: "You planned to cancel this subscription" |
| 5 | **Calendar Conflict** | You told dinner group "see you Thursday" → then schedule a meeting Thursday → popup: "This conflicts with your dinner plan" |

---

## Quick Start

### Docker (Recommended — works on Linux / Windows / macOS)

```bash
git clone https://github.com/Akshat74747/argus-whatsapp-assistant
cd whatsapp-chat-rmd-argus/argus
cp .env.example .env          # Fill in GEMINI_MODEL_KEY + Evolution API credentials
docker compose up -d           # Starts 4 containers (builds everything from source)
docker compose logs -f argus   # View Argus logs
```

> **Everything is included** — Evolution API source, QuickSave, and Argus are all in this repo. No extra downloads needed.

### Local Development

Pre-Requisites

```bash
cd argus
npm install
cp .env.example .env           # Fill in GEMINI_MODEL_KEY + Evolution API credentials
```
Start the evolution api first, and then creeat an instance at http://localhost:8080/manager

```bash
cd evolution-api
npm run dev:server
```
Once evolution api starts up, start argus

```bash
cd argus
npm run dev
```
Open chrome extensions manager and load unpacked the extension folder at argus/extension

| Container | Port | Purpose |
|-----------|------|---------|
| `argus-server` | 3000 | Express + WebSocket + Gemini AI + SQLite |
| `argus-evolution` | 8080 | Evolution API — WhatsApp Web bridge |
| `argus-postgres` | 5432 | PostgreSQL — Evolution API database |
| `argus-redis` | 6379 | Redis — Evolution API cache |

### 3. Load Chrome Extension

1. Go to `chrome://extensions/`
2. Enable **Developer mode** (top right)
3. Click **Load unpacked** → select `argus/extension/`
4. Pin the Argus extension to your toolbar

### 4. Connect WhatsApp

Argus auto-creates and configures the Evolution API instance on startup. Just scan the QR code:

```bash
# Open Evolution API manager
open http://localhost:8080/manager

# Or check Argus logs for connection status
docker compose logs -f argus | grep -i evolution
```

The instance name defaults to `arguas` (configurable via `EVOLUTION_INSTANCE_NAME` in `.env`).

### Local Development (without Docker)

```bash
# You still need postgres + redis running for Evolution API
# Only Argus itself runs locally

cd argus
npm install
cp .env.example .env    # Set GEMINI_API_KEY + EVOLUTION_* vars
npm run dev             # Starts with tsx hot reload on port 3000
```

> ⚠️ **NEVER restart the server manually** — it auto-restarts on rebuild. Only run `npx tsc` to compile.

---

## 📁 Project Structure

```
whatsapp-chat-rmd-argus/
├── argus/                          # Main application
│   ├── src/
│   │   ├── server.ts               # Express + WebSocket server, all API routes
│   │   ├── db.ts                   # SQLite + FTS5 — events, messages, contacts, triggers
│   │   ├── evolution-db.ts         # PostgreSQL direct read — Evolution API messages
│   │   ├── gemini.ts               # Gemini AI — extraction, action detection, popup blueprints, chat
│   │   ├── quicksave.ts            # QuickSave CEP v9.1 — S2A filter + dense format compression
│   │   ├── ingestion.ts            # Webhook → action detection → event extraction → triggers
│   │   ├── matcher.ts              # URL keyword extraction + FTS5 search + Gemini validation
│   │   ├── scheduler.ts            # Time-based reminders (24h, 1h, 15min) + snooze
│   │   └── types.ts                # Zod schemas — Message, Event, Webhook, Config, PopupType
│   ├── extension/                  # Chrome Extension (Manifest V3)
│   │   ├── manifest.json           # Permissions: tabs, scripting, sidePanel, <all_urls>
│   │   ├── background.js           # Service worker — WebSocket client, tab routing, context check
│   │   ├── content.js              # Injected overlay — 8 popup types, toasts, DOM form watcher
│   │   ├── styles.css              # Popup/modal CSS
│   │   ├── sidepanel.html/js       # AI Chat sidebar with markdown rendering
│   │   ├── popup.html/js/css       # Extension popup — event cards + stats
│   │   └── icons/                  # Extension icons (16/32/48/128px)
│   ├── tests/                      # Vitest — db.test.ts, ingestion.test.ts, matcher.test.ts
│   ├── data/                       # SQLite database (events.db, auto-created)
│   ├── docker-compose.yml          # 4-container stack
│   ├── Dockerfile                  # Multi-stage Node 22 Alpine build
│   ├── .env.example                # All config with defaults & comments
│   ├── tsconfig.json
│   ├── CHANGELOG.md                # Full version history
│   └── package.json                # ESM, Node 22+, Express 5
├── evolution-api/                  # WhatsApp API (forked, built from source)
│   ├── Dockerfile                  # Multi-stage Node 24 Alpine
│   └── src/                        # Evolution API source (Baileys-based)
├── quicksave/                      # QuickSave CEP v9.1 reference (read-only)
│   ├── SKILL.md                    # Full protocol specification
│   └── references/                 # PDL, S2A, NCL, KANJI docs
├── Insurance website/              # Demo ACKO clone for insurance mismatch scenario
├── aidata/                         # Project context docs (read-only)
├── RULES.md                        # Development rules & constraints
├── INFO.md                         # Architecture documentation
└── README.md                       # This file
```

---

## SQLite Database

Argus uses **SQLite** (`better-sqlite3`, synchronous API) as its sole database. All tables and FTS5 virtual tables are created automatically on startup.

### Tables

| Table | Purpose |
|-------|---------|
| `events` | Events/tasks/reminders extracted from WhatsApp |
| `messages` | Raw WhatsApp messages (source of truth) |
| `triggers` | Time and URL-based notification triggers |
| `contacts` | Contact list with message counts |
| `context_dismissals` | Per-URL dismissal suppression (30-minute window) |
| `push_subscriptions` | Browser push subscription tokens |
| `events_fts` | FTS5 virtual table over events (title, keywords, description) |

### FTS5 Search

`/api/context-check` and `/api/internal/search` use `ftsSearchEvents()` which runs a SQLite FTS5 `MATCH` query across `title`, `keywords`, and `description`. Results are ranked by BM25 relevance.

---

## ✨ Features

### 🎨 8 Popup Types

All popups are generated server-side by Gemini AI — the extension just renders whatever the server sends.

| Type | Icon | Trigger |
|------|------|---------|
| `event_discovery` | 📅 | New event extracted from WhatsApp message |
| `event_reminder` | ⏰ | Scheduled time arrives (24h / 1h / 15min before) |
| `context_reminder` | 🎯 | User visits a URL matching an event's context |
| `conflict_warning` | 🗓️ | Two events overlap in time (±60 min window) |
| `insight_card` | 💡 | AI suggestion from conversation patterns |
| `snooze_reminder` | 💤 | Snoozed event fires again after delay |
| `update_confirm` | 📝 | WhatsApp message modifies an existing event — needs approval |
| `form_mismatch` | ⚠️ | DOM form field contradicts WhatsApp memory (insurance scenario) |

### 🔄 Event Lifecycle

```
    ┌── snoozed ──┐
    │             │
discovered → scheduled → reminded → completed
    │                        │
    └── ignored          expired
```

- **discovered** — new event, waiting for user action
- **scheduled** — user approved, reminders set at 24h / 1h / 15min
- **snoozed** — postponed for N minutes
- **reminded** — reminder was shown
- **completed** — user marked done
- **ignored** — hidden, won't remind again
- **expired** — event time has passed
- **dismissed** — notification dismissed (can reappear on context trigger)

### 🎯 Action Detection

When a WhatsApp message references an existing event, Gemini detects the action:

| Action | Example Message | What Happens |
|--------|----------------|--------------|
| `cancel` / `delete` | "cancel the dinner plan" | Event deleted |
| `complete` | "done with the cashews order" | Event marked complete |
| `ignore` | "skip the meeting" | Event hidden |
| `snooze` / `postpone` | "push the meeting to next week" | Event snoozed |
| `modify` | "change dinner to Friday 9pm" | Confirmation popup shown |

### 🔍 Context-Aware Triggers

| Category | How It Works |
|----------|-------------|
| **Subscriptions** | "cancel netflix" → `context_url=netflix` → popup on netflix.com |
| **Travel** | "cashews at Zantye's in Goa" → `context_url=goa` → popup on Goa travel sites |
| **Shopping/Gifts** | Beauty → nykaa, Fashion → myntra, General → amazon URL triggers |
| **Insurance** | DOM form watcher parses car make/model/year, cross-references with chat memory |
| **Calendar** | Time conflicts detected within ±60 min window |

### 🧠 Smart Event Extraction (Gemini)

- Single Gemini call per message — classifies + extracts in one shot
- Handles Hinglish (Hindi + English), typos, informal chat
- Aggressive spam filter: price mentions, forwarded deals, brand accounts → low confidence
- Date resolution: relative dates ("kal", "Thursday", "next week") → absolute timestamps
- Event CRUD: Gemini detects if message creates, updates, or merges with existing events
- Context window: last 5 messages from same chat included for conversation continuity

### 📦 QuickSave Context Compression (CEP v9.1)

All Gemini prompts use [QuickSave](https://github.com/ktg-one/quicksave)-inspired compression:

- **S2A Filter** — ranks events by signal (time proximity, status, recency) → top 60 sent
- **Dense Format** — `#ID|TYPE|STATUS|"Title"|time|loc|sender|keywords` (~40-55% fewer tokens)
- **L2 Edge Detection** — cross-event relationships (cancel↔subscription, time conflicts, topic overlap)
- **Chat Memory** — older sidebar turns compressed into key facts, recent 6 turns stay raw
- Same token budget carries **~2x more event information**

### 🏥 DOM Form Watcher (Insurance Accuracy)

- Detects insurance-like pages (ACKO, PolicyBazaar, Digit, etc.)
- `MutationObserver` watches for dynamically added form inputs
- `input` event listeners with 1.5s debounce on all text fields
- Regex parser extracts car make/model/year from input values
- Calls `/api/form-check` → cross-references with WhatsApp chat memory
- "✏️ Fix It" button auto-fills the correct value + green highlight

---

## 📡 API Endpoints

| Endpoint | Method | Purpose |
|----------|--------|---------|
| `/api/health` | GET | Health check (DB, Evolution API, model info) |
| `/api/stats` | GET | Message/event/contact statistics |
| `/api/events` | GET | List events (filter by `?status=discovered`) |
| `/api/events/:id` | PATCH | Update event fields (title, time, location, etc.) |
| `/api/events/:id` | DELETE | Delete event permanently |
| `/api/events/:id/set-reminder` | POST | Schedule event (discovered → scheduled) |
| `/api/events/:id/snooze` | POST | Snooze event for N minutes |
| `/api/events/:id/ignore` | POST | Ignore event (hide, won't remind) |
| `/api/events/:id/complete` | POST | Mark event as done |
| `/api/events/:id/dismiss` | POST | Dismiss notification (can reappear) |
| `/api/events/:id/acknowledge` | POST | Acknowledge a reminder |
| `/api/events/:id/confirm-update` | POST | Confirm a pending modify action |
| `/api/events/day/:timestamp` | GET | Get all events for a specific day |
| `/api/webhook/whatsapp` | POST | Evolution API webhook receiver |
| `/api/context-check` | POST | Check URL for matching events |
| `/api/extract-context` | POST | Extract keywords from URL |
| `/api/form-check` | POST | Check form field vs WhatsApp memory |
| `/api/chat` | POST | AI Chat — context-aware conversation |
| `/ws` | WS | Real-time event notifications |

### Webhook Payload (Evolution API → Argus)

```json
{
  "event": "messages.upsert",
  "instance": "arguas",
  "data": {
    "key": {
      "remoteJid": "919876543210@s.whatsapp.net",
      "fromMe": false,
      "id": "3EB0ABC123..."
    },
    "pushName": "Rahul",
    "message": {
      "conversation": "Let's meet tomorrow at 5pm at Starbucks"
    },
    "messageTimestamp": 1739097600
  }
}
```

Only `messages.upsert` events are processed. All other events (`messages.update`, `connection.update`, etc.) are acknowledged but skipped.

---

## 🏗️ Tech Stack

| Layer | Technology | Version | Purpose |
|-------|-----------|---------|---------|
| Runtime | Node.js | 22.x | ESM JavaScript runtime |
| Language | TypeScript | 5.7.x | Type-safe development |
| Web Server | Express.js | 5.x | HTTP + WebSocket server |
| Database | SQLite (better-sqlite3) | 11.x | Event/message storage + FTS5 search |
| AI | Gemini 3 Flash Preview | Latest | Event extraction, popups, chat |
| WhatsApp | Evolution API | v2.3.7 | WhatsApp Web bridge (Baileys) |
| Evolution DB | PostgreSQL | 16 | Evolution API storage (direct read) |
| Cache | Redis | 7 | Evolution API cache |
| Validation | Zod | 3.24.x | Runtime schema validation |
| Real-time | ws | 8.x | WebSocket server |
| Browser | Chrome Extension | Manifest V3 | Popups, URL detection, form watching |
| Compression | QuickSave CEP | v9.1 | S2A + dense format for Gemini prompts |
| Testing | Vitest | 2.x | Fast unit tests (<3s) |
| Containers | Docker Compose | — | 4-service stack |

### What We're NOT Using

| ❌ | Why |
|----|-----|
| FAISS / vector stores | FTS5 + Gemini validation is sufficient (90%+ accuracy) |
| OpenAI / embeddings | Gemini handles everything — extraction, validation, chat |
| RAG pipelines | Two-step FTS5 → Gemini replaces traditional RAG |
| Multi-stage LLM calls | Single Gemini call per message (classify + extract) |

---

## 🐳 Docker Architecture

```
┌──────────────────────────────────────────────────────────────┐
│                    Docker Compose Network                      │
│                                                                │
│  ┌──────────────┐    ┌──────────────────┐                     │
│  │   postgres    │    │      redis       │                     │
│  │  :5432        │    │     :6379        │                     │
│  └──────┬───────┘    └────────┬─────────┘                     │
│         │                     │                                │
│         ▼                     ▼                                │
│  ┌──────────────────────────────────────┐                     │
│  │      evolution-api :8080             │ ◄── WhatsApp QR     │
│  │   WhatsApp Bridge (Baileys/Node 24)  │                     │
│  └──────────────┬───────────────────────┘                     │
│                 │ webhook POST + direct PG read                │
│                 ▼                                              │
│  ┌──────────────────────────────────────┐                     │
│  │         argus :3000                  │ ◄── Chrome Ext (WS) │
│  │   Express + WebSocket + Gemini AI    │                     │
│  │   SQLite + FTS5 (internal volume)    │                     │
│  └──────────────────────────────────────┘                     │
│                                                                │
└──────────────────────────────────────────────────────────────┘
```

### Docker Commands

```bash
cd argus

docker compose up -d --build        # Build & start all 4 containers
docker compose ps                   # Check status
docker compose logs -f argus        # Argus logs
docker compose logs -f evolution-api # Evolution logs
docker compose down                 # Stop
docker compose down -v              # Stop + delete all data
docker compose build argus          # Rebuild Argus only
docker compose up -d argus          # Restart Argus only
```

### Environment Variables

All config is in `.env` (copy from `.env.example`). Only `GEMINI_API_KEY` is required — everything else has sensible defaults.

| Variable | Default | Required | Description |
|----------|---------|----------|-------------|
| `GEMINI_API_KEY` | — | ✅ | Google AI Studio API key |
| `GEMINI_MODEL` | `gemini-3-flash-preview` | — | Gemini model ID |
| `EVOLUTION_API_KEY` | `rmd_evolution_api_key_12345` | — | Evolution API auth key |
| `EVOLUTION_INSTANCE_NAME` | `arguas` | — | WhatsApp instance name |
| `HOT_WINDOW_DAYS` | `90` | — | Context matching window (days) |
| `PROCESS_OWN_MESSAGES` | `true` | — | Process your own sent messages |
| `SKIP_GROUP_MESSAGES` | `false` | — | Skip group chat messages |
| `POSTGRES_PASSWORD` | `postgres` | — | PostgreSQL password |
| `TIMEZONE` | `Asia/Kolkata` | — | Server timezone |

---

## 📊 Performance

| Metric | Target | Notes |
|--------|--------|-------|
| Message ingestion | <500ms | Single Gemini call (classify + extract) |
| Context check | <800ms | FTS5 query <10ms + Gemini validation ~800ms |
| Database query | <10ms | SQLite FTS5 on 50k+ messages |
| Memory usage | <200MB | SQLite + Node runtime per container |
| WebSocket latency | <50ms | Event → browser overlay |
| Form mismatch check | <100ms | Regex parse + SQLite keyword search |
| Gemini cost/message | ~$0.0001 | Flash Preview pricing |
| Gemini cost/context check | ~$0.0003 | 10 candidates validated |
| QuickSave compression | ~2x density | 40-55% fewer tokens per prompt |

---

## 🧪 Development

```bash
cd argus

npm run dev          # Start with tsx hot reload
npm test             # Fast tests (~2s, Vitest)
npm run build        # Compile TypeScript (npx tsc)
npm run typecheck    # Type check only (no emit)
npm run lint         # ESLint with cache
npm run lint:fix     # Auto-fix lint issues
npm run format       # Prettier formatting
npm run db:reset     # Delete SQLite DB + restart
```

### Key Rules

- **NEVER restart the server** — it auto-restarts on rebuild. Only run `npx tsc`.
- **ALWAYS update CHANGELOG.md** before committing (append at top).
- **DO NOT use OpenAI** — Gemini only (via OpenAI-compatible endpoint).
- **DO NOT edit `aidata/*` or `quicksave/*`** — reference-only files.
- SQLite + FTS5 only — no vectors, no FAISS, no embeddings.

---

## 🙏 Acknowledgments

- [Evolution API](https://github.com/EvolutionAPI/evolution-api) — WhatsApp Web integration
- [Google Gemini](https://ai.google.dev/) — AI extraction, popups, chat
- [SQLite FTS5](https://www.sqlite.org/fts5.html) — Full-text search engine
- [QuickSave CEP v9.1](https://github.com/ktg-one/quicksave) — Context compression protocol by Kevin Tan (ktg.one)
- Chrome Extension Manifest V3 — Browser integration

---
