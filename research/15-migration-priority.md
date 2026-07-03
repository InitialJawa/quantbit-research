# Migration Priority — Quantbit v1 → v2

## MUST KEEP (preserve concept, reimplement)

These are the assets worth carrying forward. They are conceptually correct but may need improved implementation.

| Asset | File(s) | Preserve | Reimplement |
|---|---|---|---|
| **BPS algorithm** | `src/engine/core.ts` (bpsScore function) | Scoring formula, factor weights, DCA trigger logic | Extract to `services/bps/` with proper DI, unit tests, configurable weights |
| **3 Investment Profiles** | `src/contexts/EngineConfigContext.tsx` | Profile presets (conservative/moderate/aggressive), factor weight mapping | Store as seed data in DB, expose via API, remove hardcoded config |
| **Weighted factor ranking** | `src/engine/core.ts` (calculateRankings) | Pure math: weighted sum across quality/growth/value/momentum/dividend | Move to `services/ranking/`, parameterize weights, add z-score normalization |
| **Crash detection algorithm** | `src/engine/core.ts` (detectCrash) | Threshold-based crash detection logic | Move to `services/risk/crash.ts`, add configurable thresholds, backtest integration |
| **Performance metrics formulas** | `src/engine/core.ts` (CAGR, Sharpe, max drawdown, etc.) | All deterministic formulas | Move to `services/backtest/performance.ts`, ensure floating-point precision |
| **Dividend accrual logic** | `src/engine/core.ts` (dividendAccrual) | Dividend calculation during backtest | Move to `services/backtest/dividend.ts`, source yield data from DB |
| **DataStatus concept** | `src/types.ts` (DataStatus enum, DataPoint) | Status transparency pattern | Keep as type, enforce at API boundary, auto-set from DB freshness |
| **Zero auto-execute for AI** | `src/components/AIAssistant.tsx` | AI proposes, user approves pattern | Rebuild in new AI service with structured tool responses |
| **ADR documentation pattern** | `docs/adr/` | Append-only decision records | Continue series from last ADR, adopt for v2 decisions |
| **Parameterized SQL queries** | `functions/api/[[path]].ts` | D1 bindings with `?` placeholders | Continue practice, enforce via lint rule |

---

## MUST REWRITE (foundation in new architecture)

These require complete reimplementation using v2 patterns. The old code informs requirements but should not be ported directly.

| Area | Old File(s) | Why Rewrite | New Home |
|---|---|---|---|
| **API layer** | `functions/api/[[path]].ts` (1236 lines) | Monolithic, untestable, mixed concerns | `routes/` directory, one file per endpoint, Hono framework |
| **Engine module** | `src/engine/core.ts` | Monolithic 2000+ line file | `services/` sub-directories per concern |
| **AI client** | `src/ai/` (7 files) | Regex parsing, 5 providers, over-engineered | `services/ai/` with structured JSON schema, 2 providers, streaming |
| **Data pipeline** | `scripts/` (12+ files, mixed languages) | Python + JS + shell, multi-step, fragile | TypeScript-only pipeline, `scripts/pipeline/`, D1 single target |
| **Auth system** | `functions/api/[[path]].ts` (auth section) | Dev bypass, basic hashing, no rate limiting | `services/auth/` with proper hashing, session management, rate limiting |
| **Error handling** | `functions/api/[[path]].ts` (try/catch) | Stack trace leakage, inconsistent format | Middleware-based, `{ error, id }` response format, log correlation |
| **State management** | `src/contexts/` (Singletons, global state) | Module-level mutable state, global singletons | DI pattern, services receive dependencies, no module state |
| **UI layout** | `src/App.tsx`, `src/components/` | Desktop-only, no component tree discipline | Mobile-first responsive, component library, proper routing |
| **Database schema** | `db/schema.sql` | JSON blobs, no foreign keys, no indexes | Proper relations, FKs, CHECK constraints, covering indexes |
| **Market data API client** | `src/api/marketData.ts` | Mixed sources, fallback chains | Single API client targeting D1 only |

---

## MUST DELETE (not coming to v2)

These files have no place in v2. Delete them from the repository entirely.

