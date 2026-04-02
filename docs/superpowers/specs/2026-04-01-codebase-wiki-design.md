# Claude Code v2.1.88 Architecture Wiki — Design Spec

## Overview

A comprehensive, multi-page Markdown wiki documenting the architecture of Claude Code v2.1.88, reconstructed from published source maps. The wiki uses a **hybrid structure**: a narrative spine (`architecture-overview.md`) tells the full system story top-down with Mermaid diagrams, then links to self-contained deep-dive reference pages for each subsystem.

**Target audience**: Senior engineer encountering this codebase for the first time. Assumes strong general knowledge (TypeScript, React, CLI tooling) but no familiarity with Claude Code internals. Domain concepts (MCP, Ink, source map extraction) are explained; language features are not.

**Output location**: `docs/wiki/`

**Format**: Multi-page Markdown with Mermaid diagrams (renders natively in GitHub and VS Code).

---

## Page Inventory (18 pages)

### Entry Point

#### `index.md` — Master Index
- One-line description and link for every page
- **Reading guide**: suggested order for first-timers — architecture-overview → query-engine → tool-system → state-management → explore from there
- Quick-reference table grouping pages by category

### Narrative Spine

#### `architecture-overview.md` — The Map (~2,000 words + 5 diagrams)

The single most important page. Tells the full system story top-down with prose and diagrams, linking to deep-dive pages for details.

**Diagram 1 — High-Level Architecture** (Mermaid flowchart):
- ~8 boxes showing how user input flows through major layers
- User Input → CLI Entry → Command Dispatch / REPL → Query Engine → API Call → Tool Execution → back to Query Engine
- Ink TUI renders everything; State Management feeds all layers; Services underpin everything

**Diagram 2 — The Conversation Loop** (Mermaid sequence diagram):
- A single turn: message normalization → API call → response streaming → tool use detection → tool execution → result injection → next turn or compaction

**Diagram 3 — Tool Execution Lifecycle** (Mermaid flowchart):
- Tool call in API response → permission check → hook execution → tool function invocation → result formatting → JSX rendering → back to query engine

**Diagram 4 — State Architecture** (Mermaid class/block diagram):
- Three tiers: bootstrap/state.ts globals ↔ AppStateStore (Zustand-like) ↔ React contexts
- What lives where and why

**Diagram 5 — Module Dependency Map** (Mermaid flowchart):
- How major subsystems depend on each other
- Foundational (state, ink, permissions) vs. leaf nodes (voice, vim, cost tracking)

Each diagram section has 2-3 paragraphs of narrative explaining the "why" and linking to the relevant deep-dive page.

**Closing**: Reading guide, note about build system / reconstruction chapters.

---

### Core Engine (4 pages)

#### `query-engine.md` — The Conversation Loop

The beating heart of Claude Code — the state machine that drives the conversation.

Covers:
- **QueryEngine state machine** — config type (`QueryEngineConfig`), message buffer management, turn lifecycle, stop conditions
- **Message normalization** — transforming messages for the API: stripping internal types, handling tombstones, compact boundaries
- **Compaction strategies** — four mechanisms:
  1. Manual compact (user-triggered `/compact`)
  2. Auto-compact (token threshold warning)
  3. Reactive compact (feature-gated `REACTIVE_COMPACT`)
  4. Snip compact (feature-gated `HISTORY_SNIP`)
  - When each triggers, what gets preserved vs. summarized
- **Token estimation & prompt caching** — how the engine estimates context usage and manages cache boundaries
- **Post-sampling hooks** — what runs after each API response before the next turn
- **Error handling** — retry logic, `PROMPT_TOO_LONG` recovery, stop failure handling
- **`query.ts` orchestration** — the wrapper that sets up QueryEngine with tools, context, coordinator mode
- **`context.ts` system prompt assembly** — how git status, CLAUDE.md, memory, and user context are composed into the system prompt (2,000-char truncation rule)
- Mermaid diagram: state machine showing turn transitions

#### `tool-system.md` — Tool Architecture

How tools are defined, registered, and executed.

