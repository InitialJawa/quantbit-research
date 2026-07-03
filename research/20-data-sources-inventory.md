# 20 — Data Sources Inventory: Ketidak-konsistenan Single Source of Truth

> Dokumentasi lengkap tentang kegagalan arsitektur data Quantbit v1:
> **10 kategori data yang terfragmentasi di 2-3 sumber berbeda,**
> menyebabkan inkonsistensi, bug subtle, dan ketidakpercayaan pada angka.

---

## Akar Masalah

Quantbit v1 tumbuh organik tanpa keputusan sadar tentang Single Source of Truth (SOT).
Tiap komponen memilih sumber datanya sendiri-sendiri:

| Komponen | Sumber Data | Alasan |
|----------|-------------|--------|
| Backtest engine | JSON `data/years/*.json` | Build-time import, cepat |
| Market engine | In-memory singleton (Yahoo langsung) | Real-time, ga perlu nunggu DB sync |
| Production API | Cloudflare D1 | Persistent, edge-hosted |
| Dev scripts | SQLite lokal | Lebih cepat daripada D1 cold start |
| Scoring engine | `src/data/raw_stocks_data.ts` (static) | Tersedia saat compile |

Akibatnya: **data yang sama bisa punya 3 nilai berbeda** tergantung dari mana dibaca.

---

## Kategori 1: Market Data (IHSG, Gold, USD/IDR, Harga Saham, Volume)

### Sumber-sumber

| # | Sumber | Lokasi | Dipakai oleh | Update |
|---|--------|--------|-------------|--------|
| A | JSON | `data/years/<year>.json` | Backtest engine, SimulationTab | Harian (cron `sync-daily-data.ts`) |
| B | SQLite | `data/historical_market.sqlite` | Dev scripts (`build-db.ts`) | Satu kali (dari JSON) |
| C | D1 | `daily_overview` + `stock_daily` | Production API, force-sync | Harian (seed dari SQLite/JSON) |
| D | Singleton | `MKT.ihsg`, `MKT.gold`, `MKT.usdidr` | Engine (core.ts, ranker.ts, allocator.ts, etc) | Init app (dari `marketData.ts`/`useDataFeed`) |

### Masalah

- **Backtest** pake JSON (A).
- **Decision engine runtime** pake singleton MKT (D) yang ambil dari Yahoo langsung — BUKAN dari DB.
- **Production** pake D1 (C) yang di-seed dari JSON/SQLite.
- Kalau cron gagal seed D1, tapi JSON udah updated — D1 punya data lama.
- Kalau Yahoo down, singleton MKT punya data kosong — sementara JSON/D1 masih punya data bagus.
- Kalau JSON gagal update, tapi D1 berhasil — sebaliknya.

### Garis Waktu Inkonsistensi

```
Hari libur bursa:
  Yahoo:        data kosong
  MKT singleton: data terakhir (stale)
  JSON:         data terakhir (carried forward)
  D1:           is_market_day = 0

  → Engine bacanya beda tergantung urutan load
```

### Rekomendasi v2

**D1 sebagai satu-satunya SOT.** Hapus:
- ✅ `data/years/*.json` — tidak perlu, D1 bisa query langsung
- ✅ `data/historical_market.sqlite` — dev langsung pake D1 (wrangler dev --d1)
- ❌ `MKT` singleton — engine panggil API `/api/market-data` yang baca dari D1

---

## Kategori 2: Norm Scores / Stock Fundamentals

### Sumber-sumber

| # | Sumber | Lokasi | Detail |
|---|--------|--------|--------|
| A | JSON | `data/years/<year>.json` → `stockNormScores` per hari | Pre-computed dari pipeline fetch_historical_data.ts |
| B | TS static | `src/data/raw_stocks_data.ts` | 830+ ticker dengan synthetic scores |
| C | D1 | `stock_fundamentals` (quality, growth, value, momentum) | Di-compute ulang oleh `runIdx80Scan()` |
| D | Static | `src/marketData.ts` — L (leader scores) | Dari hasil build terakhir |

### Masalah

- Raw_stocks_data (B) punya data 830 tickers tapi **hanya 5 yang real** (BBCA, BBRI, BMRI, TLKM, ASII) — sisanya synthetic (`19-hidden-knowledge.md:2.5`).
- D1 stock_fundamentals (C) punya hanya 78 IDX80 tickers — di-compute dengan algoritma **berbeda** dari pipeline JSON (A).
- Pipeline JSON (A) compute scores via percentile rank (0-95) dari data historis.
- Force-sync (C) compute scores via RSI, momentum, dll dari 6 bulan data mingguan.
- **Dua hasil bisa berbeda untuk ticker yang sama pada tanggal yang sama.**