| File | Reason | Replacement Strategy |
|---|---|---|
| `functions/api/[[path]].ts` | Monolithic handler, v2 uses modular routes | Remove entirely; rewrite as `routes/` in new project |
| `src/ai/toolCallParser.ts` | Regex-based AI parsing | Replace with JSON schema structured output |
| `src/ai/devMockAI.ts` | Dev-only mock, shipped to production | Remove; use environment-based mock in v2 if needed |
| `src/components/AITestHarness.tsx` | Dev-only UI, shipped to production | Remove; QA testing done via API tests |
| `src/stocksData.ts` | 830 hardcoded tickers | Replace with `tickers` DB table + admin CRUD |
| `src/marketData.ts` | Global singletons MKT/RS/L | Remove; use DI service instances |
| `src/server/` | Express-specific server code | Remove; v2 uses Cloudflare Workers |
| `server.ts` | Express dev server | Remove; use `wrangler dev` or `vite` |
| `scripts/` (all .py, .sh, legacy .ts) | Mixed language, fragile pipeline | Rewrite as TypeScript pipeline in v2 |
| `collectors/` (Python scrapers) | Python dependency | Replace with TypeScript + D1 direct writes |
| `data/` (JSON/CSV files) | File-based data sources | Remove; all data lives in D1 |
| `docs/` (old architecture docs) | Outdated, references v1 architecture | Rewrite for v2; archive originals under `docs/archive/v1/` |
| `handover/` | Old session snapshots | Remove; v2 has new handover process |
| `external/` (git submodules) | Dependency management complexity | Remove; use npm packages instead |
| `src/engine/` (entire directory) | Monolithic engine, v2 uses services | Remove all files; rewrite as modular services |
| `db/schema.sql` | v1 database schema | Replace with v2 schema |

**Additional cleanup**:
- Remove any `.env` files with old API keys
- Remove `dist/` (v1 build output)
- Remove `wrangler.toml` (v1 Cloudflare config; rewrite for v2)
- Remove `vite.config.ts` (rewrite for v2 project structure)

---

## MUST DEFER (post-MVP)

These features existed in v1 but should NOT be rebuilt in the initial v2 MVP. Revisit after launch.

| Feature | Old Implementation | Rationale for Deferral | When to Revisit |
|---|---|---|---|
| **MCP Server** (Model Context Protocol) | External AI agent integration | No proven user demand; adds maintenance surface | Post-MVP if users request API access for external AI tools |
| **Adaptive Weights** (computeAdaptiveWeights) | Dynamic factor weighting based on market regime | Good idea, but adds complexity to MVP; fixed weights work for launch | Sprint 9+ after backtest engine is stable |
| **Strategy Comparison** | Side-by-side backtest comparison | Niche feature; most users run one strategy at a time | Sprint 9+ based on user feedback |
| **Weight Grid Search Optimizer** | Brute-force weight optimization | Computationally expensive; requires background workers | Sprint 10+ with queue system |
| **Email notifications** (Resend) | Automated alerts | No auth system yet for user preference management | Sprint 8+ after auth MVP |
| **Proactive AI agent** (Level 4) | AI that takes independent action | Zero auto-execute policy means this requires careful design; MVP chat-only is sufficient | Post-MVP with explicit opt-in |
| **AI Test Harness** | Dev-only AI testing UI | Not needed for production; use automated tests | Never — use integration tests instead |
| **Multiple AI providers** | 5 providers, 9 models | 2 providers max for MVP; add more if provider reliability issues arise | Only if OpenRouter uptime < 99.9% |
| **Gold conversion** (IDR to gold) | Gold-denominated portfolio view | Adds complexity; IDR-only for MVP | Sprint 8+ as enhancement |
| **Fundamental data refresh scheduler** | Scheduled data fetching | Manual refresh acceptable for MVP; automate post-launch | Sprint 9+ with Cron Triggers |
| **Admin dashboard** | User management, ticker CRUD UI | Not needed for MVP; use direct DB for admin tasks | When user base > 100 |
| **Watchlist feature** | Track stocks without buying | Nice-to-have; users can use portfolio for MVP | Sprint 7+ |

---

## Migration Order Summary

```
Phase 1 (Sprints 0-1): Foundation + DB
  → Initialize project, set up Hono + D1 + Zod
  → Create new database schema
  → Seed data from old sources
  → KEEP: ADR pattern

Phase 2 (Sprints 2-3): Engine Core
  → Rewrite ranking, BPS, crash detection
  → KEEP: deterministic formulas
  → DELETE: src/engine/

Phase 3 (Sprints 4): API
  → Rewrite API routes
  → KEEP: parameterized SQL
  → DELETE: functions/api/[[path]].ts

Phase 4 (Sprints 5-6): UI
  → Rewrite frontend
  → DELETE: src/components/AITestHarness.tsx
  → DELETE: src/stocksData.ts, src/marketData.ts

Phase 5 (Sprint 7): AI
  → Rewrite AI client with structured output
  → DELETE: src/ai/toolCallParser.ts, src/ai/devMockAI.ts

Phase 6 (Sprint 8): Polish
  → Remove all remaining deleted files
  → Archive old docs
  → DEFER items stay in the backlog

Phase 7 (Sprint 9): Launch
  → Production deployment
  → DEFER items queued for post-MVP
```
