# 02 — Feature Catalog

## Group Key

| Column | Meaning |
|--------|---------|
| **Complexity** | L=Low, M=Medium, H=High (relative to codebase) |
| **Recommendation** | Keep / Discard / Refactor / Replace |

---

## Core (Essential for system function)

### Backtest Engine
- **File**: `src/engine/core.ts`
- **Function**: `runStrategy()` — full backtest loop over 1500+ days. Supports 3 modes: `algo`, `custom`, `adaptive_dca`. Handles initial allocation, monthly rebalance, crash protection, adaptive DCA deployment, dividend collection, rolling IHSG window, chart data recording.
- **Dependencies**: `ranker.ts`, `allocator.ts`, `crashDetector.ts`, `metrics.ts`, `buyPressure.ts`, `dividendCache.ts`, `dividend_snapshots.json`
- **Complexity**: **H**
- **Recommendation**: **Keep**

### Buy Pressure Score (BPS)
- **File**: `src/engine/buyPressure.ts`
- **Function**: 5-factor adaptive DCA engine (`valuation*0.30 + momentum*0.25 + breadth*0.15 + drawdown*0.20 + fear*0.10`). Pure `computeBuyPressure()` + React `useBuyPressure()` hook + static `computeBuyPressureFromMarket()` for backtest. Crisis override via `withCrisisOverride()`.
- **Dependencies**: `marketRegimeEngine.ts` (isCrashActive, getIhsgDrawdown60), `marketData.ts` (MKT, RS)
- **Complexity**: **M**
- **Recommendation**: **Keep**

### 3 Investment Profiles
- **File**: `src/contexts/EngineConfigContext.tsx:37-41`
- **Function**: 3 default profiles — AMAN (Q30/G45/V10/M0/D15), AGRESIF (Q20/G60/V10/M10/D0), DIVIDEN (Q15/G20/V5/M0/D60). Custom profiles use `custom_` prefix.
- **Dependencies**: `ranker.ts` (weight application)
- **Complexity**: **L**
- **Recommendation**: **Keep**

### Crash Protection + Safe Haven
- **File**: `src/engine/crashDetector.ts` + `src/engine/core.ts:286-369` + `src/marketRegimeEngine.ts:180-187`
- **Function**: `detectCrashAlgo()` — fast crash (60d drawdown <= -sensitivity) + slow grind (price < SMA50 + SMA20 < SMA50). `detectRecoveryAlgo()` — trend recovery (price > SMA20) + momentum recovery (5d return >= 2.5%). Two gates: `isCrashActive()` (stateless, UI SOT) + `isCrisisMode()` (state machine with 20-tick cooldown, backtest-compatible).
- **Dependencies**: IHSG price window (MKT.ihsg.prices)
- **Complexity**: **M**
- **Recommendation**: **Keep**

### Market Data Pipeline
- **Files**: `scripts/fetch_historical_data.ts`, `scripts/sync-daily-data.ts`, `scripts/fetch_idx80_scan.py`, `scripts/post_process_live_market.py`, `scripts/build-db.ts`, `scripts/seed-db.ts`, `scripts/seed-d1.py`
- **Function**: Fetch Yahoo Finance + IDX API data → JSON files → SQLite/D1 → deterministic engine. GitHub Actions cron Mon-Fri.
- **Dependencies**: yahoo-finance2, better-sqlite3, curl-cffi, yfinance (Python)
- **Complexity**: **H**
- **Recommendation**: **Keep**

### Portfolio Manager
- **File**: `src/hooks/usePortfolioManager.ts`
- **Function**: CRUD operations on holdings, cash management, gold shares, trade log, watchlist, DB persistence (D1 via API), WebSocket sync.
- **Dependencies**: EngineConfigContext, NotificationContext, api service
- **Complexity**: **M**
- **Recommendation**: **Keep**

