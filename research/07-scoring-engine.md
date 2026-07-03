# Scoring Engine — Complete Documentation

**Source:** `src/engine/ranker.ts`, `src/engine/buyPressure.ts`, `src/engine/crashDetector.ts`, `src/engine/metrics.ts`, `src/marketRegimeEngine.ts`  
**Date:** 2026-07-03

---

## 1. Factor Scoring System (ranker.ts)

### 1.1 Composite Score Formula

```
score = quality * Wq + growth * Wg + value * Wv + momentum * Wm + dividend * Wd
```

Each factor is a normalized value 0–95 (or 0–100 for dividend). Weights come from user profiles.

### 1.2 Weight Profiles

| Profile   | Quality | Growth | Value | Momentum | Dividend |
|-----------|---------|--------|-------|----------|----------|
| AMAN      | 0.30    | 0.45   | 0.10  | 0.00     | 0.15     |
| AGRESIF   | 0.20    | 0.60   | 0.10  | 0.10     | 0.00     |
| DIVIDEN   | 0.15    | 0.20   | 0.05  | 0.00     | 0.60     |

Default backtest weights (from API, lines 457-459):
| Config | Quality | Growth | Value | Momentum |
|--------|---------|--------|-------|----------|
| prod   | 0.45    | 0.10   | 0.05  | 0.40     |
| res    | 0.40    | 0.25   | 0.05  | 0.30     |

Note: API weights have no `dividend` component, while engine profiles always include it (even if 0).

### 1.3 Ranking Algorithm (ranker.ts:3-29)

```python
for each ticker:
    score = (quality × Wq) + (growth × Wg) + (value × Wv) + (momentum × Wm) + (dividend × Wd)

scores.sort(descending)
for i, ticker in enumerate(sorted):
    stockRanks[ticker] = i + 1  # 1-based rank
```

Missing factors default to 50. `dividend` defaults to 0 (not 50) in the ranker — a potential bug if dividend data is absent.

### 1.4 Top N Selection (ranker.ts:31-48)

```python
eligible = [ticker for ticker in ranked if ticker in allowedTickers AND dayPrices[ticker] > 0]
eligible.sort(by rank)
return eligible[:topNCount]
```

Filters out tickers not in the selected universe (IDX80/IDX30/LQ45) and tickers with zero price.

### 1.5 Adaptive Weights (ranker.ts:56-147)

Triggered every month when `enableAdaptiveWeights = true` and `simulationMode !== "adaptive_dca"`. Requires `stepIndex >= 60` (60-day minimum lookback).

**Step 1:** Compute factor returns over 60-day window:
```python
for each factor in [quality, growth, value, momentum]:
    sort tickers by factor score (descending) at day0
    pick top N (topNCount)
    factor_return = average((pN - p0) / p0 for each ticker)
```

**Step 2:** Normalize returns to [0, 1]:
```python
min_ret = min(factor_returns)
max_ret = max(factor_returns)
range = max_ret - min_ret

if range < 0.001:
    weights[factor] = 0.25 * (1 - dividendFixed)  # equal distribution
else:
    normalized = (factor_return - min_ret) / range
    adjusted = minWeight + normalized * (maxWeight - minWeight)
    where minWeight = 0.10 * (1 - dividendFixed)
          maxWeight = 0.50 * (1 - dividendFixed)
```

**Step 3:** Normalize weights to sum to `(1 - dividendFixed)`:
```python
total = sum(weights.values())
for each factor:
    weights[factor] = (weights[factor] / total) * (1 - dividendFixed)
```
`dividendFixed` is always kept unchanged. Only Q, G, V, M are adaptive.

---

## 2. Buy Pressure Score — BPS (buyPressure.ts)

### 2.1 Formula

```
valuation  = clamp(avgValueScore, 0, 100)                     × 0.30
momentum   = clamp(50 - ihsgMonthly × 2, 0, 100)              × 0.25
breadth    = clamp((1 - breadthAbove60/watchlistCount) × 100)  × 0.15
drawdown   = drawdown60 < 0 ? clamp(-drawdown60 × 4, 0, 100)   × 0.20
fear       = clamp(riskScore, 0, 100)                          × 0.10
```

