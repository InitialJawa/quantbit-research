# 29 — Decisions to Keep

> Decisions from QuantBit v1 that must be preserved in V2.

---

## Architecture Decisions

| # | Decision | Why Keep | Evidence |
|---|----------|----------|----------|
| 1 | **Pure TS engine (no external math libraries)** | Full control, auditable, no dependency risk | `src/engine/*.ts` — all custom implementations |
| 2 | **4-level AI architecture** | Graduated trust model works | L1-L4 implemented and tested in v1 |
| 3 | **Tab-based navigation with keyboard shortcuts** | Power users love it | `useShortcuts.ts` — 1/2/3/4 tabs, / search |
| 4 | **Terminal aesthetic** | Brand differentiator | All components use black+green theme |
| 5 | **EngineConfigContext as config hub** | Single source for strategy settings | ADR-009, ADR-011 |
| 6 | **Portfolio as control center** | Resolved Portfolio vs Backtest conflict | ADR-011: Live/Draft toggle |
| 7 | **Live/Draft backtest mode** | Sandbox for experiments | `backtestUseLiveStrategy` toggle |
| 8 | **DataStatus transparency** | Builds user trust | `DataStatus.ts` — LIVE/CACHED/STALE/ESTIMATED |

## Algorithm Decisions

| # | Algorithm | Why Keep | Source |
|---|-----------|----------|--------|
| 9 | **BPS 5-factor formula** | Genuine innovation | `buyPressure.ts` — val*0.30 + mom*0.25 + breadth*0.15 + dd*0.20 + fear*0.10 |
| 10 | **BPS action mapping** | ✅ <30:none → >=90:deploy | `buyPressure.ts` — 5-level mapping |
| 11 | **Crisis override in BPS** | Safety during crashes | `buyPressure.ts` — withCrisisOverride() |
| 12 | **Weighted factor ranking** | Q*Wq + G*Wg + V*Wv + M*Wm + D*Wd | `ranker.ts` |
| 13 | **2-tier crash detection** | Fast crash + slow grind | `crashDetector.ts` |
| 14 | **Recovery verification** | Trend + momentum recovery gates | `crashDetector.ts` |
| 15 | **5-state regime decision tree** | Priority-ordered state machine | `marketRegimeEngine.ts` |
| 16 | **Emergency exit (rank ≥ 15)** | Immediate sell on bad rank | `core.ts:444-492` |
| 17 | **Routine exit (rank ≥ 10 + month-end)** | Systematic rebalancing | `core.ts:444-492` |
| 18 | **60/40 benchmark (IHSG/Gold)** | Pragmatic for IDX context | `metrics.ts` — no bond data available |
| 19 | **Equal-weight allocation** | Simple, transparent, anti-concentration | `allocator.ts` |
| 20 | **Lot-based trading (100 shares)** | IDX market standard | `allocator.ts` |

## Business Decisions

| # | Decision | Why Keep | Evidence |
|---|----------|----------|----------|
| 21 | **3 investment profiles (AMAN/AGRESIF/DIVIDEN)** | Cover primary investor types | Used throughout engine |
| 22 | **Gold as safe haven (not bonds)** | Accessible for IDX retail | Gold price tracked in daily_overview |
| 23 | **IDX market focus** | Niche focus, less competition | All tickers are IDX |
| 24 | **Cash + Gold + Stocks** | Three-asset model is simple and sufficient | Portfolio manager tracks all three |
| 25 | **Indonesian-language AI** | Target market | `systemKnowledge.ts` — Jaksel persona |

## UI Decisions

| # | Decision | Why Keep | Source |
|---|----------|----------|--------|
| 26 | **Floating AI chat** | Always accessible | `FloatingAIChat.tsx` |
| 27 | **StockDrawer slide-over** | Context without page navigation | `StockDrawer.tsx` |
| 28 | **BPS gauge + factor breakdown** | Visualizes buy pressure | `BuyPressureDashboard.tsx` |
| 29 | **ForwardDividendsForecast** | Concrete value for income investors | `ForwardDividendsForecast.tsx` |
| 30 | **SYNC TO PORTFOLIO button** | Clear action for backtest results | `SimulationTab.tsx` |

---

## Decisions That Need Modification

| # | Original Decision | Modification for V2 |
|---|------------------|---------------------|
| 1 | Custom PBKDF2 auth | → Better Auth |
| 2 | Regex AI parsing | → Structured JSON output |
| 3 | JSON file storage | → D1 single SOT |
| 4 | Monolithic API | → Hono domain modules |
| 5 | Hardcoded profile weights | → Database-stored profiles |
| 6 | Hardcoded ticker list | → DB ticker catalog |
| 7 | 252 trading days | → 247 (IDX-specific) |
| 8 | 5% risk-free rate hardcoded | → Configurable + SBN yield |
| 9 | 10K char AI memory | → Structured DB-backed memory |
| 10 | Hardcoded Jaksel persona | → Configurable persona |