Covers:
- **`Tool` type anatomy** — input schema (JSON Schema with `type:'object'`), permission model, execution function signature
- **`ToolUseContext`** — the ~300-line context object passed to all tool executions:
  - `options` (commands, debug, model, tools, verbose, thinking config, MCP clients/resources, agent definitions, budget, custom system prompt, query source, refreshTools)
  - `abortController` for cancellation
  - `readFileState` LRU cache with dedup
  - `getAppState`/`setAppState` Zustand-like store interface
  - `setAppStateForTasks` session-scoped always-shared state
  - `handleElicitation` for URL-based error handling
  - `setToolJSX` for local React rendering
  - `addNotification`, `appendSystemMessage`, `sendOSNotification`
  - File/glob/tool result budget limits
  - File history & attribution state updaters
- **Tool registration** — how tools are collected, merged with MCP tools, filtered by feature flags
- **Execution lifecycle** — `StreamingToolExecutor`, orchestration layer (`toolOrchestration.ts`), pre/post hooks (`toolHooks.ts`)
- **Permission model** — how tools declare permissions, how the permission system resolves them
- **JSX rendering** — how tools set local UI via `setToolJSX`
- **Result budgeting** — file/glob/tool result size limits and enforcement
- **Agent tool subprocess model** — how `AgentTool` spawns isolated processes with their own context
- Mermaid diagram: tool execution pipeline

#### `tool-catalog.md` — Complete Tool Reference

Every tool cataloged. Per-tool entry format:
- Name, one-line purpose
- Input schema (key parameters)
- Permission requirements
- Behavioral notes (interesting implementation details)

Grouped by category:
- **Core I/O**: BashTool, FileReadTool, FileEditTool, FileWriteTool, PowerShellTool, REPLTool
- **Search**: GlobTool, GrepTool, ToolSearchTool
- **AI/Web**: WebSearchTool, WebFetchTool
- **Task Management**: TaskCreateTool, TaskUpdateTool, TaskGetTool, TaskListTool, TaskStopTool, TaskOutputTool
- **Workspace**: EnterWorktreeTool, ExitWorktreeTool, EnterPlanModeTool, ExitPlanModeTool
- **MCP**: MCPTool, ListMcpResourcesTool, ReadMcpResourceTool, McpAuthTool
- **Collaboration**: AgentTool, SendMessageTool, TeamCreateTool, TeamDeleteTool
- **Scheduling**: ScheduleCronTool, RemoteTriggerTool
- **Configuration**: ConfigTool, SkillTool
- **Special**: AskUserQuestionTool, BriefTool, SleepTool, SyntheticOutputTool, NotebookEditTool, TodoWriteTool

#### `command-system.md` + `command-catalog.md` — Commands

**`command-system.md`** covers:
- **Command type** — structure of a command definition
- **Registration** — how `commands.ts` imports 70+ commands, feature-gated lazy loading
- **Dispatch** — how `/foo` resolves to a command handler
- **Lazy loading** — how large commands (e.g., insights.ts at 113KB) defer loading
- **Feature-gated commands** — how `bun:bundle` feature flags hide entire commands

**`command-catalog.md`** catalogs every command:
- **Session**: init, login, logout, resume, session, teleport, clear, rename, tag, fork
- **Workspace**: add-dir, memory, context, env, config, theme, color
- **Development**: commit, commit-push-pr, diff, doctor, ide, desktop, mobile, tasks
- **Productivity**: review, ultrareview, cost, usage, status, summary
- **Configuration**: model, output-style, rate-limit-options, fast, passes, hooks, keybindings
- **Integration**: skills, plugin, reload-plugins, mcp, voice, bridge, buddy
- **Analysis**: insights, effort, ctx_viz
- **Workflow**: plan, compact, rewind, workflows
- **Advanced**: brief (KAIROS), proactive, assistant, copy
- **Internal/Ant-only**: ant-trace, bughunter, security-review, agentsPlatform, ultraplan (ULTRAPLAN), torch (TORCH)

---

### Infrastructure (3 pages)

#### `state-management.md` — Three-Tier State Architecture

