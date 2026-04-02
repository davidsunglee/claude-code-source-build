# Peripheral Systems

This page covers the self-contained subsystems that live at the edges of Claude Code: voice, vim mode, the bridge (Remote Control), remote session management, keybindings, upstream proxy, the buddy companion, and the migration framework. Each is feature-gated and can be understood in isolation.

All source paths below are relative to `source/src/`.

---

## Voice

**Source:** [`voice/voiceModeEnabled.ts`](../../source/src/voice/voiceModeEnabled.ts)

Voice mode lets users speak to Claude through a push-to-talk interface. The subsystem is thin on the client side -- the heavy lifting happens server-side via the `voice_stream` endpoint on claude.ai.

### Gating

Voice is guarded by the **`VOICE_MODE`** build-time feature flag (via `bun:bundle`) and a GrowthBook kill-switch (`tengu_amber_quartz_disabled`). The positive-ternary pattern ensures that when `VOICE_MODE` is stripped from external builds, string literals for the GrowthBook flag are also eliminated.

### Auth

Voice requires Anthropic OAuth. `hasVoiceAuth()` checks two things:

1. The auth provider is Anthropic (not Bedrock, Vertex, Foundry, or API key).
2. A valid `accessToken` exists in the keychain (via `getClaudeAIOAuthTokens()`).

The first call to `getClaudeAIOAuthTokens()` spawns `security` on macOS (~20-50ms); subsequent calls are memoized.

### Public API

| Function | Purpose |
|---|---|
| `isVoiceGrowthBookEnabled()` | Kill-switch check only. Use for UI visibility (command registration, config). |
| `hasVoiceAuth()` | Auth-only check. |
| `isVoiceModeEnabled()` | Full runtime gate (`hasVoiceAuth() && isVoiceGrowthBookEnabled()`). Used by `/voice`, `ConfigTool`, `VoiceModeNotice`. |

The keybinding `space` -> `voice:pushToTalk` is registered in [`keybindings/defaultBindings.ts`](../../source/src/keybindings/defaultBindings.ts) within the `Chat` context, conditional on `feature('VOICE_MODE')`.

---

## Vim Mode

**Source:** [`vim/`](../../source/src/vim/) -- `types.ts`, `motions.ts`, `operators.ts`, `textObjects.ts`, `transitions.ts`

**Hook:** [`hooks/useVimInput.ts`](../../source/src/hooks/useVimInput.ts)

A full vim-style input subsystem for the chat prompt. The implementation is structured as a finite state machine with pure functions -- no side effects in the core logic.

### Architecture

The design is split into four layers:

1. **Types** (`types.ts`) -- The state machine definition. `VimState` is the top-level discriminated union (`INSERT` | `NORMAL`). Within `NORMAL` mode, `CommandState` tracks 11 states: `idle`, `count`, `operator`, `operatorCount`, `operatorFind`, `operatorTextObj`, `find`, `g`, `operatorG`, `replace`, `indent`. The types document the state diagram in an ASCII art comment.

2. **Motions** (`motions.ts`) -- Pure functions that resolve a motion key (h/j/k/l, w/b/e, 0/^/$, G, etc.) to a target `Cursor` position. `resolveMotion()` applies a motion `count` times and stops early on no-op. Motions are classified as inclusive (`e`, `E`, `$`) or linewise (`j`, `k`, `G`, `gg`).

3. **Operators** (`operators.ts`) -- Functions that apply operators (`delete`, `change`, `yank`) with motions, finds, or text objects. Also handles standalone commands: `x` (delete char), `r` (replace), `~` (toggle case), `J` (join lines), `p`/`P` (paste), `>>` / `<<` (indent), `o`/`O` (open line). Each function takes an `OperatorContext` with `cursor`, `text`, `setText`, `setOffset`, `enterInsert`, register get/set, and `recordChange` for dot-repeat.

4. **Transitions** (`transitions.ts`) -- The state transition table. `transition()` dispatches on the current `CommandState` to handler functions (`fromIdle`, `fromCount`, `fromOperator`, etc.). Each returns a `TransitionResult` with an optional next state and/or an execute callback.

### Text Objects (`textObjects.ts`)

