# 09 — Design Decisions

> All significant design decisions made during QuantBit v1, with analysis for V2.

---

## Decision 1: Terminal Aesthetic

**Decision:** Black+green terminal aesthetic for the entire UI.

**Source:** Root CSS, all UI components

**Why it was made:** Differentiate from typical stock apps, target power users who prefer keyboard-driven interfaces.

**Verdict:** ✅ KEEP. The terminal aesthetic is a brand differentiator. V2 should evolve it (better responsive design, consistent component library) but maintain the core identity.

---

## Decision 2: 4-Tab Layout (Market / Portfolio / Backtest / Analytics)

**Decision:** Primary navigation via 4 keyboard-accessible tabs.

**Source:** `src/App.tsx`, `src/components/AppSidebar.tsx`

**Why it was made:** Simple mental model. Each tab maps to a distinct user workflow.

**Verdict:** ✅ KEEP. Well-proven in V1. V2 should add a 5th tab (Admin/Settings) for configuration.

---

## Decision 3: Keyboard-Only Navigation

**Decision:** 1/2/3/4 for tabs, / for search, Esc for drawers.

**Source:** `src/hooks/useShortcuts.ts`

**Why it was made:** Power users navigate faster without mouse. Terminal theme supports this.

**Verdict:** ✅ KEEP. Essential for the power-user experience.

---

## Decision 4: Pure TS Engine (No External Dependencies)

**Decision:** All financial math in pure TypeScript without any external math/finance library.

**Source:** `src/engine/*.ts`

**Why it was made:** Control over every calculation. No dependency risk. Easier to audit and verify.

**Verdict:** ✅ KEEP. This was a correct decision. Financial calculations are simple enough to implement directly. Dependencies would add risk without significant benefit.

---

## Decision 5: Yahoo Finance as Primary Data Source

**Decision:** Use Yahoo Finance API (undocumented, reverse-engineered) for all market data.

**Source:** `scripts/fetch_historical_data.ts`, `scripts/sync-daily-data.ts`

**Why it was made:** Only free source for IDX historical data. No viable alternatives at the time.

**Verdict:** ⚠️ ACCEPT WITH MITIGATION. Yahoo Finance is still the only free source for IDX data, but V2 must:
- Implement graceful degradation when Yahoo fails (use cached data)
- Add data freshness monitoring
- Document the data quality limitations

---

## Decision 6: JSON as Primary Storage

**Decision:** Store all historical data as JSON files, import at build time.

**Source:** `data/years/*.json`

**Why it was made:** Simplest possible storage. No DB setup, no migrations. Works with Vite JSON import.

**Verdict:** ❌ REMOVE in V2. This was the root cause of the triple-data-source problem. V2 uses D1 as single SOT.

---

## Decision 7: Cloudflare Pages Functions (Monolithic)

**Decision:** Single `[[path]].ts` file handling all API routes.

**Source:** `functions/api/[[path]].ts`

**Why it was made:** Cloudflare Pages Functions' catch-all routing is the simplest way to deploy. Adding files per route requires more complex Cloudflare configuration.

**Verdict:** ❌ REMOVE in V2. Hono with domain modules is cleaner and testable. Cloudflare Pages Functions support directory routing now.

---

## Decision 8: Custom PBKDF2 Auth

**Decision:** Implement custom authentication with PBKDF2 hashing, session tokens, manual cookie management.

**Source:** `functions/api/[[path]].ts`

**Why it was made:** Avoid external OAuth provider dependency. Full control over auth flow.

**Verdict:** ❌ REMOVE in V2. The dev-session bypass and API key in URL are direct consequences of custom auth. Use Better Auth or a similar library.

---

## Decision 9: AI Chat with 4-Level Architecture

**Decision:** L1 Q&A → L2 Read Tools → L3 Action Approval → L4 Proactive Agent.

**Source:** `src/ai/*`, `src/hooks/useAITools.ts`

**Why it was made:** Graduated AI capability. Users trust AI more when they see it can read (L2) before it can act (L3). Proactive (L4) is the long-term vision.

**Verdict:** ✅ KEEP. The 4-level architecture is well-designed. V2 should preserve this with improved tool definitions and structured output.

---

## Decision 10: Regex-Extracted AI Tool Calls

**Decision:** AI responses contain markdown-formatted tool calls, extracted via regex.

**Source:** `src/ai/toolCallParser.ts`

**Why it was made:** AI models didn't reliably support tool calling at the time (early 2026). Markdown in response was the only way to get structured output.

**Verdict:** ❌ REMOVE in V2. Modern AI models support tool calling/JSON mode. Use structured output APIs directly.

---

## Decision 11: Jaksel AI Persona ("Rico Lubis")

**Decision:** AI persona as a casual Jakarta-based analyst mixing Indonesian-English.

