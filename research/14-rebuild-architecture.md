# Rebuild Architecture — Quantbit v2

## Core Principles

### 1. Modular API — Hono Framework

Every endpoint gets its own file. No monolithic handlers. Hono provides edge-native, TypeScript-first routing with built-in support for Cloudflare Workers, Deno, Bun, and Node.js. File-based routing convention: `routes/` directory mirrors the URL structure.

```
routes/
├── auth/
│   ├── register.ts  → POST /api/v1/auth/register
│   ├── login.ts     → POST /api/v1/auth/login
│   ├── logout.ts    → POST /api/v1/auth/logout
│   └── me.ts        → GET  /api/v1/auth/me
├── market/
│   ├── overview.ts  → GET  /api/v1/market/overview
│   ├── stocks.ts    → GET  /api/v1/market/stocks
│   └── [symbol].ts  → GET  /api/v1/market/stocks/:symbol
├── portfolio/
│   ├── index.ts     → GET  /api/v1/portfolio
│   └── rebalance.ts → POST /api/v1/portfolio/rebalance
├── backtest/
│   ├── run.ts       → POST /api/v1/backtest/run
│   └── results.ts   → GET  /api/v1/backtest/results/:id
├── ai/
│   ├── chat.ts      → POST /api/v1/ai/chat
│   └── session.ts   → GET  /api/v1/ai/sessions
└── admin/
    ├── tickers.ts   → CRUD /api/v1/admin/tickers
    └── users.ts     → GET  /api/v1/admin/users
```

### 2. Single Data Source — D1 Only

No JSON files, no SQLite fallback, no file-based caches. D1 is the sole authoritative data store. Everything reads from and writes to D1.

**Rule**: If it's not in D1, it doesn't exist. If it's in D1, that's the truth.

### 3. Structured AI Output — JSON Schema

AI responses are constrained to JSON schemas via provider-native structured output APIs. No regex parsing of natural language. If the provider doesn't support structured output, wrap the response with a JSON schema validator that rejects malformed responses.

```typescript
// Example: Zod schema for AI tool call
const BuyRecommendation = z.object({
  symbol: z.string().regex(/^[A-Z]{2,4}$/),
  quantity: z.number().int().positive(),
  reasoning: z.string().max(500),
});

const AIResponse = z.object({
  intent: z.enum(["buy", "sell", "hold", "rebalance", "info"]),
  confidence: z.number().min(0).max(1),
  tools: z.array(BuyRecommendation).max(5),
});
```

### 4. Strict Typing — No `any`

Zod for all API I/O validation and type generation. Every request body, query parameter, and response is validated against a Zod schema. Types are inferred from schemas, never handwritten.

```typescript
const RegisterSchema = z.object({
  email: z.string().email(),
  password: z.string().min(8).max(128),
  name: z.string().min(1).max(100),
});
type RegisterInput = z.infer<typeof RegisterSchema>;
```

### 5. Security First

- **No dev bypass**: Environment-specific builds, `process.env.NODE_ENV` checked at build time, not runtime
- **No keys in URLs**: All API keys in `env` or `Authorization` headers
- **No stack trace leakage**: `try/catch` returns `{ error: string, id: string }`; full error logged server-side with correlation ID
- **Rate limiting**: Per-endpoint rate limits via middleware
- **Input validation**: Every input validated by Zod before reaching business logic
- **SQL injection prevention**: Parameterized queries always (D1 binding already parameterized, but never concatenate)

### 6. Simplified AI — 1-2 Providers

Start with OpenRouter (aggregator, single API for many models). Add one fallback (Anthropic directly). No provider abstraction layer — just a thin client per provider. Streaming for all responses. Structured output via JSON schema mode.

### 7. Mobile Responsive — Tailwind from Day 1

Mobile-first responsive design. Every component designed for 320px viewport first, then expanded to desktop. Tailwind breakpoints: `sm` (640px), `md` (768px), `lg` (1024px), `xl` (1280px). Touch-friendly targets (minimum 44x44px). Bottom navigation on mobile, sidebar on desktop.

### 8. DI Over Singletons — Dependency Injection

Services receive their dependencies as constructor arguments or function parameters. No global singletons. No module-level mutable state. This enables:
- Unit testing with mock dependencies
- Clear dependency graphs
- No hidden state between requests
- Easy substitution of implementations

```typescript
// Instead of importing a singleton DB client:
class BacktestService {
  constructor(
    private readonly db: D1Database,
    private readonly ranking: RankingService,
    private readonly bps: BPSService,
  ) {}
}

// Dependencies injected at the route handler level
const app = new Hono();
app.post("/backtest/run", async (c) => {
  const backtest = new BacktestService(c.env.DB, rankingService, bpsService);
  return c.json(await backtest.run(input));
});
```

---

## Architecture Diagram

