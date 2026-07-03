# Rebuild Roadmap — Quantbit v2

> **Goal**: Rebuild Quantbit from the ground up using lessons from v1. Modular API, single data source, structured AI, mobile-first, security-first.
>
> **Timeline**: 9 sprints (2-week sprints = 18 weeks). Each sprint has a clear deliverable.

---

## Sprint 0 — Foundation (1 sprint)

**Objective**: Initialize project structure, tooling, and CI/CD. No business logic yet.

### Backlog
- [ ] Initialize monorepo with turborepo
- [ ] Create workspace structure:
  ```
  apps/
    api/       → Hono + Cloudflare Workers
    web/       → React 19 + Vite + Tailwind
  packages/
    shared/    → Zod schemas, types, constants
    db/        → D1 schema, migrations, seed
    engine/    → Core services (ranking, BPS, backtest)
  ```
- [ ] Configure TypeScript (strict mode, no `any`)
- [ ] Set up Hono with file-based routing scaffold
- [ ] Install and configure Tailwind CSS with mobile-first defaults
- [ ] Set up `wrangler.toml` for Cloudflare Workers + D1
- [ ] Set up CI/CD (GitHub Actions): lint → typecheck → test → deploy
- [ ] Configure Vitest with coverage thresholds (80%+)
- [ ] Install Zod, set up schema patterns
- [ ] Create ADR 001: Architecture Decision Record for v2
- [ ] Create `.env.example` with all required env vars
- [ ] Set up ESLint + Prettier with consistent rules

### Definition of Done
- [ ] `npm run dev` starts both API and web app
- [ ] `npm run lint` passes with zero errors
- [ ] `npm run typecheck` passes with zero errors
- [ ] `npm test` runs with passing placeholder test
- [ ] CI pipeline runs on every PR
- [ ] ADR 001 is written and committed

---

## Sprint 1 — Database (1 sprint)

**Objective**: Design and implement the D1 database schema. Migrate seed data from v1.

### Backlog
- [ ] Write D1 migration 001: `tickers` table
- [ ] Write D1 migration 002: `daily_prices` + indexes
- [ ] Write D1 migration 003: `market_overview` table
- [ ] Write D1 migration 004: `users` + `sessions` tables
- [ ] Write D1 migration 005: `portfolios` + `watchlists` tables
- [ ] Write D1 migration 006: `strategy_configs` + `backtest_results` tables
- [ ] Write D1 migration 007: `factor_scores` + `fundamentals` tables
- [ ] Write D1 migration 008: `ai_sessions` + `ai_messages` tables
- [ ] Write migration 009: Add all foreign key and check constraints
- [ ] Write migration 010: Add all indexes
- [ ] Create seed script that reads v1 JSON data and writes to D1
- [ ] Create seed script for default investment profiles
- [ ] Write database tests: insert, query, constraint violations
- [ ] Document schema in `db/README.md`

### Definition of Done
- [ ] All 10 migrations run cleanly on fresh D1
- [ ] Rollback script reverses all migrations
- [ ] Seed script populates tickers from v1 stock data
- [ ] Database tests pass (insert integrity, FK enforcement)
- [ ] Query performance acceptable for all known access patterns

---

## Sprint 2 — Engine Core: Ranking & BPS (1-2 sprints)

**Objective**: Implement the core deterministic engine services. All math, no API, no UI.

### Backlog
- [ ] Create `packages/engine/` with service interfaces
- [ ] Implement `RankingService`:
  - [ ] Quality factor calculation
  - [ ] Growth factor calculation
  - [ ] Value factor calculation
  - [ ] Momentum factor calculation
  - [ ] Dividend factor calculation
  - [ ] Weighted aggregation into final rank
  - [ ] Z-score normalization
- [ ] Implement `BPSService`:
  - [ ] BPS score = `(valueScore * 0.35) + (growthScore * 0.25) + (qualityScore * 0.20) + (momentumScore * 0.10) + (dividendScore * 0.10)`
  - [ ] DCA trigger: buy when BPS > threshold
  - [ ] Configurable factor weights (from strategy config)
- [ ] Implement `CrashDetectionService`:
  - [ ] Price-based threshold detection
  - [ ] Volume anomaly detection
  - [ ] Recovery mode detection
