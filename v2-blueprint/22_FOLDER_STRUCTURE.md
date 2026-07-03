# 22 — Folder Structure

> Ideal folder structure for QuantBit V2 monorepo.

---

## Root Structure

```
quantbit-v2/
├── src/                          # Frontend (React SPA)
│   ├── components/               # Reusable UI components
│   │   ├── ui/                   # Base UI primitives
│   │   ├── market/               # Market tab components
│   │   ├── portfolio/            # Portfolio tab components
│   │   ├── backtest/             # Backtest tab components
│   │   ├── analytics/            # Analytics tab components
│   │   ├── admin/                # Admin tab components
│   │   └── ai/                   # AI chat components
│   ├── hooks/                    # Custom React hooks
│   ├── contexts/                 # React contexts (minimal)
│   ├── lib/                      # Shared library code
│   │   ├── api/                  # API client functions
│   │   ├── engine/               # Pure engine logic (shared)
│   │   └── utils/                # Utility functions
│   ├── types/                    # TypeScript type definitions
│   ├── constants/                # App-wide constants
│   ├── App.tsx
│   └── main.tsx
│
├── functions/                    # Backend (Cloudflare Pages Functions)
│   └── api/
│       └── v1/
│           ├── auth/             # Authentication endpoints
│           ├── market/           # Market data endpoints
│           ├── portfolio/        # Portfolio endpoints
│           ├── engine/           # Engine/backtest endpoints
│           ├── ai/               # AI chat endpoints
│           └── admin/            # Admin endpoints
│
├── scripts/                      # Data pipeline scripts
│   ├── sync-market-data.ts       # Daily market sync
│   ├── compute-scores.ts         # Score computation
│   └── seed-database.ts          # Initial database seed
│
├── db/                           # Database
│   └── migrations/               # D1 migration files
│
├── tests/                        # Tests
│   ├── unit/                     # Unit tests
│   │   ├── engine/               # Engine logic tests
│   │   └── api/                  # API endpoint tests
│   ├── integration/              # Integration tests
│   └── fixtures/                 # Test data fixtures
│
├── docs/                         # Documentation
│   └── v2/                       # V2 blueprint (this directory)
│
├── .github/                      # GitHub configuration
│   └── workflows/                # CI/CD pipelines
│
├── wrangler.toml                 # Cloudflare configuration
├── package.json
├── tsconfig.json
├── tsconfig.node.json
├── vite.config.ts
├── tailwind.config.ts
├── .env.example
├── .gitignore
├── INDEX.md                      # Repository index
└── AGENTS.md                     # AI agent instructions
```

---

## Frontend Structure (Detailed)