```
┌─────────────────────────────────────────────────────────────┐
│                    Frontend (React 19 + Vite)               │
│  ┌─────────┐ ┌──────────┐ ┌─────────┐ ┌──────────────┐    │
│  │  Market  │ │ Portfolio│ │Backtest │ │  Analytics   │    │
│  │   Tab    │ │   Tab    │ │   Tab   │ │     Tab      │    │
│  └────┬─────┘ └────┬─────┘ └────┬────┘ └──────┬───────┘    │
│       └────────────┴────────────┴──────────────┘           │
│                            │                                │
│                    ┌───────┴────────┐                       │
│                    │  API Client    │                       │
│                    │  (services/)   │                       │
│                    └───────┬────────┘                       │
└────────────────────────────┼────────────────────────────────┘
                             │ HTTPS
┌────────────────────────────┼────────────────────────────────┐
│           Cloudflare Workers (Hono)                          │
│  ┌─────────────────────────┼──────────────────────────┐     │
│  │       Router (/api/v1/*)                          │     │
│  │  ┌──────┐ ┌─────┐ ┌────────┐ ┌──────┐ ┌──────┐ │     │
│  │  │ Auth │ │Data │ │Backtest│ │  AI  │ │Email │ │     │
│  │  └──────┘ └─────┘ └────────┘ └──────┘ └──────┘ │     │
│  └───────────────────────────────────────────────────┘     │
│                           │                                 │
│  ┌────────────────────────┼──────────────────────────┐     │
│  │                     Services                       │     │
│  │  ┌──────────┐ ┌───────────┐ ┌──────────────────┐ │     │
│  │  │ Ranking  │ │    BPS    │ │  Crash Detection │ │     │
│  │  └──────────┘ └───────────┘ └──────────────────┘ │     │
│  │  ┌──────────┐ ┌───────────┐ ┌──────────────────┐ │     │
│  │  │ Metrics  │ │ Rebalance │ │  Market Regime   │ │     │
│  │  └──────────┘ └───────────┘ └──────────────────┘ │     │
│  └───────────────────────────────────────────────────┘     │
│                           │                                 │
│  ┌────────────────────────┼──────────────────────────┐     │
│  │                   Data Layer                       │     │
│  │         D1 Database (single source)                │     │
│  │  tickers | daily_prices | fundamentals | users     │     │
│  └───────────────────────────────────────────────────┘     │
└─────────────────────────────────────────────────────────────┘
```

---

## Key Architectural Decisions

| Decision | v1 (Old) | v2 (New) | Rationale |
|---|---|---|---|
| Framework | Pages Functions (Cloudflare) | Hono | Edge-native, TypeScript-first, file-based routing, middleware ecosystem |
| API structure | Single `[[path]].ts` | Modular routes/ | Testability, ownership, deploy independence |
| Engine | Monolithic `core.ts` | Sub-directories per concern | Separation of concerns, parallel development |
| DI pattern | Singleton imports | Constructor injection | Testability, clear dependencies |
| Data source | JSON + SQLite + D1 | D1 only | Eliminate inconsistency |
| AI parsing | Regex on text | JSON schema | Reliability, type safety |
| AI providers | 5 providers, 9 models | 2 max | Simplicity, fewer code paths |
| Validation | Manual `if` checks | Zod everywhere | Type safety, OpenAPI generation |
| Auth | Dev bypass, basic hash | Proper hashing, no bypass | Security |
| Error handling | Stack trace leak | Error ID + log | Security, debuggability |
| UI | Desktop-only | Mobile-first responsive | Market reality |
| State management | Context + singletons | DI + Context | Predictable state flow |
| Database schema | JSON blobs, no FKs | Proper relations, FKs | Integrity, queryability |

---

## Database Schema

