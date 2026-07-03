# 03 — Business Rules

> All business rules extracted from QuantBit v1 implementation. Every rule has a source location in the v1 codebase.

---

## 1. Investment Profile Rules

**Source:** `src/engine/ranker.ts:11-19`, `src/contexts/EngineConfigContext.tsx:37-41`

```
score = quality * Wq + growth * Wg + value * Wv + momentum * Wm + dividend * Wd
```

- `score` is always between 0 and max(weight × 95)
- Rank 1 = highest score; rank N = lowest score among selected universe
- 3 fixed profiles with predefined weights (see DOMAIN_KNOWLEDGE.md §5)
- Custom profiles use `custom_` prefix + timestamp-based ID
- Profile key fallback: `activeProfileId === "agresif"` → `stockRanksRes`, everything else → `stockRanksProd`
- When `stockNormScores` is available, profile weights are always recomputed (not based on pre-computed ranks)
- Default profile weights (no match): all 0.2 (equal weight)

---

## 2. Buy Pressure Rules

**Source:** `src/engine/buyPressure.ts:50-56,96-142`

```
BPS = valuation*0.30 + momentum*0.25 + breadth*0.15 + drawdown*0.20 + fear*0.10
```

- Valuation: average value score (higher = cheaper). Clamped 0-100.
- Momentum: `clamp(50 - ihsgMonthly * 2, 0, 100)`. Negative monthly return → higher BPS.
- Breadth: `(1 - breadthAbove60 / watchlistCount) * 100`. Fewer healthy stocks → more pressure.
- Drawdown: `clamp(-drawdown60 * 4, 0, 100)`. -25% drawdown → 100 BPS.
- Fear: risk score from regime engine (0-100). Higher risk = more fear = more buy pressure.

Action mapping:
| BPS Range | Action | Deploy % |
|-----------|--------|----------|
| < 30 | none | 0% |
| 30-49 | small | 25% |
| 50-69 | normal | 50% |
| 70-89 | aggressive | 75% |
| >= 90 | deploy | 90% |

**Crisis override:** When `isCrashActive()` → BPS invalid, action=none, deployPct=0, cashPct=100.

---

## 3. Crash Detection Rules

**Source:** `src/engine/crashDetector.ts:6-34`

### Fast Crash
- IHSG 60-day drawdown from peak ≤ -crashSensitivity (default -10%)
- `ihsgPctDrop = ((currentIhsgPrice - maxIhsg60d) / maxIhsg60d) * 100`

### Slow Grind
- Price below SMA50: `currentIhsgPrice < sma50 * (1 - sensitivity*0.5/100)`
- SMA20 below SMA50: `sma20 < sma50 * (1 - sensitivity*0.2/100)`

### Recovery
- **Trend recovery**: IHSG close > SMA20
- **Momentum recovery**: 5d return ≥ 2.5% AND IHSG close > SMA20
- Both must be true to exit crash mode

### Single-Ticker Crash
- 20-day trailing window, `dropFromPeak <= -sellTrigger`
- Independent of market-level crash detection

---

## 4. Rebalancing Rules

**Source:** `src/engine/core.ts:444-492`

- **Emergency exit**: rank ≥ 15 → sell immediately regardless of month
- **Routine exit**: rank ≥ 10 AND month changed → sell at month-end
- **Swap-in**: pick from Top N candidates, must differ from sold ticker, must not already be held
- **Month change**: `currentMonth !== lastRebalanceMonth`. Initialized from day 0.
- **Algo mode only**: no rank-based exits in `custom` or `adaptive_dca` mode
- **Gate**: `config.enableCrossover` must be true for any rebalancing

---

## 5. Dividend Rules

**Source:** `src/engine/core.ts:243-273`

- **Frequency**: Annual (one credit per ticker per year)
- **Timing**: Credited after June 15 (heuristic — not all companies follow this)
- **Net amount**: 90% of gross DPS (`shares * dps * 0.90`)
- **Double-counting prevention**: `lastJulyYear` tracker prevents crediting same year twice
- **Per-ticker breakdown**: `dividendByTicker` accumulator stored in BacktestDayResult

---

## 6. Allocation Rules

**Source:** `src/engine/allocator.ts:24-65`, `src/engine/types.ts:54-58`

- **Equal weight**: `capital / topTickers.length`
- **Lot constraint**: `Math.floor(alloc / (costPerShare * 100)) * 100` → always multiples of 100
- **Volume cap**: Max 5% of daily volume
- **Entry price**: `rawPrice * (1 + slippage)`. Cost/share: `entryPrice * (1 + buyFee)`
- **Exit price**: `rawPrice * (1 - slippage)`. Proceeds: `shares * exitPrice * (1 - sellFee - tax)`
- **Pending tickers**: if no price on day 0, queue them and execute when first valid price appears

---

## 7. Gold Rules

**Source:** `src/engine/allocator.ts:86-101`

