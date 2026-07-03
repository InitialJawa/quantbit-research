# 18 — Final Recommendation

> **Should Quantbit be rebuilt from scratch?**
>
> Yes — but with clear boundaries on what is rebuilt and what is carried forward.

---

## Executive Summary

### Current State
Quantbit v1 is a **functional proof of concept** with:
- ✅ Validated business logic (BPS, ranking, crash detection)
- ✅ Working data pipeline
- ✅ Deterministic engine
- ❌ Critical architectural debt (monolithic API, regex parsing, triple data sources)
- ❌ Security issues (dev bypass, key in URL, stack trace leaks)
- ❌ Maintainability problems (1236-line API file, hardcoded data, global singletons)

The system works, but it cannot scale, cannot be maintained by a team, and cannot be deployed confidently.

### Recommendation
**Rebuild from scratch** — but keep the business logic, algorithms, and concepts.

**Do not re-implement** the architecture, file structure, state management, or API design.

---

## What Must Be Kept (Knowledge)

These are the intellectual assets from v1 that **must be carried forward**:

| Asset | Value | Source |
|-------|-------|--------|
| BPS Algorithm | Validated DCA strategy | `src/engine/buyPressure.ts` |
| Weighted Ranking | Core of quantitative investing | `src/engine/ranker.ts` |
| Crash Detection | Risk management essential | `src/engine/crashDetector.ts` |
| Metrics Formulas | Standard financial math | `src/engine/metrics.ts` |
| 3 Profiles | Investment framework | `src/engine/types.ts`, profiles config |
| Fee Model | Realistic IDX trading costs | `src/engine/allocator.ts` |
| Regime Decision Tree | Market state classification | `src/marketRegimeEngine.ts` |
| Dividend Accrual | Indonesian tax (90% net) | `src/engine/core.ts:243-273` |
| DataStatus Concept | Transparency pattern | `src/types/DataStatus.ts` |
| Circuit Breaker | AI resiliency | `src/server/aiChatHandler.ts` |
| ADR Documentation | Decision traceability | `docs/ADR-*.md` |

## What Must Be Preserved (Code Patterns)

Not copy-pasted, but **re-implemented using the same logic**:

1. **EngineConfigContext pattern**: useReducer + localStorage for strategy settings — excellent pattern
2. **Pure function engine**: No side effects in scoring functions — testable, predictable
3. **Parameterized SQL**: `env.DB.prepare("SELECT ...").bind(params)` — SQL injection safe
4. **Immutable migrations**: `CREATE TABLE IF NOT EXISTS` + never modify existing migration
5. **Provider chain pattern**: Sequential fallback with circuit breaker — resilient design
6. **Zero auto-execute**: All AI actions require user approval — safety-critical

## What Must Be Discarded (Full Rewrite)

These parts of v1 are **not worth preserving**:

| Component | Problem | New Approach |
|-----------|---------|--------------|
| `[[path]].ts` (1236 lines) | Monolithic, mixed concerns | Modular Hono routes (`/api/v1/auth/`, `/api/v1/data/`, etc.) |
| `toolCallParser.ts` | Regex parsing | Structured JSON schema from AI |
| `marketData.ts` (MKT/RS/L) | Global mutable singletons | Dependency injection, query-based |
| `stocksData.ts` (830 tickers) | Hardcoded data | Database-driven ticker list |
| `server.ts` | Express + CF divergence | Unified Hono server (dev + prod) |
| Data files (`data/*.json`) | Triple source | Single D1 database |
| `aiChatHandler.ts` | 889 lines, over-engineered | Simplified 2-provider chain |
| `core.ts` (runStrategy 670 lines) | Monolithic function | Modular service architecture |
| Dev-session bypass | Security hole | Proper auth only |
| AITestHarness | Dev tools in production | Feature flags or separate build |

## What Must Be Deferred

These features from v1 are **not essential for MVP**:

| Feature | Rationale | Target |
|---------|-----------|--------|
| Adaptive Weights | Untested, adds complexity | Post-MVP v2.1 |
| MCP Server | No proven use case | Post-MVP v2.2 |
| Multiple AI providers | 1 good provider > 5 flaky ones | Post-MVP v2.1 |
| Proactive AI agent | Confusing to users | Post-MVP v2.2 |
| Email notifications | Nice-to-have | Post-MVP v2.1 |
| Strategy comparison | Rarely used | Post-MVP v2.5 |
| Weight optimizer | Over-engineering | Post-MVP v2.5 |
| Legacy Gemini endpoints | Not used, security risk | Delete entirely |

