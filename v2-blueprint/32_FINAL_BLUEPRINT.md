# 32 — Final Blueprint

> Complete blueprint for building QuantBit V2 from scratch.

---

## The Mission

Build a terminal-styled web application for Indonesian stock (IDX) portfolio management that combines **deterministic financial scoring** (BPS, factor ranking, crash detection, market regime) with **AI-assisted decision support** (Q&A, read tools, action approval).

**Key constraint:** NO AI in financial calculations. The engine is pure TypeScript math. AI is only for Q&A and decision support.

---

## The Knowledge Base

We analyzed QuantBit v1 — a 6-month solo project — and extracted everything that matters:

| What | Count | Where |
|------|-------|-------|
| Features catalogued | 39 | `07_FEATURE_MATRIX.md` |
| Business rules documented | 13 categories | `03_BUSINESS_RULES.md` |
| Hidden knowledge items | 50 | `04_RESEARCH_FINDINGS.md` |
| Technical debts identified | 30 (5 P0) | `08_TECHNICAL_DEBT.md` |
| Security findings | 18 (3 P0) | `17_SECURITY.md` |
| Data duplication categories | 10 | `12_DATA_ARCHITECTURE.md` |
| Design decisions analyzed | 20 | `09_DESIGN_DECISIONS.md` |
| ADRs extracted | 12 | `10_ADR_EXTRACT.md` |
| Open questions | 20 | `31_OPEN_QUESTIONS.md` |

---

## Architecture (3-Sentence Summary)

**Frontend:** React 19 + Vite 6 + Tailwind CSS 4 + Recharts — terminal aesthetic, 4-tab layout (Market, Portfolio, Backtest, Analytics), keyboard shortcuts.

**Backend:** Hono on Cloudflare Pages Functions — domain-modular routes (`/api/v1/auth/*`, `/market/*`, `/portfolio/*`, `/engine/*`, `/ai/*`), Zod validation on every endpoint, Better Auth.

**Data:** D1 as single source of truth — no JSON files, no SQLite, no in-memory singletons. Normalized schema (19 tables, no JSON blobs, FKs everywhere). Workers KV for cache.

---

## The Engine (What Makes QuantBit Unique)

These 5 algorithms are the core intellectual property. Preserve them exactly:

### 1. Buy Pressure Score (BPS)
`val*0.30 + mom*0.25 + breadth*0.15 + dd*0.20 + fear*0.10`
5-level action mapping: <30 none → >90 deploy. Crisis override during crashes.

### 2. Weighted Factor Ranking
`Q*Wq + G*Wg + V*Wv + M*Wm + D*Wd` → Sort descending → rank 1..N
3 profiles: AMAN (conservative), AGRESIF (aggressive), DIVIDEN (income)

### 3. Crash Detection (2-Tier)
- Fast: IHSG 60d drawdown ≤ -sensitivity (default -10%)
- Slow grind: Price < deflated SMA50 + SMA20 < deflated SMA50
- Recovery: Price > SMA20 + 5d return ≥ 2.5%

### 4. Market Regime (5-State Priority Tree)
GOLD_DEFENSE > CASH_DEFENSE > RECOVERY_WATCH > RISK_OFF > RISK_ON
Breadth analysis: % of stocks above score 60

### 5. Rebalancing
Emergency exit (rank ≥ 15) anytime. Routine exit (rank ≥ 10) month-end.
Swap from Top N candidates. Gold as safe haven during crashes.

---

## What Changes from V1

| V1 Pattern | V2 Pattern | Why |
|-----------|-----------|-----|
| Monolithic API | Hono domain modules | Testability, maintainability |
| JSON + SQLite + D1 | D1 only | Single source of truth |
| Client-side backtest | Server-side backtest | Consistent data, persistent results |
| Regex AI parsing | Structured JSON output | Reliability |
| Custom PBKDF2 auth | Better Auth | Security, features |
| In-memory singletons | API-backed data | Consistency |
| Hardcoded config | Database-stored config | Change without deploy |
| No tests | Comprehensive tests | Regression prevention |
| No validation | Zod everywhere | Type safety, error handling |
| No monitoring | Health checks + alerts | Operational visibility |

---

## What Stays the Same

| V1 Feature | V2 Status | Reason |
|-----------|-----------|--------|
| Terminal aesthetic | ✅ Keep | Brand identity |
| 4-tab layout | ✅ Keep | Proven workflow |
| Keyboard shortcuts | ✅ Keep | Power user essential |
| BPS algorithm | ✅ Keep exactly | Genuine innovation |
| Ranking algorithm | ✅ Keep exactly | Sound factor approach |
| Crash detection | ✅ Keep exactly | 2-tier is robust |
| Market regime engine | ✅ Keep exactly | Priority tree works |
| Fee structure | ✅ Keep | Empirically derived |
| Equal-weight allocation | ✅ Keep | Simple, transparent |
| DataStatus badges | ✅ Keep | Builds user trust |

---

## What's Deferred (Post-MVP)

| Feature | Rationale |
|---------|-----------|
| MCP Server | Experimental, not integrated with UI |
| Proactive AI Agent (L4) | Not validated, complex state management |
| Adaptive Weights | Not proven effective |
| Strategy Comparison | Nice-to-have, not core |
| WebSocket Real-time | Stub in v1, polling sufficient for MVP |
| Multiple AI Providers | Start with 1, add more as needed |
| Email Notifications | Only 1 feature depends on this |