Covers:
- **Tier 1: Bootstrap state** (`bootstrap/state.ts`) — global memoized singletons: session ID, project root, cost accumulation, telemetry meters, OpenTelemetry providers, hook registration. Why it's separate from React (accessible from non-React code).
- **Tier 2: AppStateStore** (`state/AppStateStore.ts`, `state/store.ts`) — Zustand-like store with ~50 fields. The `Store<T>` implementation (subscribe, getState, setState with Object.is equality). What lives here: settings, model selection, permission mode, task list, expanded view, brief mode, etc.
- **Tier 3: React contexts** (`context/` directory) — StatsStore, notifications, FPS metrics, mailbox, modal state, overlay state, voice, queued messages. Why each gets its own context (render isolation).
- **`onChangeAppState.ts`** — side-effect handlers on state transitions
- **Selectors** — memoized selector pattern for derived state
- Mermaid diagram: three tiers with read/write arrows

#### `ink-tui.md` — Custom Terminal UI Framework

Claude Code's heavily customized Ink fork.

Covers:
- **What Ink is** — React renderer for terminal UIs using Yoga layout. This is a custom fork, not the npm `ink` package.
- **DOM layer** — virtual DOM nodes (`dom.ts`), focus management (`focus.ts`)
- **Layout engine** — Yoga-based flexbox: `layout/engine.ts`, `layout/yoga.ts`, `layout/geometry.ts`, `layout/node.ts`
- **Rendering pipeline** — render-node-to-output → render-to-screen → ANSI escape sequences
- **Component library** — Box, Text, Button, Link, Spacer, Newline, App wrapper
- **Input system** — `parse-keypress.ts`, click events, terminal focus events, mouse hit testing (`hit-test.ts`)
- **ANSI handling** — `colorize.ts`, `wrapAnsi.ts`, `Ansi.tsx`
- **Frame loop** — `frame.ts`, how render cycles are scheduled and batched
- **Custom hooks** — useAnimationFrame, useInput, useSelection, useTerminalFocus, useTerminalViewport, etc.
- **AlternateScreen** — primary vs. alternate terminal buffer switching
- **Why the fork** — likely: performance, custom input handling, hit testing, selection not in stock Ink

#### `permissions-security.md` — Trust & Permission Model

Covers:
- **Permission modes** — default (ask per tool), auto (approve all), bypass
- **Tool permission declarations** — how tools specify what they need
- **Resolution flow** — request → check cache → check rules → prompt user → cache decision
- **Denial tracking** — `denialTracking.ts`, avoiding re-prompts for denied permissions
- **Trust dialogs** — UI for permission prompts, trust-on-first-use
- **Bash safety classifier** — `BASH_CLASSIFIER` feature flag, dangerous command detection
- **Hook-based permissions** — pre-tool hooks that can approve/deny/modify tool calls
- **Session vs. persistent permissions** — what resets between sessions

---

### Subsystems (6 pages)

#### `mcp-integration.md` — Model Context Protocol

Covers:
- **What MCP is** — standard protocol for AI tool integration
- **Server lifecycle** — discovery, connection, approval workflow, health monitoring
- **Tool bridging** — how MCP server tools become Claude Code tools (MCPTool wrapping)
- **Resource system** — ListMcpResources, ReadMcpResource, context injection
- **Authentication** — OAuth flows, elicitation handlers, McpAuthTool
- **Channel permissions** — per-server permission scoping
- **Claude AI MCP servers** — built-in servers (Gmail, Google Calendar) with special auth
- **XAA integration** — Anthropic internal IDP login
- **VSCode SDK support** — IDE-hosted MCP server connection

#### `task-system.md` — Background Work & Multi-Agent Coordination

Covers:
- **Task state union type** — 7 variants: LocalShell, LocalAgent, RemoteAgent, InProcessTeammate, LocalWorkflow, MonitorMcp, Dream
- **Task lifecycle** — creation → in_progress → completed/stopped, `isBackgrounded` flag
- **Per-variant details**:
  - LocalShellTask — shell command execution with progress tracking
  - LocalAgentTask — spawning isolated agent subprocesses
  - InProcessTeammateTask — teammate execution in same process
  - RemoteAgentTask — remote agent coordination
  - DreamTask — speculative background tasks (auto-dream service)
  - LocalWorkflowTask — workflow script execution
  - MonitorMcpTask — MCP server monitoring