```sql
-- Core market data
CREATE TABLE tickers (
  id          INTEGER PRIMARY KEY AUTOINCREMENT,
  symbol      TEXT NOT NULL UNIQUE,
  name        TEXT NOT NULL,
  sector      TEXT,
  industry    TEXT,
  is_active   INTEGER NOT NULL DEFAULT 1,
  created_at  TEXT NOT NULL DEFAULT (datetime('now')),
  updated_at  TEXT NOT NULL DEFAULT (datetime('now'))
);

CREATE TABLE daily_prices (
  date        TEXT NOT NULL,
  ticker_id   INTEGER NOT NULL REFERENCES tickers(id),
  open        REAL NOT NULL,
  high        REAL NOT NULL,
  low         REAL NOT NULL,
  close       REAL NOT NULL,
  adj_close   REAL NOT NULL,
  volume      INTEGER NOT NULL,
  source      TEXT NOT NULL DEFAULT 'goapi',
  PRIMARY KEY (date, ticker_id)
);

CREATE TABLE market_overview (
  date        TEXT PRIMARY KEY,
  ihsg_close  REAL,
  gold_idr    REAL,
  usdidr      REAL
);

-- Fundamental data
CREATE TABLE factor_scores (
  date        TEXT NOT NULL,
  ticker_id   INTEGER NOT NULL REFERENCES tickers(id),
  quality     REAL,
  growth      REAL,
  value       REAL,
  momentum    REAL,
  dividend    REAL,
  rank        INTEGER,
  PRIMARY KEY (date, ticker_id)
);

CREATE TABLE fundamentals (
  ticker_id   INTEGER PRIMARY KEY REFERENCES tickers(id),
  quality     REAL,
  growth      REAL,
  value       REAL,
  momentum    REAL,
  dividend    REAL,
  sector      TEXT,
  industry    TEXT,
  updated_at  TEXT NOT NULL DEFAULT (datetime('now'))
);

-- Users & auth
CREATE TABLE users (
  id            INTEGER PRIMARY KEY AUTOINCREMENT,
  email         TEXT NOT NULL UNIQUE,
  password_hash TEXT NOT NULL,
  salt          TEXT NOT NULL,
  name          TEXT NOT NULL,
  created_at    TEXT NOT NULL DEFAULT (datetime('now'))
);

CREATE TABLE sessions (
  id          INTEGER PRIMARY KEY AUTOINCREMENT,
  user_id     INTEGER NOT NULL REFERENCES users(id),
  expires_at  TEXT NOT NULL,
  created_at  TEXT NOT NULL DEFAULT (datetime('now'))
);

-- Portfolio & strategy
CREATE TABLE portfolios (
  id          INTEGER PRIMARY KEY AUTOINCREMENT,
  user_id     INTEGER NOT NULL REFERENCES users(id),
  ticker_id   INTEGER NOT NULL REFERENCES tickers(id),
  shares      REAL NOT NULL,
  buy_price   REAL NOT NULL,
  added_at    TEXT NOT NULL DEFAULT (datetime('now'))
);

CREATE TABLE watchlists (
  id          INTEGER PRIMARY KEY AUTOINCREMENT,
  user_id     INTEGER NOT NULL REFERENCES users(id),
  ticker_id   INTEGER NOT NULL REFERENCES tickers(id),
  added_at    TEXT NOT NULL DEFAULT (datetime('now')),
  UNIQUE(user_id, ticker_id)
);

CREATE TABLE strategy_configs (
  id            INTEGER PRIMARY KEY AUTOINCREMENT,
  user_id       INTEGER NOT NULL REFERENCES users(id),
  name          TEXT NOT NULL,
  config_json   TEXT NOT NULL,
  created_at    TEXT NOT NULL DEFAULT (datetime('now')),
  updated_at    TEXT NOT NULL DEFAULT (datetime('now'))
);

CREATE TABLE backtest_results (
  id              INTEGER PRIMARY KEY AUTOINCREMENT,
  user_id         INTEGER NOT NULL REFERENCES users(id),
  config_snapshot TEXT NOT NULL,
  result_json     TEXT NOT NULL,
  created_at      TEXT NOT NULL DEFAULT (datetime('now'))
);

-- AI chat
CREATE TABLE ai_sessions (
  id          INTEGER PRIMARY KEY AUTOINCREMENT,
  user_id     INTEGER NOT NULL REFERENCES users(id),
  title       TEXT NOT NULL DEFAULT 'New Chat',
  created_at  TEXT NOT NULL DEFAULT (datetime('now')),
  updated_at  TEXT NOT NULL DEFAULT (datetime('now'))
);

CREATE TABLE ai_messages (
  id          INTEGER PRIMARY KEY AUTOINCREMENT,
  session_id  INTEGER NOT NULL REFERENCES ai_sessions(id),
  role        TEXT NOT NULL CHECK(role IN ('user', 'assistant', 'system', 'tool')),
  content     TEXT NOT NULL,
  created_at  TEXT NOT NULL DEFAULT (datetime('now'))
);

-- Indexes
CREATE INDEX idx_daily_prices_ticker ON daily_prices(ticker_id);
CREATE INDEX idx_daily_prices_date ON daily_prices(date);
CREATE INDEX idx_factor_scores_date ON factor_scores(date);
CREATE INDEX idx_sessions_user ON sessions(user_id);
CREATE INDEX idx_sessions_expires ON sessions(expires_at);
CREATE INDEX idx_portfolios_user ON portfolios(user_id);
CREATE INDEX idx_ai_messages_session ON ai_messages(session_id);
```

---

## Service Architecture

```
services/
├── ranking/
│   ├── index.ts          # RankingService class
│   ├── quality.ts        # Quality factor calculation
│   ├── growth.ts         # Growth factor calculation
│   ├── value.ts          # Value factor calculation
│   ├── momentum.ts       # Momentum factor calculation
│   └── dividend.ts       # Dividend factor calculation
├── bps/
│   └── index.ts          # BPSService — BPS score calculation
├── backtest/
│   ├── index.ts          # BacktestService — orchestrator
│   ├── rebalancing.ts    # RebalancingService
│   ├── dividend.ts       # DividendService
│   ├── gold.ts           # GoldConversionService
│   └── performance.ts    # PerformanceMetricsService
├── risk/
│   ├── crash.ts          # CrashDetectionService
│   ├── volatility.ts     # VolatilityService
│   └── drawdown.ts       # DrawdownService
├── market/
│   ├── regime.ts         # MarketRegimeService
│   └── overview.ts       # MarketOverviewService
├── auth/
│   └── index.ts          # AuthService (hash, sessions, JWTs)
├── ai/
│   └── index.ts          # AIService (structured output, streaming)
└── admin/
    └── tickers.ts        # TickerAdminService
```
