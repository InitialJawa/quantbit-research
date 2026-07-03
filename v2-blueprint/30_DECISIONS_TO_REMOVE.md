# 30 — Decisions to Remove

> Decisions from QuantBit v1 that must NOT be repeated in V2.

---

## Architectural Anti-Patterns

| # | Decision | Why Remove | V1 Evidence |
|---|----------|------------|-------------|
| 1 | **Single monolithic API file** | Cannot test, maintain, or extend | `functions/api/[[path]].ts` — 1236 lines |
| 2 | **Triple data sources (JSON + SQLite + D1)** | Same data diverges across sources | `20-data-sources-inventory.md` — 10 categories |
| 3 | **Build-time JSON imports for runtime data** | Data bundled with app, can't update without redeploy | `data/years/*.json` imported by Vite |
| 4 | **In-memory singletons (MKT, RS, L)** | Different data than DB | `src/marketData.ts` — loaded at init |
| 5 | **JSON blobs in relational DB** | Can't query with SQL | 5 columns: engine_state, idx_scan_data, etc. |
| 6 | **No input validation** | Malformed requests crash or write bad data | All endpoints use `request.json() as any` |
| 7 | **No error standardization** | Clients handle 3 different error formats | `{error:...}` vs `{content:...}` vs `{success:false}` |
| 8 | **No API versioning** | Breaking changes affect all clients simultaneously | All at `/api/...` with no prefix |
| 9 | **No pagination** | 120K rows in single response | `/api/backtest-data`, `/api/trade-logs` |
| 10 | **Hardcoded default responses** | Misleads users with fake data | GET /api/engine/state returns BBCA/BBRI |

## Security Anti-Patterns

| # | Decision | Why Remove | V1 Evidence |
|---|----------|------------|-------------|
| 11 | **Dev bypass returns "dev-user" on any error** | Unauthenticated access to production | `getUserFromSession()` — no token → "dev-user" |
| 12 | **API key in URL query string** | Logged everywhere | `?api_key=${key}` — line 1005 |
| 13 | **Stack trace in error response** | Information disclosure | `e.message || e.stack || "Unknown"` — line 699 |
| 14 | **Cron secret in POST body** | Visible in request logs | `body.secret === env.CRON_SECRET` — line 617 |
| 15 | **CORS not configured** | Cross-origin requests fail | No CORS headers in API responses |
| 16 | **Rate limiting absent** | Abuse possible | No rate limit logic anywhere |

## Algorithm Errors

| # | Decision | Why Remove | Evidence |
|---|----------|------------|-------------|
| 17 | **252 trading days for volatility** | Wrong for IDX (should be 247) | `metrics.ts:63` |
| 18 | **5% risk-free rate hardcoded** | Historical rates varied (3.5-6%) | `metrics.ts:70` |
| 19 | **Dual scoring pipelines** | Different algorithms produce different scores | `fetch_historical_data.ts` vs `runIdx80Scan()` |
| 20 | **Dividend date heuristic (June 15)** | Only accurate for some tickers | `core.ts:243` — many pay at different times |
| 21 | **Rank key fallback** | Custom profiles fallback to AMAN ranks | `core.ts:36-37` |
| 22 | **Synthetic fundamentals for 95% of tickers** | Only 5 of 830 tickers have real data | `19-hidden-knowledge.md:2.5` |
| 23 | **Average portfolio value = (start + end)/2** | Inaccurate for non-linear returns | `metrics.ts:75` |
| 24 | **Carried-forward data inconsistency** | Sometimes used, sometimes skipped | ADR-010 |
| 25 | **Missing dividend score = 0** (not 50) | Inconsistent with other factors (default=50) | `ranker.ts:17` |

## Process Anti-Patterns

| # | Decision | Why Remove | Evidence |
|---|----------|------------|-------------|
| 26 | **No architecture plan at start** | System grew organically, introducing tech debt | History of v1 evolution |
| 27 | **No testing** | Changes validated manually | No test directory in v1 |
| 28 | **Python + TypeScript in same pipeline** | Two languages, double maintenance | Python: `fetch_idx80_scan.py`, TS: `sync-daily-data.ts` |
| 29 | **Code comments instead of documentation** | 50 hidden business rules | `19-hidden-knowledge.md` |
| 30 | **Single branch development** | No review process | No PRs, no staging branch |

## Duplicated Code (Remove the Pattern)

| # | Duplicate | Location Count | Impact |
|---|-----------|---------------|--------|
| 31 | Profile weights (AMAN/AGRESIF/DIVIDEN) | 3+ files | Update → forget → bug |
| 32 | Ticker list (IDX80) | 2 files | One gets updated, other stale |
| 33 | Gold/USD yearly averages | 2 locations | Pipeline + seed script |
| 34 | Scoring algorithm | 2 locations | Different results |
| 35 | Fee definitions | 2+ locations | Configuration drift |
| 36 | Norm score ranges (0-95 assumption) | Multiple | Scores can be outside range |
| 37 | AI provider model names | 2 locations | Change model → update both |
| 38 | Backtest vs live config comparison | Multiple | JSON.stringify key comparison |
| 39 | Cash balance tracking | 3 places | users.cash + portfolios + engine_state |
| 40 | AI session memory | 2 places | localStorage + D1 |

---

## Removal Priority

| Priority | Removals | Reason |
|----------|----------|--------|
| **P0 (Must remove)** | 1, 2, 6, 11, 12, 13, 15, 26, 27 | Blockers for production quality |
| **P1 (Should remove)** | 3, 4, 5, 7, 8, 9, 14, 16, 17, 18, 19, 28 | Causes bugs or maintenance burden |
| **P2 (Good to remove)** | 10, 20, 21, 22, 23, 24, 25, 29, 30 | Quality of life improvements |

**Note:** All of these are automatically eliminated by V2's new architecture. This list exists to remind the developer why they should NOT make these same decisions again.
