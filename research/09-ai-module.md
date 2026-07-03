# Audit Modul AI — Quantbit

## AI Flow

```
User message → askAI() → buildLiveContext() → POST /api/ai/chat →
  runAiChat() → buildSystemPrompt() → buildProviderList() →
  tryProvider() in priority order → response → extractToolCalls() →
  cleanText + toolCalls → UI renders
```

## System Prompt Architecture (`src/ai/systemKnowledge.ts`)

**4-layer composition** (`buildSystemPrompt()`):

1. **Behavior** (lines 180-204): Jaksel persona — casual, data-driven, Indo-English mix. "Bukan robot kaku, tapi juga bukan preman. Think Jaksel: chill, paham pasar, ngomong pake data."
2. **System Knowledge** (lines 115-177): 14 sections — scoring (5-factor), market metrics (SMA20/50, drawdown60, breadth, exit risk), regime decision tree (GOLD_DEFENSE→CASH_DEFENSE→RISK_OFF→RECOVERY_WATCH→RISK_ON), regime scores (0-99), settings knobs, rebalancing, BPS/adaptive DCA, tool catalog, proactive rules.
3. **Live Context** (lines 210-330): config, regime, market stats, portfolio state, BPS score, strategy evaluation — injected as formatted sections.
4. **Style Reminder** (lines 330+): table format, bold key numbers, no emoji, 2-3 sentence overview.

## Provider Chain (`src/server/aiChatHandler.ts:468-584`)

Priority order (lower = higher):

| Priority | Name | Default Model | API |
|---|---|---|---|
| 1 | openrouter | `openai/gpt-oss-120b:free` | OpenRouter |
| 2 | openrouter-2 | `nvidia/nemotron-3-super-120b-a12b:free` | OpenRouter |
| 3 | openrouter-3 | `cohere/north-mini-code:free` | OpenRouter |
| 4 | openrouter-4 | `meta-llama/llama-3.3-70b-instruct:free` | OpenRouter |
| 5 | cohere | `command-a-plus-05-2026` | Cohere |
| 6 | mistral | `mistral-small-latest` | Mistral |
| 7 | groq | `groq/compound` | Groq |
| 8 | groq-fallback | `llama-3.3-70b-versatile` | Groq |
| 9 | gemini | `gemma-4-26b-a4b-it` | Google |

**DEFAULTS object** (lines 239-249). All providers use direct `fetch()` — no SDKs. OpenRouter models share one API key but hit 4 different provider pools. Cohere/Mistral use OpenAI-compatible `/chat/completions` endpoints. Gemini uses its own REST API with fallback model (`gemma-4-31b-it`).

**tryProvider()** (lines 589-640): Iterates providers ascending by priority. Catches errors, classifies (rate_limit/quota/auth/other), sets circuit breaker cooldown, logs per-provider error. Continues to next provider unless auth error.

## Circuit Breaker (`src/server/aiChatHandler.ts:168-234`)

- **429** → cooldown `COOLDOWN_429_MS` (default 5 min)
- **401/403** → cooldown `COOLDOWN_403_MS` (default 15 min)
- Cooldown via `cooldownState` (in-memory `Map<string, number>`)
- Auto-cleanup via `setTimeout` + `.unref()`
- `getAllCooldowns()` exposes status endpoint data
- `classifyError()` regex-matches `\b\d{3}\b` from error strings — fragile

## Tool System (`src/ai/toolCallParser.ts:45-118`)

**Regex-based extraction:**
- Marker regex: `\{\s*"(?:tool_call|function)"\s*:\s*`
- `findMatchingBrace()`: brace-counting with depth tracking, string-literal awareness
- Fallback regex clean: strips remaining `{tool_call:...}` blocks from text
- 8 **read-only** tools (auto-execute): `get_portfolio_state`, `get_bps_now`, `get_regime_details`, `get_ticker_metrics`, `get_market_history`, `get_backtest_config`, `get_engine_config`, `get_active_universe`
- 10 **action** tools (require user [Approve]): `buy_stock`, `sell_stock`, `move_to_gold`, `set_active_profile`, `set_universe`, `set_topN`, `toggle_dca_active`, `add_to_watchlist`, `remove_from_watchlist`, `sync_backtest_to_portfolio`

## Memory System (`src/server/aiMemory.ts`, `src/server/aiChatHandler.ts:36-85`)

- **D1-backed**: `ai_sessions` + `ai_messages` tables
- `MEMORY_MAX_CHARS = 10_000` (20 messages × ~500 chars budget)
- `getRecentMemory()`: fetches last 20 messages from past sessions (excluding current), reversed to chronological
- `formatMemoryBlock()`: injected as **Section 15** of system prompt
- `truncateToMemoryBudget()`: drops oldest entries first when over budget
- `suggestTitle()`: auto-generates session title from first user message (60 char truncate)

## Weaknesses

### 1. Regex parsing (`toolCallParser.ts:45-92`) — CRITICAL
Fragile brace-counting parser fails on:
- Nested JSON with escaped quotes
- Malformed/wrapped-in-markdown tool calls
- Models that don't reliably emit JSON blocks
- Uses fallback regex as safety net (lines 88-89) which compounds fragility
- **Fix**: Switch to structured output (OpenRouter `response_format`, Mistral `tool_choice`, Groq constrained decoding)

### 2. Dev mock (`devMockAI.ts:245 lines`) — MEDIUM
Pattern-matched canned responses. 245 lines of mock code bundled in production. Functions like `mockChat()`, `isDevMode()` could accidentally match production queries.

### 3. Geo-blocking — HIGH
Source comment (line 20): "Direct API calls (Groq, Google) are geo-blocked from many Cloudflare edge regions." These are the last 3 fallback providers. Users served from blocked regions see degraded AI.

### 4. Provider sprawl — MEDIUM
9 models across 5 providers. Sequential fallback means worst-case latency = sum of 9 timeouts. Each provider tested in `tryProvider()` sequentially with no parallel racing. OpenRouter models 1-4 share same API key but hit different pools — adds complexity without guaranteed diversity.

### 5. No streaming — MEDIUM
`chatOpenAICompatible()` awaits full `resp.text()` before returning. Users wait for entire response. No progressive display.

### 6. Memory budget — LOW
10K char hard limit. Old messages dropped silently. Past session context truncated arbitrarily.

### 7. sanitizeMessages() — HIGH
`sanitizeMessages()` (line 346-355) strips `tool` role messages AND removes tool_call JSON from assistant messages — called by ALL OpenAI-compatible providers in `chatOpenAICompatible()`. This means tool call context is lost on re-prompt. Cohere/Mistral can't see prior tool usage.

### 8. Jaksel prompt — LOW
Casual Indonesian-English mix in system prompt may confuse non-Indonesian models or produce unexpected behavior with strict instruction-following models.

### 9. OpenRouter quota — LOW
50 free requests/day on free models. $10 for 1000 requests on paid tier. No quota tracking in app.
