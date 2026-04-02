# Component Layer

The Claude Code UI is built entirely on a [custom Ink fork](ink-tui.md) that renders React components to the terminal. There is no browser, no DOM -- every Box, Text, and input widget ultimately writes ANSI escape sequences to stdout via a Yoga-based layout engine. This page catalogues the screens, components, hooks, and dialog launcher infrastructure that make up the component layer.

---

## Key Files

| Path | Role |
|------|------|
| `source/src/screens/REPL.tsx` | Main conversation screen (~5,000 lines). Owns the message loop, tool permission flow, prompt input, and transcript view. |
| `source/src/screens/Doctor.tsx` | Diagnostic screen (`/doctor` command). Runs environment checks, version comparisons, lock audits. |
| `source/src/screens/ResumeConversation.tsx` | Session resume picker. Loads session logs, shows a selector, then mounts `REPL` with restored state. |
| `source/src/dialogLaunchers.tsx` | Thin async launcher functions that bridge imperative `main.tsx` code to React dialog components. |
| `source/src/components/wizard/` | Reusable multi-step wizard framework (provider, layout, navigation footer). |

---

## Screen Components

There are exactly three top-level screen components in `source/src/screens/`. Each is mounted by `main.tsx` (or by `dialogLaunchers.tsx`) and manages its own lifecycle:

### REPL.tsx

The heart of the application. At ~5,000 compiled lines it is by far the largest single file. It owns:

- **Message state** -- the `messages` array, conversation ID, abort controllers.
- **Query orchestration** -- calling `query()`, processing streaming tool-use blocks, handling compaction.
- **Permission flow** -- rendering `PermissionRequest` when a tool needs approval, wiring `ToolUseConfirm` callbacks.
- **Prompt input** -- the `PromptInput` component plus stashed prompts, vim mode, voice integration, paste handling.
- **Transcript view** -- toggling between `prompt` and `transcript` screens, search highlighting, virtual scroll.
- **Hook wiring** -- it consumes dozens of hooks (keybindings, IDE integration, swarm initialization, voice, notifications, etc.) and passes their outputs down as props.
- **Fullscreen mode** -- conditionally wraps everything in `AlternateScreen` for fullscreen terminal rendering with mouse tracking.

Its top-level render returns a `KeybindingSetup` wrapper containing an `MCPConnectionManager`, a `Box`-based layout with the message list / virtual scroll area, spinners, permission dialogs, the prompt input, and optionally side panels for tasks and companion.

### Doctor.tsx

Mounted by the `/doctor` command. A self-contained diagnostic screen that:

- Fetches npm/GCS dist-tags to compare the installed version against latest/stable.
- Runs `getDoctorDiagnostic()` to collect environment info (OS, shell, node, model, API config).
- Checks context warnings, settings validation errors, MCP parsing warnings, keybinding warnings.
- Displays sandbox health via `SandboxDoctorSection`.
- Audits PID-based version locks.
- Enumerates active agents and their sources.
- Renders everything in `Pane` containers with `PressEnterToContinue` at the bottom.

### ResumeConversation.tsx

The session resume picker, mounted when the user runs `claude --resume` or uses the `/resume` command. It:

1. Calls `loadSameRepoMessageLogsProgressive()` to incrementally load session logs for the current repo.
2. Renders a `LogSelector` (a searchable list of past sessions) with progressive loading (`loadMoreLogs` callback).
3. Supports switching to all-project mode via `loadAllProjectsMessageLogsProgressive()`.
4. On selection, calls `loadConversationForResume()` to deserialize messages, restores file history, content replacements, agent definitions, and worktree state.
5. Handles cross-project resume (prompting with `checkCrossProjectResume`), PR filtering, fork-session mode, and session renaming.
6. Once resume data is loaded, mounts `REPL` with the restored messages and state.

---

## Component Inventory

All component subdirectories live under `source/src/components/`. There are 31 subdirectories:

