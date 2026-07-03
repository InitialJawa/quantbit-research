# 01 — System Overview

## 1. High-Level Architecture Flow

```
User → Login → App Shell → Tabs (Market / Portfolio / Backtest / Analytics)
                                    ↓
Data Sources → Collectors → Data Pipeline → Database → Engine → Scoring → AI → Dashboard
```

Tab routing (keyboard shortcuts `1/2/3/4`):
- **Market** (`MarketTab.tsx`): Stock list + search, buy/sell panel, market overview charts
- **Portfolio** (`PortfolioTracker.tsx`): Holdings, BPS dashboard, trade log, cash management
- **Backtest** (`SimulationTab.tsx`): Run strategy → results → chart → assets → SYNC TO PORTFOLIO
- **Analytics** (`AnalyticsTab.tsx`): Leaders/factors, market regime, regime history, notification rules

## 2. Data Flow

```
Yahoo Finance API → fetch_historical_data.ts (TS, yahoo-finance2)
                         ↓
                    data/years/*.json  (per-year price + score snapshots)
                    data/fundamental_snapshots.json  (dividend per share, DPS)

                    fetch_dividend_history.ts (TS)
                         ↓
                    data/fundamental_snapshots.json  (DPS by ticker/year)
                    src/data/dividend_snapshots.json  (React-importable copy)

                    build-db.ts (better-sqlite3)
                         ↓
                    data/historical_market.sqlite  (local DB for develop)

                    seed-db.ts / seed-d1.py
                         ↓
                    Cloudflare D1 (production: daily_overview, stock_daily, stock_fundamentals)

GitHub Actions (daily cron Mon-Fri 14:00 UTC + 06:30 UTC):
  1. fetch_idx80_scan.py → data/idx80_scan.json  (live Yahoo Finance snapshot)
  2. post_process_live_market.py → data/live_market.json  (IHSG, gold, USD/IDR, oil)
  3. sync-daily-data.ts → upsert data/years/<year>.json  (Yahoo EOD close via spark API)
  4. seed-d1.py → production D1 tables
  5. curl POST /api/engine/force-sync → runIdx80Scan() on CF edge (D1 idx_scan_data)
```

### Live Path (Today)
```
Yahoo Spark API (batches of 20)
  → sync-daily-data.ts (--force)
    → data/years/<year>.json (upsert merge per-ticker close + volume)
      → seed-d1.py (D1 REST API)
        → Cloudflare D1 (daily_overview, stock_daily)

idx80_scan.py (Python, curl-cffi)
  → data/idx80_scan.json (live snapshot ~80 tickers)
    → seed-d1.py → D1 idx_scan_data
      → force-sync → CF Functions runIdx80Scan() → D1 stock_fundamentals (norm_scores)
```

### React UI Reading Path
```
MKT / RS / L singletons (src/marketData.ts) — loaded at app init from static JSON imports
  ↓
useDataFeed.ts — optional live refresh via GoAPI 60s polling
  ↓
Components (MarketTab, PortfolioTracker, etc.)
```

### Backtest Path
```
data/years/*.json (BacktestDayData[])
  → runStrategy() (src/engine/core.ts)
    → ranker.ts (computeDayRankings + pickTopTickersByRank)
    → allocator.ts (computeInitialAllocation, liquidateHoldings, computeRebalanceSwap)
    → crashDetector.ts (detectCrashAlgo, detectRecoveryAlgo)
    → metrics.ts (CAGR, Sharpe, Sortino, Calmar)
  → BacktestResult → chart + table + log
  → SYNC TO PORTFOLIO → EngineConfigContext.syncFromBacktest()
```

## 3. System Components

### Frontend
- **React 19** with `motion` (framer-motion v12) for animations
- **Vite 6** bundler, Tailwind CSS 4 (native CSS `@import "tailwindcss"`)
- **Recharts 3** for charts (backtest line, market overview, BPS gauge)
- **lucide-react** icons, **sonner** toast notifications
- **motion/react** for `AnimatePresence` tab transitions
- TypeScript strict mode, `tsx` for dev server + scripts

### Backend (Production)
- **Cloudflare Pages Functions** (edge runtime, D1 binding `quantbit-db`)
- Single catch-all route: `functions/api/[[path]].ts` — auth, CRUD, AI chat, force-sync, user settings
- PBKDF2 password hashing, session tokens via Bearer header or cookie
- D1 tables: `users`, `sessions`, `portfolios`, `watchlists`, `ai_sessions`, `ai_messages`, `daily_overview`, `stock_daily`, `stock_fundamentals`, `idx_scan_data`, `user_notifications`

