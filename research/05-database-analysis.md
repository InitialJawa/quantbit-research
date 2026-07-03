# Database Schema Audit — Quantbit v1

**Source:** `db/migrations/0001_init.sql`, `0002_ai_memory.sql`, `0003_market_data.sql`  
**Date:** 2026-07-03

---

## Finding 1: Duplicate Data Across JSON Files, SQLite, and D1

Market data lives in three places:
- `data/years/*.json` — full historical day data (static assets bundled with Cloudflare)
- `data/idx80_scan.json` — IDX80 scan snapshot (static asset)
- `data/historical_market.sqlite` — local SQLite copy (dev)
- `daily_overview` + `stock_daily` + `stock_fundamentals` in D1 (production)

The API has fallback logic: if D1 query fails, it loads from static JSON files. This means the same data exists in 2–3 formats with no single source of truth.

**Recommendation v2:** Drop static JSON files entirely. D1/Database is the sole SOT. The `/api/backtest-data` endpoint should query D1 only, with zero fallback. Pre-seed D1 on deploy via migration or seed script.

---

## Finding 2: JSON Blobs Make Data Non-Queryable

Five columns store opaque JSON blobs:
- `cached_reports.data` — per-ticker report cache
- `engine_state.data` — full engine state JSON
- `idx_scan_data.data` — array of 80+ ticker objects
- `engine_snapshots.data` — historical snapshots
- `users.engine_config`, `users.active_config` — user settings

None of these can be queried with SQL WHERE clauses. `engine_state.data` holds the entire user portfolio, watchlist, config, and trade logs in one cell. `idx_scan_data.data` holds all IDX80 scan results as a JSON string.

**Recommendation v2:** Normalize:
- `engine_state` → separate `user_engine_states` table with columns for each config field
- `idx_scan_data` → `idx80_scans(ticker, scan_date, quality, growth, value, momentum, current_price, ...)` — one row per ticker per scan
- `cached_reports` → `stock_reports(ticker, user_id, report_type, data_json, updated_at)` — keep JSON but add queryable metadata
- `users.engine_config` and `users.active_config` → normalized `user_strategy_configs` table

---

## Finding 3: Missing Foreign Key Constraints

Only user_id → users(id) has FK constraints (with ON DELETE CASCADE). But:
- `stock_daily.ticker` has no FK to `stock_fundamentals.ticker` — orphan tickers can exist
- `trade_logs.ticker` has no FK to any ticker table
- `portfolios.ticker` has no FK
- `watchlists.ticker` has no FK
- No ticker catalog table exists at all

**Recommendation v2:** Create a `tickers(ticker, name, sector, industry, is_active)` catalog table. Add FK constraints from `stock_daily`, `stock_fundamentals`, `portfolios`, `watchlists`, `trade_logs` to `tickers(ticker)`. Enforce referential integrity.

---

## Finding 4: Naming Inconsistency: snake_case in DB, camelCase in Code

Database uses `snake_case`: `buy_price`, `password_hash`, `data_feed`, `active_config`, `engine_config`, `added_at`, `last_updated`.

API responses use `camelCase`: `buyPrice`, `simulationMode`, `singleSellTrigger`, `singleBuyTrigger`, `activeProfileId`.

The PATCH `/api/user/profile` handler (line 333-348) accepts both: it writes `body[key]` where `key` matches DB column names like `data_feed`, but the frontend likely sends camelCase.

**Recommendation v2:** Standardize on `snake_case` in DB (standard SQL convention). Use mapping layer in API handler to convert `camelCase` request bodies → `snake_case` queries. Document the convention in AGENTS.md. Consider naming alignment in `users.theme` and `users.data_feed` too (some are `snake_case`, some are single words).

---

## Finding 5: Derived Data Stored Instead of Computed

`stock_daily.rank_prod` and `stock_daily.rank_res` are pre-computed rank integers. These come from sorting `norm_score` values. The backtest engine sometimes recomputes ranks from `stockNormScores` anyway (core.ts:39-76). This means the same rank exists in two places: the DB column and the runtime computation.