- [ ] Implement `PerformanceMetricsService`:
  - [ ] Total return
  - [ ] CAGR
  - [ ] Sharpe ratio (risk-free rate configurable)
  - [ ] Maximum drawdown
  - [ ] Volatility (annualized)
  - [ ] Win rate
  - [ ] Profit factor
- [ ] Implement `MarketRegimeService`:
  - [ ] Trend detection (bull/bear/sideways)
  - [ ] Volatility regime (low/normal/high)
  - [ ] Regime scoring

### Testing
- [ ] Unit tests for every service function (100% coverage target)
- [ ] Property-based tests for deterministic formulas
- [ ] Regression tests comparing v1 output to v2 output for same inputs
- [ ] Edge case tests: empty data, single ticker, all zeros, extreme values

### Definition of Done
- [ ] All services implemented as classes with DI
- [ ] Zero `any` types in engine code
- [ ] 100% test coverage on engine services
- [ ] All math verified against v1 output for 3 sample datasets
- [ ] ADR 002: Engine service architecture decision

---

## Sprint 3 — Backtest Engine (1 sprint)

**Objective**: Implement the modular backtest engine that consumes ranking/BPS services.

### Backlog
- [ ] Implement `BacktestService`:
  - [ ] Accept strategy config + date range
  - [ ] Run periodic rebalancing
  - [ ] Track cash and holdings over time
  - [ ] Record all trades with timestamps
  - [ ] Support multiple rebalancing strategies (periodic, threshold, BPS-driven)
  - [ ] Support multiple allocation methods (equal weight, rank-weighted, BPS-weighted)
- [ ] Implement `RebalancingService`:
  - [ ] Calculate target allocation based on strategy
  - [ ] Generate trade orders to bridge gap between current and target
  - [ ] Handle fractional shares
  - [ ] Respect minimum holding periods
- [ ] Implement `DividendService`:
  - [ ] Accrue dividends based on yield data
  - [ ] Reinvest or accumulate mode
  - [ ] Track dividend income separately
- [ ] Wire services together in `BacktestService.run()`
- [ ] Optimize backtest performance (DB queries, memoization, batch processing)

### Testing
- [ ] Backtest smoke test: 1 ticker, 1 month → verify output shape
- [ ] Backtest determinism test: same input → same output every time
- [ ] Backtest regression test: compare to v1 results
- [ ] Performance test: 30 tickers × 5 years complete in < 5 seconds
- [ ] Edge cases: empty portfolio, all cash, single rebalance

### Definition of Done
- [ ] `BacktestService.run()` produces correct, deterministic results
- [ ] Backtest output includes: value chart data, allocation over time, trade log, metrics
- [ ] Performance meets target (< 5s for 30 tickers × 5 years)
- [ ] ADR 003: Backtest engine architecture

---

## Sprint 4 — API Layer (1 sprint)

**Objective**: Build the Hono API with modular routes, Zod validation, error handling, and auth.

### Backlog
- [ ] Set up Hono router with `/api/v1/` prefix
- [ ] Build auth routes:
  - [ ] `POST /api/v1/auth/register` — create user account
  - [ ] `POST /api/v1/auth/login` — create session, return token
  - [ ] `POST /api/v1/auth/logout` — destroy session
  - [ ] `GET /api/v1/auth/me` — current user info
- [ ] Build market data routes:
  - [ ] `GET /api/v1/market/overview` — IHSG, gold, USD/IDR
  - [ ] `GET /api/v1/market/stocks` — all active tickers with latest price
  - [ ] `GET /api/v1/market/stocks/:symbol` — single stock detail + history
  - [ ] `GET /api/v1/market/ranking` — ranked stocks with factor scores
- [ ] Build portfolio routes:
  - [ ] `GET /api/v1/portfolio` — user holdings
  - [ ] `POST /api/v1/portfolio/holdings` — add/update holding
  - [ ] `DELETE /api/v1/portfolio/holdings/:id` — remove holding
  - [ ] `POST /api/v1/portfolio/rebalance` — calculate rebalance orders
- [ ] Build backtest routes:
  - [ ] `POST /api/v1/backtest/run` — execute backtest
  - [ ] `GET /api/v1/backtest/results/:id` — fetch results
  - [ ] `GET /api/v1/backtest/results` — list user's backtests
