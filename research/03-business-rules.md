# 03 — Business Rules

Extracted from implementation. File paths link to the exact code.

---

## 1. Investment Profile Rules

**Source**: `src/engine/ranker.ts:11-19`

```
score = quality * Wq + growth * Wg + value * Wv + momentum * Wm + dividend * Wd
```

- **3 fixed profiles** (`src/contexts/EngineConfigContext.tsx:37-41`):
  - `aman`: Q=0.30, G=0.45, V=0.10, M=0.00, D=0.15
  - `agresif`: Q=0.20, G=0.60, V=0.10, M=0.10, D=0.00
  - `dividen`: Q=0.15, G=0.20, V=0.05, M=0.00, D=0.60
- **Custom profiles** use `"custom_" + Date.now().toString(36)` as ID (`src/contexts/EngineConfigContext.tsx:303`)
- **Profile key fallback** (`src/engine/core.ts:36-37`): `activeProfileId === "agresif"` → `stockRanksRes`, everything else → `stockRanksProd`. For data with `stockNormScores`, profile is always recomputed.
- **Default profile weights** for backtest when no profile matches: all 0.2 (equal, `src/engine/types.ts` does not define — hardcoded in historical data generation)

**Rule**: score is always between 0 and max(weight*95). Rank 1 = highest score. Dividend score defaults to 50 when missing, else 0-95 via percentile rank of yield.

---

## 2. Buy Pressure Rules

**Source**: `src/engine/buyPressure.ts:50-56,96-142`

```
BPS = valuation*0.30 + momentum*0.25 + breadth*0.15 + drawdown*0.20 + fear*0.10
```

- **valuation** (30%): avg value score (higher = cheaper = more buy pressure). Clamped 0-100.
- **momentum** (25%): `clamp(50 - ihsgMonthly * 2, 0, 100)`. Negative monthly return → higher score.
- **breadth** (15%): `(1 - breadthAbove60 / watchlistCount) * 100`. Fewer healthy stocks → more pressure.
- **drawdown** (20%): `clamp(-drawdown60 * 4, 0, 100)`. -25% drawdown → 100.
- **fear** (10%): risk score from regime engine (0-100). Higher risk = more fear = more buy.

**Action mapping** (`src/engine/buyPressure.ts:62-68`):
| Score Range | Action | Deploy % |
|-------------|--------|----------|
| < 30 | none | 0% |
| 30-49 | small | 25% |
| 50-69 | normal | 50% |
| 70-89 | aggressive | 75% |
| >= 90 | deploy | 90% |

**Crisis override** (`src/engine/buyPressure.ts:148-160`): When `isCrashActive()` → valid=false, action=none, deployPct=0, cashPct=100.

---

## 3. Crash Detection Rules

**Source**: `src/engine/crashDetector.ts:6-34`

**Fast crash**: `ihsgPctDrop <= -crashSensitivity` (default 10%). IHSG 60d drawdown from peak.
- `ihsgPctDrop = ((currentIhsgPrice - maxIhsg60d) / maxIhsg60d) * 100`

**Slow grind**: `currentIhsgPrice < sma50 * (1 - sensitivity*0.5/100) AND sma20 < sma50 * (1 - sensitivity*0.2/100)`
- Price below deflated SMA50 + SMA20 below deflated SMA50 = bearish confirmation

**Recovery** (`src/engine/crashDetector.ts:53-71`):
- **Trend recovery**: IHSG close > SMA20
- **Momentum recovery**: 5d return >= 2.5% AND IHSG close > SMA20

**Single-ticker crash** (`src/engine/crashDetector.ts:36-51`): 20-day trailing window, `dropFromPeak <= -sellTrigger`.

---

## 4. Rebalancing Rules

**Source**: `src/engine/core.ts:444-492`

- **Emergency exit**: rank >= 15 → sell immediately regardless of month
- **Routine exit**: rank >= 10 AND month changed → sell at month-end
- **Swap-in ticker**: pick from Top N candidates, must differ from sold ticker, must not already be held
- **Month change detection**: `currentMonth !== lastRebalanceMonth`. Initialized from day0 month.
- **Algo mode only**: no rank-based exits in `custom` or `adaptive_dca` mode
- **Constraint**: `config.enableCrossover` must be true

---

## 5. Dividend Rules

**Source**: `src/engine/core.ts:243-273`

- **Frequency**: Annual, credited after June 15th (`currentMonth >= 5 && date >= 15`)
- **Net amount**: 90% of gross DPS (`shares * dps * 0.90`)
- **Last July year tracker**: prevents double-counting (`lastJulyYear`). Each year credits once.
- **Per-ticker tracking**: `dividendByTicker` accumulator for UI breakdown (`src/engine/types.ts:92-93`)

---

## 6. Allocation Rules

**Source**: `src/engine/allocator.ts:24-65`

- **Per-stock allocation**: `capital / topTickers.length` (equal weight)
- **Lot-based**: `Math.floor(alloc / (costPerShare * 100)) * 100` → multiples of 100 shares
- **Volume cap**: Max 5% of daily volume: `Math.floor((dailyVol * 0.05) / 100) * 100`
- **Fees** (`src/engine/types.ts:54-58`):
  - Buy fee: 0.15%
  - Sell fee: 0.25%
  - Tax: 0.10%
  - Slippage: 0.25%
