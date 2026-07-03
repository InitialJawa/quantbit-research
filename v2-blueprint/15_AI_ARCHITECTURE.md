# 15 — AI Architecture

> AI system architecture for QuantBit V2, based on v1's 4-level system with improved structured output.

---

## Architecture Overview

```
User Message
  → System Prompt (context: portfolio, market, regime, config, history)
  → AI Provider (OpenRouter JSON mode)
  → Structured Response
    → Text response for UI display
    → Tool calls (typed, no regex)
  → L1: Display text only
  → L2: Execute read-only tools (auto)
  → L3: Execute action tools (require [Approve])
```

---

## Key Changes from V1

| Aspect | V1 | V2 |
|--------|----|----|
| Tool call extraction | Regex from markdown | JSON mode / tool calling API |
| Persona | Hardcoded "Rico Lubis" | Configurable + default clean persona |
| Provider priority | 7 providers with circuit breaker | 1 primary (OpenRouter) + 2 fallback |
| Memory | 10K char prompt injection | Structured session/message DB |
| System prompt | 14-section document in code | Modular prompt construction |
| Proactive agent | 6 BPS rules (not validated) | Same concept but testable |

---

## 4-Level Architecture (Preserved from V1)

### Level 1: Q&A
- User asks questions about portfolio, market, strategies
- AI responds with text only (no tools)
- No approval needed

### Level 2: Read-Only Tools
- AI can execute: `getMarketOverview`, `getStockInfo`, `searchStocks`, `getTopMovers`, `getHistoricalData`, `getPortfolio`, `getBPSScore`, `getRegimeState`
- All tools execute automatically
- Results shown in chat as formatted data

### Level 3: Action Tools
- AI can propose: `buyStock`, `sellStock`, `buyGold`, `sellGold`, `rebalancePortfolio`, `updateStrategy`, `runBacktest`, `syncBacktestToPortfolio`
- Each action requires user [Approve] before execution
- Approval card shows: action type, ticker, quantity, price, estimated cost/proceeds
- User can reject with feedback

### Level 4: Proactive Agent (Deferred to Post-MVP)
- System monitors BPS transitions, config changes, market regime changes
- Proactively sends notifications via AI chat
- 5-minute cooldown per rule type to prevent spam
- Not in MVP — implement after core AI chat is stable

---

## Provider Configuration

### Primary (MVP)
```typescript
const providers = [
  {
    name: "openrouter",
    model: "meta-llama/llama-4-scout-128b-instruct",  // Free tier
    supportsJSON: true,
    supportsToolCalls: true,
  },
  {
    name: "openrouter-fallback", 
    model: "openai/gpt-4o-mini",  // Paid but reliable
    supportsJSON: true,
    supportsToolCalls: true,
  },
];
```

**Strategy:**
- Start with free tier (Llama 4 Scout via OpenRouter)
- Fallback to paid tier (GPT-4o-mini) on free rate limits
- Add Groq/Gemini as experimental options post-MVP
- Circuit breaker: 429 → 5min cooldown, 401/403 → 15min cooldown

---

## System Prompt (Modular)

```typescript
// System prompt is built from modules, not a single document

interface SystemPromptModule {
  name: string;
  content: string;
  isActive: (context: AIContext) => boolean;
}

const modules: SystemPromptModule[] = [
  personaModule,        // "You are a helpful financial analyst..."
  capabilitiesModule,   // What the AI can do
  toolCatalogModule,    // Available tools with schemas
  contextModule,        // Current portfolio + market data
  historyModule,        // Past chat messages (truncated)
  rulesModule,          // Business rules (regime > BPS, etc.)
];
```

**Benefits over v1's monolithic prompt:**
- Each module is testable independently
- Modules can be toggled based on context
- Cleaner prompt construction
- Easier to add/remove capabilities

---

## Structured Output Schema