### Contoh Inkonsistensi

```
Ticker BBCA:
  JSON (A):     quality: 82, growth: 71, value: 45, momentum: 63
  D1 (C):       quality: 76, growth: 65, value: 48, momentum: 58
  TS static (B): quality: 90, growth: 80, value: 30, momentum: 70

  → Final score BBCA tergantung dari mana komponen engine baca.
```

### Rekomendasi v2

- ✅ Hanya **satu** algoritma scoring — di backend, bukan di split antara pipeline dan force-sync.
- ✅ Semua hasil scoring disimpan di D1 `stock_fundamentals` — tidak ada duplikat.
- ✅ Hapus `raw_stocks_data.ts` static — tidak perlu.
- ✅ Backtest panggil API `/api/stock-fundamentals` untuk ambil scores di tanggal tertentu.

---

## Kategori 3: Rank Data (rank_prod / rank_res)

### Sumber-sumber

| # | Sumber | Lokasi | Detail |
|---|--------|--------|--------|
| A | D1 | `stock_daily.rank_prod` + `stock_daily.rank_res` | Pre-computed di seed/force-sync |
| B | JSON | `stockRanksProd` + `stockRanksRes` di year files | Pre-computed di pipeline |
| C | Runtime | `core.ts:39-76` | Di-compute ulang dari `stockNormScores` |

### Masalah

- Rank di-compute dari sorting `norm_score`. Tapi kalau norm_score aja berbeda (lihat Kategori 2), rank pasti berbeda.
- `core.ts:39-76` ngitung ulang rank dari `stockNormScores` yang dibaca dari year JSON — ngasih nilai beda dari rank yang tersimpan di D1.
- Backtest pake rank hasil recompute (C), production pake rank D1 (A), local dev pake rank JSON (B).

### Rekomendasi v2

- ✅ Hapus kolom `rank_prod` dan `rank_res` — rank di-compute on-the-fly dari norm_score pada saat query.
- ✅ Generated column atau view di D1 untuk performance.

---

## Kategori 4: Dividend Data

### Sumber-sumber

| # | Sumber | Lokasi | Detail |
|---|--------|--------|--------|
| A | JSON master | `data/fundamental_snapshots.json` | Generated oleh `fetch_dividend_history.ts` |
| B | JSON copy | `src/data/dividend_snapshots.json` | Dicopy manual untuk Vite import |

### Masalah

- B adalah copy dari A. Kalau lupa copy, A updated tapi B stale.
- **Tidak ada auto-copy di pipeline.** File `fetch_dividend_history.ts` punya komentar "copy to src/data" tapi tidak dijalankan otomatis.
- Engine (`core.ts:243`) pake logika **hardcoded date heuristik** (15 Juni) — bukan dari data ini (`19-hidden-knowledge.md:1.8`).

### Rekomendasi v2

- ✅ Simpan dividend data di D1: `dividends(ticker, year, dividend_per_share, pay_date, type)`.
- ✅ Hapus file JSON statis.
- ✅ Engine baca dividend dari D1 per ticker per tahun, bukan dari heuristik tanggal.

---

## Kategori 5: IDX80 Scan Data

### Sumber-sumber

| # | Sumber | Lokasi | Detail |
|---|--------|--------|--------|
| A | JSON | `data/idx80_scan.json` | Dari cron Python `fetch_idx80_scan.py` |
| B | D1 | `idx_scan_data` (JSON blob) | Di-sync via force-sync endpoint |
| C | Partial | Embedded di `data/live_market.json` | Harga + perubahan harian |

### Masalah

- A dan B **bisa berbeda** karena force-sync jalan di akhir pipeline — kalau force-sync gagal, D1 punya data kemarin.
- live_market.json (C) punya subset dari data scan — dan formatnya berbeda (map ticker→price vs array of objects).
- `idx_scan_data` di D1 adalah **JSON blob** — tidak bisa di-query (`05-database-analysis.md:Finding 2`).

### Rekomendasi v2

- ✅ Normalisasi D1: `idx80_scans(ticker, scan_date, current_price, pe_ratio, pb_ratio, ...)` — satu baris per ticker.
- ✅ Hapus file JSON statis.
- ✅ Live market baca dari D1 juga (query `idx80_scans` terbaru).

