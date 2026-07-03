# 27 — V2 Bootstrap Guide

> Step-by-step guide to building QuantBit V2 from an empty repository.

---

## Prerequisites

```bash
# Required tools
node >= 22
npm >= 10
git
Cloudflare account (free)
GitHub account (free)

# Optional
wrangler CLI (npm install -g wrangler)
```

---

## Step 1: Initialize Repository

```bash
mkdir quantbit-v2 && cd quantbit-v2
git init
git checkout -b main
```

## Step 2: Create Frontend Scaffold

```bash
npm create vite@latest . -- --template react-ts
npm install
```

## Step 3: Install Dependencies

```bash
# Frontend
npm install motion react-router-dom recharts lucide-react sonner
npm install -D tailwindcss @tailwindcss/vite

# Shared/Backend
npm install hono @hono/zod-validator zod
npm install better-auth

# Dev
npm install -D vitest @testing-library/react @playwright/test
```

## Step 4: Configure Tooling

```bash
# Create config files:
# tsconfig.json — strict mode, path aliases (~/src)
# vite.config.ts — Tailwind plugin, path aliases
# wrangler.toml — D1 + KV bindings
# vitest.config.ts — coverage thresholds
# .env.example — env var template
```

## Step 5: Create Folder Structure

```bash
mkdir -p src/{components/{ui,market,portfolio,backtest,analytics,admin,ai},hooks,contexts,lib/{api,engine,utils},types,constants}
mkdir -p functions/api/v1/{auth,market,portfolio,engine,ai,admin,middleware}
mkdir -p scripts
mkdir -p db/migrations
mkdir -p tests/{unit,integration,fixtures}
```

## Step 6: Set Up D1 Database

```bash
npx wrangler d1 create quantbit-v2-db
npx wrangler d1 migrations create quantbit-v2-db init
# Edit db/migrations/0000_init.sql — copy schema from 18_DATABASE_DESIGN.md
npx wrangler d1 migrations apply quantbit-v2-db --local

# Seed ticker list
npm run db:seed
```

## Step 7: Write Engine

```bash
# Copy engine modules into src/lib/engine/
# Each file corresponds to a v1 engine module:
# - core.ts → runStrategy()
# - ranker.ts → computeDayRankings()
# - allocator.ts → position sizing
# - crashDetector.ts → crash detection
# - buyPressure.ts → BPS calculation
# - metrics.ts → performance metrics
# - marketRegime.ts → regime classification
# - dividendCache.ts → DPS lookup

# Write tests for each module
npm run test:unit  # Green!
```

## Step 8: Build API

```bash
# Set up Hono in functions/api/v1/index.ts
# Add middleware (CORS, auth, rate-limit, error handler)
# Implement each domain module

npm run dev  # Test endpoints with curl
```

## Step 9: Build Frontend

```bash
# Start from App.tsx → AppShell → Tabs
# Implement each tab component
# Wire up API calls via custom hooks

npm run dev  # UI appears in browser
```

## Step 10: Add AI

```bash
# Set up OpenRouter client
# Build modular system prompt
# Implement structured output parsing
# Build FloatingAIChat component
# Test: "How is my portfolio?" → AI responds
```

## Step 11: Set Up Pipeline

```bash
# Write scripts/sync-market-data.ts
# Write scripts/compute-scores.ts
# Test: npm run pipeline:sync
# Set up GitHub Actions cron
```

## Step 12: Deploy

```bash
npm run build
npx wrangler pages deploy dist
npx wrangler d1 migrations apply quantbit-v2-db --remote
```

---

## Day-by-Day Quick Start

```bash
# Day 1: Foundation
npm create vite@latest quantbit-v2 -- --template react-ts
cd quantbit-v2
npm install hono zod @hono/zod-validator better-auth
npm install motion recharts lucide-react sonner
npm install -D tailwindcss @tailwindcss/vite vitest
# Configure files
git add -A && git commit -m "feat: project scaffold"

# Day 2: Database + Engine
npx wrangler d1 create quantbit-v2-db
# Create migration + schema
# Write engine modules
git add -A && git commit -m "feat: database schema + engine"

# Day 3: API + Basic UI
# Set up Hono routes
# Create AppShell + tabs
git add -A && git commit -m "feat: API + UI scaffold"

# Day 4-5: Complete UI
# Implement all tabs
# Wire API calls
# Keyboard shortcuts
git add -A && git commit -m "feat: complete UI"

# Day 6: Backtest UI
# Connect to engine via API
# Chart + results display
git add -A && git commit -m "feat: backtest UI"

# Day 7: AI + Deploy
# AI chat integration
# Production deploy
git add -A && git commit -m "feat: AI chat + production deploy"
```

**Note:** This 7-day quick start is aspirational. The 33-day plan in 26_REBUILD_PHASES.md is more realistic for a complete, tested, production-ready system.
