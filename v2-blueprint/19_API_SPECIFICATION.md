# 19 — API Specification

> Complete API specification for QuantBit V2.

---

## Base URL

| Environment | URL |
|-------------|-----|
| Production | `https://quantbit-v2.pages.dev/api/v1` |
| Preview | `https://preview.quantbit-v2.pages.dev/api/v1` |
| Local dev | `http://localhost:5173/api/v1` |

---

## Response Envelope

All API responses follow this structure:

```typescript
// Success
{
  ok: true,
  data: T,
  dataStatus?: "LIVE" | "CACHED" | "STALE" | "ESTIMATED",
  pagination?: {
    page: number;
    perPage: number;
    total: number;
    totalPages: number;
  }
}

// Error
{
  ok: false,
  error: {
    code: string;      // Machine-readable
    message: string;   // Human-readable
    details?: unknown; // Validation errors, etc.
  }
}
```

---

## Authentication

### POST /auth/register
```
Request:  { email: string, password: string, name?: string }
Response: { user: User, session: Session }
Errors:   400 (validation), 409 (email exists)
```

### POST /auth/login
```
Request:  { email: string, password: string }
Response: { user: User, session: Session }
Errors:   401 (invalid credentials), 429 (rate limited)
```

### POST /auth/logout
```
Headers:  Authorization: Bearer <session_token>
Response: { ok: true }
Errors:   401 (invalid session)
```

### GET /auth/me
```
Headers:  Authorization: Bearer <session_token>
Response: { user: User, profile: UserProfile }
Errors:   401 (invalid session)
```

---

## Market

### GET /market/overview
```
Query:    (none)
Response: {
  ihsg: { close: number, change: number, changePct: number },
  gold: { close: number, change: number, changePct: number },
  usdidr: { rate: number, change: number },
  regime: RegimeState,
  lastUpdated: string,
  dataStatus: DataStatus
}
```

### GET /market/stocks
```
Query:    ?page=1&perPage=50&search=&sector=&sortBy=score&order=desc
Response: {
  stocks: Array<{
    ticker: string, name: string, sector: string,
    price: number, changePct: number,
    score: number, rank: number,
    dataStatus: DataStatus
  }>,
  pagination: Pagination
}
```

### GET /market/tickers
```
Query:    ?active=true&idx80=true&sector=
Response: { tickers: Ticker[] }
Cache:    1 hour (KV cache)
```

### GET /market/regime
```
Query:    ?history=30  // Days of regime history
Response: {
  current: RegimeState,
  history: Array<{ date: string, regime: string, marketHealth: number }>
}
```

### GET /market/fundamentals/:ticker
```
Response: {
  ticker: string,
  quality: number, growth: number, value: number,
  momentum: number, dividend: number,
  finalScore: number,
  lastUpdated: string
}
```

### GET /market/backtest-data
```
Query:    ?dateFrom=2024-01-01&dateTo=2024-12-31
Response: {
  overview: Array<{ date, ihsgClose, goldClose, usdidrRate }>,
  stocks: Array<{ date, ticker, close, adjClose, volume }>,
  scores: Array<{ ticker, scoreDate, quality, growth, value, momentum }>
}
```

---

## Portfolio

### GET /portfolio
```
Headers:  Authorization: Bearer <session_token>
Response: {
  holdings: Array<{ ticker, shares, buyPrice, currentPrice, value, gainPct }>,
  cash: number,
  gold: { grams: number, value: number },
  totalValue: number,
  bps: BuyPressureScore,
  dataStatus: DataStatus
}
```

### POST /portfolio/trade
```
Headers:  Authorization: Bearer <session_token>
Request:  { ticker: string, action: "buy"|"sell", shares: number }
Response: { trade: TradeLog, portfolio: Portfolio }
Errors:   400 (validation), 409 (insufficient funds/shares), 404 (ticker)
```

### GET /portfolio/watchlist
```
Headers:  Authorization: Bearer <session_token>
Response: { watchlist: Ticker[] }
```

### POST /portfolio/watchlist
```
Headers:  Authorization: Bearer <session_token>
Request:  { ticker: string }
Response: { watchlist: Ticker[] }
Errors:   409 (already in watchlist)
```