- **Task UI** — status pills, task list component, shell progress display
- **Task tools** — TaskCreate/Update/Get/List/Stop/Output interactions
- **Coordinator mode** — multi-agent coordination (COORDINATOR_MODE feature gate)

#### `plugin-skill-system.md` — Extending Claude Code

Covers:
- **Plugin architecture** — plugin type definition, loading pipeline, caching
- **Bundled plugins** — what ships built-in, `builtinPlugins.ts`
- **Plugin commands & skills** — how plugins register commands and skills
- **Skill system** — skill discovery, `loadSkillsDir.js`, bundled skills, `bundledSkills.ts`
- **SkillTool** — how the tool invokes skills dynamically
- **Cache invalidation** — `clearPluginCommandCache`, `clearPluginSkillsCache`

#### `service-layer.md` — Backend Services

Covers:
- **API client** (`services/api/`) — bootstrap, client construction, `withRetry` + `FallbackTriggeredError`, usage tracking (`logging.ts`, `EMPTY_USAGE`), prompt dumping for debugging
- **Compaction services** — autoCompact thresholds, reactiveCompact (feature-gated), snipCompact with history projection
- **Analytics** — event logging with PII verification, GrowthBook feature gating/experimentation
- **OAuth** — authentication flows for external services
- **LSP integration** — Language Server Protocol for code intelligence
- **Team memory sync** — cross-session memory synchronization
- **Remote managed settings** — MDM-style settings distribution
- **Auto-dream** — background speculative task automation
- **Rate limiting & policy** — request throttling, usage policy enforcement
- **Session memory** — per-session state persistence
- **Prompt suggestions** — intelligent suggestion generation

#### `component-layer.md` — UI Components & React Hooks

Covers:
- **Screen components** — REPL (main interactive), Doctor (diagnostics), ResumeConversation (session picker)
- **Component inventory by category**:
  - Design system: ThemedBox, ThemedText, color utilities
  - Messages: message rendering, diffs, structured output
  - Tools: tool execution UI
  - Permissions: permission dialogs, trust dialogs
  - Tasks: task list, status pills, progress
  - Agents: agent status, colors, swarm management
  - Settings: configuration panels
  - Wizard: multi-step navigation
- **Dialog launchers** — the 100+ thin launcher functions in `dialogLaunchers.tsx`, why they exist (lazy loading, separation of concerns)
- **Hook inventory** — all 150+ hooks grouped by category:
  - Input: useTextInput, useArrowKeyHistory, usePasteHandler, useVimInput
  - Commands: useCommandKeybindings, useCommandQueue, useQueueProcessor
  - State: useSettings, useMainLoopModel, useAppState
  - Tools: useCanUseTool, useMergedTools, useTools
  - UI: useTerminalSize, useSelection, useVirtualScroll, useTimeout
  - Integration: useIDEIntegration, useInboxPoller, useRemoteSession
  - Advanced: useSwarmInitialization, useForkSubagent, useBackgroundTasks
  - Config: useDynamicConfig, useSettingsChange, useVoiceIntegration
  - Analytics: usePromptSuggestion, useSkillsChange, useUpdateNotification
- **The Wizard pattern** — multi-step navigation used across agent creation, installation, setup flows

#### `cost-tracking.md` — Token Accounting

Covers:
- **Cost state** — per-model token counts (input, output, cache read, cache creation), web search requests, API duration, tool duration
- **`calculateUSDCost`** — pricing per model
- **Session persistence** — `saveCurrentSessionCosts` / `restoreCostStateForSession` via project config
- **`useCostSummary` hook** — React integration, exit-time cost reporting
- **`formatTotalCost`** — chalk-styled display with per-model breakdowns
- **Billing access gating** — `hasConsoleBillingAccess` check

---

### Peripheral + Build + Reconstruction (3 pages)

#### `peripheral-systems.md` — Smaller Subsystems

