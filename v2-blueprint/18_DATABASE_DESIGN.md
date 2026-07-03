# 18 — Database Design

> Ideal database schema for QuantBit V2. Designed to eliminate v1's JSON blobs, missing FKs, and naming inconsistencies.

---

## Design Principles

1. **No JSON blobs** — Every entity has its own table with typed columns
2. **Foreign keys everywhere** — No orphaned references
3. **snake_case** — Consistent naming for all tables and columns
4. **Audit trail** — Every table has `created_at`, `updated_at`
5. **Indexes match queries** — Composite indexes for common query patterns

---

## Schema

### Ticker Catalog (NEW — didn't exist in V1)

```sql
CREATE TABLE tickers (
  ticker        TEXT PRIMARY KEY,  -- 4-letter IDX code (e.g., BBCA)
  name          TEXT NOT NULL,     -- Company name
  sector        TEXT NOT NULL,     -- IDX-IC sector
  industry      TEXT NOT NULL,     -- IDX-IC industry
  is_active     INTEGER DEFAULT 1, -- Still listed
  is_idx80      INTEGER DEFAULT 0, -- Currently in IDX80
  listed_date   TEXT,              -- IPO date
  created_at    TEXT DEFAULT (datetime('now')),
  updated_at    TEXT DEFAULT (datetime('now'))
);

CREATE INDEX idx_tickers_sector ON tickers(sector);
CREATE INDEX idx_tickers_idx80 ON tickers(is_idx80, is_active);
```

### Users (Better Auth managed)

```sql
-- Better Auth manages its own schema
-- Minimal custom user table if needed
CREATE TABLE user_profiles (
  id            TEXT PRIMARY KEY,
  display_name  TEXT,
  theme         TEXT DEFAULT 'terminal',
  created_at    TEXT DEFAULT (datetime('now')),
  updated_at    TEXT DEFAULT (datetime('now'))
);
```

### Market Data

```sql
-- Daily market overview (replaces daily_overview)
CREATE TABLE market_daily (
  date          TEXT NOT NULL,      -- ISO date "2024-01-02"
  ihsg_close    REAL NOT NULL,     -- IHSG closing value
  ihsg_open     REAL,
  ihsg_high     REAL,
  ihsg_low      REAL,
  gold_close    REAL NOT NULL,     -- Gold price (IDR)
  usdidr_rate   REAL NOT NULL,     -- USD/IDR exchange rate
  is_market_day INTEGER DEFAULT 1, -- 0 = holiday/weekend
  created_at    TEXT DEFAULT (datetime('now')),
  PRIMARY KEY (date)
);

CREATE INDEX idx_market_daily_date ON market_daily(date);

-- Per-stock daily data (replaces stock_daily)
CREATE TABLE stock_daily (
  date          TEXT NOT NULL,
  ticker        TEXT NOT NULL,
  close         REAL NOT NULL,
  adj_close     REAL NOT NULL,
  open          REAL,
  high          REAL,
  low           REAL,
  volume        BIGINT,
  is_carried_forward INTEGER DEFAULT 0,
  created_at    TEXT DEFAULT (datetime('now')),
  PRIMARY KEY (date, ticker),
  FOREIGN KEY (ticker) REFERENCES tickers(ticker)
);

CREATE INDEX idx_stock_daily_date ON stock_daily(date);
CREATE INDEX idx_stock_daily_ticker ON stock_daily(ticker);
CREATE INDEX idx_stock_daily_composite ON stock_daily(date, ticker);

-- Stock fundamentals / norm scores (replaces stock_fundamentals)
CREATE TABLE stock_scores (
  ticker        TEXT NOT NULL,
  score_date    TEXT NOT NULL,      -- Date these scores were valid
  quality       REAL,               -- 0-95 normalized
  growth        REAL,
  value         REAL,
  momentum      REAL,
  dividend      REAL,
  final_score   REAL GENERATED ALWAYS AS (
    COALESCE(quality, 50) + COALESCE(growth, 50) + 
    COALESCE(value, 50) + COALESCE(momentum, 50) + 
    COALESCE(dividend, 50)
  ) STORED,
  source        TEXT DEFAULT 'pipeline',  -- pipeline / force-sync
  created_at    TEXT DEFAULT (datetime('now')),
  PRIMARY KEY (ticker, score_date),
  FOREIGN KEY (ticker) REFERENCES tickers(ticker)
);

CREATE INDEX idx_stock_scores_date ON stock_scores(score_date);
CREATE INDEX idx_stock_scores_final ON stock_scores(final_score DESC);

-- IDX80 scan snapshots (replaces idx_scan_data JSON blob)
CREATE TABLE idx80_scans (
  ticker        TEXT NOT NULL,
  scan_date     TEXT NOT NULL,      -- ISO datetime
  current_price REAL,
  change_pct    REAL,
  pe_ratio      REAL,
  pb_ratio      REAL,
  market_cap    BIGINT,
  volume        BIGINT,
  dividend_yield REAL,
  week_52_high  REAL,
  week_52_low   REAL,
  created_at    TEXT DEFAULT (datetime('now')),
  PRIMARY KEY (ticker, scan_date),
  FOREIGN KEY (ticker) REFERENCES tickers(ticker)
);

CREATE INDEX idx_idx80_scans_date ON idx80_scans(scan_date);
```

