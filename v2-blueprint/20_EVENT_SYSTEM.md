# 20 — Event System

> Event flows within QuantBit V2.

---

## Event Types

```typescript
enum SystemEvent {
  // Market Events
  MARKET_DATA_UPDATED,      // Daily sync completed
  MARKET_REGIME_CHANGED,    // Regime state transition
  CRASH_DETECTED,           // Crash condition triggered
  CRASH_RECOVERED,          // Recovery detected
  SCORES_UPDATED,           // Norm scores recomputed
  
  // Portfolio Events
  TRADE_EXECUTED,           // Buy/sell completed
  PORTFOLIO_VALUATION_CHANGED,
  WATCHLIST_MODIFIED,
  CONFIG_UPDATED,           // Strategy settings changed
  
  // Backtest Events
  BACKTEST_STARTED,
  BACKTEST_COMPLETED,
  BACKTEST_FAILED,
  
  // System Events
  PIPELINE_RUN_STARTED,
  PIPELINE_RUN_COMPLETED,
  PIPELINE_RUN_FAILED,
  AI_CHAT_MESSAGE,
}
```

---

## Event Flow Diagram

```
┌─────────────────────────────────────────────────────────────────────┐
│                         EVENT SOURCES                                │
│                                                                      │
│  Daily Cron    User Action    Market Data     AI Chat    Backtest    │
│  (GitHub Act.) (UI/API)       Update         Response   Complete    │
│       │            │              │              │          │         │
│       ▼            ▼              ▼              ▼          ▼         │
│  ┌──────────────────────────────────────────────────────────┐       │
│  │                  EVENT DISPATCHER                          │       │
│  │  - Fire-and-forget (no guaranteed delivery)               │       │
│  │  - Best-effort delivery within same worker                │       │
│  │  - No persistent event log (MVP)                          │       │
│  └──────────┬───────────────────────────────────────┬────────┘       │
│             │                                       │                 │
│             ▼                                       ▼                 │
│  ┌────────────────────┐              ┌────────────────────────┐      │
│  │   UI UPDATES        │              │   SIDE EFFECTS          │      │
│  │  - Refresh market   │              │  - Trigger notification │      │
│  │  - Update dashboard │              │  - Invalidate cache     │      │
│  │  - Show notification│              │  - Log to console       │      │
│  └────────────────────┘              └────────────────────────┘      │
└─────────────────────────────────────────────────────────────────────┘
```

---

## V1 → V2 Changes

**V1 had no event system.** State changes were handled by:
- React re-renders (local state changes)
- Manual refresh buttons
- Polling (60s for market data)

**V2 adds minimal event system** for:
- Cache invalidation (KV cache cleared on data update)
- UI refresh (components re-fetch when data changes)
- Notification triggers (regime change, crash detected)
- Backtest completion notification

---

## Event Flows

### Flow 1: Daily Data Pipeline

```
GitHub Actions Cron
  → Pipeline Script (TS/Node)
    → Fetches Yahoo EOD data
    → Upserts D1: market_daily, stock_daily
    → Computes scores (stock_scores)
    → Updates idx80_scans
    → Triggers: PIPELINE_RUN_COMPLETED
      → Invalidates KV cache (market/regime/tickers)
      → Updates health status
```

### Flow 2: User Trade

```
User clicks "Beli" in UI
  → POST /api/v1/portfolio/trade
    → Validates trade (Zod)
    → Checks sufficient funds (D1 query)
    → Executes trade (D1 write: portfolios, cash_holdings, trade_logs)
    → Triggers: TRADE_EXECUTED
      → Returns updated portfolio
    → Triggers: PORTFOLIO_VALUATION_CHANGED
      → Re-fetch portfolio components
```

### Flow 3: Backtest Completion

```
User triggers backtest
  → POST /api/v1/engine/run-backtest
    → INSERT backtest_sessions (status: "running")
    → Returns sessionId immediately
  
Background (same request):
  → Query D1 for market data in date range
  → Run engine (pure TS)
  → Store results (UPDATE backtest_sessions, INSERT backtest_logs)
  → Triggers: BACKTEST_COMPLETED
    → (Polling from UI: GET /engine/backtest/:id)
```

### Flow 4: Market Regime Change

```
User views Portfolio tab
  → GET /api/v1/market/regime
    → Computes regime from D1 market_daily data
    → Returns current regime state
    → (Regime is computed on-read, not pre-computed)

No persistent event in MVP. Regime state changes only visible on next API call.
```

---

## Event Subscriptions (Frontend)

```typescript
// Simple observer pattern — no Pub/Sub, no WebSocket

class EventBus {
  private listeners = new Map<SystemEvent, Set<() => void>>();
  
  on(event: SystemEvent, callback: () => void) {
    if (!this.listeners.has(event)) {
      this.listeners.set(event, new Set());
    }
    this.listeners.get(event)!.add(callback);
    return () => this.listeners.get(event)?.delete(callback);
  }
  
  emit(event: SystemEvent) {
    this.listeners.get(event)?.forEach(cb => cb());
  }
}

// Example usage:
eventBus.on(SystemEvent.TRADE_EXECUTED, () => {
  refetchPortfolio();
  refetchBPS();
});
```

---

## Cache Invalidation Strategy

| Event | Invalidates |
|-------|-------------|
| MARKET_DATA_UPDATED | `market/overview`, `market/stocks`, `market/regime` |
| SCORES_UPDATED | `market/fundamentals/*`, `market/backtest-data` |
| TRADE_EXECUTED | `portfolio` (all endpoints) |
| CONFIG_UPDATED | `engine/config` |
| BACKTEST_COMPLETED | `engine/backtest/:id` |

---

## Out of Scope (MVP)

The following event features are NOT in V2 MVP:
- **Persistent event log** — Events are not stored
- **WebSocket/SSE** — No real-time push
- **Eventual consistency guarantees** — No event sourcing
- **Cross-worker event propagation** — Events only within same request
- **Delayed/deferred event processing** — No event queues

**When to add:** Post-MVP, if polling-based refresh becomes too slow for users.
