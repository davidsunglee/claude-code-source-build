# Tool Catalog

> For tool architecture -- the `Tool` type, `buildTool()`, execution lifecycle, permission checking, and `ToolUseContext` -- see [Tool System](tool-system.md). This page is the reference catalog of every built-in tool.

All tools are registered in `source/src/tools.ts` via `getAllBaseTools()`. The function returns a flat array; conditional tools are gated by feature flags, environment variables, or runtime checks. Tools are defined with `buildTool()` in their respective directories under `source/src/tools/`.

---

## Core I/O

Tools for running commands, reading files, and writing files. These are the fundamental primitives most tasks depend on.

| Name | Wire Name | Purpose | Key Input Parameters | Permission | Notable Behavior |
|------|-----------|---------|---------------------|------------|------------------|
| BashTool | `Bash` | Execute shell commands | `command`, `timeout?`, `run_in_background?`, `dangerouslyDisableSandbox?`, `description?` | Per-command approval; regex allow/deny rules | Working directory persists between calls; supports sandbox mode; background execution with notifications; steers model away from `find`/`grep`/`cat`/`sed` in favor of dedicated tools |
| FileReadTool | `Read` | Read files from the filesystem | `file_path`, `offset?`, `limit?`, `pages?` (PDF only) | Read permission (auto-approved for most paths) | Reads up to 2000 lines by default; supports images (multimodal), PDFs, Jupyter notebooks, directories (via error message suggesting `ls`); returns `cat -n` format with line numbers |
| FileEditTool | `Edit` | Make targeted edits to existing files | `file_path`, `old_string`, `new_string`, `replace_all?` | Write permission (file path-based rules) | Requires file to have been read first (staleness guard); validates `old_string` is unique unless `replace_all`; handles quote normalization; notifies LSP servers of changes; tracks file history for undo |
| FileWriteTool | `Write` | Create new files or fully overwrite existing ones | `file_path`, `content` | Write permission (file path-based rules) | Requires prior read of existing files; creates parent directories automatically; notifies LSP and VSCode of changes; prefers Edit tool for modifications to existing files |
| PowerShellTool | `PowerShell` | Execute PowerShell commands on Windows | `command`, `timeout?`, `run_in_background?` | Per-command approval (same model as BashTool) | Windows-only; enabled via `CLAUDE_CODE_USE_POWERSHELL_TOOL` env var; ant-native builds default on, external builds opt-in |
| REPLTool | `REPL` | Batch operations in an isolated VM | (passthrough to wrapped primitives) | Inherits permissions from wrapped tools | Ant-only (`USER_TYPE=ant`); when enabled, hides `Read`, `Write`, `Edit`, `Glob`, `Grep`, `Bash`, `NotebookEdit`, and `Agent` tools -- they become accessible only through the REPL VM |

## Search

Tools for finding files and searching content across the codebase.

| Name | Wire Name | Purpose | Key Input Parameters | Permission | Notable Behavior |
|------|-----------|---------|---------------------|------------|------------------|
| GlobTool | `Glob` | Find files by name pattern | `pattern`, `path?` | Read permission | Returns up to 100 files sorted by modification time; concurrency-safe and read-only; omitted when embedded search tools (`bfs`/`ugrep`) are available in ant-native builds |
| GrepTool | `Grep` | Search file contents with regex | `pattern`, `path?`, `glob?`, `output_mode?`, `-A?`, `-B?`, `-C?`, `-n?`, `-i?`, `type?`, `head_limit?`, `offset?`, `multiline?` | Read permission | Built on ripgrep; three output modes: `content`, `files_with_matches` (default), `count`; default `head_limit` of 250; pagination via `offset`; omitted when embedded search tools are available |
| ToolSearchTool | `ToolSearch` | Fetch full schemas for deferred tools | `query`, `max_results?` | None (always allowed) | Only present when tool search is optimistically enabled (many tools loaded); supports `select:<name>` syntax for direct lookup or keyword search; returns tool definitions as `<functions>` blocks |

## AI / Web

Tools for web access -- searching the internet and fetching page content.

