# 11 — Component Boundaries

> All system components identified from QuantBit v1, with their responsibilities, interfaces, and recommended V2 boundaries.

---

## Component Map (V2)

```
┌─────────────────────────────────────────────────────────────────────┐
│                          FRONTEND (React)                          │
│  ┌─────────┐ ┌────────────┐ ┌──────────┐ ┌───────────┐ ┌───────┐ │
│  │ Market  │ │ Portfolio  │ │ Backtest │ │ Analytics │ │ Admin │ │
│  │ Tab     │ │ Tracker    │ │ (Sim)    │ │ Tab       │ │ Tab   │ │
│  └────┬────┘ └─────┬──────┘ └────┬─────┘ └─────┬─────┘ └───┬───┘ │
│       └────────────┼─────────────┼──────────────┼───────────┘     │
│                    ▼              ▼                               │
│              ┌─────────────────────────┐                          │
│              │     API Client (fetch)  │                          │
│              └───────────┬─────────────┘                          │
└──────────────────────────┼────────────────────────────────────────┘
                           │ HTTP
┌──────────────────────────┼────────────────────────────────────────┐
│                 EDGE (Cloudflare)                                  │
│  ┌───────────────────────┴─────────────────────┐                  │
│  │              Hono Router (API Gateway)       │                  │
│  │  /api/v1/auth  /api/v1/market  /api/v1/ai   │                  │
│  │  /api/v1/engine /api/v1/portfolio /api/v1/admin               │
│  └───────┬───────────────┬──────────────┬──────┘                  │
│          │               │              │                          │
│  ┌───────▼──────┐ ┌──────▼──────┐ ┌─────▼──────┐                  │
│  │   Auth      │ │   Engine    │ │   AI Chat  │                  │
│  │   Module    │ │   Module    │ │   Module   │                  │
│  └───────┬──────┘ └──────┬──────┘ └─────┬──────┘                  │
│          │               │              │                          │
│          ▼               ▼              ▼                          │
│  ┌──────────────────────────────────────────────────────┐         │
│  │                    D1 Database                        │         │
│  └──────────────────────────────────────────────────────┘         │
│  ┌──────────────────────────────────────────────────────┐         │
│  │          Workers KV (Cache Layer)                    │         │
│  └──────────────────────────────────────────────────────┘         │
└──────────────────────────────────────────────────────────────────┘
                           │
┌──────────────────────────┼────────────────────────────────────────┐
│               EXTERNAL                                              │
│  ┌───────────┐ ┌────────────────┐ ┌──────────────────┐           │
│  │  Yahoo    │ │   GoAPI        │ │   AI Providers   │           │
│  │  Finance  │ │   (realtime)   │ │   (OpenRouter)   │           │
│  └───────────┘ └────────────────┘ └──────────────────┘           │
└──────────────────────────────────────────────────────────────────┘
```

---

## Frontend Components

### 1. Market Tab
- **Responsibility:** Display market overview (IHSG, gold, USD/IDR), stock list with search/filter, stock detail drawer, buy/sell actions
- **Consumes:** `/api/v1/market/overview`, `/api/v1/market/stocks`, `/api/v1/portfolio`
- **V2 Change:** Reads from API (not in-memory singletons)
- **Source:** `src/components/MarketTab.tsx`

### 2. Portfolio Tracker
- **Responsibility:** Holdings table, BPS dashboard, wealth chart, trade log, cash management
- **Consumes:** `/api/v1/portfolio/*`, `/api/v1/engine/bps`
- **V2 Change:** D1-backed CRUD, not localStorage
- **Source:** `src/components/PortfolioTracker.tsx`

### 3. Simulation Tab (Backtest)
- **Responsibility:** Configure strategy, run backtest, view results (chart, log, assets), SYNC TO PORTFOLIO
- **Consumes:** `/api/v1/engine/backtest` (trigger), `/api/v1/engine/backtest/{id}` (results)
- **V2 Change:** Server-side backtest execution via API, not client-side JSON imports
- **Source:** `src/components/SimulationTab.tsx`

### 4. Analytics Tab
- **Responsibility:** Leaders/factors table, market regime chart, regime history, notification rules
- **Consumes:** `/api/v1/market/regime`, `/api/v1/market/leaders`
- **V2 Change:** None significant
- **Source:** `src/components/AnalyticsTab.tsx`

### 5. Admin Tab (NEW)
- **Responsibility:** Ticker list management, pipeline monitoring, user management, config editor
- **Consumes:** Multiple admin endpoints
- **V2:** New component, not in v1

---

## Common UI Components

