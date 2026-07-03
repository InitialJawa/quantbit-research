# 00 — Executive Summary: QuantBit V2 Blueprint

## The Problem

QuantBit v1 was built iteratively over 6 months by a solo developer exploring IDX stock portfolio management. It works as a functional proof-of-concept but suffers from 5 fatal architectural flaws:

| # | Flaw | Impact |
|---|------|--------|
| 1 | **Triple data sources** (JSON + SQLite + D1 + in-memory singletons) | Same data has different values depending on which source is read |
| 2 | **Monolithic API** — 1236-line `[[path]].ts` mixing auth, market, AI, email | Cannot test, maintain, or extend individual routes |
| 3 | **No single source of truth** — engine, backtest, and production read from different sources | Backtest results differ from live portfolio behavior |
| 4 | **Security bypasses** — dev-session auth, API key in URL, stack trace leakage | Production exposes sensitive information |
| 5 | **Hidden knowledge** — 50+ magic numbers, hardcoded assumptions, undocumented business rules | Any new developer would misinterpret core logic |

## What We Built That Works

Despite the flaws, QuantBit v1 produced genuine innovation:

- **Buy Pressure Score (BPS)** — 5-factor adaptive DCA engine (valuation/momentum/breadth/drawdown/fear) with crisis override
- **Weighted Factor Ranking** — quality × growth × value × momentum × dividend scoring with 3 investment profiles (AMAN/AGRESIF/DIVIDEN)
- **Crash Protection** — 2-tier detection (fast crash + slow grind) with safe haven (gold/cash) allocation
- **Market Regime Engine** — 5-state decision tree (GOLD_DEFENSE → RISK_ON) with breadth analysis
- **AI Chat with 4-Level Architecture** — Q&A → Read Tools → Action Approval → Proactive Agent
- **24 Features** that form a complete stock portfolio management system

## The Path Forward

Build QuantBit V2 **from scratch** using all knowledge from v1. Apply the following 8 principles:

1. **Single Source of Truth** — D1 database as THE source; no JSON, no SQLite, no in-memory copies
2. **Domain-Modular API** — Hono framework with per-domain route modules
3. **Zod Validation** — Every API input validated at the boundary
4. **Edge-Native Architecture** — Cloudflare Pages Functions + D1 + Workers KV cache
5. **Normalized Database** — No JSON blobs, proper FKs, composite indexes, partitioning
6. **Unified Scoring** — One scoring algorithm, computed server-side, stored in DB
7. **Structured AI Output** — No regex parsing, typed tool calls from the model
8. **Deterministic Financial Math** — No AI in calculations, AI only for Q&A and recommendations

## Blueprint Summary

| Aspect | V1 | V2 |
|--------|----|----|
| API Framework | Manual if/else routing | Hono with domain modules |
| Database | JSON + SQLite + D1 (3 copies) | D1 only (single source) |
| Validation | None (any → cast) | Zod schemas on every endpoint |
| Auth | Custom PBKDF2 + "dev-session" bypass | Better Auth integration |
| Scoring | Split between pipeline + force-sync | Unified server-side algorithm |
| AI Output | Regex-extracted tool calls | Structured output from model |
| Data Pipeline | Python + TypeScript scripts | Single unified pipeline |
| Backtest Data | Build-time JSON imports | Server-side D1 queries |
| UI State | localStorage + D1 (dual sources) | D1 SOT, localStorage cache only |

## Rebuild Effort Estimate

| Phase | Duration | Outcome |
|-------|----------|---------|
| Sprint 0: Foundation | 2 days | Hono + D1 + Zod scaffold, CI/CD |
| Sprint 1: Database | 3 days | Schema, migrations, seed scripts |
| Sprint 2: Engine | 5 days | Core engine, ranking, allocation, crash detection |
| Sprint 3: API | 3 days | All domain endpoints with validation |
| Sprint 4: UI | 5 days | Complete UI with all 4 tabs |
| Sprint 5: Backtest UI | 3 days | Backtest integration in UI |
| Sprint 6: AI | 4 days | AI chat with structured output |
| Sprint 7: Pipeline | 3 days | Data pipeline, cron, automation |
| Sprint 8: Polish | 3 days | Testing, documentation, deployment |
| Sprint 9: Launch | 2 days | Production deployment, monitoring |
| **Total** | **33 days** | Complete QuantBit V2 |

## Key Decisions

| Decision | Rationale |
|----------|-----------|
| **Must rebuild from scratch** | V1 architecture cannot be refactored incrementally (monolithic API, triple data sources, no validation) |
| **D1 over Postgres** | Already on Cloudflare ecosystem, edge-native, zero cold-start with prewarm, no separate DB server |
| **Hono over Express** | Edge-native (Cloudflare), type-safe, lightweight, middleware ecosystem |
| **Better Auth over custom** | V1 auth had dev bypass, API key in URL, non-standard password storage |
| **AI structured output** | Eliminates fragile regex parsing, improves reliability |
| **No MCP in MVP** | Experimental feature, not critical for core functionality |
| **No adaptive weights** | Not proven effective in v1 testing — deprioritized |
| **No multiple AI providers** | Start with one (OpenRouter), add fallback providers later |