**Source:** `src/ai/systemKnowledge.ts`

**Why it was made:** Target Indonesian retail investors. Casual tone feels approachable. "Jaksel" (South Jakarta) persona is recognizable.

**Verdict:** ✅ KEEP but make configurable. The persona works well for Indonesian users, but V2 should:
- Make the persona a configuration option
- Allow switching personas (formal, casual, English)
- Remove hardcoded slang from system prompt

---

## Decision 12: EngineConfig as Single Source of Truth

**Decision:** `EngineConfigContext` is THE source of truth for all strategy settings. Portfolio is the control center.

**Source:** `src/contexts/EngineConfigContext.tsx`, `docs/AGENTS.md`

**Why it was made:** Resolved inconsistency between Portfolio and Backtest settings. Live mode = Portfolio settings, Draft mode = experiment.

**Verdict:** ✅ KEEP. This was a late-stage improvement (Sesi 12 ADR-011) that correctly solved the Portfolio vs Backtest alignment problem. V2 should implement this architecture from day one.

---

## Decision 13: 50% Reserve Buffer for Gold

**Decision:** When gold is the safe haven, keep 50% cash (not all in gold).

**Source:** `src/engine/core.ts` crash protection logic

**Why it was made:** Gold can lose value too. 50% buffer provides optionality to buy stocks during recovery.

**Verdict:** ⚠️ REVIEW. This heuristic was ad-hoc. V2 should either:
- Make the reserve buffer configurable
- Or derive it from market conditions (e.g., regime state)

---

## Decision 14: 252 Trading Days Assumption

**Decision:** Use 252 trading days for annualized volatility calculation.

**Source:** `src/engine/metrics.ts:63`

**Why it was made:** Standard for US markets. Developer was more familiar with US market conventions.

**Verdict:** ❌ CHANGE to 247 in V2. IDX has different holiday schedule. 252 overstates annualized volatility by ~2%.

---

## Decision 15: 5% Risk-Free Rate

**Decision:** Hardcode 5% risk-free rate for Sharpe/Sortino calculations.

**Source:** `src/engine/metrics.ts:70`

**Why it was made:** Simple assumption. BI rate was approximately 5% at the time.

**Verdict:** ⚠️ MAKE CONFIGURABLE. V2 should:
- Default to BI rate or SBN yield
- Allow user to override
- Use historic rates for backtest periods

---

## Decision 16: 60/40 Benchmark (IHSG/Gold)

**Decision:** Benchmark is 60% IHSG + 40% Gold (not traditional 60/40 stocks/bonds).

**Source:** `src/engine/metrics.ts:81-84`

**Why it was made:** Indonesian bond market data is hard to get. Gold is a reasonable proxy for "safe" asset in IDX context.

**Verdict:** ✅ KEEP but make configurable. The 60/40 IHSG/Gold benchmark is pragmatic given data availability. V2 should also support traditional benchmarks.

---

## Decision 17: Real-time via GoAPI Polling

**Decision:** 60-second polling of GoAPI for live prices.

**Source:** `src/hooks/useDataFeed.ts`

**Why it was made:** Free real-time IDX data, easy to implement.

**Verdict:** ⚠️ KEEP polling for MVP, add WebSocket later. GoAPI may not be production-ready. V2 should implement proper WebSocket when a stable provider is available.

---

## Decision 18: Adaptive DCA Mode

**Decision:** 3 backtest modes: `algo` (rank-based), `custom` (manual), `adaptive_dca` (BPS-based).

**Source:** `src/engine/types.ts`, `src/engine/core.ts`

**Why it was made:** Different users have different strategies. Adaptive DCA experiments with using BPS for entry timing.

**Verdict:** ✅ KEEP all 3 modes. Each serves a distinct use case. Adaptive DCA is a unique feature (not just DCA, but BPS-timed DCA).

---

## Decision 19: Gold + Cash + Stocks Portfolio Model

**Decision:** Portfolio composed of three asset types: stocks, cash, and gold grams.

**Source:** `src/hooks/usePortfolioManager.ts`

**Why it was made:** Gold is the most accessible safe haven for Indonesian retail investors. Cash is the default during defensive regimes.

**Verdict:** ✅ KEEP. Three-asset model is simple and appropriate for IDX retail investors. V2 should consider adding bonds as a 4th asset class.

---

## Decision 20: Equal-Weight Allocation

**Decision:** All stocks in portfolio receive equal capital allocation (not market-cap weighted).

**Source:** `src/engine/allocator.ts`

**Why it was made:** Simpler, more transparent, avoids concentration risk. Also matches the ranking paradigm (top N = equal trust).

**Verdict:** ✅ KEEP for V2 MVP. Equal-weight is a valid strategy. V2 could support market-cap or confidence-weighted allocation as an option.
