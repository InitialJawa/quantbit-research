# Security Analysis — Quantbit

## Methodology
Reviewed `functions/api/[[path]].ts` (1236 lines, auth/API gateway), password hashing, session management, and API key handling. Findings ranked P0-P3.

---

## Critical (P0)

### P0.1 Dev-session bypass (`functions/api/[[path]].ts:57-72`)

```
getUserFromSession():
  62: if (!token) return "dev-user";
  67: if (token === "dev-session") return "dev-user";
  71: return row?.user_id ?? "dev-user";
```

**Issue**: Every code path returns a non-null user. No token → "dev-user". `dev-session` token → "dev-user". Invalid/expired session → "dev-user". This means **all endpoints are accessible without authentication**, including AI chat, portfolio state, and engine config.

The `dev-session` token is documented in comments (lines 63-66) as a dev-mode shortcut matching `src/services/api.ts` localStorage hack (`quantbit_session` = `"dev-session"`). This bypass works in production.

**Risk**: Any unauthenticated request gains full access to AI chat (usage costs), portfolio data, and backtest execution.

**Fix**: Remove dev bypass from production build. Return `401 Unauthorized` when no valid session. Use environment flag (`env.DEV_MODE`) to gate dev behavior.

### P0.2 Stack trace leakage (`functions/api/[[path]].ts:698-700`)

```ts
} catch (e: any) {
  return json({ error: "Internal error: " + (e.message || e.stack || "Unknown") }, 500);
}
```

**Issue**: Full `e.stack` sent to client on unhandled exceptions. Exposes internal file paths, line numbers, function names, and potentially env variable values in error messages.

**Risk**: Information disclosure. Attackers can map internal structure.

**Fix**: Return generic "Internal error" in production. Log `e.stack` server-side only. Gate with `env.PRODUCTION`.

### P0.3 API key in URL (`functions/api/[[path]].ts:1005`)

```ts
const resp = await fetch(`https://api.goapi.io/stock/idx/prices?api_key=${key}`);
```

**Issue**: GoAPI API key transmitted as URL query parameter. URLs are logged by proxies, CDNs, browser history, and server access logs. Exposed in Cloudflare Worker fetch logs.

**Fix**: Use `Authorization: Bearer` header. GoAPI may not support headers — if not, proxy through a keyless endpoint.

---

## High (P1)

### P1.1 No request validation
**Issue**: All request bodies typed as `any`. No Zod/validation library used anywhere in `functions/api/[[path]].ts`. Malformed requests pass through until they hit type errors.

**Fix**: Validate all request bodies with Zod schemas at the handler boundary.

### P1.2 Weak auth check for force-sync
**Issue**: `POST /api/engine/force-sync` checks `body.secret === env.CRON_SECRET`. Secret sent in request body — visible in request logs. No HMAC, no signature.

**Fix**: Use `Authorization: Bearer <secret>` header. Validate with constant-time comparison (`crypto.subtle.timingSafeEqual`).

### P1.3 Password storage (`functions/api/[[path]].ts`)
**Issue**: Salt and hash concatenated in single field (`salt:hash`). PBKDF2 with 100K iterations — below OWASP 2026 minimum (600K for PBKDF2-HMAC-SHA256). Single field prevents independent salt verification.

**Fix**: Store salt and hash in separate columns. Increase iterations to 600K minimum. Consider Argon2id.

### P1.4 No rate limiting
**Issue**: Auth endpoints (`/api/auth/login`, `/api/auth/register`, `/api/auth/forgot-password`) unprotected against brute force. No IP-based rate limiting.

**Risk**: Credential stuffing and brute force attacks.

**Fix**: Rate limit auth endpoints (5 attempts/min per IP). Use KV store for distributed rate limiting with Workers.

### P1.5 Session management
**`aiChatHandler.ts`**:
- Sessions expire after 30 days (D1 SQL: `datetime('now', '+30 days')`)
- No refresh mechanism — expired sessions force re-login
- No session revoke on password change — old sessions remain valid
- Checked at line 69: `expires_at > datetime('now')` — token valid even if user changed password

**Fix**: Revoke all sessions on password change. Add sliding expiry (refresh on active use). Store session creation timestamp for audit.

---

## Medium (P2)

### P2.1 No RBAC
**Issue**: All authorization is `userId`-based. No role separation (admin vs user vs read-only). Every authenticated user can access every endpoint.

**Fix**: Add `roles` table and JWT claims for role-based access control.

### P2.2 No MFA
**Issue**: No multi-factor authentication. Password-only authentication.

**Fix**: Add TOTP support via `otplib`. Store MFA secret in user record. Require on login.

### P2.3 No email verification
**Issue**: Registration accepts any email. No verification flow. Email field not validated.

**Fix**: Send verification email on registration. Block login until verified.

### P2.4 No password strength
**Issue**: No minimum length, complexity, or common-password check on registration.

**Fix**: Enforce minimum 8 characters, check against common password list, use zxcvbn for strength scoring.

### P2.5 CORS not explicit
**Issue**: Using Cloudflare Pages default CORS behavior. No explicit `Access-Control-Allow-Origin` in API handlers.

**Fix**: Set explicit CORS headers per endpoint or use `env.ALLOWED_ORIGINS`.

### P2.6 Legacy endpoints active (`/api/gemini/*`)
**Issue**: Line 690-691 routes `/api/gemini/market-summary` and `/api/gemini/chat` — legacy AI endpoints. Unmaintained attack surface.

**Fix**: Remove or gate behind feature flag. Audit for dead code paths.

---

## Low (P3)

### P3.1 No CSRF protection
**Issue**: Cookie-based auth possible (session cookie from `set-cookie` on login). No CSRF token on mutation endpoints.

**Fix**: Use `SameSite=Strict` cookie attribute. Add CSRF token for cookie-based auth.

### P3.2 HTTPS only in dev
**Issue**: Dev mode (`server.ts` Express) uses HTTP. No HTTPS enforcement middleware.

**Fix**: Redirect HTTP → HTTPS in all environments. HSTS header in production.

### P3.3 No input sanitization
**Issue**: `ticker.toUpperCase()` but no length/special-char check. Ticker values could contain injection payloads (SQLi via ticker, though D1 uses parameterized queries).

**Fix**: Validate ticker length (4-5 chars), regex `/^[A-Z]{3,5}$/` after `.replace('.JK','')`.

### P3.4 Large AI key surface
**Issue**: 7+ API keys required: OpenRouter, Cohere, Mistral, Groq, Gemini, GoAPI, Yahoo. Any compromised key gives AI access at project expense.

**Fix**: Key rotation policy. Monitor for unusual usage patterns. Set spending limits on provider dashboards.
