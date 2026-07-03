# 17 — Security

> Security analysis and recommendations for QuantBit V2.

---

## Security Posture Comparison

| Aspect | V1 Status | V2 Target |
|--------|-----------|-----------|
| Authentication | Custom PBKDF2 + "dev-session" bypass | Better Auth with proper session management |
| API Keys | In URL query string (`?api_key=`) | Authorization header |
| Error Messages | Stack traces returned to client | Sanitized errors |
| Auth Fallback | `getUserFromSession()` returns "dev-user" on error | Returns null → 401 |
| Secrets | In POST body (`body.secret`) | Authorization header |
| Input Validation | `request.json() as any` | Zod schemas on every endpoint |
| CORS | Not explicit | Explicit allowed origins |
| Rate Limiting | None | Per-endpoint rate limits |
| Password Storage | `salt:hash` in single column | bcrypt or PBKDF2 with standard format |

---

## V1 Security Issues (All Fixed in V2)

### P0 — Critical

| # | Issue | File | Line | Severity |
|---|-------|------|------|----------|
| 1 | Dev-session bypass — no token → "dev-user" | `[[path]].ts` | 57 | P0 |
| 2 | Dev-session bypass — any error → "dev-user" | `[[path]].ts` | 69 | P0 |
| 3 | GoAPI key in URL query string | `[[path]].ts` | 1005 | P0 |

### P1 — High

| # | Issue | File | Line | Severity |
|---|-------|------|------|----------|
| 4 | Stack trace in error response | `[[path]].ts` | 699 | P1 |
| 5 | Cron secret in POST body | `[[path]].ts` | 617 | P1 |
| 6 | No input validation (any → cast) | `[[path]].ts` | everywhere | P1 |
| 7 | Weak password column format | `db/migrations/0001_init.sql` | schema | P1 |
| 8 | No CORS headers | `[[path]].ts` | (missing) | P1 |

### P2 — Medium

| # | Issue | File | Line | Severity |
|---|-------|------|------|----------|
| 9 | No rate limiting | `[[path]].ts` | (missing) | P2 |
| 10 | Hardcoded default engine state (BBCA/BBRI) | `[[path]].ts` | 205-210 | P2 |
| 11 | Inconsistent error responses (3 envelopes) | `[[path]].ts` | multiple | P2 |
| 12 | Mixed real/simulated trade entries | `db/migrations/0001_init.sql` | schema | P2 |
| 13 | AI legacy endpoints still registered | `[[path]].ts` | 689-691 | P2 |

### P3 — Low

| # | Issue | File | Line | Severity |
|---|-------|------|------|----------|
| 14 | No password complexity validation | `[[path]].ts` | 80 | P3 |
| 15 | User ID enumeration risk (sequential UUID) | `[[path]].ts` | 74 | P3 |
| 16 | Environment variables not validated at startup | `[[path]].ts` | top | P3 |
| 17 | No session expiry enforcement | `[[path]].ts` | session logic | P3 |

---

## V2 Security Architecture

### Authentication (Better Auth)

```typescript
// Better Auth handles:
// - Password hashing (bcrypt)
// - Session management (JWT + refresh)
// - OAuth providers (Google, GitHub optional)
// - Email verification (optional)
// - Rate limiting on login attempts

// No dev bypass. No custom auth logic.
// All protected routes return 401 on invalid session.
```

### API Key Management

```typescript
// NEVER in URL. Always in headers.

// Good:
Authorization: Bearer sk-or-v1-...
X-API-Key: goapi_...

// Bad (v1):
?api_key=goapi_...
```

### Input Validation (Zod)

```typescript
// Every endpoint validates input before processing
const TradeSchema = z.object({
  ticker: z.string().regex(/^[A-Z]{4}$/),  // IDX ticker format
  action: z.enum(["buy", "sell"]),
  shares: z.number().int().positive().multipleOf(100),  // Lot-based
});

// Malformed requests return 400 with structured error
// No (req.json() as any) anywhere
```

### Error Handling

```typescript
// NEVER return stack traces
// NEVER return internal variable names
// NEVER return SQL queries

// ALWAYS return structured error codes
{
  ok: false,
  error: {
    code: "VALIDATION_ERROR",
    message: "Invalid ticker format",
    details: { ticker: ["Must be 4 uppercase letters"] },
  },
}
```

### CORS

```typescript
const corsOptions = {
  origin: [
    "https://quantbit-v2.pages.dev",
    "http://localhost:5173",
    "http://localhost:8788",
  ],
  allowMethods: ["GET", "POST", "PATCH", "DELETE"],
  allowHeaders: ["Content-Type", "Authorization"],
  credentials: true,
  maxAge: 86400,  // Cache preflight for 24h
};
```

### Rate Limiting

```typescript
// In-memory rate limiter (per-worker)
const limits = {
  "/api/v1/ai/chat": { max: 10, window: 60_000 },       // 10/min
  "/api/v1/auth/login": { max: 5, window: 60_000 },      // 5/min
  "/api/v1/engine/run-backtest": { max: 3, window: 300_000 },  // 3/5min
  "/api/*": { max: 100, window: 60_000 },                // 100/min default
};
```

---

## Security Checklist (V2 Launch)

- [ ] Auth: Better Auth configured, no dev bypass
- [ ] API keys: All in Authorization headers, never in URLs
- [ ] Error responses: Sanitized, no stack traces, structured codes
- [ ] Input validation: Zod on every endpoint
- [ ] CORS: Explicit allowed origins
- [ ] Rate limiting: Per-endpoint limits configured
- [ ] Password: Complex password requirement
- [ ] Session: Expiry enforced, refresh tokens rotate
- [ ] Admin endpoints: Protected by ADMIN_KEY
- [ ] D1: No sensitive data in query strings
- [ ] GitHub: Secrets stored in GitHub Actions secrets
- [ ] Cloudflare: Tokens scoped to minimum required permissions
