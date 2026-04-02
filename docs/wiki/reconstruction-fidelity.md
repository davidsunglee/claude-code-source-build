# Reconstruction Fidelity

> **Context.** The source code in this repository was not originally published by Anthropic — it was extracted from the published `cli.js.map` source map for Claude Code v2.1.88. Source map extraction recovers the vast majority of application logic faithfully, but several categories of code are either unavailable, reconstructed by hand, or present only as stubs. This page catalogs every known gap and approximation so you know exactly what you are reading.

---

## Summary Fidelity Table

| Category | Status | Notes |
|---|---|---|
| Application TypeScript/TSX source | ~100% recovered | 4,756 modules embedded in `cli.js.map` |
| npm / open-source dependencies | ~80 packages reinstalled from npm | Exact versions matched via `package.json` |
| `@ant/*` internal packages | 4 packages stubbed or reconstructed | Not published; see section below |
| Native addon binaries | 4 `.node` files present | macOS pre-builts only; arm64 |
| Feature flags | 81 flags found; 4 enabled in build | `bun:bundle` import replaced by inline shim |
| React/Reconciler internals | 4 files pinned from source map | Prevents version skew with npm install |
| Vendor native packages | 3 packages shimmed from workspace source | Compiled on demand against local headers |

---

## Unavailable `@ant/*` Packages

These four first-party packages are referenced by the extracted source but were never published to npm. The build script lists them in `unavailableOverlayPackages` and generates stubs automatically.

### `@ant/claude-for-chrome-mcp`

A Chrome extension MCP integration that allows Claude Code to communicate with a companion browser extension. The package provides a Chrome DevTools Protocol bridge so the agent can inspect DOM elements, read console output, and issue browser commands via MCP tool calls. It was never released publicly.

**Build behavior.** If source-map content is available for this package, it is used as-is. Otherwise, the build generates a `package.json` (version `0.0.0-stub`) pointing to an `index.js` containing:

```js
// AUTO-STUB: unavailable package
export default new Proxy({}, { get: (t, k) => () => {} });
```

Known subpath imports (`sentinelApps`, `types`, `subGates`) receive the same Proxy stub.

### `@ant/computer-use-input`

Mouse, keyboard, and low-level input injection via a native addon. The pre-built binary `source/native-addons/computer-use-input.node` is present in this repository, and the build copies it into `.cache/workspace/node_modules/@ant/computer-use-input/prebuilds/computer-use-input.node`. On first build, the package entry itself is stubbed; subsequent passes wire the stub to the real `.node` file once the package directory exists.

### `@ant/computer-use-mcp`

The computer-use MCP server — a full MCP implementation that exposes screenshot, click, type, scroll, drag, and app-management tools to the Claude agent. Two files from this package that were missing from the source map are manually reconstructed by `restoreMissingSourceMapFiles()` (see section below). The rest of the package loads from source-map content when available.

### `@ant/computer-use-swift`

Screen capture and macOS app management via a Swift-backed native addon. The pre-built binary `source/native-addons/computer-use-swift.node` is present and copied to `.cache/workspace/node_modules/@ant/computer-use-swift/prebuilds/computer_use.node` during the build. Like `computer-use-input`, the package entry starts as a Proxy stub and is promoted to the real binary once pre-builts are in place.

### How the Proxy stub works

When a package has no real source, every property access on the default export returns a no-op function:

```js
export default new Proxy({}, { get: (t, k) => () => {} });
```

Named exports discovered via previous build failures are added as `export const <name> = undefined;` entries. For deeply-nested imports (e.g. typed constants), named exports are promoted from `undefined` to the actual constant values extracted from the type definitions.

---

## Reconstructed Type-Only Modules

`restoreMissingSourceMapFiles()` in `scripts/build-cli.mjs` (line 378) writes two files that were referenced by the extracted source but absent from the source map. Both files existed in the original build only as TypeScript type declarations (erased at compile time), so their runtime content must be inferred from call sites and type definitions.

### `node_modules/@ant/computer-use-mcp/src/executor.ts`

Hand-reconstructed interfaces for the computer-use executor abstraction:

- `ScreenshotResult` — `{ base64, width, height, displayId, scaleX, scaleY }`
- `DisplayGeometry` — `{ displayId, x, y, width, height, scaleFactor }`
- `FrontmostApp` — `{ bundleId, name, pid }`
- `InstalledApp` — `{ bundleId, name, path }`
- `RunningApp` — `{ bundleId, name, pid, isHidden }`
- `ResolvePrepareCaptureResult` — `{ displayId, geometry }`
- `ComputerExecutor` — the full executor interface with 11 async methods (screenshot, click, doubleClick, type, key, moveMouse, drag, scroll, getInstalledApps, getRunningApps, getFrontmostApp, getDisplayGeometry, resolveAndPrepareCapture)

