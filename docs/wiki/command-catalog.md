# Command Catalog

Commands are user-invocable slash actions in the Claude Code REPL. For architecture details — type system, dispatch pipeline, lazy loading, and feature gating — see [command-system.md](command-system.md). The source of truth is `source/src/commands.ts` (the `COMMANDS()` array and `INTERNAL_ONLY_COMMANDS` list) and the `index.ts` (or `index.tsx`) file in each command's subdirectory.

---

## How to read this catalog

| Column | Meaning |
|--------|---------|
| **Name** | The string after `/` |
| **Type** | `prompt` = expands to a model message; `local` = runs synchronously, returns text; `local-jsx` = renders an Ink UI panel |
| **Description** | Taken verbatim from `description` field in source |
| **Aliases** | Alternative names that resolve to the same command |
| **Gate / Availability** | Feature flag (`feature('FLAG')`) or availability requirement (`claude-ai`, `console`) or `ant-only` (runtime `USER_TYPE === 'ant'` check) |

Commands marked **internal-only** are removed from the public binary at build time (see `INTERNAL_ONLY_COMMANDS` in `commands.ts`).

---

## Session Management

| Name | Type | Description | Aliases | Gate / Availability |
|------|------|-------------|---------|---------------------|
| `init` | prompt | Initialize a new CLAUDE.md file with codebase documentation | — | — |
| `init-verifiers` | prompt | Create verifier skill(s) for automated verification of code changes | — | internal-only |
| `login` | local-jsx | Sign in with your Anthropic account (or "Switch Anthropic accounts" if already authed) | — | Hidden when `DISABLE_LOGIN_COMMAND`; only shown when not using 3P services |
| `logout` | local-jsx | Sign out from your Anthropic account | — | Hidden when `DISABLE_LOGOUT_COMMAND`; only shown when not using 3P services |
| `resume` | local-jsx | Resume a previous conversation | `continue` | — |
| `session` | local-jsx | Show remote session URL and QR code | `remote` | Only enabled/visible in remote mode |
| `teleport` | — | (internal-only stub in open build; teleport to remote session) | — | internal-only |
| `clear` | local | Clear conversation history and free up context | `reset`, `new` | — |
| `rename` | local-jsx | Rename the current conversation | — | — |
| `tag` | local-jsx | Toggle a searchable tag on the current session | — | ant-only |
| `branch` | local-jsx | Create a branch of the current conversation at this point | `fork` (only when `FORK_SUBAGENT` feature is off) | — |
| `fork` | local-jsx | Fork the current conversation into a sub-agent | — | `FORK_SUBAGENT` feature flag |
| `exit` | local-jsx | Exit the REPL | `quit` | — |

---

## Workspace

| Name | Type | Description | Aliases | Gate / Availability |
|------|------|-------------|---------|---------------------|
| `add-dir` | local-jsx | Add a new working directory | — | — |
| `memory` | local-jsx | Edit Claude memory files | — | — |
| `context` | local-jsx | Visualize current context usage as a colored grid | — | Interactive sessions only |
| `env` | local-jsx | (internal-only) Configure environment variables | — | internal-only |
| `config` | local-jsx | Open config panel | `settings` | — |
| `theme` | local-jsx | Change the theme | — | — |
| `color` | local-jsx | Set the prompt bar color for this session | — | — |
| `files` | local | List all files currently in context | — | ant-only (`USER_TYPE === 'ant'`) |
| `remote-env` | local-jsx | Configure the default remote environment for teleport sessions | — | `claude-ai` + policy `allow_remote_sessions` |

---

## Development & Git

| Name | Type | Description | Aliases | Gate / Availability |
|------|------|-------------|---------|---------------------|
| `commit` | prompt | Create a git commit | — | internal-only |
| `commit-push-pr` | prompt | Commit, push, and open a PR | — | internal-only |
| `diff` | local-jsx | View uncommitted changes and per-turn diffs | — | — |
| `review` | prompt | Review a pull request | — | — |
| `ultrareview` | local-jsx | ~10–20 min · Finds and verifies bugs in your branch. Runs in Claude Code on the web | — | — |
| `doctor` | local-jsx | Diagnose and verify your Claude Code installation and settings | — | Hidden when `DISABLE_DOCTOR_COMMAND` |
| `ide` | local-jsx | Manage IDE integrations and show status | — | — |
| `desktop` | local-jsx | Continue the current session in Claude Desktop | `app` | `claude-ai`; macOS and Windows x64 only |
| `mobile` | local-jsx | Show QR code to download the Claude mobile app | `ios`, `android` | — |
| `branch` | local-jsx | Create a branch of the current conversation at this point | `fork` (conditional) | — |
| `autofix-pr` | — | (internal-only stub in open build; auto-fix CI failures on a PR) | — | internal-only |
| `pr_comments` | prompt | Get comments from a GitHub pull request | — | — |
| `security-review` | prompt | Complete a security review of the pending changes on the current branch | — | internal-only |

