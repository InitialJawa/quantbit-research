# 13 — Backend Architecture

> Backend architecture for QuantBit V2 — Hono-based, domain-modular, edge-native.

---

## Technology Stack

| Layer | Technology | Rationale |
|-------|-----------|-----------|
| Runtime | Cloudflare Pages Functions | Edge-native, zero cold-start, free tier |
| Framework | **Hono** | Type-safe, lightweight, Cloudflare-native, middleware ecosystem |
| Database | **D1** (Cloudflare) | SQLite-compatible, edge-replicated, no server management |
| Cache | **Workers KV** | Read-optimized, global distribution |
| Auth | **Better Auth** | Production-ready auth, OAuth support, session management |
| Validation | **Zod** | Already in v1 dependencies, type-safe schemas |
| AI | OpenRouter API | Aggregates multiple providers, handles fallback |
| Email | Resend (optional) | Simple transactional email API |

---

## API Structure

```
functions/api/v1/
├── auth/
│   ├── register.ts      POST   /api/v1/auth/register
│   ├── login.ts         POST   /api/v1/auth/login
│   ├── logout.ts        POST   /api/v1/auth/logout
│   └── me.ts            GET    /api/v1/auth/me
├── market/
│   ├── overview.ts      GET    /api/v1/market/overview
│   ├── stocks.ts        GET    /api/v1/market/stocks
│   ├── tickers.ts       GET    /api/v1/market/tickers
│   ├── regime.ts        GET    /api/v1/market/regime
│   ├── fundamentals.ts  GET    /api/v1/market/fundamentals/:ticker
│   └── backtest-data.ts GET    /api/v1/market/backtest-data
├── portfolio/
│   ├── index.ts         GET    /api/v1/portfolio
│   ├── trade.ts         POST   /api/v1/portfolio/trade
│   ├── watchlist.ts     GET    /api/v1/portfolio/watchlist
│   └── history.ts       GET    /api/v1/portfolio/history
├── engine/
│   ├── bps.ts           GET    /api/v1/engine/bps
│   ├── run-backtest.ts  POST   /api/v1/engine/run-backtest
│   ├── backtest.ts      GET    /api/v1/engine/backtest/:id
│   └── backtests.ts     GET    /api/v1/engine/backtests
├── ai/
│   ├── chat.ts          POST   /api/v1/ai/chat
│   └── sessions.ts      GET    /api/v1/ai/sessions
└── admin/
    ├── health.ts        GET    /api/v1/admin/health
    └── tickers.ts       PATCH  /api/v1/admin/tickers
```

**Compare with V1:** Single `[[path]].ts` → modular directory structure. Each file < 200 lines.

---

## Middleware Stack

```
Request
  → CORS middleware
  → Rate limit middleware
  → Auth middleware (except public endpoints)
  → Request validation (Zod)
  → Response envelope middleware
  → Error handler middleware
  → Route handler
  → Response
```

### CORS
```typescript
app.use("*", cors({
  origin: ["https://quantbit.pages.dev", "http://localhost:5173"],
  allowMethods: ["GET", "POST", "PATCH", "DELETE"],
  allowHeaders: ["Content-Type", "Authorization"],
  credentials: true,
}));
```

### Auth Middleware
```typescript
app.use("/api/v1/portfolio/*", async (c, next) => {
  const session = await getSession(c);
  if (!session) return c.json({ ok: false, error: "Unauthorized" }, 401);
  c.set("userId", session.userId);
  await next();
});
```

### Response Envelope
```typescript
interface ApiResponse<T = unknown> {
  ok: boolean;
  data?: T;
  error?: {
    code: string;
    message: string;
    details?: unknown;
  };
  pagination?: {
    page: number;
    perPage: number;
    total: number;
    totalPages: number;
  };
}
```

---

## Engine Deployment (V2 Change)

**V1:** Engine ran client-side in the browser. Backtest imported JSON files at build time.

**V2:** Engine runs server-side on Cloudflare edge:
1. Backtest triggered via `POST /api/v1/engine/run-backtest`
2. Server queries D1 for market data
3. Server executes backtest (same pure TS engine)
4. Server stores results in D1 `backtest_sessions` + `backtest_logs`
5. Server returns summary + chart data to frontend

**Benefits:**
- Uses D1 (single SOT) instead of build-time JSON imports
- Backtest results are consistent with live portfolio
- Backtest history persists across sessions
- Large computations run on edge, not in user's browser

---

## Error Handling Strategy

```typescript
// Central error handler
app.onError((err, c) => {
  console.error(`[${c.req.method}] ${c.req.url}:`, err);
  
  if (err instanceof ZodError) {
    return c.json({
      ok: false,
      error: {
        code: "VALIDATION_ERROR",
        message: "Request validation failed",
        details: err.flatten(),
      },
    }, 400);
  }
  
  return c.json({
    ok: false,
    error: {
      code: "INTERNAL_ERROR",
      message: "An internal error occurred",
    },
  }, 500);
});
```

**Key rules:**
- NEVER return stack traces to client
- Log full errors to Cloudflare console
- Return structured error codes for programmatic handling
- 400 for validation, 401 for auth, 403 for forbidden, 404 for not found, 500 for server errors

---

## Rate Limiting

```typescript
// Per-user rate limits
const RATE_LIMITS = {
  "/api/v1/ai/chat": { window: 60_000, max: 10 },  // 10 requests/min
  "/api/v1/engine/run-backtest": { window: 300_000, max: 3 },  // 3 per 5 min
  "/api/*": { window: 60_000, max: 100 },  // Default: 100 req/min
};
```

---

## Backend Security

- **No API keys in URLs** — All secrets via `Authorization` header or env vars
- **No dev bypass** — Auth middleware returns 401 on invalid/absent session
- **No stack traces** — Sanitized error responses
- **Rate limiting** — Prevent abuse on AI and backtest endpoints
- **Input validation** — Zod rejects malformed requests before handlers execute
- **CORS** — Explicit allowed origins
