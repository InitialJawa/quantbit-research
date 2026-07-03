# Technical Debt — Quantbit

All items ranked by priority. File paths and line numbers refer to actual codebase.

---

## Critical (P0)

### 1. Monolithic API handler
**File**: `functions/api/[[path]].ts` — 1236 lines
**Issue**: Single file routing ALL endpoints: auth, AI chat, backtest, market data, fundamentals, IDX scan, Gemini legacy, Yahoo proxy, GoAPI proxy, session management. Mixed concerns — every deployment touches this file regardless of change scope.
**Debt**: Cannot independently test or deploy API endpoints. Any change risks regression across unrelated features.
**Fix**: Split into `functions/api/auth/`, `functions/api/ai/`, `functions/api/data/`, `functions/api/engine/` with shared middleware.

### 2. No type safety (`any` types)
**File**: `functions/api/[[path]].ts` — pervasive
**Issue**: All request bodies typed as `any`. All parsed JSON cast to `any`. No Zod/Valibot/Typia validation at the API boundary. Error messages constructed from `.message || .stack || "Unknown"`.
**Debt**: Runtime type errors on malformed input. Impossible to audit request schemas. IDE provides zero autocomplete for response shapes.
**Fix**: Zod schemas at every endpoint. `z.infer<typeof schema>` for response types.

### 3. Dev bypass in production
**File**: `functions/api/[[path]].ts:57-72`
**Issue**: `getUserFromSession()` returns `"dev-user"` for any unauthenticated request. Intentional for dev convenience. Active in production.
**Debt**: Zero authentication enforcement on all endpoints. AI chat costs attributable to no one.
**Fix**: Remove in production via `env.DEV_MODE` gate or environment variable.

### 4. API key in URL parameter
**File**: `functions/api/[[path]].ts:1005`
**Issue**: `api_key=${key}` in URL query parameter for GoAPI. Also line 709: Google Gemini API key in URL.
**Debt**: API keys logged by every proxy, CDN edge, and server in the request chain.
**Fix**: Switch to `Authorization` header. If upstream doesn't support it, proxy through a server-side endpoint.

### 5. Stack trace leakage
**File**: `functions/api/[[path]].ts:699`
**Issue**: `e.stack` sent to client in error responses.
**Debt**: Full internal stack traces exposed to users and attackers.
**Fix**: Mask in production. Log server-side only.

---

## High (P1)

### 6. Regex-based tool calling
**File**: `src/ai/toolCallParser.ts:45-92`
**Issue**: Brace-counting parser with fallback regex for extracting `{"tool_call": {...}}` from AI responses. Fragile with nested JSON, escaped quotes, code fences.
**Debt**: Silent failures (malformed JSON caught and ignored at line 76). Fallback regex (lines 88-89) can strip valid content. No structured output format negotiated with providers.
**Fix**: Use provider-native function calling (OpenRouter `tools` param, Mistral `tool_choice`, Groq `response_format`).

### 7. Triple data sources
**Files**: `src/data/raw_stocks_data.ts`, `src/stocksData.ts`, `src/marketData.ts`, `src/fundamentalsCache.ts`, `src/data/fundamental_idx_all.json`
**Issue**: Stock data loaded from:
1. Static JSON files (`raw_stocks_data.ts`, `fundamental_idx_all.json`)
2. D1 database (`stock_fundamentals`, `idx_scan_data`)
3. Enriched in-memory caches (`PF`, `FD`, `L` singletons)

Fallback chain: static → D1 → estimated. Each source has different fields, formats, and freshness. Consistency not guaranteed.
**Debt**: Data drift between sources. Bugs surface as "why does stock X show different values in Portfolio vs Backtest?"
**Fix**: Single source of truth: load from D1, write pipeline to update D1 periodically. Remove static JSON fallbacks.

### 8. Global mutable singletons
**Files**: `src/marketData.ts` (`PF`, `FD`, `L`, `MKT`, `RS`), `src/stocksData.ts` (`PARSED_KNOWN_STOCKS`)
**Issue**: Module-level mutable state imported by 10+ consumers. Mutations happen at import time, on data feed callbacks, and from market regime engine updates. Order-dependent initialization.
**Debt**: Impossible to isolate for testing. Tests that import these modules share state. Concurrent access unsafe.
**Fix**: Inversion of Control — pass state through context/DI. Side-effect-free initializers.