- [ ] Build strategy routes:
  - [ ] `POST /api/v1/strategies` — save strategy config
  - [ ] `GET /api/v1/strategies` — list user's configs
  - [ ] `PUT /api/v1/strategies/:id` — update config
  - [ ] `DELETE /api/v1/strategies/:id` — delete config
- [ ] Build admin routes (dev-only):
  - [ ] `CRUD /api/v1/admin/tickers` — manage ticker list
  - [ ] `GET /api/v1/admin/users` — list users
- [ ] Implement middleware:
  - [ ] Auth middleware (session validation)
  - [ ] Zod validation middleware (auto-validate request body/query/params)
  - [ ] Error handler middleware (catch all errors, return `{ error, id }`)
  - [ ] Rate limiting middleware (per-route limits)
  - [ ] CORS middleware (controlled origins)
  - [ ] Request logging middleware (structured logs)
- [ ] Write API tests (integration tests against D1)

### Definition of Done
- [ ] All routes return correct responses with proper status codes
- [ ] Auth flow works: register → login → authenticated request → logout
- [ ] Zod validation returns 400 with clear messages on invalid input
- [ ] Error handler returns `{ error: string, id: string }` for all 5xx errors
- [ ] Rate limiting blocks excessive requests
- [ ] Integration tests cover all routes (happy path + error cases)
- [ ] ADR 004: API architecture and versioning

---

## Sprint 5 — UI Core (1-2 sprints)

**Objective**: Build the frontend shell with authentication, market view, and portfolio view.

### Backlog
- [ ] Set up React 19 + Vite project with Tailwind
- [ ] Create design system:
  - [ ] Color palette (consistent with financial app)
  - [ ] Typography scale
  - [ ] Spacing system
  - [ ] Component library skeleton: Button, Input, Card, Badge, Table, Modal, Loading, ErrorState
- [ ] Build app shell:
  - [ ] Responsive layout (mobile bottom nav, desktop sidebar)
  - [ ] Header with user menu
  - [ ] Route-based tab navigation
  - [ ] Loading and error states for every route
- [ ] Build auth UI:
  - [ ] Login page (email + password)
  - [ ] Register page (name + email + password + confirm)
  - [ ] Auth guard (redirect to login if unauthenticated)
  - [ ] Token storage and auto-refresh
- [ ] Build market tab:
  - [ ] Stock list with search/filter
  - [ ] Stock row: symbol, name, price, change %, BPS score, rank
  - [ ] Stock detail: price chart, fundamentals, factor scores
  - [ ] Market overview: IHSG, gold, USD/IDR cards
- [ ] Build portfolio tab:
  - [ ] Holdings list: symbol, shares, buy price, current value, P&L
  - [ ] Portfolio summary: total value, total return, allocation pie
  - [ ] Add holding form (symbol, shares, buy price, date)
- [ ] Build strategy settings panel:
  - [ ] Profile selector (conservative/moderate/aggressive)
  - [ ] Custom weight sliders (quality/growth/value/momentum/dividend)
  - [ ] Rebalancing frequency selector
  - [ ] Save/load strategy (localStorage + API)

### Definition of Done
- [ ] Login/register flow works end-to-end
- [ ] Market tab loads and displays stocks from API
- [ ] Portfolio tab displays holdings with calculated P&L
- [ ] Strategy settings persist and affect displayed data
- [ ] UI is responsive at 375px, 768px, 1024px, 1440px
- [ ] All states covered: loading, empty, error, success

---

## Sprint 6 — Backtest UI (1 sprint)

**Objective**: Build the backtest configuration and results interface.

### Backlog
- [ ] Build backtest config form:
  - [ ] Date range picker (start date, end date)
  - [ ] Initial capital input
  - [ ] Ticker selector (multi-select)
  - [ ] Allocation method (equal weight / rank-weighted / BPS-weighted)
  - [ ] Rebalancing strategy (periodic / threshold / BPS-driven)
  - [ ] Rebalancing frequency (monthly / quarterly / annually)
  - [ ] Dividend mode (accumulate / distribute)
  - [ ] "Run Backtest" button