Where:
- `avgValueScore` — average value score across all tracked stocks (higher = cheaper)
- `ihsgMonthly` — IHSG monthly return percentage (negative = market down = opportunity)
- `breadthAbove60` — number of stocks with composite score >= 60
- `watchlistCount` — total stocks in universe
- `drawdown60` — 60-day drawdown (negative = drop from peak)
- `riskScore` — from market regime engine (higher = riskier)

### 2.2 Action Mapping

| Score Range | Action     | Deploy % |
|-------------|------------|----------|
| 0–29        | none       | 0%       |
| 30–49       | small      | 25%      |
| 50–69       | normal     | 50%      |
| 70–89       | aggressive | 75%      |
| 90–100      | deploy     | 90%      |

### 2.3 Crisis Override

When `isCrashActive()` returns true, BPS is overridden to:
```
action = "none", deployPct = 0, cashPct = 100, valid = false
```

### 2.4 React Hook (useBuyPressure)

Wires to live market data through `MKT`, `RS`, `getIhsgDrawdown60()`, and the active engine profile. Memoized on `activeProfileId` and profile weights.

### 2.5 Static Helper (computeBuyPressureFromMarket)

Used by the backtesting engine at each month-end for adaptive DCA mode. Takes raw values instead of module state.

---

## 3. Crash Detection (crashDetector.ts)

### 3.1 Fast Crash (ihsgPrices, currentPrice, crashSensitivity)

```python
window = last 60 IHSG closing prices
max60 = max(window)
ihsgDropPct = ((currentPrice - max60) / max60) × 100

fastCrash = ihsgDropPct <= -crashSensitivity
```

Example: crashSensitivity = 10 → trigger at -10% or worse from 60d peak.

### 3.2 Slow Grind (bearish trend confirmation)

```python
sma20 = SMA(last 20 prices)
sma50 = SMA(last 50 prices)

grindPriceRatio = 1 - (crashSensitivity × 0.5 / 100)
grindSmaRatio   = 1 - (crashSensitivity × 0.2 / 100)

slowGrind = (currentPrice < sma50 × grindPriceRatio) AND (sma20 < sma50 × grindSmaRatio)
```

Example: crashSensitivity = 10 → price below 95% of SMA50 AND SMA20 below 98% of SMA50.

### 3.3 Recovery Detection (Algo Mode)

```python
sma20 = SMA(last 20 prices)
ihsg5dReturn = ((currentPrice - price5daysAgo) / price5daysAgo) × 100

trendRecovery = currentPrice > sma20
momentumRecovery = (ihsg5dReturn >= 2.5%) AND (currentPrice > sma20)

recovery = trendRecovery OR momentumRecovery
```

### 3.4 Single-Stock Crash (detectCrashSingle)

```python
window = last 20 prices + current
trailingHigh = max(window)
dropFromPeak = ((currentPrice - trailingHigh) / trailingHigh) × 100

crash = dropFromPeak <= -sellTrigger
```

### 3.5 Single-Stock Recovery (detectRecoverySingle)

```python
newTrough = min(trough, currentPrice)
riseFromTrough = ((currentPrice - newTrough) / newTrough) × 100

recovery = riseFromTrough >= buyTrigger
```

---

## 4. Market Regime Engine (marketRegimeEngine.ts:311-442)

### 4.1 Decision Tree

```
                    ┌─────────────────────────────────────┐
                    │ isCrashActive() AND bearishTrend?    │
                    └─────────┬───────────────────────────┘
                              │ YES
                    ┌─────────▼───────────┐
                    │ GOLD_DEFENSE         │
                    │ HOLD_GOLD            │
                    └─────────────────────┘
                              │ NO
                    ┌─────────▼───────────┐
                    │ isCrashActive()?     │
                    └─────────┬───────────┘
                              │ YES
                    ┌─────────▼───────────┐
                    │ CASH_DEFENSE         │
                    │ HOLD_CASH            │
                    └─────────────────────┘
                              │ NO
                    ┌─────────▼───────────┐
                    │ bearishTrend?        │
                    │ (!aboveMA20 && !aboveMA50)
                    └─────────┬───────────┘
                              │ YES
                    ┌─────────▼─────────────┐
                    │ RECOVERY_WATCH         │
                    │ WAIT_RECOVERY          │
                    └───────────────────────┘
                              │ NO
                    ┌─────────▼───────────────────┐
                    │ recoveringTrend AND lowBreadth? │
                    │ (aboveMA20 && !aboveMA50)        │
                    └─────────┬───────────────────┘
                              │ YES
                    ┌─────────▼─────────────┐
                    │ RECOVERY_WATCH         │
                    │ WAIT_RECOVERY          │
                    └───────────────────────┘
                              │ NO
                    ┌─────────▼───────────┐
                    │ recoveringTrend?     │
                    └─────────┬───────────┘
                              │ YES
                    ┌─────────▼───────────┐
                    │ RISK_OFF             │
                    │ WAIT_RECOVERY        │
                    └─────────────────────┘
                              │ NO
                    ┌─────────▼───────────┐
                    │ lowBreadth?          │
                    │ (above60 < 15% of total)
                    └─────────┬───────────┘
                              │ YES
                    ┌─────────▼───────────┐
                    │ RISK_OFF             │
                    │ WAIT_RECOVERY        │
                    └─────────────────────┘
                              │ NO
                    ┌─────────▼───────────┐
                    │ RISK_ON              │
                    │ BUY_STOCKS           │
                    └─────────────────────┘
```