---

## Productivity

| Name | Type | Description | Aliases | Gate / Availability |
|------|------|-------------|---------|---------------------|
| `cost` | local | Show the total cost and duration of the current session | — | Hidden for claude.ai subscribers (except ant users) |
| `usage` | local-jsx | Show plan usage limits | — | `claude-ai` |
| `extra-usage` | local-jsx | Configure extra usage to keep working when limits are hit | — | Overage provisioning allowed; interactive |
| `status` | local-jsx | Show Claude Code status including version, model, account, API connectivity, and tool statuses | — | — |
| `stats` | local-jsx | Show your Claude Code usage statistics and activity | — | — |
| `summary` | local | (internal-only) Summarize the conversation | — | internal-only |
| `tasks` | local-jsx | List and manage background tasks | `bashes` | — |
| `effort` | local-jsx | Set effort level for model usage | — | — |
| `copy` | local-jsx | Copy Claude's last response to clipboard (or /copy N for the Nth-latest) | — | — |
| `export` | local-jsx | Export the current conversation to a file or clipboard | — | — |
| `compact` | local | Clear conversation history but keep a summary in context | — | Hidden when `DISABLE_COMPACT` |
| `insights` | prompt | Generate a report analyzing your Claude Code sessions | — | — |
| `btw` | local-jsx | Ask a quick side question without interrupting the main conversation | — | — |

---

## Configuration

| Name | Type | Description | Aliases | Gate / Availability |
|------|------|-------------|---------|---------------------|
| `model` | local-jsx | Set the AI model for Claude Code | — | — |
| `fast` | local-jsx | Toggle fast mode (Haiku-only) | — | `claude-ai` or `console`; `FAST_MODE` feature flag |
| `effort` | local-jsx | Set effort level for model usage | — | — |
| `passes` | local-jsx | Share a free week of Claude Code with friends | — | `claude-ai`; hidden when not eligible |
| `permissions` | local-jsx | Manage allow & deny tool permission rules | `allowed-tools` | — |
| `hooks` | local-jsx | View hook configurations for tool events | — | — |
| `keybindings` | local | Open or create your keybindings configuration file | — | `KEYBINDING_CUSTOMIZATION` feature flag |
| `rate-limit-options` | local-jsx | Show options when rate limit is reached | — | `claude-ai`; hidden from help (internal use) |
| `privacy-settings` | local-jsx | View and update your privacy settings | — | Consumer subscribers only |
| `reset-limits` | local-jsx | (internal-only) Reset usage limits | — | internal-only |
| `advisor` | local | Configure the advisor model | — | — |
| `output-style` | local-jsx | Deprecated: use /config to change output style | — | Hidden |
| `sandbox` | local-jsx | Toggle sandbox mode for shell commands | — | — |
| `vim` | local | Toggle between Vim and Normal editing modes | — | — |
| `upgrade` | local-jsx | Upgrade to Max for higher rate limits and more Opus | — | `claude-ai`; hidden for enterprise |

---

## Integration

| Name | Type | Description | Aliases | Gate / Availability |
|------|------|-------------|---------|---------------------|
| `skills` | local-jsx | List available skills | — | — |
| `plugin` | local-jsx | Manage Claude Code plugins | `plugins`, `marketplace` | — |
| `reload-plugins` | local | Activate pending plugin changes in the current session | — | — |
| `mcp` | local-jsx | Manage MCP servers | — | — |
| `install-github-app` | local-jsx | Set up Claude GitHub Actions for a repository | — | `claude-ai` or `console`; hidden when `DISABLE_INSTALL_GITHUB_APP_COMMAND` |
| `install-slack-app` | local | Install the Claude Slack app | — | `claude-ai` |
| `chrome` | local-jsx | Claude in Chrome (Beta) settings | — | `claude-ai`; interactive only |
| `agents` | local-jsx | Manage agent configurations | — | — |

