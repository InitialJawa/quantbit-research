# 19 — Hidden Knowledge

> **Business logic yang tersembunyi di dalam kode.**
> **Asumsi yang tidak pernah didokumentasikan.**
> **Konfigurasi penting yang hanya diketahui dari implementasi.**
> **Magic numbers, hardcode, dan dependency tersembunyi.**
> **Hal-hal yang wajib dipindahkan ke dokumentasi sebelum membangun ulang.**

Dokumen ini adalah yang paling penting untuk dibaca SEBELUM menulis baris kode pertama di Quantbit v2.

---

## Bagian 1: Magic Numbers

### 1.1 Trading Fees (src/engine/types.ts:54-59, allocator.ts)
```typescript
// DEFAULT_FEES
buyFee: 0.0015   // 0.15%
sellFee: 0.0025  // 0.25%
tax: 0.0010      // 0.10%
slippage: 0.0025 // 0.25%
```
**Hidden assumption**: Ini untuk investor retail IDX via sekuritas konvensional. Fee ini TIDAK sama untuk semua broker. Broker online (Ajaib, Stockbit) punya fee berbeda.

### 1.2 Gold Spread (src/engine/allocator.ts:90-101)
```typescript
// Buy gold: 1% premium
const goldBuyPrice = goldPrice * 1.01;
// Sell gold: 1% discount
const goldSellPrice = goldPrice * 0.99;
```
**Hidden assumption**: Gold price yang digunakan adalah harga emas Antam/Lokal (dari IHSG data). Spread 2% (1% each way) adalah asumsi. Tidak pernah diverifikasi dengan harga emas real-time aktual.

### 1.3 Risk-Free Rate (src/engine/metrics.ts:70)
```typescript
const rf = 0.050; // 5%
```
**Hidden assumption**: Risk-free rate 5% untuk Sharpe/Sortino. Ini adalah asumsi. BI rate historis 3.5-6%. Harusnya diambil dari data aktual (SBN yield), bukan hardcoded.

### 1.4 Trading Days (src/engine/metrics.ts:63)
```typescript
const annVolatility = calcStdDev(dailyReturns) * Math.sqrt(252) / 100;
```
**Hidden assumption**: 252 trading days/year (AS market). Bursa Indonesia biasanya 240-247 hari (libur Lebaran, Natal, tahun baru). Error ~2-5% di annualized volatility.

### 1.5 Lot Size (src/engine/allocator.ts)
```typescript
let maxLots = Math.floor(perStockAlloc / (costPerShareWithFee * 100));
let sharesToBuy = maxLots * 100;
```
**Hidden assumption**: 100 shares = 1 lot. Ini benar untuk IDX. TAPI beberapa aturan baru OJK menyebutkan kemungkinan fractional shares. Perlu monitor.

### 1.6 Volume Constraint (src/engine/allocator.ts:52)
```typescript
const maxVolShares = Math.floor((dailyVol * 0.05) / 100) * 100;
```
**Hidden assumption**: Maksimal 5% dari volume harian. Ini adalah asumsi likuiditas — tidak diverifikasi. Untuk saham kecil (IDX80 non-LQ45), 5% bisa sangat besar atau sangat kecil.

### 1.7 Dividend Tax (src/engine/core.ts:249)
```typescript
const divPaid = Math.round(shares * dps * 0.90); // net 90%
```
**Hidden assumption**: Pajak dividen 10% (PPh final). Ini benar untuk wajib pajak Indonesia yang punya NPWP. TAPI:
- Warga asing: 20%
- WP tanpa NPWP: 20%
- Tax treaty: bervariasi

### 1.8 Dividend Date (src/engine/core.ts:243)
```typescript
if (currentYear > lastJulyYear && currentMonth >= 5 && dateObj.getDate() >= 15) {
```
**Hidden assumption**: Dividen dibayarkan setiap tahun setelah 15 Juni. Ini TIDAK AKURAT untuk semua emiten. Banyak emiten bayar dividen di bulan berbeda (Maret, April, Agustus). Beberapa bayar 2x setahun (interim + final).

### 1.9 Crash Sensitivity Default (marketRegimeEngine.ts:44)
```typescript
_crashSensitivity = 10; // default 10%
```
**Hidden assumption**: IHSG drawdown 10% = crash. Ini tidak pernah divalidasi secara statistik. 10% drawdown terjadi 3-5x per tahun di IDX. Apakah itu benar-benar "crash"?