### Portfolio

```sql
CREATE TABLE portfolios (
  id            TEXT PRIMARY KEY,
  user_id       TEXT NOT NULL,
  ticker        TEXT NOT NULL,
  shares        INTEGER NOT NULL DEFAULT 0,
  buy_price     REAL NOT NULL,      -- Average buy price
  added_at      TEXT DEFAULT (datetime('now')),
  updated_at    TEXT DEFAULT (datetime('now')),
  FOREIGN KEY (user_id) REFERENCES user_profiles(id),
  FOREIGN KEY (ticker) REFERENCES tickers(ticker),
  UNIQUE(user_id, ticker)
);

CREATE INDEX idx_portfolios_user ON portfolios(user_id, ticker);

CREATE TABLE cash_holdings (
  user_id       TEXT PRIMARY KEY,
  cash_amount   REAL NOT NULL DEFAULT 0,
  gold_grams    REAL NOT NULL DEFAULT 0,
  updated_at    TEXT DEFAULT (datetime('now')),
  FOREIGN KEY (user_id) REFERENCES user_profiles(id)
);

CREATE TABLE watchlists (
  user_id       TEXT NOT NULL,
  ticker        TEXT NOT NULL,
  added_at      TEXT DEFAULT (datetime('now')),
  PRIMARY KEY (user_id, ticker),
  FOREIGN KEY (user_id) REFERENCES user_profiles(id),
  FOREIGN KEY (ticker) REFERENCES tickers(ticker)
);
```

### Trades (Split from V1)

