# 04 — Research Findings

> Summary of all 20 research documents from the QuantBit v1 reverse engineering audit.

---

## Research Document Index

| # | Document | Key Finding |
|---|----------|-------------|
| 01 | System Overview | Architecture flow, components, workflows |
| 02 | Feature Catalog | 39 features (9 Core, 14 Secondary, 4 Experimental, 8 Deprecated) |
| 03 | Business Rules | 13 rule categories extracted from implementation |
| 04 | Data Flow | 7 pipeline steps, 5 data duplication points |
| 05 | Database Analysis | 11 findings (JSON blobs, missing FKs, naming) |
| 06 | API Analysis | 14 issues (monolithic, no validation, security holes) |
| 07 | Scoring Engine | All formulas with exact weights and action mappings |
| 08 | Backtesting Engine | Workflow, assumptions, 8 weaknesses |
| 09 | AI Module | Prompt, flow, 4-level architecture, 9 weaknesses |
| 10 | Performance Analysis | 4 bottleneck categories |
| 11 | Security Analysis | 18 findings (3 P0, 5 P1, 6 P2, 4 P3) |
| 12 | Technical Debt | 30 items (5 P0, 10 P1, 10 P2, 5 P3) |
| 13 | Lessons Learned | 10 things that worked, 10 that failed |
| 14 | Rebuild Architecture | New Hono-based architecture with 8 principles |
| 15 | Migration Priority | 12 Keep, 10 Rewrite, 16 Delete, 11 Defer |
| 16 | Rebuild Roadmap | 10 sprints (Sprint 0-9) |
| 17 | Risk Analysis | 12 risks with matrix, mitigation, Go/No-Go |
| 18 | Final Recommendation | Architecture verdict: rebuild from scratch |
| 19 | Hidden Knowledge | 50 hidden items (magic numbers, hardcodes, assumptions) |
| 20 | Data Sources Inventory | 10 categories of data duplication |

---

## Three Most Critical Findings

### Finding 1: Triple Data Sources (Critical)

Market data exists in 3-4 places that can diverge:
- **JSON** `data/years/*.json` — backtest reads from here (build-time import)
- **SQLite** `data/historical_market.sqlite` — local dev
- **D1** `daily_overview` + `stock_daily` — production
- **In-memory** `MKT.ihsg` singleton — engine reads from here (direct Yahoo)

**Impact:** Backtest results may differ from live portfolio behavior because they read from different data sources with different timestamps and transformation logic.

### Finding 2: Monolithic API (Critical)

`functions/api/[[path]].ts` is 1236 lines handling:
- Authentication (signup, login, session)
- User profile CRUD
- Market data (backtest-data, fundamentals)
- AI chat (5 providers with circuit breaker)
- Engine state management
- Email notifications
- Force-sync for IDX80
- GoAPI price proxy

**Impact:** Cannot test in isolation. One buggy route can crash the entire API. No middleware, no validation, no error standardization.

### Finding 3: Security Bypasses (Critical)

- `getUserFromSession()` returns `"dev-user"` when no token provided (line 57)
- `getUserFromSession()` returns `"dev-user"` on any error (line 69)
- GoAPI key in query string: `?api_key=${key}` (line 1005)
- Stack traces returned to client: `e.message || e.stack || "Unknown"` (line 699)
- Force-sync secret in POST body, not Authorization header (line 617)

**Impact:** Unauthenticated users can access AI chat, create sessions, send messages. API keys and secrets logged in plaintext.

---

## Key Research Insights

### Scoring Engine
- BPS algorithm is the genuine innovation — 5-factor formula with crisis override
- Rank-based percentile normalization (0-95) is sound but data source inconsistencies corrupt results
- Profile weights (AMAN/AGRESIF/DIVIDEN) are duplicated in 3+ files

### Backtesting Engine
- Robust 1500+ day loop with rebalancing, crash protection, dividend collection
- 8 weaknesses found including: no cash drag modeling, equal weight assumption, date heuristics
- Backtest reads from JSON imports while engine reads from in-memory singletons — will never match

### AI Module
- 4-level architecture (Q&A → Read → Action → Proactive) is well-designed
- 9 weaknesses: fragile regex parsing, memory leak risk, untested proactive agent
- Jaksel persona (Rico Lubis) is hardcoded in systemKnowledge.ts — not configurable

### Hidden Knowledge
50 undocumented items discovered across the codebase:
- 20 magic numbers (fees, spreads, risk-free rate, trading days, thresholds)
- 7 hardcoded datasets (profile weights, ticker lists, yearly gold/USD rates, synthetic fundamentals)
- 8 undocumented assumptions (DB != actual SOT, score range not guaranteed, rank fallback)
- 7 hidden dependencies (Yahoo undocumented API, GoAPI not production-ready, D1 lock-in)
- 3 environment configs (10+ env vars needed, model names, wrangler config)

---

## Top 5 Recommendations for V2

| # | Recommendation | Basis |
|---|---------------|-------|
| 1 | **D1 as single SOT** | Eliminates all data inconsistency — 10 categories of duplication found |
| 2 | **Domain-modular API with Hono** | Solves monolithic API — 14 distinct API issues identified |
| 3 | **Structured AI output** | Eliminates fragile regex parsing — 3 regex extraction patterns in 2 files |
| 4 | **Normalized database schema** | 5 JSON blob columns, missing FKs on 4 tables, naming inconsistency |
| 5 | **Proper authentication** | 3 security bypasses at P0 level — dev-session, API key in URL, stack trace leak |