### 1.10 Crash Cooldown (marketRegimeEngine.ts:75, 83)
```typescript
_crisisCooldown = 20; // 20 data points
```
**Hidden assumption**: Cooldown 20 data points (trading days = ~1 bulan). Ini dihitung dalam trading days, bukan calendar days. Berarti cooldown 20 hari kerja (~28 hari kalender). Tidak didokumentasikan di mana pun.

### 1.11 Recovery Threshold (crashDetector.ts:64)
```typescript
const momentumRecovery = ihsg5dReturn >= 2.5 && currentIhsgPrice > sma20;
```
**Hidden assumption**: Recovery jika IHSG naik 2.5% dalam 5 hari + di atas SMA20. Angka 2.5% adalah magic number.

### 1.12 SMA Period dan Slow Grind Ratios (crashDetector.ts:22-23)
```typescript
const grindPriceRatio = 1 - (crashSensitivity * 0.5) / 100;
const grindSmaRatio = 1 - (crashSensitivity * 0.2) / 100;
```
**Hidden assumption**: Slow grind = price < SMA50 * ratio (0.5x crash sensitivity) AND SMA20 < SMA50 * ratio (0.2x crash sensitivity). Formula ini tidak muncul di dokumentasi mana pun.

### 1.13 Memory Budget (aiChatHandler.ts:38)
```typescript
export const MEMORY_MAX_CHARS = 10_000; // ~20 messages
```
**Hidden assumption**: 10K chars untuk memory. Berdasarkan "20 messages × ~500 chars/msg". Ini arbitrer.

### 1.14 OpenRouter Referrer Headers (aiChatHandler.ts:469)
```typescript
const orHeaders = { "HTTP-Referer": "https://quantbit.pages.dev", "X-Title": "Quantbit" };
```
**Hidden dependency**: OpenRouter mensyaratkan HTTP-Referer dan X-Title headers. Tanpa ini, beberapa model OpenRouter (terutama free) menolak request.

### 1.15 Provider Priorities (aiChatHandler.ts:468-584)
```typescript
// OpenRouter models: priority 1-4
// Cohere: priority 5
// Mistral: priority 6
// Groq: priority 7-8
// Gemini: priority 9
```
**Hidden strategy**: OpenRouter di-prioritaskan karena tidak geo-block dari Cloudflare edge. Provider lain adalah fallback, diurutkan dari yang paling mungkin work ke paling mungkin di-block.

### 1.16 Cooldown Durations
```typescript
cooldown429Ms: 5 * 60 * 1000;  // 5 menit
cooldown403Ms: 15 * 60 * 1000; // 15 menit
```
**Hidden assumption**: Rate limit (429) cooldown 5 menit, auth error (401/403) cooldown 15 menit. Ini tidak berasal dari dokumentasi provider mana pun.

### 1.17 RSI Period (marketRegimeEngine.ts:193)
```typescript
export function computeRSI(data: number[], period: number = 14): number | null {
```
**Hidden assumption**: RSI period default 14. Standar umum.

### 1.18 Chart Sampling Rate (core.ts:494)
```typescript
if (stepIndex % 8 === 0 || stepIndex === filtered.length - 1) {
```
**Hidden assumption**: Chart data di-sampling setiap 8 hari. Untuk backtest 5 tahun (~1250 hari) → ~156 data points. Ini untuk mengurangi ukuran response.

### 1.19 60/40 Benchmark (metrics.ts:81-84)
```typescript
const bench6040FinalVal = Math.round(
  (0.6 * (lastIhsgPrice / initialIhsgPrice) + 0.4 * (lastGoldPrice / initialGoldPrice)) * cap
);
const bench6040ReturnPct = 0.6 * ihsgReturnPct + 0.4 * goldReturnPct;
```
**Hidden assumption**: 60% IHSG + 40% Gold sebagai benchmark. Tidak ada rationale yang didokumentasikan. Kenapa bukan 60/40 saham/obligasi (standar portofolio tradisional)?

### 1.20 Average Portfolio Value (metrics.ts:75)
```typescript
const avgPortfolioVal = (cap + currentPortfolioVal) / 2;
```
**Hidden assumption**: Portfolio value tumbuh linear. Ini tidak akurat untuk portfolio dengan return tidak linear. Harusnya pakai time-weighted average.

---

## Bagian 2: Hardcoded Data

### 2.1 Hardcoded Profile Weights (multiple files)
**AMAN**: Q=0.30, G=0.45, V=0.10, M=0.00, D=0.15
**AGRESIF**: Q=0.20, G=0.60, V=0.10, M=0.10, D=0.00
**DIVIDEN**: Q=0.15, G=0.20, V=0.05, M=0.00, D=0.60

