# 04 — Data Flow

```
[Yahoo Finance API] ───────────────────────────────────────────────────────┐
  ^JKSE, GC=F, USDIDR=X, 93 IDX tickers                                   │
  yahoo-finance2 (fetch_historical_data.ts)                                │
  yfinance (python, idx80_scan.py)                                         │
  Yahoo Spark API (sync-daily-data.ts)                                     │
                                                                           ▼
┌──────────────────────────────────────────────────────────────────────────────────┐
│                               DATA PIPELINE STEPS                                 │
└──────────────────────────────────────────────────────────────────────────────────┘

STEP 1 — Fetch Historical (one-time / manual via workflow_dispatch)
──────────────────────────────────────────────────────────────────────────────────────
fetch_historical_data.ts (src/scripts/)
  Input:  Yahoo Finance API (historical, chart modules)
  Output: data/historical_market_data.json
          (Array of BacktestDayData: date, ihsgPrice, goldPrice, usdidrRate,
           stockPrices, stockAdjPrices, stockVolumes, stockOpens, stockHighs,
           stockLows, stockRanksProd, stockRanksRes, stockNormScores)
  Format: Single JSON, one array entry per trading day
  Transform:
    - Downloads OHLCV for ^JKSE, GC=F, USDIDR=X + 90 IDX tickers
    - Computes stockNormScores from IDX Warehouse fundamentals (ROE, PB, PE, DER
      → quality, growth, value, momentum via percentile rank 0-95)
    - Falls back to historical yearly averages for gold (HISTORICAL_GOLD_USD_YEARLY)
      and USDIDR (HISTORICAL_USDIDR_YEARLY) with monthly linear interpolation
    - Dividend per share from IDX Warehouse + Yahoo dividend events
    - Pre-2021 data archived (pinned to 2021-01-01 onwards)

STEP 2 — Split Data
──────────────────────────────────────────────────────────────────────────────────────
split-data.ts (src/scripts/)
  Input:  data/historical_market_data.json
  Output: data/years/<year>.json  (one file per year, 2021-2026)
  Format: Array of daily objects with same structure as input
  Purpose: Enables incremental updates per year instead of rewriting 40MB+ JSON

STEP 3 — Build Local SQLite
──────────────────────────────────────────────────────────────────────────────────────
build-db.ts (src/scripts/)
  Input:  data/historical_market_data.json
  Output: data/historical_market.sqlite
  Tables:
    - daily_market (date PK, ihsgPrice, goldPrice, usdidrRate, stockPrices TEXT JSON,
      stockAdjPrices TEXT JSON, stockVolumes TEXT JSON, stockOpens TEXT JSON,
      stockHighs TEXT JSON, stockLows TEXT JSON, stockRanksProd TEXT JSON,
      stockRanksRes TEXT JSON)
    - ai_sessions, ai_messages (mirrors production D1 schema for local AI testing)
  Transform: JSON.stringify all nested objects (since better-sqlite3 stores TEXT)

STEP 4 — Dividend History
──────────────────────────────────────────────────────────────────────────────────────
fetch_dividend_history.ts (src/scripts/)
  Input:  Yahoo Finance historical dividend events
  Output: data/fundamental_snapshots.json
          src/data/dividend_snapshots.json (copied for React import)
  Format: { "BBCA": { "2021": { dividend_per_share: 170 }, ... }, ... }
  Transform: Sums per-ticker dividend events per year

STEP 5A — Daily Cron (GitHub Actions) — Live Market Fetch
──────────────────────────────────────────────────────────────────────────────────────
.daily-data-pipeline.yml  |  schedule: 0 14 * * 1-5 (UTC)

fetch_idx80_scan.py
  Input:  Yahoo Finance (via curl-cffi anti-bot headers)
  Output: data/idx80_scan.json
  Format: { lastUpdated, stocks: [{ ticker, companyName, sector, industry,
           currentPrice, changePercent, peRatio, pbRatio, marketCap, volume,
           dividendYield, fiftyTwoWeekHigh, fiftyTwoWeekLow }] }
  Purpose: Live snapshot of ~80 IDX80 tickers with current prices

post_process_live_market.py
  Input:  data/idx80_scan.json + direct fetch GC=F
  Output: data/live_market.json
  Format: { last_update, ihsg: { value, daily, weekly, monthly },
           usdidr: { value, ... }, gold: { value, ... }, oil: { value, ... },
           stock_prices: { "BBCA": 10250, ... } }
  Purpose: Derives IHSG/gold/USD/IDR from scan data + computes daily/weekly/monthly

STEP 5B — Daily Cron — EOD Sync
──────────────────────────────────────────────────────────────────────────────────────
.daily-data-pipeline.yml  |  schedule: 30 6 * * 1-5 (UTC = 13:30 WIB)

sync-daily-data.ts
  Input:  Yahoo Finance Spark API (batches of 20 tickers)
  Output: data/years/<year>.json  (upserted — merge per-ticker close+volume into
           existing year file for current date)
  Format: Same as historical year files, with date key lookup for merge
  Transform:
    - Fetches `/v8/finance/spark?symbols=<batch>` — returns close + volume
    - Looks up existing year file, updates the date entry
    - If file doesn't exist, creates new one
    - Ticker mapping: clean → Yahoo symbol (".JK" suffix, ^JKSE, GC=F, USDIDR=X)

STEP 6 — Seed Production D1
──────────────────────────────────────────────────────────────────────────────────────
seed-d1.py
  Input:  data/historical_market.sqlite (or live_market.json + idx80_scan.json)
  Output: Cloudflare D1 tables via REST API:
    - daily_overview: date, ihsg_close, gold_close, usdidr_rate, is_market_day
    - stock_daily: date, ticker, close, adj_close, volume, is_carried_forward
    - stock_fundamentals: ticker, quality, growth, value, momentum, norm_scores (JSON),
                          last_updated
  Transform:
    - Batch inserts via D1 batch API (max 100 per batch)
    - Upsert semantics (INSERT OR REPLACE)

STEP 7 — Force Sync (Production)
──────────────────────────────────────────────────────────────────────────────────────
POST /api/engine/force-sync (CF Pages Functions)
  Triggered by: curl at end of daily cron pipeline (with CRON_SECRET)
  Handler: runIdx80Scan() in functions/api/[[path]].ts:1093
  Input:  D1 stock_daily (6mo weekly), D1 daily_overview
  Output: D1 idx_scan_data (full JSON snapshot, replace)
          D1 stock_fundamentals (per-ticker upsert of norm_scores)
  Transform:
    - For each IDX80 ticker, gets 6mo weekly data
    - Computes quality (RSI-based), growth (price momentum), value (P/B inverse),
      momentum (price rate of change)
    - Converts to normalized scores 0-95 via rank-based percentile
    - Writes to stock_fundamentals for use by ranker.ts

┌──────────────────────────────────────────────────────────────────────────────────┐
│                              FRONTEND DATA FLOW                                   │
└──────────────────────────────────────────────────────────────────────────────────┘

App Init:
  src/data/dividend_snapshots.json  →  dividendCache (engine/dividendCache.ts)
  src/data/raw_stocks_data.ts       →  RAW_STOCKS_DATA static array
  src/marketData.ts                 →  L (leaders), PF (profiles), FD (fundamentals)
                                       RS (regime state), MKT (market), CW_AMAN/MAP

  MarketRegimeEngine:
    setIhsgHistory() ← MKT.ihsg.prices from data or local DB
    refreshRSFromRegime() → computes regime → writes RS singleton

  EngineConfigContext:
    localStorage "idx_engine_config" → EngineConfig
    Profiles, strategy settings, backtest config

React Components:
  MarketTab:
    Reads: MKT.ihsg, MKT.gold, MKT.usdidr, L (leaders)
    Displays: market overview charts, stock table, search

  PortfolioTracker:
    Reads: engineConfig, usePortfolioManager (positions, cash, gold)
    Displays: holdings, BPS dashboard, wealth chart, trade log

  SimulationTab (Backtest):
    Reads: data/years/*.json (imported at build time via Vite JSON import)
    Writes: backtestResult to EngineConfigContext
    Sync: syncFromBacktest() → promote to portfolio config

  AnalyticsTab:
    Reads: RS singleton, L, MKT, computeMarketRegime()
    Displays: regime chart, leader/factor tables, notification rules

AI Layer:
  FloatingAIChat → askAI():
    buildLiveContext(engineConfig, MKT, RS, portfolio, alerts, ...)
    → POST /api/ai/chat → system prompt + messages
    ← response + extractToolCalls() → cleanText + AIToolCall[]
    → executeTool() for read-only → buildPendingAction() for actions

┌──────────────────────────────────────────────────────────────────────────────────┐
│                           DATA DUPLICATION POINTS                                 │
└──────────────────────────────────────────────────────────────────────────────────┘

1. Market data exists in THREE places:
   - data/years/*.json (source of truth for backtest)
   - data/historical_market.sqlite (local dev, built from JSON)
   - Cloudflare D1 (production, seeded from SQLite)
   → All derived from same fetch but may diverge if cron fails

2. Dividend data:
   - data/fundamental_snapshots.json (master, generated by fetch_dividend_history.ts)
   - src/data/dividend_snapshots.json (copy for React Vite import)
   → These MUST match. Script copies from data/ to src/data/.

3. Stock fundamentals:
   - src/data/raw_stocks_data.ts (static text table, pre-computed scores)
   - src/marketData.ts L (leader scores, static JSON)
   - D1 stock_fundamentals (computed by force-sync runIdx80Scan())
   → Static data in src/ is from last full build. D1 has latest scores.
   → MKT/RS singletons are loaded from static imports at app init, NOT from D1.

4. EngineConfig:
   - localStorage "idx_engine_config" (browser)
   - Cloudflare D1 users table (synced via PATCH /api/user/profile)
   - Backtest draft config (localStorage, separate key)
   → Three sources that must be reconciled on login

5. IHSG data:
   - MKT.ihsg.prices (in-memory singleton, populated at init)
   - data/live_market.json (daily snapshot from GitHub Actions)
   - D1 daily_overview (production, used by force-sync)
   → MKT.ihsg.prices is the SOT for all decision engines (DB rule from AGENTS.md).

┌──────────────────────────────────────────────────────────────────────────────────┐
│                              FILE FORMATS                                         │
└──────────────────────────────────────────────────────────────────────────────────┘

data/years/<year>.json:
  Array of {
    date: string,           // "2024-01-02"
    ihsgPrice: number,
    goldPrice: number,
    usdidrRate: number,
    stockPrices: { "BBCA": 10250, "BBRI": 5675, ... },
    stockVolumes: { "BBCA": 15000000, ... },
    stockAdjPrices: { "BBCA": 10180, ... },
    stockOpens: { "BBCA": 10200, ... },
    stockHighs: { "BBCA": 10300, ... },
    stockLows: { "BBCA": 10150, ... },
    stockRanksProd: { "BBCA": 3, "BBRI": 7, ... },
    stockRanksRes: { "BBCA": 5, ... },
    stockNormScores: { "BBCA": { quality: 78, growth: 62, value: 45, momentum: 91, dividend: 50 }, ... }
  }

data/idx80_scan.json:
  { lastUpdated: "2026-07-03T14:02:00+07:00",
    stocks: [{ ticker: "BBCA.JK", companyName: "Bank Central Asia", sector: "Banking",
              currentPrice: 10250, changePercent: 1.25, peRatio: 18.5, pbRatio: 3.2,
              marketCap: 1.2e12, volume: 5000000, dividendYield: 3.8,
              fiftyTwoWeekHigh: 11500, fiftyTwoWeekLow: 8500 }] }

data/live_market.json:
  { last_update: "2026-07-03 14:02:00",
    ihsg: { value: 7225.5, daily: 0.45, weekly: -1.2, monthly: 3.1 },
    usdidr: { value: 16250, daily: -0.1, weekly: 0.3, monthly: 0.8 },
    gold: { value: 1350000, daily: 0.2, weekly: 0.5, monthly: 2.0 },
    oil: { value: 78.5, daily: -0.5, weekly: -2.1, monthly: -4.0 },
    stock_prices: { "BBCA": 10250, "BBRI": 5675, ... } }

D1 Tables:
  daily_overview(date TEXT PK, ihsg_close REAL, gold_close REAL, usdidr_rate REAL, is_market_day INTEGER)
  stock_daily(date TEXT, ticker TEXT, close REAL, adj_close REAL, volume INTEGER, is_carried_forward INTEGER)
    PRIMARY KEY(date, ticker)
  stock_fundamentals(ticker TEXT PK, quality REAL, growth REAL, value REAL, momentum REAL,
    norm_scores TEXT JSON, last_updated TEXT)
  idx_scan_data(id INTEGER PK, data TEXT JSON, created_at TEXT)
  users, sessions, portfolios, watchlists, ai_sessions, ai_messages, user_notifications
```
