# Performance Analysis — Quantbit

## Database Bottlenecks

### 1. Missing composite index on `stock_daily`
**Issue**: `stock_daily` table queried by `(date, ticker)` range — no composite index, full table scan per query.
**Impact**: CRITICAL — 120K+ rows scanned for every backtest data fetch.
**Fix**: `CREATE INDEX idx_stock_daily_date_ticker ON stock_daily(date, ticker);`

### 2. No query caching
**Issue**: Every `runStrategy()` call re-queries all market data from D1/API. No memoization or cache layer between identical backtest runs.
**Impact**: HIGH — repeatedly querying same data range wastes DB throughput.
**Fix**: In-memory LRU cache keyed by `(dateRange, universe)`, invalidate on fresh data sync.

### 3. JSON text parsing
**Issue**: `raw_metrics` stored as JSON text column. Parsed per row in API response with `JSON.parse()`.
**Impact**: MEDIUM — adds ~0.5-2ms per row for every read.
**Fix**: Extract frequently-queried fields to typed columns; keep JSON for rare-access metadata.

## Code Bottlenecks

### 1. Monolithic `runStrategy()` (`src/engine/core.ts:27-582`)
**Issue**: 556-line function handling 500+ day loop with nested operations (rank computation, rebalance logic, crash detection, TR/Nav calculation, dividend adjustment, portfolio simulation). Hard to optimize or parallelize.
**Impact**: CRITICAL — single-threaded loop blocking the event loop for seconds.
**Fix**: Break into phases: (a) data fetch + rank precompute, (b) day-loop with minimal branching, (c) post-processing.

### 2. O(n²) history (fixed, but instructive) (`core.ts:157`)
**Issue**: Comment documents prior O(n²) bug: "was O(n²) because we called `filtered.slice(0, stepIndex + 1).map(...)` on every day — ~1.1M iterations for 1500-day backtest." Fixed with rolling window (`ihsgRollingWindow`).
**Impact**: HIGH (was) — serves as warning for similar patterns in codebase.
**Fix**: Audit remaining loops for O(n²) patterns in nested operations.

### 3. Object spreads in loop (`src/engine/core.ts` multiple locations)
**Issue**: `{ ...d }`, `{ ...day.stockRanks }`, `{ ...pos }` spread operators executed on every iteration of 500+ day loop. Each spread creates new objects, triggering GC pressure.
**Impact**: MEDIUM — adds ~10-15% overhead on hot path.
**Fix**: Use mutable updates with typed arrays (`Float64Array`) for numeric fields; reuse object references where safe.

### 4. MKT/RS singletons (`src/marketData.ts`)
**Issue**: Global mutable singletons (`MKT`, `RS`, `L`, `PF`, `FD`) imported directly by 10+ modules. Re-computed on every data feed update regardless of consumer state.
**Impact**: HIGH — any data update cascades through all consumers.
**Fix**: Implement event-based invalidation with lazy recomputation. Convert to explicit dependency injection for testability.

## Memory Bottlenecks

### 1. `RAW_STOCKS_DATA` imported synchronously
**Issue**: `RAW_STOCKS_DATA` (~30 entries in `src/data/raw_stocks_data.ts:1`) imported at module level by `src/stocksData.ts`, `src/mcp/index.ts`. Module-level execution blocks initial render. Enriched at import time from `idx80_scan.json` (JSON import adds synchronous parse cost).
**Impact**: MEDIUM — 30 entries is small, but the import graph blocks app initialization.
**Fix**: Lazy-load with dynamic `import()`, cache after first access.

### 2. 41MB JSON file in memory
**Issue**: `src/data/fundamental_idx_all.json` (41MB, 51,662 entries) loaded into memory. Any consumer that imports this pays the full memory cost.
**Impact**: CRITICAL — 41MB heap allocation on page load. Mobile browsers will OOM or swap.
**Fix**: Server-side only (D1/API endpoint). Never bundle static JSON of this size in client.

### 3. `chartData[]` in runStrategy (`core.ts:151`)
**Issue**: `chartData: ChartPoint[]` grows with day count (one entry per day × number of tracked metrics). Held in memory for entire backtest.
**Impact**: MEDIUM — 500+ entries, each with multiple price points, adds ~50-100KB per call.
**Fix**: Store only summary statistics; compute chart points lazily.

