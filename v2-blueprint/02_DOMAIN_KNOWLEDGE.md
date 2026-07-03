# 02 — Domain Knowledge

> Business domain knowledge extracted from QuantBit v1 that must be preserved in V2.

---

## 1. IDX Market Structure

### Trading Hours
- **Regular session:** 09:00–15:00 WIB (UTC+7)
- **Pre-opening:** 08:30–09:00 WIB
- **No DST** — Indonesia does not observe daylight saving time

### Lot Size
- 1 lot = 100 shares (standard for IDX)
- All order quantities must be multiples of 100
- Partial lots (odd lots) trade at a discount through negotiated boards

### Trading Calendar
- Monday–Friday (Saturday/Sunday closed)
- Public holidays: ~15-20 days/year (Islamic holidays, Christmas, New Year, Independence Day)
- Effective trading days: ~240-247/year (not 252 as in US markets)

### Settlement
- T+2 settlement
- Buy: pay T+0, receive shares T+2
- Sell: shares frozen T+0, receive cash T+2

---

## 2. IDX Indices

| Index | Description | Tickers |
|-------|-------------|---------|
| IHSG (JKSE) | Composite index — all listed stocks | ~830 |
| IDX30 | Top 30 by liquidity + fundamentals | 30 |
| IDX80 | Top 80 (includes IDX30 + 50 more) | 80 |
| LQ45 | Top 45 by liquidity | 45 |
| IDXBUMN20 | State-owned enterprises | 20 |
| JII70 | Sharia-compliant | 70 |

**IMPORTANT:** IDX80 composition changes every 6 months (May and November). The ticker list must be updated regularly.

### Sector Classification (IDX-IC)
- Energy
- Basic Materials
- Industrials
- Consumer Non-Cyclicals
- Consumer Cyclicals
- Healthcare
- Financials
- Infrastructure
- Technology
- Transportation & Logistics
- Properties & Real Estate

---

## 3. Trading Fees (IDX)

| Component | Rate | Description |
|-----------|------|-------------|
| Buy fee | 0.15% | Broker commission (varies by broker) |
| Sell fee | 0.25% | Broker commission + clearing fee |
| Tax | 0.10% | Final tax on stock sale proceeds |
| Slippage | 0.25% | Estimated market impact |

**Source:** `src/engine/types.ts:54-59`

**Note:** These are estimates for retail investors via conventional brokers. Online brokers (Ajaib, Stockbit) may charge 0.1-0.3%. The system should support configurable fee schedules.

---

## 4. Dividend Rules (Indonesia)

| Rule | Detail |
|------|--------|
| Tax rate | 10% (WPN with NPWP), 20% (non-NPWP/foreign) |
| Frequency | Annual (most), semi-annual (interim + final for some) |
| Payment | Typically 1-2 months after AGM |
| Ex-date | T+2 before record date |
| Recording | Cash dividend, stock dividend, or scrip dividend |
| **Source** | `src/engine/core.ts:243-273` |

**Key hidden knowledge:** Most Indonesian companies pay dividends after their AGM (April-June). Some pay interim dividends (December). The heuristic "June 15" used in v1 is an approximation.

---

## 5. Investment Profiles

Three profiles discovered from user behavior and implemented in v1:

| Profile | Quality | Growth | Value | Momentum | Dividend | Target Investor |
|---------|---------|--------|-------|----------|----------|-----------------|
| AMAN (Conservative) | 30% | 45% | 10% | 0% | 15% | Risk-averse, long-term |
| AGRESIF (Aggressive) | 20% | 60% | 10% | 10% | 0% | Growth-seeker, high risk |
| DIVIDEN (Dividend) | 15% | 20% | 5% | 0% | 60% | Income-focused |

**Source:** `src/contexts/EngineConfigContext.tsx:37-41`

**Reality check:** These weights were designed by the developer based on general portfolio theory, NOT validated by backtesting or user research. They should be treated as starting defaults that users should tune.

---

## 6. Stock Scoring Framework

Every stock is scored on 5 factors (normalized 0-95 via percentile rank):

| Factor | Meaning | Data Source |
|--------|---------|-------------|
| **Quality** | Financial health (ROE, DER, margins) | IDX Warehouse, RSI-based |
| **Growth** | Revenue/profit growth trajectory | Price momentum over 6 months |
| **Value** | Undervaluation (P/B, P/E inverse) | Inverse of market multiples |
| **Momentum** | Short-term price trend | Price rate of change |
| **Dividend** | Dividend yield percentile | Historical DPS / current price |

**Final score** = `Q*Wq + G*Wg + V*Wv + M*Wm + D*Wd`

Ranking: Sort all tickers by final score descending → assign rank 1, 2, 3...N

---

## 7. Buy Pressure Score (BPS)

A 5-factor composite for timing DCA deployments (0-100 scale):

| Factor | Weight | Direction | Formula |
|--------|--------|-----------|---------|
| Valuation | 30% | Higher = cheaper | Average value score (clamped 0-100) |
| Momentum | 25% | Higher = oversold | `clamp(50 - monthlyReturn*2, 0, 100)` |
| Breadth | 15% | Higher = more pressure | `(1 - ratioAbove60)*100` |
| Drawdown | 20% | Higher = deeper | `clamp(-drawdown60*4, 0, 100)` |
| Fear | 10% | Higher = riskier | Risk score from regime engine (0-100) |

**Action Mapping:**

| BPS Range | Action | Deploy % |
|-----------|--------|----------|
| < 30 | None | 0% |
| 30-49 | Small | 25% |
| 50-69 | Normal | 50% |
| 70-89 | Aggressive | 75% |
| ≥ 90 | Deploy | 90% |

Crisis override: If `isCrashActive()` → BPS = 0, deploy = 0%, cash = 100%.

---

## 8. Market Regime States

The regime engine combines technical indicators (RSI, MACD, SMA20/50, drawdown60) + breadth analysis to classify market regime:

| State | Meaning | Action |
|-------|---------|--------|
| GOLD_DEFENSE | Crisis + bearish | Liquidate stocks → buy gold |
| CASH_DEFENSE | Crisis but not bearish | Liquidate → hold cash |
| RECOVERY_WATCH | Below MAs, or recovering with weak breadth | Wait, do nothing |
| RISK_OFF | Mixed signals or low breadth | Reduce exposure |
| RISK_ON | Bullish conditions | Full stock allocation |

**Decision priority:** First match in this order wins (top = highest priority).

---

## 9. Data Status Taxonomy

Every data point in the system carries a freshness indicator:

| Status | Meaning | Source |
|--------|---------|--------|
| LIVE | Real-time data (< 5 min old) | GoAPI polling, WebSocket |
| CACHED | From last sync (< 24h old) | D1 daily snapshot |
| STALE | Older than acceptable | Last known values |
| ESTIMATED | Computed/interpolated | Synthetic, no direct source |

**Source:** `src/types/DataStatus.ts`

---

## 10. Gold as Safe Haven

In QuantBit, gold is used as a crash hedge (not a profit center):

- Gold price source: IHSG data component (not direct gold market)
- Buy: 1% premium over spot (`goldPrice * 1.01`)
- Sell: 1% discount (`goldPrice * 0.99`)
- Effective spread: ~2% round-trip
- Only used when `safeHavenAsset === "emas"` during crash protection
- Gold position is tracked in grams, not currency

**Known issue:** Gold spread of 2% may differ from actual Antam/gold dealer spreads.