**Recommendation v2:** Drop `rank_prod` and `rank_res` columns. Compute ranks at query time from `norm_score` or `stockNormScores`. If performance is a concern, add a generated column or materialized view.

---

## Finding 6: Unused or Redundant Fields

- `users.cash` — also tracked in `portfolios` table and `engine_state`. Which is the source of truth? The engine state JSON blob contains its own cash field. Three-way redundancy.
- `users.data_feed` — only two choices ("yahoo" / "goapi"), but the app always reads from API. Not referenced in any production code path.
- `simulationMode` singleTicker/singleSellTrigger/singleBuyTrigger — used only for legacy single-stock mode. Backtest still supports it, but no UI exposes it.
- `users.theme` — only affects UI, stored in DB per-user but also in `localStorage`. Not critical to keep in DB.

**Recommendation v2:** Remove `users.cash` (portfolio value comes from positions + prices). Remove `users.data_feed`. Consolidate single-ticker fields into a separate `user_single_stock_configs` table if still needed.

---

## Finding 7: Password Storage — Salt:Hash in Single Field

`users.password_hash` stores `salt:hash` concatenated with a colon separator. The login handler splits on `:` to extract salt. This is a custom scheme instead of using dedicated columns.

**Recommendation v2:** Use separate columns: `users(password_salt TEXT, password_hash TEXT)`. Or better, use a standard hashing library (e.g., `bcrypt`) that stores salt + hash together in a standard format (already self-delimiting). The current PBKDF2 approach is fine cryptographically, but the column name is misleading — it stores `salt:hash`, not just hash.

---

## Finding 8: No Audit Trail for Market Data

`daily_overview`, `stock_daily`, `stock_fundamentals` have no `created_by`, `updated_by`, or version columns. When multiple data pipeline runs insert rows, there's no way to trace which run produced which data.

**Recommendation v2:** Add `created_by TEXT DEFAULT 'system'`, `updated_by TEXT`, and `version INTEGER DEFAULT 1` to market data tables. This enables auditability and conflict resolution when multiple sources write to the same table.

---

## Finding 9: Table Size — stock_daily Grows Unbounded (120,793 rows)

No partition strategy. Every backtest query (`GET /api/backtest-data`) does a full date-range scan across all tickers. With 80 tickers × ~1500 trading days = 120K rows, this is manageable now but will grow year over year.

**Recommendation v2:** Implement date-range partitioning by year. Query pattern is always `WHERE date >= ? AND date <= ? ORDER BY date, ticker`. A partition-per-year would reduce scan size by ~90%.

---

## Finding 10: Missing Indexes

Missing indexes:
- `portfolios(user_id, ticker)` — portfolio lookups by both user and ticker
- `sessions(expires_at)` — session cleanup queries (exists: `idx_sessions_expires` — actually this does exist)
- `daily_overview(date, ihsg_close)` — if queries filter by date range and select ihsg
- `stock_fundamentals(final_score)` — ranking queries sort by `final_score`
- `stock_daily(date, ticker, norm_score)` — covering index for backtest queries

**Recommendation v2:** Add composite indexes matching query patterns. For `stock_daily`, add `(date, norm_score)` to support rank computation.

---

## Finding 11: trade_logs.simulated — Boolean Mixing Real and Simulated

`trade_logs.simulated INTEGER DEFAULT 0` is a boolean flag. Real trades and simulated/backtest trades live in the same table. When the backtest engine logs trades, they're mixed with actual user trades. There's no way to isolate backtest runs from each other.

**Recommendation v2:** Split into two tables: `trade_logs` (user's real trades) and `backtest_logs` (per-backtest-run logs with session_id FK). Add `backtest_logs(backtest_session_id, date, type, ticker, shares, price, message)` with `backtest_sessions(id, user_id, config_snapshot, started_at, ended_at)`. This enables per-run analysis and clean separation.