### Backend (Local Dev)
- **Express server** (`server.ts`) with better-sqlite3 replacing D1
- Hatched auth (no real OAuth), AI chat using same `aiChatHandler.ts`

### Engine (src/engine/) — Pure TS, Zero AI
- `core.ts` — `runStrategy()`: 1500+ day backtest loop with crash protection, adaptive DCA, rebalancing, dividend collection, chart recording
- `ranker.ts` — `computeDayRankings()`: 5-factor weighted scoring, `pickTopTickersByRank()`, `computeAdaptiveWeights()`
- `buyPressure.ts` — 5-factor BPS with crisis override, React hook + pure function
- `allocator.ts` — Initial allocation, liquidation, gold purchase/sale, rebalance swap, sell proceeds
- `crashDetector.ts` — `detectCrashAlgo()` (fast crash + slow grind), `detectRecoveryAlgo()` (trend + momentum), single-ticker variants
- `metrics.ts` — CAGR, volatility, Sharpe, Sortino, Calmar, turnover, win rate, 60/40 benchmark
- `notificationRules.ts` — 4 threshold-based rules
- `dividendCache.ts` — In-memory DPS lookup by ticker/year

### AI Layer (src/ai/)
- **5 providers** with circuit breaker:
  1. OpenRouter `openai/gpt-oss-120b:free` (OpenInference pool)
  2. OpenRouter `nvidia/nemotron-3-super-120b:free` (Nvidia pool)
  3. OpenRouter `cohere/north-mini-code:free` (Cohere pool)
  4. OpenRouter `meta-llama/llama-3.3-70b-instruct:free` (Venice pool)
  5. Groq compound (direct, geo-blocked risk)
  6. Groq llama (direct fallback)
  7. Google Gemini/Gemma (direct, geo-blocked)
- **Circuit breaker**: 429 → 5min cooldown, 401/403 → 15min cooldown
- **4 levels**: L1 Q&A → L2 read-only tools → L3 actions (require approve) → L4 proactive agent
- **systemKnowledge.ts**: 14-section system prompt with Jaksel persona, tool catalog, BPS/regime formulas
- **devMockAI.ts**: Pattern-match mock for offline dev

### Data Pipeline
- **GitHub Actions**: `daily-data-pipeline.yml` — schedule `0 14 * * 1-5` + `30 6 * * 1-5`
- **Python** scripts: `fetch_idx80_scan.py` (curl-cffi for anti-bot), `post_process_live_market.py`, `seed-d1.py`
- **TypeScript** scripts: `fetch_historical_data.ts`, `sync-daily-data.ts`, `build-db.ts`, `seed-db.ts`, `migrate-normscores.ts`
- **Collectors**: `collectors/fetch_idx_fundamental.py` — IDX API fundamental data → parquet + JSON

## 4. Key Workflows

### Backtest
```
User sets config → "Jalankan Backtest" button → runStrategy(dayData, config, weights)
  → rank each day → pick Top N → allocate → loop with rebalance/crash/dividend
  → compute metrics → render chart + table → "SYNC TO PORTFOLIO"
```

### Portfolio
```
EngineConfigContext (SOT for strategy) → usePortfolioManager
  → holdings + BPS dashboard + wealth metrics (gold + stocks + cash)
  → buy/sell via drawer → dispatch actions → update context
```

### AI Chat
```
FloatingAIChat → user message → askAI() → buildLiveContext(engineConfig, MKT, RS, portfolio)
  → POST /api/ai/chat → system prompt + memory + message
  → provider chain → response → extractToolCalls() → clean text + AIToolCall[]
  → read-only tools execute immediately → actions require [Approve] card
```

### Market Data (Daily Cron)
```
GitHub Actions:
  1. Python fetch_idx80_scan.py → data/idx80_scan.json (live prices 80 tickers)
  2. Python post_process_live_market.py → data/live_market.json
  3. TSX sync-daily-data.ts --force → upsert year files (Yahoo spark EOD)
  4. Python seed-d1.py → D1 production (daily_overview, stock_daily)
  5. curl POST /api/engine/force-sync → CF runIdx80Scan() → D1 idx_scan_data + stock_fundamentals
```