### Rebalancing Engine
- **File**: `src/engine/core.ts:444-492`
- **Function**: Rank-based crossover. Emergency exit (rank >= 15 immediate). Routine exit (rank >= 10, month-end). Swap candidates from Top N. No self-swap. Algo mode only.
- **Dependencies**: `ranker.ts` (pickTopTickersByRank), `allocator.ts` (computeSellProceeds, computeRebalanceSwap)
- **Complexity**: **M**
- **Recommendation**: **Keep**

### Performance Metrics
- **File**: `src/engine/metrics.ts`
- **Function**: CAGR, volatility (annualized, 252 trading days), Sharpe (RF=5%), Sortino, Calmar, turnover, win rate, 60/40 benchmark (60% IHSG + 40% gold).
- **Dependencies**: None (pure math)
- **Complexity**: **L**
- **Recommendation**: **Keep**

### Weighted Factor Ranking
- **File**: `src/engine/ranker.ts`
- **Function**: `computeDayRankings()` — score = quality*Wq + growth*Wg + value*Wv + momentum*Wm + dividend*Wd. Sorts descending → rank 1..N. `pickTopTickersByRank()` filters by universe + price > 0.
- **Dependencies**: ProfileWeights, stockNormScores
- **Complexity**: **L**
- **Recommendation**: **Keep**

---

## Secondary (Enhance but not critical)

### AI Chat (4 Levels)
- **Files**: `src/ai/aiClient.ts`, `src/server/aiChatHandler.ts`, `src/components/FloatingAIChat.tsx`, `src/hooks/useAITools.ts`, `src/ai/toolCallParser.ts`
- **Function**: L1 Q&A → L2 read-only tools (8 tools, no approval) → L3 actions (10 actions, require [Approve]) → L4 proactive agent (6 BPS rules, 5min cooldown, transition-based). Jaksel persona. 5 providers with circuit breaker.
- **Dependencies**: systemKnowledge.ts, MKT/RS singletons, EngineConfigContext
- **Complexity**: **H**
- **Recommendation**: **Keep**

### Market Regime Engine
- **File**: `src/marketRegimeEngine.ts`
- **Function**: Decision tree: GOLD_DEFENSE > CASH_DEFENSE > RECOVERY_WATCH > RISK_OFF > RISK_ON. Computes marketHealth, opportunity, risk, confidence, capitalDeployment. RSI, MACD, SMA20/50, drawdown60, breadth analysis. `refreshRSFromRegime()` syncs RS singleton.
- **Dependencies**: `marketData.ts` (MKT, L, RS, CW_AMAN, CW_MAP), `crashDetector.ts`, `constants/idx80.ts`
- **Complexity**: **H**
- **Recommendation**: **Keep**

### IDX80 Scanner
- **File**: `src/components/LeadersTab.tsx` (UI), `functions/api/[[path]].ts:1093` (runIdx80Scan server-side)
- **Function**: Fetches Yahoo 6mo weekly data for ~80 IDX80 tickers, computes quality/growth/value/momentum scores from price trends, writes to D1 `idx_scan_data` + `stock_fundamentals`. Triggered by force-sync API + daily cron.
- **Dependencies**: D1, Yahoo Finance via CF edge fetch
- **Complexity**: **M**
- **Recommendation**: **Keep**

### Dividend Forecasting / Cache
- **File**: `src/engine/dividendCache.ts` + `src/engine/dividendCache.ts` (imported by core.ts)
- **Function**: `setDividendCache()` / `getDividendPerShare()` — in-memory DPS lookup by ticker/year. Used in dividend annual credit (core.ts:243-273, net 90% of gross).
- **Dependencies**: `data/dividend_snapshots.json`
- **Complexity**: **L**
- **Recommendation**: **Keep**

### Weight Profile Management
- **File**: `src/components/ManageProfilesModal.tsx`
- **Function**: UI for managing weight profiles — sliders for Q/G/V/M/D weights, add/delete custom profiles (custom_ prefix), name editing.
- **Dependencies**: EngineConfigContext
- **Complexity**: **L**
- **Recommendation**: **Keep**

