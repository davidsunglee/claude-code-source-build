# Cost Tracking

Claude Code tracks every token sent to and received from the Anthropic API, converts those counts to USD estimates using per-model pricing tables, and persists the running totals so they survive session resumes. At process exit the system prints a chalk-styled summary broken down by model.

This page covers the two files that own that responsibility, plus the pricing module they delegate to.

---

## Key Files

| File | Role |
|------|------|
| [`source/src/cost-tracker.ts`](../../source/src/cost-tracker.ts) | Central module. Re-exports state accessors, provides session persistence, formatting, and the `addToTotalSessionCost` accumulator. |
| [`source/src/costHook.ts`](../../source/src/costHook.ts) | React hook (`useCostSummary`) that wires cost reporting into the process `exit` event. |
| [`source/src/utils/modelCost.ts`](../../source/src/utils/modelCost.ts) | Per-model pricing tables and the `calculateUSDCost` function. |
| [`source/src/bootstrap/state.ts`](../../source/src/bootstrap/state.ts) | Owns the mutable singleton `STATE` object where all counters live. |
| [`source/src/entrypoints/sdk/coreSchemas.ts`](../../source/src/entrypoints/sdk/coreSchemas.ts) | Zod schema for `ModelUsage` (the per-model bucket shape). |

---

## Cost State

All cost data lives in the global `STATE` singleton defined in `bootstrap/state.ts`. The relevant fields are:

| State field | Type | Description |
|-------------|------|-------------|
| `totalCostUSD` | `number` | Running USD total across all models. |
| `totalAPIDuration` | `number` | Cumulative wall-clock time (ms) spent waiting on API responses. |
| `totalAPIDurationWithoutRetries` | `number` | Same as above, excluding retried requests. |
| `totalToolDuration` | `number` | Cumulative wall-clock time (ms) spent executing tools. |
| `totalLinesAdded` / `totalLinesRemoved` | `number` | Code-change counters (tracked alongside cost for the summary). |
| `hasUnknownModelCost` | `boolean` | Set to `true` when a model with no pricing entry is encountered. |
| `modelUsage` | `{ [modelName: string]: ModelUsage }` | Per-model breakdown of all token counts and cost. |

Each `ModelUsage` value contains:

```
inputTokens          — prompt tokens (non-cached)
outputTokens         — completion tokens
cacheReadInputTokens — tokens served from the prompt cache
cacheCreationInputTokens — tokens written into the prompt cache
webSearchRequests    — server-side web search requests
costUSD              — accumulated USD cost for this model
contextWindow        — model context window size
maxOutputTokens      — model max output token limit
```

Token totals (`getTotalInputTokens`, etc.) are computed on the fly by summing across all models in `modelUsage` via a `sumBy` helper.

---

## Exports

`cost-tracker.ts` re-exports a curated surface from `bootstrap/state.ts` plus its own functions. The full public API:

| Export | Source | Returns |
|--------|--------|---------|
| `getTotalCost()` | re-export of `getTotalCostUSD` from state | Cumulative USD cost (`number`) |
| `getTotalDuration()` | state | Wall-clock ms since session start (`Date.now() - startTime`) |
| `getTotalAPIDuration()` | state | Cumulative API wait ms |
| `getTotalAPIDurationWithoutRetries()` | state | API wait ms excluding retries |
| `getTotalInputTokens()` | state | Sum of `inputTokens` across all models |
| `getTotalOutputTokens()` | state | Sum of `outputTokens` across all models |
| `getTotalCacheReadInputTokens()` | state | Sum of `cacheReadInputTokens` across all models |
| `getTotalCacheCreationInputTokens()` | state | Sum of `cacheCreationInputTokens` across all models |
| `getTotalWebSearchRequests()` | state | Sum of `webSearchRequests` across all models |
| `getModelUsage()` | state | The full `{ [model]: ModelUsage }` map |
| `getUsageForModel(model)` | state | Single-model `ModelUsage` or `undefined` |
| `hasUnknownModelCost()` | state | `true` if any API call used an unrecognized model |
| `setHasUnknownModelCost()` | state | Marks the flag (called from `modelCost.ts`) |
| `resetCostState()` | state | Zeros all counters, resets `startTime`, clears `modelUsage` |
| `resetStateForTests()` | state | Full state reset (guarded by `NODE_ENV === 'test'`) |
| `addToTotalLinesChanged(added, removed)` | state | Increments code-change counters |
| `getTotalLinesAdded()` / `getTotalLinesRemoved()` | state | Read code-change counters |
| `formatCost(cost, maxDecimalPlaces?)` | cost-tracker | Formats a number as `$X.XX` (2 dp when > $0.50, otherwise `maxDecimalPlaces`, default 4) |