- [ ] Build results display:
  - [ ] Summary cards: total return, CAGR, Sharpe ratio, max drawdown, volatility
  - [ ] Portfolio value chart over time (recharts or equivalent)
  - [ ] Allocation chart (stacked area, ticker colors)
  - [ ] Performance metrics table (row per ticker)
- [ ] Build trade log viewer:
  - [ ] Expandable table of all trades
  - [ ] Trade details: date, ticker, action (buy/sell), shares, price, value
  - [ ] Filter by ticker, action type, date range
- [ ] Build portfolio sync:
  - [ ] "Send to Portfolio" button → creates holdings from backtest result
  - [ ] Confirmation dialog with summary
- [ ] Build compare feature (post-MVP deferred):
  - [ ] Save backtest for comparison
  - [ ] Side-by-side chart overlay

### Definition of Done
- [ ] Backtest runs and displays results within 5 seconds
- [ ] Chart renders portfolio value over time correctly
- [ ] Trade log loads without performance degradation for 1000+ trades
- [ ] "Send to Portfolio" creates correct holdings
- [ ] Responsive at mobile and desktop

---

## Sprint 7 — AI Integration (1 sprint)

**Objective**: Implement AI chat with structured output and action approval.

### Backlog
- [ ] Implement AI service:
  - [ ] OpenRouter client (primary provider)
  - [ ] Anthropic client (fallback, if needed)
  - [ ] Structured output via JSON schema mode
  - [ ] Streaming support (SSE)
  - [ ] Context window management
- [ ] Define structured tool schemas:
  - [ ] `getStockInfo(symbol)` → structured stock data
  - [ ] `getPortfolioSummary()` → user portfolio
  - [ ] `runBacktest(config)` → backtest result
  - [ ] `analyzeMarket()` → market overview + regime
  - [ ] `recommendAction(config)` → buy/sell/hold with reasoning
  - [ ] `explainConcept(topic)` → educational response
- [ ] Build chat UI:
  - [ ] Message list (user + assistant, streaming text)
  - [ ] System message indicators (tool calls, staus updates)
  - [ ] Input area with send button
  - [ ] Loading indicator during AI response
- [ ] Build action approval UI:
  - [ ] Tool call card: shows intent, parameters, reasoning
  - [ ] Approve / reject / modify buttons
  - [ ] Confirmed actions execute and show result
- [ ] Implement chat sessions:
  - [ ] New session creates timestamp + title
  - [ ] Session list (sidebar on desktop, drawer on mobile)
  - [ ] Message persistence (load previous session on select)
- [ ] Add AI settings:
  - [ ] Model selector (limited to 2-3 options)
  - [ ] Temperature slider
  - [ ] Max tokens

### Definition of Done
- [ ] AI responds to queries with structured, parseable output
- [ ] Tool calls display approval UI before execution
- [ ] Streaming works: tokens appear as generated
- [ ] Sessions persist across page reloads
- [ ] All tool schemas validated on both client and server
- [ ] ADR 005: AI architecture and structured output

---

## Sprint 8 — Polish (1 sprint)

**Objective**: Production readiness — performance, security, testing, documentation.

### Backlog
- [ ] Mobile responsive audit:
  - [ ] Test all views at 320px, 375px, 414px, 768px
  - [ ] Fix touch target sizes (< 44px)
  - [ ] Fix overflow issues
  - [ ] Test with mobile network throttling
- [ ] Performance optimization:
  - [ ] Bundle size analysis (target < 200KB gzipped)
  - [ ] Code splitting by route
  - [ ] Lazy load non-critical components (AI chat, backtest results)
  - [ ] Optimize re-renders (React.memo, useMemo, useCallback)
  - [ ] API response caching (Cloudflare Cache API)
  - [ ] Image optimization
- [ ] Security audit:
  - [ ] No secrets in client bundle
  - [ ] CSRF protection
  - [ ] XSS prevention (React sanitize, CSP headers)
  - [ ] Rate limiting verified
  - [ ] Auth session expiry verified
  - [ ] SQL injection prevention verified (parameterized queries)
  - [ ] No `eval()` or `new Function()`
  - [ ] Dependency audit (`npm audit`)