The file is marked `// Reconstructed — type-only module (erased at build time)`.

### `node_modules/@ant/computer-use-mcp/src/subGates.ts`

Reconstructed constants for the `CuSubGates` feature toggle object:

- `ALL_SUB_GATES_OFF` — all six gates (`pixelValidation`, `clipboardPasteMultiline`, `mouseAnimation`, `hideBeforeAction`, `autoTargetDisplay`, `clipboardGuard`) set to `false`
- `ALL_SUB_GATES_ON` — same gates all set to `true`

The file is marked `// Reconstructed — CuSubGates on/off constants`.

---

## Native Addon Status

Four pre-built `.node` binaries are included in `source/native-addons/`. All were compiled for macOS (Apple Silicon / arm64). No Linux or Windows builds are present.

| File | Used by | Purpose | Build wiring |
|---|---|---|---|
| `audio-capture.node` | `audio-capture-napi` (vendor) | Microphone capture for voice mode | Linked via `nativePackageTargets` |
| `computer-use-input.node` | `@ant/computer-use-input` | Mouse/keyboard input injection | Copied to package `prebuilds/` by `ensureNativeAddonPrebuilds()` |
| `computer-use-swift.node` | `@ant/computer-use-swift` | Screen capture and macOS app management | Copied to package `prebuilds/` by `ensureNativeAddonPrebuilds()` |
| `image-processor.node` | `sharp` (via vendor) | Image resizing/processing | Available in workspace; `sharp` package.json is stubbed to `lib/index.js` |

All `.node` files are platform-specific compiled binaries. They load correctly on macOS arm64. On other platforms they will either fail to load or not be reached (the computer-use features gate on macOS detection).

---

## Feature Flag Inventory

Feature flags originate from `bun:bundle` imports in the original build:

```ts
import { feature } from 'bun:bundle';
```

At build time, the `patchFeatureFlags()` function (line 592) replaces every such import with an inline constant function:

```js
const feature = (flag) => (["BUILDING_CLAUDE_APPS","BASH_CLASSIFIER","TRANSCRIPT_CLASSIFIER","CHICAGO_MCP"]).includes(flag);
```

This means **only the four flags in `enabledBundleFeatures` return `true`** in this reconstruction. All other flags evaluate to `false`, which silently disables large portions of functionality that exist in the source.

**Total flags found in `source/src/`:** 81

### Enabled in This Build

These four flags are in `enabledBundleFeatures` and evaluate to `true`:

| Flag | Effect |
|---|---|
| `BUILDING_CLAUDE_APPS` | Enables Claude Apps ("Kairos") build mode with app creation tooling |
| `BASH_CLASSIFIER` | Enables the ML-based bash command safety classifier |
| `TRANSCRIPT_CLASSIFIER` | Enables the transcript-level safety classifier |
| `CHICAGO_MCP` | Enables the "Chicago" MCP channel integration |

### AI / Model Flags

| Flag | Likely purpose |
|---|---|
| `ULTRATHINK` | Extended thinking / deep reasoning mode |
| `ULTRAPLAN` | Enhanced planning mode for complex tasks |
| `KAIROS` | Full Kairos (Claude Apps) agent mode — the primary conditional for app-building features |
| `KAIROS_BRIEF` | Brief/summary variant of Kairos output |
| `KAIROS_CHANNELS` | Kairos notification channel support |
| `KAIROS_DREAM` | Speculative/dream-mode variant of Kairos |
| `KAIROS_GITHUB_WEBHOOKS` | GitHub webhook handling within Kairos |
| `KAIROS_PUSH_NOTIFICATION` | Push notification delivery from Kairos |
| `CONTEXT_COLLAPSE` | Aggressive context window collapsing for long sessions |
| `PROACTIVE` | Proactive assistant mode (unsolicited suggestions) |
| `TOKEN_BUDGET` | Token budget enforcement / hard limits |
| `PROMPT_CACHE_BREAK_DETECTION` | Detects unexpected prompt cache breaks |
| `CACHED_MICROCOMPACT` | Uses cached summaries for micro-compaction |
| `ANTI_DISTILLATION_CC` | Anti-distillation content controls |
| `NATIVE_CLIENT_ATTESTATION` | Client attestation for model trust |

### UI / UX Flags

