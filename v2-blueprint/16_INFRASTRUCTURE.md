# 16 — Infrastructure

> Hosting, deployment, environment, and CI/CD for QuantBit V2.

---

## Hosting

| Service | Purpose | Provider | Cost |
|---------|---------|----------|------|
| Frontend | React SPA | Cloudflare Pages | Free |
| Backend API | Hono edge functions | Cloudflare Pages Functions | Free (included with Pages) |
| Database | D1 SQLite | Cloudflare D1 | Free tier (5GB) |
| Cache | Workers KV | Cloudflare KV | Free tier (1GB) |
| AI Proxy | OpenRouter API | OpenRouter | Free + pay-as-you-go |
| Email | Notifications | Resend | Free tier (100 emails/day) |
| CI/CD | Build + test + deploy | GitHub Actions | Free |
| Monitoring | Error tracking | Cloudflare Dashboard | Free |
| Cron | Daily data pipeline | GitHub Actions | Free |

**Total hosting cost:** $0/month (within free tier limits)

---

## Environment Configuration

```env
# .env.local (dev) / Cloudflare Pages secrets (prod)

# Database
D1_BINDING=DB (automatic via wrangler.toml)

# API Keys
OPENROUTER_API_KEY=sk-or-v1-...
GOAPI_API_KEY=... (optional, real-time data)

# Auth
AUTH_SECRET=... (Better Auth secret)

# Cron / Admin
CRON_SECRET=... (GitHub Actions → API authentication)
ADMIN_KEY=... (Admin API access)

# Email (optional)
RESEND_API_KEY=re_...

# Feature Flags
ENABLE_PROACTIVE_AGENT=false
ENABLE_MULTIPLE_PROVIDERS=false
```

**Total:** 6-8 env vars (down from 10+ in V1)

---

## wrangler.toml

```toml
name = "quantbit-v2"
compatibility_date = "2026-07-03"
pages_build_output_dir = "dist"

[[d1_databases]]
binding = "DB"
database_name = "quantbit-v2-db"
database_id = "..."

[[kv_namespaces]]
binding = "CACHE"
id = "..."

[env.production.variables]
ENABLE_PROACTIVE_AGENT = "false"
ENABLE_MULTIPLE_PROVIDERS = "false"

[env.preview.variables]
ENABLE_PROACTIVE_AGENT = "false"
ENABLE_MULTIPLE_PROVIDERS = "false"
```

---

## CI/CD Pipeline (GitHub Actions)

```yaml
name: V2 CI/CD
on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main]

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
      - run: npm ci
      - run: npm run lint
      - run: npm run typecheck
      - run: npm run test

  deploy:
    needs: test
    if: github.ref == 'refs/heads/main'
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
      - run: npm ci
      - run: npm run build
      - run: npx wrangler pages deploy dist --project-name=quantbit-v2
```

**V2 improvements over V1:**
- Tests run before deploy (V1: no tests)
- Type checking (V1: no typecheck script)
- Separate preview/production branches (V1: single branch)

---

## Data Pipeline (GitHub Actions Cron)

```yaml
name: Daily Data Pipeline
on:
  schedule:
    - cron: "0 14 * * 1-5"  # 14:00 UTC = 21:00 WIB (after market close)
  workflow_dispatch:  # Manual trigger

jobs:
  sync:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - run: npm ci
      - run: npm run pipeline:sync  # Unified pipeline script
        env:
          D1_API_TOKEN: ${{ secrets.D1_API_TOKEN }}
          CRON_SECRET: ${{ secrets.CRON_SECRET }}
      - run: curl -X POST https://quantbit-v2.pages.dev/api/v1/admin/force-sync
        env:
          CRON_SECRET: ${{ secrets.CRON_SECRET }}
```

**V2 improvements over V1:**
- One unified script (`pipeline:sync`) instead of 4 separate scripts (Python + TS)
- No intermediate JSON files
- Direct D1 writes via binding (not REST API)
- Cron secret in `Authorization` header, not POST body

---

## Database Migrations

```bash
# Migration workflow
npx wrangler d1 migrations create quantbit-v2-db init
npx wrangler d1 migrations create quantbit-v2-db add_ticker_catalog

# Apply to local
npx wrangler d1 migrations apply quantbit-v2-db --local

# Apply to production
npx wrangler d1 migrations apply quantbit-v2-db --remote

# Seed data (for new databases)
npm run db:seed  # Imports historical data from backup source
```

**V2 improvement:** Use D1 migrations consistently (V1 had migrations but also had seed scripts that bypassed them).

---

## Deployment Environments

| Environment | URL | Database | Purpose |
|-------------|-----|----------|---------|
| Local dev | `http://localhost:5173` | Local D1 (wrangler dev) | Development |
| Preview | `https://preview.quantbit-v2.pages.dev` | Preview D1 | PR testing |
| Production | `https://quantbit-v2.pages.dev` | Production D1 | Live |

**Deployment flow:**

```
Developer pushes to develop
  → CI runs tests + typecheck
  → Deploys to Preview
  → Developer verifies on Preview
  → PR merged to main
  → CI runs tests + typecheck
  → Deploys to Production
  → Daily cron runs data pipeline
```

---

## Monitoring (V2)

| What | How | Alert |
|------|-----|-------|
| API errors > 1% | Cloudflare Dashboard analytics | Email |
| Data pipeline failure | GitHub Actions status badge | Email |
| D1 query latency > 500ms | Cloudflare Dashboard | None (log only) |
| Rate limit hits | Custom logging | None (info only) |
| Database size > 80% | Cloudflare Dashboard | Email |

**V1 had zero monitoring** — errors only discovered when users reported them.

---

## Backup Strategy

- D1: No automated backup (Cloudflare limitation)
- Workaround: Weekly export via `wrangler d1 execute --command="SELECT * FROM ..."`
- Critical data (user accounts, portfolios) exported daily to JSON via cron
- Historical market data: Re-generated from Yahoo Finance if needed