### Strategy Settings Panel
- **File**: `src/components/StrategySettingsPanel.tsx`
- **Function**: 10 unified fields: profile, simulationMode, universe, customUniverse, topNCount, crashSensitivity, safeHavenAsset, enableAdaptiveWeights, reserveBufferPct, enableCrossover. Used by both Portfolio + Backtest sidebar.
- **Dependencies**: EngineConfigContext
- **Complexity**: **M**
- **Recommendation**: **Keep**

### DataStatus Transparency System
- **Files**: `src/types/DataStatus.ts`, `src/utils/getDataStatus.ts`, `src/components/DataBadge.tsx`
- **Function**: LIVE / CACHED / STALE / ESTIMATED enum. Badge shows data freshness. Every data point must use DataStatus.
- **Dependencies**: None
- **Complexity**: **L**
- **Recommendation**: **Keep**

### Notification System
- **File**: `src/contexts/NotificationContext.tsx` + `src/engine/notificationRules.ts`
- **Function**: 3 methods (addNotification, fireRule, clearNotification). 4 threshold-based rules: tickerOutOfTopN, crashProtectionTriggered, customUniverseBreach, singleModeTrigger.
- **Dependencies**: EngineConfigContext (for config)
- **Complexity**: **L**
- **Recommendation**: **Keep**

### Keyboard Shortcuts
- **File**: `src/hooks/useShortcuts.ts`
- **Function**: 1/2/3/4 → tabs, / → search focus, Esc → close drawers. Ignores input-focused elements.
- **Dependencies**: None
- **Complexity**: **L**
- **Recommendation**: **Keep**

### Explain Button
- **File**: `src/components/ExplainButton.tsx`
- **Function**: Sends UI context label to FloatingAIChat to trigger AI explanation of current panel.
- **Dependencies**: FloatingAIChat (via context or prop)
- **Complexity**: **L**
- **Recommendation**: **Keep**

### Digital Wallet UI
- **File**: `src/components/DigitalWalletUI.tsx`
- **Function**: Cash in/out, gold buy/sell, trade log display, floating FAB on mobile.
- **Dependencies**: usePortfolioManager
- **Complexity**: **L**
- **Recommendation**: **Keep**

### Stock Drawer
- **File**: `src/components/StockDrawer.tsx`
- **Function**: Slide-over panel for selected ticker: price, fundamentals, buy/sell, chart, watchlist toggle.
- **Dependencies**: usePortfolioManager
- **Complexity**: **L**
- **Recommendation**: **Keep**

### Market Overview Charts
- **File**: `src/components/MarketOverviewCharts.tsx`
- **Function**: Recharts area charts for IHSG, gold, USD/IDR in Market tab header.
- **Dependencies**: MKT singleton
- **Complexity**: **L**
- **Recommendation**: **Keep**

### Forward Dividend Forecast
- **File**: `src/components/ForwardDividendsForecast.tsx`
- **Function**: Dividend projection table based on current holdings + estimated DPS.
- **Dependencies**: dividend_snapshots.json, portfolio state
- **Complexity**: **L**
- **Recommendation**: **Keep**

### Buy Pressure Dashboard
- **File**: `src/components/BuyPressureDashboard.tsx`
- **Function**: BPS gauge + 5-factor breakdown bar chart. Uses `useBuyPressure()` hook.
- **Dependencies**: buyPressure.ts
- **Complexity**: **L**
- **Recommendation**: **Keep**

### Reconstruct Portfolio History
- **File**: `src/utils/reconstructPortfolioHistory.ts`
- **Function**: Rebuild portfolio value timeline from trade log + market data for Portfolio chart.
- **Dependencies**: None
- **Complexity**: **M**
- **Recommendation**: **Keep**

---

