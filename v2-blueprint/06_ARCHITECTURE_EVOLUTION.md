# 06 — Architecture Evolution

> How QuantBit v1's architecture evolved from prototype to production, and the architectural decisions along the way.

---

## Phase 1: Prototype (Pre-Git History)

**Goal:** Build a working backtest engine + basic UI

```
[Single React app]
  ├── Engine (core.ts, ranker.ts) — pure TS math
  ├── Static JSON data (data/years/*.json)
  └── Simple tab UI (Market, Portfolio, Backtest, Analytics)
```

**Key decisions made here:**
- 4-tab layout (Market / Portfolio / Backtest / Analytics)
- Terminal aesthetic
- Keyboard shortcuts (1/2/3/4)
- Engine is pure TS, no external dependencies

---

## Phase 2: Data Pipeline (Sesi 1-5)

**Goal:** Automate market data collection

```
[Data Pipeline] → [JSON files] → [Engine]
  Yahoo Finance API
  fetch_historical_data.ts (yahoo-finance2)
```

**Key decisions:**
- Yahoo Finance API as primary data source
- JSON files for storage (simple, no DB needed)
- `fetch_historical_data.ts` — one-time historical download
- `stockNormScores` pre-computed and stored in JSON

**Technical debt introduced:**
- Large JSON files (40MB+ each)
- Static data bundled with app (Vite import)

---

## Phase 3: Local SQLite (Sesi 5-7)

**Goal:** Enable local development without full data pipeline

```
[Data Pipeline] → [JSON] → [SQLite] → [Local Express Server]
```

**Key decisions:**
- `better-sqlite3` for local DB
- `server.ts` Express server emulating production endpoints
- SQLite mirrors D1 schema (mostly)

**Technical debt introduced:**
- Triple data storage (JSON + SQLite + eventually D1)
- SQLite schema diverges from JSON format

---

## Phase 4: Cloudflare D1 + Auth (Sesi 7-10)

**Goal:** Production deployment with user accounts

```
[Data Pipeline] → [JSON] → [SQLite] → [D1]
                               ↘ Production
  ┌──────────────────────────────────────────────┐
  │  Cloudflare Pages Functions                  │
  │  [[path]].ts — monolithic API handler        │
  │  D1 — production DB                          │
  │  Custom auth (PBKDF2, session tokens)        │
  └──────────────────────────────────────────────┘
```

**Key decisions:**
- Cloudflare Pages Functions for serverless API
- D1 for production database
- Custom auth (no OAuth provider)
- `seed-d1.py` to sync SQLite → D1

**Technical debt introduced:**
- Monolithic `[[path]].ts` — all routes in one file
- Dev-session bypass for local testing
- JSON blobs in D1 tables

---

## Phase 5: Add AI (Sesi 10-13)

**Goal:** AI assistant for portfolio

```
[React UI] → [FloatingAIChat] → [askAI()] → [[path]].ts → AI Provider
```

**Key decisions:**
- 4-level AI architecture (Q&A→Read→Action→Proactive)
- 5 providers with circuit breaker (OpenRouter primary)
- `systemKnowledge.ts` — 14-section system prompt
- Jaksel persona ("Rico Lubis")
- Regex-extracted tool calls

**Technical debt introduced:**
- 7 provider integrations with complex fallback logic
- Regex parsing of AI output
- Hardcoded persona
- Memory management in prompt (10K char limit)

---

## Phase 6: Backtest UI + Dashboard (Sesi 12-14)

**Goal:** Full backtest integration in UI

```
[Backtest results] → [SYNC TO PORTFOLIO] → [Live portfolio]
```

**Key decisions:**
- `syncFromBacktest()` promotes backtest results to portfolio
- Live/Draft toggle for backtest config
- Single Source of Truth via EngineConfigContext
- BPS dashboard, ForwardDividends, StockDrawer

**Technical debt introduced:**
- EngineConfigContext in localStorage (not synced to DB)
- Backtest reads JSON, portfolio reads API — different data
- Complex notification system with 4 rules (1 stub)

---

## Phase 7: Optimization + Crash Fixes (Sesi 14-16)

**Goal:** Fix crashes, optimize backtest, improve UI

**Key decisions:**
- Cumulative gradient fix for backtest (chart smoothing)
- Improved crash detection logic
- Regime engine improvements
- Landing page deployment

**Technical debt introduced:**
- Multiple quick fixes without refactoring
- Growing complexity of `[[path]].ts`
- No testing — changes validated manually

---

## Architecture Decision Log (V1)

| Decision | Phase | Impact | V2 Recommendation |
|----------|-------|--------|-------------------|
| JSON as primary storage | 2 | Triple data sources | D1 only |
| Monolithic API | 4 | Cannot extend | Hono domain modules |
| Custom auth | 4 | Security bypasses | Better Auth |
| Regex AI parsing | 5 | Fragile | Structured output |
| localStorage config | 6 | Dual sources | D1 + cache |
| 252 trading days | 2 | Wrong for IDX | 247 days |
| 5% risk-free rate | 2 | Inaccurate | BI rate / SBN yield |
| Yahoo Finance API | 2 | Undocumented, fragile | Multiple sources + fallback |
| Python data pipeline | 4 | Two languages | Unified TS |
| D1 ecosystem lock-in | 4 | Migration cost | Accept for V2 (edge-native) |

---

## Architecture Principles for V2

Derived from every mistake and success in v1's evolution:

1. **Data first, code second** — Design the data model before writing business logic
2. **One source of truth** — Every data point lives in exactly one place
3. **Validate at boundaries** — Every API input, every data write, validated with Zod
4. **Engine is pure** — No AI in financial calculations. AI is a UI for recommendations.
5. **Edge-native** — Cloudflare ecosystem (Hono + D1 + KV) from day one
6. **Testable by design** — Pure functions, dependency injection, no singletons
7. **Configuration over hardcode** — Every magic number is a config value
8. **Documentation as code** — ADRs, business rules, and architecture blueprints in the repo