Each gets a focused section:
- **Voice** — voice I/O wrapper, STT streaming, voice keyterms, `VOICE_MODE` feature gate
- **Vim mode** — motions, operators, text objects, transitions, state machine, `useVimInput` integration
- **Bridge** — remote session attachment, bridge permission callbacks, messaging socket, `BRIDGE_MODE` feature gate
- **Remote** — remote agent execution, teleport integration
- **Keybindings** — chord support, `~/.claude/keybindings.json` customization, command hook integration
- **Upstream proxy** — proxy relay for network routing
- **Buddy** — companion agent system
- **Migrations** — data/schema migration system for config evolution

#### `build-system.md` — The Reconstruction Pipeline

Covers:
- **Overview** — what this build does: extract source from `cli.js.map`, reconcile with npm, generate shims, patch flags, bundle with Bun
- **Source map extraction** — `prepareWorkspace()`: 4,756 modules, keepPaths set, `.prepared.json` caching (builderVersion + mtime + size)
- **Overlay dependency system** — ~80 npm packages, `ensureOverlayDependencies()`, stamp-based caching, `--legacy-peer-deps`
- **Shim generation** — four types:
  1. Source alias shims (`src/*` proxies via `generateSourceAliasShims`)
  2. Native package shims (NAPI → TypeScript via `generateNativePackageShims`)
  3. Package entry shims (missing `index.js` via `generatePackageEntryShims`)
  4. Package subpath shims (deep imports via `generatePackageSubpathShims`)
  - The proxy module template (3-line re-export)
- **Stub generation** — `generateMissingLocalStubs()`, `inferStubExports()`, `renderStubExpression()` smart value inference
- **Feature flag patching** — `patchFeatureFlags()`: replacing `import { feature } from 'bun:bundle'` with runtime array check
- **Bun bundling** — `runBunBuild()`: target node, ESM, .md/.txt as text loaders, minification
- **Build retry loop** — 6-attempt reconciliation: auto-detecting missing packages, auto-adding stub exports
- **Finalization** — wrapper with shebang, localStorage polyfill, MACRO object, runtime vendor copy, chmod
- Mermaid diagram: full build pipeline flowchart

#### `reconstruction-fidelity.md` — What's Real, What's Missing

Covers:
- **Unavailable `@ant/*` packages** — the 4 stubbed packages and what each represents:
  - `@ant/claude-for-chrome-mcp` — Chrome extension MCP integration
  - `@ant/computer-use-input` — mouse/keyboard input (native addon available)
  - `@ant/computer-use-mcp` — computer use MCP server (partially reconstructed)
  - `@ant/computer-use-swift` — screen capture/app management (native addon available)
- **Reconstructed type-only modules** — `executor.ts`, `subGates.ts` manually reconstructed from type signatures
- **Native addon status** — which `.node` binaries are included, which work, which don't
- **Feature flag inventory** — full catalog of ~90 known flags, which 4 are enabled, what disabled ones likely control
- **Missing source content** — modules with `null` sourcesContent
- **Vendor packages** — 4 vendored native modules and their reconstruction status
- **Pinned React files** — why 4 React production files must come from source map, not npm
- **Overall fidelity assessment** — honest summary of reconstruction coverage

---

## Cross-Cutting Conventions

All subsystem pages follow this structure:
1. **Context preamble** — 2-3 sentences positioning the subsystem within the larger architecture, with a link back to the overview
2. **Key files** — table of primary source files with paths and one-line descriptions
3. **Architecture** — how the subsystem works internally
4. **Interfaces** — how other subsystems interact with this one
5. **Feature gates** — any feature flags that affect this subsystem
6. **Mermaid diagrams** — where they add clarity (not every page needs one)

Cross-links use relative Markdown links: `[query engine](query-engine.md)`.

Mermaid diagrams use GitHub-compatible fenced blocks:
````
```mermaid
graph TD
  A --> B
```
````

---

## What This Wiki Is Not

- Not a user guide (doesn't explain how to use Claude Code)
- Not API documentation (doesn't document public interfaces for consumers)
- Not a changelog (doesn't track what changed between versions)
- Not a build guide (the README already covers how to build)

It is a **technical architecture reference** for understanding how Claude Code works internally.