| Component | Responsibility | V2 Change |
|-----------|---------------|-----------|
| AppSidebar | Tab navigation, strategy settings panel | None |
| FloatingAIChat | AI chat interface with tool approval cards | Structured output |
| StockDrawer | Ticker detail panel | None |
| DigitalWalletUI | Cash in/out, gold buy/sell | None |
| BuyPressureDashboard | BPS gauge + factor breakdown | None |
| MarketOverviewCharts | IHSG/gold/USD area charts | None |
| DataBadge | Data freshness indicator | None |
| ManageProfilesModal | Weight profile CRUD | None |
| StrategySettingsPanel | 10 unified strategy fields | None |

---

## Backend Components

### 1. Auth Module
- **Responsibility:** Register, login, logout, session validation, password management
- **Data:** `users`, `sessions` tables
- **V2 Change:** Better Auth integration, remove dev bypass
- **Source:** `functions/api/[[path]].ts` (currently merged)

### 2. Market Module
- **Responsibility:** Market overview, stock list, IDX80 scan, fundamentals, regime state
- **Data:** `daily_overview`, `stock_daily`, `stock_fundamentals`, `idx80_scans`, `tickers`
- **V2 Change:** Unified scoring, DB-backed ticker list
- **Source:** `functions/api/[[path]].ts` (currently merged)

### 3. Portfolio Module
- **Responsibility:** Portfolio CRUD, watchlist CRUD, trade log CRUD
- **Data:** `portfolios`, `watchlists`, `trade_logs` tables
- **V2 Change:** Normalized tables, no JSON blobs
- **Source:** `functions/api/[[path]].ts` (currently merged)

### 4. Engine Module
- **Responsibility:** BPS computation, backtest execution, regime computation, force-sync, crash detection
- **Data:** Reads from market/portfolio tables
- **V2 Change:** Server-side execution, API-triggered
- **Source:** `src/engine/*` (client-side), `functions/api/[[path]].ts` (force-sync)

### 5. AI Chat Module
- **Responsibility:** Chat sessions, message history, AI provider proxy, tool execution
- **Data:** `ai_sessions`, `ai_messages`
- **V2 Change:** Structured output, remove regex parsing
- **Source:** `src/ai/*` (client + server)

### 6. Admin Module (NEW)
- **Responsibility:** Health checks, pipeline monitoring, ticker management
- **Data:** Admin-only access to system tables
- **V2:** New component, not in v1

---

## Engine Components (Pure Logic)

| Component | Responsibility | Input | Output | V2 Change |
|-----------|---------------|-------|--------|-----------|
| Ranker | Weighted scoring + ranking | Norm scores, profile weights | Ranked ticker list | None (algorithm) |
| Allocator | Position sizing, entry/exit | Capital, prices, fees | Buy/sell orders | None |
| CrashDetector | Fast crash + slow grind detection | IHSG price window | Crash state + type | None |
| BuyPressure | 5-factor BPS computation | Market/regime/portfolio state | BPS score + action | None |
| Metrics | Performance calculations | Portfolio timeline | CAGR, Sharpe, etc | Fix trading days (247) |
| DividendCache | DPS lookup by ticker/year | Ticker, year | DPS amount | D1-backed storage |
| MarketRegime | Market state classification | Technical indicators | Regime state | None |
| NotificationRules | Threshold-based alerts | Engine config + market state | Fired rules | Complete 4th rule stub |

---

## Data Pipeline Components

| Component | Responsibility | V2 Change |
|-----------|---------------|-----------|
| Historical Fetcher | Download historical data (Yahoo) | Unified with daily sync |
| Daily Sync | EOD price update | Direct D1 writes, no JSON |
| IDX80 Scanner | Live market snapshot | D1-backed, no intermediate JSON |
| Score Computer | Factor scoring algorithm | Single unified algorithm |
| DB Seeder | Initialize/update D1 tables | Migration-based, not seed scripts |

---

## Boundary Rules (V2)

### Rule 1: Read Allowed
- Frontend → API: Always
- Backend → Database: Always
- Backend → External (Yahoo/GoAPI): Only through dedicated proxy endpoints
- AI → External: Never directly through /api/ai/chat

### Rule 2: Write Allowed
- Frontend → Database: Never (writes only via API)
- Backend → Database: Always (with validation)
- AI → Database: Only through approved action tools (L3+), never directly

### Rule 3: Cache Allowed
- Workers KV: For read-heavy, slow-changing data (market regime, ticker list)
- localStorage (frontend): Cache-only, never source of truth
- Never cache write operations
- Cache invalidation: TTL-based + explicit refresh endpoints

### Rule 4: Deployment Boundary
- Frontend: Deployed independently (Cloudflare Pages)
- Backend: Deployed independently (Cloudflare Pages Functions)
- Database: Migrated separately (D1 migrations)
- Data Pipeline: Cron-triggered, independent of app deploy