- **Entry price**: `rawPrice * (1 + slippage)`. Cost per share: `entryPrice * (1 + buyFee)`.
- **Exit price**: `rawPrice * (1 - slippage)`. Proceeds: `shares * exitPrice * (1 - sellFee - tax)`.
- **Pending tickers**: tickers with no price on day 0 get queued; executed when first valid price appears

---

## 7. Gold Rules

**Source**: `src/engine/allocator.ts:86-101`

- **Buy**: `goldBuyPrice = goldPrice * 1.01` (1% premium). All cash → gold grams.
- **Sell**: `goldSellPrice = goldPrice * 0.99` (1% discount). `cash = goldGrams * goldSellPrice`.
- Used during crash protection: `safeHavenAsset === "emas"` → buy gold after liquidation

---

## 8. Regime Rules

**Source**: `src/marketRegimeEngine.ts:349-377`

**Decision tree** (first match wins):
1. Crisis + bearish trend → **GOLD_DEFENSE** → HOLD_GOLD
2. Crisis only → **CASH_DEFENSE** → HOLD_CASH
3. Bearish trend (below MA20 + MA50) → **RECOVERY_WATCH** → WAIT_RECOVERY
4. Recovering (above MA20, below MA50) + low breadth (< 15% stocks >= 60) → **RECOVERY_WATCH**
5. Recovering but breadth ok → **RISK_OFF**
6. Low breadth → **RISK_OFF**
7. Everything else → **RISK_ON** → BUY_STOCKS

**Risk score formulas** (`src/marketRegimeEngine.ts:379-397`):
- MarketHealth: trend(±40/-15/5) + breadth*30 + ihsg*2 (clamped 1-99)
- Opportunity: RISK_ON=60+breadth*30, RECOVERY=40+breadth*20, others=15+breadth*15
- Risk: GOLD=85, RISK_ON=15+(1-breadth)*20, others=40+(1-breadth)*20
- CapitalDeployment: RISK_ON=min(95,40+breadth*40), RECOVERY=25, RISK_OFF=15, other=0

**Breadth**: `above60 / universeLen`. Low breadth = `above60 < lenL * 0.15`.

---

## 9. Metrics Rules

**Source**: `src/engine/metrics.ts`

- **Risk-free rate**: 5.0% (hardcoded in `src/engine/metrics.ts:70`)
- **CAGR**: `(final/cap)^(1/years) - 1`. Years = `daysDiff / 365.25`.
- **Volatility**: `stdDev(dailyReturns) * sqrt(252) / 100`
- **Sharpe**: `(CAGR - RF) / annVolatility`. If vol = 0, Sharpe = 0.
- **Sortino**: `(CAGR - RF) / downsideVol`. Downside vol uses negative returns only.
- **Calmar**: `CAGR / (maxDrawdown / 100)`. If maxDD = 0, Calmar = 0.
- **60/40 benchmark**: `0.6 * ihsgReturn + 0.4 * goldReturn`
- **Turnover**: `totalTransactionVolume / avgPortfolioVal`
- **Win rate**: `positiveReturnDays / totalDays`

---

## 10. AI Rules

**Source**: `src/ai/systemKnowledge.ts`

- **Tool catalog** (Section 13): 8 read-only + 10 action tools
- **Zero auto-execute**: All AIAction must pass through `AIActionApprovalCard` with user [Approve] before dispatch
- **Jaksel persona**: Santai, pake data, campur Indo-English, natural, jangan kasar
- **Structured response format**: Overview 1-2 kalimat → tabel data → bullet list → action suggestion. Markdown tables + bold for numbers
- **Response must be in Indonesian/mixed**: No corporate tone, no emoji, no TL;DR headers
- **BPS vs Regime priority**: "REGIME MENANG. Macro > micro." (`systemKnowledge.ts:165`)
- **Proactive rules** (Section 14): 6 BPS transition rules — system notifies, doesn't act. Cooldown 5min per rule.
- **Memory**: Past sessions injected as Section 15. Max 10K chars, truncates oldest first.

---

## 11. Config Persistence Rules

**Source**: `src/contexts/EngineConfigContext.tsx`

- **Key**: `localStorage.getItem("idx_engine_config")` — single key for engine config + profiles + backtest draft
- **Profile migration**: Legacy "prod"→"aman", "res"→"agresif". Old IDs stripped on load.
- **Backtest sync**: `STRATEGY_KEYS` array defines which fields compose "strategy" — compared via JSON.stringify between engineConfig and backtestConfig
- **DCA toggle**: `dcaActive` key (boolean, defaults true). Not persisted in legacy.
- **Backtest useLiveStrategy toggle**: localStorage `quantbit_backtest_use_live_strategy` ("1"/"0")

---

## 12. Data Status Rules

**Source**: `src/types/DataStatus.ts`

- **LIVE**: Real-time data (from GoAPI polling or WebSocket)
- **CACHED**: Data from last sync (within acceptable age)
- **STALE**: Data older than acceptable threshold
- **ESTIMATED**: Computed/interpolated data (no direct source)

---

## 13. Notification Rules

**Source**: `src/engine/notificationRules.ts`

- `rule_tickerOutOfTopN` (`:20-33`): `currentRank > topN` → triggered
- `rule_crashProtectionTriggered` (`:35-52`): IHSG 60d drawdown <= -crashSensitivity → triggered
- `rule_customUniverseBreach` (`:54-70`): ticker not in customUniverse list → triggered (custom mode only)
- `rule_singleModeTrigger` (`:72-77`): stub — always returns false currently