### 9. No query caching
**Files**: `src/engine/core.ts`, `functions/api/[[path]].ts`
**Issue**: Every backtest run re-queries the entire date range from D1. No memoization layer. Same backtest parameters produce same DB load every time.
**Debt**: DB throughput bottleneck. Multiple tabs/sessions all hit D1.
**Fix**: LRU in-memory cache for market data queries. Invalidate on new data pipeline run.

### 10. Hardcoded ticker lists
**Files**: `src/constants/idx80.ts` (87 tickers across 3 index lists), `src/data/raw_stocks_data.ts` (30 stocks)
**Issue**: Ticker lists hardcoded as string arrays. Fundamental data for 51,662+ stocks in `fundamental_idx_all.json` (41MB) bundled in client. IDX80 membership changes quarterly — these lists must be manually updated.
**Debt**: Stale universe → wrong backtest results. 41MB JSON file in client bundle.
**Fix**: Fetch universe from API/D1 on app load. Remove 41MB JSON from client bundle — serve from API only.

### 11. Bundled dev tools
**Files**: `src/components/AITestHarness.tsx` (dev-only test panel), `src/ai/devMockAI.ts` (pattern-matched mock)
**Issue**: Dev-only code imported in production build. `AITestHarness` exposes 4 test tabs (Tools/Actions/Cooldown/Storage).
**Debt**: Extra bundle weight. Dev mock AI could accidentally match production queries.
**Fix**: Dynamic import with `import.meta.env.DEV` guard.

### 12. No API versioning
**Files**: `functions/api/[[path]].ts`
**Issue**: All endpoints at `/api/<name>` with no version prefix. Breaking changes impossible without coordinated frontend deploy.
**Debt**: Fear of refactoring. Dead endpoints persist (`/api/gemini/*`).
**Fix**: `/api/v1/<name>` prefix strategy.

### 13. Monolithic engine
**File**: `src/engine/core.ts:27-582`
**Issue**: `runStrategy()` is 556 lines handling data fetch, rank computation, portfolio simulation, crash detection, rebalancing, logging, chart generation.
**Debt**: No single piece can be unit-tested in isolation. Logic branches on `simulationMode` (`algo`/`custom`/`adaptive_dca`) with deeply nested conditionals.
**Fix**: Extract into focused modules: `dataLoader.ts`, `rankComputer.ts`, `portfolioSimulator.ts`, `crashDetector.ts`, `rebalanceEngine.ts`.

### 14. Missing database indexes
**Files**: `db/schema.sql` (or equivalent schema definition)
**Issue**: `stock_daily` queried by `(date, ticker)` range but has no composite index. Full table scans for common queries.
**Debt**: Query performance degrades linearly with table growth.
**Fix**: `CREATE INDEX idx_stock_daily_date_ticker ON stock_daily(date, ticker);` Also index `idx_scan_data(updated_at)`, `sessions(user_id, expires_at)`.

### 15. Password storage format
**File**: `functions/api/[[path]].ts`
**Issue**: Salt and hash concatenated in single DB column (`salt:hash`). PBKDF2 100K iterations (below OWASP 600K).
**Debt**: Cannot migrate hashing algorithm without parsing field. Low iteration count weakens brute-force resistance.
**Fix**: Separate `password_salt` + `password_hash` columns. Migrate to Argon2id.

---

## Medium (P2)

### 16. Mixed naming conventions
**Issue**: `camelCase` (TypeScript variables) vs `snake_case` (SQL columns, JSON responses). Inconsistent: `lastBacktestProfile` but `last_rebalance_month` in engine.
**Debt**: Every data boundary requires renaming. Adds cognitive load.
**Fix**: Pick one convention. Translate at serialization boundary.

### 17. No foreign keys
**Files**: D1 schema / SQL schema
**Issue**: All table relationships are implicit. `ai_messages.session_id` references `ai_sessions.id` only by convention. CASCADE deletes rely on application code.
**Debt**: Orphaned rows on delete. Data integrity depends on correct application logic.
**Fix**: Add `FOREIGN KEY ... ON DELETE CASCADE` to D1 schema.

### 18. JSON blobs in tables
**Issue**: At least 4 tables store JSON text columns: `ai_messages.tool_calls`, `ai_messages.metadata`, `idx_scan_data.data` (full scan snapshot), `stock_fundamentals.raw_data`.
**Debt**: Cannot query within JSON without application-side parsing. Column data opaque to migrations.
**Fix**: Normalize frequently-queried fields. Keep JSON for infrequent access.