---

## Analysis

| Name | Type | Description | Aliases | Gate / Availability |
|------|------|-------------|---------|---------------------|
| `insights` | prompt | Generate a report analyzing your Claude Code sessions | — | — |
| `ctx_viz` | — | (internal-only stub in open build; context visualization) | — | internal-only |

---

## Workflow

| Name | Type | Description | Aliases | Gate / Availability |
|------|------|-------------|---------|---------------------|
| `plan` | local-jsx | Enable plan mode or view the current session plan | — | — |
| `compact` | local | Clear conversation history but keep a summary in context | — | Hidden when `DISABLE_COMPACT` |
| `rewind` | local | Restore the code and/or conversation to a previous point | `checkpoint` | — |
| `workflows` | local-jsx | (workflow scripts) List and run workflow scripts | — | `WORKFLOW_SCRIPTS` feature flag |
| `web-setup` | local-jsx | Setup Claude Code on the web (requires connecting your GitHub account) | — | `CCR_REMOTE_SETUP` feature flag + `claude-ai` |

---

## Advanced (Feature-Gated)

| Name | Type | Description | Aliases | Gate / Availability |
|------|------|-------------|---------|---------------------|
| `brief` | local-jsx | Toggle brief-only mode | — | `KAIROS` or `KAIROS_BRIEF` feature flag |
| `proactive` | — | (proactive suggestions) | — | `PROACTIVE` or `KAIROS` feature flag |
| `assistant` | — | (KAIROS assistant mode) | — | `KAIROS` feature flag |
| `remote-control` | local-jsx | Connect this terminal for remote-control sessions | `rc` | `BRIDGE_MODE` feature flag |
| `voice` | local | Toggle voice mode | — | `VOICE_MODE` feature flag; `claude-ai` |
| `buddy` | — | (buddy sub-agent) | — | `BUDDY` feature flag |
| `web-setup` | local-jsx | Setup Claude Code on the web | — | `CCR_REMOTE_SETUP` feature flag; `claude-ai` |
| `subscribe-pr` | — | (subscribe to PR webhooks) | — | `KAIROS_GITHUB_WEBHOOKS` feature flag |
| `ultraplan` | local-jsx | ~10–30 min · Claude Code on the web drafts an advanced plan you can edit and approve | — | `ULTRAPLAN` feature flag |
| `torch` | — | (torch mode) | — | `TORCH` feature flag |
| `peers` | — | (peer session inbox) | — | `UDS_INBOX` feature flag |

---

## Internal / Ant-Only

These commands are either removed from the public binary at build time (`INTERNAL_ONLY_COMMANDS`) or restricted to `USER_TYPE === 'ant'` at runtime.

| Name | Type | Description | Gate |
|------|------|-------------|------|
| `ant-trace` | — | (tracing / observability) | internal-only (stub in open build) |
| `bughunter` | — | (automated bug hunting) | internal-only (stub in open build) |
| `security-review` | prompt | Complete a security review of the pending changes on the current branch | internal-only |
| `agents-platform` | — | (agents platform management) | ant-only (`USER_TYPE === 'ant'`) |
| `ultraplan` | local-jsx | ~10–30 min · Claude Code on the web drafts an advanced plan | internal-only + `ULTRAPLAN` flag |
| `subscribe-pr` | — | Subscribe to PR webhook events | internal-only + `KAIROS_GITHUB_WEBHOOKS` flag |
| `backfill-sessions` | — | (backfill session data) | internal-only (stub in open build) |
| `break-cache` | — | (break prompt cache for testing) | internal-only (stub in open build) |
| `mock-limits` | — | (mock rate/usage limits for testing) | internal-only (stub in open build) |
| `reset-limits` | — | Reset usage limits | internal-only (stub in open build) |
| `env` | — | Configure environment variables | internal-only (stub in open build) |
| `teleport` | — | Teleport current session to remote | internal-only (stub in open build) |
| `summary` | — | Summarize the conversation | internal-only (stub in open build) |
| `commit` | prompt | Create a git commit | internal-only |
| `commit-push-pr` | prompt | Commit, push, and open a PR | internal-only |
| `ctx_viz` | — | Context visualization | internal-only (stub in open build) |
| `good-claude` | — | (internal feedback / training signal) | internal-only (stub in open build) |
| `issue` | — | (file an internal issue) | internal-only (stub in open build) |
| `init-verifiers` | prompt | Create verifier skill(s) for automated verification | internal-only |
| `autofix-pr` | — | Auto-fix CI failures on a PR | internal-only (stub in open build) |
| `tag` | local-jsx | Toggle a searchable tag on the current session | ant-only at runtime |
| `files` | local | List all files currently in context | ant-only at runtime |
| `version` | local | Print the version this session is running | ant-only at runtime |
| `bridge-kick` | local | Inject bridge failure states for manual recovery testing | ant-only at runtime |
| `onboarding` | — | (first-run onboarding flow) | internal-only (stub in open build) |
| `perf-issue` | — | (file a performance issue report) | internal-only (stub in open build) |
| `oauth-refresh` | — | (force OAuth token refresh) | internal-only (stub in open build) |
| `debug-tool-call` | — | (debug a specific tool call) | internal-only (stub in open build) |
| `share` | — | (share conversation) | internal-only (stub in open build) |

