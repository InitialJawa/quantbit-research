# 08 — Technical Debt

> All technical debt identified in QuantBit v1, categorized by severity.

---

## P0 — Critical (Must Fix Before V2 MVP)

| # | Debt | Location | Impact | Fix |
|---|------|----------|--------|-----|
| 1 | **Monolithic API** | `functions/api/[[path]].ts` (1236 lines) | All routes in one file; cannot test, extend, or maintain | Hono domain modules |
| 2 | **No input validation** | All API endpoints `request.json() as any` | Malformed requests crash or write bad data | Zod schemas on every endpoint |
| 3 | **Dev bypass in production auth** | `getUserFromSession()` returns "dev-user" on any error | Unauthenticated access to AI and data endpoints | Remove bypass, return 401 |
| 4 | **API key in URL** | `?api_key=${key}` (line 1005) | Key logged everywhere | Authorization header |
| 5 | **Stack trace in error response** | `e.message || e.stack || "Unknown"` (line 699) | Information disclosure | Generic error messages |

## P1 — High (Fix in V2)

| # | Debt | Location | Impact | Fix |
|---|------|----------|--------|-----|
| 6 | **Triple data sources** | JSON + SQLite + D1 + in-memory | Inconsistent data across components | D1 as single SOT |
| 7 | **JSON blobs in DB** | 5 columns: `engine_state`, `idx_scan_data`, `cached_reports`, `engine_snapshots`, `users.engine_config` | Cannot query, index, or validate | Normalize to separate tables |
| 8 | **Missing FKs** | `stock_daily`, `portfolios`, `watchlists`, `trade_logs` | Orphan records, no referential integrity | Create ticker catalog table + FK constraints |
| 9 | **Regex-extracted AI output** | `toolCallParser.ts`, `cleanText.ts` | Fragile — format change breaks parsing | Structured AI output (JSON mode) |
| 10 | **Dual scoring pipelines** | `fetch_historical_data.ts` vs `runIdx80Scan()` | Different algorithms produce different scores | Unified server-side scoring |
| 11 | **Build-time JSON imports** | `data/years/*.json` via Vite import | Data bundled with app, can't update without redeploy | API-backed data, D1 as source |
| 12 | **Hidden business rules** | 50+ items undocumented | New developers will misinterpret | Document all rules, make configurable |
| 13 | **Hardcoded ticker list** | `runIdx80Scan()`, `idx80.ts` | Stale after 6 months | Database-backed ticker catalog |
| 14 | **Hardcoded profile weights** | 3+ files | Update one, forget the others | Database-stored profiles |
| 15 | **Inconsistent error responses** | 3 different error envelope patterns | Clients must handle all 3 | Standard `{ok, data, error}` envelope |

## P2 — Medium (Fix During V2)

| # | Debt | Location | Impact | Fix |
|---|------|----------|--------|-----|
| 16 | **No pagination** | `/api/backtest-data`, `/api/trade-logs` | 120K rows in one response | Page + limit parameters |
| 17 | **No API versioning** | All at `/api/...` | Breaking changes affect all clients | `/api/v1/` prefix |
| 18 | **Naming inconsistency** | snake_case in DB, camelCase in API | Confusion, potential mapping bugs | Standardize with mapper layer |
| 19 | **Derived data stored** | `rank_prod`, `rank_res` in DB | Same data in DB + runtime | Compute at query time |
| 20 | **trade_logs.simulated flag** | Real + simulated trades mixed | Can't isolate backtest runs | Separate tables: `trade_logs` + `backtest_logs` |
| 21 | **No audit trail** | Market data tables | Can't trace which run produced data | Add version/created_by columns |
| 22 | **Duplicate AI handlers** | Unified `/api/ai/chat` + legacy Gemini | Unused code, maintenance burden | Remove legacy handlers |
| 23 | **Unused DB fields** | `users.cash`, `users.data_feed`, `users.theme` | Clutter, confusion | Remove unused columns |
| 24 | **CORS not explicit** | Missing CORS headers | Cross-origin requests fail | Add explicit CORS middleware |
| 25 | **Hardcoded default state** | GET /api/engine/state returns BBCA/BBRI fake data | Misleads users | Return 404, meaningful error |

## P3 — Low (Nice-to-Have)

| # | Debt | Location | Impact | Fix |
|---|------|----------|--------|-----|
| 26 | **No indexes** | Missing composites on `(date, ticker)`, `(user_id, ticker)` | Slow queries on large datasets | Add composite indexes matching query patterns |
| 27 | **No partitioning** | `stock_daily` at 120K rows, growing unbounded | Performance degrades yearly | Partition by year |
| 28 | **Password storage format** | `salt:hash` in single column | Non-standard, misleading column name | Use bcrypt or separate columns |
| 29 | **Library versions not pinned** | `package.json` uses `^` ranges | Builds may break with minor updates | Pin exact versions |
| 30 | **No environment validation** | Dotenv reads fail silently | App runs with missing config | Validate env vars at startup |

---

## Technical Debt That Resolves Itself in V2

These debts exist because V1 was the first iteration. V2's architecture eliminates them by design:

| Debt | How V2 Eliminates It |
|------|---------------------|
| Monolithic API | Hono domain modules |
| Triple data sources | D1 single SOT |
| No validation | Zod at every boundary |
| JSON blobs | Normalized schema |
| Missing FKs | Ticker catalog table |
| Build-time imports | API-backed data |
| Regex AI parsing | Structured output |
| Hardcoded profiles | DB-stored profiles |
| Inconsistent errors | Standard envelope |
| No pagination | Default + configurable limits |

---

## Technical Debt Cost Estimate

| Severity | Count | Estimated Fix Cost (Developer Days) |
|----------|-------|-------------------------------------|
| P0 | 5 | 2 days |
| P1 | 10 | 5 days |
| P2 | 10 | 3 days |
| P3 | 5 | 1 day |
| **Total** | **30** | **11 days** |

**Note:** These costs are NOT meaningful for V2 because V2 is a full rewrite. These debts are eliminated automatically by the new architecture. The cost is included in the rebuild estimate.
