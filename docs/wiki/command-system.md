# Command System

> **Context**: Commands are user-invocable actions triggered by typing `/name` in the REPL input. They are distinct from *tools* (model-invocable capabilities). Some commands generate prompts that feed into the model; others execute locally and return results directly to the UI. This page covers the type system, registration, dispatch, lazy loading, feature gating, and skill integration. For a per-command reference, see [Command Catalog](command-catalog.md). For the tool system, see [Tool System](tool-system.md).

---

## Key Files

| File | Purpose |
|------|---------|
| `src/types/command.ts` | `CommandBase`, `PromptCommand`, `LocalCommand`, `LocalJSXCommand`, `LocalCommandResult`, helper types |
| `src/commands.ts` | `COMMANDS()` array, `loadAllCommands()`, `getCommands()`, `findCommand()`, filtering sets, skill pipelines |
| `src/commands/*/index.ts` | Individual command metadata (one per command directory) |
| `src/skills/loadSkillsDir.ts` | Discovers `.claude/skills/` directories and converts SKILL.md files into `PromptCommand` objects |
| `src/skills/bundledSkills.ts` | Built-in skills shipped with the binary |
| `src/utils/plugins/loadPluginCommands.ts` | Loads commands and skills from installed plugins |

---

## Command Types

`Command` is a discriminated union: `CommandBase & (PromptCommand | LocalCommand | LocalJSXCommand)`. The discriminant is the `type` field.

### PromptCommand (`type: 'prompt'`)

Generates text content that is injected into the conversation as a user message and sent to the model. The model then acts on the prompt.

```ts
type PromptCommand = {
  type: 'prompt'
  progressMessage: string          // shown in UI while model processes
  contentLength: number            // character count for token estimation
  source: SettingSource | 'builtin' | 'mcp' | 'plugin' | 'bundled'
  getPromptForCommand(args: string, context: ToolUseContext): Promise<ContentBlockParam[]>
  // Optional fields:
  argNames?: string[]
  allowedTools?: string[]          // restrict which tools the model may use
  model?: string                   // override the model for this command
  effort?: EffortValue
  context?: 'inline' | 'fork'     // 'fork' runs in a sub-agent
  agent?: string                   // agent type when forked
  hooks?: HooksSettings            // hooks to register when skill is invoked
  skillRoot?: string               // base dir for skill resource resolution
  paths?: string[]                 // glob patterns for path-scoped visibility
  disableNonInteractive?: boolean
  pluginInfo?: { pluginManifest: PluginManifest; repository: string }
}
```

Examples: `/review`, `/init`, `/commit`, `/security-review`, `/statusline`, `/insights`.

The `getPromptForCommand()` method can do arbitrary async work -- reading files, running shell commands via `executeShellCommandsInPrompt()`, fetching default branch names -- before returning the prompt content blocks.

### LocalCommand (`type: 'local'`)

Executes a local function and returns a `LocalCommandResult` (text, compaction result, or skip). Does not render JSX. Used for simple, non-interactive operations.

```ts
type LocalCommand = {
  type: 'local'
  supportsNonInteractive: boolean  // whether it works in SDK/headless mode
  load: () => Promise<LocalCommandModule>
}
```

Examples: `/clear`, `/compact`, `/vim`, `/keybindings`, `/stickers`, `/release-notes`, `/advisor`.

### LocalJSXCommand (`type: 'local-jsx'`)

Lazily loads a module that returns a React (Ink) component tree. Used for commands that need interactive UI: pickers, forms, confirmation dialogs, scrollable lists.

```ts
type LocalJSXCommand = {
  type: 'local-jsx'
  load: () => Promise<LocalJSXCommandModule>
}
```

The loaded module exports a `call(onDone, context, args)` function that returns `React.ReactNode`. The `onDone` callback signals completion and controls post-command behavior (display mode, whether to query the model, meta messages, next input prefill).

Examples: `/help`, `/config`, `/resume`, `/model`, `/mcp`, `/permissions`, `/doctor`.

---

## CommandBase Properties

All three command types share `CommandBase`:

| Property | Type | Default | Purpose |
|----------|------|---------|---------|
| `name` | `string` | required | Primary slash-command name (e.g., `'compact'`) |
| `description` | `string` | required | User-facing description shown in typeahead and help |
| `aliases` | `string[]` | `undefined` | Alternative names (e.g., `/reset` and `/new` for `/clear`) |
| `isEnabled` | `() => boolean` | `true` | Dynamic enablement check (feature flags, env vars, platform) |
| `isHidden` | `boolean` | `false` | If true, hidden from typeahead/help but still invocable |
| `availability` | `CommandAvailability[]` | `undefined` | Auth/provider requirement: `'claude-ai'` or `'console'` |
| `argumentHint` | `string` | `undefined` | Gray hint text after the command name in typeahead |
| `whenToUse` | `string` | `undefined` | Detailed usage scenarios (from Skill spec) |
| `hasUserSpecifiedDescription` | `boolean` | `undefined` | Whether description was set by user (vs auto-derived) |
| `version` | `string` | `undefined` | Command/skill version string |
| `disableModelInvocation` | `boolean` | `undefined` | If true, model cannot invoke via SkillTool |
| `userInvocable` | `boolean` | `undefined` | Whether users can invoke by typing `/skill-name` |
| `loadedFrom` | enum | `undefined` | Origin: `'commands_DEPRECATED'`, `'skills'`, `'plugin'`, `'managed'`, `'bundled'`, `'mcp'` |
| `kind` | `'workflow'` | `undefined` | Distinguishes workflow-backed commands (badged in autocomplete) |
| `immediate` | `boolean` | `undefined` | If true, executes immediately without waiting for a stop point |
| `isSensitive` | `boolean` | `undefined` | If true, args are redacted from conversation history |
| `isMcp` | `boolean` | `undefined` | Whether this is an MCP-provided command |
| `userFacingName` | `() => string` | `name` | Override when displayed name differs from internal name |

Helper functions in `types/command.ts`:

- **`getCommandName(cmd)`** -- resolves the user-visible name, falling back to `cmd.name`.
- **`isCommandEnabled(cmd)`** -- resolves `isEnabled()`, defaulting to `true`.

---

## Registration Architecture

### The COMMANDS() Memoized Array

`COMMANDS()` in `commands.ts` is a **memoized function** (via `lodash-es/memoize`) that returns an array of all built-in commands. It is declared as a function rather than a top-level constant because some commands read from config or auth state at construction time, and config is not available during module initialization.

The array contains:

1. **Direct imports** -- approximately 87 commands imported at the top of `commands.ts` as ES module imports (e.g., `import clear from './commands/clear/index.js'`). These are always included in the bundle.

2. **Feature-gated lazy imports** -- approximately 15 commands loaded via `require()` behind `feature()` guards from `bun:bundle`. Dead code elimination removes these at build time when the flag is off.

3. **Conditional commands** -- some are included only under runtime conditions:
   - `login` and `logout` are included only when `!isUsing3PServices()` (excluded for Bedrock/Vertex/Foundry).
   - `INTERNAL_ONLY_COMMANDS` are included only when `USER_TYPE === 'ant'` and `!IS_DEMO`.

### INTERNAL_ONLY_COMMANDS

A separate exported array of commands restricted to Anthropic internal users. Includes commands like `/commit`, `/commit-push-pr`, `/version`, `/bridge-kick`, `/mock-limits`, and various stubs. In the external (open-source) build, many of these are compiled to `{ isEnabled: () => false, isHidden: true, name: 'stub' }`.

---

## Dispatch Pipeline

When the user types `/compact some args` and presses Enter:

```
getCommands(cwd)
  |
  +-> loadAllCommands(cwd)     [memoized -- one-time load per cwd]
  |     |
  |     +-> getSkills(cwd)     [skill dirs, plugin skills, bundled skills, builtin plugin skills]
  |     +-> getPluginCommands()
  |     +-> getWorkflowCommands(cwd)  [feature-gated: WORKFLOW_SCRIPTS]
  |     +-> COMMANDS()         [built-in commands, appended last]
  |
  +-> getDynamicSkills()       [skills discovered during file operations]
  |
  +-> filter: meetsAvailabilityRequirement() && isCommandEnabled()
  |
  +-> dedup dynamic skills by name against base commands
  |
  = Command[]

findCommand('compact', commands)
  |
  +-> linear scan: match on name, getCommandName(), or aliases
  |
  = Command | undefined
```

Key design decisions:

1. **Skills before built-ins**: In `loadAllCommands`, the merge order is `bundledSkills > builtinPluginSkills > skillDirCommands > workflowCommands > pluginCommands > pluginSkills > COMMANDS()`. This means user-defined skills and plugins can shadow built-in commands.

2. **Availability is not memoized**: `meetsAvailabilityRequirement()` runs fresh on every `getCommands()` call because auth state can change mid-session (e.g., after `/login`).