- **Buy gold**: `goldBuyPrice = goldPrice * 1.01` (1% premium). All cash → gold grams.
- **Sell gold**: `goldSellPrice = goldPrice * 0.99` (1% discount). `cash = goldGrams * goldSellPrice`.
- Only executed during crash protection with `safeHavenAsset === "emas"`
- Gold tracked in grams, uses 100% of available cash at time of purchase

---

## 8. Market Regime Rules

**Source:** `src/marketRegimeEngine.ts:349-397`

Decision tree (first match wins):

1. **GOLD_DEFENSE**: Crisis active AND IHSG < SMA50 → liquidate stocks, buy gold
2. **CASH_DEFENSE**: Crisis active only → liquidate stocks, hold cash
3. **RECOVERY_WATCH**: IHSG < SMA20 AND IHSG < SMA50 (bearish) OR breadth < 15%
4. **RISK_OFF**: IHSG > SMA20 BUT IHSG < SMA50 AND breadth OK → reduce exposure
5. **RISK_ON**: Everything else → full stock allocation

Risk score formulas:
- MarketHealth: trend(±40/-15/5) + breadth*30 + ihsg*2 (clamped 1-99)
- Opportunity: RISK_ON=60+breadth*30, RECOVERY=40+breadth*20, others=15+breadth*15
- Risk: GOLD=85, RISK_ON=15+(1-breadth)*20, others=40+(1-breadth)*20
- CapitalDeployment: RISK_ON=min(95,40+breadth*40), RECOVERY=25, RISK_OFF=15, other=0

**Breadth**: `above60 / universeLen`. Low breadth = `< 15%` above score 60.

---

## 9. Performance Metrics Rules

**Source:** `src/engine/metrics.ts`

| Metric | Formula | Notes |
|--------|---------|-------|
| CAGR | `(finalValue/startValue)^(1/years) - 1` | years = `daysDiff/365.25` |
| Volatility | `stdDev(dailyReturns) * sqrt(252)` | Assumes 252 trading days |
| Sharpe | `(CAGR - 5%) / annVolatility` | Risk-free rate hardcoded 5% |
| Sortino | `(CAGR - 5%) / downsideVol` | Only negative returns |
| Calmar | `CAGR / (maxDrawdown/100)` | Max drawdown as negative % |
| Max DD | Minimum daily portfolio value / peak | Tracked daily |
| 60/40 Benchmark | `0.6*ihsgReturn + 0.4*goldReturn` | Not traditional 60/40 stocks/bonds |
| Turnover | `totalTransactionVolume / avgPortfolioVal` | avgPortfolioVal = `(start+end)/2` |
| Win Rate | `positiveReturnDays / totalDays` | Days with non-negative return |

---

## 10. AI Interaction Rules

**Source:** `src/ai/systemKnowledge.ts`

- **No auto-execute**: All AI actions must pass through user [Approve] card
- **Response language**: Indonesian/mixed Indonesian-English, natural, no corporate tone
- **Response format**: Overview 1-2 sentences → data table → bullet list → action suggestion
- **Regime > BPS**: "REGIME MENANG. Macro > micro." When regime and BPS conflict, regime wins
- **Tool catalog**: 8 read-only (no approval) + 10 action tools (require [Approve])
- **Proactive agent cooldown**: 5 minutes per rule type
- **Memory limit**: Max 10K characters per session (oldest truncated first)

---

## 11. Config Persistence Rules

**Source:** `src/contexts/EngineConfigContext.tsx`

- localStorage key: `idx_engine_config` (single key for all config + profiles + backtest draft)
- Profile migration: legacy "prod" → "aman", "res" → "agresif" (auto-converted on load)
- Backtest sync: `STRATEGY_KEYS` array defines which fields compose "strategy"
- Strategy equality check: `JSON.stringify(engineConfigKeys) === JSON.stringify(backtestConfigKeys)`
- DCA toggle: `dcaActive` key, defaults to true
- Backtest mode toggle: localStorage `quantbit_backtest_use_live_strategy` ("1"/"0")

---

## 12. Notification Rules

**Source:** `src/engine/notificationRules.ts`

| Rule | Trigger | Source |
|------|---------|--------|
| Ticker out of Top N | `currentRank > topN` | Rebalancing engine |
| Crash protection | IHSG 60d drawdown ≤ -crashSensitivity | Crash detector |
| Custom universe breach | Ticker not in customUniverse list | Custom mode only |
| Single mode trigger | (stub — always returns false) | Legacy |

---

## 13. Data Freshness Rules

**Source:** `src/types/DataStatus.ts`

| Status | Age | Source |
|--------|-----|--------|
| LIVE | < 5 minutes | Real-time feed (GoAPI) |
| CACHED | < 24 hours | D1 daily snapshot |
| STALE | > 24 hours | Last known value |
| ESTIMATED | N/A | Synthetic/interpolated data |

Every data point displayed in the UI must carry a DataStatus badge showing its freshness.