```sql
-- Real trades (user actions)
CREATE TABLE trade_logs (
  id            TEXT PRIMARY KEY,
  user_id       TEXT NOT NULL,
  ticker        TEXT NOT NULL,
  action        TEXT NOT NULL CHECK(action IN ('buy', 'sell', 'gold_buy', 'gold_sell')),
  shares        INTEGER,              -- NULL for gold trades
  grams         REAL,                 -- NULL for stock trades
  price         REAL NOT NULL,
  total         REAL NOT NULL,        -- shares * price (or grams * price)
  fee           REAL DEFAULT 0,
  tax           REAL DEFAULT 0,
  executed_at   TEXT DEFAULT (datetime('now')),
  FOREIGN KEY (user_id) REFERENCES user_profiles(id),
  FOREIGN KEY (ticker) REFERENCES tickers(ticker)
);

CREATE INDEX idx_trade_logs_user ON trade_logs(user_id, executed_at DESC);

-- Backtest sessions
CREATE TABLE backtest_sessions (
  id            TEXT PRIMARY KEY,
  user_id       TEXT NOT NULL,
  config_snapshot TEXT NOT NULL,     -- JSON of backtest config
  date_start    TEXT NOT NULL,       -- Backtest date range
  date_end      TEXT NOT NULL,
  status        TEXT DEFAULT 'running' CHECK(status IN ('running', 'completed', 'failed')),
  results_json  TEXT,                -- Full results (JSON)
  started_at    TEXT DEFAULT (datetime('now')),
  completed_at  TEXT,
  FOREIGN KEY (user_id) REFERENCES user_profiles(id)
);

CREATE INDEX idx_backtest_sessions_user ON backtest_sessions(user_id, started_at DESC);

-- Backtest trade logs (isolated from real trades)
CREATE TABLE backtest_logs (
  id              TEXT PRIMARY KEY,
  session_id      TEXT NOT NULL,
  date            TEXT NOT NULL,
  action          TEXT NOT NULL CHECK(action IN ('buy', 'sell', 'gold_buy', 'gold_sell', 'dividend', 'rebalance')),
  ticker          TEXT,
  shares          INTEGER,
  price           REAL,
  total           REAL,
  message         TEXT,
  FOREIGN KEY (session_id) REFERENCES backtest_sessions(id),
  FOREIGN KEY (ticker) REFERENCES tickers(ticker)
);

CREATE INDEX idx_backtest_logs_session ON backtest_logs(session_id, date);
```

### Strategy Configuration (Replaces JSON blobs)

```sql
CREATE TABLE strategy_profiles (
  id              TEXT PRIMARY KEY,  -- 'aman', 'agresif', 'dividen', or 'custom_xxx'
  user_id         TEXT,
  name            TEXT NOT NULL,
  weight_quality  REAL NOT NULL DEFAULT 0.25,
  weight_growth   REAL NOT NULL DEFAULT 0.25,
  weight_value    REAL NOT NULL DEFAULT 0.25,
  weight_momentum REAL NOT NULL DEFAULT 0.25,
  weight_dividend REAL NOT NULL DEFAULT 0.00,
  is_default      INTEGER DEFAULT 0,
  created_at      TEXT DEFAULT (datetime('now')),
  updated_at      TEXT DEFAULT (datetime('now')),
  FOREIGN KEY (user_id) REFERENCES user_profiles(id),
  CHECK (ABS(weight_quality + weight_growth + weight_value + weight_momentum + weight_dividend - 1.0) < 0.01)
);

CREATE TABLE user_strategy_configs (
  user_id             TEXT PRIMARY KEY,
  active_profile_id   TEXT NOT NULL DEFAULT 'aman',
  simulation_mode     TEXT DEFAULT 'algo' CHECK(simulation_mode IN ('algo', 'custom', 'adaptive_dca')),
  top_n_count         INTEGER DEFAULT 5,
  crash_sensitivity   REAL DEFAULT 10,
  safe_haven_asset    TEXT DEFAULT 'emas' CHECK(safe_haven_asset IN ('emas', 'cash')),
  reserve_buffer_pct  REAL DEFAULT 50,
  enable_crossover    INTEGER DEFAULT 1,
  dca_active          INTEGER DEFAULT 1,
  universe            TEXT DEFAULT 'idx80',  -- 'idx80', 'custom', 'all'
  custom_universe     TEXT,                  -- JSON array of tickers
  data_feed           TEXT DEFAULT 'yahoo',
  updated_at          TEXT DEFAULT (datetime('now')),
  FOREIGN KEY (user_id) REFERENCES user_profiles(id)
);
```

### AI