**Key thresholds:**
- `lowBreadth`: fewer than 15% of tracked stocks have composite score >= 60
- `bearishTrend`: price below both SMA20 and SMA50
- `recoveringTrend`: price above SMA20 but below SMA50
- `bullishTrend`: price above both SMA20 and SMA50

### 4.2 Health Score (0–99)

```python
trendComponent = +40 if bullish, -15 if bearish, +5 otherwise
breadthComponent = (above60 / totalTracked) × 30
ihsgComponent = max(0, min(20, 20 + ihsgMonthly × 2))

healthScore = min(99, max(1, round(trendComponent + breadthComponent + ihsgComponent)))
```

### 4.3 Opportunity Score (0–99)

```python
if regime == RISK_ON:
    score = 60 + (above60 / totalTracked) × 30
elif regime == RECOVERY_WATCH:
    score = 40 + (above60 / totalTracked) × 20
else:
    score = 15 + (above60 / totalTracked) × 15
```

### 4.4 Risk Score (0–99)

```python
if regime == GOLD_DEFENSE:
    score = 85
elif regime == RISK_ON:
    score = 15 + (1 - above60/totalTracked) × 20
else:
    score = 40 + (1 - above60/totalTracked) × 30
```

### 4.5 Capital Deployment (%)

```python
if regime == RISK_ON:
    deployPct = min(95, 40 + (above60 / totalTracked) × 40)
elif regime == RECOVERY_WATCH:
    deployPct = 25
elif regime == RISK_OFF:
    deployPct = 15
else:
    deployPct = 0
```

---

## 5. Performance Metrics (metrics.ts)

### 5.1 CAGR
```python
years = daysDiff / 365.25
CAGR = (finalValue / capital)^(1/years) - 1
```
Returned as percentage (× 100).

### 5.2 Sharpe Ratio
```python
annualVolatility = stdDev(dailyReturns) × sqrt(252) / 100
rf = 5%
Sharpe = (CAGR - rf) / annualVolatility
```
Assumes risk-free rate of 5% and 252 trading days per year.

### 5.3 Sortino Ratio
```python
negativeReturns = filter(dailyReturns, r < 0)
downsideVol = stdDev(negativeReturns) × sqrt(252) / 100
if downsideVol == 0: downsideVol = annualVolatility  # fallback

Sortino = (CAGR - rf) / downsideVol
```
Only penalizes negative volatility — more relevant for retail investors than Sharpe.

### 5.4 Calmar Ratio
```python
Calmar = CAGR / (maxDrawdown / 100)
```
Ratio of CAGR to maximum drawdown (as decimal).

### 5.5 Turnover
```python
averagePortfolioValue = (capital + finalValue) / 2
Turnover = totalTransactionVolume / averagePortfolioValue
```
Returned as percentage.

### 5.6 Win Rate
```python
positiveDays = count(r > 0 for r in dailyReturns)
WinRate = positiveDays / totalDays
```
Returned as percentage.

### 5.7 60/40 Benchmark
```python
benchFinalValue = (0.6 × ihsgReturn + 0.4 × goldReturn) × capital
benchReturn = 0.6 × ihsgReturnPct + 0.4 × goldReturnPct
```
Simple benchmark: 60% IHSG, 40% gold. No rebalancing.
