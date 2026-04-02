# Claude Code Architecture Wiki — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Produce an 18-page Markdown wiki documenting the internal architecture of Claude Code v2.1.88, reconstructed from source maps.

**Architecture:** Hybrid wiki structure — a narrative spine (`architecture-overview.md`) tells the full system story top-down with Mermaid diagrams, linking to self-contained deep-dive reference pages per subsystem. All pages live in `docs/wiki/`. Cross-links use relative Markdown links. Diagrams use GitHub-compatible Mermaid fenced blocks.

**Tech Stack:** Markdown, Mermaid diagrams, GitHub-flavored Markdown

**Spec:** `docs/superpowers/specs/2026-04-01-codebase-wiki-design.md`

**Audience:** Senior engineer encountering the codebase for the first time. Assumes TypeScript/React/CLI knowledge, explains domain concepts (MCP, Ink, source maps).

---

## File Structure

All files created in `docs/wiki/`:

| File | Responsibility |
|------|---------------|
| `index.md` | Master index, reading guide, page table |
| `architecture-overview.md` | Narrative spine with 5 Mermaid diagrams |
| `query-engine.md` | QueryEngine state machine, conversation loop, compaction |
| `tool-system.md` | Tool type anatomy, ToolUseContext, execution lifecycle |
| `tool-catalog.md` | All 47+ tools cataloged by category |
| `command-system.md` | Command type, registration, dispatch, lazy loading |
| `command-catalog.md` | All 87+ commands cataloged by category |
| `state-management.md` | Three-tier state: bootstrap, Zustand store, React contexts |
| `ink-tui.md` | Custom Ink fork: DOM, layout, rendering, input, hooks |
| `permissions-security.md` | Permission modes, resolution flow, bash classifier |
| `mcp-integration.md` | MCP server lifecycle, tool bridging, auth, resources |
| `task-system.md` | 7 task variants, lifecycle, coordinator mode |
| `plugin-skill-system.md` | Plugin loading, skill discovery, SkillTool |
| `service-layer.md` | API client, compaction, analytics, OAuth, LSP |
| `component-layer.md` | Screens, component inventory, 85+ hooks, dialog launchers |
| `cost-tracking.md` | Token accounting, per-model pricing, session persistence |
| `peripheral-systems.md` | Voice, Vim, Bridge, Remote, Keybindings, Buddy, Migrations |
| `build-system.md` | Source map extraction, shims, feature flags, Bun bundling |
| `reconstruction-fidelity.md` | Stubbed packages, native addons, feature flag inventory |

---

## Task Dependency Order

Tasks must execute in this order because later pages reference earlier ones via cross-links, and the narrative spine references all subsystem pages:

1. **Task 1**: index.md (skeleton — final TOC descriptions updated at end)
2. **Tasks 2-5**: Core engine pages (query-engine, tool-system, tool-catalog, command-system + command-catalog)
3. **Tasks 6-8**: Infrastructure pages (state-management, ink-tui, permissions-security)
4. **Tasks 9-14**: Subsystem pages (mcp, tasks, plugins, services, components, cost)
5. **Task 15**: peripheral-systems.md
6. **Tasks 16-17**: Build & reconstruction pages
7. **Task 18**: architecture-overview.md (written last — references all other pages)
8. **Task 19**: Finalize index.md with accurate descriptions, final commit

---

## Cross-Cutting Conventions

Every subsystem page (Tasks 2-17) follows this template structure:

```markdown
# [Page Title]

> [2-3 sentence context preamble positioning within the architecture, with link back to overview]

## Key Files

| File | Purpose |
|------|---------|
| `source/src/path/to/file.ts` | One-line description |

## [Architecture sections — specific to each page]

## Interfaces

How other subsystems interact with this one.

## Feature Gates

Any feature flags affecting this subsystem (if applicable).
```

The agent writing each page MUST:
1. Read the actual source files listed in the "Research context" section before writing
2. Verify all type names, export names, and file paths against the actual source
3. Use relative Markdown links for cross-references: `[query engine](query-engine.md)`
4. Use Mermaid fenced blocks where specified: ` ```mermaid `
5. Not invent details — if something is unclear from source, say "implementation details are in the compiled bundle"

---

## Tasks

### Task 1: Index Skeleton

**Files:**
- Create: `docs/wiki/index.md`

- [ ] **Step 1: Create docs/wiki/ directory and index.md**

Create the master index with placeholder descriptions. The reading guide and page table structure are final; one-line descriptions will be refined in Task 19 after all pages exist.

```markdown
# Claude Code v2.1.88 — Architecture Wiki

An internal architecture reference for Claude Code, reconstructed from published source maps. This wiki documents how the CLI works — its conversation engine, tool system, state management, UI framework, and more.

**This is not** a user guide, API reference, changelog, or build guide. It is a technical deep-dive for engineers who want to understand the internals.

**Version**: 2.1.88 | **Modules**: 4,756 | **Source**: Extracted from `cli.js.map`

---

## Reading Guide

**First time?** Start here:

1. [Architecture Overview](architecture-overview.md) — the full system story in ~2,000 words with diagrams
2. [Query Engine](query-engine.md) — the conversation loop that drives everything
3. [Tool System](tool-system.md) — how tools are defined, registered, and executed
4. [State Management](state-management.md) — the three-tier state architecture

Then explore whatever interests you. Each page is self-contained.

---

## Pages

### Core Engine

| Page | Description |
|------|-------------|
| [Architecture Overview](architecture-overview.md) | Narrative spine — the full system story with Mermaid diagrams |
| [Query Engine](query-engine.md) | QueryEngine state machine, conversation loop, compaction strategies |
| [Tool System](tool-system.md) | Tool type anatomy, ToolUseContext, execution lifecycle |
| [Tool Catalog](tool-catalog.md) | Complete reference for all 47+ built-in tools |
| [Command System](command-system.md) | Command types, registration, dispatch, lazy loading |
| [Command Catalog](command-catalog.md) | Complete reference for all 87+ commands |

### Infrastructure

| Page | Description |
|------|-------------|
| [State Management](state-management.md) | Three-tier state: bootstrap globals, Zustand store, React contexts |
| [Ink TUI](ink-tui.md) | Custom Ink fork — DOM, Yoga layout, rendering pipeline, input system |
| [Permissions & Security](permissions-security.md) | Permission modes, trust model, bash safety classifier |

### Subsystems

| Page | Description |
|------|-------------|
| [MCP Integration](mcp-integration.md) | Model Context Protocol — servers, tools, resources, auth |
| [Task System](task-system.md) | Background tasks, agent spawning, teammates, coordinator mode |
| [Plugin & Skill System](plugin-skill-system.md) | Plugin architecture, skill discovery, SkillTool |
| [Service Layer](service-layer.md) | API client, compaction services, analytics, OAuth, LSP |
| [Component Layer](component-layer.md) | Screens, React components, 85+ hooks, dialog launchers |
| [Cost Tracking](cost-tracking.md) | Token accounting, per-model pricing, session persistence |

### Peripheral Systems & Build

