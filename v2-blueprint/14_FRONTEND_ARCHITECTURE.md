# 14 — Frontend Architecture

> Frontend architecture for QuantBit V2 — React + Vite + Tailwind.

---

## Technology Stack

| Layer | Technology | Rationale |
|-------|-----------|-----------|
| Framework | React 19 | Mature ecosystem, Concurrent features |
| Build | Vite 6 | Fast HMR, TypeScript-native |
| Styling | Tailwind CSS 4 | Utility-first, consistent with v1 aesthetic |
| Charts | Recharts 3 | Same as v1, lightweight, composable |
| Icons | lucide-react | Same as v1 |
| Animation | motion (framer-motion v12) | Same as v1, `AnimatePresence` for tab transitions |
| State | React Context + Hooks | Keep it simple — no Redux for this scale |
| HTTP | fetch + custom hooks | Lightweight, no heavy client library |
| Routing | Custom tab system | Same as v1 (no React Router needed) |

---

## Component Tree (V2)

```
App
├── AppHeader
│   ├── ConnectionStatus
│   ├── SearchBar
│   └── UserMenu
├── AppSidebar
│   ├── TabNavigation (1/2/3/4/5)
│   └── StrategySettingsPanel
├── MainContent
│   ├── MarketTab
│   │   ├── MarketOverviewCharts
│   │   ├── StockTable / StockGrid
│   │   ├── StockDrawer (conditional)
│   │   └── DataBadge
│   ├── PortfolioTracker
│   │   ├── HoldingsTable
│   │   ├── BuyPressureDashboard
│   │   ├── DigitalWalletUI
│   │   ├── ForwardDividendsForecast
│   │   └── PortfolioChart
│   ├── SimulationTab
│   │   ├── BacktestConfigPanel
│   │   ├── BacktestChart
│   │   ├── BacktestAssetsBreakdown
│   │   ├── BacktestLog
│   │   └── PromoteToPortfolioButton
│   ├── AnalyticsTab
│   │   ├── LeadersTable
│   │   ├── RegimeChart
│   │   ├── NotificationRules
│   │   └── FactorAnalysis
│   └── AdminTab (NEW)
│       ├── TickerManager
│       ├── PipelineMonitor
│       └── SystemHealth
└── FloatingAIChat
    ├── ChatMessages
    ├── ChatInput
    ├── AIToolApprovalCard
    └── ProactiveNotification
```

---

## Data Flow (V2)

```
User Interaction
  → Component dispatches action
  → Custom hook calls API (fetch)
  → API processes on edge
  → API returns typed response
  → Hook updates local state
  → React re-renders
```

**Key differences from V1:**
- V1: Components read from in-memory singletons (MKT, RS, L)
- V2: All data comes from API. No in-memory singletons at app level.
- V1: Engine runs client-side
- V2: Engine runs server-side, UI displays results
- V1: localStorage as config source of truth
- V2: D1 as config source of truth, localStorage as cache

---

## State Management

```typescript
// No global state store. Context + hooks per domain.

// Auth state
useAuth() → { user, login, logout, isAuthenticated }

// Portfolio state  
usePortfolio() → { holdings, cash, gold, trades, buy, sell }

// Market state
useMarket() → { ihsg, gold, usdidr, stocks, tickers, dataStatus }

// Engine state
useEngine() → { bps, regime, backtestResults, runBacktest }

// Config state
useConfig() → { profiles, strategySettings, updateProfile, updateStrategy }

// AI state
useAIChat() → { messages, sendMessage, pendingActions, approveAction }
```

Each hook:
- Manages its own loading/error/data states
- Calls the appropriate API endpoints
- Caches responses in React state (not localStorage)
- Invalidates on user action or explicit refresh

---

## Folder Structure (V2)

```
src/
├── components/       # Reusable UI components
│   ├── ui/           # Base UI (Button, Input, Card, Badge, Modal)
│   ├── market/       # Market tab components
│   ├── portfolio/    # Portfolio tab components
│   ├── backtest/     # Backtest tab components
│   ├── analytics/    # Analytics tab components
│   ├── admin/        # Admin tab components
│   └── ai/           # AI chat components
├── hooks/            # Custom React hooks
├── contexts/         # React contexts
├── lib/              # API client, utilities
│   ├── api/          # API client functions
│   ├── engine/       # Pure engine logic (used server-side too)
│   └── utils/        # Shared utilities
├── types/            # TypeScript type definitions
├── constants/        # App-wide constants
├── App.tsx
└── main.tsx
```

---

## Styling Convention

- Tailwind CSS 4 with `@import "tailwindcss"` (postcss-free)
- Terminal aesthetic: `bg-black`, `text-green-400`, `border-green-500/30`
- Consistent color palette: Black bg, Linux green (#00FF41 / green-400), amber accents for warnings
- No CSS modules or styled-components — Tailwind utility classes only
- `motion` for animations (tab transitions, card reveals, BPS gauge)

---

## Performance Targets

- Initial load: < 2s (first meaningful paint)
- Tab switch: < 100ms (instant)
- Backtest results render: < 500ms (results from server, render only)
- AI response: < 5s (provider-dependent, show loading state)
- Bundle size: < 500KB gzipped (excluding charts)
