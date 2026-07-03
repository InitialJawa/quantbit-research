# Lessons Learned — Quantbit v1 Post-Mortem

## What Worked

### 1. Deterministic Engine Separate from AI (ADR-002)

The decision to ban AI from financial math was the single most important architectural choice. Every calculation — ranking, BPS, backtest returns, crash detection — runs through pure deterministic functions with no LLM involvement. This saved the project from accuracy disasters that plague AI-first financial tools. The engine produces identical results given identical inputs, which is non-negotiable for auditability and backtest reproducibility.

**Evidence**: `src/engine/core.ts` contains zero AI calls. All math is `number` operations on database-sourced `DataPoint` structs.

### 2. DB as Single Source of Truth (Migration 0003)

After an early period of JSON files + SQLite + D1 coexisting (see What Failed #3), the team committed to D1 as the sole authoritative data store. This eliminated:
- Carried-forward stale prices from session to session
- Silent fallback chains producing different values per environment
- Price divergence between market overview and stock detail views
- Impossible-to-debug "where did this number come from" bugs

**Evidence**: The switch to `daily_overview` and `stock_daily` reads in `engine/core.ts` removed all file-based data paths.

### 3. EngineConfigContext

The React context + `useReducer` + `localStorage` pattern for strategy settings (`src/contexts/EngineConfigContext.tsx`) proved robust. Single source of truth for all strategy parameters, persistable across sessions, and testable by dispatching actions. No prop drilling, no Redux overhead.

**Key design decisions that worked**:
- `useReducer` over `useState` for complex nested state
- `localStorage` sync without blocking render
- Zod schema for config validation on load
- Immutable updates via spread, never mutate in place

### 4. DataStatus Transparency

Surface-level status labels (LIVE / CACHED / STALE / ESTIMATED) built user trust by being honest about data freshness. When the market is closed, users see CACHED with a timestamp. When an API call fails, they see STALE with the last-known value. This pattern prevents the illusion of live data.

**Evidence**: `DataStatus` enum used throughout API responses and UI indicators.

### 5. Zero Auto-Execute for AI

AI responses never execute trades or modify portfolios without explicit user approval. This is a safety-critical pattern for any financial application. The AI can recommend, analyze, and explain — but the action button is always user-initiated.

**Evidence**: All AI tool calls return structured proposals; `executeAction()` is guarded by confirmation dialogs.

### 6. Append-Only ADR System

Architecture Decision Records (`docs/adr/`) provided decision traceability across the project lifecycle. Each ADR documents the context, options considered, decision, and consequences. This was invaluable for onboarding new contributors and understanding why past choices were made.

**Evidence**: ADR series from 001 (initial architecture) through 010+ covering storage, AI, auth, data pipeline decisions.

### 7. Circuit Breaker Pattern

Provider failure isolation prevented cascading failures. If one data source or AI provider goes down, the circuit breaker trips to degraded mode rather than crashing the entire app. Users see STALE data indicators instead of error screens.

**Evidence**: `CircuitState` enum (CLOSED / OPEN / HALF_OPEN) in provider wrappers.

### 8. BPS Concept (Best Price Score)

Data-driven DCA replaced calendar-based DCA with an objective scoring system. Instead of "buy every month on the 15th," BPS evaluates "is this a good time to buy based on price relative to fundamentals?" This is a genuinely novel contribution from the project.

**Evidence**: BPS formula: `(valueScore * 0.35) + (growthScore * 0.25) + (qualityScore * 0.20) + (momentumScore * 0.10) + (dividendScore * 0.10)`.

### 9. Three Investment Profiles

The Conservative / Moderate / Aggressive framework gave retail investors a clear on-ramp. Each profile maps to concrete weighting presets and risk parameters. This abstraction hides complexity while maintaining power-user configurability.

**Evidence**: Profile definitions in engine config with distinct factor weight presets.

### 10. Test Coverage (203+ Tests)

The test suite caught regressions during refactors and provided confidence for deployment. Vitest + testing-library coverage of engine logic, React components, hooks, and contexts.

**Evidence**: `vitest.config.ts` with 203+ test files across engine, components, hooks, and contexts.

---

## What Failed

### 1. Monolithic API File — 1236 Lines

`functions/api/[[path]].ts` grew into a 1236-line monster handling auth, market data, backtest, AI, email, and every other concern in a single file. This is the single biggest architectural mistake in the project.

**Symptoms**:
- Impossible to test individual endpoints
- Merge conflicts on every PR
- One bug in auth blocked all API deploys
- No clear ownership boundaries
- Mixed concerns: request parsing, business logic, DB access, response formatting

**Lesson**: One file = one responsibility. Every endpoint deserves its own file.

### 2. Regex Tool Calling

`src/ai/toolCallParser.ts` used fragile regex patterns to extract tool calls from AI text responses. The regex would break with:
- Different phrasing by the AI
- Extra whitespace or line breaks
- Unicode characters in stock names
- JSON nested inside markdown code blocks

**Symptoms**:
- Constant maintenance as AI providers updated their response formats
- Silent failures: regex returns null, AI response shown as plain text
- Untestable: regex state machine intertwined with prompt engineering

**Lesson**: Structured output (JSON schema) is the only reliable way to parse AI tool calls. Regex for structured data is tech debt, not engineering.

### 3. Triple Data Sources — JSON + SQLite + D1

The project started with JSON files (`src/stocksData.ts`, `src/marketData.ts`), added SQLite for the Express server, then added D1 for Cloudflare. For a period, all three coexisted with fallback chains that produced inconsistent results.

**Symptoms**:
- Prices in market tab differed from prices in backtest tab
- STALE data shown when fresh data existed in another source
- Complex fallback logic: `JSON → SQLite → D1 → hardcoded defaults`
- Impossible to reason about which source was authoritative for any given view

**Lesson**: One database, one source of truth. Multiple representations of the same data guarantee divergence.

### 4. Hardcoded Ticker Lists

`src/stocksData.ts` contained 830+ hardcoded stock tickers with metadata. Tickers change every 6 months (delistings, IPOs, mergers, ticker changes). Maintenance required editing a TypeScript file and redeploying.

**Symptoms**:
- PRs for "add delisted ticker" or "update stock name"
- No way for users to add custom tickers
- Stale data for stocks that changed names or sectors
- 100KB+ file in the bundle just for metadata

**Lesson**: Ticker metadata belongs in a database table with an admin UI. Never hardcode dynamic business data.

### 5. Dev Code in Production

Two critical examples:
- **Dev-session bypass**: `IS_DEV_SESSION` flag in auth allowed skipping login during development. Shipped to production.
- **AITestHarness**: Full AI testing UI bundled in production builds. Exposed internal tool schemas and system prompts.

**Symptoms**:
- Security vulnerability: anyone could bypass auth with the right header
- Bundle size bloat: production JS included dev tools
- System prompt leakage via client-side bundle

**Lesson**: Dev-only code must be excluded at build time, not at runtime. Environment-specific builds with strict guards.

### 6. API Key in URL

GoAPI key was passed as a query parameter in URLs logged to the console during development:
```
https://api.goapi.io/stock/candlestick?api_key=YOUR_KEY_HERE&symbol=...
```

**Symptoms**:
- Key leaked in CI logs
- Key leaked in dev console screenshots
- Key visible in network tab of any user

**Lesson**: API keys go in headers or environment variables. Never in URLs. Never logged.

### 7. Over-Engineered AI Architecture

4 levels of AI capability, 5 providers, 9 model configurations — all for a chat interface. The complexity added ~3000 lines of abstraction code for functionality that users interact with as a simple text input.

**Levels**: Basic → Intermediate → Advanced → Proactive
**Providers**: OpenRouter, Anthropic, OpenAI, Perplexity, Google
**Models**: 9 different model strings with fallback chains

**Symptoms**:
- Provider fallback logic was untested code paths
- Most users never needed beyond Level 2
- Adding a new provider required touching 5+ files
- Model selection confused users

**Lesson**: Chat needs 1-2 providers max. Start simple, add complexity only when usage data proves the need.

### 8. Data Pipeline Fragility

Data ingestion involved multiple languages (Python scripts, TypeScript ETL, shell scripts), multiple formats (CSV, JSON, SQL), and multiple manual steps.

**Pipeline flow (at worst)**:
```
Python scraper → CSV → Python cleaner → JSON → Shell script → SQLite → SQL dump → D1 import
```

**Symptoms**:
- `scripts/` directory had 12+ files in different languages
- No pipeline orchestration: each step run manually
- Failures detected only when data didn't appear in the app
- No validation between pipeline stages

**Lesson**: Single-language pipeline, automated orchestration, validation at every stage, alerting on failure.

### 9. Stack Trace Leakage

Error responses included `e.stack` sent directly to the client. This exposed internal file paths, line numbers, and call stacks to anyone who triggered an error.

**Symptoms**:
- Attackers could map the codebase structure from error messages
- Internal server paths visible in network responses
- `500 Internal Server Error` responses included 20+ line stack traces

**Lesson**: Never send stack traces to clients. Log server-side, return generic error IDs that can be correlated with logs.

### 10. Mobile Ignored

The entire app was designed for desktop viewports. No responsive breakpoints, no touch interactions, no mobile navigation. For a retail investment app in Indonesia — a mobile-first market — this was a critical miss.

**Symptoms**:
- UI broken below 1024px width
- No touch-friendly targets (buttons too small)
- No mobile navigation pattern (no bottom nav, no drawer)
- `overflow-x: auto` on every page

**Lesson**: Design mobile-first from day 1. Indonesia has 350M+ mobile connections. Desktop-only excludes the majority of users.

---

## What Must Be Repeated

1. **Deterministic engine pattern** — Pure functions, no AI in math, testable and auditable
2. **DB as single source of truth** — One database, no fallback chains, no file-based data
3. **BPS concept** — Data-driven DCA scoring; improve implementation with better normalization
4. **ADR decision records** — Append-only, dated, contextualized decisions
5. **DataStatus transparency** — Honest freshness indicators build user trust
6. **Circuit breaker pattern** — Provider failure isolation prevents cascading failures
7. **Parameterized SQL queries** — Never concatenate user input into SQL strings

## What Must NOT Be Repeated

1. **Monolithic API file** — One endpoint = one file
2. **Regex for structured data** — JSON schema or nothing
3. **Multiple data representations** — One DB, one schema, one source
4. **Dev-session bypass** — Environment-specific builds, never runtime flags
5. **API key in URL** — Headers or env vars only
6. **Stack trace in error response** — Log server-side, return error IDs
7. **`any` types** — Every function signature must be fully typed
8. **Bundle dev tools in production** — Dead code elimination at build time
