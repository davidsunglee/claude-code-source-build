# Service Layer

> The service layer is the infrastructure backbone of Claude Code — a collection of focused modules under `source/src/services/` that handle every concern that sits below the UI and above raw utilities: API communication, conversation compaction, analytics, authentication, code intelligence, and more. These services are consumed by the [query engine](query-engine.md) and other core systems; they are not exposed directly to users. The directory contains roughly 30 subdirectories plus several standalone files.

## Key Files

| File | Purpose |
|------|---------|
| `services/api/bootstrap.ts` | Startup fetch of `client_data` and `additional_model_options` from `/api/claude_cli/bootstrap` |
| `services/api/client.ts` | `getAnthropicClient()` — constructs the correct SDK client for direct API, Bedrock, Vertex, or Foundry |
| `services/api/withRetry.ts` | `withRetry()` async generator — retry logic, back-off, 529 handling, fast-mode fallback, `FallbackTriggeredError` |
| `services/api/logging.ts` | `logAPIQuery()`, `logAPIError()`, `logAPISuccessAndDuration()` — per-request analytics; re-exports `EMPTY_USAGE` |
| `services/api/emptyUsage.ts` | `EMPTY_USAGE` constant — zero-initialized `NonNullableUsage` object |
| `services/api/errors.ts` | `PROMPT_TOO_LONG_ERROR_MESSAGE`, `classifyAPIError()`, `isPromptTooLongMessage()`, `parsePromptTooLongTokenCounts()` |
| `services/compact/autoCompact.ts` | `autoCompactIfNeeded()`, threshold constants, circuit-breaker logic |
| `services/compact/compact.ts` | `compactConversation()`, `CompactionResult` interface, `POST_COMPACT_*` constants |
| `services/analytics/index.ts` | `logEvent()`, `attachAnalyticsSink()`, PII marker types |
| `services/analytics/growthbook.ts` | GrowthBook client init, `getFeatureValue_CACHED_MAY_BE_STALE()`, `onGrowthBookRefresh()` |
| `services/oauth/index.ts` | `OAuthService` class — PKCE authorization code flow for Claude.ai login |
| `services/lsp/LSPServerManager.ts` | `createLSPServerManager()` — file-extension-routed LSP lifecycle and RPC dispatch |
| `services/mcp/client.ts` | MCP client connections over stdio, SSE, and Streamable HTTP transports |
| `services/policyLimits/index.ts` | Org-level policy restriction fetch; fail-open with ETag caching |
| `services/settingsSync/index.ts` | Incremental upload of local settings to remote (interactive) and download in CCR |
| `services/teamMemorySync/index.ts` | Per-repo shared team memory sync with delta-upload semantics |
| `services/tools/toolOrchestration.ts` | `runTools()` — concurrent tool dispatch up to `CLAUDE_CODE_MAX_TOOL_USE_CONCURRENCY` (default 10) |
| `services/notifier.ts` | `sendNotification()` — routes completion notifications to terminal, OS, or custom channels |
| `services/voice.ts` | Audio recording via native NAPI module (CoreAudio/ALSA) with SoX fallback |

---

## Service Overview