```sql
CREATE TABLE ai_sessions (
  id            TEXT PRIMARY KEY,
  user_id       TEXT NOT NULL,
  title         TEXT,
  created_at    TEXT DEFAULT (datetime('now')),
  updated_at    TEXT DEFAULT (datetime('now')),
  FOREIGN KEY (user_id) REFERENCES user_profiles(id)
);

CREATE TABLE ai_messages (
  id            TEXT PRIMARY KEY,
  session_id    TEXT NOT NULL,
  role          TEXT NOT NULL CHECK(role IN ('user', 'assistant', 'system')),
  content       TEXT NOT NULL,
  tool_calls    TEXT,              -- JSON array of tool calls
  tool_results  TEXT,              -- JSON array of results
  created_at    TEXT DEFAULT (datetime('now')),
  FOREIGN KEY (session_id) REFERENCES ai_sessions(id)
);

CREATE INDEX idx_ai_messages_session ON ai_messages(session_id, created_at);
```

### Notifications

```sql
CREATE TABLE notification_rules (
  id            TEXT PRIMARY KEY,
  user_id       TEXT NOT NULL,
  rule_type     TEXT NOT NULL CHECK(rule_type IN ('ticker_out_of_topn', 'crash_protection', 'universe_breach', 'custom')),
  ticker        TEXT,              -- Optional ticker filter
  threshold     REAL,              -- Rule-specific threshold
  enabled       INTEGER DEFAULT 1,
  created_at    TEXT DEFAULT (datetime('now')),
  FOREIGN KEY (user_id) REFERENCES user_profiles(id),
  FOREIGN KEY (ticker) REFERENCES tickers(ticker)
);

CREATE TABLE user_notifications (
  id            TEXT PRIMARY KEY,
  user_id       TEXT NOT NULL,
  rule_id       TEXT,
  type          TEXT NOT NULL,
  message       TEXT NOT NULL,
  is_read       INTEGER DEFAULT 0,
  created_at    TEXT DEFAULT (datetime('now')),
  FOREIGN KEY (user_id) REFERENCES user_profiles(id),
  FOREIGN KEY (rule_id) REFERENCES notification_rules(id)
);
```

---

## Migration Plan (V1 → V2)

| V1 Table | V2 Table | Migration Strategy |
|----------|----------|-------------------|
| daily_overview | market_daily | Direct mapping (rename columns) |
| stock_daily | stock_daily | Drop rank_prod/rank_res, add FK |
| stock_fundamentals | stock_scores | Normalize blob, add score_date |
| idx_scan_data | idx80_scans | Normalize JSON array to rows |
| users.engine_config | user_strategy_configs | Split JSON blob to columns |
| users.active_config | user_strategy_configs | Same as above |
| users.cash, users.theme, etc. | cash_holdings, user_profiles | Split across tables |
| portfolios | portfolios | Add FK to tickers |
| trade_logs | trade_logs + backtest_logs | Split by simulated flag |
| watchlists | watchlists | Add FK to tickers |
| engine_state | backtest_sessions | Separate table |
| (none) | tickers | New — seed from IDX80 list |

---

## Index Strategy

| Table | Index | Query Pattern |
|-------|-------|---------------|
| stock_daily | `(date, ticker)` | Backtest date-range queries |
| stock_daily | `(date, final_score)` | Rank computation |
| tickers | `(is_idx80, is_active)` | IDX80 list queries |
| market_daily | `(date)` | Date-range queries |
| stock_scores | `(score_date, final_score DESC)` | Latest scores + ranking |
| idx80_scans | `(scan_date)` | Latest snapshot |
| portfolios | `(user_id, ticker)` | Portfolio lookups |
| trade_logs | `(user_id, executed_at DESC)` | User trade history |
| backtest_logs | `(session_id, date)` | Backtest log queries |

---

## Data Types Convention

| SQL Type | Usage | Reason |
|----------|-------|--------|
| TEXT | All IDs, dates, VARCHAR-typed data | SQLite has no native DATE/DATETIME |
| REAL | Prices, scores, percentages | Floating point precision needed |
| INTEGER | Counts, flags, booleans | SQLite native integer |
| BIGINT | Volume, market cap | Can exceed INTEGER range |
| BLOB | Binary data only | Not used for text/JSON |

**Rule:** JSON is stored as TEXT only when:
- The structure is truly dynamic (tool_calls, tool_results)
- Metadata columns allow querying
- Never for core business entities (portfolio, config, scans)
