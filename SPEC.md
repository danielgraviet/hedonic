# Hedonic Compass — Technical Specification

> A personal experience tracking system that learns what makes you thrive.

**Version:** 1.0 | **Date:** February 2026  
**Stack:** TypeScript · Telegram Bot · Supabase · Claude API · Railway

---

## Table of Contents

1. [Project Overview](#1-project-overview)
2. [System Architecture](#2-system-architecture)
3. [Database Schema](#3-database-schema)
4. [Project Structure](#4-project-structure)
5. [Claude Extraction Prompt](#5-claude-extraction-prompt)
6. [Weekly Digest](#6-weekly-digest)
7. [Environment Variables](#7-environment-variables)
8. [Deployment: Railway](#8-deployment-railway)
9. [Recommended Build Phases](#9-recommended-build-phases)
10. [Open Questions & Future Decisions](#10-open-questions--future-decisions)

---

## 1. Project Overview

Hedonic Compass is a personal experience tracking system that helps you learn what genuinely makes you thrive — and what drains you. Over time, it builds a structured picture of your hedonic profile and delivers weekly, actionable insights so you can deliberately design a life with more of what you love.

### 1.1 Core Philosophy

Most journaling tools optimize for capture. Hedonic Compass optimizes for **learning**. The goal is not to record your life — it's to understand it. Every entry is raw data that feeds a growing model of your personal flourishing.

> **Vision:** You speak a few sentences into your phone. The system silently extracts structure, stores it, and every Sunday delivers a digest that tells you exactly what to do more of — and less of — in the coming week.

### 1.2 Key Design Principles

- **Zero friction** — logging must feel as natural as texting a friend
- **Invisible structure** — the system extracts categories, intensity, and sentiment automatically; the user never fills out a form
- **Actionable output** — insights drive behavior change, not just reflection
- **Private by default** — all data belongs to the user, stored in their own Supabase instance

---

## 2. System Architecture

### 2.1 High-Level Overview

```
User speaks → Wispr Flow transcribes → Telegram message
    → Grammy bot receives text
    → Claude API extracts structure (JSON)
    → Supabase stores raw + structured entry
    → Bot sends "✓ Logged"

Every Sunday 7pm:
    → Cron fires → Query Supabase (last 7 days)
    → Claude synthesizes digest
    → Send via Telegram + Email
```

### 2.2 Technology Stack

| Layer | Technology |
|---|---|
| Bot Framework | [Grammy](https://grammy.dev/) — modern Telegram bot framework for Node.js |
| Runtime | Node.js 20 LTS + TypeScript |
| AI / Extraction | Anthropic Claude API (`claude-haiku-4-5` for extraction, `claude-opus-4-5` for digest) |
| Database | Supabase (PostgreSQL) |
| Hosting | Railway (auto-deploy from GitHub) |
| Scheduler | `node-cron` within the Railway service |
| Email | [Resend](https://resend.com) — transactional email API, generous free tier |

### 2.3 Data Flow: Logging an Entry

Every time you send a message to the bot:

1. User sends text via Telegram (typed or via Wispr Flow voice-to-text)
2. Grammy bot receives message, extracts `text` + `telegram_user_id` + `timestamp`
3. Bot calls Claude API with extraction prompt + raw text
4. Claude returns structured JSON: `sentiment`, `intensity`, `category`, `keywords`, etc.
5. Bot stores both `raw_text` AND extracted fields in Supabase
6. Bot sends a brief acknowledgment back to user (e.g. `✓ Logged`)

**Total time: ~1–2 seconds, completely invisible to the user.**

### 2.4 Data Flow: Weekly Digest

1. `node-cron` fires every Sunday at 7:00 PM (configured timezone)
2. Digest service queries Supabase for all entries from the past 7 days
3. Entries sent to Claude API with digest synthesis prompt
4. Claude returns: top themes, patterns, do-more / do-less recommendations
5. Formatted digest sent via Telegram message
6. Same digest sent via email (Resend API)

---

## 3. Database Schema

### 3.1 Table: `experiences`

The core table. Stores both the raw user text and all structured fields extracted by Claude.

| Field | Type | Description |
|---|---|---|
| `id` | `uuid` | Primary key, auto-generated |
| `created_at` | `timestamptz` | Timestamp of the entry (auto-set by Supabase) |
| `telegram_user_id` | `bigint` | Telegram user ID (enables multi-user support later) |
| `raw_text` | `text` | The exact message the user sent, unmodified |
| `sentiment` | `text` | Extracted sentiment: `positive`, `negative`, or `mixed` |
| `intensity` | `int2` | Intensity score 1–10 (1 = mild, 10 = peak experience) |
| `category` | `text` | Primary category (see categories below) |
| `subcategory` | `text` | Optional finer detail (e.g. `coding`, `running`, `meal`) |
| `time_of_day` | `text` | Inferred if mentioned: `morning`, `afternoon`, `evening`, `night` |
| `keywords` | `text[]` | Array of extracted keywords and themes |
| `context_notes` | `text` | Additional context Claude extracted (solo vs social, location, etc.) |
| `extraction_model` | `text` | Which Claude model was used (for future comparison) |

**Valid categories:** `deep_work` · `creative` · `learning` · `social` · `physical` · `emotional` · `other`

### 3.2 Table: `weekly_digests`

Stores a record of every digest sent, for historical reference.

| Field | Type | Description |
|---|---|---|
| `id` | `uuid` | Primary key |
| `created_at` | `timestamptz` | When the digest was generated |
| `week_start` | `date` | Monday of the week this digest covers |
| `week_end` | `date` | Sunday of the week this digest covers |
| `entry_count` | `int4` | Number of experiences logged this week |
| `digest_text` | `text` | The full generated digest content |
| `telegram_sent` | `bool` | Whether Telegram delivery succeeded |
| `email_sent` | `bool` | Whether email delivery succeeded |

### 3.3 Row Level Security (RLS)

> ⚠️ **Enable RLS on both tables from day one.** Even as a single user, RLS protects your data at the database level and makes adding multi-user support trivial later.

```sql
-- Enable RLS
ALTER TABLE experiences ENABLE ROW LEVEL SECURITY;
ALTER TABLE weekly_digests ENABLE ROW LEVEL SECURITY;
```

---

## 4. Project Structure

```
hedonic/
├── src/
│   ├── bot/
│   │   ├── index.ts          # Grammy bot setup + webhook registration
│   │   ├── handlers.ts       # Message handler — main entry point for each message
│   │   └── responses.ts      # Telegram reply templates
│   ├── extraction/
│   │   ├── extractor.ts      # Claude API call + extraction logic
│   │   └── prompts.ts        # All Claude prompts in one place
│   ├── digest/
│   │   ├── digest.ts         # Weekly digest generation logic
│   │   ├── scheduler.ts      # node-cron setup
│   │   └── email.ts          # Resend email delivery
│   ├── db/
│   │   ├── supabase.ts       # Supabase client initialization
│   │   └── queries.ts        # All database queries in one place
│   └── index.ts              # App entry point
├── .env.example              # Environment variable template
├── package.json
├── tsconfig.json
└── railway.json              # Railway deployment config
```

---

## 5. Claude Extraction Prompt

### 5.1 Design Goal

The extraction prompt is the most critical piece of the system. It must reliably convert a casual, natural-language entry into structured JSON — every single time, with no user intervention.

### 5.2 System Prompt (`src/extraction/prompts.ts`)

```
You are an experience extraction engine for a personal wellbeing tracker.
Your job is to extract structured data from a user's natural-language entry.

Always return ONLY valid JSON with this exact shape:
{
  "sentiment": "positive" | "negative" | "mixed",
  "intensity": number (1-10),
  "category": "deep_work" | "creative" | "learning" | "social" | "physical" | "emotional" | "other",
  "subcategory": string | null,
  "time_of_day": "morning" | "afternoon" | "evening" | "night" | null,
  "keywords": string[],
  "context_notes": string | null
}

Rules:
- intensity 8-10 = peak/transformative; 5-7 = notable; 1-4 = mild
- keywords: 2-5 short phrases capturing the essence
- context_notes: solo vs social, where, who with (only if clearly mentioned)
- If the user describes something hard BUT rewarding, score positive with high intensity
- Return ONLY the JSON object. No preamble, no explanation.
```

### 5.3 Example Extraction

**Input:**
```
Just finished a brutal but satisfying 10-mile run this morning. Legs are destroyed
but my head is completely clear — best I've felt all week.
```

**Output:**
```json
{
  "sentiment": "positive",
  "intensity": 9,
  "category": "physical",
  "subcategory": "running",
  "time_of_day": "morning",
  "keywords": ["long run", "mental clarity", "physical challenge", "satisfaction"],
  "context_notes": "solo activity, outdoor"
}
```

---

## 6. Weekly Digest

### 6.1 Digest Prompt

The digest prompt receives all entries from the past 7 days as a JSON array and returns a structured, actionable narrative.

```
You are a personal wellbeing analyst reviewing someone's week of logged experiences.
Based on the entries provided, write a weekly digest with this structure:

1. THIS WEEK AT A GLANCE (2-3 sentences, warm and direct tone)
2. YOUR PEAK MOMENTS (top 2-3 highest-intensity positive experiences)
3. PATTERNS I NOTICED (e.g. "Your best moments were all solo, in the morning")
4. DO MORE OF THIS (2-3 specific, concrete recommendations)
5. DO LESS OF THIS (1-2 honest observations, compassionate but direct)
6. QUESTION TO CARRY INTO NEXT WEEK (one thought-provoking question)

Tone: like a thoughtful friend who knows you well — honest, warm, specific.
Do not be generic. Reference the actual content of their entries.
```

### 6.2 Digest Delivery

| Channel | Details |
|---|---|
| Telegram | Sent to the same bot chat, formatted with bold headers using Telegram's MarkdownV2 |
| Email | Sent via Resend API — plain HTML template, mobile-friendly, no tracking pixels |
| Schedule | Every Sunday at 7:00 PM in the configured timezone (default: `America/Denver`) |
| Minimum entries | Only sends if ≥3 entries exist for the week. Otherwise sends an encouraging nudge. |

---

## 7. Environment Variables

All secrets are stored as Railway environment variables. Never commit these to git.

Copy `.env.example` to `.env` for local development.

| Variable | Description |
|---|---|
| `TELEGRAM_BOT_TOKEN` | From @BotFather on Telegram |
| `ANTHROPIC_API_KEY` | From [console.anthropic.com](https://console.anthropic.com) |
| `SUPABASE_URL` | Your Supabase project URL |
| `SUPABASE_SERVICE_KEY` | Service role key (not anon key — bypasses RLS for server use) |
| `RESEND_API_KEY` | From [resend.com](https://resend.com) dashboard |
| `DIGEST_EMAIL_TO` | Your email address for digest delivery |
| `DIGEST_EMAIL_FROM` | Verified sender address in Resend |
| `TELEGRAM_USER_ID` | Your Telegram user ID (restricts bot to owner only) |
| `DIGEST_TIMEZONE` | e.g. `America/Denver` |
| `WEBHOOK_SECRET` | Random string for Telegram webhook verification |

---

## 8. Deployment: Railway

### 8.1 Setup Steps

1. Push project to a GitHub repository
2. Create new Railway project → **Deploy from GitHub repo**
3. Add all environment variables in the Railway dashboard
4. Railway auto-detects Node.js and runs `npm run build && npm start`
5. Railway provides a public HTTPS URL — use this to register your Telegram webhook

### 8.2 `railway.json`

```json
{
  "$schema": "https://railway.app/railway.schema.json",
  "build": { "builder": "NIXPACKS" },
  "deploy": {
    "startCommand": "node dist/index.js",
    "restartPolicyType": "ON_FAILURE",
    "restartPolicyMaxRetries": 3
  }
}
```

> 💡 Railway's Hobby plan ($5/month) is more than sufficient. The bot is webhook-driven (not polling), so it uses almost no resources between messages.

---

## 9. Recommended Build Phases

| Phase | Goal |
|---|---|
| **Phase 1 — Core Loop** | Telegram bot receives message → Claude extracts → stores to Supabase → sends `✓ Logged` |
| **Phase 2 — Digest** | Sunday cron generates and sends digest via Telegram + email |
| **Phase 3 — Polish** | Better ack messages, error handling, retry logic, timezone config |
| **Phase 4 — Insights** | Query commands (`/week`, `/month`), entry history, category breakdowns |
| **Phase 5 — Optional** | pgvector semantic search, multi-user support, web dashboard |

> **Recommendation:** Start with Phase 1 only. Get the logging loop working perfectly before building the digest. The core value of this system is in data quality — a clean extraction pipeline is worth more than ten features.

---

## 10. Open Questions & Future Decisions

Intentionally deferred to keep Phase 1 simple. Revisit after 30 days of real use.

- **Timezone handling** — hardcode `America/Denver` for now; make configurable in Phase 3
- **Multi-user support** — the schema supports it (`telegram_user_id` field exists), but Phase 1 restricts to owner only via `TELEGRAM_USER_ID` env var
- **Entry correction** — define a `/delete last` command in Phase 3 for when you log something by mistake
- **Category taxonomy** — the 7 categories are a starting point; review after 30 days and adjust based on your actual entries
- **Semantic search** — Supabase's `pgvector` extension can power "find entries similar to this feeling" — consider in Phase 5

---

*— End of Specification —*