- [ ] Testing:
  - [ ] E2E tests (Playwright): login → view market → run backtest → chat with AI
  - [ ] Integration tests for all API routes
  - [ ] Load test: 100 concurrent users hitting market overview
  - [ ] Accessibility audit: keyboard navigation, screen reader
- [ ] Documentation:
  - [ ] README with setup instructions
  - [ ] API documentation (automated from Zod + Hono)
  - [ ] Deployment guide
  - [ ] ADRs updated for all decisions
- [ ] Monitoring setup:
  - [ ] Error tracking (Sentry or similar)
  - [ ] Performance monitoring (Web Vitals)
  - [ ] API latency tracking
  - [ ] D1 query performance tracking

### Definition of Done
- [ ] Lighthouse score > 90 on all metrics
- [ ] Bundle size < 200KB gzipped
- [ ] All E2E tests pass
- [ ] Security audit passes with zero critical findings
- [ ] API docs published and accurate
- [ ] Monitoring alerts configured

---

## Sprint 9 — MVP Launch (1 sprint)

**Objective**: Production deployment, onboarding, go-live.

### Backlog
- [ ] Production deployment:
  - [ ] Cloudflare Workers (API)
  - [ ] Cloudflare Pages (web)
  - [ ] D1 database (production instance)
  - [ ] Custom domain setup
  - [ ] SSL/TLS verified
- [ ] User onboarding:
  - [ ] Welcome email or in-app guide
  - [ ] First-run experience: create portfolio → run backtest → see ranking
  - [ ] Tooltips and walkthrough for key features
- [ ] Backup strategy:
  - [ ] Daily D1 backup to R2
  - [ ] Point-in-time recovery tested
  - [ ] Disaster recovery runbook
- [ ] Monitoring go-live:
  - [ ] Uptime monitoring (Pingdom or equivalent)
  - [ ] Error alerting (PagerDuty or Slack)
  - [ ] Performance dashboard
- [ ] Launch checklist:
  - [ ] All env vars set in production
  - [ ] Rate limits configured for production traffic
  - [ ] CORS set to production domain only
  - [ ] Analytics enabled (Plausible or equivalent)
  - [ ] Terms of service and privacy policy published
  - [ ] Contact/support channel set up

### Definition of Done
- [ ] App is live at production URL
- [ ] New user can sign up, log in, and run a backtest in < 2 minutes
- [ ] Backup system verified (test restore)
- [ ] Monitoring shows green on all services
- [ ] Post-launch ADR written reflecting any lessons
- [ ] README updated for production deployment

---

## Post-MVP Backlog

These are tracked but not scheduled for MVP:

| Feature | Dependencies | Notes |
|---|---|---|
| MCP Server | AI service stable | External AI agent integration |
| Adaptive Weights | MarketRegimeService | Dynamic weight adjustment |
| Strategy Comparison | Backtest UI stable | Side-by-side backtest results |
| Weight Grid Search | Backtest engine + queue | Brute-force optimization |
| Email notifications | Auth + user preferences | Resend integration |
| Proactive AI agent | AI service + zero-execute guard | Opt-in autonomous mode |
| Gold conversion | Backtest engine | IDR → gold valuation |
| Admin dashboard | User base > 100 | Admin CRUD UI |
| Watchlist | Portfolio MVP | Track stocks without buying |
| Fundamental refresh | Data pipeline | Automated scheduler |

---

## Risk Register

| Risk | Likelihood | Impact | Mitigation |
|---|---|---|---|
| D1 latency higher than expected | Medium | High (app feels slow) | Benchmark early (Sprint 1); implement caching (Sprint 8) |
| Backtest engine slower than v1 | Low | Medium | Performance test in Sprint 3; optimize queries |
| Hono learning curve | Medium | Low (team learns) | Spike in Sprint 0; use simple patterns first |
| AI structured output breaks with new models | Medium | High | Pin model versions; integration tests in CI |
| Mobile responsive takes longer than estimated | High | Medium | Start mobile-first in Sprint 5; don't defer |
| Seed data migration incomplete | Medium | Medium | Validate seed in Sprint 1; manual QA in Sprint 8 |
| Rate limiting blocks legitimate users | Low | Medium | Configurable per-endpoint; monitor in Sprint 9 |
| Auth security vulnerability | Low | Critical | Security audit in Sprint 8; penetration test before launch |