## Recommended Architecture

### Tech Stack (v2)

| Layer | Technology | Rationale |
|-------|-----------|-----------|
| Framework | **Hono** | Edge-native, TypeScript, file-based routing |
| Database | **Cloudflare D1** | SQLite-compatible, edge-native, single source |
| Frontend | **React 19 + Vite + Tailwind 4** | Stable, fast, familiar |
| Charts | **Recharts** | Battle-tested, React-native |
| AI | **OpenRouter** (primary) + **Cohere** (fallback) | No geo-blocking, generous free tier |
| Validation | **Zod** | Schema validation, type generation |
| Auth | **Better Auth** or **Hono auth middleware** | PBKDF2/Argon2, proper session management |
| CI/CD | **GitHub Actions** | Same as v1, proven |
| Testing | **Vitest + Playwright** | Same as v1, proven |

### Architecture Principles
1. **Modular API** — every resource in its own file
2. **Single data source** — D1 only, no file fallbacks
3. **Structured AI output** — JSON schema, no regex
4. **Strict types** — no `any`, Zod everywhere
5. **DI over singletons** — services receive dependencies
6. **Mobile-first responsive** — not desktop-only
7. **Security from day 1** — no bypass, no keys in URLs, no stack traces
8. **Simple AI** — 2 providers max, streaming, structured tools

### Database Design (v2)
- **17 tables** (v1 had 14), all with proper FKs
- **No JSON blobs** — every field is structured
- **Composite indexes** on (date, ticker_id) for market data
- **Separate salt and hash** columns for passwords
- **Proper audit trail** with created_at/updated_at
- **ticker_id FK** — consistent, no string manipulation

### Engine Design (v2)
- **Modular services** — ranking/, bps/, backtest/, metrics/, risk/, rebalance/
- **Pure functions** — no side effects, all inputs explicit
- **Dependency injection** — services receive DB adapter, not import singletons
- **Configurable profiles** — seed data, not hardcoded in multiple places
- **Validation layer** — Zod schemas for engine inputs

---

## Build Order

### Phase 1 — Foundation (Sprin 0-1)
```
Data infrastructure → Database schema → Tick seed → API skeleton
```
Why first: Everything depends on data. Get this right.

### Phase 2 — Engine (Sprint 2-3)
```
Ranking → BPS → Crash → Backtest → Metrics → Rebalancing
```
Why second: Core value proposition. Validate business logic early.

### Phase 3 — Delivery (Sprint 4-6)
```
API endpoints → UI core → Backtest UI → Charts
```
Why third: Build interface for the engine.

### Phase 4 — AI (Sprint 7)
```
AI client → Chat UI → Tool system
```
Why last: AI is presentation layer. Engine works without it.

### Phase 5 — Polish (Sprint 8-9)
```
Responsive → Testing → Security → Documentation → Launch
```

---

## Verdict

| Criteria | Score | Notes |
|----------|-------|-------|
| Business logic | ✅ Excellent | BPS, ranking, crash detection validated |
| Architecture | ❌ Poor | Monolithic API, regex, singletons |
| Security | ❌ Poor | Dev bypass, key in URL, stack traces |
| Performance | ⚠️ Adequate | O(n²) fixed, but no caching |
| Maintainability | ❌ Poor | 1236-line file, mixed concerns |
| Testability | ⚠️ Medium | Unit tests exist, but hard to add |
| Scalability | ❌ Poor | Single-file API, no horizontal scaling |
| UX | ⚠️ Medium | Good desktop, no mobile |
| Data quality | ⚠️ Medium | Synthetic fundamentals, dual sources |

**Final Verdict**: Rebuild from scratch. Keep the algorithms. Discard the implementation.

**Estimated effort**: 10-12 sprints (2.5-6 months depending on team size)

**Risk-adjusted confidence**: 75% (15% for data pipeline, 10% for formula accuracy, mitigated by validation)

---

## One Last Thing

Before starting the rebuild, **run the old backtest engine on known data and save the results**. These results become your regression test suite for the new engine. If v2 produces different numbers from v1 on the same inputs, either v2 has a bug or v1 had a bug — either way, you'll catch it early.

This is the single most important validation step for the entire rebuild.