## Experimental (Trial / Not Proven)

### MCP Server
- **File**: `src/mcp/index.ts`
- **Function**: Model Context Protocol server (Stdio transport). 5 tools (get_market_overview, get_stock_info, search_stocks, get_top_movers, get_historical_data) + 3 resources (market_overview, stocks_list, stock_detail). Uses Python db-query for D1 fallback.
- **Complexity**: **M**
- **Recommendation**: **Keep** (but mark as experimental — not integrated with main UI)

### AI Test Harness
- **File**: `src/components/AITestHarness.tsx`
- **Function**: Dev-only test panel (4 tabs: Tools/Actions/Cooldown/Storage). Activated via `?dev=ai` query param. Tests AI tool execution, action approval, cooldown logic, localStorage state.
- **Complexity**: **M**
- **Recommendation**: **Keep** (dev utility)

### Adaptive Weights
- **File**: `src/engine/ranker.ts:56-147`
- **Function**: `computeAdaptiveWeights()` — dynamically adjusts factor weights based on trailing N-day factor returns. Dividend fixed. Adaptive factors (Q/G/V/M) rebalanced via min/max normalization. Used when `enableAdaptiveWeights=true`.
- **Complexity**: **M**
- **Recommendation**: **Keep** (but not proven in production)

### Strategy Comparison
- **File**: `scripts/compare_strategies.ts`
- **Function**: Run multiple backtest configs and compare results. CLI tool, not integrated in UI.
- **Complexity**: **L**
- **Recommendation**: **Refactor** into reusable module

### Weight Grid Search Optimizer
- **File**: `scripts/backtest_optimize_weights.ts`
- **Function**: Grid search over weight combinations to find optimal profile parameters.
- **Complexity**: **M**
- **Recommendation**: **Keep** (research tool)

### Realtime WebSocket
- **File**: `src/hooks/useDataFeed.ts` (GoAPI polling 60s), `src/services/api.ts` (WebSocket stub `ws://`)
- **Function**: 60-second GoAPI polling for live IHSG/gold/USD. WebSocket line exists but unused (commented out).
- **Complexity**: **L**
- **Recommendation**: **Refactor** — replace polling with proper WebSocket when GoAPI is stable

---

## Deprecated / Removed (Superseded)

### Legacy Gemini Endpoints
- **Path**: `/api/gemini/*`
- **Status**: Defunct. Original AI chat used deprecated Gemini endpoint. Replaced by unified `/api/ai/chat`
- **Recommendation**: **Discard** (code already removed from routes)

### Old AI Assistant
- **File**: `src/components/AIAssistant.tsx` — deleted
- **Status**: Called defunct `/api/gemini/chat`. Replaced by FloatingAIChat.
- **Recommendation**: **Discard** (deleted)

### DashboardGrid.tsx
- **File**: `src/components/_archive/DashboardGrid.tsx`
- **Status**: Archived during ADR-003 refactor. Replaced by per-tab components.
- **Recommendation**: **Discard** (archived)

### DiagnosticsTab.tsx
- **Status**: Deleted FASE 2.1 (2026-06-26)
- **Recommendation**: **Discard** (deleted)

### BottomNav.tsx / NavDrawer.tsx
- **Status**: Deleted FASE 2.1. Superseded by AppSidebar + AppHeader.
- **Recommendation**: **Discard** (deleted)

### AICockpit.tsx
- **Status**: Deleted. Superseded by FloatingAIChat.
- **Recommendation**: **Discard** (deleted)

### DeepReport.tsx
- **Status**: Deleted.
- **Recommendation**: **Discard** (deleted)

### Legacy "prod"/"res" Profile IDs
- **File**: `src/contexts/EngineConfigContext.tsx:178-179`
- **Status**: Migrated → "aman"/"agresif". Old IDs auto-converted on load.
- **Recommendation**: **Discard** (migration code can be removed after all users migrated)
