# 23 — Coding Standards

> Recommended coding standards for QuantBit V2.

---

## Language

**TypeScript 5.x with strict mode.** No JavaScript files.

```json
// tsconfig.json
{
  "compilerOptions": {
    "strict": true,
    "noUncheckedIndexedAccess": true,
    "noImplicitReturns": true,
    "noFallthroughCasesInSwitch": true,
    "exactOptionalPropertyTypes": true,
    "forceConsistentCasingInFileNames": true
  }
}
```

---

## Naming Conventions

| Concept | Convention | Example |
|---------|-----------|---------|
| Files/Directories | `kebab-case` | `buy-pressure-dashboard.tsx` |
| TypeScript types | `PascalCase` | `interface BacktestConfig` |
| Functions | `camelCase` | `computeDayRankings()` |
| Variables | `camelCase` | `const topTickers: Ticker[]` |
| Constants | `UPPER_SNAKE_CASE` | `const DEFAULT_FEES = {...}` |
| Database columns | `snake_case` | `buy_price`, `last_updated` |
| API properties | `camelCase` | `buyPrice`, `lastUpdated` |
| React components | `PascalCase` | `export function StockDrawer()` |
| React hooks | `use` prefix | `usePortfolio()` |
| Event handlers | `handle` prefix | `handleBuyStock()` |
| CSS classes | Tailwind utility classes | (no custom CSS) |

---

## Code Organization

### File Size Limit

- **Components:** Max 300 lines
- **Hooks:** Max 150 lines
- **API handlers:** Max 200 lines
- **Engine modules:** Max 400 lines

### Import Order

```typescript
// 1. External libraries
import React, { useCallback } from "react";
import { motion } from "motion/react";

// 2. Internal modules (alphabetical by path)
import { useMarket } from "~/hooks/useMarket";
import { Button } from "~/components/ui/Button";

// 3. Types
import type { MarketData } from "~/types/market";

// 4. Constants
import { DEFAULT_FEES } from "~/constants/fees";

// 5. Relative imports (last)
import { formatCurrency } from "./utils";
```

### Path Aliases

```typescript
// vite.config.ts
{
  resolve: {
    alias: {
      "~": path.resolve(__dirname, "src"),
    },
  },
}
```

---

## TypeScript Patterns

### Prefer `interface` over `type` for objects
```typescript
// Good
interface BacktestConfig {
  profile: string;
  topN: number;
  crashSensitivity: number;
}

// Avoid for objects
type BacktestConfig = {
  profile: string;
  topN: number;
};
```

### Use `type` for unions and utility types
```typescript
type MarketRegime = "GOLD_DEFENSE" | "CASH_DEFENSE" | "RECOVERY_WATCH" | "RISK_OFF" | "RISK_ON";
type DataStatus = "LIVE" | "CACHED" | "STALE" | "ESTIMATED";
```

### Branded types for domain primitives
```typescript
type Ticker = string & { __brand: "Ticker" };
type UserId = string & { __brand: "UserId" };

function validateTicker(s: string): Ticker {
  if (!/^[A-Z]{4}$/.test(s)) throw new Error(`Invalid ticker: ${s}`);
  return s as Ticker;
}
```

### No `any`. Prefer `unknown` with type guards.
```typescript
// Bad
function parseJSON(s: string): any { ... }

// Good
function parseJSON<T>(s: string): T {
  return JSON.parse(s) as T;
}
```

---

## Error Handling

### No try/catch at component level unless presenting to user
```typescript
// Good — hook handles errors internally
function usePortfolio() {
  const [error, setError] = useState<ApiError | null>(null);
  
  async function fetchPortfolio() {
    try {
      const result = await api.get("/portfolio");
      setData(result.data);
    } catch (err) {
      setError(err instanceof ApiError ? err : new ApiError("UNKNOWN"));
    }
  }
}
```

### Use Result pattern for engine functions
```typescript
type Result<T, E = Error> = { ok: true; value: T } | { ok: false; error: E };

function computeDayRankings(scores: Score[]): Result<Ranking[], ScoringError> {
  if (scores.length === 0) return { ok: false, error: { code: "EMPTY_INPUT" } };
  // ... computation
  return { ok: true, value: rankings };
}
```

---

## Testing Standards

- **Engine functions:** 100% coverage. Pure TS math must be verified.
- **API endpoints:** Integration tests with mocked D1.
- **Components:** Smoke tests (render + click).
- **No snapshot tests** (too brittle for UI).
- **Golden file tests** for backtest results: run backtest → compare with known-good output.

---

## Comments

- **No comments that explain what the code does.** Code should be self-documenting.
- **Comments only for** why (business logic decisions), not what or how.
- **JSDoc** for public API functions and exported types.

```typescript
// Why: IDX has ~247 trading days on average, not 252 (US standard).
// Using 252 overstates annualized volatility by ~2%.
const IDX_TRADING_DAYS = 247;
```

---

## Git Workflow

```
main        → Production-ready code
develop     → Integration branch
feature/*   → Individual features (branched from develop)
fix/*       → Bug fixes
```

**Commit convention:**
- `feat:` — New feature
- `fix:` — Bug fix
- `docs:` — Documentation
- `refactor:` — Code restructuring
- `test:` — Tests
- `chore:` — Build/config changes

---

## Accessibility (Minimum)

- All interactive elements keyboard-accessible
- Color not the only indicator (DataStatus badges use both color and text)
- Focus indicators visible (terminal green outline)
- Alt text on charts

---

## Performance

- Lazy-load tab components (not all tabs mount at once)
- Debounce search inputs (300ms)
- Memoize expensive computations (`useMemo`, `useCallback`)
- Virtualized stock list if > 100 items
- Bundle size awareness: import from `lucide-react` specific icons, not barrel