| Page | Description |
|------|-------------|
| [Peripheral Systems](peripheral-systems.md) | Voice, Vim, Bridge, Remote, Keybindings, Buddy, Migrations |
| [Build System](build-system.md) | Source map extraction, overlay dependencies, Bun bundling |
| [Reconstruction Fidelity](reconstruction-fidelity.md) | What's real, what's stubbed, feature flag inventory |
```

- [ ] **Step 2: Commit**

```bash
git add docs/wiki/index.md
git commit -m "docs: add wiki index skeleton"
```

---

### Task 2: Query Engine

**Files:**
- Create: `docs/wiki/query-engine.md`

**Research context** — the agent MUST read these files before writing:
- `source/src/QueryEngine.ts` — QueryEngineConfig type, submitMessage() generator, instance state
- `source/src/query.ts` — main query loop, State type, stop conditions, compaction triggers, error recovery
- `source/src/context.ts` — getGitStatus(), getUserContext(), getSystemContext(), MAX_STATUS_CHARS
- `source/src/query/config.ts` — QueryConfig, gates
- `source/src/query/deps.ts` — QueryDeps injection
- `source/src/query/stopHooks.ts` — handleStopHooks, hook types
- `source/src/query/tokenBudget.ts` — BudgetTracker, checkTokenBudget
- `source/src/services/compact/autoCompact.ts` — thresholds, AUTOCOMPACT_BUFFER_TOKENS
- `source/src/services/compact/compact.ts` — CompactionResult, compactConversation

**Content requirements:**

The page must cover all of the following sections with accurate details from the source:

1. **Context preamble** — position as the heart of Claude Code, link to overview
2. **Key Files table** — all files listed in research context above
3. **QueryEngine State Machine** — QueryEngineConfig fields (all ~20), submitMessage() as async generator, key instance variables (mutableMessages, abortController, permissionDenials, totalUsage, discoveredSkillNames, loadedNestedMemoryPaths)
4. **The Main Query Loop** — the `query()` function's `while(true)` loop with State type fields (messages, toolUseContext, autoCompactTracking, maxOutputTokensRecoveryCount, hasAttemptedReactiveCompact, turnCount, transition, etc.). Document each continue site.
5. **Turn Mechanics** — what constitutes a turn, turnCount tracking, token budget carryover
6. **Stop Conditions** — max_turns, stop_hook_prevented, completed, aborted_streaming, aborted_tools, blocking_limit, prompt_too_long, image_error
7. **Compaction Strategies** — all four with thresholds:
   - Auto-compact: AUTOCOMPACT_BUFFER_TOKENS (13K), WARNING_THRESHOLD_BUFFER_TOKENS (20K), getAutoCompactThreshold()
   - Reactive compact: triggered on PTL 413, hasAttemptedReactiveCompact circuit breaker
   - Snip compact: HISTORY_SNIP feature gate, snipTokensFreed tracking
   - Microcompact: cached msgId-based truncation, CACHED_MICROCOMPACT feature gate
   - Also: session memory compaction (experimental), context collapse (CONTEXT_COLLAPSE)
8. **Token Estimation** — tokenCountWithEstimation(), effective window calculation (context_window - reserved_output - 13K buffer), CLAUDE_CODE_AUTO_COMPACT_WINDOW env override
9. **System Prompt Assembly** — context.ts: getGitStatus() parallel git commands, getUserContext() CLAUDE.md loading, getSystemContext() cache breaker, MAX_STATUS_CHARS=2000 truncation
10. **Post-Sampling Hooks** — fire-and-forget after model response
11. **Error Handling** — FallbackTriggeredError model switch, PTL recovery cascade (collapse drain → reactive compact → surface error), max_output_tokens escalation (3 attempts), ImageSizeError
12. **Feature Gates table** — REACTIVE_COMPACT, CONTEXT_COLLAPSE, HISTORY_SNIP, BG_SESSIONS, EXPERIMENTAL_SKILL_SEARCH, TEMPLATES, CACHED_MICROCOMPACT
13. **Mermaid diagram** — state machine showing the main loop's turn transitions, continue sites, and stop conditions

- [ ] **Step 1: Read source files and write query-engine.md**

The agent reads all listed source files, then writes the complete page following the content requirements above.

- [ ] **Step 2: Commit**

```bash
git add docs/wiki/query-engine.md
git commit -m "docs: add query engine wiki page"
```

---

### Task 3: Tool System

**Files:**
- Create: `docs/wiki/tool-system.md`

**Research context** — read these files:
- `source/src/Tool.ts` — Tool interface, ToolUseContext, ToolInputJSONSchema, ToolResult, ValidationResult, SetToolJSXFn, ToolPermissionContext, buildTool()
- `source/src/tools.ts` — getAllBaseTools(), getTools(), assembleToolPool(), getMergedTools(), feature-gated conditional imports
- `source/src/services/tools/StreamingToolExecutor.ts` — concurrent execution, TrackedTool, sibling abort
- `source/src/services/tools/toolExecution.ts` — runToolUse(), execution flow, error classification
- `source/src/services/tools/toolHooks.ts` — runPreToolUseHooks(), runPostToolUseHooks(), hook permission resolution
- `source/src/services/tools/toolOrchestration.ts` — runTools(), batch partitioning, concurrency control

**Content requirements:**

1. **Context preamble** — tools as Claude Code's interface with the world
2. **Key Files table**
3. **Tool Type Anatomy** — the full Tool interface from buildTool(): name, inputSchema, outputSchema, call(), description(), prompt(), checkPermissions(), userFacingName(), renderToolUseMessage(), renderToolResultMessage(), isConcurrencySafe(), isReadOnly(), isDestructive(), isEnabled(), validateInput(), and all optional methods. Show the buildTool() defaults.
4. **ToolUseContext Deep Dive** — all major fields grouped by category: options, abort, state access, file cache, UI/JSX, notifications, hooks, agent context, budgets, file history. Explain why it's so large (single context object threaded through all tool executions).
5. **Tool Registration** — getAllBaseTools() as source of truth, feature-gated conditional imports pattern, getTools() filtering (SIMPLE mode, REPL mode, deny rules, isEnabled), assembleToolPool() dedup and sort for prompt-cache stability.
6. **Execution Lifecycle** — full flow with Mermaid diagram:
   - runTools() → partitionToolCalls() → concurrent vs serial batches
   - runToolUse() → find tool → validate input → check permissions → pre-hooks → tool.call() → post-hooks → persist large results
   - StreamingToolExecutor: TrackedTool states (queued/executing/completed/yielded), sibling abort on Bash error, max concurrency (default 10, configurable via CLAUDE_CODE_MAX_TOOL_USE_CONCURRENCY)
7. **Permission Model** — ToolPermissionContext (mode, rules), checkPermissions() flow, deny rules filtering, hook-based permission overrides
8. **Result Budgeting** — maxResultSizeChars per tool (BashTool: 30K, FileEditTool: 100K, FileReadTool: Infinity), disk persistence for oversized results
9. **Agent Tool Subprocess Model** — how AgentTool spawns isolated agents with their own context, worktree isolation, background task auto-spawning
10. **Mermaid diagram** — tool execution pipeline from API response through permission check, hooks, execution, result formatting

- [ ] **Step 1: Read source files and write tool-system.md**
- [ ] **Step 2: Commit**

```bash
git add docs/wiki/tool-system.md
git commit -m "docs: add tool system wiki page"
```

---

### Task 4: Tool Catalog

**Files:**
- Create: `docs/wiki/tool-catalog.md`

**Research context** — read these files:
- `source/src/tools.ts` — full tool list from getAllBaseTools() and conditional imports
- Representative tool directories under `source/src/tools/` — read the main file in at least: BashTool, FileReadTool, FileEditTool, FileWriteTool, GlobTool, GrepTool, AgentTool, WebFetchTool, WebSearchTool, TaskCreateTool, EnterWorktreeTool, MCPTool, SkillTool, AskUserQuestionTool, LSPTool, NotebookEditTool

**Content requirements:**

1. **Context preamble** — reference to tool-system.md for architecture, this page is the reference catalog
2. **Per-tool entry format** — table rows or subsections with: Name, one-line purpose, key input parameters, permission requirements, notable behavior
3. **Grouped by category** — each category gets a heading with a brief intro:

   **Core I/O** (6 tools): BashTool, FileReadTool, FileEditTool, FileWriteTool, PowerShellTool, REPLTool
   **Search** (3): GlobTool, GrepTool, ToolSearchTool
   **AI/Web** (2): WebSearchTool, WebFetchTool
   **Task Management** (6): TaskCreateTool, TaskUpdateTool, TaskGetTool, TaskListTool, TaskStopTool, TaskOutputTool
   **Workspace** (4): EnterWorktreeTool, ExitWorktreeTool, EnterPlanModeTool, ExitPlanModeTool
   **MCP** (4): MCPTool, ListMcpResourcesTool, ReadMcpResourceTool, McpAuthTool
   **Collaboration** (4): AgentTool, SendMessageTool, TeamCreateTool, TeamDeleteTool
   **Scheduling** (2): CronCreateTool/CronDeleteTool/CronListTool, RemoteTriggerTool
   **Configuration** (2): ConfigTool, SkillTool
   **Special** (6+): AskUserQuestionTool, BriefTool, SleepTool, SyntheticOutputTool, NotebookEditTool, TodoWriteTool
   **Feature-Gated** (note): MonitorTool, PushNotificationTool, SendUserFileTool, SubscribePRTool, WebBrowserTool, TerminalCaptureTool, WorkflowTool, etc. — list with their feature gates

4. **Feature gate summary table** at the bottom mapping each conditional tool to its feature flag

- [ ] **Step 1: Read tool source files and write tool-catalog.md**
- [ ] **Step 2: Commit**

```bash
git add docs/wiki/tool-catalog.md
git commit -m "docs: add tool catalog wiki page"
```

---

### Task 5: Command System + Command Catalog

**Files:**
- Create: `docs/wiki/command-system.md`
- Create: `docs/wiki/command-catalog.md`

**Research context** — read these files:
- `source/src/commands.ts` — COMMANDS() array, loadAllCommands(), findCommand(), getCommands(), cache management, filtering (REMOTE_SAFE_COMMANDS, BRIDGE_SAFE_COMMANDS)
- `source/src/types/command.ts` — CommandBase, PromptCommand, LocalCommand, LocalJSXCommand, LocalCommandResult
- 3-4 representative command directories under `source/src/commands/` (e.g., clear, help, review, compact)

**Content requirements for command-system.md:**

1. **Context preamble**
2. **Key Files table**
3. **Command Types** — the three discriminated union variants:
   - `PromptCommand` (type: 'prompt'): getPromptForCommand(), model-invocable skills, progressMessage, contentLength, source, context ('inline' | 'fork'), allowedTools, hooks
   - `LocalCommand` (type: 'local'): load() lazy loader, call() returns LocalCommandResult (text | compact | skip)
   - `LocalJSXCommand` (type: 'local-jsx'): load() lazy loader, call(onDone, context, args) returns React.ReactNode
4. **CommandBase properties** — all shared fields: name, aliases, description, isEnabled, isHidden, availability, argumentHint, whenToUse, loadedFrom, kind, immediate, isSensitive, userInvocable, disableModelInvocation
5. **Registration Architecture** — COMMANDS() memoized array, 87 direct imports, feature-gated lazy imports, loadAllCommands() combining: built-in + skill dirs + plugin commands + plugin skills + bundled skills + workflow commands
6. **Dispatch Pipeline** — getCommands(cwd) → loadAllCommands() → availability filter → isEnabled filter → dedup → findCommand(name)
7. **Lazy Loading Pattern** — how large commands defer imports until invocation, the lazy shim pattern
8. **Feature-Gated Commands** — how feature() gates hide commands entirely (e.g., BRIDGE_MODE, VOICE_MODE, ULTRAPLAN, TORCH, KAIROS)
9. **Filtering Utilities** — REMOTE_SAFE_COMMANDS (17), BRIDGE_SAFE_COMMANDS (6), filterCommandsForRemoteMode()
10. **Skill Loading Pipeline** — getSkillToolCommands(), getSlashCommandToolSkills(), getMcpSkillCommands()

**Content requirements for command-catalog.md:**

1. **Context preamble** — reference to command-system.md for architecture
2. **Per-command entry format** — Name, type (prompt/local/local-jsx), one-line description, aliases, feature gate (if any)
3. **Grouped by category** with headings — list ALL 87+ commands. Categories:
   - **Session Management**: init, login, logout, resume, session, teleport, clear, rename, tag, fork, exit
   - **Workspace**: add-dir, memory, context, env, config, theme, color, files
   - **Development & Git**: commit, commit-push-pr, diff, review, ultrareview, doctor, ide, desktop, mobile, branch, autofix-pr, pr_comments
   - **Productivity**: cost, usage, status, summary, tasks, effort, output-style
   - **Configuration**: model, fast, passes, permissions, hooks, keybindings, rate-limit-options, privacy-settings, reset-limits
   - **Integration**: skills, plugin, reload-plugins, mcp, install-github-app, install-slack-app, chrome
   - **Analysis**: insights, ctx_viz
   - **Workflow**: plan, compact, rewind, workflows
   - **Advanced** (feature-gated): brief/KAIROS, proactive, assistant, bridge, voice, buddy, remote-setup, remote-control-server, subscribe-pr
   - **Internal/Ant-only**: ant-trace, bughunter, security-review, agents-platform, ultraplan, torch, good-claude, backfill-sessions, break-cache, mock-limits, etc.
4. **Feature gate table** at bottom

- [ ] **Step 1: Read source files and write command-system.md**
- [ ] **Step 2: Read command directories and write command-catalog.md**
- [ ] **Step 3: Commit**

```bash
git add docs/wiki/command-system.md docs/wiki/command-catalog.md
git commit -m "docs: add command system and catalog wiki pages"
```

---

### Task 6: State Management

**Files:**
- Create: `docs/wiki/state-management.md`

**Research context** — read these files:
- `source/src/state/store.ts` — Store<T> type, createStore()
- `source/src/state/AppStateStore.ts` — AppState type (~60 fields), getDefaultAppState(), IDLE_SPECULATION_STATE
- `source/src/state/AppState.tsx` — AppStoreContext, AppStateProvider, useAppState()
- `source/src/state/onChangeAppState.ts` — state change handlers
- `source/src/state/selectors.ts` — getViewedTeammateTask(), getActiveAgentForInput()
- `source/src/context/` — all context providers: stats.tsx, notifications.tsx, fpsMetrics.tsx, mailbox.tsx, modalContext.tsx, overlayContext.tsx, promptOverlayContext.tsx, QueuedMessageContext.tsx, voice.tsx
- `source/src/bootstrap/state.ts` — if accessible (may be in compiled bundle)

**Content requirements:**

1. **Context preamble**
2. **Key Files table**
3. **Tier 1: Bootstrap State** — global singletons: session ID, project root, cwd tracking, cost/usage accumulation, telemetry meters/counters, OpenTelemetry providers, hook registration. Why it's separate from React state (needs to be accessible from services, CLI entry, non-React code).
4. **Tier 2: AppStateStore** — Store<T> implementation (subscribe, getState, setState with Object.is equality, onChange callback). AppState type with ALL ~60 fields grouped by category:
   - Session & Settings (settings, verbose, mainLoopModel, etc.)
   - UI State (expandedView, isBriefOnly, selectedIPAgentIndex, etc.)
   - Agent & Identity (agent, kairosEnabled, standaloneAgentContext)
   - Remote/Bridge (~15 replBridge* fields, remoteSessionUrl, etc.)
   - Tool Permissions (toolPermissionContext, activeOverlays)
   - Tasks & Agents (tasks map, agentNameRegistry, foregroundedTaskId, teamContext, etc.)
   - MCP & Plugins (mcp object, plugins object)
   - Notifications & UI Overlays (notifications, elicitation, promptSuggestion)
   - File & Attribution (fileHistory, attribution, todos)
   - Special Features (thinkingEnabled, speculation, fastMode, effortValue)
   - Computer Use & Sandbox (computerUseMcpState, workerSandboxPermissions)
5. **Tier 3: React Contexts** — table of all 9 context providers with their purpose, type, and key exports:
   - MailboxContext (lazy message queue)
   - PromptOverlayContext (floating overlay above prompt)
   - OverlayContext (Escape key coordination)
   - ModalContext (modal bounds)
   - QueuedMessageContext (message queue markers)
   - NotificationsContext (toast queue with priority)
   - StatsContext (metrics/counters/histograms)
   - FpsMetricsContext (performance)
   - VoiceContext (feature-gated)
6. **State Change Handlers** — onChangeAppState: what changes trigger side effects (permission mode sync, model persistence, expanded view, verbose, settings, tungsten panel)
7. **Selectors** — getViewedTeammateTask(), getActiveAgentForInput() with the ActiveAgentForInput union type
8. **Provider Nesting Order** — AppStateProvider → MailboxProvider → VoiceProvider → app tree
9. **Mermaid diagram** — three tiers with arrows showing read/write relationships

- [ ] **Step 1: Read source files and write state-management.md**
- [ ] **Step 2: Commit**

```bash
git add docs/wiki/state-management.md
git commit -m "docs: add state management wiki page"
```

---

### Task 7: Ink TUI

**Files:**
- Create: `docs/wiki/ink-tui.md`

**Research context** — read these files/directories:
- `source/src/ink/` — list all files, read: root.ts, dom.ts, focus.ts, render-to-screen.ts, render-node-to-output.ts, output.ts, frame.ts, styles.ts, parse-keypress.ts, hit-test.ts, colorize.ts, wrapAnsi.ts
- `source/src/ink/layout/` — engine.ts, yoga.ts, geometry.ts, node.ts
- `source/src/ink/components/` — App.tsx, Box.tsx, Text.tsx, Button.tsx, Link.tsx
- `source/src/ink/hooks/` — all files
- `source/src/ink/events/` — all files

**Content requirements:**

1. **Context preamble** — explain what Ink is (React for terminals) and that Claude Code ships a custom fork
2. **Key Files table** — organized by layer (DOM, layout, rendering, input, components, hooks)
3. **Why a Custom Fork** — likely reasons: performance, custom input handling (mouse hit testing, selection), features stock Ink doesn't support (clickable links, alternate screen, ANSI passthrough)
4. **DOM Layer** — dom.ts virtual DOM nodes, focus.ts focus management
5. **Layout Engine** — Yoga-based flexbox: engine.ts orchestration, yoga.ts bindings, geometry.ts calculations, node.ts node types
6. **Rendering Pipeline** — render-node-to-output.ts → render-to-screen.ts → ANSI escape sequences. How output.ts manages the output buffer.
7. **Component Library** — Box (flexbox container), Text (styled text), Button (clickable), Link (hyperlink with supports-hyperlinks detection), Spacer, Newline, App wrapper. AlternateScreen for buffer switching.
8. **Input System** — parse-keypress.ts keypress parsing, events/ directory (input events, click events, terminal focus events), hit-test.ts for mouse interaction
9. **ANSI Handling** — colorize.ts, wrapAnsi.ts for text wrapping with ANSI awareness, Ansi.tsx component for raw escape codes
10. **Frame Loop** — frame.ts render cycle scheduling, how renders are batched
11. **Custom Hooks** — table of all hooks in ink/hooks/: useAnimationFrame, useApp, useInput, useInterval, useSelection, useStdin, useTabStatus, useTerminalFocus, useTerminalTitle, useTerminalViewport

- [ ] **Step 1: Read source files and write ink-tui.md**
- [ ] **Step 2: Commit**

```bash
git add docs/wiki/ink-tui.md
git commit -m "docs: add Ink TUI wiki page"
```

---

### Task 8: Permissions & Security

**Files:**
- Create: `docs/wiki/permissions-security.md`

**Research context** — read these files:
- `source/src/Tool.ts` — ToolPermissionContext, getEmptyToolPermissionContext(), checkPermissions()
- `source/src/tools.ts` — filterToolsByDenyRules()
- `source/src/services/tools/toolExecution.ts` — permission check in runToolUse()
- `source/src/services/tools/toolHooks.ts` — hook-based permission overrides
- Files in `source/src/hooks/toolPermission/` if they exist
- Files in `source/src/utils/permissions/` if they exist
- Grep for `BASH_CLASSIFIER` in source/src/ to find the bash safety classifier

**Content requirements:**

1. **Context preamble**
2. **Key Files table**
3. **Permission Modes** — default (ask per tool), auto (approve all), bypass, plan, bubble, ungated_auto. When each is used.
4. **ToolPermissionContext** — mode, alwaysAllowRules, alwaysDenyRules, alwaysAskRules, isBypassPermissionsModeAvailable, isAutoModeAvailable
5. **Resolution Flow** — step by step: deny rules filtering (assembleToolPool) → tool-level checkPermissions() → pre-tool hooks → user prompt → cache decision. Show as Mermaid flowchart.
6. **Deny Rules** — how filterToolsByDenyRules works, MCP server prefix stripping
7. **Denial Tracking** — denialTracking.ts, how denied permissions avoid re-prompts
8. **Trust Dialogs** — TrustDialog component, trust-on-first-use patterns
9. **Bash Safety Classifier** — BASH_CLASSIFIER feature flag, how dangerous commands are detected, what triggers it
10. **Hook-Based Permissions** — pre-tool hooks that can approve/deny/modify, resolveHookPermissionDecision()
11. **Session vs. Persistent Permissions** — what resets between sessions, what persists in settings

- [ ] **Step 1: Read source files and write permissions-security.md**
- [ ] **Step 2: Commit**

```bash
git add docs/wiki/permissions-security.md
git commit -m "docs: add permissions and security wiki page"
```

---

### Task 9: MCP Integration

**Files:**
- Create: `docs/wiki/mcp-integration.md`

**Research context** — read these files:
- `source/src/services/mcp/client.ts` — fetchToolsForClient(), fetchCommandsForClient(), fetchResourcesForClient()
- `source/src/services/mcp/useManageMCPConnections.ts` — connection lifecycle hook
- `source/src/services/mcp/types.ts` — MCPServerConnection union, ScopedMcpServerConfig
- `source/src/services/mcp/config.ts` — configuration management
- `source/src/services/mcp/auth.ts` — authentication
- `source/src/services/mcp/elicitationHandler.ts` — elicitation flow
- `source/src/services/mcp/normalization.ts` — tool name normalization
- `source/src/services/mcp/claudeai.ts` — Claude AI proxy servers
- `source/src/services/mcp/vscodeSdkMcp.ts` — VS Code integration
- `source/src/services/mcp/xaa.ts` — XAA integration

**Content requirements:**

1. **Context preamble** — what MCP is (Model Context Protocol), why it matters for extensibility
2. **Key Files table** — all 23 files in services/mcp/
3. **Server Lifecycle** — MCPServerConnection union (Connected, Failed, NeedsAuth, Pending, Disabled), connection states, useManageMCPConnections hook, reconnection with exponential backoff (max 5 attempts), cleanup
4. **Transport Types** — stdio, sse, http, ws, sdk, claudeai-proxy, sse-ide, ws-ide
5. **Tool Bridging** — fetchToolsForClient() → MCPTool wrapping → assembleToolPool() merge with built-in tools. Name normalization (mcp__serverName__toolName).
6. **Resource System** — ListMcpResources, ReadMcpResource tools, how resources inject context into conversations
7. **Authentication** — OAuth flows, McpAuthTool, elicitation handler for interactive auth (-32042 error handling)
8. **Channel Permissions** — per-server permission scoping, channelPermissions.ts, channelAllowlist.ts
9. **Claude AI MCP Servers** — built-in proxy servers (Gmail, Google Calendar), claudeai.ts special handling
10. **XAA Integration** — Cross-App Access, xaaIdpLogin.ts identity provider flow
11. **VS Code SDK** — vscodeSdkMcp.ts, how IDE-hosted MCP servers connect via SdkControlTransport
12. **Configuration** — ScopedMcpServerConfig, config scopes (local/user/project/dynamic/enterprise/claudeai/managed), envExpansion.ts

- [ ] **Step 1: Read source files and write mcp-integration.md**
- [ ] **Step 2: Commit**

```bash
git add docs/wiki/mcp-integration.md
git commit -m "docs: add MCP integration wiki page"
```

---

### Task 10: Task System

**Files:**
- Create: `docs/wiki/task-system.md`

**Research context** — read these files:
- `source/src/tasks/types.ts` — TaskState union, TaskType enum, TaskStatus, isTerminalTaskStatus, isBackgroundTask
- `source/src/Task.ts` — TaskHandle, TaskContext, TaskStateBase
- `source/src/tasks.ts` — getAllTasks(), getTaskByType()
- Task variant directories: `source/src/tasks/DreamTask/`, `LocalAgentTask/`, `LocalShellTask/`, `RemoteAgentTask/`, `InProcessTeammateTask/`
- `source/src/tasks/stopTask.ts`, `source/src/tasks/pillLabel.ts`
- Grep for `COORDINATOR_MODE` in source/src/

**Content requirements:**

1. **Context preamble**
2. **Key Files table**
3. **Task State Union** — all 7 variants: LocalShell, LocalAgent, RemoteAgent, InProcessTeammate, LocalWorkflow, MonitorMcp, Dream. TaskType enum values, TaskStatus values.
4. **Task Lifecycle** — creation → pending → running → completed/failed/killed. isTerminalTaskStatus(), isBackgroundTask() (checks status + isBackgrounded flag).
5. **Per-Variant Details**:
   - LocalShellTask: shell command execution, progress tracking, guards.ts safety, killShellTasks.ts cleanup
   - LocalAgentTask: spawned agent subprocess, context isolation
   - InProcessTeammateTask: same-process teammate, transcript viewer
   - RemoteAgentTask: remote agent coordination
   - DreamTask: speculative background tasks from auto-dream service
   - LocalWorkflowTask: WORKFLOW_SCRIPTS feature gate
   - MonitorMcpTask: MONITOR_TOOL feature gate
6. **Task UI** — status pills (pillLabel.ts), task list component, shell progress display, foregrounding (foregroundedTaskId)
7. **Task Tools** — how TaskCreate/Update/Get/List/Stop/Output tools interact with the task state in AppState
8. **Coordinator Mode** — COORDINATOR_MODE feature gate, multi-agent coordination, getCoordinatorUserContext()
9. **Feature Gates** — COORDINATOR_MODE, WORKFLOW_SCRIPTS, MONITOR_TOOL, KAIROS_DREAM

- [ ] **Step 1: Read source files and write task-system.md**
- [ ] **Step 2: Commit**

```bash
git add docs/wiki/task-system.md
git commit -m "docs: add task system wiki page"
```

---

### Task 11: Plugin & Skill System

**Files:**
- Create: `docs/wiki/plugin-skill-system.md`

**Research context** — read these files:
- `source/src/plugins/builtinPlugins.ts`
- `source/src/plugins/bundled/` — list contents
- `source/src/skills/bundledSkills.ts`
- `source/src/skills/loadSkillsDir.ts`
- `source/src/skills/mcpSkillBuilders.ts`
- `source/src/tools/SkillTool/` — main file
- `source/src/commands.ts` — loadAllCommands() for how plugins/skills are combined
- `source/src/types/plugin.ts` if it exists
- Grep for `clearPluginCommandCache` and `clearPluginSkillsCache` in source/src/

**Content requirements:**

1. **Context preamble**
2. **Key Files table**
3. **Plugin Architecture** — plugin type definition, how plugins register commands and skills, loading pipeline, caching
4. **Bundled Plugins** — what ships built-in, builtinPlugins.ts contents
5. **Plugin Command Loading** — getPluginCommands(), getPluginSkills() from commands.ts, how they merge with built-in commands
6. **Skill System** — skill discovery from directories, loadSkillsDir.ts, bundled skills from bundledSkills.ts, mcpSkillBuilders.ts for MCP-sourced skills
7. **SkillTool** — how the tool invokes skills dynamically, getPromptForCommand() expansion, inline vs fork execution context
8. **Skill Types** — PromptCommand fields for skills: source, pluginInfo, skillRoot, context, agent, hooks, paths
9. **Cache Invalidation** — clearPluginCommandCache(), clearPluginSkillsCache(), clearCommandsCache() nuclear option

- [ ] **Step 1: Read source files and write plugin-skill-system.md**
- [ ] **Step 2: Commit**

```bash
git add docs/wiki/plugin-skill-system.md
git commit -m "docs: add plugin and skill system wiki page"
```

---

### Task 12: Service Layer

**Files:**
- Create: `docs/wiki/service-layer.md`

**Research context** — read these files:
- `source/src/services/api/` — bootstrap.ts, claude.ts, client.ts, withRetry.ts, errors.ts, logging.ts, dumpPrompts.ts
- `source/src/services/compact/` — autoCompact.ts, compact.ts, reactiveCompact.ts (if exists), snipCompact.ts (if exists)
- `source/src/services/analytics/` — index.ts, growthbook.ts
- `source/src/services/` — list all subdirectories to find: oauth/, lsp/, plugins/, mcp/ (covered separately), tools/ (covered separately), and any others

**Content requirements:**

1. **Context preamble**
2. **Key Files table** — organized by service
3. **API Client** — bootstrap.ts initialization, client.ts HTTP construction, withRetry.ts retry logic + FallbackTriggeredError, logging.ts usage tracking + EMPTY_USAGE constant, errors.ts error categorization + PROMPT_TOO_LONG_ERROR_MESSAGE, dumpPrompts.ts for debugging
4. **Compaction Services** — summarize (detailed coverage is in query-engine.md): autoCompact thresholds and tracking, compact.ts CompactionResult type and compactConversation() orchestration, POST_COMPACT_MAX_FILES_TO_RESTORE (5), POST_COMPACT_TOKEN_BUDGET (50K), reactive compact for PTL recovery, snip compact for history truncation
5. **Analytics** — event logging with PII verification, GrowthBook feature gating/experimentation, growthbook.ts integration
6. **OAuth** — authentication flows for external services
7. **LSP Integration** — Language Server Protocol for code intelligence, what it enables
8. **Team Memory Sync** — cross-session memory synchronization
9. **Remote Managed Settings** — MDM-style settings distribution
10. **Auto-Dream** — background speculative task automation, how DreamTasks are spawned
11. **Rate Limiting & Policy** — request throttling, usage policy enforcement
12. **Session Memory** — per-session state persistence
13. **Prompt Suggestions** — intelligent suggestion generation, integration with useCostSummary hook

- [ ] **Step 1: Read source files and write service-layer.md**
- [ ] **Step 2: Commit**

```bash
git add docs/wiki/service-layer.md
git commit -m "docs: add service layer wiki page"
```

---

### Task 13: Component Layer

**Files:**
- Create: `docs/wiki/component-layer.md`

**Research context** — read/list these files:
- `source/src/screens/` — Doctor.tsx, REPL.tsx, ResumeConversation.tsx
- `source/src/components/` — list ALL subdirectories (31+)
- `source/src/hooks/` — list ALL files (85+)
- `source/src/dialogLaunchers.tsx` — scan for all launcher function names
- `source/src/components/wizard/` — if exists, read for wizard pattern

**Content requirements:**

1. **Context preamble** — the UI layer built on the custom Ink fork (link to ink-tui.md)
2. **Key Files table** — screens and major component directories
3. **Screen Components** — REPL.tsx (main interactive), Doctor.tsx (diagnostics), ResumeConversation.tsx (session picker). Brief description of each.
4. **Component Inventory** — table of all 31+ component subdirectories with one-line descriptions:
   - Design system (design-system/, ui/)
   - Messages (messages/)
   - Code display (HighlightedCode/, StructuredDiff/, diff/)
   - User input (PromptInput/, CustomSelect/)
   - MCP (mcp/)
   - Skills (skills/)
   - Tasks (tasks/)
   - Teams (teams/)
   - Permissions (permissions/, TrustDialog/)
   - Agents (agents/)
   - Settings (Settings/)
   - Shell (shell/)
   - Sandbox (sandbox/)
   - Wizard (wizard/)
   - Help (HelpV2/)
   - Spinner (Spinner/)
   - Logo (LogoV2/)
   - Memory (memory/)
   - etc.
5. **Dialog Launchers** — explain the pattern in dialogLaunchers.tsx: 100+ thin launcher functions that lazy-import and render one-off dialog JSX. List representative examples. Explain why (lazy loading, separation of concerns, keeps main.tsx manageable).
6. **Hook Inventory** — complete table of all 85+ hooks grouped by category, with one-line purpose for each:
   - Input handling (useTextInput, useArrowKeyHistory, usePasteHandler, useVimInput, useSearchInput, useInputBuffer)
   - Commands (useCommandKeybindings, useCommandQueue, useQueueProcessor)
   - State (useSettings, useMainLoopModel, useDynamicConfig, useSettingsChange)
   - Tools (useCanUseTool, useMergedTools, useMergedCommands, useMergedClients)
   - UI (useTerminalSize, useVirtualScroll, useBlink, useAfterFirstRender, useMinDisplayTime, useTimeout, useElapsedTime, useTypeahead)
   - IDE (useIDEIntegration, useIdeAtMentioned, useIdeConnectionStatus, useIdeLogging, useIdeSelection)
   - Notifications (notifs/ directory, useInboxPoller, useIssueFlagBanner, useUpdateNotification)
   - Keybindings (useGlobalKeybindings, useKeybinding, useExitOnCtrlCD, useDoublePress)
   - Collaboration (useRemoteSession, useSwarmInitialization, useSwarmPermissionPoller, useTeammateViewAutoExit, useInProcessTeammateTask, useMailboxBridge)
   - Voice (useVoice, useVoiceEnabled, useVoiceIntegration)
   - Suggestions (usePromptSuggestion, usePluginRecommendationBase, useLspPluginRecommendation, useClaudeCodeHintRecommendation)
   - Session (useSessionBackgrounding, useSSHSession, useScheduledTasks, useReplBridge)
   - Data (useDiffData, useTurnDiffs, useDiffInIDE, useMemoryUsage, useFileHistorySnapshotInit)
   - Copy/Clipboard (useCopyOnSelect, useClipboardImageHint)
   - Analytics (useSkillsChange, useSkillImprovementSurvey, usePrStatus)
7. **The Wizard Pattern** — multi-step navigation used for agent creation, installation, setup flows

- [ ] **Step 1: Read/list source files and write component-layer.md**
- [ ] **Step 2: Commit**

```bash
git add docs/wiki/component-layer.md
git commit -m "docs: add component layer wiki page"
```

---

### Task 14: Cost Tracking

**Files:**
- Create: `docs/wiki/cost-tracking.md`

**Research context** — read these files:
- `source/src/cost-tracker.ts`
- `source/src/costHook.ts`

**Content requirements:**

1. **Context preamble**
2. **Key Files table**
3. **Cost State** — per-model token counts: input, output, cache read, cache creation. Web search requests. API duration. Tool duration.
4. **Exports** — getTotalCost(), getTotalDuration(), getTotalAPIDuration(), getTotalInputTokens(), getTotalOutputTokens(), getTotalCacheReadInputTokens(), getTotalCacheCreationInputTokens(), getTotalWebSearchRequests(), getTotalToolDuration(), getModelUsage(), getUsageForModel(), hasUnknownModelCost(), resetCostState()
5. **calculateUSDCost** — how per-model pricing works
6. **Session Persistence** — getStoredSessionCosts(sessionId), restoreCostStateForSession(sessionId), saveCurrentSessionCosts(fpsMetrics) via project config
7. **formatTotalCost** — chalk-styled display with per-model breakdowns
8. **useCostSummary Hook** — React integration, exit-time cost reporting, conditional on hasConsoleBillingAccess()
9. **addToTotalSessionCost** — how API usage accumulates

- [ ] **Step 1: Read source files and write cost-tracking.md**
- [ ] **Step 2: Commit**

```bash
git add docs/wiki/cost-tracking.md
git commit -m "docs: add cost tracking wiki page"
```

---

### Task 15: Peripheral Systems

**Files:**
- Create: `docs/wiki/peripheral-systems.md`

**Research context** — read/list these directories:
- `source/src/voice/` — voiceModeEnabled.ts
- `source/src/vim/` — motions.ts, operators.ts, textObjects.ts, transitions.ts, types.ts
- `source/src/bridge/` — list all 32 files, read: bridgeApi.ts, bridgeConfig.ts, bridgeMain.ts, bridgeMessaging.ts, bridgePermissionCallbacks.ts, initReplBridge.ts, types.ts
- `source/src/keybindings/` — list all 14 files, read: defaultBindings.ts, parser.ts, resolver.ts, schema.ts, match.ts
- `source/src/remote/` — all 4 files
- `source/src/buddy/` — all 6 files: companion.ts, CompanionSprite.tsx, prompt.ts, sprites.ts, types.ts, useBuddyNotification.tsx
- `source/src/migrations/` — all 12 migration files
- `source/src/upstreamproxy/` — all files

**Content requirements:**

Each subsystem gets its own section with: overview, key files, how it works, feature gate (if any).

1. **Voice** — voiceModeEnabled.ts, VOICE_MODE feature gate, voice I/O wrapper, STT streaming, voice keyterms integration with hooks (useVoice, useVoiceEnabled, useVoiceIntegration)
2. **Vim Mode** — 5 files: motions (cursor movement), operators (d/c/y actions), text objects (iw/aw/ip), transitions (mode changes: normal/insert/visual), types. How useVimInput hook integrates with text input.
3. **Bridge** — 32-file subsystem for remote CLI-to-web communication. bridgeApi.ts API interface, bridgeMessaging.ts message routing, replBridge transport layer, SessionsWebSocket, permission callbacks, JWT auth (jwtUtils.ts, workSecret.ts), trusted device management. BRIDGE_MODE feature gate.
4. **Remote** — RemoteSessionManager.ts session management, sdkMessageAdapter.ts, SessionsWebSocket.ts, remotePermissionBridge.ts
5. **Keybindings** — parser.ts parsing user keybinding definitions, resolver.ts resolving key sequences, match.ts matching keypress to bindings, schema.ts validation, defaultBindings.ts defaults, chord support, ~/.claude/keybindings.json customization, reservedShortcuts.ts
6. **Upstream Proxy** — proxy relay for network routing
7. **Buddy** — companion agent (Spindle): CompanionSprite.tsx rendering, sprites.ts sprite definitions, prompt.ts buddy prompting, useBuddyNotification.tsx
8. **Migrations** — list all 12 migration files with one-line purpose each. Pattern: model upgrades (Fennec→Opus, Opus→Opus1m, Sonnet45→Sonnet46), settings migrations (auto-updates, bypass permissions, MCP servers, REPL bridge)

- [ ] **Step 1: Read source files and write peripheral-systems.md**
- [ ] **Step 2: Commit**

```bash
git add docs/wiki/peripheral-systems.md
git commit -m "docs: add peripheral systems wiki page"
```

---

### Task 16: Build System

**Files:**
- Create: `docs/wiki/build-system.md`

**Research context** — read:
- `scripts/build-cli.mjs` — the entire 1637-line build script
- `README.md` — for build instructions context
- `source/package.json` — version, dependencies

**Content requirements:**

1. **Context preamble** — this is a reconstruction build, not Anthropic's original build system
2. **Key Files table** — build script, source map, overlays, native addons
3. **Overview** — what this build does in one paragraph: extract → reconcile → shim → patch → bundle
4. **Source Map Extraction** — prepareWorkspace(): parsing 4,756 modules from cli.js.map, the keepPaths Set, shouldSkipSourceMapWrite() for overlay packages, caching via .prepared.json (builderVersion=7 + mtime + size)
5. **Overlay Dependency System** — ~80 npm packages in baseOverlayDependencyPackages, ensureOverlayDependencies() with stamp-based caching, npm install flags (--no-package-lock, --ignore-scripts, --no-audit, --legacy-peer-deps), dynamic addition via extraOverlayPackages during error reconciliation
6. **Shim Generation** — four types with examples:
   - Source alias shims: src/* → node_modules/src/* proxies via generateSourceAliasShims()
   - Native package shims: nativePackageTargets Map (audio-capture-napi→vendor/audio-capture-src/index.ts, etc.)
   - Package entry shims: generatePackageEntryShims() for packages missing index.js, 11-candidate resolution priority
   - Package subpath shims: generatePackageSubpathShims() for deep imports, 7-candidate resolution
   - The proxy module template (3-line re-export pattern)
7. **Stub Generation** — generateMissingLocalStubs(), collectMissingLocalImports() scanning for broken imports, inferStubExports() smart detection (named imports, namespace imports, dynamic property access), renderStubExpression() pattern-based value inference:
   - is*/has*/should*/can* → `(..._args) => false`
   - get*/create*/build*/load* → `__makeStub(...)`
   - use* → `(..._args) => ({})` (React hooks)
   - *Dialog/*Component/*Panel → `() => null`
   - The Proxy-based __makeStub factory with nested stub chaining
8. **Feature Flag Patching** — patchFeatureFlags(): replacing `import { feature } from 'bun:bundle'` with `const feature = (flag) => ([...]).includes(flag)`. The enabledBundleFeatures Set (4 flags). ~117 flags in source.
9. **Bun Bundling** — runBunBuild() config: entry point (src/entrypoints/cli.tsx), target node, ESM format, .md/.txt text loaders, minification control, 256MB max buffer
10. **Build Retry Loop** — main() 6-attempt loop, reconcileBuildErrors() detecting "Could not resolve" → add to extraOverlayPackages, "No matching export" → add to stubExportAugmentations
11. **Finalization** — finalizeBuild(): wrapper with shebang, createBanner() (with the humorous source leak comment), localStorage polyfill (Map-backed), MACRO object (frozen publicMacroValues), copyRuntimeVendorAssets(), atomic rename, chmod 755
12. **Mermaid diagram** — full build pipeline flowchart: main loop → prepareWorkspace → ensureOverlayDeps → generateAugmentations → runBunBuild → success? → finalize / reconcile errors → retry

- [ ] **Step 1: Read build script and write build-system.md**
- [ ] **Step 2: Commit**

```bash
git add docs/wiki/build-system.md
git commit -m "docs: add build system wiki page"
```

---

### Task 17: Reconstruction Fidelity

**Files:**
- Create: `docs/wiki/reconstruction-fidelity.md`

**Research context** — read:
- `scripts/build-cli.mjs` — unavailableOverlayPackages, pinnedOverlaySourceMapPaths, nativePackageTargets, specialPackageTargets, restoreMissingSourceMapFiles(), ensureNativeAddonPrebuilds()
- `source/native-addons/` — list all .node files
- Grep for `feature('` in source/src/ to catalog all ~117 feature flags
- `scripts/build-cli.mjs` lines 376-458 for reconstructed files

**Content requirements:**

1. **Context preamble** — honest framing: source map extraction recovers most code but not everything
2. **Unavailable @ant/* Packages** — the 4 stubbed packages with detailed explanation:
   - `@ant/claude-for-chrome-mcp` (8 modules in source map) — Chrome extension MCP integration, fully stubbed with Proxy no-ops
   - `@ant/computer-use-input` (1 module) — mouse/keyboard input via native addon, native .node binary IS included
   - `@ant/computer-use-mcp` (10 modules) — computer use MCP server, partially reconstructed (executor.ts and subGates.ts manually rebuilt from type signatures)
   - `@ant/computer-use-swift` (1 module) — screen capture/app management via native addon, native .node binary IS included
   - How stubs work: ensureUnavailablePackageEntries() creates Proxy-based no-op modules with stubExportAugmentations for named exports
3. **Reconstructed Type-Only Modules** — two manually reconstructed files:
   - executor.ts: 7 interfaces (ScreenshotResult, DisplayGeometry, FrontmostApp, InstalledApp, RunningApp, ResolvePrepareCaptureResult, ComputerExecutor with 15 methods)
   - subGates.ts: ALL_SUB_GATES_OFF and ALL_SUB_GATES_ON constants with 6 boolean properties
4. **Native Addon Status** — table:
   - computer-use-swift.node — included, copied to prebuilds/, macOS only
   - computer-use-input.node — included, copied to prebuilds/, macOS only
   - image-processor.node — included (Sharp binding), cross-platform
   - audio-capture.node — included, platform-specific
   - Status: binaries are pre-built for macOS ARM64, may not work on other platforms
5. **Feature Flag Inventory** — complete table of all ~117 known flags organized by category:
   - Enabled in build (4): BUILDING_CLAUDE_APPS, BASH_CLASSIFIER, TRANSCRIPT_CLASSIFIER, CHICAGO_MCP
   - AI/Model features: ULTRATHINK, ABLATION_BASELINE, ANTI_DISTILLATION_CC, TOKEN_BUDGET, etc.
   - UI/UX features: AUTO_THEME, MESSAGE_ACTIONS, STREAMLINED_OUTPUT, HISTORY_PICKER, etc.
   - Platform features: VOICE_MODE, BRIDGE_MODE, COORDINATOR_MODE, FORK_SUBAGENT, etc.
   - Agent features: KAIROS, KAIROS_BRIEF, KAIROS_DREAM, KAIROS_CHANNELS, KAIROS_PUSH_NOTIFICATION, KAIROS_GITHUB_WEBHOOKS, BUDDY, etc.
   - Infrastructure: DAEMON, BG_SESSIONS, UDS_INBOX, DIRECT_CONNECT, SSH_REMOTE, CCR_AUTO_CONNECT, etc.
   - Compaction: REACTIVE_COMPACT, CONTEXT_COLLAPSE, HISTORY_SNIP, CACHED_MICROCOMPACT, COMPACTION_REMINDERS
   - Internal/Debug: DUMP_SYSTEM_PROMPT, BREAK_CACHE_COMMAND, OVERFLOW_TEST_TOOL, PERFETTO_TRACING, etc.
   - What enabling more flags would unlock
6. **Pinned React Files** — why 4 React production files come from source map, not npm: react.production.js, react-compiler-runtime.production.js, react-reconciler.production.js, react-reconciler-constants.production.js. Reason: Claude Code uses React Compiler, and the production builds in the source map include compiler-specific runtime code that doesn't match the standard npm versions.
7. **Vendor Packages** — 4 vendored native modules: image-processor-src, audio-capture-src, modifiers-napi-src, url-handler-src. Mapped via nativePackageTargets.
8. **Overall Fidelity Assessment** — honest summary:
   - Source code: ~100% recovered (4,756 modules with content, 0 null)
   - npm dependencies: ~100% (80 packages installed from registry)
   - Internal @ant packages: 4 unavailable, stubbed
   - Native addons: included for macOS, platform-limited
   - Feature flags: 4 of ~117 enabled (build-time choice, configurable)
   - Build system: fully functional reconstruction pipeline

- [ ] **Step 1: Read build script and source, write reconstruction-fidelity.md**
- [ ] **Step 2: Commit**

```bash
git add docs/wiki/reconstruction-fidelity.md
git commit -m "docs: add reconstruction fidelity wiki page"
```

---

### Task 18: Architecture Overview (Narrative Spine)

**Files:**
- Create: `docs/wiki/architecture-overview.md`

**This task MUST be done last** — it references every other page and its diagrams must accurately reflect the documented architecture.

**Research context** — re-read:
- All previously written wiki pages (skim for accuracy of cross-links)
- `source/src/main.tsx` — entry point
- `source/src/setup.ts` — initialization
- `source/src/replLauncher.tsx` — REPL launch

**Content requirements:**

~2,000 words + 5 Mermaid diagrams. Structure:

1. **Opening** — What Claude Code is (AI-powered CLI for software engineering), what this codebase is (v2.1.88 reconstructed from source maps), the 4,756-module application.

2. **Diagram 1: High-Level Architecture** — Mermaid flowchart showing ~8 boxes:
   ```
   User Input → CLI Entry (setup.ts, main.tsx)
     → Command Dispatch (commands.ts) / REPL (replLauncher.tsx)
     → Query Engine (QueryEngine.ts, query.ts)
     → Claude API (services/api/)
     → Tool Execution (tools/, services/tools/)
     → Ink TUI renders everything (ink/)
     → State Management feeds all layers (state/, bootstrap/)
     → Services underpin everything (services/)
   ```
   2-3 paragraphs: how a user interaction flows through the system. Link to [query engine](query-engine.md), [tool system](tool-system.md).

3. **Diagram 2: The Conversation Loop** — Mermaid sequence diagram showing one turn:
   ```
   User → QueryEngine: submitMessage(prompt)
   QueryEngine → query(): main loop
   query() → API: callModel(messages, tools)
   API → query(): streaming response
   query() → StreamingToolExecutor: tool_use blocks
   StreamingToolExecutor → Tool: call(args, context)
   Tool → StreamingToolExecutor: ToolResult
   StreamingToolExecutor → query(): tool results
   query() → query(): append results, continue loop
   query() → QueryEngine: return Terminal
   ```
   2-3 paragraphs: explain the loop, compaction triggers, stop conditions. Link to [query engine](query-engine.md).

4. **Diagram 3: Tool Execution Lifecycle** — Mermaid flowchart:
   ```
   API Response → Extract tool_use blocks
     → Partition: concurrent vs serial
     → For each tool:
       → Validate input → Check permissions → Pre-hooks
       → tool.call() → Post-hooks → Format result
     → Inject results into message history
   ```
   2-3 paragraphs: permission model, hook system. Link to [tool system](tool-system.md), [permissions](permissions-security.md).

5. **Diagram 4: State Architecture** — Mermaid block diagram showing three tiers:
   ```
   Tier 1: Bootstrap (global singletons) ← Services, CLI
   Tier 2: AppStateStore (~60 fields) ← React components, hooks
   Tier 3: React Contexts (9 providers) ← Specific UI subsystems
   ```
   2-3 paragraphs: why three tiers, what lives where. Link to [state management](state-management.md).

6. **Diagram 5: Module Dependency Map** — Mermaid flowchart showing subsystem dependencies:
   ```
   Foundational: State, Ink, Permissions
     ↑
   Core: Query Engine, Tool System, Command System
     ↑
   Services: API, Compaction, Analytics, MCP, OAuth
     ↑
   Features: Tasks, Plugins/Skills, Cost Tracking
     ↑
   Leaf: Voice, Vim, Bridge, Buddy, Remote
   ```
   2-3 paragraphs: foundational vs leaf, how to navigate the wiki. Link to all subsystem pages.

7. **Closing** — Reading guide recap, note about [build system](build-system.md) and [reconstruction fidelity](reconstruction-fidelity.md) for understanding the codebase's provenance.

- [ ] **Step 1: Read entry point files and all wiki pages, write architecture-overview.md**
- [ ] **Step 2: Commit**

```bash
git add docs/wiki/architecture-overview.md
git commit -m "docs: add architecture overview (narrative spine)"
```

---

### Task 19: Finalize Index and Final Review

**Files:**
- Modify: `docs/wiki/index.md`

- [ ] **Step 1: Review all wiki pages for broken cross-links**

Read every page and verify that all `[link text](page.md)` references point to existing pages with correct filenames.

- [ ] **Step 2: Update index.md descriptions**

Refine the one-line descriptions in the page tables to accurately reflect the final content of each page. Add any missing pages.

- [ ] **Step 3: Add .gitignore entry for .superpowers/**

```bash
echo '.superpowers/' >> .gitignore
```

- [ ] **Step 4: Final commit**

```bash
git add docs/wiki/index.md .gitignore
git commit -m "docs: finalize wiki index and cross-links"
```

---

## Self-Review Checklist

**Spec coverage:**
- [x] index.md — Task 1, 19
- [x] architecture-overview.md — Task 18
- [x] query-engine.md — Task 2
- [x] tool-system.md — Task 3
- [x] tool-catalog.md — Task 4
- [x] command-system.md — Task 5
- [x] command-catalog.md — Task 5
- [x] state-management.md — Task 6
- [x] ink-tui.md — Task 7
- [x] permissions-security.md — Task 8
- [x] mcp-integration.md — Task 9
- [x] task-system.md — Task 10
- [x] plugin-skill-system.md — Task 11
- [x] service-layer.md — Task 12
- [x] component-layer.md — Task 13
- [x] cost-tracking.md — Task 14
- [x] peripheral-systems.md — Task 15
- [x] build-system.md — Task 16
- [x] reconstruction-fidelity.md — Task 17

All 18 pages from the spec are covered. Cross-cutting conventions (template structure, cross-links, Mermaid format) are documented in the plan header.

**Placeholder scan:** No TBDs, TODOs, or "similar to Task N" references. Each task has specific research context files and content requirements.

**Type consistency:** File names, type names (QueryEngineConfig, ToolUseContext, AppState, MCPServerConnection, TaskState, etc.), and function names are consistent across all tasks, sourced from the same research data.