Note: `getTotalToolDuration()` is imported from state but is **not** re-exported. It is only used internally by `saveCurrentSessionCosts`.

---

## calculateUSDCost

Defined in [`source/src/utils/modelCost.ts`](../../source/src/utils/modelCost.ts).

```ts
calculateUSDCost(resolvedModel: string, usage: Usage): number
```

Resolves the model name to a canonical short name via `getCanonicalName`, looks up the matching `ModelCosts` entry, and computes:

```
(input_tokens / 1M)  * inputTokens_price
+ (output_tokens / 1M) * outputTokens_price
+ (cache_read / 1M)    * promptCacheReadTokens_price
+ (cache_creation / 1M) * promptCacheWriteTokens_price
+ web_search_requests   * webSearchRequests_price
```

### Pricing tiers

Prices are in USD per million tokens (or per request for web search).

| Tier constant | Input | Output | Cache write | Cache read | Web search | Models |
|---------------|-------|--------|-------------|------------|------------|--------|
| `COST_HAIKU_35` | $0.80 | $4 | $1 | $0.08 | $0.01 | Claude 3.5 Haiku |
| `COST_HAIKU_45` | $1 | $5 | $1.25 | $0.10 | $0.01 | Claude Haiku 4.5 |
| `COST_TIER_3_15` | $3 | $15 | $3.75 | $0.30 | $0.01 | Sonnet 3.5v2, 3.7, 4, 4.5, 4.6 |
| `COST_TIER_15_75` | $15 | $75 | $18.75 | $1.50 | $0.01 | Opus 4, 4.1 |
| `COST_TIER_5_25` | $5 | $25 | $6.25 | $0.50 | $0.01 | Opus 4.5, Opus 4.6 (standard) |
| `COST_TIER_30_150` | $30 | $150 | $37.50 | $3 | $0.01 | Opus 4.6 (fast mode) |

Opus 4.6 has dynamic pricing: when fast mode is enabled and the response's `usage.speed` is `"fast"`, the higher `COST_TIER_30_150` applies; otherwise `COST_TIER_5_25` is used. This check lives in `getModelCosts`.

When a model is not found in `MODEL_COSTS`, the system falls back to the default main-loop model's pricing (or `COST_TIER_5_25` as a last resort), logs an analytics event (`tengu_unknown_model_cost`), and sets the `hasUnknownModelCost` flag so the cost summary can display a warning.

---

## Session Persistence

Cost state is saved to and restored from the project-level config file so that resumed sessions carry forward their accumulated totals. Three functions handle this:

### getStoredSessionCosts(sessionId)

Reads the project config. Returns a `StoredCostState` object only if `lastSessionId` in the config matches the given `sessionId`. This prevents cross-session cost contamination. The returned object includes `totalCostUSD`, `totalAPIDuration`, `totalAPIDurationWithoutRetries`, `totalToolDuration`, line-change counts, `lastDuration`, and `modelUsage` (with `contextWindow` and `maxOutputTokens` recomputed from current model configs).

### restoreCostStateForSession(sessionId)