| Directory | Description |
|-----------|-------------|
| `agents/` | Agent management UI -- list, detail, editor, color picker, model/tool selectors, `new-agent-creation/` wizard with 12 step components. |
| `ClaudeCodeHint/` | Plugin hint menu -- recommends Claude Code plugins to install. |
| `CustomSelect/` | Custom select input components -- single select, multi-select (`SelectMulti`), navigation and state hooks. Replaces Ink's built-in select. |
| `design-system/` | Foundational UI primitives -- `Dialog`, `Pane`, `Divider`, `Byline`, `Tabs`, `FuzzyPicker`, `ProgressBar`, `Ratchet`, `StatusIcon`, `ListItem`, `LoadingState`, `ThemedBox/Text`, `ThemeProvider`, `KeyboardShortcutHint`, color utilities. |
| `DesktopUpsell/` | Desktop app upsell prompt shown at startup. |
| `diff/` | Diff viewer dialog -- `DiffDialog`, `DiffDetailView`, `DiffFileList`. |
| `FeedbackSurvey/` | Feedback survey system -- survey views, transcript sharing, frustration detection, post-compact surveys, memory surveys, debounced digit input. |
| `grove/` | Grove component -- a standalone `Grove.tsx` renderer. |
| `HelpV2/` | Help overlay -- general help, command listing (`Commands.tsx`, `General.tsx`, `HelpV2.tsx`). |
| `HighlightedCode/` | Syntax-highlighted code rendering with fallback. |
| `hooks/` | Hook configuration UI -- `HooksConfigMenu`, `PromptDialog`, event/hook/matcher mode selectors, hook viewer. |
| `LogoV2/` | Startup logo and welcome screen -- animated asterisk/Clawd, condensed logo, welcome text, feed columns, channels notice, various upsell notices. |
| `LspRecommendation/` | LSP plugin recommendation menu. |
| `ManagedSettingsSecurityDialog/` | Security dialog for managed (enterprise) settings with utility helpers. |
| `mcp/` | MCP (Model Context Protocol) UI -- server settings, tool list/detail views, elicitation dialog, capabilities section, reconnect, parsing warnings, stdio/remote/agent server menus. |
| `memory/` | Memory management UI -- `MemoryFileSelector`, `MemoryUpdateNotification`. |
| `messages/` | Message rendering -- 30+ message components for every message type: user text/image/command/bash, assistant text/thinking/tool-use, system errors, rate limits, task assignments, hook progress, plan approval, compact boundaries, and more. |
| `Passes/` | Guest passes UI. |
| `permissions/` | Permission request dialogs -- specialized renderers for bash, file edit, file write, filesystem, sed edit, notebook edit, shell, web fetch, computer use, skill, plan mode entry/exit, sandbox, plus shared helpers (`PermissionDialog`, `PermissionPrompt`, `PermissionExplanation`, rule explanation). |
| `PromptInput/` | The main text input area -- `PromptInput.tsx` plus footer, suggestions, help menu, mode indicator, queued commands, stash notice, history search, shimmer effect, voice indicator, sandbox hint, paste handling, truncation. |
| `sandbox/` | Sandbox configuration UI -- config tab, dependencies tab, overrides tab, doctor section, top-level settings. |
| `Settings/` | Settings screens -- `Config.tsx`, `Settings.tsx`, `Status.tsx`, `Usage.tsx`. |
| `shell/` | Shell output rendering -- `OutputLine`, `ShellProgressMessage`, `ShellTimeDisplay`, `ExpandShellOutputContext`. |
| `skills/` | Skills menu (`SkillsMenu.tsx`). |
| `Spinner/` | Loading spinners -- multiple animation types (`FlashingChar`, `ShimmerChar`, `SpinnerGlyph`, `GlimmerMessage`), teammate spinner trees/lines, stalled animation, shimmer animation. |
| `StructuredDiff/` | Structured diff rendering with color diffing and fallback. |
| `tasks/` | Background task UI -- task list dialogs, status displays, detail dialogs for async agents, in-process teammates, remote sessions, dreams, shell tasks, tool activity rendering. |
| `teams/` | Teams UI -- `TeamsDialog`, `TeamStatus`. |
| `TrustDialog/` | Trust/safety dialog for project trust decisions. |
| `ui/` | Generic UI primitives -- `OrderedList`, `OrderedListItem`, `TreeSelect`. |
| `wizard/` | Multi-step wizard framework (see [The Wizard Pattern](#the-wizard-pattern) below). |

---

## Dialog Launchers

`source/src/dialogLaunchers.tsx` contains thin async launcher functions that bridge the imperative startup code in `main.tsx` to React dialog components. The file exists because `main.tsx` needs to show modal UI (session pickers, trust dialogs, teleport prompts) before the main REPL is mounted, but those dialogs are React components that need a render root.

### The Pattern

Every launcher follows the same structure:

1. **Async function** -- returns a `Promise` of the dialog's result type.
2. **Dynamic import** -- `await import('./components/SomeDialog.js')` to lazy-load the component. This keeps the initial bundle lean; dialog code is only fetched when actually needed.
3. **`showSetupDialog` wrapper** -- calls `showSetupDialog<T>(root, done => <Component ... onComplete={done} />)` which renders the component into the Ink root and resolves the promise when the `done` callback fires.

This pattern exists for two reasons:
- **Startup performance** -- the dialogs are rarely needed, so lazy-importing them avoids loading their code (and transitive dependencies) on every launch.
- **Imperative/React bridge** -- `main.tsx` is largely imperative (async function with sequential steps). These launchers let it `await` a React dialog's result as a simple promise.

### Launcher Functions (7 total)

| Function | Component | Purpose |
|----------|-----------|---------|
| `launchSnapshotUpdateDialog` | `SnapshotUpdateDialog` | Agent memory snapshot update prompt (merge/keep/replace). |
| `launchInvalidSettingsDialog` | `InvalidSettingsDialog` | Shows settings validation errors, offers continue or exit. |
| `launchAssistantSessionChooser` | `AssistantSessionChooser` | Pick a bridge session to attach to (`claude assistant` mode). |
| `launchAssistantInstallWizard` | `NewInstallWizard` | Install wizard when no assistant sessions exist. |
| `launchTeleportResumeWrapper` | `TeleportResumeWrapper` | Interactive teleport session picker for remote resume. |
| `launchTeleportRepoMismatchDialog` | `TeleportRepoMismatchDialog` | Pick a local checkout when the remote repo does not match. |
| `launchResumeChooser` | `ResumeConversation` | Full session resume picker. Uses `renderAndRun` instead of `showSetupDialog`, wrapping in `<App><KeybindingSetup>`. |

`launchResumeChooser` is the outlier: it uses `renderAndRun` rather than `showSetupDialog` because the resume flow needs the full `App` provider tree (state store, FPS metrics, keybindings) rather than a bare dialog.

---

## Hook Inventory

Hooks live in two locations:
- `source/src/hooks/` -- 82 files at the top level, plus two subdirectories
- `source/src/hooks/notifs/` -- 16 notification-related hooks
- `source/src/hooks/toolPermission/` -- permission context, logging, and 3 handler strategies

This gives roughly 100+ hook modules. They are grouped by category below.

### Input Handling

| Hook | File | Purpose |
|------|------|---------|
| `useTextInput` | `useTextInput.ts` | Core text input state and key handling. |
| `useInputBuffer` | `useInputBuffer.ts` | Buffered input accumulation for paste detection. |
| `usePasteHandler` | `usePasteHandler.ts` | Detects and processes pasted content (images, text refs). |
| `useVimInput` | `useVimInput.ts` | Vim-mode key bindings for the prompt input. |
| `useSearchInput` | `useSearchInput.ts` | Search mode input handling (within transcript). |
| `useHistorySearch` | `useHistorySearch.ts` | Reverse-search through command history. |
| `useArrowKeyHistory` | `useArrowKeyHistory.tsx` | Up/down arrow to cycle through input history. |
| `useTypeahead` | `useTypeahead.tsx` | Typeahead/autocomplete suggestions. |

### Commands & Queue

| Hook | File | Purpose |
|------|------|---------|
| `useCommandQueue` | `useCommandQueue.ts` | Queues slash commands for sequential execution. |
| `useQueueProcessor` | `useQueueProcessor.ts` | Processes queued commands one at a time. |
| `useMergedCommands` | `useMergedCommands.ts` | Merges built-in commands with MCP/plugin commands. |

### State & Configuration

| Hook | File | Purpose |
|------|------|---------|
| `useSettings` | `useSettings.ts` | Reads and watches settings changes. |
| `useSettingsChange` | `useSettingsChange.ts` | Reacts to settings file changes on disk. |
| `useDynamicConfig` | `useDynamicConfig.ts` | Dynamic configuration from remote feature flags. |
| `useMainLoopModel` | `useMainLoopModel.ts` | Resolves the current main-loop model name. |
| `useApiKeyVerification` | `useApiKeyVerification.ts` | Validates API key status. |
| `useFileHistorySnapshotInit` | `useFileHistorySnapshotInit.ts` | Initializes file history snapshot state on mount. |
| `useMemoryUsage` | `useMemoryUsage.ts` | Tracks process memory usage. |
| `useElapsedTime` | `useElapsedTime.ts` | Tracks elapsed wall-clock time for a turn. |
| `useMinDisplayTime` | `useMinDisplayTime.ts` | Ensures a minimum display duration before hiding UI. |
| `useTimeout` | `useTimeout.ts` | Generic timeout hook. |
| `useAfterFirstRender` | `useAfterFirstRender.ts` | Runs a callback after the first render completes. |
| `useBlink` | `useBlink.ts` | Blinking cursor animation toggle. |
| `useDoublePress` | `useDoublePress.ts` | Detects double-press of a key. |
| `useDeferredHookMessages` | `useDeferredHookMessages.ts` | Queues hook-generated messages for deferred processing. |

### Tools & Permissions

| Hook | File | Purpose |
|------|------|---------|
| `useCanUseTool` | `useCanUseTool.tsx` | Determines whether a tool can be used given current permissions. |
| `useMergedTools` | `useMergedTools.ts` | Merges built-in tools with MCP/plugin-provided tools. |
| `useMergedClients` | `useMergedClients.ts` | Merges MCP client connections. |
| `useManagePlugins` | `useManagePlugins.ts` | Plugin lifecycle management (install, update, remove). |
| `toolPermission/PermissionContext` | `toolPermission/PermissionContext.ts` | Permission context provider for tool-use decisions. |
| `toolPermission/permissionLogging` | `toolPermission/permissionLogging.ts` | Logs permission decisions for debugging. |
| `toolPermission/handlers/interactiveHandler` | `toolPermission/handlers/interactiveHandler.ts` | Interactive (human-in-the-loop) permission resolution. |
| `toolPermission/handlers/coordinatorHandler` | `toolPermission/handlers/coordinatorHandler.ts` | Coordinator-mode permission delegation. |
| `toolPermission/handlers/swarmWorkerHandler` | `toolPermission/handlers/swarmWorkerHandler.ts` | Swarm worker permission forwarding to leader. |

### UI & Display

| Hook | File | Purpose |
|------|------|---------|
| `useTerminalSize` | `useTerminalSize.ts` | Tracks terminal rows/columns, reacts to resize. |
| `useVirtualScroll` | `useVirtualScroll.ts` | Virtual scrolling for large message lists. |
| `useLogMessages` | `useLogMessages.ts` | Manages the log/message display list. |
| `useDiffData` | `useDiffData.ts` | Computes diff data for file changes. |
| `useDiffInIDE` | `useDiffInIDE.ts` | Opens diffs in the connected IDE. |
| `useTurnDiffs` | `useTurnDiffs.ts` | Tracks per-turn file diffs. |
| `usePrStatus` | `usePrStatus.ts` | Fetches and tracks PR status. |
| `useIssueFlagBanner` | `useIssueFlagBanner.ts` | Controls the issue/flag banner display. |
| `useAssistantHistory` | `useAssistantHistory.ts` | Manages assistant conversation history for display. |

### IDE Integration

| Hook | File | Purpose |
|------|------|---------|
| `useIDEIntegration` | `useIDEIntegration.tsx` | Top-level IDE connection and communication. |
| `useIdeConnectionStatus` | `useIdeConnectionStatus.ts` | Tracks whether the IDE bridge is connected. |
| `useIdeSelection` | `useIdeSelection.ts` | Reads the user's current selection in the IDE. |
| `useIdeAtMentioned` | `useIdeAtMentioned.ts` | Detects @-mentions from the IDE sidebar. |
| `useIdeLogging` | `useIdeLogging.ts` | Sends log messages to the IDE for display. |
| `useDirectConnect` | `useDirectConnect.ts` | Direct-connect mode for IDE-to-CLI communication. |

### Notifications

The `hooks/notifs/` subdirectory contains 16 notification hooks:

| Hook | Purpose |
|------|---------|
| `useAutoModeUnavailableNotification` | Warns when auto-mode cannot be used. |
| `useCanSwitchToExistingSubscription` | Prompts about switching to an existing subscription. |
| `useDeprecationWarningNotification` | Displays deprecation warnings. |
| `useFastModeNotification` | Notifies about fast-mode availability. |
| `useIDEStatusIndicator` | Shows IDE connection status in notifications. |
| `useInstallMessages` | Displays installation-related messages. |
| `useLspInitializationNotification` | Notifies about LSP server initialization status. |
| `useMcpConnectivityStatus` | Shows MCP server connectivity status. |
| `useModelMigrationNotifications` | Notifies about model migrations. |
| `useNpmDeprecationNotification` | Warns about npm package deprecation. |
| `usePluginAutoupdateNotification` | Notifies about plugin auto-updates. |
| `usePluginInstallationStatus` | Shows plugin installation progress. |
| `useRateLimitWarningNotification` | Warns when approaching rate limits. |
| `useSettingsErrors` | Surfaces settings validation errors. |
| `useStartupNotification` | Displays startup-time notifications. |
| `useTeammateShutdownNotification` | Notifies when a teammate agent shuts down. |

### Keybindings

| Hook | File | Purpose |
|------|------|---------|
| `useGlobalKeybindings` | `useGlobalKeybindings.tsx` | Registers application-wide keybindings (exports `GlobalKeybindingHandlers`). |
| `useCommandKeybindings` | `useCommandKeybindings.tsx` | Registers keybindings that trigger slash commands (exports `CommandKeybindingHandlers`). |
| `useExitOnCtrlCD` | `useExitOnCtrlCD.ts` | Ctrl+C / Ctrl+D exit handling. |
| `useExitOnCtrlCDWithKeybindings` | `useExitOnCtrlCDWithKeybindings.ts` | Exit handling integrated with the keybinding system. |

### Collaboration & Swarm

| Hook | File | Purpose |
|------|------|---------|
| `useSwarmInitialization` | `useSwarmInitialization.ts` | Initializes swarm mode (multi-agent coordination). |
| `useSwarmPermissionPoller` | `useSwarmPermissionPoller.ts` | Polls for permission requests from swarm workers. |
| `useMailboxBridge` | `useMailboxBridge.ts` | Bridges mailbox messages between leader and worker agents. |
| `useInboxPoller` | `useInboxPoller.ts` | Polls the agent inbox for incoming messages. |
| `useTeammateViewAutoExit` | `useTeammateViewAutoExit.ts` | Auto-exits the teammate detail view when the task completes. |
| `useBackgroundTaskNavigation` | `useBackgroundTaskNavigation.ts` | Keyboard navigation between background tasks. |
| `useTaskListWatcher` | `useTaskListWatcher.ts` | Watches the task list for changes. |
| `useTasksV2` | `useTasksV2.ts` | Task state management (v2 API). |

### Voice

| Hook | File | Purpose |
|------|------|---------|
| `useVoice` | `useVoice.ts` | Core voice recording and transcription. |
| `useVoiceEnabled` | `useVoiceEnabled.ts` | Checks whether voice mode is available. |
| `useVoiceIntegration` | `useVoiceIntegration.tsx` | Full voice integration -- keybindings, interim text, strip-trailing. |

### Suggestions & Completions

| Hook | File | Purpose |
|------|------|---------|
| `fileSuggestions` | `fileSuggestions.ts` | File path suggestions for @-mentions. |
| `unifiedSuggestions` | `unifiedSuggestions.ts` | Unified suggestion engine combining multiple sources. |
| `usePromptSuggestion` | `usePromptSuggestion.ts` | Contextual prompt suggestions. |
| `renderPlaceholder` | `renderPlaceholder.ts` | Renders placeholder text in the prompt input. |

### Session & Remote

| Hook | File | Purpose |
|------|------|---------|
| `useRemoteSession` | `useRemoteSession.ts` | Manages remote session connections. |
| `useSSHSession` | `useSSHSession.ts` | SSH session lifecycle for remote tool execution. |
| `useSessionBackgrounding` | `useSessionBackgrounding.ts` | Handles session backgrounding and foregrounding. |
| `useScheduledTasks` | `useScheduledTasks.ts` | Manages scheduled/cron agent tasks. |
| `useAwaySummary` | `useAwaySummary.ts` | Generates a summary of activity while the user was away. |
| `useTeleportResume` | `useTeleportResume.tsx` | Handles teleport-based session resume. |

### Data & Analytics

| Hook | File | Purpose |
|------|------|---------|
| `useCancelRequest` | `useCancelRequest.ts` | Cancels in-flight API requests (exports `CancelRequestHandler`). |
| `useSkillImprovementSurvey` | `useSkillImprovementSurvey.ts` | Triggers skill improvement surveys after tool use. |
| `useSkillsChange` | `useSkillsChange.ts` | Reacts to skill definition changes. |
| `useReplBridge` | `useReplBridge.tsx` | Bridges REPL state to external consumers (IDE, tests). |

### Clipboard & Notifications (misc)

| Hook | File | Purpose |
|------|------|---------|
| `useCopyOnSelect` | `useCopyOnSelect.ts` | Copies selected text to clipboard. |
| `useClipboardImageHint` | `useClipboardImageHint.ts` | Hints about clipboard image paste capability. |
| `useNotifyAfterTimeout` | `useNotifyAfterTimeout.ts` | Sends a system notification after a timeout. |
| `useUpdateNotification` | `useUpdateNotification.ts` | Notifies about available CLI updates. |
| `useChromeExtensionNotification` | `useChromeExtensionNotification.tsx` | Chrome extension integration notifications. |
| `useOfficialMarketplaceNotification` | `useOfficialMarketplaceNotification.tsx` | Official marketplace notifications. |
| `usePromptsFromClaudeInChrome` | `usePromptsFromClaudeInChrome.tsx` | Receives prompts forwarded from Claude in Chrome. |
| `useLspPluginRecommendation` | `useLspPluginRecommendation.tsx` | Recommends LSP plugins. |
| `usePluginRecommendationBase` | `usePluginRecommendationBase.tsx` | Base hook for plugin recommendation logic. |
| `useClaudeCodeHintRecommendation` | `useClaudeCodeHintRecommendation.tsx` | Claude Code hint/plugin recommendation. |

---

## The Wizard Pattern

The `source/src/components/wizard/` directory provides a reusable multi-step navigation framework used wherever the UI needs to walk the user through a sequence of steps.

### Architecture

The wizard is built from four pieces:

| Module | Role |
|--------|------|
| `WizardProvider.tsx` | React context provider. Manages `currentStepIndex`, `wizardData` (arbitrary state passed between steps), `navigationHistory` (for back navigation), and `isCompleted`. Calls `onComplete(wizardData)` when the wizard finishes. |
| `useWizard.ts` | Context consumer hook. Returns `currentStepIndex`, `totalSteps`, `title`, `showStepCounter`, `goNext`, `goBack`, `updateData`, and the accumulated `wizardData`. |
| `WizardDialogLayout.tsx` | Standard visual frame. Wraps each step in a `Dialog` (from the design system) with an auto-generated title suffix like "(2/5)" and a `WizardNavigationFooter`. |
| `WizardNavigationFooter.tsx` | Bottom bar showing navigation hints (up/down to navigate, Enter to select, Esc to go back). Integrates with `useExitOnCtrlCDWithKeybindings` for exit handling. |

The public API is exported from `index.ts`:
- Types: `WizardContextValue`, `WizardProviderProps`, `WizardStepComponent`
- Components: `WizardProvider`, `WizardDialogLayout`, `WizardNavigationFooter`
- Hook: `useWizard`

### Navigation Model

- **Forward** -- `goNext()` pushes the current step index onto `navigationHistory` and increments. On the last step, it sets `isCompleted = true`, which triggers `onComplete`.
- **Back** -- `goBack()` pops from `navigationHistory`. If history is empty and the user is on the first step, it calls `onCancel`.
- **Data accumulation** -- each step calls `updateData(partial)` to merge its results into the shared `wizardData` object. The final `onComplete` callback receives the fully accumulated data.

### Usage: Agent Creation Wizard

The most complex wizard consumer is `components/agents/new-agent-creation/CreateAgentWizard.tsx`, which defines 12 steps:

| Step | Component | Collects |
|------|-----------|----------|
| 1 | `TypeStep` | Agent type selection |
| 2 | `MethodStep` | Creation method |
| 3 | `DescriptionStep` | Agent description |
| 4 | `PromptStep` | System prompt |
| 5 | `ModelStep` | Model selection |
| 6 | `ToolsStep` | Tool allowlist |
| 7 | `MemoryStep` | Memory configuration |
| 8 | `ColorStep` | Agent color |
| 9 | `LocationStep` | Save location |
| 10 | `GenerateStep` | Auto-generation of config |
| 11 | `ConfirmStepWrapper` | Preview and confirmation |
| 12 | `ConfirmStep` | Final confirmation |

Each step component uses `useWizard()` to read existing data and call `goNext()` / `updateData()`. The `WizardDialogLayout` provides consistent framing across all steps.

---

## See Also

- [Ink TUI](ink-tui.md) -- the custom rendering engine underneath all components
- [State Management](state-management.md) -- Zustand store and contexts consumed by these hooks
- [Permissions & Security](permissions-security.md) -- the permission dialog components in depth
- [Task System](task-system.md) -- the background task and agent spawning UI
- [Plugin & Skill System](plugin-skill-system.md) -- skills menu and plugin recommendation hooks
- [Peripheral Systems](peripheral-systems.md) -- voice, vim, keybindings, and other hook-driven subsystems