```
src/
├── components/
│   ├── ui/
│   │   ├── Button.tsx
│   │   ├── Input.tsx
│   │   ├── Card.tsx
│   │   ├── Badge.tsx             # DataStatus badge, score badges
│   │   ├── Modal.tsx
│   │   ├── Drawer.tsx
│   │   ├── Table.tsx
│   │   ├── Tabs.tsx
│   │   ├── Spinner.tsx
│   │   ├── Skeleton.tsx
│   │   ├── EmptyState.tsx
│   │   ├── ErrorState.tsx
│   │   └── LoadingState.tsx
│   │
│   ├── market/
│   │   ├── MarketTab.tsx
│   │   ├── MarketOverviewCharts.tsx
│   │   ├── StockTable.tsx
│   │   ├── StockGrid.tsx         # Mobile: card grid instead of table
│   │   ├── StockDrawer.tsx
│   │   └── StockSearch.tsx
│   │
│   ├── portfolio/
│   │   ├── PortfolioTracker.tsx
│   │   ├── HoldingsTable.tsx
│   │   ├── BuyPressureDashboard.tsx  # BPS gauge + factor breakdown
│   │   ├── DigitalWalletUI.tsx
│   │   ├── ForwardDividendsForecast.tsx
│   │   ├── PortfolioChart.tsx
│   │   └── TradeLog.tsx
│   │
│   ├── backtest/
│   │   ├── SimulationTab.tsx
│   │   ├── BacktestConfigPanel.tsx
│   │   ├── BacktestChart.tsx
│   │   ├── BacktestAssetsBreakdown.tsx
│   │   ├── BacktestLog.tsx
│   │   └── PromoteToPortfolioButton.tsx
│   │
│   ├── analytics/
│   │   ├── AnalyticsTab.tsx
│   │   ├── LeadersTable.tsx
│   │   ├── RegimeChart.tsx
│   │   ├── NotificationRules.tsx
│   │   └── FactorAnalysis.tsx
│   │
│   ├── admin/
│   │   ├── AdminTab.tsx
│   │   ├── TickerManager.tsx
│   │   ├── PipelineMonitor.tsx
│   │   └── SystemHealth.tsx
│   │
│   └── ai/
│       ├── FloatingAIChat.tsx
│       ├── ChatMessages.tsx
│       ├── ChatInput.tsx
│       ├── AIToolApprovalCard.tsx
│       └── ProactiveNotification.tsx
│
├── hooks/
│   ├── useAuth.ts
│   ├── usePortfolio.ts
│   ├── useMarket.ts
│   ├── useEngine.ts
│   ├── useConfig.ts
│   ├── useAIChat.ts
│   ├── useShortcuts.ts
│   └── useDataStatus.ts
│
├── contexts/
│   ├── AuthContext.tsx            # Minimal — only auth state
│   └── ConfigContext.tsx          # Minimal — only strategy config
│
├── lib/
│   ├── api/
│   │   ├── client.ts             # Base API client (fetch wrapper)
│   │   ├── auth.ts               # Auth API calls
│   │   ├── market.ts             # Market API calls
│   │   ├── portfolio.ts          # Portfolio API calls
│   │   ├── engine.ts             # Engine API calls
│   │   └── ai.ts                 # AI API calls
│   │
│   ├── engine/                    # Pure engine logic (shared with backend)
│   │   ├── core.ts               # Backtest runner
│   │   ├── ranker.ts             # Scoring + ranking
│   │   ├── allocator.ts          # Position sizing
│   │   ├── crashDetector.ts      # Crash detection
│   │   ├── buyPressure.ts        # BPS computation
│   │   ├── metrics.ts            # Performance metrics
│   │   ├── marketRegime.ts       # Regime classification
│   │   ├── dividendCache.ts      # Dividend lookup
│   │   └── notificationRules.ts  # Notification trigger logic
│   │
│   └── utils/
│       ├── formatting.ts         # Number, date, currency formatting
│       ├── validation.ts         # Shared validation functions
│       ├── reconstructPortfolioHistory.ts
│       └── dataStatus.ts         # Data freshness determination
│
├── types/
│   ├── api.ts                    # API request/response types
│   ├── market.ts                 # Market data types
│   ├── portfolio.ts              # Portfolio types
│   ├── engine.ts                 # Engine config/types
│   ├── ai.ts                     # AI chat types
│   └── common.ts                 # Shared types (DataStatus, etc.)
│
└── constants/
    ├── fees.ts                   # Default fee rates
    ├── profiles.ts               # Default profile weights
    ├── tickers.ts                # IDX80 ticker list (fallback only)
    └── market.ts                 # Trading days, sectors, etc.
```

---

## Backend Structure (Detailed)

```
functions/api/v1/
├── index.ts                      # Hono app setup, CORS, middleware
├── auth/
│   ├── register.ts
│   ├── login.ts
│   ├── logout.ts
│   └── me.ts
├── market/
│   ├── overview.ts
│   ├── stocks.ts
│   ├── tickers.ts
│   ├── regime.ts
│   ├── fundamentals.ts
│   └── backtest-data.ts
├── portfolio/
│   ├── index.ts                  # GET /portfolio
│   ├── trade.ts
│   ├── watchlist.ts
│   └── history.ts
├── engine/
│   ├── bps.ts
│   ├── run-backtest.ts
│   ├── backtest.ts               # GET /engine/backtest/:id
│   ├── backtests.ts              # GET /engine/backtests
│   └── config.ts
├── ai/
│   ├── chat.ts
│   └── sessions.ts
├── admin/
│   ├── health.ts
│   ├── tickers.ts
│   └── force-sync.ts
└── middleware/
    ├── auth.ts                   # Auth middleware
    ├── cors.ts                   # CORS middleware
    ├── rate-limit.ts             # Rate limiting
    └── error-handler.ts          # Error handler
```

---

## V1 → V2 Folder Changes

| V1 Path | V2 Path | Change |
|---------|---------|--------|
| `functions/api/[[path]].ts` | `functions/api/v1/*` | Monolithic → modular |
| `src/engine/*` | `src/lib/engine/*` (shared) + `functions/api/v1/engine/*` | Split to shared library + API handlers |
| `src/ai/*` | `src/lib/api/ai.ts` + `src/components/ai/*` | Clear frontend/backend split |
| `src/server/*` | (removed) | No Express server in V2 |
| `src/mcp/*` | (removed) | Deferred to post-MVP |
| `src/scripts/*` | `scripts/*` | Moved to root |
| `src/data/*` | (removed) | No static data imports |
| `data/years/*` | (removed) | D1-only storage |
| `docs/` | `docs/` (restructured) | Added v2/ subdirectory |
| `research/` | `docs/v2/` | Integrated into blueprint |