### 4. `logs[]` in memory (`core.ts:152`)
**Issue**: All trade logs stored in array for entire backtest period. `logs: TradeLog[]` grows unbounded with number of trades.
**Impact**: MEDIUM — 500+ trades × 10 fields = ~50KB. Compounds on repeated runs.
**Fix**: Stream logs to callback or ring buffer; keep only last N in memory.

## Render Bottlenecks

### 1. `EngineConfigContext` global state
**Issue**: Single `React.Context` holding all strategy settings. Any state change re-renders all consumers (Portfolio, Backtest, Market, Notifications, AI).
**Impact**: HIGH — cascading re-renders on every profile switch, universe change, or setting toggle.
**Fix**: Split into focused contexts (EngineConfigDataContext, EngineConfigActionsContext) and `React.memo` on leaf consumers.

### 2. `useBuyPressure` recomputation
**Issue**: Hook re-computes buy pressure score on every `activeProfileId` change, irrespective of whether data dependencies changed.
**Impact**: MEDIUM — redundant CPU work on profile tab switch.
**Fix**: Memoize with `useMemo` and explicit dependency list.

### 3. No virtualization
**Issue**: Stock lists, portfolio items rendered as flat arrays. No `react-window`/`react-virtual` for large lists.
**Impact**: MEDIUM — 100+ stock entries rendered as DOM nodes always.
**Fix**: Virtualize any list >20 items.

### 4. `SimulationTab` monolith
**Issue**: Single component handling backtest config, results, charts, trade logs, and controls.
**Impact**: HIGH — large reconciliation cost on any state update.
**Fix**: Split into focused sub-components with `React.memo`.

## Network Bottlenecks

### 1. API monolith (`functions/api/[[path]].ts:1236 lines`)
**Issue**: Single file routing all endpoints. No parallel processing — requests handled sequentially per Worker instance.
**Impact**: HIGH — no parallel data fetching for independent endpoints.
**Fix**: Split into focused modules with parallel fetch coordination in the client.

### 2. Data transfer size
**Issue**: `/api/backtest-data` returns complete JSON of all backtest days. No pagination or field selection.
**Impact**: MEDIUM — large response payloads on mobile networks.
**Fix**: Add `?fields=summary&limit=50` query param support.

### 3. AI provider chain latency
**Issue**: Sequential fallback across 9 models. Worst-case = sum of 9 timeouts (9 × ~30s = 270s).
**Impact**: CRITICAL — user waits minutes for AI failure cascade.
**Fix**: Parallel race with earliest-success. Lower timeouts (5s per attempt). Cap total attempts at 3.

### 4. Yahoo Finance workers (`runIdx80Scan`, `functions/api/[[path]].ts:1093`)
**Issue**: 15 concurrent `fetch()` workers for IDX80 scan. Each fetches 6 months of weekly data. Rate-limiting by Yahoo is unpredictable.
**Impact**: MEDIUM — may trigger Yahoo blocking at ~15 concurrent requests.
**Fix**: Implement request queue with 3-5 concurrent limit and retry backoff.

## Duplicate Processing

### 1. Rank computation in 3 places
**Issue**: Stock ranks computed in:
- Data pipeline (IDX80 scan → `idx_scan_data`)
- Engine (`core.ts` rank reorder loop)
- `marketRegimeEngine.ts` (regime scoring)

**Impact**: HIGH — 3x CPU for same calculation, risk of divergence.
**Fix**: Compute ranks once in data pipeline, store as materialized view in D1, read by all consumers.

### 2. Data transformation chain
**Issue**: JSON → `BacktestDayData` → `bridgeHistoricalData()` → mapped format. Multiple transforms on same raw data.
**Impact**: MEDIUM — each transform copies arrays.
**Fix**: Define a canonical format early; transform at boundary only.

### 3. Price validation duplicated
**Issue**: Same ticker price validated in data loader, engine entry, report generation.
**Impact**: LOW — redundant but safe.
**Fix**: Centralize validation in `validateTickerPrice()`.

### 4. Factor computation in 3+ places
**Issue**: Momentum/quality/growth/value computed from price data in data pipeline, engine, and market regime engine.
**Impact**: HIGH — algorithmic divergence risk (different functions compute slightly different values).
**Fix**: Factor computation lives in one module; consumers import the functions.