---

## Kategori 6: User Config (EngineConfig)

### Sumber-sumber

| # | Sumber | Lokasi | Detail |
|---|--------|--------|--------|
| A | localStorage | `idx_engine_config` | Full config, update tiap kali user ganti setting |
| B | D1 | `users.engine_config` + `users.active_config` (JSON blobs) | Disync via PATCH `/api/user/profile` dan GET `/api/engine/state` |
| C | localStorage | Backtest draft (key terpisah) | Konfigurasi backtest sementara |

### Masalah

- **Tiga sumber yang harus di-reconcile setiap login.** Kalau user ganti setting di device A lalu login di device B, config B pakai data D1 yang mungkin lebih lama.
- Konflik: localStorage (A) lebih baru dari D1 (B)? Atau sebaliknya? **Tidak ada mekanisme merge** — siapa yang menulis terakhir, dia yang menang.
- Backtest draft (C) bisa expired — user lupa ada draft, lalu ngerjain ulang.

### Rekomendasi v2

- ✅ D1 sebagai SOT untuk user config. localStorage hanya sebagai cache dengan timestamp.
- ✅ Mekanisme merge sederhana: bandingkan timestamp, pakai yang terbaru.
- ✅ JSON blobs di-normalisasi ke tabel `user_strategy_configs` dengan kolom individual.

---

## Kategori 7: Cash / Portfolio Value

### Sumber-sumber

| # | Sumber | Lokasi | Detail |
|---|--------|--------|--------|
| A | D1 | `users.cash` | Satu kolom number |
| B | D1 | `portfolios` table | Per-ticker position dengan buy_price, shares |
| C | D1 | `engine_state` JSON blob | Field `cash` + `portfolio[]` di dalam JSON |

### Masalah

- **Triple redundancy.** Cash ada di 3 tempat berbeda dalam database yang sama. Mana yang benar?
- Portfolio positions juga duplikat: `portfolios` (B) vs `engine_state.portfolio` (C).
- Kalau API update `users.cash` (A) tapi lupa update `engine_state` (C) — inkonsistensi.
- Kalau user execute trade lewat tool call AI, AI update `engine_state` (C) tapi mungkin tidak update `users.cash` (A).

### Rekomendasi v2

- ✅ Hapus `users.cash` — cash dihitung dari summary portfolio + cash position di `portfolios`.
- ✅ Hapus `engine_state` — seluruh state dipecah ke tabel-tabel normal.
- ✅ Portfolio value di-compute di query time: `SUM(portfolios.cash) + SUM(portfolios.shares * current_price)`.

---

## Kategori 8: Profile Weights (AMAN / AGRESIF / DIVIDEN)

### Sumber-sumber

| # | Sumber | Lokasi | Detail |
|---|--------|--------|--------|
| A | React | `EngineConfigContext.tsx` — `DEFAULT_PROFILES` | Pre-loaded |
| B | API | `functions/api/[[path]].ts:457-459` — `defaultWeights` | Untuk backtest-data endpoint |
| C | Engine | `marketRegimeEngine.ts:326-333` — `CW_AMAN` | Untuk regime-based weight switching |

### Masalah

- **Triple maintenance.** Ganti bobot AMAN dari Q=0.30 ke Q=0.35? Harus update 3 file berbeda.
- Nilai di B bisa berbeda dari A — karena API di-deploy secara independen dari frontend build.
- Nilai di C adalah referensi untuk "regime-aware weights" — tapi logika switching-nya tidak konsisten dengan A.

### Rekomendasi v2

- ✅ Profile weights disimpan di database: `profiles(id, name, weight_quality, weight_growth, weight_value, weight_momentum, weight_dividend)`.
- ✅ Frontend dan backend baca dari endpoint yang sama: `GET /api/profiles`.
- ✅ Hanya ada 1 sumber kebenaran.

---

## Kategori 9: Ticker List IDX80

### Sumber-sumber

| # | Sumber | Lokasi | Detail |
|---|--------|--------|--------|
| A | API | `functions/api/[[path]].ts:1101-1114` | 78 ticker hardcoded di `runIdx80Scan()` |
| B | Constants | `src/constants/idx80.ts` | 80-83 ticker |

### Masalah

- A dan B bisa berbeda. IDX80 composition berubah setiap 6 bulan. Kalau update constants (B) tapi lupa update API (A) — force-sync me-scan ticker berbeda dari frontend.
- Ticker A adalah subset dari B — `runIdx80Scan()` punya 78 ticker, constants punya 80-83.

