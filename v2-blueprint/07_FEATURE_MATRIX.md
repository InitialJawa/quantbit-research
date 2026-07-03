# 07 — Feature Matrix

> Complete inventory of all features in QuantBit v1 with V2 disposition.

---

## Core Features (Essential)

| # | Feature | V1 File(s) | Status | V2 Disposition | Priority |
|---|---------|------------|--------|----------------|----------|
| 1 | Backtest Engine | `src/engine/core.ts` | Stable | **Rewrite** — same algorithm, D1-backed data | P0 |
| 2 | Buy Pressure Score | `src/engine/buyPressure.ts` | Stable | **Keep** — preserve algorithm exactly | P0 |
| 3 | 3 Investment Profiles | `src/contexts/EngineConfigContext.tsx` | Stable | **Rewrite** — store in DB instead of code | P0 |
| 4 | Crash Protection | `src/engine/crashDetector.ts` | Stable | **Keep** — preserve 2-tier + recovery logic | P0 |
| 5 | Market Data Pipeline | `scripts/fetch_historical_data.ts` | Fragile | **Rewrite** — unified TS pipeline | P1 |
| 6 | Portfolio Manager | `src/hooks/usePortfolioManager.ts` | Stable | **Rewrite** — D1-backed CRUD | P0 |
| 7 | Rebalancing Engine | `src/engine/core.ts:444-492` | Stable | **Keep** — emergency + routine exits | P1 |
| 8 | Performance Metrics | `src/engine/metrics.ts` | Stable | **Keep** — pure math, no changes needed | P1 |
| 9 | Weighted Factor Ranking | `src/engine/ranker.ts` | Stable | **Keep** — preserve algorithm | P0 |

---

## Secondary Features (Important)

| # | Feature | V1 File(s) | Status | V2 Disposition | Priority |
|---|---------|------------|--------|----------------|----------|
| 10 | AI Chat (4 Levels) | `src/ai/*`, `src/components/FloatingAIChat.tsx` | Stable | **Rewrite** — structured output | P1 |
| 11 | Market Regime Engine | `src/marketRegimeEngine.ts` | Stable | **Keep** — preserve decision tree | P1 |
| 12 | IDX80 Scanner | `functions/api/[[path]].ts:1093` | Fragile | **Rewrite** — unified scoring | P1 |
| 13 | Dividend Forecasting | `src/engine/dividendCache.ts` | Basic | **Rewrite** — D1-backed | P2 |
| 14 | Weight Profile Management | `src/components/ManageProfilesModal.tsx` | Stable | **Keep** — UI is mature | P2 |
| 15 | Strategy Settings Panel | `src/components/StrategySettingsPanel.tsx` | Stable | **Keep** — UI is mature | P1 |
| 16 | DataStatus Transparency | `src/types/DataStatus.ts`, `src/components/DataBadge.tsx` | Stable | **Keep** — build V2 around this concept | P1 |
| 17 | Notification System | `src/contexts/NotificationContext.tsx` | Partial | **Rewrite** — 4 rules, only 3 work | P2 |
| 18 | Keyboard Shortcuts | `src/hooks/useShortcuts.ts` | Stable | **Keep** — copy as-is | P2 |
| 19 | Explain Button | `src/components/ExplainButton.tsx` | Stable | **Keep** — simple, valuable | P2 |
| 20 | Digital Wallet UI | `src/components/DigitalWalletUI.tsx` | Stable | **Keep** — UI is mature | P2 |
| 21 | Stock Drawer | `src/components/StockDrawer.tsx` | Stable | **Keep** — UI is mature | P1 |
| 22 | Market Overview Charts | `src/components/MarketOverviewCharts.tsx` | Stable | **Keep** — simple Recharts | P2 |
| 23 | Forward Dividends Forecast | `src/components/ForwardDividendsForecast.tsx` | Stable | **Keep** — simple, valuable | P2 |
| 24 | Buy Pressure Dashboard | `src/components/BuyPressureDashboard.tsx` | Stable | **Keep** — BPS gauge + breakdown | P1 |
| 25 | Reconstruct Portfolio History | `src/utils/reconstructPortfolioHistory.ts` | Stable | **Rewrite** — D1-backed | P2 |

---

## Experimental Features (Trial)

| # | Feature | V1 File(s) | Status | V2 Disposition | Priority |
|---|---------|------------|--------|----------------|----------|
| 26 | MCP Server | `src/mcp/index.ts` | Experimental | **Defer** — post-MVP | P4 |
| 27 | AI Test Harness | `src/components/AITestHarness.tsx` | Dev-only | **Keep** — useful for dev | P3 |
| 28 | Adaptive Weights | `src/engine/ranker.ts:56-147` | Experimental | **Defer** — not proven | P4 |
| 29 | Strategy Comparison | `scripts/compare_strategies.ts` | CLI | **Refactor** — reusable module | P3 |
| 30 | Weight Grid Search | `scripts/backtest_optimize_weights.ts` | CLI | **Keep** — research tool | P3 |
| 31 | Real-time WebSocket | `src/hooks/useDataFeed.ts`, `src/services/api.ts` | Stub | **Defer** — implement later | P4 |

---

## Deprecated Features (Remove)

| # | Feature | Status | Reason |
|---|---------|--------|--------|
| 32 | Legacy Gemini Endpoints (`/api/gemini/*`) | Defunct | Replaced by unified `/api/ai/chat` |
| 33 | Old AI Assistant (`AIAssistant.tsx`) | Deleted | Replaced by FloatingAIChat |
| 34 | DashboardGrid.tsx | Archived | Replaced by per-tab components |
| 35 | DiagnosticsTab.tsx | Deleted | Not needed |
| 36 | BottomNav.tsx / NavDrawer.tsx | Deleted | Superseded by AppSidebar |
| 37 | AICockpit.tsx | Deleted | Superseded by FloatingAIChat |
| 38 | DeepReport.tsx | Deleted | Not needed |
| 39 | Legacy "prod"/"res" Profile IDs | Migrated | Auto-converted on load |

---

## New Features in V2 (Not in V1)

| # | Feature | Rationale |
|---|---------|-----------|
| 40 | **Ticker Catalog Table** | Normalize all ticker references, FK constraints |
| 41 | **Configuration API** | `/api/v1/config/*` — profiles, tickers, settings |
| 42 | **Admin Dashboard** | Manage ticker lists, monitor pipeline health |
| 43 | **Pipeline Monitoring** | Alert on failed cron runs, data freshness |
| 44 | **Rate-Limited API** | Throttle AI requests, prevent abuse |
| 45 | **Golden File Tests** | Snapshot-based backtest verification |

---

## Feature Disposition Summary

| Disposition | Count | Features |
|-------------|-------|----------|
| **Keep** | 21 | Core algorithms, mature UIs, proven concepts |
| **Rewrite** | 9 | Engine, pipeline, AI, portfolio (same logic, new arch) |
| **Defer** | 3 | MCP, adaptive weights, WebSocket (post-MVP) |
| **Refactor** | 1 | Strategy comparison (reusable module) |
| **Delete** | 8 | Legacy endpoints, deprecated components |
| **New** | 6 | Tick catalog, config API, admin, monitoring, tests |

**Net V2 feature count:** 21 kept + 9 rewritten + 1 refactored + 6 new = **37 features**