---

## Additional Commands (Always Present)

These commands appear in the `COMMANDS()` array but don't fit neatly into a category above.

| Name | Type | Description | Aliases | Gate / Availability |
|------|------|-------------|---------|---------------------|
| `help` | local-jsx | Show help and available commands | — | — |
| `feedback` | local-jsx | Submit feedback about Claude Code | `bug` | Disabled for Bedrock/Vertex/Foundry users and ants |
| `release-notes` | local | View release notes | — | — |
| `upgrade` | local-jsx | Upgrade to Max for higher rate limits and more Opus | — | `claude-ai` |
| `terminal-setup` | local-jsx | Install Shift+Enter (or Option+Enter on Apple Terminal) key binding for newlines | — | Hidden on native CSI-u terminals |
| `statusline` | prompt | Set up Claude Code's status line UI | — | — |
| `stickers` | local | Order Claude Code stickers | — | — |
| `think-back` | local-jsx | Your 2025 Claude Code Year in Review | — | `tengu_thinkback` Statsig gate |
| `thinkback-play` | local | Play the thinkback animation | — | `tengu_thinkback` Statsig gate; hidden |
| `heapdump` | local | Dump the JS heap to ~/Desktop | — | Hidden |

---

## Feature Gate Reference

| Feature Flag | Commands Gated |
|--------------|---------------|
| `KAIROS` | `brief`, `proactive`, `assistant`, `subscribe-pr` |
| `KAIROS_BRIEF` | `brief` (also enabled by `KAIROS`) |
| `KAIROS_GITHUB_WEBHOOKS` | `subscribe-pr` |
| `PROACTIVE` | `proactive` (also enabled by `KAIROS`) |
| `BRIDGE_MODE` | `remote-control` (bridge), `remote-control-server` |
| `DAEMON` + `BRIDGE_MODE` | `remote-control-server` |
| `VOICE_MODE` | `voice` |
| `FORK_SUBAGENT` | `fork` (standalone); removes `fork` alias from `branch` |
| `WORKFLOW_SCRIPTS` | `workflows` |
| `CCR_REMOTE_SETUP` | `web-setup` |
| `ULTRAPLAN` | `ultraplan` |
| `TORCH` | `torch` |
| `BUDDY` | `buddy` |
| `UDS_INBOX` | `peers` |
| `HISTORY_SNIP` | `force-snip` (internal) |
| `EXPERIMENTAL_SKILL_SEARCH` | Clears skill index cache on reload |
| `tengu_thinkback` (Statsig) | `think-back`, `thinkback-play` |
| `tengu_cobalt_lantern` (GrowthBook) | `web-setup` (also requires policy) |

### Availability Requirements

Commands with an `availability` array are hidden from users who do not match any listed auth type. See `meetsAvailabilityRequirement()` in `commands.ts`.

| Availability Value | Meaning |
|-------------------|---------|
| `claude-ai` | claude.ai OAuth subscriber (Pro/Max/Team/Enterprise) |
| `console` | Direct Console API key user (api.anthropic.com, not via claude.ai) |

Commands requiring `availability` include: `usage`, `desktop`, `install-github-app` (also `console`), `install-slack-app`, `chrome`, `voice`, `upgrade`, `fast` (also `console`), `passes`, `remote-env`, `web-setup`, `extra-usage` (indirectly via overage check).
