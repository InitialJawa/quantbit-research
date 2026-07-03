# 31 — Open Questions

> Questions that still need research or resolution before/during V2 development.

---

## Data & Finance

### Q1: Where to get reliable IDX historical data?

**Context:** Yahoo Finance is the only free source used in v1, but it's undocumented and unreliable. GoAPI has limited ticker coverage (9 tickers).

**Options:**
1. Stick with Yahoo Finance + graceful degradation (v1 approach)
2. Pay for IDX data feed (cost: $??/month)
3. Use IDX Warehouse (CGI) API — but it's slow and rate-limited
4. Accept imperfect data and focus on making it transparent (DataStatus)

**Needed:** Research data source options. Try Rust-based approach for faster Yahoo parsing?

### Q2: What is the actual IDX trading day count?

**Context:** v1 used 252 (US standard). IDX has ~240-247 depending on year.

**Action:** Count trading days for 2021-2025 from actual market data. Use the average.

### Q3: Is the 60/40 IHSG/Gold benchmark reasonable?

**Context:** v1 used 60% IHSG + 40% Gold as benchmark. Traditional portfolios use 60/40 stocks/bonds.

**Questions:**
- What is the correlation of IHSG vs Gold over 5 years?
- Is Gold a better safe haven than Indonesian bonds (SBN)?
- Should SBN yield data be available/feasible to include?

### Q4: Where to get reliable dividend data?

**Context:** v1 used Yahoo Finance dividend events + IDX Warehouse for 5 tickers only.

**Action needed:** Find a source for historical dividend data for all 80+ IDX80 tickers. RTI, IDX, or paid data feed.

### Q5: What fee rates do Indonesian brokers actually charge?

**Context:** v1 used buy=0.15%, sell=0.25%, tax=0.10%, slippage=0.25%.

**Action:** Survey 3-5 Indonesian online brokers (Ajaib, Stockbit, Bareksa, Mandiri Sekuritas) for actual fee schedules.

---

## Engine & Scoring

### Q6: Can the BPS algorithm be validated statistically?

**Context:** BPS was designed heuristically (5 factors with assigned weights). It has never been validated to actually predict good entry points.

**Action:** Run BPS against historical data:
- At each BPS threshold crossing, what was the forward 1-month/3-month return?
- Does BPS > 70 actually precede better returns than BPS < 30?
- Are the factor weights optimal, or is equal-weight better?

### Q7: Should profile weights be validated?

**Context:** AMAN, AGRESIF, DIVIDEN weights were designed by the developer, not optimized.

**Action:** Grid search optimal weights for each profile type against historical data. Compare actual performance of default weights vs optimized weights.

### Q8: Does crash sensitivity = 10% make sense for IDX?

**Context:** IHSG has drawdowns of 5-15% several times per year. 10% sensitivity may trigger too often or not often enough.

**Action:** Analyze IHSG drawdown distribution 2021-2025. Determine optimal sensitivity that balances false positives vs catching real crashes.

### Q9: Is the 50% reserve buffer for gold too conservative?

**Context:** When buying gold during crash, only 50% of cash goes to gold (50% stays as cash).

**Action:** Backtest different reserve ratios (0%, 25%, 50%, 100%) to find optimal recovery-to-drawdown ratio.

---

## AI

### Q10: What is the best AI model for financial Q&A in Indonesian?

**Context:** v1 used Llama 4 Scout via OpenRouter free tier. GPT-4o-mini as paid fallback.

**Action needed:** Benchmark 3+ models on:
- Tool call accuracy (structured output success rate)
- Financial reasoning (portfolio questions, market analysis)
- Indonesian language quality
- Response latency
- Cost per query

### Q11: Should the proactive agent be built?

**Context:** v1 defined L4 (proactive agent) but never validated it works well. 6 BPS rules with unclear user value.

**Action:** User research: "Would you want the app to proactively tell you when to buy? Or do you prefer asking when you want?"

### Q12: Is structured output reliable enough without regex fallback?

**Context:** v1 used regex because models couldn't reliably output JSON. Modern models (GPT-4o, Llama 4) support JSON mode.

**Action:** Test JSON mode with 100 varied prompts. Measure success rate. If < 95%, keep regex as fallback.

---

## Infrastructure

### Q13: Does D1 have sufficient performance for backtest queries?

**Context:** Backtest needs to scan ~120K rows of stock_daily. D1 limits: 80KB per statement, 1000ms CPU time.

**Action:** Benchmark a 5-year backtest query on D1. If > 10 seconds, add Worker-level caching or pre-compute daily summaries.

### Q14: Is Better Auth compatible with Cloudflare Pages Functions?

**Context:** Better Auth is designed for Node.js. Cloudflare Pages Functions use Workers runtime (Service Workers, not Node.js).

**Action:** Test Better Auth compatibility. If not, find alternative:
1. Custom auth (simplified v1 pattern, no dev bypass)
2. Cloudflare Access / Zero Trust
3. Lucia Auth (Workers-compatible)

### Q15: How to handle D1's 80KB statement limit for large backtest inserts?

**Context:** Backtest results with 5 years of daily log entries may exceed D1's statement size limit.

**Options:**
1. Batch inserts (100 per batch as v1 did)
2. Store chart data as pre-computed JSON, not per-day rows
3. Use Workers KV for large result blobs

---

## User Experience

### Q16: Do users actually use AI chat for portfolio decisions?

**Context:** v1 had AI chat, but no analytics on usage. Unknown if users ask meaningful questions or just experiment.

**Action:** Add anonymous usage tracking (opt-in):
- % of users who send at least 1 AI message
- Average messages per session
- Most common question types
- Tool execution success rate

### Q17: Is the terminal aesthetic universally appealing?

**Context:** Terminal aesthetic may intimidate non-technical users.

**Action:** A/B test with a lighter theme option. Track conversion/retention.

### Q18: What is the minimum viable product?

**Context:** 37 features planned for V2 (21 kept + 9 rewritten + 1 refactored + 6 new).

**Question:** What's the smallest set of features that delivers user value? (Define "must-have" vs "nice-to-have" more rigorously than the existing feature matrix.)

**Suggested MVP:** Portfolio + Market view + Basic backtest = core value. AI chat can be v2.1.

---

## Strategic

### Q19: Should QuantBit be open source or commercial?

**Context:** v1 was private/experimental. No business model.

**Options:**
1. Open source (MIT/GPL) — community contributions, free hosting
2. Commercial (subscription) — need payment integration, premium features
3. Hybrid — core open source, premium hosted version

**Decision needed before launch:** Affects folder structure, licensing, payment integration.

### Q20: Is there actually a market for QuantBit?

**Context:** Built by one developer for personal use. No user research.

**Action needed:** Interview 5-10 Indonesian retail investors:
- How do they currently manage portfolios?
- What tools do they use?
- What problems do they have?
- Would they pay for QuantBit?
- What feature would make them switch from Excel/Stockbit/Ajaib?

---

## Summary

| Category | Open Questions | Priority |
|----------|---------------|----------|
| Data sources | Q1, Q4 | HIGH — block pipeline |
| Financial validation | Q2, Q3, Q5, Q6, Q7, Q8, Q9 | MEDIUM — improve accuracy |
| AI | Q10, Q11, Q12 | HIGH — block AI features |
| Infrastructure | Q13, Q14, Q15 | HIGH — block deployment |
| User research | Q16, Q17, Q18 | MEDIUM — inform priorities |
| Strategic | Q19, Q20 | LOW — can defer to post-MVP |