| Name | Wire Name | Purpose | Key Input Parameters | Permission | Notable Behavior |
|------|-----------|---------|---------------------|------------|------------------|
| WebSearchTool | `WebSearch` | Search the web via Claude's built-in web search | `query`, `allowed_domains?`, `blocked_domains?` | None (auto-approved) | Delegates to the Anthropic API's `web_search_20250305` tool type; runs a sub-query against a small/fast model to extract results; returns search hits with titles and URLs |
| WebFetchTool | `WebFetch` | Fetch and extract content from a URL | `url`, `prompt` | Per-domain approval; some hosts pre-approved | Converts fetched HTML to Markdown; applies the `prompt` to the converted content; truncates to `MAX_MARKDOWN_LENGTH`; supports `shouldDefer` for tool search optimization |

## Task Management

Tools for the structured task/todo system (v2). These replace the legacy `TodoWrite` approach with a persistent, server-backed task list.

| Name | Wire Name | Purpose | Key Input Parameters | Permission | Notable Behavior |
|------|-----------|---------|---------------------|------------|------------------|
| TaskCreateTool | `TaskCreate` | Create a new task | `subject`, `description`, `activeForm?`, `metadata?` | None (always allowed) | Only enabled when TodoV2 is active (`isTodoV2Enabled()`); the `activeForm` field provides text shown in the UI spinner (e.g., "Running tests") |
| TaskUpdateTool | `TaskUpdate` | Update an existing task | `taskId`, `subject?`, `description?`, `status?`, `activeForm?`, `addBlocks?`, `addBlockedBy?`, `owner?`, `metadata?` | None (always allowed) | Supports status transitions including `deleted`; integrates with teammate mailbox for swarm coordination; fires task-completed hooks |
| TaskGetTool | `TaskGet` | Retrieve a task by ID | `taskId` | None (always allowed) | Read-only; returns full task details including status, blocks, and blockedBy relationships |
| TaskListTool | `TaskList` | List all tasks | (none) | None (always allowed) | Read-only; filters out tasks with `_internal` metadata; returns summary view with id, subject, status, owner, and blockedBy |
| TaskStopTool | `TaskStop` | Stop a running background task | `task_id?`, `shell_id?` | None (always allowed) | Aliases: `KillShell` (deprecated); accepts both `task_id` and legacy `shell_id`; works for any background task type (shell, agent, etc.) |
| TaskOutputTool | `TaskOutput` | Retrieve output from a background task | (wire name: `TaskOutput`) | None (always allowed) | Always included in `getAllBaseTools()`; used to surface results of background tasks to the conversation |

## Workspace

Tools that control the session's working mode or environment.

| Name | Wire Name | Purpose | Key Input Parameters | Permission | Notable Behavior |
|------|-----------|---------|---------------------|------------|------------------|
| EnterWorktreeTool | `EnterWorktree` | Create an isolated git worktree and switch into it | `name?` | None (always allowed) | Only enabled when worktree mode is active (`isWorktreeModeEnabled()`); validates slug format; creates worktree via `git worktree add` or configured hooks; saves worktree state for session persistence |
| ExitWorktreeTool | `ExitWorktree` | Leave the current worktree | `action` (`keep` or `remove`), `discard_changes?` | None (always allowed) | Requires explicit `discard_changes: true` to remove worktrees with uncommitted work; counts changed files and unmerged commits as safety check |
| EnterPlanModeTool | `EnterPlanMode` | Switch to plan mode for exploration/design | (none) | User approval (permission dialog) | Disabled when `--channels` is active (no terminal for the exit dialog); cannot be used in agent (subagent) contexts; transitions permission state for read-only exploration |
| ExitPlanModeTool | `ExitPlanMode` | Present plan and request approval to proceed | Plan content (generated internally) | User approval | Actually `ExitPlanModeV2Tool` in the source; integrates with auto-mode, swarm plan approval, and plan file persistence; supports teammate plan approval flow |

## MCP

Tools for interacting with Model Context Protocol servers and their resources.

