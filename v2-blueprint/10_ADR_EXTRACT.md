# 10 — ADR Extract

> Summary of all Architecture Decision Records found in the QuantBit v1 repository.

---

## ADR Index

| ADR | Title | Status | Date |
|-----|-------|--------|------|
| ADR-001 | Initial architecture | ACCEPTED | 2026-04-XX |
| ADR-002 | Terminal aesthetic + keyboard nav | ACCEPTED | 2026-05-XX |
| ADR-003 | Tab refactor — DashboardGrid → per-tab components | ACCEPTED | 2026-05-XX |
| ADR-004 | Engine engine redesign — domain modules | ACCEPTED | 2026-05-XX |
| ADR-005 | AI Architecture — 4-level system | ACCEPTED | 2026-05-XX |
| ADR-006 | EngineConfigContext as config hub | ACCEPTED | 2026-05-XX |
| ADR-007 | Cloudflare Pages + D1 backend | ACCEPTED | 2026-05-XX |
| ADR-008 | Data flow redesign — JSON → SQLite → D1 | ACCEPTED | 2026-05-XX |
| ADR-009 | Single Source of Truth — EngineConfigContext | ACCEPTED | 2026-06-XX |
| ADR-010 | Carried-forward data in backtest | ACCEPTED | 2026-06-XX |
| ADR-011 | Portfolio strategy control center + Live/Draft backtest | ACCEPTED | 2026-06-XX |
| ADR-012 | AI structured output + provider fallback | ACCEPTED | 2026-06-XX |

**Source files:** `docs/ADR-*.md`

---

## ADR-001: Initial Architecture

**Decision:** React SPA + engine modules + static JSON data.
**Rationale:** Fastest path to working prototype. No backend needed initially.
**V2 Verdict:** ✅ Surpassed. V2 adds production backend, but the thin-client architecture is preserved.

---

## ADR-002: Terminal Aesthetic + Keyboard Nav

**Decision:** Black+green terminal look. Keyboard shortcuts for tabs.
**Rationale:** Brand differentiation. Power user focus.
**V2 Verdict:** ✅ KEEP. Core identity.

---

## ADR-003: Tab Refactor

**Decision:** Replace DashboardGrid with individual tab components (MarketTab, PortfolioTracker, SimulationTab, AnalyticsTab).
**Rationale:** Better code organization, independent rendering, easier maintenance.
**V2 Verdict:** ✅ KEEP. Per-tab components is the right pattern.

---

## ADR-004: Engine Redesign

**Decision:** Split engine into domain modules: core.ts, ranker.ts, allocator.ts, crashDetector.ts, metrics.ts, buyPressure.ts.
**Rationale:** Each engine concern is testable independently. Clear separation between scoring (ranker), allocation (allocator), risk (crashDetector), and performance (metrics).
**V2 Verdict:** ✅ KEEP. Module structure is clean. V2 should add dependency injection for testability.

---

## ADR-005: AI Architecture — 4 Levels

**Decision:** L1 Q&A → L2 Read Tools → L3 Action Approval → L4 Proactive Agent.
**Rationale:** Graduated trust model. Users see AI capability grow from reading to acting.
**V2 Verdict:** ✅ KEEP with improvement. Use structured output instead of regex.

---

## ADR-006: EngineConfigContext as Config Hub

**Decision:** Centralize all strategy configuration in EngineConfigContext. localStorage persistence.
**Rationale:** Single source of truth for settings. Avoids prop drilling through 4 tabs.
**V2 Verdict:** ⚠️ IMPROVE. Centralized config is correct, but sync to D1 for persistence across devices.

---

## ADR-007: Cloudflare Pages + D1

**Decision:** Cloudflare Pages Functions for API, D1 for production database.
**Rationale:** Free tier, edge deployment, no server management. D1 is SQLite-compatible, minimal migration effort.
**V2 Verdict:** ✅ KEEP. This ecosystem decision is sound. V2 adds Hono on top for better DX.

---

## ADR-008: Data Flow Redesign

**Decision:** JSON → SQLite → D1 pipeline with seed scripts.
**Rationale:** Incremental migration from JSON-only to database-backed. Let each step be tested independently.
**V2 Verdict:** ❌ SIMPLIFY. Direct D1 writes from a unified pipeline. No intermediate formats.

---

## ADR-009: Single Source of Truth

**Decision:** EngineConfigContext is the single source of truth for all strategy settings.
**Rationale:** Multiple sources caused inconsistencies in earlier versions.
**V2 Verdict:** ✅ KEEP. V2 expands this to data as well (D1 as SOT for ALL data).

---

## ADR-010: Carried-Forward Data

**Decision:** Backtest uses carried-forward data when real data is unavailable. Flag with `isCarriedForward`.
**Rationale:** Avoids gaps in backtest timeline. User can see which data is estimated.
**V2 Verdict:** ⚠️ IMPROVE. The flag is good, but V2 should:
- Use STALE status instead of carried-forward
- Clearly mark estimated data in UI
- Never use carried-forward data for live trading decisions

---

## ADR-011: Portfolio Strategy Control Center

**Decision:** Portfolio tab is the control center. Live mode syncs settings, Draft mode allows experiments.
**Rationale:** The Portfolio vs Backtest setting inconsistency was the most confusing UX issue. This resolves it.
**V2 Verdict:** ✅ KEEP. Implement from day one in V2.

---

## ADR-012: AI Structured Output + Provider Fallback

**Decision:** AI tool calls extracted from markdown. 5 providers with circuit breaker fallback.
**Rationale:** AI models couldn't reliably output structured data at the time. Multiple providers needed because some are geo-blocked on Cloudflare.
**V2 Verdict:** ⚠️ PARTIALLY KEEP. Provider fallback is good. Structured output must change to JSON mode. Add more fallback providers.

---

## Cross-ADR Analysis

### Conflicting Decisions

| Conflict | ADRs | Analysis |
|----------|------|----------|
| **JSON as SOT (ADR-008) vs D1 as SOT (ADR-007)** | ADR-007, ADR-008 | ADR-007 won — D1 is the production database. But ADR-008's JSON pipeline remained as intermediate storage, causing the triple-source problem. **V2:** D1 is sole SOT from day one. |
| **localStorage as config source (ADR-006) vs D1 as config source (ADR-009)** | ADR-006, ADR-009 | ADR-006 chose localStorage for speed. ADR-009 chose D1 for persistence. Both exist simultaneously. **V2:** D1 as SOT, localStorage as cache with timestamp-based merge. |

### ADRs That Need Reversal in V2

| ADR | Decision | Why Reverse |
|-----|----------|-------------|
| ADR-008 | JSON → SQLite → D1 pipeline | Too many intermediate formats. Direct D1 writes. |
| ADR-006 | localStorage as config persistence | D1 sync needed for cross-device use. |

### ADRs That Are Foundational for V2

| ADR | Principle | V2 Application |
|-----|-----------|---------------|
| ADR-004 | Domain-modular engine | Same module structure |
| ADR-005 | 4-level AI | Same graduated trust model |
| ADR-009 | Single Source of Truth | Expanded to all data |
| ADR-011 | Portfolio as control center | Same Live/Draft pattern |
| ADR-012 | Provider fallback | More fallback providers, structured output |
