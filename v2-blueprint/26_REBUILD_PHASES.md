# 26 — Rebuild Phases

> Phased approach to building QuantBit V2 from scratch.

---

## Phase Overview

| Phase | Duration | Focus | Deliverable |
|-------|----------|-------|-------------|
| Sprint 0 | 2 days | Foundation | Empty project with CI/CD, folder structure, tools |
| Sprint 1 | 3 days | Database | D1 schema, migrations, seed scripts |
| Sprint 2 | 5 days | Engine | All pure engine modules with tests |
| Sprint 3 | 3 days | API | Hono routes, auth, middleware |
| Sprint 4 | 5 days | UI | React app with all 4 main tabs |
| Sprint 5 | 3 days | Backtest UI | Backtest tab with config + results |
| Sprint 6 | 4 days | AI | AI chat with structured output |
| Sprint 7 | 3 days | Pipeline | Data sync, cron, scores computation |
| Sprint 8 | 3 days | Polish | Testing, docs, edge cases, responsive |
| Sprint 9 | 2 days | Launch | Production deploy, monitoring, backup |
| **Total** | **33 days** | | **Complete QuantBit V2** |

---

## Sprint 0: Foundation (Days 1-2)

### Goals
- Initialize monorepo with all tooling
- Set up CI/CD pipeline
- Establish coding standards
- Create folder structure

### Tasks
- [ ] `npm create vite@latest` — React + TypeScript
- [ ] Configure `tsconfig.json` (strict mode, path aliases)
- [ ] Set up Tailwind CSS 4
- [ ] Set up Hono for backend
- [ ] Create `wrangler.toml` with D1 + KV bindings
- [ ] Create `AGENTS.md` for AI agent instructions
- [ ] Create `INDEX.md` for repo navigator
- [ ] Write CI/CD workflow (lint → typecheck → test → deploy)
- [ ] Set up `vitest` with coverage
- [ ] Copy `docs/v2/` blueprint files
- [ ] Create folder structure (src/, functions/, scripts/, tests/)

### Output
- Empty but fully configured project
- CI/CD passing
- Blueprint loaded as reference

---

## Sprint 1: Database (Days 3-5)

### Goals
- Implement D1 schema
- Create migrations
- Write seed scripts
- Engine database access layer

### Tasks
- [ ] Create migration `0000_init.sql`:
  - tickers, market_daily, stock_daily, stock_scores
  - idx80_scans, portfolios, cash_holdings, watchlists
  - trade_logs, backtest_sessions, backtest_logs
  - strategy_profiles, user_strategy_configs
  - ai_sessions, ai_messages
  - notification_rules, user_notifications
- [ ] Create migration `0001_seed_tickers.sql` — seed IDX80 ticker list
- [ ] Write `scripts/seed-database.ts` — historical data import
- [ ] Write D1 query helpers (typed, with error handling)
- [ ] Add appropriate indexes and FKs
- [ ] Add audit columns (created_at, updated_at)

### Output
- Database schema applied and migrated
- Seed script loading historical data
- Query layer ready for API development

---

## Sprint 2: Engine (Days 6-10)

### Goals
- Port all engine modules from v1
- Write comprehensive unit tests
- Ensure deterministic behavior

### Tasks
- [x] Knowledge captured — no reverse engineering needed
- [ ] Write `src/lib/engine/core.ts` — `runStrategy()` backtest loop
- [ ] Write `src/lib/engine/ranker.ts` — weighted scoring + ranking
- [ ] Write `src/lib/engine/allocator.ts` — position sizing, fees, lots
- [ ] Write `src/lib/engine/crashDetector.ts` — 2-tier crash detection
- [ ] Write `src/lib/engine/buyPressure.ts` — 5-factor BPS
- [ ] Write `src/lib/engine/metrics.ts` — CAGR, Sharpe, Sortino, etc.
- [ ] Write `src/lib/engine/marketRegime.ts` — regime classification
- [ ] Write `src/lib/engine/dividendCache.ts` — DPS lookup
- [ ] Write `src/lib/engine/notificationRules.ts` — threshold rules
- [ ] Generate golden test files from v1 backtest output
- [ ] Achieve 90%+ line coverage on engine tests

### Critical formulas to preserve exactly:
| Formula | Weights/Parameters | Source |
|---------|-------------------|--------|
| BPS | val*0.30 + mom*0.25 + breadth*0.15 + dd*0.20 + fear*0.10 | buyPressure.ts |
| Ranking | Q*Wq + G*Wg + V*Wv + M*Wm + D*Wd | ranker.ts |
| Crash | 60d drawdown ≤ -10% OR slow grind (price<SMA50 + SMA20<SMA50) | crashDetector.ts |
| Regime | 5-state priority tree | marketRegimeEngine.ts |
| Fees | buy=0.15%, sell=0.25%, tax=0.10%, slippage=0.25% | types.ts |

### Output
- Complete engine with all functions
- 90%+ test coverage
- Golden file tests matching v1 output

---

## Sprint 3: API (Days 11-13)

### Goals
- Implement all Hono routes
- Auth with Better Auth
- Input validation with Zod
- Error standardization

