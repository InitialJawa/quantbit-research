# API Endpoint Audit — Quantbit v1

**Source:** `functions/api/[[path]].ts` (1236 lines)  
**Date:** 2026-07-03

---

## Issue 1: Monolithic File — 1236 Lines, Mixed Concerns

The entire API surface lives in one file. Auth, data CRUD, market data, AI chat, email, and external price proxies are all handled in sequence via if/else branches. This makes it impossible to:
- Test routes in isolation
- Add middleware selectively
- Change one domain without risk to others
- Reason about request lifecycle

**Fix v2:** Split into domain modules:
- `functions/api/auth.ts` — signup, login, logout, me
- `functions/api/portfolio.ts` — portfolio CRUD
- `functions/api/watchlist.ts` — watchlist CRUD
- `functions/api/trade-logs.ts` — trade log CRUD
- `functions/api/market.ts` — backtest-data, fundamentals, idx80, sync-status
- `functions/api/engine.ts` — engine state, force-sync
- `functions/api/ai.ts` — AI chat, sessions, messages (unified + legacy)
- `functions/api/proxy.ts` — GoAPI, Yahoo live prices
- `functions/api/admin.ts` — notifications, health

Use Cloudflare Pages Functions directory routing (`functions/api/auth/signup.ts`).

---

## Issue 2: No API Versioning

All endpoints are at `/api/...` with no version prefix. Breaking changes (e.g., renaming `buy_price` to `buyPrice` in response) would break all clients simultaneously.

**Fix v2:** Prefix all endpoints with `/api/v1/`. Keep `/api/...` as a backward-compat redirect during migration, then drop in v2 release.

---

## Issue 3: Inconsistent Naming — signup vs login, snake_case vs camelCase

`POST /api/auth/signup` — inconsistent with login (no "signin"). Response bodies mix conventions: some endpoints return `{success: true, data: ...}`, others return `{user: ..., session: ...}`, others return `{watchlist: ...}`, others return `{stocks: ...}`.

**Fix v2:** Consistent naming:
- `POST /api/v1/auth/register` (instead of signup)
- All responses use `{ok: boolean, data?: T, error?: string}` envelope
- DB `snake_case` → API `camelCase` via explicit mapper (not `rows.results` passed through)

---

## Issue 4: No Request Validation — Zod Not Used Despite Being in Dependencies

Every endpoint reads `request.json() as any`. No type validation, no schema enforcement. A malformed request (missing required fields, wrong types) would either crash with 500 or silently write bad data.

**Fix v2:** Define Zod schemas for every request body:
```ts
const SignupSchema = z.object({
  email: z.string().email(),
  password: z.string().min(8),
  name: z.string().optional(),
});
```
Validate in a middleware wrapper. Return structured 400 errors on validation failure.

---

## Issue 5: Dev Bypass — "dev-session" Token Returns "dev-user"

`getUserFromSession()` (line 57) returns `"dev-user"` when:
- No token provided (`!token → "dev-user"`)
- Token is "dev-session" (`token === "dev-session" → "dev-user"`)
- Session lookup fails (catch → `"dev-user"`)

This means AI endpoints (chat, sessions, messages) never require real authentication. Any unauthenticated request can create sessions and send messages.

**Fix v2:** Remove the "dev-session" bypass. AI endpoints should require real auth. If dev mode is needed, gate it behind `env.DEV_MODE === "true"` and log warnings. The `catch → "dev-user"` fallback should return `null` and let the endpoint return 401.

---

## Issue 6: Legacy Endpoints — `/api/gemini/*` Unused

Three legacy Gemini endpoints (`/api/gemini/analyze`, `/api/gemini/market-summary`, `/api/gemini/chat`) are kept for backward compatibility but no frontend code calls them anymore. The unified `/api/ai/chat` endpoint handles all AI interactions now.

**Fix v2:** Remove in v2. Add a transitional 301 redirect if needed during migration window.

---

## Issue 7: Hardcoded Responses — Default Engine State, Market Sync Error

