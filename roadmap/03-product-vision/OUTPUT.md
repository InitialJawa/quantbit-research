# 🎯 FASE 3: Product Vision — QuantBit V2

> **Status**: ✅ COMPLETED
> **Date**: 2024-12-03
> **Based on**: Market Research (1000+ data point)

---

## 📖 Vision Statement

> **"QuantBit V2 adalah platform IDX portfolio management yang menggabungkan auto-import dari multiple broker, solid backtest engine, dan AI assistant untuk Q&A & explanation — dengan UI yang familiar untuk pemula dan terminal aesthetic untuk power user."**

---

## 🎯 High-Level Goals

### Vision 1-3 Years

| Timeframe | Goal | Metric |
|-----------|------|--------|
| **1 Year (MVP)** | Launch minimum viable product dengan fitur core | - 1K+ registered users<br>- 100+ active users<br>- 10+ user reviews |
| **2 Years (Growth)** | Scale ke 10K+ active users dengan multi-feature | - 10K+ active users<br>- 100K+ registered users<br>- 5+ broker integration |
| **3 Years (Dominance)** | Market leader di IDX portfolio management | - 100K+ active users<br>- 1M+ registered users<br>- 10+ broker integration<br>- 100+ user reviews (4.5+ rating) |

---

## 🎯 OKRs (Objectives & Key Results)

### Objective 1: Launch MVP dengan fitur core yang solid
**Timeline**: 3 bulan

| Key Result | Q1 | Q2 | Q3 |
|------------|----|----|----|
| Registered users | 0 | 500 | 1K+ |
| Active users | 0 | 50 | 100+ |
| Feature completion | 0% | 50% | 100% |
| User satisfaction | N/A | N/A | 4.0+ rating |

### Objective 2: Build solid backtest engine dengan golden test validation
**Timeline**: 3 bulan

| Key Result | Q1 | Q2 | Q3 |
|------------|----|----|----|
| Backtest accuracy | N/A | N/A | 95%+ match with v1 |
| Golden test coverage | N/A | N/A | 100% |
| Backtest speed | N/A | N/A | < 500ms for 1000 rows |

### Objective 3: Implement AI assistant untuk Q&A & explanation
**Timeline**: 2 bulan

| Key Result | Q1 | Q2 | Q3 |
|------------|----|----|----|
| AI accuracy | N/A | N/A | 90%+ |
| Response time | N/A | N/A | < 5s |
| User satisfaction | N/A | N/A | 4.5+ rating |

### Objective 4: Achieve 10K+ active users dalam 2 tahun
**Timeline**: 24 bulan

| Key Result | Month 6 | Month 12 | Month 18 | Month 24 |
|------------|---------|----------|----------|----------|
| Active users | 100 | 500 | 2K | 10K+ |
| Monthly growth | 10% | 15% | 20% | 25% |
| Retention rate | N/A | N/A | N/A | 70%+ |

---

## 🎯 Product Pillars

### Pillar 1: Simplicity
**Mission**: Buat produk yang mudah digunakan, bukan rumit.

**Guiding Principles**:
- **37 fitur MVP** — Bukan 50+
- **UI familiar** — Kayak Excel untuk pemula
- **Simple workflow** — Auto-import + manual hybrid
- **Clear onboarding** — 5 menit setup

### Pillar 2: Accuracy
**Mission**: Data & engine yang akurat, bukan asal-asalan.

**Guiding Principles**:
- **D1 single source of truth** — Tidak ada fragmented data
- **Golden test validation** — Backtest match dengan v1
- **Real-time data** — Update otomatis dari broker
- **Transparency** — DataStatus badge (LIVE/CACHED/STALE)

### Pillar 3: Intelligence
**Mission**: AI yang membantu, bukan yang mengambil alih.

**Guiding Principles**:
- **AI untuk Q&A & explanation** — Bukan auto-decision
- **4-level trust model** — L1 Q&A, L2 read-only, L3 action (user approval), L4 proactive (post-MVP)
- **Structured output** — JSON mode, bukan regex
- **Human-in-the-loop** — User approval untuk action tools

### Pillar 4: Performance
**Mission**: Cepat, scalable, reliable.

**Guiding Principles**:
- **Edge-native** — Cloudflare Pages + D1 + KV
- **Fast response** — < 2s initial load, < 5s AI response
- **Scalable** — Handle 100K+ users
- **Reliable** — 99.9% uptime

---

## 🎯 User Experience Vision

### Onboarding (5 menit)
1. **Sign up** — Email + password (Better Auth)
2. **Setup broker** — Pilih broker (Rekeningku, Bibit, Ajaib, dll)
3. **Import data** — Auto-import atau manual
4. **Choose profile** — AMAN, AGRESIF, DIVIDEN
5. **Done** — Dashboard ready

