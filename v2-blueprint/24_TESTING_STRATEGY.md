# 24 — Testing Strategy

> Testing strategy for QuantBit V2.

---

## Test Pyramid

```
          ╱╲
         ╱  ╲          E2E: 2-3 critical user journeys
        ╱    ╲
       ╱──────╲        Integration: API endpoints + data pipeline
      ╱        ╲
     ╱──────────╲      Unit: Engine logic, utils, validation
    ╱            ╲
   ╱──────────────╲
  ╱                ╲
 ╱──────────────────╲  Golden file: Backtest results (pre-computed)
```

---

## Unit Tests (80% of all tests)

### Engine (Priority: Critical)

**What to test:** Every pure function in `src/lib/engine/`

```
src/lib/engine/
├── core.test.ts          # runStrategy() with known inputs
├── ranker.test.ts        # computeDayRankings(), pickTopTickersByRank()
├── allocator.test.ts     # computeInitialAllocation(), computeSellProceeds()
├── crashDetector.test.ts # detectCrashAlgo(), detectRecoveryAlgo()
├── buyPressure.test.ts   # computeBuyPressure(), withCrisisOverride()
├── metrics.test.ts       # CAGR, Sharpe, Sortino, Calmar, MaxDD
├── marketRegime.test.ts  # computeMarketRegime(), risk formulas
├── dividendCache.test.ts # getDividendPerShare()
└── notificationRules.test.ts
```

**Test approach:**
- Pure functions with deterministic inputs → deterministic outputs
- No mocks needed (pure TS, no I/O)
- Edge cases: empty arrays, negative prices, missing scores, zero volume

**Example:**

```typescript
describe("computeDayRankings", () => {
  it("should rank highest score first", () => {
    const scores = [
      { ticker: "BBCA", quality: 90, growth: 80, value: 70, momentum: 60, dividend: 50 },
      { ticker: "BBRI", quality: 80, growth: 70, value: 60, momentum: 50, dividend: 40 },
    ];
    const weights = { quality: 0.25, growth: 0.25, value: 0.25, momentum: 0.25, dividend: 0 };
    
    const result = computeDayRankings(scores, weights);
    
    expect(result[0]!.ticker).toBe("BBCA"); // score: (90+80+70+60)/4 = 75
    expect(result[1]!.ticker).toBe("BBRI"); // score: (80+70+60+50)/4 = 65
  });
  
  it("should return empty array for empty input", () => {
    expect(computeDayRankings([], mockWeights)).toEqual([]);
  });
  
  it("should handle missing dividend score defaulting to 50", () => {
    const scores = [{ ticker: "BBCA", quality: 90, growth: 80, value: 70, momentum: 60 }];
    const result = computeDayRankings(scores, { ...mockWeights, dividend: 0.1 });
    // Should not crash. Missing dividend defaults to 50.
    expect(result).toHaveLength(1);
  });
});
```

**Target: 100% line coverage for engine modules.**

---

### API Client (Priority: High)

**What to test:** Request building, error handling, response parsing

**Test approach:**
- Mock `fetch` with known responses
- Test each API client function returns correctly typed data
- Test error handling (network errors, malformed responses, 4xx/5xx)

---

### Utils (Priority: Medium)

**What to test:** `src/lib/utils/*`

- `formatCurrency()` — IDR formatting
- `formatPercentage()` — Percent formatting with sign
- `dataStatus()` — Freshness determination logic

---

## Integration Tests (15% of all tests)

### API Endpoints

**What to test:** Each endpoint with a test D1 database

```
functions/api/v1/auth/register.test.ts
functions/api/v1/market/overview.test.ts
functions/api/v1/portfolio/trade.test.ts
functions/api/v1/engine/run-backtest.test.ts
functions/api/v1/ai/chat.test.ts
```

**Test approach:**
- Use `wrangler d1` local test database
- Seed with known test data
- Send HTTP requests with Hono's `app.request()`
- Assert response status, body, and D1 state changes

### Data Pipeline

**What to test:** `scripts/sync-market-data.ts`

- With mock Yahoo Finance responses
- Verify correct D1 writes (upsert, not duplicate)
- Error handling: partial data, missing tickers, rate limits

---

## Golden File Tests (3% of all tests)

**What:** Pre-computed backtest results for known inputs

**Why:** Backtest is the most complex function. Changes to any engine module can affect results. Golden file tests catch regressions.

**How:**
```
tests/fixtures/
├── backtest-input-2023.json       # Known market data
├── backtest-expected-aman.json    # Expected results for AMAN profile
└── backtest-expected-agresif.json
```

```typescript
describe("runStrategy golden files", () => {
  it("AMAN profile should produce expected CAGR for 2023", () => {
    const input = loadFixture("backtest-input-2023.json");
    const expected = loadFixture("backtest-expected-aman.json");
    
    const result = runStrategy(input, AMAN_CONFIG);
    
    expect(result.cagr).toBeCloseTo(expected.cagr, 2);
    expect(result.sharpe).toBeCloseTo(expected.sharpe, 2);
    expect(result.maxDrawdown).toBeCloseTo(expected.maxDrawdown, 2);
    expect(result.finalValue).toBeCloseTo(expected.finalValue, -3); // Nearest 1000
  });
});
```

**Generate golden files:**
```bash
npm run test:generate-golden # Run current engine → save expected outputs
npm run test                 # Run and compare with golden files
```

---

## E2E Tests (2% of all tests)

**What:** 2-3 critical user journeys

1. **Backtest → Sync to Portfolio** — Configure strategy, run backtest, sync, verify portfolio
2. **Trade → Portfolio update** — Buy stock, verify holdings, sell, verify cash
3. **AI Chat → Approve action** — Ask AI, approve proposal, verify execution

**Tool:** Playwright (same as v1 screenshot captures)

---

## Test Configuration

```json
// vitest.config.ts
{
  "test": {
    "include": ["tests/**/*.test.ts"],
    "coverage": {
      "include": ["src/lib/engine/**", "src/lib/api/**", "src/lib/utils/**"],
      "thresholds": {
        "lines": 80,
        "functions": 80,
        "branches": 70
      }
    }
  }
}
```

---

## CI Integration

```yaml
# .github/workflows/test.yml
name: Tests
on: [push, pull_request]

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - run: npm ci
      - run: npm run lint
      - run: npm run typecheck
      - run: npm run test:unit
      - run: npm run test:integration
      - run: npm run test:golden
      - run: npm run test:e2e  # If Playwright installed
```

---

## What NOT to Test

- **UI rendering details** (too brittle, little value)
- **CSS styles** (Tailwind utility classes are tested by Tailwind team)
- **Third-party integrations** (API providers, AI providers — test your code, not theirs)
- **Database migrations** (D1 migration tooling tested by Cloudflare)
- **Auth flows** (Better Auth tested by its own team — test your integration only)