```typescript
// V2 uses JSON mode — no regex parsing

interface AIResponse {
  type: "qa" | "tool_call" | "action_proposal";
  content: string;  // Display text
  confidence: "high" | "medium" | "low";
  
  // Only for tool_call type
  toolCalls?: Array<{
    tool: "getMarketOverview" | "getStockInfo" | ...;
    params: Record<string, unknown>;
    id: string;  // For result correlation
  }>;
  
  // Only for action_proposal type  
  action?: {
    type: "buyStock" | "sellStock" | ...;
    params: {
      ticker: string;
      shares?: number;
      price?: number;
    };
    reasoning: string;  // Why this action
    estimatedImpact: string;  // Expected outcome
  };
}
```

**Compare with V1:** V1 required `extractToolCalls()` function with 3 regex patterns. V2 schema is typed, validated via Zod, and doesn't need regex.

---

## Memory Management

```typescript
interface Session {
  id: string;
  userId: string;
  createdAt: string;
  updatedAt: string;
  context: {
    lastTopic: string;
    mentionedTickers: string[];
    pendingActions: number;
  };
}

interface Message {
  id: string;
  sessionId: string;
  role: "user" | "assistant" | "system";
  content: string;
  toolCalls?: AIToolCall[];
  toolResults?: AIToolResult[];
  createdAt: string;
}
```

**Memory Strategy:**
- Last 10K characters from current session (same as v1)
- Inject up to 3 most recent relevant past sessions
- Truncate oldest messages first
- Store ALL messages in D1 (not just current session)

---

## Tool Schema (V2)

```typescript
const TOOLS = {
  // Read-only (L2 — auto-execute)
  getMarketOverview: {
    description: "Get current market conditions (IHSG, gold, USD/IDR, regime)",
    parameters: {},
  },
  getStockInfo: {
    description: "Get detailed info for a ticker",
    parameters: { ticker: "string" },
  },
  searchStocks: {
    description: "Search stocks by name or ticker",
    parameters: { query: "string" },
  },
  getTopMovers: {
    description: "Get top gainers/losers for today",
    parameters: { limit: "number (optional)" },
  },
  getHistoricalData: {
    description: "Get historical prices for a ticker",
    parameters: { ticker: "string", days: "number" },
  },
  getPortfolio: {
    description: "Get current portfolio holdings",
    parameters: {},
  },
  getBPSScore: {
    description: "Get current Buy Pressure Score",
    parameters: {},
  },
  getRegimeState: {
    description: "Get current market regime and risk levels",
    parameters: {},
  },
  
  // Action (L3 — require [Approve])
  buyStock: {
    description: "Buy shares of a stock",
    parameters: { ticker: "string", shares: "number" },
    requiresApproval: true,
  },
  sellStock: {
    description: "Sell shares of a stock",
    parameters: { ticker: "string", shares: "number" },
    requiresApproval: true,
  },
  buyGold: {
    description: "Buy gold with available cash",
    parameters: { amount: "number" },
    requiresApproval: true,
  },
  sellGold: {
    description: "Sell gold grams",
    parameters: { grams: "number" },
    requiresApproval: true,
  },
  runBacktest: {
    description: "Run a backtest with current or custom config",
    parameters: { config: "BacktestConfig (optional)" },
    requiresApproval: true,
  },
  updateStrategy: {
    description: "Update strategy settings",
    parameters: { profile: "string", topN: "number" },
    requiresApproval: true,
  },
};
```

---

## Business Rules for AI (Preserved from V1)

- **Regime > BPS**: "REGIME MENANG. Macro > micro." When BPS says deploy but regime is CASH_DEFENSE, regime wins.
- **No auto-execute**: All L3 actions must pass through user [Approve]
- **Response language**: Indonesian/mixed (for Indonesian users), or English (configurable)
- **Response format**: Overview → data table → bullet list → action suggestion
- **Data-driven**: "Pake data, bukan feeling." Every claim must reference current data.
