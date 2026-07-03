# 28 — Risk Register

> Risks to QuantBit V2 development and operation.

---

## Risk Matrix

```
Likelihood: 1 (Rare) → 5 (Almost certain)
Impact:     1 (Negligible) → 5 (Catastrophic)
```

| # | Risk | Likelihood | Impact | Score | Mitigation |
|---|------|-----------|--------|-------|------------|
| 1 | Yahoo Finance API changes/dies | 3 | 5 | 15 | Multiple data sources, graceful degradation |
| 2 | Backtest results differ from v1 | 2 | 4 | 8 | Golden file tests comparing v1 output |
| 3 | D1 free tier limits reached | 3 | 3 | 9 | Monitoring, cleanup routines, upgrade plan |
| 4 | Cloudflare Pages Functions cold start | 4 | 2 | 8 | Pre-warm cron, Workers D1 proxy |
| 5 | AI provider rate limiting | 4 | 3 | 12 | Multiple fallback providers, circuit breaker |
| 6 | User data loss or corruption | 1 | 5 | 5 | Weekly D1 exports, audit columns |
| 7 | V2 not feature-complete vs V1 | 3 | 3 | 9 | Feature matrix tracked, deprioritize non-essential |
| 8 | Learning curve for Hono/D1 tooling | 2 | 2 | 4 | Docs, AGENTS.md, known patterns documented |
| 9 | IDX80 ticker list stale | 4 | 2 | 8 | DB-backed list, next-update reminder calendar |
| 10 | GoAPI becomes paid/unavailable | 2 | 3 | 6 | Real-time optional, cached data as fallback |
| 11 | Data pipeline cron fails silently | 3 | 4 | 12 | Pipeline monitoring, status badge in admin dashboard |
| 12 | Team member leaves with context | 2 | 4 | 8 | Comprehensive docs (33 files), AGENTS.md chain |

---

## Risk Details

### Risk 1: Yahoo Finance API Changes/Dies (Score: 15)

**Context:** v1 relied entirely on Yahoo Finance (undocumented, reverse-engineered API). The daily sync and force-sync both use Yahoo endpoints that change without notice.

**Impact:** Market data stops flowing. Backtests return old data. Scores stop computing.

**Mitigation:**
- Implement graceful degradation: if Yahoo fails, use most recent cached data (marked STALE)
- Add data freshness monitoring to alert on stale data
- Document Yahoo as an imperfect source — users should understand data may be delayed
- Post-MVP: add a second data source (IDX API, financial data feed)

### Risk 5: AI Provider Rate Limiting (Score: 12)

**Context:** OpenRouter free tier models have strict rate limits. If all free models hit limits, AI chat becomes unavailable.

**Impact:** AI features stop working. Users see "AI unavailable" message.

**Mitigation:**
- Circuit breaker with 5-minute cooldown on rate-limited providers
- Multiple fallback models (free → paid free tier → paid)
- Transparent status in AI chat: "Currently using fallback model (GPT-4o-mini)"
- Rate limit UI: show remaining requests in tooltip

### Risk 11: Data Pipeline Cron Fails Silently (Score: 12)

**Context:** v1's cron pipeline had silent failure modes — if a ticker failed to fetch, it was silently skipped. No alerts on partial failure.

**Impact:** Stale market data, inconsistent scores, user confusion.

**Mitigation:**
- Pipeline status tracking: store last-run timestamp + result in D1 admin table
- Admin dashboard shows pipeline health badge
- Compare expected vs actual rows after each run (80 tickers × 1 day = 80 rows)
- Alert if pipeline hasn't run in 48 hours

---

## Go/No-Go Criteria

### Go (Proceed to production)

- [ ] Engine passes golden file tests (CAGR, Sharpe within 0.5% of v1)
- [ ] API endpoints all return correct responses
- [ ] Data pipeline runs successfully with real Yahoo data
- [ ] AI chat responds with structured output (no regex fallback)
- [ ] Auth: login, register, logout all work
- [ ] Portfolio: buy, sell, view holdings all work
- [ ] Backtest: configure, run, view results, sync to portfolio all work
- [ ] No P0/P1 security issues open

### No-Go (Delay production)

- [ ] Engine golden file tests fail by > 5%
- [ ] Data pipeline cannot complete a full run
- [ ] Auth bypass exists (similar to v1's "dev-session")
- [ ] AI provider returns unstructured responses
- [ ] User portfolio data can be viewed by other users

---

## Contingency Plans

| Trigger | Plan |
|---------|------|
| Yahoo Finance API fully blocked | Switch to manual data upload + cached data. AI chat still works for Q&A. |
| D1 performance degrades | Upgrade D1 plan, add KV caching layer, archive old data to separate D1. |
| Better Auth incompatible with Cloudflare | Fallback to custom auth (simplified, no dev bypass) using code from v1's auth pattern. |
| AI provider ecosystem collapses | Revert to regex-based parsing (v1 pattern). Degraded but functional. |
| Developer burned out | Stop at Sprint 5 (complete app without AI). AI can be added later. |