**Hidden in**: src/contexts/EngineConfigContext.tsx (DEFAULT_PROFILES), functions/api/[[path]].ts:457-459 (defaultWeights for backtest-data), marketRegimeEngine.ts:326-333 (CW_AMAN ref)
**Problem**: Duplicated in 3+ places. Any change must update all.

### 2.2 Hardcoded IDX80 Ticker List (functions/api/[[path]].ts:1101-1114)
```typescript
const tickers = [
    "ADRO.JK", "AKRA.JK", "AMRT.JK", ... // 78 tickers
];
```
**Hidden in**: `runIdx80Scan()` function
**Problem**: IDX80 composition changes every 6 months (May & November). This list WILL be stale. Also duplicated in `src/constants/idx80.ts`.

### 2.3 Hardcoded Default Portfolio (functions/api/[[path]].ts:205-210)
```typescript
const defaultState = {
    portfolio: [
        { ticker: "BBCA", shares: 500, buyPrice: 9900, addedAt: ... },
        { ticker: "BBRI", shares: 1000, buyPrice: 4900, addedAt: ... },
    ],
    cash: 100000000,
    // ...
};
```
**Hidden in**: GET /api/engine/state
**Problem**: Returns fake data when no real portfolio exists. BBCA at 9900 and BBRI at 4900 are rough estimates — not real prices.