| Name | Wire Name | Purpose | Key Input Parameters | Permission | Notable Behavior |
|------|-----------|---------|---------------------|------------|------------------|
| MCPTool | `mcp__<server>__<tool>` | Execute an MCP server's tool | (passthrough -- defined by the MCP server) | Per-call approval (`passthrough` behavior) | Base definition is a stub -- `mcpClient.ts` clones and overrides `name`, `description`, `prompt`, `call`, and `userFacingName` per-server-tool; `isMcp: true` flag enables MCP-specific handling throughout the system |
| ListMcpResourcesTool | `ListMcpResourcesTool` | List available resources from MCP servers | `server?` | None (always allowed) | Read-only; always included in `getAllBaseTools()` (but filtered from `getTools()` as a special tool); `shouldDefer: true` |
| ReadMcpResourceTool | `ReadMcpResourceTool` | Read a specific MCP resource by URI | `server`, `uri` | None (always allowed) | Read-only; persists binary content to disk and returns path; always included but filtered as special tool |
| McpAuthTool | `mcp__<server>__authenticate` | Start OAuth flow for an unauthenticated MCP server | (none) | Per-call approval | Not a statically registered tool -- dynamically created by `createMcpAuthTool()` for servers that need OAuth; returns an authorization URL; real tools replace it once auth completes |

## Collaboration

Tools for multi-agent coordination -- spawning subagents, messaging teammates, and managing swarm teams.

| Name | Wire Name | Purpose | Key Input Parameters | Permission | Notable Behavior |
|------|-----------|---------|---------------------|------------|------------------|
| AgentTool | `Agent` | Spawn a subagent for a subtask | `prompt`, `agent_type?`, various context params | None (always allowed) | Legacy wire name: `Task`; runs a full `query()` loop in a child context with its own tool pool; supports built-in agent types (`Explore`, `Plan`, `verification`); one-shot built-in agents skip the agentId/SendMessage trailer |
| SendMessageTool | `SendMessage` | Send a message to a teammate or peer | `to`, `summary?`, `message`, `structured_message?` | None (always allowed) | Supports structured messages (`shutdown_request`, `shutdown_response`, `plan_approval_response`); broadcast via `*`; UDS peer addressing (`uds:<path>`); bridge addressing (`bridge:<session-id>`) when `UDS_INBOX` feature is enabled |
| TeamCreateTool | `TeamCreate` | Create a multi-agent swarm team | `team_name`, `description?`, `agent_type?` | None (always allowed) | Only enabled when agent swarms are active (`isAgentSwarmsEnabled()`); generates unique team names; writes team file; registers for session cleanup; lazy-loaded to break circular dependency |
| TeamDeleteTool | `TeamDelete` | Disband a swarm team and clean up | (none) | None (always allowed) | Only enabled when agent swarms are active; cleans up team directories and teammate color assignments; clears the leader team name |

## Scheduling

Tools for scheduling recurring or one-shot prompts and managing remote triggers.

| Name | Wire Name | Purpose | Key Input Parameters | Permission | Notable Behavior |
|------|-----------|---------|---------------------|------------|------------------|
| CronCreateTool | `CronCreate` | Schedule a recurring or one-shot prompt | `cron`, `prompt`, `recurring?`, `durable?` | None (always allowed) | Feature-gated: `AGENT_TRIGGERS`; standard 5-field cron expression in local time; `durable: true` persists to `.claude/scheduled_tasks.json`; max 50 jobs; auto-expires recurring jobs after configurable max age |
| CronDeleteTool | `CronDelete` | Cancel a scheduled cron job | `id` | None (always allowed) | Feature-gated: `AGENT_TRIGGERS`; accepts the job ID returned by CronCreate |
| CronListTool | `CronList` | List active cron jobs | (none) | None (always allowed) | Feature-gated: `AGENT_TRIGGERS`; read-only; returns job details including human-readable schedule |
| RemoteTriggerTool | `RemoteTrigger` | Manage scheduled remote agent triggers via claude.ai API | `action` (`list`/`get`/`create`/`update`/`run`), `trigger_id?`, `body?` | None (always allowed) | Feature-gated: `AGENT_TRIGGERS_REMOTE`; calls the CCR triggers API with in-process OAuth; `isReadOnly` depends on the action |

