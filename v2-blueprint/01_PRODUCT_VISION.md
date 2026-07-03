# 01 — Product Vision

## Vision Statement

QuantBit is a terminal-styled web application for Indonesian stock (IDX) portfolio management that combines deterministic financial scoring with AI-assisted decision support. It helps retail investors make data-driven decisions without relying on AI for financial calculations.

## Original Vision (V1)

**Source:** Project never had an explicit PRD. Vision extracted from codebase naming, BEHAVIOR.md, and AGENTS.md.

QuantBit started as an AI-assisted portfolio manager for individual IDX investors. The original vision was:
- "A second brain for your portfolio"
- Real-time market data + AI chat for stock analysis + automated portfolio tracking
- Terminal aesthetic (black+green) for the power-user feel

**What went right:** The terminal aesthetic, the AI chat integration, the BPS scoring
**What went wrong:** The data architecture grew organically without a plan

## Target Users

| Persona | Needs | How QuantBit Helps |
|---------|-------|-------------------|
| **Retail IDX Investor** (Primary) | Track portfolio, get buy/sell signals, understand market regime | BPS dashboard, crash detection, AI explanations |
| **Backtest Power User** (Secondary) | Test strategies before deploying | Full backtest engine with configurable profiles |
| **Dividend Hunter** (Niche) | Track dividend income, forecast forward dividends | Dividend tracking, DIVIDEN profile, forward forecast |
| **AI Curious Investor** (Niche) | Ask questions about portfolio in natural language | AI chat with 4-level architecture |

## Business Value Delivered

| Value | How Achieved | Evidence from V1 |
|-------|-------------|------------------|
| **Data-driven decisions** | Weighted factor ranking removes emotional bias | 5-factor formula with debug-grade transparency |
| **Crash protection** | Automatic gold/cash allocation during market downturns | `crashDetector.ts` with 2-tier detection |
| **Time savings** | AI chat answers portfolio questions instantly | 8 read-only tools for market/portfolio queries |
| **Strategy validation** | Backtest before deploying real money | Full 1500+ day backtest engine |
| **Dividend visibility** | Forward dividend forecasting | `ForwardDividendsForecast.tsx` + dividend cache |

## What NOT to Build (V2 Scope Exclusion)

| Feature | Reason for Exclusion | Evidence from V1 |
|---------|---------------------|------------------|
| **MCP Server** | Experimental, never integrated with main UI | `src/mcp/index.ts` — standalone, no UI |
| **Adaptive Weights** | Not proven effective, unstable | `ranker.ts:56-147` — marked experimental |
| **Multiple AI Providers** | Adds complexity without user-visible benefit | 7 providers with circuit breaker — 90%+ traffic via OpenRouter |
| **Proactive Agent (L4)** | Requires complex state management, low usage | `systemKnowledge.ts` Section 14 — 6 BPS rules, never validated |
| **Strategy Comparison** | CLI tool, not integrated with UI | `scripts/compare_strategies.ts` |
| **Weight Grid Search** | Research tool, not user-facing | `scripts/backtest_optimize_weights.ts` |
| **Single Ticker Mode** | Legacy, no UI exposes it | `singleSellTrigger`/`singleBuyTrigger` fields |
| **Email Notifications** | Resend dependency, only 1 feature uses it | `functions/api/[[path]].ts:629-647` |
| **WebSocket Real-time** | Never implemented (stub only) | `src/services/api.ts` — commented out WebSocket |

## V2 Product Pillars

1. **Deterministic Engine** — All financial math is pure, tested, and auditable. No AI in calculations.
2. **Single Source of Truth** — Every data point lives in exactly one place. No duplicates.
3. **Modular Architecture** — Each domain (auth, market, portfolio, engine, AI) is independent.
4. **Developer Experience** — Zod validation, TypeScript strict, comprehensive testing, clear documentation.
5. **Edge-Native** — Runs on Cloudflare edge for low latency, zero server management.