### Rekomendasi v2

- ✅ Ticker list disimpan di database: `tickers(ticker, name, sector, industry, is_active, is_idx80)`.
- ✅ Admin endpoint untuk update komposisi IDX80 tanpa deploy.
- ✅ API dan frontend baca dari DB yang sama.

---

## Kategori 10: AI Chat History

### Sumber-sumber

| # | Sumber | Lokasi | Detail |
|---|--------|--------|--------|
| A | D1 | `ai_sessions` + `ai_messages` | Persistent di production |
| B | localStorage | `quantbit_chat_history` | Cached di browser |

### Masalah

- **Dua sumber yang tidak pernah di-sync.** Chat di D1 (A) tidak muncul di localStorage (B). Chat di browser (B) hilang kalau clear cache.
- AI context dibangun dari D1 (A) untuk production — tapi localStorage (B) kadang dipakai untuk dev.
- Kapasitas terbatas: localStorage maks ~5MB, D1 bisa lebih besar tapi kena cold start.

### Rekomendasi v2

- ✅ D1 sebagai SOT untuk chat history. localStorage hanya cache read-only.
- ✅ Mekanisme lazy load: ambil dari D1, simpan ke localStorage sebagai cache, next load pake localStorage dulu lalu sync dengan D1.

---

## Ringkasan

### Matriks Duplikasi

| Kategori | Jumlah Sumber | Lokasi Berbeda | Tingkat Keparahan |
|----------|:-------------:|:--------------:|:-----------------:|
| 1. Market data | 4 | JSON, SQLite, D1, Singleton | KRITIS |
| 2. Norm scores | 4 | JSON, TS static, D1, Static leader | KRITIS |
| 3. Rank data | 3 | D1, JSON, Runtime | TINGGI |
| 4. Dividend | 2 | JSON master, JSON copy | RENDAH |
| 5. IDX80 scan | 3 | JSON, D1 blob, Live JSON | SEDANG |
| 6. User config | 3 | localStorage, D1 blob, Draft | SEDANG |
| 7. Cash/value | 3 | users.cash, portfolios, engine_state | TINGGI |
| 8. Profile weights | 3 | React, API, Engine | SEDANG |
| 9. Ticker list | 2 | API hardcode, Constants | RENDAH |
| 10. AI chat history | 2 | D1, localStorage | RENDAH |

### Dampak

| Dampak | Contoh |
|--------|--------|
| **Salah perhitungan** | Backtest pakai JSON score 82, engine runtime pakai D1 score 76 → final score beda |
| **Salah decision** | Engine pake data Yahoo langsung (tidak termasuk carried forward) → beli/jual di harga berbeda |
| **User data loss** | Config di localStorage ga sync ke D1 → ganti device, config hilang |
| **Maintenance gila** | Ganti 1 number → update 3 file → 1 lupa → bug produksi |
| **Debug mimisan** | Tanya "kenapa hasil backtest beda dari produksi?" — karena data sumbernya beda |

### Root Cause

1. **Tidak ada arsitek data di awal** — tiap dev solve problem sendiri
2. **Campuran batch & real-time** — JSON buat backtest (batch) vs D1 buat API (real-time) vs singleton (real-time)
3. **Workflow tidak terintegrasi** — pipeline data jalan terpisah dari app lifecycle
4. **SQLite di dev, D1 di prod** — dua DB yang harus selalu identik
5. **Static React imports** — paksa data di-bundle, padahal bisa di-fetch dari API

### Solusi v2

Sudah di-cover di `14-rebuild-architecture.md` dan `16-rebuild-roadmap.md`:

1. **D1 = satu-satunya SOT** — tidak ada JSON, SQLite, atau singleton in-memory
2. **API Gateway** — semua komponen baca via Hono API endpoints, bukan langsung ke file/DB
3. **Normalized schema** — tidak ada JSON blobs, setiap entity punya tabel sendiri dengan FK
4. **Seed/deploy otomatis** — D1 di-seed via migration pipeline, bukan split antara pipeline dan app
5. **Cache layer** — Redis/Workers KV untuk performance, D1 tetap SOT

---

> **Dokumen ini adalah bagian dari seri audit reverse engineering Quantbit v1.**
> Jangan copy-paste kode. Gunakan knowledge ini untuk membangun v2 dengan arsitektur yang benar.