## Configuration

Tools for modifying Claude Code settings and executing skills.

| Name | Wire Name | Purpose | Key Input Parameters | Permission | Notable Behavior |
|------|-----------|---------|---------------------|------------|------------------|
| ConfigTool | `Config` | Get or set Claude Code settings | `setting`, `value?` | Per-setting approval | Ant-only (`USER_TYPE=ant`); supports settings like theme, model, permissions; omitting `value` performs a get operation |
| SkillTool | `Skill` | Execute a registered skill (slash command) | `skill`, `args?` | Per-skill approval (permission rules) | Resolves skill by name from local, bundled, and MCP prompt commands; forks a subagent with the skill's prompt injected; supports model overrides from skill frontmatter; tracks invoked skills for dedup |

## Special

Tools with unique behaviors that do not fit neatly into other categories.

| Name | Wire Name | Purpose | Key Input Parameters | Permission | Notable Behavior |
|------|-----------|---------|---------------------|------------|------------------|
| AskUserQuestionTool | `AskUserQuestion` | Present multiple-choice questions to the user | `question`, `options[]` (label, description, preview?), `multiSelect?` | None (always allowed) | Users always see an "Other" option for free-text input; supports preview rendering (markdown or HTML) for side-by-side comparison; plan mode restrictions apply |
| BriefTool | `SendUserMessage` | Send a message to the user (chat/brief view) | `message`, `attachments?`, `status` (`normal` or `proactive`) | None (always allowed) | Legacy wire name: `Brief`; primary output channel in brief/chat mode; supports file attachments; `proactive` status triggers push notification routing; feature-gated by `KAIROS` or `KAIROS_BRIEF` |
| SleepTool | `Sleep` | Wait for a specified duration | `duration_ms` | None (always allowed) | Feature-gated: `PROACTIVE` or `KAIROS`; preferred over `Bash(sleep ...)`; does not hold a shell process; user can interrupt; receives periodic `<tick>` check-ins |
| SyntheticOutputTool | `StructuredOutput` | Return structured JSON output | (dynamic -- matches user-provided JSON schema) | None (always allowed) | Only enabled in non-interactive (SDK/CLI) sessions; validates output against a provided JSON schema via Ajv; not included in the standard tool list -- added specially; uses WeakMap cache for schema compilation |
| NotebookEditTool | `NotebookEdit` | Edit Jupyter notebook cells | `notebook_path`, `cell_id?`, `new_source`, `cell_type?`, `edit_mode?` | Write permission (file path-based) | Supports `replace`, `insert`, and `delete` edit modes; handles `.ipynb` JSON structure; FileEditTool redirects `.ipynb` files here |
| TodoWriteTool | `TodoWrite` | Manage the session task checklist (legacy) | `todos` (array of todo items) | None (always allowed) | Only enabled when TodoV2 is **not** active (mutually exclusive with TaskCreate/TaskUpdate/etc.); replaced by the Task Management tools in newer configurations |
| LSPTool | `LSP` | Query Language Server Protocol servers | `operation`, `filePath`, `line`, `character?`, `query?` | Read permission | Only enabled when `ENABLE_LSP_TOOL=true` env var is set; operations: `goToDefinition`, `findReferences`, `hover`, `documentSymbol`, `workspaceSymbol`, `goToImplementation`, `prepareCallHierarchy`, `incomingCalls`, `outgoingCalls` |

---

## Feature-Gated Tools

Many tools are conditionally included in `getAllBaseTools()` based on feature flags, environment variables, or runtime checks. The table below summarizes every gate.