Supports `iw`/`aw` (word), `iW`/`aW` (WORD), quoted objects (`i"`, `a'`, `` i` ``), and bracket objects (`i(`, `a[`, `i{`, `a<`). Bracket matching handles nesting depth; quote matching pairs within a single line.

### Persistent State

`PersistentState` carries data across commands: last change (for `.` dot-repeat), last find (for `;`/`,`), register contents, and whether the register is linewise.

### useVimInput Hook

The React hook (`hooks/useVimInput.ts`) wraps `useTextInput` and adds the vim state machine. It maintains `VimState` and `PersistentState` in refs, translates Ink key events into vim keystrokes, and dispatches through `transition()`. Mode changes (`INSERT` <-> `NORMAL`) are exposed via `onModeChange` callback.

---

## Bridge (Remote Control)

**Source:** [`bridge/`](../../source/src/bridge/) -- 31 files

The bridge subsystem implements **Remote Control**, which lets users drive a CLI instance from the claude.ai web interface. A local CLI registers as a "bridge environment," polls for work (sessions), spawns child `claude` processes per session, and pipes messages over WebSocket.

### Build-Time Gate

The **`BRIDGE_MODE`** feature flag (via `bun:bundle`) gates the entire subsystem. When stripped, all GrowthBook flag string literals are also eliminated from the bundle. Runtime entitlement is checked via `isBridgeEnabled()` in [`bridgeEnabled.ts`](../../source/src/bridge/bridgeEnabled.ts), which requires:

- A claude.ai subscription (excludes Bedrock/Vertex/Foundry/API keys)
- The `tengu_ccr_bridge` GrowthBook gate

### Key Files

| File | Purpose |
|---|---|
| `bridgeApi.ts` | HTTP client for the `/v1/environments` API. Handles registration, polling, ack, stop, deregister, heartbeat, permission events, session archive/reconnect. Includes `BridgeFatalError` for non-retryable errors and `withOAuthRetry` for 401 token refresh. |
| `bridgeConfig.ts` | Auth/URL resolution. Consolidates `CLAUDE_BRIDGE_*` dev overrides (ant-only). Two layers: `*Override()` for env-var dev tokens, non-Override versions fall through to OAuth. |
| `bridgeMain.ts` | The main loop (`runBridgeLoop`). Registers the environment, polls for work with exponential backoff, spawns sessions, manages session lifecycle (timeout, heartbeat, graceful shutdown). Supports multi-session spawn modes (`single-session`, `worktree`, `same-dir`). |
| `bridgeMessaging.ts` | Transport-layer helpers shared by both the env-based and env-less bridge cores. Handles ingress routing (parses WebSocket messages, deduplicates echoes via `BoundedUUIDSet`), server control requests (`initialize`, `set_model`, `interrupt`, `set_permission_mode`, `set_max_thinking_tokens`), and title extraction. |
| `bridgeEnabled.ts` | Runtime and blocking entitlement checks. `isBridgeEnabled()` is cached/stale (for render paths). `isBridgeEnabledBlocking()` awaits GrowthBook init. `getBridgeDisabledReason()` returns a diagnostic string. Also gates env-less bridge v2, CSE shim, min version check, CCR auto-connect, and CCR mirror mode. |
| `types.ts` | Protocol types: `WorkResponse`, `WorkSecret`, `BridgeConfig`, `SessionHandle`, `BridgeApiClient`, `BridgeLogger`, `SpawnMode` (`single-session` / `worktree` / `same-dir`), `BridgeWorkerType` (`claude_code` / `claude_code_assistant`). |
| `jwtUtils.ts` | Token refresh scheduling for long-running bridge sessions. |
| `bridgePermissionCallbacks.ts` | Wiring for permission request/response flow between web UI and local CLI. |
| `bridgePointer.ts` | Pointer/URL management for the bridge connection. |
| `bridgeUI.ts` | Terminal UI for the bridge: status display, session list, QR code toggle, shimmer animation. |
| `flushGate.ts` | Ensures outbound messages are flushed before transport teardown. |
| `inboundAttachments.ts` | Resolves attachments received from the web UI. |
| `inboundMessages.ts` | Processes inbound user messages from the web UI. |
| `initReplBridge.ts` | Initializes the REPL-side bridge (outbound message forwarding from local REPL to web). |
| `remoteBridgeCore.ts` | Core bridge logic for the env-based path. |
| `envLessBridgeConfig.ts` | Configuration for the v2 env-less bridge path. |
| `replBridge.ts` | REPL bridge wiring (ties the bridge transport to the REPL message loop). |
| `replBridgeHandle.ts` | Handle abstraction for a REPL bridge connection. |
| `replBridgeTransport.ts` | Transport abstraction (SSE/WebSocket) for bridge messaging. |
| `sessionRunner.ts` | Spawns child `claude` processes for each bridge session. |
| `sessionIdCompat.ts` | `cse_*` <-> `session_*` ID format shim for compat with claude.ai frontend. |
| `workSecret.ts` | Decodes the base64url-encoded `WorkSecret` (JWT, API URL, sources, auth tokens). |
| `trustedDevice.ts` | Trusted device token for elevated security tier sessions. |
| `capacityWake.ts` | Wakes the poll loop when session capacity frees up. |
| `pollConfig.ts` / `pollConfigDefaults.ts` | Polling interval configuration. |
| `createSession.ts` | Session creation logic. |
| `codeSessionApi.ts` | API client for code session endpoints. |
| `bridgeDebug.ts` | Debug file logging. |
| `debugUtils.ts` | Debug formatting helpers (`describeAxiosError`, `debugBody`, `extractErrorDetail`). |
| `bridgeStatusUtil.ts` | Status formatting (`formatDuration`). |

### Message Flow

1. **Registration:** `bridgeApi.registerBridgeEnvironment()` sends machine name, directory, branch, git repo URL, max sessions, and worker type. Receives `environment_id` and `environment_secret`.

2. **Polling:** `bridgeApi.pollForWork()` long-polls for `WorkResponse` items (type `session` or `healthcheck`). Uses the environment secret for auth.

3. **Session lifecycle:** On receiving work, the bridge decodes the `WorkSecret` (JWT + API URL), spawns a child `claude` process via `sessionRunner`, and acknowledges the work item. The child connects via WebSocket/SSE to the session ingress endpoint.

4. **Messaging:** `bridgeMessaging.ts` routes inbound WebSocket frames: `control_response` goes to permission handlers, `control_request` goes to the control request handler, and `user` messages go to the REPL input. Echo dedup uses `BoundedUUIDSet` (a FIFO ring buffer with O(capacity) memory).

5. **Teardown:** On session end, the bridge sends an `SDKResultSuccess` message, archives the session, and optionally deregisters the environment.

---

## Remote

**Source:** [`remote/`](../../source/src/remote/) -- 4 files

The remote subsystem is the **client side** of remote session viewing -- the complement of the bridge. While the bridge runs on the machine hosting the CLI, the remote subsystem runs in a local CLI that connects to a remote session (e.g., `claude remote`).

### RemoteSessionManager (`RemoteSessionManager.ts`)

Coordinates WebSocket subscription, HTTP POST message sending, and permission request/response flow. Key behaviors:

- **Connect:** Creates a `SessionsWebSocket` and wires callbacks for messages, control requests, and lifecycle events.
- **Handle messages:** Routes `control_request` (permission prompts from CCR) to `onPermissionRequest` callback. Routes `control_cancel_request` to cancel pending permission UI. Forwards `SDKMessage` to the message callback.
- **Send messages:** Uses `sendEventToRemoteSession()` (HTTP POST via the teleport API) to send user messages.
- **Permission flow:** `respondToPermissionRequest()` sends a `control_response` with `allow` (plus `updatedInput`) or `deny` (plus `message`) via the WebSocket.
- **Interrupt:** `cancelSession()` sends an `interrupt` control request through the WebSocket.
- Supports a `viewerOnly` mode (used by `claude assistant`) where interrupts are suppressed and reconnect timeouts are disabled.

### SessionsWebSocket (`SessionsWebSocket.ts`)

WebSocket client for `wss://api.anthropic.com/v1/sessions/ws/{sessionId}/subscribe`. Features:

- Auth via `Authorization` header on upgrade (both Bun and Node/ws paths).
- Auto-reconnect with up to 5 attempts on transient close.
- Special handling for code 4001 (session not found) -- retries up to 3 times, since compaction can cause transient 4001s.
- Permanent close codes (4003 = unauthorized) stop reconnection immediately.
- 30-second ping interval for keepalive.
- Supports both Bun's native WebSocket and the `ws` package for Node.

### sdkMessageAdapter (`sdkMessageAdapter.ts`)

Converts `SDKMessage` (the wire format from CCR) to REPL-internal `Message` types. Handles:

- `assistant` -> `AssistantMessage`
- `stream_event` -> `StreamEvent`
- `result` -> `SystemMessage` (only on error; success results are suppressed)
- `system` (init, status, compact_boundary) -> `SystemMessage`
- `tool_progress` -> `SystemMessage` with tool timing info
- `user` -> conditionally converted based on options (`convertToolResults`, `convertUserTextMessages`)

Unknown message types are gracefully ignored with debug logging.

### remotePermissionBridge (`remotePermissionBridge.ts`)

Creates synthetic `AssistantMessage` and `Tool` stubs for remote permission prompts. When the remote CCR has tools the local CLI doesn't know about (e.g., MCP tools), `createToolStub()` provides a minimal `Tool` implementation that routes to the fallback permission UI.

---

## Keybindings

**Source:** [`keybindings/`](../../source/src/keybindings/) -- 14 files

A declarative, user-customizable keybinding system with context-scoped resolution and multi-keystroke chord support.

### Architecture

The system is layered:

1. **Schema** (`schema.ts`) -- Zod schemas defining the valid structure of `keybindings.json`. Defines 18 contexts (`Global`, `Chat`, `Autocomplete`, `Confirmation`, `Transcript`, `HistorySearch`, `Settings`, `Tabs`, `Attachments`, `Footer`, `MessageSelector`, `DiffDialog`, `ModelPicker`, `Select`, `Plugin`, `Help`, `Task`, `ThemePicker`) and 70+ action identifiers organized by namespace (`app:*`, `chat:*`, `autocomplete:*`, `confirm:*`, `history:*`, `transcript:*`, `select:*`, `diff:*`, `voice:*`, etc.). Bindings can be action strings, `command:*` patterns (execute a slash command), or `null` (unbind).

2. **Parser** (`parser.ts`) -- Converts keystroke strings (`"ctrl+shift+k"`, `"ctrl+x ctrl+e"`) into structured `ParsedKeystroke` objects. Supports modifier aliases: `ctrl`/`control`, `alt`/`opt`/`option`/`meta`, `cmd`/`command`/`super`/`win`, `shift`. `parseChord()` splits space-separated multi-keystroke sequences. `parseBindings()` flattens `KeybindingBlock[]` into `ParsedBinding[]`.

3. **Match** (`match.ts`) -- Low-level matching of Ink's `Key` + `input` against a `ParsedKeystroke`. Maps Ink's boolean flags (key.escape, key.return, key.upArrow, etc.) to string key names. Handles the Ink quirk where `key.meta=true` for escape. Alt and meta are collapsed into one logical modifier (terminal limitation).

4. **Resolver** (`resolver.ts`) -- Two resolution functions:
   - `resolveKey()` -- Single-keystroke resolution. Filters bindings by active contexts, last-match-wins (so user overrides shadow defaults).
   - `resolveKeyWithChordState()` -- Multi-keystroke chord resolution. Builds the chord sequence progressively, checks for prefix matches against longer chords, and returns `chord_started` (waiting for more keys), `match`, `chord_cancelled`, `unbound`, or `none`. Escape cancels a pending chord.

5. **Default bindings** (`defaultBindings.ts`) -- The built-in binding table, organized by context block. Platform-adaptive: image paste is `ctrl+v` everywhere except Windows (`alt+v`); mode cycling is `shift+tab` with a `meta+m` fallback for Windows terminals without VT mode support. Feature-gated bindings (e.g., `VOICE_MODE`, `QUICK_SEARCH`, `TERMINAL_PANEL`, `MESSAGE_ACTIONS`, `KAIROS`/`KAIROS_BRIEF`) are conditionally spread.

### Other Files

| File | Purpose |
|---|---|
| `loadUserBindings.ts` | Loads and validates `~/.claude/keybindings.json`. |
| `reservedShortcuts.ts` | Prevents users from rebinding `ctrl+c` and `ctrl+d` (double-press handling). |
| `validate.ts` | Validates user binding blocks against the schema. |
| `useKeybinding.ts` | React hook that integrates the resolver with Ink's input system. |
| `useShortcutDisplay.ts` | Hook for rendering shortcut hints in the UI. |
| `shortcutFormat.ts` | Formatting helpers for shortcut display. |
| `template.ts` | Template generation for `keybindings.json` scaffolding. |
| `KeybindingContext.tsx` | React context provider for keybinding state. |
| `KeybindingProviderSetup.tsx` | Setup component that merges default + user bindings. |

---

## Upstream Proxy

**Source:** [`upstreamproxy/`](../../source/src/upstreamproxy/) -- `upstreamproxy.ts`, `relay.ts`

A CONNECT-over-WebSocket relay for routing HTTPS traffic from CCR (Claude Code Remote) session containers through an organization-configured upstream proxy. This is a security feature that lets organizations inject credentials (e.g., Datadog API keys) into outbound traffic from remote sessions.

### How It Works

The system has two parts:

**`upstreamproxy.ts`** -- Container-side initialization, called once from `init.ts`:

1. Reads the session token from `/run/ccr/session_token`.
2. Calls `prctl(PR_SET_DUMPABLE, 0)` via Bun FFI to block same-UID ptrace (prevents prompt-injected `gdb -p $PPID` from scraping the token from the heap).
3. Downloads the upstream proxy CA certificate from the CCR API and concatenates it with the system CA bundle so subprocesses (curl, gh, python) trust the MITM proxy.
4. Starts the local CONNECT relay.
5. Unlinks the token file (token stays heap-only).
6. Sets `HTTPS_PROXY`, `SSL_CERT_FILE`, `NODE_EXTRA_CA_CERTS`, `REQUESTS_CA_BUNDLE`, `CURL_CA_BUNDLE` for all agent subprocesses.

Everything fails open -- a broken proxy setup never breaks an otherwise-working session. Gated by `CLAUDE_CODE_REMOTE` + `CCR_UPSTREAM_PROXY_ENABLED` environment variables (set server-side).

**`relay.ts`** -- The TCP-to-WebSocket relay:

- Listens on `127.0.0.1:0` (ephemeral port).
- Accepts HTTP CONNECT from curl/gh/kubectl/etc.
- Tunnels bytes over WebSocket to the CCR upstream proxy endpoint (`/v1/code/upstreamproxy/ws`).
- Bytes are wrapped in `UpstreamProxyChunk` protobuf messages (hand-encoded, single-field `bytes data = 1`), because the CCR ingress is GKE L7 with path-prefix routing and no raw CONNECT support.
- 512KB max chunk size. 30-second ping interval for keepalive.
- Supports both Bun (`Bun.listen`) and Node (`net.createServer`) runtimes. Bun path handles write backpressure explicitly (partial `sock.write()` returns); Node buffers internally.

---

## Buddy (Companion)

**Source:** [`buddy/`](../../source/src/buddy/) -- 6 files

A companion creature system (internally called "Spindle") that renders an ASCII sprite beside the user's input box. Gated by the **`BUDDY`** build-time feature flag.

### Companion Generation (`companion.ts`)

Each user gets a deterministic companion derived from `hash(userId + salt)` using a seeded Mulberry32 PRNG. The roll produces `CompanionBones`:

- **Species:** One of 18 types (duck, goose, blob, cat, dragon, octopus, owl, penguin, turtle, snail, ghost, axolotl, capybara, cactus, robot, rabbit, mushroom, chonk).
- **Rarity:** Weighted random (`common` 60%, `uncommon` 25%, `rare` 10%, `epic` 4%, `legendary` 1%). Non-common companions get a hat.
- **Eye:** One of 6 styles (`·`, `✦`, `×`, `◉`, `@`, `°`).
- **Hat:** One of 8 styles for non-common rarities (crown, tophat, propeller, halo, wizard, beanie, tinyduck).
- **Shiny:** 1% chance.
- **Stats:** 5 stats (DEBUGGING, PATIENCE, CHAOS, WISDOM, SNARK) with one peak, one dump, rest scattered. Rarity raises the stat floor.

`CompanionBones` are always regenerated from `hash(userId)` -- never persisted. Only the `CompanionSoul` (name, personality) is stored in config, making species renames and array edits safe. The roll is cached since it's called from hot paths (500ms sprite tick, per-keystroke input, per-turn observer).

### Types (`types.ts`)

Defines the `Companion` type (`CompanionBones & CompanionSoul & { hatchedAt: number }`), rarity weights, rarity star displays (`★` through `★★★★★`), and rarity-to-theme-color mappings. Species names are encoded as hex char codes to avoid triggering model-codename canary checks in the build output.

### Sprites (`sprites.ts`)

Each species has 3 animation frames (5 lines tall, 12 characters wide). Frame 0 is the rest pose, frame 1 is a fidget, frame 2 may use the hat slot (line 0) for species-specific effects (smoke, antenna, etc.). `renderSprite()` substitutes the `{E}` placeholder with the companion's eye character and overlays the hat. `renderFace()` produces a compact inline representation for each species.

### Prompt Integration (`prompt.ts`)

`getCompanionIntroAttachment()` generates a `companion_intro` attachment for the system prompt, introducing the companion by name and species. The intro tells Claude to stay out of the way when the user addresses the companion directly. Skips if already announced in the current message history or if `companionMuted` is set.

### CompanionSprite (`CompanionSprite.tsx`)

The React component that renders the sprite, speech bubble, and pet-hearts animation. Runs a 500ms tick for idle animation (mostly rest, occasional fidget, rare blink). The speech bubble shows for ~10 seconds and fades over the last ~3 seconds. Pet hearts float upward over 5 ticks (~2.5 seconds). The sprite is hidden during fullscreen mode.

### useBuddyNotification (`useBuddyNotification.tsx`)

Shows a rainbow `/buddy` teaser notification on startup during the teaser window (April 1-7, 2026) when no companion has been hatched yet. The notification auto-dismisses after 15 seconds.

---

## Migrations

**Source:** [`migrations/`](../../source/src/migrations/) -- 11 files

Each migration is a standalone function called during startup. Migrations are idempotent -- they check preconditions, apply the change, and log an analytics event.

| File | Purpose |
|---|---|
| `migrateAutoUpdatesToSettings.ts` | Moves user-set `autoUpdates: false` preference from global config to `settings.json` env var (`DISABLE_AUTOUPDATER=1`). Preserves user intent while allowing native installations to auto-update. |
| `migrateBypassPermissionsAcceptedToSettings.ts` | Moves `bypassPermissionsModeAccepted` from global config to `settings.json` as `skipDangerousModePermissionPrompt`. |
| `migrateEnableAllProjectMcpServersToSettings.ts` | Moves MCP server approval fields (`enableAllProjectMcpServers`, `enabledMcpjsonServers`, `disabledMcpjsonServers`) from project config to local settings. Merges server lists to avoid duplicates. |
| `migrateFennecToOpus.ts` | Ant-only. Remaps removed fennec model aliases to Opus 4.6 aliases (`fennec-latest` -> `opus`, `fennec-fast-latest` -> `opus[1m]` + fast mode). |
| `migrateLegacyOpusToCurrent.ts` | First-party only. Migrates explicit Opus 4.0/4.1 model strings (`claude-opus-4-20250514`, `claude-opus-4-1-20250805`, etc.) to the `opus` alias. Sets a timestamp for one-time notification. |
| `migrateOpusToOpus1m.ts` | Migrates users with `opus` pinned to `opus[1m]` when eligible for the merged Opus 1M experience (Max/Team Premium on first-party). Pro subscribers and 3P users are skipped. |
| `migrateReplBridgeEnabledToRemoteControlAtStartup.ts` | Renames the `replBridgeEnabled` config key to `remoteControlAtStartup`. The old key was an implementation detail that leaked into user-facing config. |
| `migrateSonnet1mToSonnet45.ts` | Pins `sonnet[1m]` users to explicit `sonnet-4-5-20250929[1m]` since the `sonnet` alias now resolves to Sonnet 4.6. Also migrates the in-memory model override. Tracked by `sonnet1m45MigrationComplete` completion flag. |
| `migrateSonnet45ToSonnet46.ts` | Migrates Pro/Max/Team Premium first-party users off explicit Sonnet 4.5 strings back to the `sonnet`/`sonnet[1m]` alias (which now resolves to Sonnet 4.6). Sets a notification timestamp for non-new users. |
| `resetAutoModeOptInForDefaultOffer.ts` | One-shot. Clears `skipAutoPermissionPrompt` for users who accepted the old 2-option auto-mode dialog but don't have `auto` as their default, so they see the new "make it my default mode" option. Only runs when auto mode is `enabled` (not `opt-in`). |
| `resetProToOpusDefault.ts` | Auto-migrates Pro first-party users to the Opus default. Sets a notification timestamp if the user was on the default model (no custom setting). |

### Pattern

Most migrations follow this structure:

1. Check a precondition (subscription tier, config key existence, completion flag).
2. Read from the specific settings source (`userSettings`, `localSettings`, or global config) -- never merged settings, to avoid silent promotion or infinite re-runs.
3. Write the migrated value.
4. Remove the old key or set a completion flag.
5. Log an analytics event (`tengu_migrate_*`).
