# 21 — UI Flow

> Complete UI flow for QuantBit V2, including tab navigation, user workflows, and state transitions.

---

## Tab Structure

```
┌──────────────────────────────────────────────────────────────────────┐
│  [1] Market  [2] Portfolio  [3] Backtest  [4] Analytics  [5] Admin  │
│            ──── keyboard shortcuts (1/2/3/4/5) ────                   │
└──────────────────────────────────────────────────────────────────────┘
```

**Change from V1:** Added Admin tab (5). All other tabs map 1:1.

---

## Flow 1: Daily User Session

```
1. Open app → Login screen (if no session)
   → Enter email/password or Register
   → POST /auth/login → session token → localStorage

2. Main App loads:
   → GET /portfolio → holdings, cash, gold, BPS
   → GET /market/overview → IHSG, gold, USD/IDR
   → GET /market/regime → current regime state
   → GET /engine/config → strategy settings
   → DataStatus badges shown for all data

3. Default tab: Portfolio (shows your positions + BPS)
   → Holdings table: ticker, shares, avg price, current price, gain/loss
   → BPS dashboard: gauge + factor breakdown
   → Portfolio chart: value over time (reconstructed from trade log)

4. User explores Market tab:
   → Stock list with search/filter
   → Stock scores and ranks
   → Click ticker → StockDrawer opens
     → Price chart, fundamentals, buy/sell panel

5. User adjusts strategy:
   → AppSidebar → StrategySettingsPanel
   → Change profile, top N, crash sensitivity
   → PATCH /engine/config → saved to D1

6. User executes AI chat:
   → FloatingAIChat button (bottom-right)
   → Type question: "How is my portfolio doing?"
   → AI responds with portfolio summary, BPS, recommendations
   → If AI proposes action → [Approve] card appears
   → User approves/rejects

7. Logout (end of session)
```

---

## Flow 2: Backtest Workflow

```
1. User navigates to Backtest tab (3)
   → Shows current config (from Live strategy)
   → [Live mode = ON] → Fields disabled, tooltip: "Locked to Portfolio"
   → [Live mode = OFF] → Fields editable (Draft mode)

2. User adjusts backtest config:
   → Change profile, date range, top N
   → OR toggle to Live mode and go change Portfolio settings

3. User clicks "Jalankan Backtest"
   → POST /engine/run-backtest
   → Loading state: "Computing... (est. 5 seconds)"
   → Polling: GET /engine/backtest/:id every 2 seconds
   → Results appear:
     → Summary cards: CAGR, Sharpe, Max DD, Final Value
     → Chart: Portfolio value vs 60/40 benchmark
     → Breakdown: Cash vs Gold vs Stocks over time
     → Log: Rebalance events, crash deployments, dividend credits

4. User evaluates results
   → Can save backtest session (auto-saved)
   → Can compare with previous sessions
   → Click "SYNC TO PORTFOLIO" (Live mode only or Promote from Draft)
     → PATCH /engine/config → update with backtest config
     → GET /portfolio → refresh with new strategy
```

---

## Flow 3: Trade Execution

```
1. From Market tab:
   → Find stock → Click → StockDrawer opens
   → See current price, score, fundamentals
   → Enter shares (must be multiple of 100)
   → Click "Beli" → POST /portfolio/trade
   → Success notification → Portfolio tab shows updated holdings

2. From Portfolio tab:
   → Holdings table → Click ticker
   → See current position, P&L
   → Enter shares to sell → Click "Jual"
   → Or sell all → "Jual Semua"

3. From AI Chat:
   → "Beli BBCA 5 lot"
   → AI proposes action: buyStock { ticker: "BBCA", shares: 500 }
   → [Approve] card shows: cost, current price, estimated fee
   → User approves → POST /portfolio/trade
   → Result shown in chat
```

---

## Flow 4: First-Time User Onboarding

```
1. Register → enter email + password
   → Login → empty portfolio

2. Empty state:
   → Portfolio tab: "Belum ada portofolio.
      Jalankan backtest untuk memulai atau tambah saham secara manual."
   → Backtest tab: Enable Draft mode, configure strategy
   → Run backtest → see results with default settings
   → Click "SYNC TO PORTFOLIO"
   → Now Portfolio tab shows holdings (from backtest)

3. OR: Manual portfolio creation
   → Market tab → search stocks → StockDrawer → "Beli"
   → Add cash via DigitalWalletUI → "Tambah Saldo"
   → Buy stocks one by one
   → Watch BPS and regime for entry timing
```

---

## Flow 5: AI Chat Proactive (Post-MVP)

```
Proactive triggers (checked every 5 minutes minimum):
  → Market regime changes → RISK_ON → "Market membaik, timing untuk deploy"
  → BPS crosses threshold → "BPS naik ke 75, saatnya beli"
  → Portfolio valuation drops → "Portofolio turun 5% dalam seminggu"

Not in MVP. Implement after core AI chat is stable.
```

---

## UI Flow Diagram

```
                    ┌───────────────┐
                    │   Login/      │
                    │   Register    │
                    └───────┬───────┘
                            │
                    ┌───────▼───────┐
                    │   App Shell   │
                    │  (Main App)   │
                    └───────┬───────┘
                            │
          ┌─────────────────┼─────────────────┬──────────────┐
          │                 │                 │              │
    ┌─────▼─────┐   ┌──────▼──────┐  ┌───────▼──────┐  ┌────▼────┐
    │  Market   │   │  Portfolio  │  │   Backtest   │  │Analytics│
    │  Tab (1)  │   │  Tab (2)    │  │   Tab (3)    │  │ Tab (4) │
    └───────────┘   └─────────────┘  └──────────────┘  └─────────┘
                                                  │
                                            ┌─────▼──────┐
                                            │  Admin (5)  │
                                            └────────────┘
```

---

## Responsive Layout

| Viewport | Layout | Behavior |
|----------|--------|----------|
| Desktop (>1024px) | Sidebar + main content | Full terminal interface |
| Tablet (768-1024px) | Hamburger menu + main | Collapsed sidebar |
| Mobile (<768px) | Bottom tab bar + main | Stacked panels, simplified tables |

---

## Error States (Every Component)

Every data-displaying component must handle:

1. **Loading**: Skeleton/spinner while data loads
2. **Error**: Error message + retry button
3. **Empty**: Descriptive empty state + call to action
4. **Stale data**: Show data + "Data may be stale" warning + DataBadge
5. **Offline**: Connection status indicator in header + retry on reconnect