| Tool | Gate | Condition |
|------|------|-----------|
| GlobTool, GrepTool | Embedded search check | Omitted when `hasEmbeddedSearchTools()` returns true (ant-native builds with `bfs`/`ugrep` bundled) |
| ConfigTool | `USER_TYPE` | Only when `process.env.USER_TYPE === 'ant'` |
| TungstenTool | `USER_TYPE` | Only when `process.env.USER_TYPE === 'ant'` (internal tool) |
| REPLTool | `USER_TYPE` | Only when `process.env.USER_TYPE === 'ant'` |
| SuggestBackgroundPRTool | `USER_TYPE` | Only when `process.env.USER_TYPE === 'ant'` (internal tool) |
| PowerShellTool | Platform + env | Windows only; `CLAUDE_CODE_USE_POWERSHELL_TOOL` env var (default on for ant, opt-in for external) |
| TaskCreateTool, TaskGetTool, TaskUpdateTool, TaskListTool | Runtime check | `isTodoV2Enabled()` must return true |
| TodoWriteTool | Runtime check | `isTodoV2Enabled()` must return **false** (mutually exclusive with Task* tools) |
| EnterWorktreeTool, ExitWorktreeTool | Runtime check | `isWorktreeModeEnabled()` must return true |
| TeamCreateTool, TeamDeleteTool | Runtime check | `isAgentSwarmsEnabled()` must return true |
| LSPTool | Environment variable | `ENABLE_LSP_TOOL` must be truthy |
| ToolSearchTool | Runtime check | `isToolSearchEnabledOptimistic()` must return true (heuristic based on tool count) |
| SleepTool | Feature flag | `feature('PROACTIVE')` or `feature('KAIROS')` |
| CronCreateTool, CronDeleteTool, CronListTool | Feature flag | `feature('AGENT_TRIGGERS')` |
| RemoteTriggerTool | Feature flag | `feature('AGENT_TRIGGERS_REMOTE')` |
| MonitorTool | Feature flag | `feature('MONITOR_TOOL')` |
| SendUserFileTool | Feature flag | `feature('KAIROS')` |
| PushNotificationTool | Feature flag | `feature('KAIROS')` or `feature('KAIROS_PUSH_NOTIFICATION')` |
| SubscribePRTool | Feature flag | `feature('KAIROS_GITHUB_WEBHOOKS')` |
| BriefTool (SendUserMessage) | Feature flag | `feature('KAIROS')` or `feature('KAIROS_BRIEF')` (build-time); runtime GrowthBook gate for full enablement |
| WebBrowserTool | Feature flag | `feature('WEB_BROWSER_TOOL')` |
| OverflowTestTool | Feature flag | `feature('OVERFLOW_TEST_TOOL')` (testing only) |
| CtxInspectTool | Feature flag | `feature('CONTEXT_COLLAPSE')` |
| TerminalCaptureTool | Feature flag | `feature('TERMINAL_PANEL')` |
| SnipTool | Feature flag | `feature('HISTORY_SNIP')` |
| ListPeersTool | Feature flag | `feature('UDS_INBOX')` |
| WorkflowTool | Feature flag | `feature('WORKFLOW_SCRIPTS')` |
| VerifyPlanExecutionTool | Environment variable | `CLAUDE_CODE_VERIFY_PLAN === 'true'` |
| SyntheticOutputTool (StructuredOutput) | Runtime check | Not in `getAllBaseTools()` -- added separately when `isSyntheticOutputToolEnabled()` returns true (non-interactive sessions only) |
| McpAuthTool | Dynamic | Created per-server by `createMcpAuthTool()` when an MCP server requires OAuth; not in `getAllBaseTools()` |
| TestingPermissionTool | Environment variable | `NODE_ENV === 'test'` only |

---

## Source Reference

- Tool registration: [`source/src/tools.ts`](../../source/src/tools.ts) -- `getAllBaseTools()`, `getTools()`, `assembleToolPool()`
- Tool type definition: [`source/src/Tool.ts`](../../source/src/Tool.ts) -- `Tool`, `ToolDef`, `buildTool()`
- Tool directories: `source/src/tools/<ToolName>/` -- each tool has its own directory with implementation, prompt, constants, and UI files
- Tool constants/disallowed lists: [`source/src/constants/tools.ts`](../../source/src/constants/tools.ts)