### Daily Workflow (3 langkah)
1. **Check dashboard** — Profit/loss, BPS, regime
2. **Review AI insights** — Q&A atau explanation
3. **Take action** — Buy/sell (user approval) atau rebalance

### Power User Workflow (Terminal aesthetic)
1. **Keyboard shortcuts** — Quick navigation
2. **Custom views** — Personalize dashboard
3. **Advanced backtest** — Multi-strategy comparison
4. **Deep analysis** — AI untuk deep dive

---

## 🎯 Technical Vision

### Architecture
```
┌─────────────────────────────────────────┐
│         React 19 + Vite 6               │
│         (Frontend, Client-side)         │
└──────────────┬──────────────────────────┘
               │
               ▼
┌─────────────────────────────────────────┐
│         Hono on Cloudflare Pages        │
│         (Backend API, Server-side)      │
└──────────────┬──────────────────────────┘
               │
               ▼
┌─────────────────────────────────────────┐
│         Cloudflare D1                   │
│         (Single Source of Truth)        │
└──────────────┬──────────────────────────┘
               │
               ▼
┌─────────────────────────────────────────┐
│         Workers KV (Cache)              │
└─────────────────────────────────────────┘
```

### Data Flow
```
External Sources (Yahoo, GoAPI, AI)
    │
    ▼
Data Pipeline (Cron)
    │
    ▼
Cloudflare D1 (Single Source of Truth)
    │
    ├───────▶ Hono API / KV Cache / Workers
    │
    └───────▶ React Frontend
```

### AI Architecture
```
User Question
    │
    ▼
OpenRouter (JSON mode / structured output)
    │
    ├───────▶ L1 Q&A (text only)
    ├───────▶ L2 Read-only tools (auto-executed)
    ├───────▶ L3 Action tools (user approval)
    └───────▶ L4 Proactive agent (post-MVP)
```

---

## 🎯 Differentiation

### vs Excel
| Feature | Excel | QuantBit V2 |
|---------|-------|-------------|
| Auto-update | ❌ Manual | ✅ Auto |
| Real-time | ❌ No | ✅ Yes |
| AI assistant | ❌ No | ✅ Yes |
| Backtest | ❌ Manual | ✅ Solid |
| Multi-broker | ❌ No | ✅ Yes |
| AI Q&A | ❌ No | ✅ Yes |

### vs Competitor Apps
| Feature | Stockbit | Ajaib | Bibit | QuantBit V2 |
|---------|----------|-------|-------|-------------|
| Auto-import | ❌ | ✅ (1 broker) | ✅ (1 broker) | ✅ (multiple) |
| Backtest | ❌ | ❌ | ❌ | ✅ Solid |
| AI Q&A | ❌ | ⚠️ Recommendation | ⚠️ Salesy | ✅ Q&A & explanation |
| Multi-broker | ❌ | ❌ | ❌ | ✅ Yes |
| Terminal UI | ❌ | ❌ | ❌ | ✅ Yes (power user) |

---

## 🎯 Success Metrics

### Product Metrics
- **MAU (Monthly Active Users)**: 10K+ by year 2
- **Retention Rate**: 70%+ by year 2
- **NPS**: 50+ by year 2
- **Revenue**: Break-even by year 2, profit by year 3

### Technical Metrics
- **Uptime**: 99.9%
- **Response Time**: < 2s (frontend), < 5s (AI)
- **Backtest Accuracy**: 95%+ match with v1
- **AI Accuracy**: 90%+

### Business Metrics
- **CAC**: < Rp 50K
- **LTV**: > Rp 500K
- **LTV/CAC**: > 10x
- **Churn Rate**: < 10% monthly

---

## 🎯 Risks & Mitigation

### Risk 1: User Skeptis AI Finance
**Mitigation**:
- AI untuk Q&A & explanation, bukan auto-decision
- Transparency: explain decisions step-by-step
- User approval untuk semua action tools

### Risk 2: Data Source Changes
**Mitigation**:
- Multiple data sources (Yahoo + GoAPI + AI)
- Fallback mechanism
- Cache strategy (Workers KV)

### Risk 3: Backtest Accuracy
**Mitigation**:
- Golden test validation
- Match with v1 outputs
- Continuous testing

### Risk 4: Competition
**Mitigation**:
- Unique features (multi-broker, solid backtest, AI Q&A)
- Strong UX (simple for pemula, powerful for power user)
- Community building

---

## ✅ Next Steps

### ➡️ FASE 4: PRD (Product Requirement Document)
**Goal**: Dokumentasi requirement produk secara detail — fitur, user persona, use case

**Output**: `roadmap/04-prd/OUTPUT.md`

**Pertanyaan ke Kamu**:
- Fitur apa yang benar-benar dibutuhkan user pertama?
- Ingat riset Fase 2 — user mau simple (37 fitur), bukan 50+
- Prioritas: Must/Should/Nice

**Lanjut ke FASE 4? Ada yang disesuaikan?**