`GET /api/engine/state` (line 204-220) returns hardcoded default state with BBCA/BBRI positions and hardcoded config. `POST /api/market/sync` (line 670-672) always returns a hardcoded error.

**Fix v2:** Remove hardcoded defaults. Return meaningful errors (e.g., "No engine state saved yet" with 404). `market/sync` should either work or be removed entirely.

---

## Issue 8: API Key in URL — GoAPI

Line 1005: `fetch(`https://api.goapi.io/stock/idx/prices?api_key=${key}`)` — API key in query string. This gets logged by Cloudflare, any proxy, and potentially by GoAPI's own logs.

**Fix v2:** Use `Authorization: Bearer` header instead of query parameter. If GoAPI doesn't support header-based auth, add a warning comment and consider rotating the key regularly.

---

## Issue 9: Stack Trace Leakage — Internal Error Messages

Line 699: `"Internal error: " + (e.message || e.stack || "Unknown")` — stack traces are returned to the client in production. This exposes internal code structure, file paths, and potentially sensitive logic.

**Fix v2:** Return `{error: "Internal server error"}` with status 500 in catch. Log the full error to console/server logs but never expose it to client. Use `env.ENVIRONMENT` to gate verbose errors to dev only.

---

## Issue 10: No Pagination — /api/backtest-data and /api/trade-logs

`GET /api/backtest-data` (line 354) returns ALL data for the selected year range with no pagination. For a 5-year backtest with 80 tickers, this is ~120K rows compressed into one response. `GET /api/trade-logs` returns ALL user trade logs with no limit.

**Fix v2:** Add `?page=1&perPage=100&offset=` parameters. For backtest-data, this is less critical (data is processed server-side anyway), but trade-logs absolutely needs pagination. Also add `?limit=100` fallback if no params provided.

---

## Issue 11: Inconsistent Error Responses

Three different error envelope patterns coexist:
- `{error: "message"}` — most endpoints (line 36-38)
- `{content: "message", diagnostic: {...}}` and non-200 status — AI chat error (line 842-846)
- `{success: false, error: "message"}` — GoAPI fallback (line 1018)

Clients must handle all three. This causes bugs when the frontend expects `response.error` but gets `response.content`.

**Fix v2:** Standardize on `{ok: false, error: {code: string, message: string, details?: any}}` for all error responses. Non-200 HTTP status codes for transport-level issues (401, 403, 404, 500). Never return errors as 200 with `success: false`.

---

## Issue 12: Weak Auth Check — force-sync Uses Secret in Body

Line 617: `body.secret === env.CRON_SECRET` — the cron secret is sent in the request body as JSON. This means it appears in Cloudflare logs, request logs, and any intermediary. Also, the CRON job trigger (GitHub Actions) sends this secret as POST body.

**Fix v2:** Use `Authorization: Bearer <secret>` header instead. Add signature-based verification (e.g., HMAC with timestamp + body hash) for replay protection.

---

## Issue 13: Duplicate AI Handlers — Unified Chat + Legacy Gemini

The unified `/api/ai/chat` handler (line 675) and legacy Gemini handlers (lines 689-691) both exist. The unified handler uses a sophisticated provider-fallback chain (Gemini → Groq → OpenRouter), while the Gemini legacy only calls Gemini directly.

**Fix v2:** Drop legacy handlers. Route old `/api/gemini/*` calls to the unified handler internally with a deprecation warning header.

---

## Issue 14: CORS Not Explicit — Uses Default Cloudflare Behavior

There is no explicit CORS configuration. Cloudflare Pages Functions do not add CORS headers by default. If the frontend runs on a different domain or port, all API calls will fail with CORS errors.

**Fix v2:** Add explicit CORS headers at the top of the request handler:
```ts
const corsHeaders = {
  "Access-Control-Allow-Origin": "*",
  "Access-Control-Allow-Methods": "GET, POST, PATCH, DELETE, OPTIONS",
  "Access-Control-Allow-Headers": "Content-Type, Authorization, Cookie",
};
if (method === "OPTIONS") return new Response(null, { headers: corsHeaders });
```
Scope the `Allow-Origin` to the specific frontend domain in production.