| Flag | Likely purpose |
|---|---|
| `AUTO_THEME` | Automatic terminal theme detection |
| `HISTORY_PICKER` | Visual conversation history picker |
| `HISTORY_SNIP` | Snippet extraction from conversation history |
| `STREAMLINED_OUTPUT` | Simplified/streamlined response rendering |
| `MESSAGE_ACTIONS` | In-message action buttons (copy, edit, etc.) |
| `TERMINAL_PANEL` | Side-panel terminal integration |
| `CONNECTOR_TEXT` | Connector/relay text rendering |
| `QUICK_SEARCH` | Quick-search overlay in the TUI |
| `VOICE_MODE` | Voice input/output mode via audio capture |
| `NATIVE_CLIPBOARD_IMAGE` | Paste images from the native clipboard |
| `REVIEW_ARTIFACT` | Review/preview artifacts before accepting |

### Platform / Infrastructure Flags

| Flag | Likely purpose |
|---|---|
| `IS_LIBC_GLIBC` | Linux glibc detection (selects native binary variant) |
| `IS_LIBC_MUSL` | Linux musl detection (Alpine/Docker environments) |
| `POWERSHELL_AUTO_MODE` | Windows PowerShell auto-detection |
| `TREE_SITTER_BASH` | Tree-sitter-based bash AST parsing |
| `TREE_SITTER_BASH_SHADOW` | Shadow/fallback tree-sitter bash parser |
| `PERFETTO_TRACING` | Perfetto-format performance tracing |
| `SLOW_OPERATION_LOGGING` | Logs operations exceeding a time threshold |
| `ALLOW_TEST_VERSIONS` | Permits loading test/pre-release model versions |
| `HARD_FAIL` | Converts warnings to hard failures (CI mode) |
| `ENHANCED_TELEMETRY_BETA` | Extended telemetry collection (opt-in beta) |
| `SHOT_STATS` | Per-turn statistics/shot counters |
| `COWORKER_TYPE_TELEMETRY` | Telemetry on co-worker interaction patterns |
| `MEMORY_SHAPE_TELEMETRY` | Telemetry on memory/context shape |
| `CCR_AUTO_CONNECT` | Cloud Code Remote auto-connect |
| `CCR_MIRROR` | CCR session mirroring |
| `CCR_REMOTE_SETUP` | CCR remote workspace setup flow |

### Agent / Multi-Agent Flags

| Flag | Likely purpose |
|---|---|
| `DAEMON` | Background daemon mode for persistent sessions |
| `BRIDGE_MODE` | Bridge/relay mode connecting two Claude instances |
| `COORDINATOR_MODE` | Multi-agent coordinator role |
| `FORK_SUBAGENT` | Spawn subagent in a forked process |
| `BG_SESSIONS` | Background session management |
| `AGENT_TRIGGERS` | Event-driven agent trigger system |
| `AGENT_TRIGGERS_REMOTE` | Remote trigger delivery to agents |
| `AGENT_MEMORY_SNAPSHOT` | Snapshot agent memory across sessions |
| `VERIFICATION_AGENT` | Spawns a secondary agent to verify outputs |
| `BUILTIN_EXPLORE_PLAN_AGENTS` | Built-in Explore/Plan sub-agents |
| `MONITOR_TOOL` | Monitoring tool for observing agent behavior |
| `TORCH` | "Torch" distributed agent feature |
| `LODESTONE` | Lodestone coordination service |
| `BUDDY` | Buddy/companion agent mode |
| `UDS_INBOX` | Unix domain socket inbox for IPC |

### Compaction / Memory Flags

| Flag | Likely purpose |
|---|---|
| `REACTIVE_COMPACT` | Reactive (event-driven) compaction triggers |
| `COMPACTION_REMINDERS` | User reminders when compaction is due |
| `EXTRACT_MEMORIES` | Extract and persist memories from conversations |
| `AGENT_MEMORY_SNAPSHOT` | (Also listed under Agent above) |
| `TEAMMEM` | Team-shared memory store |
| `AWAY_SUMMARY` | Summary generated when user is away |
| `CONTEXT_COLLAPSE` | (Also listed under AI/Model above) |

### Skills / Plugins Flags

| Flag | Likely purpose |
|---|---|
| `MCP_SKILLS` | Load skills via MCP server |
| `EXPERIMENTAL_SKILL_SEARCH` | Fuzzy skill search UI |
| `SKILL_IMPROVEMENT` | Skill improvement suggestion hooks |
| `RUN_SKILL_GENERATOR` | In-session skill generator tool |
| `TEMPLATES` | Template-based prompt/skill system |
| `WORKFLOW_SCRIPTS` | Scripted workflow execution |