### 19. Legacy Gemini endpoints
**File**: `functions/api/[[path]].ts:690-691`
**Issue**: `/api/gemini/market-summary` and `/api/gemini/chat` still routed. Code at line 705 (`callAI()`) uses old Gemini API format.
**Debt**: Unmaintained code path. Gemini API may change format.
**Fix**: Remove or mark deprecated. Serve 410 Gone if called.

### 20. Inconsistent error responses
**Files**: `functions/api/[[path]].ts`
**Issue**: 3+ error response formats:
- `{ error: string }` (most endpoints)
- `{ success: false, error: string }` (GoAPI handler)
- `{ ok: false, content, provider, status, diagnostic }` (AI chat)
**Debt**: Client must parse different shapes per endpoint.
**Fix**: Standardize on `{ success: boolean, data?: T, error?: string }`.

### 21. No pagination
**Issue**: List endpoints (`/api/sessions`, `/api/market-data`, `/api/backtest-results`) return all data. No `?page`, `?limit`, `?offset` support.
**Debt**: Growing response payloads. Mobile network strain.
**Fix**: Default page size 25. Cursor-based pagination for large datasets.

### 22. Dev-only code in production
**Files**: `src/ai/devMockAI.ts` (245 lines), `src/components/AITestHarness.tsx`
**Issue**: Mock AI and test harness included in production build.
**Debt**: Bundle bloat. Test harness UI accessible in production.
**Fix**: Dynamic import with `import.meta.env.DEV`.

### 23. 5 AI providers / 9 models
**File**: `src/server/aiChatHandler.ts:468-584`
**Issue**: Provider chain of 9 models across 5 providers. Sequential fallback. Most users need only 2-3.
**Debt**: 7+ API keys to maintain. 9x surface area for failures. Configuration complexity (11 env vars for AI).
**Fix**: Reduce to 2-3 providers (OpenRouter + 1 fallback). Remove direct Groq/Gemini (geo-blocked).

### 24. No responsive design
**Issue**: Components assume desktop viewport. No mobile-first breakpoints. Wide tables overflow on mobile.
**Debt**: Mobile usage growing in Indonesia (primary market).
**Fix**: Implement responsive grid. Horizontal scroll for data tables.

### 25. Magic numbers
**Files**: `src/engine/core.ts`, `src/marketRegimeEngine.ts`, `src/ai/systemKnowledge.ts`
**Issue**: Hardcoded constants: `0.0015`/`0.0025`/`0.0010`/`0.0025` (fees), `0.05` (risk-free rate), `252` (trading days), `60` (days for peak), `0.15` (breadth threshold), `0.30`/`0.25`/`0.15`/`0.20`/`0.10` (BPS weights).
**Debt**: Impossible to configure without code change. No documentation of derivation.
**Fix**: Extract to `src/constants/engine.ts` with named exports and comments.

---

## Low (P3)

### 26. Comments out of sync
**Issue**: Comments referencing deleted code, renamed functions, or outdated behavior. Line count stale in several docstrings.
**Debt**: Misleading documentation. Engineers must read code to override comments.
**Fix**: Remove stale comments. Keep docstrings for public API only.

### 27. No package-lock.json optimization
**Issue**: Full dependency tree without deduplication audit. `npm ls` shows hoisting issues.
**Debt**: Larger bundle, slower installs.
**Fix**: `npm dedupe`, audit with `npm ls --all`.

### 28. console.log in production
**Issue**: `console.log()` statements in production code paths (`functions/api/[[path]].ts`, `src/engine/core.ts`).
**Debt**: Log noise in Cloudflare Worker logs. PII leak risk.
**Fix**: Replace with structured logging or remove.

### 29. Hardcoded strings
**Issue**: Config names (`"Aman"`, `"Agresif"`, `"Dividen"`), error messages, and profile labels hardcoded throughout codebase.
**Debt**: Internationalization impossible without full audit.
**Fix**: Extract to locale/constants module.

### 30. No environment validation
**Issue**: Missing env vars silently fail — `OPENROUTER_API_KEY` unset means OpenRouter providers simply don't appear in chain. No startup validation or clear error.
**Debt**: Deployments with missing keys degrade silently. Debugging requires checking each provider.
**Fix**: Validate all required env vars at server startup. Log missing vars clearly. Fail fast on critical missing keys.