3. **Dynamic skills** discovered during file operations are inserted before built-in commands but after other skills/plugins.

---

## Lazy Loading Pattern

Almost every command uses the lazy loading pattern. The `index.ts` file exports only lightweight metadata; the actual implementation is deferred to a separate module loaded via `load()`:

```ts
// commands/clear/index.ts — loaded at startup (tiny)
const clear = {
  type: 'local',
  name: 'clear',
  description: 'Clear conversation history and free up context',
  aliases: ['reset', 'new'],
  supportsNonInteractive: false,
  load: () => import('./clear.js'),    // heavy module, loaded on first use
} satisfies Command
```

For `PromptCommand`, the pattern differs. Most define `getPromptForCommand()` inline since the prompt logic is typically lightweight. However, the `/insights` command uses an explicit lazy shim because its module is 113KB:

```ts
const usageReport: Command = {
  type: 'prompt',
  name: 'insights',
  description: 'Generate a report analyzing your Claude Code sessions',
  contentLength: 0,
  progressMessage: 'analyzing your sessions',
  source: 'builtin',
  async getPromptForCommand(args, context) {
    const real = (await import('./commands/insights.js')).default
    if (real.type !== 'prompt') throw new Error('unreachable')
    return real.getPromptForCommand(args, context)
  },
}
```

A few very simple commands (e.g., `/advisor`, `/bridge-kick`, `/version`) define their `call` function inline and return `Promise.resolve({ call })` from `load()`, skipping the separate module entirely.

---

## Feature-Gated Commands

The `feature()` function from `bun:bundle` enables build-time dead code elimination. When a feature flag is off, the bundler removes the `require()` and the entire command module from the final binary.

```ts
const voiceCommand = feature('VOICE_MODE')
  ? require('./commands/voice/index.js').default
  : null

// In COMMANDS():
...(voiceCommand ? [voiceCommand] : []),
```

Feature-gated commands use `require()` (not `import()`) because `feature()` must evaluate at bundle time. The `require()` is synchronous and only executes if the flag is enabled.

### Active Feature Gates

| Feature Flag | Command(s) |
|-------------|------------|
| `PROACTIVE` or `KAIROS` | `/proactive` |
| `KAIROS` or `KAIROS_BRIEF` | `/brief` |
| `KAIROS` | `/assistant` |
| `BRIDGE_MODE` | `/bridge` |
| `DAEMON` + `BRIDGE_MODE` | `/remote-control-server` |
| `VOICE_MODE` | `/voice` |
| `HISTORY_SNIP` | `/force-snip` |
| `WORKFLOW_SCRIPTS` | `/workflows` (command + `getWorkflowCommands` loader) |
| `CCR_REMOTE_SETUP` | `/remote-setup` |
| `KAIROS_GITHUB_WEBHOOKS` | `/subscribe-pr` |
| `ULTRAPLAN` | `/ultraplan` |
| `TORCH` | `/torch` |
| `UDS_INBOX` | `/peers` |
| `FORK_SUBAGENT` | `/fork` |
| `BUDDY` | `/buddy` |
| `EXPERIMENTAL_SKILL_SEARCH` | `clearSkillIndexCache` (not a command, but a cache utility) |
| `MCP_SKILLS` | MCP skill filtering in `getMcpSkillCommands()` |

---

## Filtering Utilities

### REMOTE_SAFE_COMMANDS

A `Set<Command>` of commands safe to use in `--remote` mode. These only affect local TUI state and do not depend on the local filesystem, git, shell, or IDE. Used to pre-filter the command list before the CCR init message arrives.

Members: `session`, `exit`, `clear`, `help`, `theme`, `color`, `vim`, `cost`, `usage`, `copy`, `btw`, `feedback`, `plan`, `keybindings`, `statusline`, `stickers`, `mobile`.

### BRIDGE_SAFE_COMMANDS

A `Set<Command>` of `local`-type commands safe to execute when input arrives over the Remote Control bridge (mobile/web client). `prompt`-type commands are allowed by construction (they just expand to text). `local-jsx` commands are always blocked (they render Ink UI that cannot be displayed remotely).

Members: `compact`, `clear`, `cost`, `summary`, `releaseNotes`, `files`.

### filterCommandsForRemoteMode()

```ts
function filterCommandsForRemoteMode(commands: Command[]): Command[]
```

Returns only the commands in `REMOTE_SAFE_COMMANDS`. Used in `main.tsx` before the REPL renders in `--remote` mode.

### isBridgeSafeCommand()