| Directory / File | Responsibility |
|-----------------|----------------|
| `api/` | Anthropic SDK client construction, HTTP retry, usage logging, error classification |
| `compact/` | Conversation compaction — auto-triggered and manual, post-compact cleanup, micro-compact |
| `analytics/` | Event queue, sink routing, GrowthBook feature flags and experiments |
| `oauth/` | OAuth 2.0 + PKCE flow for Claude.ai subscription login |
| `lsp/` | Language Server Protocol client lifecycle and per-file request routing |
| `mcp/` | Model Context Protocol client connections, tool/resource proxying |
| `policyLimits/` | Org-level feature restriction fetch and enforcement |
| `settingsSync/` | Remote sync of user settings and memory files |
| `teamMemorySync/` | Shared repo-scoped team memory with server-wins merge |
| `tools/` | Tool execution orchestration, streaming executor, hooks |
| `AgentSummary/` | Background sub-agent progress summarization for coordinator mode |
| `SessionMemory/` | Session-scoped memory read/write utilities |
| `extractMemories/` | Memory extraction and consolidation prompts |
| `autoDream/` | Background memory consolidation via time/session gates |
| `MagicDocs/` | AI-assisted documentation generation prompts |
| `plugins/` | Plugin installation, CLI commands, and operation helpers |
| `PromptSuggestion/` | Prompt suggestion and speculation generation |
| `toolUseSummary/` | Per-session tool-use summary tracking |
| `tips/` | In-session tip delivery |
| `remoteManagedSettings/` | Remote-controlled settings with ETag caching and background polling |
| `diagnosticTracking.ts` | Diagnostic event collection |
| `internalLogging.ts` | Ant-only internal structured logging |
| `notifier.ts` | OS/terminal notification dispatch |
| `tokenEstimation.ts` | Rough token count estimation helpers |
| `voice.ts` | Push-to-talk audio capture |
| `vcr.ts` | Request recording/replay for testing |
| `claudeAiLimits.ts` | Rate-limit metadata and overage messaging for Claude.ai subscribers |
| `mockRateLimits.ts` | `/mock-limits` command support for Anthropic employees |

---

## API Client

### Bootstrap (`bootstrap.ts`)

On startup, `fetchBootstrapData()` issues a GET to `/api/claude_cli/bootstrap` and persists the response to `globalConfig.clientDataCache` and `additionalModelOptionsCache`. It is skipped entirely when:

- Privacy level is set to essential traffic only (`isEssentialTrafficOnly()`)
- The API provider is not first-party
- No usable OAuth token (with `user:profile` scope) and no API key is available

Service-key OAuth tokens lack the `user:profile` scope and would 403, so the bootstrap falls back to API key auth for console users. Responses are validated via a Zod schema; only changed data triggers a disk write.

### Client Construction (`client.ts`)

`getAnthropicClient()` returns the correct SDK client for the configured provider. Selection is environment-variable driven:

| Env var | Client |
|---------|--------|
| `CLAUDE_CODE_USE_BEDROCK=1` | `@anthropic-ai/bedrock-sdk` `AnthropicBedrock` |
| `CLAUDE_CODE_USE_FOUNDRY=1` | `@anthropic-ai/foundry-sdk` `AnthropicFoundry` (Azure) |
| `CLAUDE_CODE_USE_VERTEX=1` | `@anthropic-ai/vertex-sdk` `AnthropicVertex` |
| _(none)_ | `@anthropic-ai/sdk` `Anthropic` (direct or Claude.ai OAuth) |

Every client receives a standard set of default headers: `x-app: cli`, `User-Agent`, `X-Claude-Code-Session-Id`, and optional container/remote-session identifiers. A custom `buildFetch` wrapper injects a `x-client-request-id` UUID on first-party requests so timeouts (which carry no server request ID) can be correlated in server logs.

For Vertex, `getAnthropicClient()` constructs a `GoogleAuth` instance per call. The `projectId` fallback to `ANTHROPIC_VERTEX_PROJECT_ID` is used only when no project environment variable or key file is present — this avoids a 12-second GCE metadata server timeout outside GCP.

### Retry Logic (`withRetry.ts`)

`withRetry()` is an async generator that wraps every API operation. Key behaviors:

- **Default retries**: 10 (overridable via `CLAUDE_CODE_MAX_RETRIES`)
- **Backoff**: exponential starting at `BASE_DELAY_MS = 500 ms`, capped at 32 s by default, with 25 % jitter
- **529 (overloaded)**: Only foreground `QuerySource` values retry (e.g., `repl_main_thread`, `sdk`, `agent:default`); background sources (`compact`, summaries, classifiers) bail immediately to avoid cascade amplification
- **Model fallback**: After `MAX_529_RETRIES` (3) consecutive 529s on a non-custom Opus model, `FallbackTriggeredError` is thrown to signal the caller to switch models
- **`CannotRetryError`**: Wraps the original error when retries are exhausted, preserving the original stack
- **`FallbackTriggeredError`**: Thrown (not yielded) when the 529 count exceeds the threshold and a `fallbackModel` is configured — the caller replaces the model and starts a new retry loop
- **Persistent mode** (`CLAUDE_CODE_UNATTENDED_RETRY`): Retries 429/529 indefinitely with up to 5-minute back-off; yields heartbeat `SystemAPIErrorMessage` chunks every 30 s so the host does not mark the session idle
- **Fast mode integration**: On 429/529 during fast mode, short retry-after values retry with the same model (preserving prompt cache); longer values trigger a fast-mode cooldown and fall back to standard speed

Auth errors (401, 403 "token revoked", Bedrock 403, Vertex 401, ECONNRESET/EPIPE) cause the client to be recreated. OAuth 401s additionally trigger `handleOAuth401Error()` to refresh the token before the next attempt.

### Usage Logging (`logging.ts` and `emptyUsage.ts`)

Three public functions cover the request lifecycle:

- `logAPIQuery()` — fires `tengu_api_query` before the request, recording model, message count, temperature, betas, and permission mode
- `logAPIError()` — fires `tengu_api_error` on failure, including error classification, gateway detection (LiteLLM, Helicone, Portkey, Cloudflare AI Gateway, Kong, Braintrust, Databricks), and the `x-client-request-id` for server log correlation
- `logAPISuccessAndDuration()` — fires `tengu_api_success` with full token accounting (input, output, cache read, cache creation) and content length breakdowns; also emits an OpenTelemetry `api_request` event

`EMPTY_USAGE` lives in `emptyUsage.ts` as a separately importable zero-initialized `NonNullableUsage` object. It was split from `logging.ts` specifically to avoid pulling the entire API error/message dependency chain into `replBridge.ts`.

### Error Classification (`errors.ts`)

`PROMPT_TOO_LONG_ERROR_MESSAGE` (`'Prompt is too long'`) is the canonical prefix for prompt-length errors surfaced to users. Supporting helpers:

- `isPromptTooLongMessage(msg)` — checks an `AssistantMessage` for the prefix
- `parsePromptTooLongTokenCounts(raw)` — extracts actual/limit token counts from the raw API string (e.g., `"prompt is too long: 137500 tokens > 135000 maximum"`)
- `classifyAPIError()` — returns a short category string used in analytics

---

## Compaction Services

Compaction manages the conversation history when it approaches the model's context limit. The [query engine](query-engine.md) covers the full integration; this section documents the service-layer pieces.

### Auto-Compact Thresholds (`autoCompact.ts`)

| Constant | Value | Meaning |
|----------|-------|---------|
| `AUTOCOMPACT_BUFFER_TOKENS` | 13,000 | Reserve below effective context window before auto-compact fires |
| `WARNING_THRESHOLD_BUFFER_TOKENS` | 20,000 | Reserve below threshold for yellow warning UI |
| `ERROR_THRESHOLD_BUFFER_TOKENS` | 20,000 | Reserve below threshold for red error UI |
| `MANUAL_COMPACT_BUFFER_TOKENS` | 3,000 | Reserve below effective context window before blocking manual input |

`getAutoCompactThreshold(model)` computes `effectiveContextWindow - AUTOCOMPACT_BUFFER_TOKENS`, where `effectiveContextWindow` is the model's context window minus up to 20,000 tokens reserved for the compaction summary output (capped at the model's max output tokens). The `CLAUDE_AUTOCOMPACT_PCT_OVERRIDE` env var can force a percentage-based threshold for testing.

`autoCompactIfNeeded()` runs a circuit breaker: after 3 consecutive failures it stops trying for the session (prevents ~250 K wasted API calls/day seen in production). It first attempts session-memory compaction via `trySessionMemoryCompaction()`, falling back to a full `compactConversation()` call.