### Tasks
- [ ] Set up Hono app with middleware stack
- [ ] Implement auth endpoints (register, login, logout, me)
- [ ] Implement market endpoints (overview, stocks, tickers, regime, fundamentals)
- [ ] Implement portfolio endpoints (CRUD, trade, watchlist, history)
- [ ] Implement engine endpoints (BPS, config, run-backtest, results)
- [ ] Implement admin endpoints (health, tickers, force-sync)
- [ ] Write Zod schemas for all request bodies
- [ ] Write API client for frontend (`src/lib/api/*`)
- [ ] Integration tests for 3 critical endpoints

### Output
- Complete REST API
- Auth working with Better Auth
- All endpoints validated with Zod
- Integration tests passing

---

## Sprint 4: UI (Days 14-18)

### Goals
- Build all 4 main tabs
- Reuse UI patterns from v1
- Terminal aesthetic

### Tasks
- [ ] Create base UI components (Button, Input, Card, Badge, Modal, Drawer, Table)
- [ ] Create AppShell (AppHeader, AppSidebar, tab navigation)
- [ ] Build MarketTab (overview charts, stock table, stock drawer)
- [ ] Build PortfolioTracker (holdings, BPS dashboard, digital wallet, chart)
- [ ] Build AnalyticsTab (leaders, regime, notifications)
- [ ] Implement keyboard shortcuts (1/2/3/4, /, Esc)
- [ ] Implement DataStatus badges on all market data
- [ ] Responsive layout (desktop + mobile)
- [ ] Empty states and error states for all data displays

### Output
- Complete UI with 4 tabs
- All data flowing from API (not in-memory singletons)
- Responsive and accessible

---

## Sprint 5: Backtest UI (Days 19-21)

### Goals
- Connect backtest engine to UI
- Server-side backtest execution
- Result display (chart, table, log)

### Tasks
- [ ] Build BacktestConfigPanel (Draft/Live mode toggle)
- [ ] Trigger server-side backtest via API
- [ ] Polling for backtest completion
- [ ] Backtest chart (portfolio vs benchmark)
- [ ] Backtest results summary (CAGR, Sharpe, Max DD)
- [ ] Assets breakdown (cash vs gold vs stocks over time)
- [ ] Backtest log (rebalance events, crash deployments)
- [ ] SYNC TO PORTFOLIO button
- [ ] Backtest session history

### Output
- Full backtest workflow from config to results
- Server-side execution confirmed working

---

## Sprint 6: AI (Days 22-25)

### Goals
- AI chat with structured output
- 4-level architecture
- Tool definitions

### Tasks
- [ ] Set up OpenRouter integration
- [ ] Build modular system prompt
- [ ] Implement JSON mode for structured output
- [ ] Write tool definitions (8 read-only + 8 action)
- [ ] Build FloatingAIChat component
- [ ] Build AIToolApprovalCard
- [ ] Implement L1 (Q&A) and L2 (read tools)
- [ ] Implement L3 (action with approve/reject)
- [ ] Implement L4 (proactive agent) — deferred to post-MVP
- [ ] Write AI session management (create, continue, history)

### Output
- AI chat with all 3 levels working
- Structured output with no regex parsing
- Tool execution functional

---

## Sprint 7: Pipeline (Days 26-28)

### Goals
- Data sync automation
- Score computation
- GitHub Actions cron

### Tasks
- [ ] Write `scripts/sync-market-data.ts` — unified Yahoo sync
- [ ] Write `scripts/compute-scores.ts` — unified scoring algorithm
- [ ] Set up GitHub Actions daily cron workflow
- [ ] Implement force-sync admin endpoint
- [ ] Test pipeline with real Yahoo Finance data
- [ ] Error handling and retry logic
- [ ] Pipeline health monitoring

### Output
- Automated daily data pipeline
- Scores computed and stored in D1
- Cron pipeline running in production

---

## Sprint 8: Polish (Days 29-31)

### Goals
- Edge cases
- Performance optimization
- Documentation
- Security audit

### Tasks
- [ ] Handle all loading/error/empty states in UI
- [ ] Implement stale data handling + DataStatus everywhere
- [ ] Review and fix any TODO/FIXME comments
- [ ] Performance audit (bundle size, API latency, query optimization)
- [ ] Add missing D1 indexes
- [ ] Security review (auth, secrets, validation)
- [ ] Update all documentation
- [ ] Load testing (basic)

### Output
- Production-ready application
- Security audit passed
- Documentation complete

---

## Sprint 9: Launch (Days 32-33)

### Goals
- Deploy to production
- Monitor and verify
- Backup strategy

### Tasks
- [ ] Final deployment to Cloudflare Pages
- [ ] Verify all endpoints in production
- [ ] Run daily data pipeline in production
- [ ] Set up monitoring (error tracking, health checks)
- [ ] Configure weekly database backups
- [ ] Create runbook (how to diagnose common issues)
- [ ] Launch announcement

### Output
- QuantBit V2 running in production
- Monitoring and backups configured
- Runbook documented
