# 12 — Data Architecture

> Data architecture for QuantBit V2, designed to avoid the triple-data-source problem of v1.

---

## Architecture Principle

**Single Source of Truth (SOT):** ALL data lives in D1. No JSON files, no SQLite copies, no in-memory duplicates. Every component reads from D1 via the API.

```
                     ┌─────────────────────┐
                     │   External Sources   │
                     │  (Yahoo, GoAPI, AI)  │
                     └──────────┬──────────┘
                                │
                     ┌──────────▼──────────┐
                     │   Data Pipeline      │
                     │  (writes to D1)     │
                     └──────────┬──────────┘
                                │
                     ┌──────────▼──────────┐
                     │   D1 Database        │ ←── SOLE SOURCE OF TRUTH
                     │  (Cloudflare)        │
                     └──────────┬──────────┘
                     ┌──────────┼──────────┐
                     │          │          │
              ┌──────▼───┐ ┌───▼────┐ ┌───▼──────┐
              │  API      │ │  KV    │ │  Workers │
              │  (Hono)   │ │ (Cache)│ │ (Engine) │
              └──────┬───┘ └────────┘ └──────────┘
                     │
              ┌──────▼───┐
              │  Frontend │
              │  (React)  │
              └──────────┘
```

---

## Data Flow Diagram

### Historical Data Pipeline
```
Yahoo Finance (historical)
  → scripts/ (unified pipeline)
    → D1: daily_overview (upsert)
    → D1: stock_daily (upsert)
    → D1: stock_fundamentals (recompute scores)
```

### Daily Data Pipeline
```
GitHub Actions (cron):
  Yahoo Finance (spark/EOD)
    → scripts/ (unified pipeline)
      → D1: daily_overview (today upsert)
      → D1: stock_daily (today upsert)
      → D1: stock_fundamentals (recompute active tickers)
```

### Backtest Data Flow (V2 — New)
```
User clicks "Jalankan Backtest" in UI
  → POST /api/v1/engine/backtest { config, profile, dateRange }
    → Hono handler queries D1: SELECT * FROM stock_daily WHERE date BETWEEN ? AND ?
    → Hono handler queries D1: SELECT * FROM stock_fundamentals
    → Hono handler runs engine (pure TS on edge)
    → Stores result in D1: backtest_sessions + backtest_logs
    → Returns result summary + chart data to frontend
```

**Compare with V1:** Backtest was client-side, reading from JSON imports. V2 runs server-side, reading from D1.

### Live Portfolio Flow
```
User loads Portfolio tab
  → GET /api/v1/portfolio/{userId}
    → D1 query: portfolios, cash, gold, trade_logs
  → GET /api/v1/market/overview
    → D1 query: latest daily_overview + idx80_scans
  → GET /api/v1/engine/bps
    → D1 query: regime state + market data
    → Compute BPS server-side
    → Return BPS + action recommendation

User executes buy/sell
  → POST /api/v1/portfolio/trade { ticker, action, shares }
    → Validate against D1 (current prices, portfolio state)
    → Execute trade
    → Write to D1: portfolios, trade_logs
    → Return updated portfolio
```

---

## Data Categories

### Market Data (D1 SOT)
| Category | Source | Update Frequency | V1 Problem | V2 Fix |
|----------|--------|-----------------|------------|--------|
| Daily OHLCV | Yahoo EOD | Daily (cron) | JSON + SQLite + D1 | D1 only |
| IHSG history | Derived from ^JKSE | Daily | MKT singleton + D1 | D1 only |
| IDX80 scan | Yahoo live | Daily | JSON + D1 blob | D1 normalized |
| Stock scores | Computed | Daily (force-sync) | Static + D1 + JSON | D1 only, unified algorithm |
| Ticker list | DB-administered | Semi-annual | Hardcoded in 2 places | DB table + API |

### User Data (D1 SOT)
| Category | Storage | V1 Problem | V2 Fix |
|----------|---------|------------|--------|
| Profile | users table | N/A | Better Auth managed |
| Portfolio | portfolios + cash + gold | Triple redundancy (users.cash + portfolios + engine_state) | Single normalized schema |
| Watchlist | watchlists | Missing FK | FK to tickers table |
| Trade logs | trade_logs | Mixed real/simulated | Split: trade_logs + backtest_logs |
| Config | user_strategy_configs | JSON blob in users table | Normalized table |
| AI sessions | ai_sessions + ai_messages | localStorage + D1 | D1 SOT, localStorage cache |

### Cache Data (KV)
| Category | TTL | Rationale |
|----------|-----|-----------|
| Market regime state | 5 min | Computed frequently, changes slowly |
| Ticker list | 1 hour | Changes semi-annually |
| Stock fundamentals | 1 hour | Updated daily |
| User session | Session lifetime | Auth optimization |

---

## Data Freshness (DataStatus)

Every API response that includes market data must include a `dataStatus` field:

```typescript
type DataStatus = "LIVE" | "CACHED" | "STALE" | "ESTIMATED";

interface MarketDataResponse {
  data: T;
  dataStatus: DataStatus;
  lastUpdated: string;  // ISO 8601
  nextUpdate: string;   // Estimated next refresh
}
```

**Freshness Rules:**
- LIVE: Data fetched within last 5 minutes
- CACHED: Data served from D1, last sync within 24 hours
- STALE: Data older than 24 hours
- ESTIMATED: Synthetic/interpolated data (no direct source)

---

## Data Volume Estimates (V2)

| Table | Est. Rows | Growth Rate | Query Pattern |
|-------|-----------|-------------|---------------|
| stock_daily | 120K (80 × 1500 days) | +80 rows/day | Date-range scans |
| daily_overview | 1.5K (1500 days) | +1 row/day | Date-range scans |
| stock_fundamentals | 800 | Static | Point lookups by ticker |
| idx80_scans | 80 | 80 rows/scan (replace) | Full table scan |

**Optimization Strategy:**
- Partition `stock_daily` by year (D1 doesn't support native partitioning, use separate tables or date-based queries)
- Composite index on `stock_daily(date, ticker)` for backtest queries
- Composite index on `stock_daily(date, norm_score)` for rank computation
- Index on `daily_overview(date)` for time-series queries