### 2.4 Hardcoded Gold/USD Rates (scripts/fetch_historical_data.ts:8-25)
```typescript
const HISTORICAL_GOLD_USD_YEARLY: Record<number, number> = {
   2000: 280, 2001: 265, ... 2026: 2600,
};
const HISTORICAL_USDIDR_YEARLY: Record<number, number> = {
   2000: 8400, 2001: 10400, ... 2026: 16600,
};
```
**Hidden assumption**: Yearly averages interpolated monthly. For 2026, value 2600 and 16600 are GUESSES (2026 isn't over yet).

### 2.5 Hardcoded Fundamental Snapshots (scripts/fetch_historical_data.ts:55-95)
```typescript
const FUNDAMENTAL_SNAPSHOTS: Record<string, Record<number, {...}>> = {
    BBCA: { 2018: { roe: 0.20, pb: 4.1, ... }, ... },
    BBRI: { 2018: { roe: 0.17, pb: 2.3, ... }, ... },
    BMRI: { ... },
    TLKM: { ... },
    ASII: { ... },
    // Only 5 tickers!
};
```
**Only 5 of 830+ tickers have real fundamental data**. The rest get synthetic data:
```typescript
// Line 55 comment: "For the rest of the 80, we generate a stable but
// slightly randomized set of static fundamentals"
```

### 2.6 Hardcoded Yahoo Price Fallback (scripts/fetch_historical_data.ts)
The entire script has extensive fallback logic when Yahoo Finance fails:
- Generated price data
- Flat volume data
- Interpolated market data
- The data is "close enough" but not real

### 2.7 Hardcoded 15 Concurrent Workers (functions/api/[[path]].ts:1154)
```typescript
await Promise.all(Array.from({ length: 15 }, () => worker()));
```
**Hidden assumption**: 15 concurrent fetches to Yahoo Finance is acceptable. Yahoo may rate-limit or block at higher concurrency.

---

## Bagian 3: Asumsi Tidak Terdokumentasi

### 3.1 "DB = Single Source of Truth" — Tapi Tidak Selalu
AGENTS.md Rule 4: "DB = single source of truth untuk market data"

**Reality**: 
- Backtest engine reads from API response (functions/api/[[path]].ts)
- marketRegimeEngine reads from MKT singleton (marketData.ts)
- BPS reads from MKT/RS singletons
- These singletons are populated by useDataFeed hook which calls Yahoo API directly

**Hidden truth**: Engine components DON'T actually query the database. They read from in-memory singletons that are loaded from API responses.

### 3.2 Normalized Scores Range
**Assumption**: Stock norm scores range 0-95 (from ranker.ts comment: "already normalized 0-95 by data pipeline")
**Reality**: Scores can be higher or lower depending on the normalization algorithm in `migrate-normscores.ts`. Not guaranteed to be 0-95.

### 3.3 Carried-Forward Data
**Assumption**: Data is "clean"
**Reality**: Some data points have `isCarriedForward: true` flag. The engine sometimes uses stale/carried data, sometimes skips it. Inconsistency documented in ADR-010.

### 3.4 Rank Key Fallback
```typescript
const activeProfileKey: "stockRanksProd" | "stockRanksRes" =
    config.activeProfileId === "agresif" ? "stockRanksRes" : "stockRanksProd";
```
**Hidden logic**: Custom profiles fallback to "aman" rank keys. Alasan: "data only has Prod/Res". Ini adalah workaround, bukan design intention.

### 3.5 Dividend Score = 0 in Weighted Sum (ranker.ts:17)
```typescript
(ns.dividend ?? 0) * profileWeights.dividend;
```
**Hidden behavior**: When dividend score is missing, default is 0 (not 50 like the other factors). This means missing dividend data = zero contribution, even if profile has dividend weight > 0. This is INCONSISTENT with other factors that default to 50.

### 3.6 BPS avgValueScore Default (aiClient.ts:126)
```typescript
50, /* avgValueScore — use 50 as default; precise per-universe score requires leader scan */
```
**Hidden behavior**: BPS sent to AI uses default value score = 50 (neutral). This means the BPS shown to AI may differ from the engine's BPS which uses actual computed value scores.

### 3.7 "Rico Lubis" Persona
**Hidden in**: systemKnowledge.ts (BEHAVIOR section)
The AI persona "Rico Lubis" (Jaksel analyst) is:
- Not configurable
- Not mentioned outside the systemKnowledge.ts file
- Hardcoded to use Indonesian-English mix
- Has specific slang ("gas", "cekidot", "jalankan")

**Impact**: Users outside Indonesia may find this confusing. The persona cannot be changed without modifying source code.

### 3.8 Dev Mode Detection (aiChatHandler.ts:711-733)
```typescript
if (isDev) {
    lines.push("**Solusi (dev mode):**");
    lines.push("1. Edit file `.env.local` di root project");
} else {
    lines.push("**Solusi (production — Cloudflare Dashboard):**");
}
```
**Hidden behavior**: The AI error message changes based on `isDev` flag. In dev mode, it tells user to edit `.env.local`. In production, it points to Cloudflare Dashboard.

---

## Bagian 4: Dependency Tersembunyi

### 4.1 Yahoo Finance API — Undocumented
**Used by**: scripts/, functions/api/[[path]].ts (runIdx80Scan)
**Endpoint**: `query1.finance.yahoo.com/v8/finance/chart/{ticker}`
**Status**: Undocumented, reverse-engineered, can break anytime
**Fallback**: Synthetic data generation (not reliable)

### 4.2 GoAPI — Live Not Production-Ready
**Used by**: functions/api/[[path]].ts:1001-1021
**Key in URL**: `api.goapi.io/stock/idx/prices?api_key=${key}`
**Status**: Only fetches 9 specific tickers (BBCA, BBRI, BMRI, TLKM, ASII, ADRO, PTBA, ESSA, GOTO)
**Problem**: Hardcoded watchlist, not configurable

### 4.3 Cloudflare D1 — Ecosystem Lock-In
**Used by**: functions/api/[[path]].ts
**Compatibel with**: SQLite (mostly)
**Limitations**:
- No row-level security (without Workers)
- No point-in-time recovery
- 10GB max (per database)
- Cold starts on first query

### 4.4 Resend.com — Email Sending
**Used by**: functions/api/[[path]].ts:629-647 (POST /api/send-notification)
**Dependency**: Requires `RESEND_API_KEY` env var
**Status**: Only 1 feature (notifications) depends on this

### 4.5 better-sqlite3 — Native Build Issues
**Used by**: scripts/ (local data pipeline)
**Problem**: Requires native compilation. Fails in many CI environments. Python fallback (`seed-db.py`) exists for this reason.

### 4.6 cloudscraper — Python Dependency
**Used by**: collectors/fetch_idx_fundamental.py
**Problem**: Bypasses Cloudflare protection. May break when Cloudflare updates anti-bot measures. Not production-grade.

### 4.7 Data File Size Constraints
**Current sizes**:
- `fundamental_idx_all.json`: 41 MB (all tickers, 60 months)
- `data/years/*.json`: ~200+ MB total (all years)
- `RAW_STOCKS_DATA`: ~2 MB (830 tickers in TypeScript)
- `idx80_scan.json`: 190 KB (current snapshot)

**Hidden constraint**: Vite build will bundle these for production. 41MB JSON = slow initial load.

---

## Bagian 5: Konfigurasi Environment

### 5.1 Required Environment Variables (.env.example)
```
OPENROUTER_API_KEY=sk-or-v1-...
GROQ_API_KEY=gsk_...
GEMINI_API_KEY=AIza...
COHERE_API_KEY=...
MISTRAL_API_KEY=...
GOAPI_API_KEY=...
RESEND_API_KEY=...
CRON_SECRET=...
EMAIL_FROM=QuantBit <onboarding@resend.dev>
EMAIL_TO=user@example.com
EMAIL_USER=user@example.com
```

**Hidden requirement**: Total 10+ environment variables needed for full functionality. Missing any = degraded experience.

### 5.2 OpenRouter Default Models (aiChatHandler.ts:239-249)
```
openrouter: "openai/gpt-oss-120b:free"
openrouter-2: "nvidia/nemotron-3-super-120b-a12b:free"
openrouter-3: "cohere/north-mini-code:free"
openrouter-4: "meta-llama/llama-3.3-70b-instruct:free"
cohere: "command-a-plus-05-2026"
mistral: "mistral-small-latest"
groq: "groq/compound"
groq-fallback: "llama-3.3-70b-versatile"
gemini: "gemma-4-26b-a4b-it"
gemini-fallback: "gemma-4-31b-it"
```

**Hidden dependency**: These model names CAN change (new versions, deprecation). If a model name becomes invalid, that provider silently fails and falls to next.

### 5.3 Wrangler Configuration (wrangler.toml)
```toml
name = "quantbit-terminal"
compatibility_date = "2026-06-22"
pages_build_output_dir = "dist"

[[d1_databases]]
binding = "DB"
database_name = "quantbit-db"
database_id = "..."

[env.production.variables]
GROQ_MODEL = "groq/compound"
GROQ_FALLBACK_MODEL = "llama-3.3-70b-versatile"
```

**Hidden**: D1 database ID is hardcoded. Different environments (dev/staging/prod) need different IDs.

---

## Bagian 6: Hal Lain yang Wajib Didokumentasikan

### 6.1 Data Pipeline Error Handling
All data pipeline scripts have silent failure modes:
- `try/catch { /* skip ticker */ }` — Yahoo fetch failures silently skip tickers
- `catch { /* Skip year if fetch fails */ }` — missing years silently skipped
- Empty arrays returned for no data
- No alerts when data pipeline partially fails

### 6.2 Synthetic vs Real Data Mix
The system mixes real and synthetic data without clear labeling:
- Historical prices: Real (Yahoo Finance)
- Fundamentals for top 5 tickers: Real
- Fundamentals for rest: Synthetic (generated)
- Gold prices before 2021: Synthetic (interpolated from yearly averages)
- USD/IDR before 2021: Synthetic

### 6.3 Timezone Handling
- Market data timestamps: WIB (UTC+7)
- Yahoo Finance data: UTC
- GitHub Actions cron: UTC
- SQLite dates: ISO string (no timezone)
- D1 dates: `datetime('now')` = UTC

**No explicit timezone conversion in the data pipeline.** Risk: date misalignment of 1 day during DST transitions (though Indonesia doesn't observe DST).

### 6.4 localStorage Strategy
- `engineConfig` — JSON string, full state
- `quantbit_session` — session token
- `quantbit_chat_history` — AI chat history (capped at 100 messages)
- `ai_test_harness_logs` — dev-only

### 6.5 Debug Mode (AITestHarness)
The AITestHarness component has 4 tabs:
1. **Tools** — Test each read-only tool
2. **Actions** — Test each action (with approval simulation)
3. **Cooldown** — View/reset AI provider cooldowns
4. **Storage** — View/edit localStorage

This is accessible in production if the component is rendered (controlled by `import.meta.env.DEV` in some but not all paths).

---

## Ringkasan

| Kategori | Jumlah Item | Impact |
|----------|-------------|--------|
| Magic Numbers | 20 | HIGH — formulas tidak terdokumentasi |
| Hardcoded Data | 7 | HIGH — stale data, maintenance burden |
| Asumsi Tidak Terdokumentasi | 8 | MEDIUM — bisa menyebabkan bug subtle |
| Dependency Tersembunyi | 7 | MEDIUM — risk of silent failure |
| Environment Config | 3 | LOW — documented in .env.example |
| Hal Lain | 5 | MEDIUM — perlu diperhatikan di v2 |

**Total**: 50 item hidden knowledge yang WAJIB didokumentasikan sebelum membangun v2.

> **Peringatan**: Jika hidden knowledge ini tidak dipindahkan ke dokumentasi v2, risiko misinterpretasi bisnis logic sangat tinggi. Setiap magic number di atas adalah hasil dari trial-and-error di v1. Jangan asumsi angka-angka ini tanpa memahami konteksnya.