---

## Rebuild Plan (33 Days)

| Sprint | Days | Deliverable |
|--------|------|-------------|
| **Sprint 0**: Foundation | 2 | Empty project with CI/CD, folder structure, tools |
| **Sprint 1**: Database | 3 | D1 schema, migrations, seed scripts |
| **Sprint 2**: Engine | 5 | All engine modules with unit tests (90%+ coverage) |
| **Sprint 3**: API | 3 | Hono routes, auth, validation, middleware |
| **Sprint 4**: UI | 5 | React app with 4 main tabs, responsive |
| **Sprint 5**: Backtest UI | 3 | Server-side backtest, chart, results, sync |
| **Sprint 6**: AI | 4 | AI chat with structured output |
| **Sprint 7**: Pipeline | 3 | Data sync cron, scores computation |
| **Sprint 8**: Polish | 3 | Edge cases, performance, documentation, audit |
| **Sprint 9**: Launch | 2 | Production deploy, monitoring, backup |
| **Total** | **33** | Complete QuantBit V2 |

---

## Tech Stack

```
Frontend:     React 19 + Vite 6 + Tailwind CSS 4 + TypeScript strict
Backend:      Hono + Cloudflare Pages Functions
Database:     D1 (Cloudflare SQLite) — single source of truth
Cache:        Workers KV
Auth:         Better Auth
Validation:   Zod
AI:           OpenRouter (JSON mode / structured output)
Charts:       Recharts 3
Animation:    motion (framer-motion v12)
Icons:        lucide-react
CI/CD:        GitHub Actions
Hosting:      Cloudflare Pages (free tier)
```

---

## Database (19 Tables)

```
tickers              — Ticker catalog (FK target for all ticker references)
user_profiles        — User display preferences
pipeline_runs        — Pipeline execution log

market_daily         — Daily IHSG, gold, USD/IDR
stock_daily          — Per-ticker daily OHLCV
stock_scores         — Per-ticker norm scores per date

idx80_scans          — IDX80 live snapshot (normalized)

portfolios           — User stock holdings
cash_holdings        — User cash + gold
watchlists           — User watchlist
trade_logs           — Real user trades
backtest_sessions    — Backtest runs
backtest_logs        — Backtest per-run logs

strategy_profiles    — Weight profiles (AMAN/AGRESIF/DIVIDEN/custom)
user_strategy_configs — Per-user strategy settings

ai_sessions          — AI chat sessions
ai_messages          — AI chat messages

notification_rules   — User notification rules
user_notifications   — Fired notifications
```

---

## API (26 Endpoints)

```
Auth:       register, login, logout, me
Market:     overview, stocks, tickers, regime, fundamentals, backtest-data
Portfolio:  get, trade, watchlist CRUD, history
Engine:     bps, run-backtest, backtest-by-id, backtests-list, config CRUD
AI:         chat, sessions list, session detail
Admin:      health, tickers CRUD, force-sync
```

---

## UI (4 Main Tabs + 1 Admin Tab)

| Tab | Key | Content |
|-----|-----|---------|
| Market | 1 | Overview charts, stock list, search, StockDrawer |
| Portfolio | 2 | Holdings, BPS dashboard, wallet, chart, dividends |
| Backtest | 3 | Config panel (Live/Draft), chart, results, log, sync |
| Analytics | 4 | Leaders, regime history, factor analysis, notifications |
| Admin | 5 | Ticker manager, pipeline monitor, health |

---

## Validation Criteria (Go for Launch)

- [ ] Engine passes golden file tests (CAGR, Sharpe within 0.5% of v1 verified output)
- [ ] All 26 API endpoints return correct responses with proper auth
- [ ] Data pipeline completes successfully with real Yahoo Finance data
- [ ] AI chat produces structured JSON output (no regex fallback in production)
- [ ] Auth: register, login, logout, session expiry all working
- [ ] Portfolio: buy, sell, view holdings, watchlist CRUD all working
- [ ] Backtest: configure, run, view results, sync to portfolio all working
- [ ] No P0/P1 security vulnerabilities (see `17_SECURITY.md`)
- [ ] DataStatus badges shown on all market data
- [ ] Responsive on desktop + mobile

---

## Final Words

QuantBit v1 was a successful proof-of-concept that proved 5 things:
1. The BPS algorithm genuinely measures market entry timing
2. Factor-based ranking works for IDX stock selection
3. AI chat + 4-level architecture is usable for portfolio management
4. The terminal aesthetic resonates with power users
5. A solo developer can build a complete financial application

But it also proved 5 things that failed:
1. No architecture plan → triple data sources
2. No testing → bugs only discovered by user reports
3. No validation → malformed requests crash the API
4. Security theater → dev bypass in production
5. Hidden knowledge → 50 undocumented assumptions

**QuantBit V2 preserves everything that worked and eliminates everything that failed.**
Build from scratch. Carry only the knowledge. Do not copy the code.

> "The second system is the most dangerous system a man ever designs."
> — Fred Brooks, *The Mythical Man-Month*

**Don't over-design V2. Build what's documented here, launch, and iterate.**
