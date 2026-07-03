# Backtesting Engine — Complete Documentation

**Source:** `src/engine/core.ts` (670 lines), `src/engine/allocator.ts` (162 lines), `src/engine/types.ts` (134 lines)  
**Date:** 2026-07-03

---

## 1. Architecture Overview

### 1.1 Entry Point

```ts
function runStrategy(input: StrategiesInput): BacktestResult
```

Defined at `src/engine/core.ts:27`. Everything starts here.

### 1.2 Input (StrategiesInput)

```ts
interface StrategiesInput {
  dayData: BacktestDayData[];      // Historical market data (daily)
  config: BacktestConfig;           // Strategy parameters
  profileWeights: ProfileWeights;   // Factor weights (Q/G/V/M/D)
  universeTickers: {                // Available ticker universes
    idx80: string[];
    idx30: string[];
    lq45: string[];
  };
  fees?: ExecutionFees;             // Transaction cost model
}
```

### 1.3 Output (BacktestResult)

```ts
interface BacktestResult {
  finalValue: number;
  ihsgFinalValue: number;
  goldFinalValue: number;
  totalReturnPct: number;
  cagr: number;
  sharpe: number;
  sortino: number;
  calmar: number;
  maxDrawdown: number;
  winRatePct: number;
  turnoverPct: number;
  totalTrades: number;
  totalDividends: number;
  dividendByTicker: Record<string, number>;
  bench6040FinalVal: number;
  bench6040ReturnPct: number;
  logs: TradeLog[];
  chartData: ChartPoint[];
  bpsHistory?: BpsSnapshot[];       // Adaptive DCA only
  crashTriggered?: boolean;
  crashCount?: number;
  finalPositions?: Record<string, number>;
  finalCash?: number;
  finalGoldGrams?: number;
}
```

---

## 2. Processing Pipeline

### Phase 1 — Data Preparation (lines 39-82)

```python
for each day in dayData:
    if day has stockNormScores:
        # Enrich with dividend scores from FUNDAMENTAL_SNAPSHOTS
        for each ticker in day:
            if dividend is missing:
                yieldPct = dividend_per_share[ticker][year] / price
                rank-normalize yields to 0-95
        # Recompute ranks from norm scores using current profile weights
        day.stockRanks = computeDayRankings(enrichedScores, currentWeights)
    else:
        # Use pre-computed ranks (Prod or Res key)
        day.stockRanks = day[activeProfileKey] or day.stockRanks
```

**Dividend enrichment detail:** When `stockNormScores` is available but dividend score is missing, the engine reads from `dividend_snapshots.json`. If DPS is found, dividend score is computed as a rank-normalized percentile (0-95) across all tickers — not the raw yield. This ensures relative comparison.

**Rank key fallback:** Custom profiles and "dividen" profile fall back to "aman" rank key (`stockRanksProd`) as a defensive default.

### Phase 2 — Date Filtering (lines 84-86)

```python
filtered = [day for day in dayData if simStartDate <= day.date <= simEndDate]
```

If empty, throws error: "Tidak ada data dalam rentang tanggal yang dipilih."

### Phase 3 — Initial Capital Allocation (lines 92-126)

```python
bufferCash = capital × (reserveBufferPct / 100)  # reserved, never deployed
investable = capital - bufferCash

if simulationMode == "algo":
    topTickers = pickTopTickersByRank(day0.ranks, day0.prices, universe, topNCount)
elif simulationMode == "custom":
    topTickers = customUniverse (filtered to those with price > 0)
elif simulationMode == "adaptive_dca":
    topTickers = []  # no pre-computed; each month re-evaluates

initialAlloc = computeInitialAllocation(investable, topTickers, day0.prices, day0.volumes, fees)
```

**Allocation algorithm** (allocator.ts:24-65):
```python
perStockAlloc = investable / topTickers.length

for each ticker:
    entryPrice = rawPrice × (1 + slippage)
    costPerShare = entryPrice × (1 + buyFee)
    maxShares = floor(perStockAlloc / costPerShare / 100) × 100  # round to lots
    maxVolShares = floor(volume × 0.05 / 100) × 100  # 5% of daily volume
    sharesToBuy = min(maxShares, maxVolShares)
```

### Phase 4 — Daily Loop (lines 186-516)

For each day in `filtered[]`:

#### 4a. Pending Ticker Execution (lines 197-219)
Tickers that had no price on day0 get executed when their first price appears. Each pending ticker buys at market price with slippage, rounded to lots.

#### 4b. Portfolio Valuation (lines 221-241)
```python
stocksValue = sum(shares[ticker] × day.stockPrices[ticker])
goldValue = goldGrams × day.goldPrice
portfolioValue = cash + goldValue + stocksValue + bufferCash

dailyReturns.push((todayVal - yesterdayVal) / yesterdayVal × 100)

if portfolioValue > maxVal:
    maxVal = portfolioValue
else:
    drawdown = (maxVal - portfolioValue) / maxVal × 100
    maxDrawdown = max(maxDrawdown, drawdown)
```

#### 4c. Dividend Accrual (lines 243-273, annually in June)
```python
if currentMonth == June AND day >= 15 AND year > lastDividendYear:
    for each position:
        dps = getDividendPerShare(ticker, date)
        if dps > 0:
            dividend = shares × dps × 0.90  # 10% tax
            cash += dividend
```
Dividends are credited once per year (June). 90% net (10% withholding tax).

#### 4d. Crash Detection (lines 275-307)
```python
if enableCrashProtection AND (algo OR custom):
    crash = detectCrashAlgo(ihsgRollingWindow, currentIhsgPrice, crashSensitivity)
    # See 07-scoring-engine.md for formula
```

On crash trigger:
1. Liquidate all stock positions → cash
2. If safeHavenAsset == "emas": convert cash → gold (1% buy spread)
3. Set `inCrashState = true`, `crashCooldown = 20` trading days
4. Log CRASH_TRIGGER

#### 4e. Crash Cooldown & Recovery (lines 309-367)
After 20 trading days in crash state, check for recovery:
```python
recovery = detectRecoveryAlgo(ihsgRollingWindow, currentIhsgPrice)
```

On recovery:
1. Sell all gold (0.99 × goldPrice, 1% sell spread)
2. Re-enter top N tickers using same allocation algorithm
3. Set `crashCooldown = 20` (prevent re-trigger oscillation)

#### 4f. Adaptive DCA — Monthly BPS Deployment (lines 375-431)
Only active when `simulationMode === "adaptive_dca"`. Runs on month change:

```python
ihsgMonthly = ((currentIhsg - ihsg21DaysAgo) / ihsg21DaysAgo) × 100
drawdown60 = ((currentIhsg - peak60) / peak60) × 100
breadthAbove60 = count(stocks with rank <= topNCount × 3)
avgValueScore = average(value score across all stocks)

bps = computeBuyPressureFromMarket(ihsgMonthly, drawdown60, breadthAbove60, ...)
deployAmount = cash × bps.deployPct / 100  # deploy % of AVAILABLE cash

if deployAmount > 0:
    topTickers = pickTopTickersByRank(...)
    alloc = computeInitialAllocation(deployAmount, topTickers, ...)
    positions += alloc.positions
```

Key difference from algo mode: **no monthly rank-based rotation**. Once deployed, positions are held until crash.

#### 4g. Adaptive Weights — Monthly Factor Rebalancing (lines 433-440)
Only active when `enableAdaptiveWeights === true`. Runs on month change:
```python
currentWeights = computeAdaptiveWeights(filtered, stepIndex, 60, currentWeights, topNCount)
newRanks = computeDayRankings(day.stockNormScores, currentWeights)
day.stockRanks = newRanks
```
See 07-scoring-engine.md section 1.5 for weight computation formula.

#### 4h. Rebalancing — Monthly Rank-Based Crossover (lines 442-492)
Only active when `enableCrossover === true` AND `simulationMode === "algo"`.

```python
for each owned ticker:
    currentRank = day.stockRanks[ticker]
    isEmergencyExit = currentRank >= 15  # always trigger
    isRoutineExit = isMonthChange AND currentRank >= 10

    if emergencyExit OR routineExit:
        sellProceeds = sell(ticker)
        swapInTicker = topCandidates.find(t => t != ticker AND not already held)
        if swapInTicker found:
            buy(swapInTicker, with sellProceeds)
```