```ts
function isBridgeSafeCommand(cmd: Command): boolean
```

Returns `true` for `prompt`-type commands, `false` for `local-jsx`, and checks `BRIDGE_SAFE_COMMANDS` membership for `local`-type. Used when slash commands arrive over the Remote Control bridge.

### meetsAvailabilityRequirement()

```ts
function meetsAvailabilityRequirement(cmd: Command): boolean
```

Checks the command's `availability` field against the current auth context. Commands with `availability: ['claude-ai']` are only shown to Claude.ai subscribers. Commands with `availability: ['console']` are shown to direct API key users on `api.anthropic.com`. Commands without `availability` are available everywhere.

---

## Skill Loading Pipeline

Commands don't just come from `src/commands/`. The system also discovers commands from skills, plugins, workflows, and MCP servers.

### getSkills(cwd)

An internal function that loads skills from four sources in parallel:

1. **`getSkillDirCommands(cwd)`** -- scans `.claude/skills/` directories up the directory hierarchy and in additional configured directories. Each `SKILL.md` file with YAML frontmatter becomes a `PromptCommand`.

2. **`getPluginSkills()`** -- loads skill commands from installed plugins.

3. **`getBundledSkills()`** -- returns skills shipped with the binary (synchronous).

4. **`getBuiltinPluginSkillCommands()`** -- returns skills from enabled built-in plugins.

All four sources are error-tolerant: failures are logged and the source returns `[]`.

### getSkillToolCommands(cwd)

```ts
const getSkillToolCommands = memoize(async (cwd: string): Promise<Command[]>)
```

Returns all `prompt`-type commands that the **model** can invoke via SkillTool. Filters for:
- `type === 'prompt'`
- `!disableModelInvocation`
- `source !== 'builtin'`
- Has a user-specified description, `whenToUse`, or is from `bundled`/`skills`/`commands_DEPRECATED` sources

### getSlashCommandToolSkills(cwd)

```ts
const getSlashCommandToolSkills = memoize(async (cwd: string): Promise<Command[]>)
```

Returns commands that are **skills** (as opposed to general commands). Used by the `/skills` listing. Filters for:
- `type === 'prompt'`
- `source !== 'builtin'`
- Has `hasUserSpecifiedDescription` or `whenToUse`
- `loadedFrom` is `'skills'`, `'plugin'`, `'bundled'`, or `disableModelInvocation` is true

### getMcpSkillCommands(mcpCommands)

```ts
function getMcpSkillCommands(mcpCommands: readonly Command[]): readonly Command[]
```

Filters MCP-provided commands to only those that are model-invocable skills. Gated behind `feature('MCP_SKILLS')`. MCP commands live in `AppState.mcp.commands` rather than in the main `getCommands()` pipeline, so callers that need MCP skills in their index thread them through separately.

### Cache Management

Multiple memoization layers cache expensive command loading:

- **`loadAllCommands.cache`** -- memoized by `cwd`.
- **`getSkillToolCommands.cache`** -- memoized by `cwd`.
- **`getSlashCommandToolSkills.cache`** -- memoized by `cwd`.

Two cache-clearing functions:

- **`clearCommandMemoizationCaches()`** -- clears only the memoization caches. Used when dynamic skills are added during a session.
- **`clearCommandsCache()`** -- clears everything: memoization caches, plugin command cache, plugin skills cache, and skill directory caches.

---

## Availability vs. Enablement

Two separate mechanisms control command visibility:

| Mechanism | Purpose | When evaluated |
|-----------|---------|----------------|
| `availability` | Auth/provider requirement (static) | Every `getCommands()` call |
| `isEnabled()` | Dynamic toggle (feature flags, env, platform) | Every `getCommands()` call |
| `isHidden` | UI visibility (hidden from typeahead/help but still invocable) | At render time |

A command with `availability: ['claude-ai']` is completely invisible to API key users. A command with `isEnabled: () => false` is invisible to everyone. A command with `isHidden: true` can still be invoked by typing its name directly.

---

## Naming Conventions

Commands live in one of two places:

- **`src/commands/<name>/index.ts`** -- most commands, with a directory for the metadata and implementation files.
- **`src/commands/<name>.ts`** -- simple commands that need only a single file (e.g., `review.ts`, `commit.ts`, `advisor.ts`).

The metadata file exports the command object as `default`. Some files export multiple commands (e.g., `context/index.ts` exports both `context` and `contextNonInteractive`; `review.ts` exports both `review` and `ultrareview`).