Calls `getStoredSessionCosts` and, if a match is found, passes the data to `setCostStateForRestore` in `bootstrap/state.ts`. That function overwrites the state fields and adjusts `startTime` backward by `lastDuration` so that `getTotalDuration()` (which computes `Date.now() - startTime`) continues to accumulate correctly. Returns `true` if restoration succeeded, `false` otherwise.

### saveCurrentSessionCosts(fpsMetrics?)

Writes the current cost state into the project config via `saveCurrentProjectConfig`. Persisted fields include all cost/duration/line-change counters, the full per-model usage map (with token counts and `costUSD` per model), and the current `sessionId`. Optionally accepts `FpsMetrics` to persist render performance data alongside cost data.

---

## formatTotalCost

```ts
formatTotalCost(): string
```

Produces a multi-line, `chalk.dim`-styled string intended for terminal output. The format:

```
Total cost:            $X.XXXX
Total duration (API):  Xm Xs
Total duration (wall): Xm Xs
Total code changes:    N lines added, N lines removed
Usage by model:
       claude-sonnet-4:  1,234 input, 567 output, 890 cache read, 12 cache write ($0.0123)
        claude-opus-4.6:  4,567 input, 890 output, 1,234 cache read, 56 cache write ($0.4567)
```

If `hasUnknownModelCost()` is true, the cost line appends: `(costs may be inaccurate due to usage of unknown models)`.

The per-model breakdown is generated by `formatModelUsage` (internal), which groups entries by canonical short name (so variant model IDs like dated snapshots roll up under the same label). Web search requests are included in the per-model line only when non-zero. Each model line ends with `(cost)` using the `formatCost` helper.

---

## useCostSummary Hook

Defined in [`source/src/costHook.ts`](../../source/src/costHook.ts).

```ts
function useCostSummary(getFpsMetrics?: () => FpsMetrics | undefined): void
```

A React hook (used in the Ink TUI layer) that registers a handler on the Node.js `process.on('exit')` event. When the process exits, the handler:

1. Checks `hasConsoleBillingAccess()` -- if true, writes `formatTotalCost()` to stdout.
2. Calls `saveCurrentSessionCosts(getFpsMetrics?.())` to persist the session state regardless of billing access.

The hook's cleanup function (`return` from `useEffect`) removes the listener to prevent duplicates if the component remounts. The empty dependency array (`[]`) means this runs once on mount.

`hasConsoleBillingAccess()` (from `utils/billing.ts`) returns `false` when the `DISABLE_COST_WARNINGS` environment variable is truthy, suppressing the exit summary.

---

## addToTotalSessionCost

```ts
addToTotalSessionCost(cost: number, usage: Usage, model: string): number
```

This is the main entry point that the API client calls after every successful API response. It:

1. **Updates per-model usage** via `addToTotalModelUsage` (internal), which initializes or increments the `ModelUsage` bucket for the given model. Fields updated: `inputTokens`, `outputTokens`, `cacheReadInputTokens`, `cacheCreationInputTokens`, `webSearchRequests`, `costUSD`, `contextWindow`, and `maxOutputTokens`.

2. **Updates global state** via `addToTotalCostState(cost, modelUsage, model)`, which sets the model's entry in `STATE.modelUsage` and adds `cost` to `STATE.totalCostUSD`.

3. **Records telemetry** by pushing values to OpenTelemetry counters (`costCounter`, `tokenCounter`) with model and token-type attributes. When fast mode is active and the response speed is `"fast"`, the attributes include `speed: 'fast'`.

4. **Handles advisor (sub-model) usage** by calling `getAdvisorUsage(usage)`, which extracts usage from any advisor iterations embedded in the response. For each advisor usage entry, it calculates the cost via `calculateUSDCost`, logs an analytics event (`tengu_advisor_tool_token_usage`), and recursively calls `addToTotalSessionCost` to fold the advisor's tokens into the running totals.

5. **Returns** the total cost added, including any advisor contributions.