**Swap rules:**
- No self-swap (won't swap to same ticker)
- No duplicate positions (won't swap to already-held ticker)
- `lastRebalanceMonth` initialized from day0 (prevents day-1 false trigger)
- Custom mode SKIPS rebalancing entirely

#### 4i. Chart Data Sampling (lines 494-515)
Every 8th day (and last day):
```python
chartData.push({
    date: day.date,
    "Strategi Rebalancer": portfolioValue,
    "Benchmark IHSG": (ihsgPrice / initialIhsgPrice) × capital,
    "Benchmark Emas": (goldPrice / initialGoldPrice) × capital,
    ranks: day.ranks,
})
```

### Phase 5 — Metrics Computation (lines 518-581)

After the daily loop completes:
```python
currentPortfolioVal = cash + stocksValue + goldValue + bufferCash
metrics = computeMetrics({
    cap, currentPortfolioVal, day0Date, lastDayDate,
    dailyReturns, maxDrawdownValue, totalTransactionVolume,
    initialIhsgPrice, lastIhsgPrice, initialGoldPrice, lastGoldPrice,
})
```

Returns the final `BacktestResult` object.

---

## 3. Fee & Execution Model (allocator.ts)

| Parameter | Value | Notes |
|-----------|-------|-------|
| Slippage  | 0.25% | Applied to entry AND exit price |
| Buy fee   | 0.15% | Broker commission |
| Sell fee  | 0.25% | Broker commission |
| Tax       | 0.10% | Transaction tax |
| Lot size  | 100 shares | IDX standard |
| Max volume | 5% of daily volume | Per ticker per day |
| Dividend  | 90% net | 10% withholding tax |
| Gold buy spread | 1% | `price × 1.01` |
| Gold sell spread | 1% | `price × 0.99` |
| Buffer cash | `reserveBufferPct%` | Never deployed |

**Execution formula for buy:**
```python
entryPrice = rawPrice × (1 + slippage)          # slippage-adjusted price
costPerShare = entryPrice × (1 + buyFee)        # cost with commission
maxShares = floor(availableCash / costPerShare / 100) × 100  # round down to lots
maxVolShares = floor(volume × 0.05 / 100) × 100  # 5% volume cap
sharesToBuy = min(maxShares, maxVolShares)
```

**Execution formula for sell:**
```python
exitPrice = rawPrice × (1 - slippage)
proceeds = shares × exitPrice × (1 - sellFee - tax)
```

---

## 4. Three Simulation Modes

### Algo Mode
- Top N pick from universe based on composite score
- Monthly rank-based rebalancing (crossover)
- Crash protection → liquidate → safe haven → recover → re-enter

### Custom Mode
- User-defined exclusive universe (no rank-based pick)
- No rebalancing (crossover disabled)
- Crash protection still active

### Adaptive DCA Mode
- No initial allocation (positions deployed month-by-month)
- No rebalancing (no crossover)
- BPS-driven deployment each month
- Crash protection still active

---

## 5. Assumptions & Limitations

### Model Assumptions
- 252 trading days per year
- Risk-free rate = 5%
- Dividends only once per year (June 15+), not tracked per actual ex-date
- Gold price is tracked daily (not tick-by-tick)
- All orders fill at computed price (no order book simulation)
- No market impact beyond 5% volume cap

### Known Weaknesses

1. **Monolithic function (670 lines)** — `runStrategy()` handles data prep, main loop, crash logic, rebalancing, chart sampling, and result assembly in one function. Impossible to unit test sub-steps.

2. **O(n²) history history** — Fixed in A5 fix (line 161-162) by using rolling `ihsgRollingWindow` array capped at 60. Previously called `filtered.slice(0, stepIndex).map(...)` every iteration.

3. **All modes mixed in same loop** — `config.simulationMode` branches create combinatorial complexity. 11 conditionals checking mode inside the main loop.

4. **No parallel processing** — Single-threaded sequential. A 5-year backtest runs in ~50ms currently, but adding tickers or expanding the date range scales linearly.

5. **Data loaded from API each run** — `dayData` is fetched via `GET /api/backtest-data` every time the user clicks "Run". No client-side caching of historical data.

6. **Crash cooldown in trading days** — `crashCooldown = 20` is 20 trading days (~1 calendar month). Not configurable. Uses wall-clock-like countdown rather than market events.

7. **Single configuration per run** — No batch mode or parameter sweep. To compare AMAN vs AGRESIF, user must run twice manually.

8. **Memory allocation grows with day count** — `chartData`, `logs`, and `bpsHistory` arrays grow with the number of trading days. For 1500-day backtest, chartData ≈ 188 points, logs ≈ 100-300 entries. Not critical but could be optimized with ring buffers.