### DELETE /portfolio/watchlist/:ticker
```
Headers:  Authorization: Bearer <session_token>
Response: { watchlist: Ticker[] }
```

### GET /portfolio/history
```
Query:    ?page=1&perPage=50
Headers:  Authorization: Bearer <session_token>
Response: {
  trades: TradeLog[],
  pagination: Pagination
}
```

---

## Engine

### GET /engine/bps
```
Headers:  Authorization: Bearer <session_token> (optional)
Response: {
  bps: number,
  action: "none"|"small"|"normal"|"aggressive"|"deploy",
  deployPct: number,
  factors: {
    valuation: number,
    momentum: number,
    breadth: number,
    drawdown: number,
    fear: number
  },
  currentRegime: RegimeState,
  dataStatus: DataStatus
}
```

### POST /engine/run-backtest
```
Headers:  Authorization: Bearer <session_token>
Request:  {
  config: BacktestConfig,  // Strategy settings
  dateFrom?: string,
  dateTo?: string,
  profileId?: string       // Profile weights
}
Response: {
  sessionId: string,
  status: "running",
  estimatedTime: number     // Seconds
}
```

### GET /engine/backtest/:sessionId
```
Headers:  Authorization: Bearer <session_token>
Response: {
  session: BacktestSession,
  results: BacktestResults,  // CAGR, Sharpe, Sortino, etc.
  chartData: Array<{date, portfolio, benchmark, cash, gold, stocks}>,
  log: BacktestLogEntry[],
  assets: BacktestAsset[]
}
```

### GET /engine/backtests
```
Headers:  Authorization: Bearer <session_token>
Query:    ?page=1&perPage=10
Response: {
  sessions: BacktestSessionSummary[],
  pagination: Pagination
}
```

### PATCH /engine/config
```
Headers:  Authorization: Bearer <session_token>
Request:  Partial<UserStrategyConfig>
Response: { config: UserStrategyConfig }
Errors:   400 (validation)
```

### GET /engine/config
```
Headers:  Authorization: Bearer <session_token>
Response: { config: UserStrategyConfig }
```

---

## AI

### POST /ai/chat
```
Headers:  Authorization: Bearer <session_token>
Request:  {
  message: string,
  sessionId?: string,  // Continue existing session
  context?: {
    portfolio?: boolean,  // Include portfolio context
    market?: boolean,     // Include market context
  }
}
Response: {
  sessionId: string,
  response: AIResponse,   // Text + optional tool calls
  actions?: ActionProposal[]  // Pending approval
}
```

### GET /ai/sessions
```
Headers:  Authorization: Bearer <session_token>
Response: { sessions: AISessionSummary[] }
```

### GET /ai/sessions/:id
```
Headers:  Authorization: Bearer <session_token>
Response: { session: AISession, messages: AIMessage[] }
```

---

## Admin

### GET /admin/health
```
Headers:  Authorization: Bearer <admin_key>
Response: {
  status: "ok" | "degraded",
  version: string,
  database: { status: "ok" | "error", latency: number },
  dataPipeline: { lastRun: string, status: "success" | "failed" },
  uptime: number
}
```

### GET /admin/tickers
```
Headers:  Authorization: Bearer <admin_key>
Response: { tickers: Ticker[], idx80Count: number }
```

### PATCH /admin/tickers
```
Headers:  Authorization: Bearer <admin_key>
Request:  { ticker: string, is_idx80: boolean }
Response: { ticker: Ticker }
```

### POST /admin/force-sync
```
Headers:  Authorization: Bearer <cron_secret>
Response: { status: "completed", tickersUpdated: number }
```

---

## Error Codes

| Code | HTTP | Meaning |
|------|------|---------|
| VALIDATION_ERROR | 400 | Request body/schema invalid |
| UNAUTHORIZED | 401 | No/invalid session |
| FORBIDDEN | 403 | Insufficient permissions |
| NOT_FOUND | 404 | Resource doesn't exist |
| CONFLICT | 409 | Resource conflict (duplicate, insufficient funds) |
| RATE_LIMITED | 429 | Too many requests |
| INTERNAL_ERROR | 500 | Server error (never shows details) |
| PROVIDER_ERROR | 502 | External provider (AI/GoAPI) failure |
| TIMEOUT | 504 | Backend computation timeout |