Auto-compact is suppressed entirely when context-collapse mode is active (to prevent racing with the 90 %/95 % commit/blocking flow).

### Compaction Core (`compact.ts`)

`compactConversation()` drives the full compaction flow: it strips images from messages, invokes a forked agent to summarize the conversation, then reconstructs the message array from a `CompactionResult`.

```ts
export interface CompactionResult {
  boundaryMarker: SystemMessage      // inserted SystemCompactBoundaryMessage
  summaryMessages: UserMessage[]     // the generated summary, injected as user turn(s)
  attachments: AttachmentMessage[]   // re-injected file reads and context
  hookResults: HookResultMessage[]   // post-compact hook outputs
  messagesToKeep?: Message[]         // messages retained verbatim after the boundary
  userDisplayMessage?: string        // optional message shown in the UI
  preCompactTokenCount?: number
  postCompactTokenCount?: number
  truePostCompactTokenCount?: number
  compactionUsage?: ReturnType<typeof getTokenUsage>
}
```

Key post-compact restore constants:

| Constant | Value |
|----------|-------|
| `POST_COMPACT_MAX_FILES_TO_RESTORE` | 5 |
| `POST_COMPACT_TOKEN_BUDGET` | 50,000 |
| `POST_COMPACT_MAX_TOKENS_PER_FILE` | 5,000 |
| `POST_COMPACT_MAX_TOKENS_PER_SKILL` | 5,000 |
| `POST_COMPACT_SKILLS_TOKEN_BUDGET` | 25,000 |

The `compact/` directory also contains `microCompact.ts` and `apiMicrocompact.ts` for lightweight per-turn cache-editing compaction, `sessionMemoryCompact.ts` for session-memory-based pruning, and `grouping.ts` which organizes messages into API rounds for the summary prompt.

---

## Analytics

### Event Queue and Sink (`analytics/index.ts`)

The analytics module has no imports, eliminating cycle risk. `logEvent()` and `logEventAsync()` queue events into an in-memory array if no sink is attached yet. `attachAnalyticsSink()` (called during app initialization) drains the queue via `queueMicrotask` — ensuring startup latency is unaffected.

**PII safety** is enforced at the type level. Event metadata values typed as plain `string` are rejected by TypeScript unless the caller explicitly casts to one of two marker types:

- `AnalyticsMetadata_I_VERIFIED_THIS_IS_NOT_CODE_OR_FILEPATHS` — for strings that have been verified to not contain code or paths; routed to all sinks including Datadog
- `AnalyticsMetadata_I_VERIFIED_THIS_IS_PII_TAGGED` — for values routed to PII-access-controlled BigQuery columns via `_PROTO_*` payload keys; `stripProtoFields()` removes them before Datadog fanout

### GrowthBook Feature Gating (`analytics/growthbook.ts`)

The GrowthBook SDK client is initialized with user attributes (session ID, device ID, platform, subscription type, organization UUID, etc.) and fetches feature definitions from the remote eval endpoint. The client is recreated when auth state changes (e.g., after login).

Key functions:

- `getFeatureValue_CACHED_MAY_BE_STALE(featureName, defaultValue)` — the primary feature flag lookup; cached and safe to call in hot paths. The "stale" caveat reflects that the remote eval may not yet have completed on first call
- `onGrowthBookRefresh(listener)` — subscribe to feature value changes; fires once immediately if init has already completed (handles the race where GrowthBook's network response lands before REPL mount)
- Experiment exposures are deduplicated per session via `loggedExposures` and forwarded to `logGrowthBookExperimentTo1P()` for first-party event logging

---

## OAuth Service

`OAuthService` in `services/oauth/index.ts` implements the OAuth 2.0 authorization code flow with PKCE for Claude.ai subscription login.

**Flow:**

1. Generate a PKCE `code_verifier` and `code_challenge` (S256)
2. Open two URLs in parallel: one for automatic browser redirect (localhost callback), one for manual code paste
3. Start a local HTTP listener (`AuthCodeListener`) on a dynamic port to receive the callback
4. Exchange the authorization code + verifier for access/refresh tokens
5. Fetch the profile to populate `subscriptionType` and `rateLimitTier`
6. Return `OAuthTokens` to the caller (`installOAuthTokens` in `utils/auth.ts`)

The `skipBrowserOpen` option allows SDK control protocol consumers (`claude_authenticate`) to own the URL display, passing both the automatic and manual URLs to their own UI.

Token storage, refresh scheduling, and 401 handling are all in `utils/auth.ts`, not the OAuth service itself.

---

## LSP Integration

`services/lsp/` provides Language Server Protocol support for code intelligence (diagnostics, go-to-definition, hover, etc.).

`createLSPServerManager()` returns an `LSPServerManager` interface that:

- Loads configured LSP servers from `config.ts` on `initialize()`
- Routes requests to the correct `LSPServerInstance` based on file extension
- Manages the LSP document sync lifecycle: `openFile()`, `changeFile()`, `saveFile()`, `closeFile()`
- Exposes `sendRequest<T>(filePath, method, params)` for generic LSP JSON-RPC calls

Each `LSPServerInstance` wraps an `LSPClient` (stdio transport) and tracks open documents. `LSPDiagnosticRegistry` aggregates diagnostics from all running servers for display in the UI.

Server configuration lives in `lsp/config.ts`; `manager.ts` handles process spawning and crash recovery.

---

## MCP Integration

`services/mcp/` is the Model Context Protocol client implementation — it manages connections from Claude Code (as MCP client) to external MCP servers. The MCP wiki page covers this in detail; see [MCP Integration](mcp-integration.md).

At the service layer, `mcp/client.ts` establishes connections over three transports:

- **stdio** — spawns a local process (`StdioClientTransport`)
- **SSE** — connects to a remote HTTP endpoint (`SSEClientTransport`)
- **Streamable HTTP** — newer HTTP transport (`StreamableHTTPClientTransport`)

`MCPConnectionManager.tsx` owns connection lifecycle and re-connection. `channelAllowlist.ts` and `channelPermissions.ts` enforce which MCP channels and operations are permitted.

---

## Other Services

### Tool Orchestration (`services/tools/`)

`toolOrchestration.ts` exports `runTools()`, which dispatches multiple tool-use blocks concurrently (up to `CLAUDE_CODE_MAX_TOOL_USE_CONCURRENCY`, default 10). `StreamingToolExecutor.ts` handles streaming output from long-running tools. `toolHooks.ts` executes pre/post-tool hooks.

### Policy Limits (`services/policyLimits/`)

Fetches organization-level feature restrictions from the Anthropic API. Eligible for console (API key) users and Claude.ai Team/Enterprise subscribers. Fails open — if the fetch fails, the session continues without restrictions. Uses ETag caching and background polling, following the same pattern as `remoteManagedSettings/`.

### Settings Sync (`services/settingsSync/`)

In interactive CLI sessions, uploads changed local settings entries to the remote incrementally. In CCR (Claude Code Remote) sessions, downloads remote settings before plugin installation. Uses `axios` with OAuth bearer tokens.

### Team Memory Sync (`services/teamMemorySync/`)

Syncs repo-scoped team memory (identified by git remote hash) across all authenticated org members. Pull is server-wins; push is delta-only (only keys whose content hash differs from `serverChecksums`). File deletions do not propagate. All mutable state (ETag, watcher suppression) is closure-scoped per caller instance to allow test isolation.

### Agent Summary (`services/AgentSummary/`)

Periodically forks a sub-agent's conversation (every ~30 s) to generate a 1–2 sentence progress summary for coordinator-mode UI display. Shares the same `CacheSafeParams` as the parent agent to maximize prompt cache hits.

### AutoDream (`services/autoDream/`)

Background memory consolidation service. Fires the `/dream` consolidation prompt as a forked sub-agent when both a time gate (hours since `lastConsolidatedAt`) and a session-count gate are satisfied. Uses a file-based lock to prevent concurrent consolidation across processes.

### Notifier (`services/notifier.ts`)

`sendNotification()` dispatches turn-completion notifications. Routes to the user's preferred channel (terminal bell, macOS native, iTerm2, custom hook), then logs which method was actually used to analytics.

### Voice (`services/voice.ts`)

Audio recording for push-to-talk input. Uses a native NAPI module (`audio-capture-napi`) that links against CoreAudio on macOS. The module is loaded lazily on the first keypress (not at startup) to avoid blocking the event loop during `dlopen`. Falls back to `sox rec` or `arecord` (ALSA) on Linux when the native module is unavailable.

---

## Interfaces

### `AnalyticsSink`

```ts
type AnalyticsSink = {
  logEvent(eventName: string, metadata: LogEventMetadata): void
  logEventAsync(eventName: string, metadata: LogEventMetadata): Promise<void>
}
```

Implemented by the 1P event logger and Datadog exporter; attached via `attachAnalyticsSink()`.

### `LSPServerManager`

```ts
type LSPServerManager = {
  initialize(): Promise<void>
  shutdown(): Promise<void>
  getServerForFile(filePath: string): LSPServerInstance | undefined
  ensureServerStarted(filePath: string): Promise<LSPServerInstance | undefined>
  sendRequest<T>(filePath: string, method: string, params: unknown): Promise<T | undefined>
  getAllServers(): Map<string, LSPServerInstance>
  openFile(filePath: string, content: string): Promise<void>
  changeFile(filePath: string, content: string): Promise<void>
  saveFile(filePath: string): Promise<void>
  closeFile(filePath: string): Promise<void>
  isFileOpen(filePath: string): boolean
}
```

### `OAuthService` (selected methods)

```ts
class OAuthService {
  startOAuthFlow(
    authURLHandler: (url: string, automaticUrl?: string) => Promise<void>,
    options?: { loginWithClaudeAi?, inferenceOnly?, skipBrowserOpen?, ... }
  ): Promise<OAuthTokens>

  handleManualAuthCodeInput(params: { authorizationCode: string; state: string }): void
  cleanup(): void
}
```

### `CompactionResult`

See the [Compaction Core](#compaction-core-compactts) section above for the full interface.

### `RetryContext` / `RetryOptions`

```ts
interface RetryContext {
  maxTokensOverride?: number
  model: string
  thinkingConfig: ThinkingConfig
  fastMode?: boolean
}
```

Passed to the `operation` callback on each attempt so the caller can adjust `max_tokens` after a context-overflow 400 error.

---

## Feature Gates

Several service behaviors are gated on GrowthBook features or build-time `feature()` flags:

| Flag | Service | Effect |
|------|---------|--------|
| `UNATTENDED_RETRY` | `withRetry` | Enables persistent 429/529 retry mode (`CLAUDE_CODE_UNATTENDED_RETRY` env var) |
| `REACTIVE_COMPACT` | `autoCompact` | Suppresses proactive auto-compact; relies on API 413 to trigger reactive compact |
| `CONTEXT_COLLAPSE` | `autoCompact` | Suppresses auto-compact when context-collapse mode is active |
| `BASH_CLASSIFIER` | `withRetry` | Adds `bash_classifier` query source to the foreground 529-retry allowlist (Ant builds only) |
| `PROMPT_CACHE_BREAK_DETECTION` | `autoCompact` | Notifies the cache-break detector on session-memory compaction |
| `CACHED_MICROCOMPACT` | `logging` | Logs `cacheDeletedInputTokens` in the `tengu_api_success` event |
| `KAIROS` | `compact` | Integrates session transcript module during compaction |
| `tengu_disable_keepalive_on_econnreset` | `withRetry` | Disables HTTP keep-alive after ECONNRESET/EPIPE to prevent stale socket reuse |
| `tengu_cobalt_raccoon` | `autoCompact` | Enables reactive-only mode (GrowthBook remote value) |
