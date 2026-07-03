# 05 — Lessons Learned

> What worked, what failed, and what must change in V2.

---

## What Worked Well (Keep in V2)

### 1. Terminal Aesthetic
The black+green terminal aesthetic created a unique identity. Users associated the interface with "serious tool" rather than "trading app."

### 2. BPS Algorithm
The 5-factor Buy Pressure Score is the most innovative component. The crisis override, adaptive thresholds, and 5-level action mapping are well-designed. **Must preserve algorithm exactly.**

### 3. Weighted Factor Ranking
Quality × Growth × Value × Momentum × Dividend scoring with configurable profiles is sound. The percentile rank normalization (0-95) removes scale bias between factors.

### 4. Crash Protection (2-Tier)
Fast crash + slow grind detection with gold/cash safe haven is the most important risk management feature. The cooldown prevents oscillation.

### 5. AI 4-Level Architecture
L1 Q&A → L2 Read Tools → L3 Action Approval → L4 Proactive Agent is a clean progression. The circuit breaker pattern for providers works well.

### 6. Market Regime Engine
5-state decision tree with priority ordering correctly captures market dynamics. Breadth analysis (how many stocks above score 60) is a useful secondary signal.

### 7. DataStatus Transparency
LIVE/CACHED/STALE/ESTIMATED badges on every data point builds user trust. The system doesn't pretend all data is equally current.

### 8. Keyboard Shortcuts
1/2/3/4 for tabs, / for search, Esc for drawers. Power users appreciate keyboard-first navigation.

### 9. Backtest-to-Portfolio Sync
SYNC TO PORTFOLIO allows users to test strategies in simulation and then deploy. The live/draft toggle (backtestUseLiveStrategy) enables experimentation without risk.

### 10. Dividend Tracking
ForwardDividendsForecast and dividend cache provide concrete value for income-focused investors.

---

## What Failed (Fix in V2)

### 1. No Single Source of Truth (Critical)
**Failure:** Same data in JSON, SQLite, D1, and in-memory singletons. Backtest results don't match live portfolio.
**Fix:** D1 is THE source. Everything reads from D1. No JSON imports, no SQLite, no in-memory copies.

### 2. Monolithic API (Critical)
**Failure:** 1236-line `[[path]].ts` — every route in one file, no middleware, no validation.
**Fix:** Hono with domain modules (auth/portfolio/market/engine/ai). Zod validation at every boundary.

### 3. No Architecture Plan (Major)
**Failure:** System grew organically without upfront design. Each new feature picked its own data source.
**Fix:** Write the blueprint (this document) before writing code. Architecture decisions documented as ADRs.

### 4. Hidden Business Rules (Major)
**Failure:** 50 undocumented items — magic numbers, hardcoded assumptions, synthetic data.
**Fix:** All business rules documented upfront. Configuration files instead of hardcoded constants.

### 5. Regex-Parsed AI Output (Major)
**Failure:** AI tool calls extracted via regex from markdown text blocks. Fragile — any output format change breaks parsing.
**Fix:** Use AI structured output (JSON mode / tool calls). No regex parsing.

### 6. Dual Scoring Pipelines (Major)
**Failure:** fetch_historical_data.ts computes scores one way, force-sync runIdx80Scan() computes them another way.
**Fix:** One unified scoring algorithm, computed server-side, stored in D1.

### 7. No Input Validation (Major)
**Failure:** All API endpoints read `request.json() as any`. No Zod, no type safety.
**Fix:** Every endpoint has Zod schema. Validation errors return structured 400 responses.

### 8. Dev Bypass in Production (Critical)
**Failure:** `getUserFromSession()` returns "dev-user" on any error, including "no token provided."
**Fix:** Return 401 on authentication failure. Dev mode gated behind env variable + explicit flag.

### 9. Build-Time JSON Imports (Major)
**Failure:** Backtest reads `data/years/*.json` via Vite JSON import. Data is bundled with the app, can't be updated without redeploy.
**Fix:** Backtest queries D1 via API. No JSON imports for runtime data.

### 10. No Testing Strategy (Major)
**Failure:** Engine (pure TS math) has no unit tests. API has no integration tests. Backtest results verified manually.
**Fix:** Unit tests for every engine function. Integration tests for API endpoints. Golden file tests for backtest.

---

## Things to Never Repeat

| # | Mistake | Why It's Dangerous |
|---|---------|-------------------|
| 1 | **API key in URL query string** | Logged everywhere — Cloudflare, proxy, server logs |
| 2 | **"dev-session" auth bypass** | Production security theater |
| 3 | **Stack trace in error response** | Information disclosure to attackers |
| 4 | **Hardcoded ticker list** | Stale after 6 months (IDX80 rebalance) |
| 5 | **Secret in POST body** | Visible in request logs |
| 6 | **Catch-all error to "dev-user"** | Masks auth failures |
| 7 | **JSON blobs in relational DB** | Can't query, can't index, can't validate |
| 8 | **Same formula in 3 files** | Maintenance nightmare |
| 9 | **Mixed real/synthetic data** | Users can't distinguish accurate vs estimated |
| 10 | **No pagination on data APIs** | 120K rows in one response |

---

## Things to Carry Forward

| # | Asset | Why Keep |
|---|-------|----------|
| 1 | **BPS formula + weights** | Genuine innovation, tested for 6 months |
| 2 | **Ranking algorithm** | Sound factor-based approach |
| 3 | **Crash detection logic** | 2-tier + recovery verification is robust |
| 4 | **Market regime decision tree** | Priority-ordered state machine works |
| 5 | **Terminal UI aesthetic** | Brand identity |
| 6 | **DataStatus transparency** | Builds user trust |
| 7 | **Keyboard shortcuts** | Power user expectation |
| 8 | **Backtest-to-Portfolio sync** | Core workflow |
| 9 | **AI 4-level architecture** | Clean separation of concerns |
| 10 | **Fee structures** | Empirically derived from IDX brokerage |