### Internal / Debug Flags

| Flag | Likely purpose |
|---|---|
| `NEW_INIT` | New session initialization flow |
| `OVERFLOW_TEST_TOOL` | Tool for testing context overflow behavior |
| `BREAK_CACHE_COMMAND` | Debug command to manually break prompt cache |
| `FILE_PERSISTENCE` | File-based session persistence (non-default) |
| `DOWNLOAD_USER_SETTINGS` | Download settings from remote store |
| `UPLOAD_USER_SETTINGS` | Upload settings to remote store |
| `COMMIT_ATTRIBUTION` | Attribute git commits to Claude |
| `UNATTENDED_RETRY` | Auto-retry in unattended/headless mode |
| `MESSAGEQUEUEMANAGER` / `messageQueueManager` | Internal message queue (seen in hook usage) |

---

## Pinned React Files

The `pinnedOverlaySourceMapPaths` set names four React files that are extracted from the source map and written to the workspace even though `react` and `react-reconciler` are listed in `overlayManagedPackages` (i.e., reinstalled from npm):

```
node_modules/react/cjs/react.production.js
node_modules/react/cjs/react-compiler-runtime.production.js
node_modules/react-reconciler/cjs/react-reconciler.production.js
node_modules/react-reconciler/cjs/react-reconciler-constants.production.js
```

**Why pin these?** The application was compiled against a specific internal build of React that may contain Anthropic-specific patches or a version that does not exactly match what npm distributes. Using the source-map copy of these four CJS production files ensures that the React fiber internals, compiler runtime, and reconciler constants exactly match the version the rest of the source was written against. If the npm-installed version were used instead, subtle fiber identity mismatches could cause hooks to behave incorrectly or the reconciler to reject unexpected fiber fields.

The `react/compiler-runtime` and `react-reconciler/constants.js` subpath imports are additionally registered in `specialPackageTargets` so the bundler can resolve them:

```js
const specialPackageTargets = new Map([
  ['react/compiler-runtime',           'node_modules/react/cjs/react-compiler-runtime.production.js'],
  ['react-reconciler/constants.js',    'node_modules/react-reconciler/cjs/react-reconciler-constants.production.js'],
]);
```

---

## Vendor (Native) Packages

`nativePackageTargets` lists three packages whose npm releases are not compatible with the workspace (typically because they contain N-API native addons that must be compiled locally):

| Package name | Workspace source | Purpose |
|---|---|---|
| `audio-capture-napi` | `vendor/audio-capture-src/index.ts` | Wraps `audio-capture.node` for microphone access in voice mode |
| `modifiers-napi` | `vendor/modifiers-napi-src/index.ts` | Keyboard modifier key state detection |
| `color-diff-napi` | `src/native-ts/color-diff/index.ts` | Fast pixel-level color difference for computer-use validation |

For each of these, the build generates a proxy `index.js` in `node_modules/<package>/` that re-exports from the workspace source path. This means the TypeScript source compiles and links against the workspace vendor directory instead of a binary npm package.

---

## Overall Fidelity Assessment

The reconstruction is high-fidelity for everything that lives in TypeScript source: the 4,756 modules captured in `cli.js.map` are reproduced verbatim, and the ~80 open-source npm dependencies are reinstalled at exact versions. The main gaps are:

1. **Feature flag coverage.** 77 of 81 feature flags default to `false`. Large subsystems — Kairos app building, voice mode, daemon mode, bridge mode, coordinator mode, team memory, and most telemetry paths — are compiled in but not exercised by default. The source is readable but the codepaths are inactive.

2. **Four `@ant/*` packages.** These contain unpublished first-party code. Three (`@ant/claude-for-chrome-mcp`, `@ant/computer-use-mcp`, `@ant/computer-use-input`) have partial or full source in the map; `@ant/computer-use-swift` is effectively binary-only at the TypeScript layer.

3. **Two reconstructed interface files.** `executor.ts` and `subGates.ts` inside `@ant/computer-use-mcp` are hand-reconstructed from type information, not extracted from the source map. Their runtime effect is zero (they are pure type exports), but the shapes may not be byte-for-byte identical to Anthropic's originals.

4. **Native addons are macOS arm64 only.** Features that depend on `computer-use-swift.node`, `computer-use-input.node`, `audio-capture.node`, or `image-processor.node` will not work on Linux or Windows hosts.

For the purpose of reading and understanding the application logic, none of these gaps are significant. The source of every major subsystem is present and accurate.

---

*See also: [Build System](build-system.md) for the full extraction and patching pipeline.*
