# Plugin & Skill System

This document describes how Claude Code discovers, loads, and executes **plugins** and **skills**. A plugin is a distributable package (marketplace or built-in) that can contribute commands, skills, hooks, agents, MCP servers, and output styles. A skill is a prompt-type command that the model can invoke at runtime through the `Skill` tool. Skills can originate from plugins, from on-disk `.claude/skills/` directories, from bundled TypeScript code compiled into the CLI, or from MCP servers.

---

## Key Files

| File | Purpose |
|------|---------|
| [`source/src/types/plugin.ts`](../../source/src/types/plugin.ts) | `BuiltinPluginDefinition`, `LoadedPlugin`, `PluginComponent` type definitions |
| [`source/src/types/command.ts`](../../source/src/types/command.ts) | `PromptCommand`, `Command`, `CommandBase` type definitions |
| [`source/src/plugins/builtinPlugins.ts`](../../source/src/plugins/builtinPlugins.ts) | Built-in plugin registry (register, enable/disable, convert to commands) |
| [`source/src/plugins/bundled/index.ts`](../../source/src/plugins/bundled/index.ts) | `initBuiltinPlugins()` -- scaffolding for user-toggleable built-in plugins |
| [`source/src/skills/bundledSkills.ts`](../../source/src/skills/bundledSkills.ts) | `registerBundledSkill()`, `getBundledSkills()`, file extraction for bundled skills |
| [`source/src/skills/bundled/index.ts`](../../source/src/skills/bundled/index.ts) | `initBundledSkills()` -- registers all bundled skills at startup |
| [`source/src/skills/loadSkillsDir.ts`](../../source/src/skills/loadSkillsDir.ts) | Loads skills from `/skills/` and legacy `/commands/` directories, dynamic skill discovery, conditional path-filtered skills |
| [`source/src/skills/mcpSkillBuilders.ts`](../../source/src/skills/mcpSkillBuilders.ts) | Dependency-inversion bridge so MCP code can call `createSkillCommand` / `parseSkillFrontmatterFields` without import cycles |
| [`source/src/utils/plugins/loadPluginCommands.ts`](../../source/src/utils/plugins/loadPluginCommands.ts) | `getPluginCommands()`, `getPluginSkills()`, `clearPluginCommandCache()`, `clearPluginSkillsCache()` |
| [`source/src/utils/plugins/cacheUtils.ts`](../../source/src/utils/plugins/cacheUtils.ts) | `clearAllPluginCaches()`, `clearAllCaches()`, orphaned plugin version cleanup |
| [`source/src/commands.ts`](../../source/src/commands.ts) | `loadAllCommands()`, `getCommands()`, `getSkillToolCommands()`, `clearCommandsCache()` -- the central aggregation point |
| [`source/src/tools/SkillTool/SkillTool.ts`](../../source/src/tools/SkillTool/SkillTool.ts) | The `Skill` tool definition -- validates, permissions-checks, and executes skills |
| [`source/src/tools/SkillTool/prompt.ts`](../../source/src/tools/SkillTool/prompt.ts) | Builds the Skill tool's system prompt listing, with budget-aware description truncation |
| [`source/src/tools/SkillTool/constants.ts`](../../source/src/tools/SkillTool/constants.ts) | `SKILL_TOOL_NAME = 'Skill'` |

---

## Plugin Architecture

### Plugin type definitions

A **plugin** is represented at runtime as a `LoadedPlugin` (defined in `source/src/types/plugin.ts`). Key fields:

```
LoadedPlugin {
  name: string
  manifest: PluginManifest
  path: string               // filesystem path, or 'builtin' sentinel
  source: string             // e.g. "my-plugin@marketplace-name" or "my-plugin@builtin"
  repository: string
  enabled?: boolean
  isBuiltin?: boolean        // true for built-in plugins
  commandsPath?: string      // default commands/ directory
  commandsPaths?: string[]   // additional command paths from manifest
  commandsMetadata?: Record<string, CommandMetadata>
  skillsPath?: string        // default skills/ directory
  skillsPaths?: string[]     // additional skill paths from manifest
  hooksConfig?: HooksSettings
  mcpServers?: Record<string, McpServerConfig>
  ...
}
```

For built-in plugins specifically, the definition type is `BuiltinPluginDefinition`:

```
BuiltinPluginDefinition {
  name: string
  description: string
  version?: string
  skills?: BundledSkillDefinition[]
  hooks?: HooksSettings
  mcpServers?: Record<string, McpServerConfig>
  isAvailable?: () => boolean
  defaultEnabled?: boolean
}
```

Plugin IDs use the format `{name}@{marketplace}`. Built-in plugins use the sentinel marketplace name `builtin`, producing IDs like `my-plugin@builtin`.

### How plugins register commands and skills

Plugins contribute skills and commands through two mechanisms:

1. **Filesystem-based**: Plugin directories contain `commands/` and `skills/` subdirectories with markdown files. `getPluginCommands()` walks `commandsPath`/`commandsPaths` and `getPluginSkills()` walks `skillsPath`/`skillsPaths`. Skill directories must contain a `SKILL.md` file. Regular command files are any `.md` file. Names are derived as `{pluginName}:{commandBaseName}`, with nested directories producing colon-separated namespaces (e.g., `my-plugin:sub:cmd`).

2. **Inline content**: Plugin manifests can declare commands via `commandsMetadata` with a `content` field (no `source` file reference). These are parsed identically to file-based commands but use a virtual file path.

For built-in plugins, skills are provided as `BundledSkillDefinition` objects on the `skills` array. These are converted to `Command` objects by `skillDefinitionToCommand()` in `builtinPlugins.ts`, which sets `source: 'bundled'` and `loadedFrom: 'bundled'`.

### Loading pipeline

Plugin loading follows this sequence:

1. `loadAllPluginsCacheOnly()` (from `pluginLoader.ts`) loads all enabled marketplace plugins from the local cache (no network).
2. `getPluginCommands()` iterates over each enabled plugin, calling `loadCommandsFromDirectory()` for filesystem-based commands and processing `commandsMetadata` entries for inline commands.
3. `getPluginSkills()` similarly iterates enabled plugins, calling `loadSkillsFromDirectory()` to find `SKILL.md` files in each plugin's skills directories.
4. Both functions run in parallel per-plugin. Within each plugin, a `loadedPaths` set prevents duplicate file loading.

Both `getPluginCommands()` and `getPluginSkills()` are memoized with `lodash-es/memoize`. In `--bare` mode, marketplace plugins are skipped unless explicit `--plugin-dir` paths are provided.

### Caching

All plugin loading functions use lodash `memoize()`. This means the first call performs disk I/O and all subsequent calls return the cached result. Cache invalidation is explicit -- see [Cache Invalidation](#cache-invalidation) below.

---

## Bundled Plugins

### What ships built-in

The `source/src/plugins/bundled/index.ts` file defines `initBuiltinPlugins()`, which is the entry point for registering built-in plugins. Currently no built-in plugins are registered -- the file is scaffolding for migrating bundled skills that should be user-toggleable via the `/plugin` UI.

The distinction between "bundled skills" and "built-in plugins" is important:

- **Bundled skills** (`source/src/skills/bundled/`): Always-on skills compiled into the CLI. Registered via `registerBundledSkill()`. Cannot be toggled by users.
- **Built-in plugins** (`source/src/plugins/bundled/`): Appear in the `/plugin` UI under a "Built-in" section. Users can enable/disable them (persisted to user settings). Can provide skills, hooks, and MCP servers.

### builtinPlugins.ts contents

The `builtinPlugins.ts` module manages a `Map<string, BuiltinPluginDefinition>` registry:

- `registerBuiltinPlugin(definition)` -- adds a plugin to the map.
- `getBuiltinPlugins()` -- returns `{ enabled, disabled }` arrays of `LoadedPlugin` objects, checking each plugin's `isAvailable()` and the user's settings (`enabledPlugins[pluginId]`). Falls back to `defaultEnabled` (defaults to `true`).
- `getBuiltinPluginSkillCommands()` -- returns `Command[]` from all enabled built-in plugins' skill definitions. Calls `skillDefinitionToCommand()` which maps `BundledSkillDefinition` to a `Command` with `source: 'bundled'`.
- `isBuiltinPluginId(pluginId)` -- checks if an ID ends with `@builtin`.
- `clearBuiltinPlugins()` -- for testing.

### Bundled skills catalog

The following skills are registered in `source/src/skills/bundled/index.ts` via `initBundledSkills()`:

| Registration call | Feature gate |
|---|---|
| `registerUpdateConfigSkill()` | always |
| `registerKeybindingsSkill()` | always |
| `registerVerifySkill()` | always |
| `registerDebugSkill()` | always |
| `registerLoremIpsumSkill()` | always |
| `registerSkillifySkill()` | always |
| `registerRememberSkill()` | always |
| `registerSimplifySkill()` | always |
| `registerBatchSkill()` | always |
| `registerStuckSkill()` | always |
| `registerDreamSkill()` | `KAIROS` or `KAIROS_DREAM` |
| `registerHunterSkill()` | `REVIEW_ARTIFACT` |
| `registerLoopSkill()` | `AGENT_TRIGGERS` |
| `registerScheduleRemoteAgentsSkill()` | `AGENT_TRIGGERS_REMOTE` |
| `registerClaudeApiSkill()` | `BUILDING_CLAUDE_APPS` |
| `registerClaudeInChromeSkill()` | `shouldAutoEnableClaudeInChrome()` |
| `registerRunSkillGeneratorSkill()` | `RUN_SKILL_GENERATOR` |

Additionally, some bundled skills have reference files (the `files` field on `BundledSkillDefinition`). When present, `registerBundledSkill()` sets up lazy extraction: on first invocation, files are written to a deterministic directory under `getBundledSkillsRoot()`, and the prompt is prefixed with `Base directory for this skill: <dir>`. Extraction uses `O_EXCL | O_NOFOLLOW` flags and `0o600` permissions for security.

---

## Plugin Command Loading

The central aggregation happens in `commands.ts`. The key functions:

### getPluginCommands()

Defined in `source/src/utils/plugins/loadPluginCommands.ts`. Memoized. Loads from all enabled marketplace plugins:

1. Calls `loadAllPluginsCacheOnly()` to get the enabled plugin list.
2. For each plugin, loads commands from `commandsPath` (default directory), `commandsPaths` (additional manifest paths), and `commandsMetadata` (inline content entries).
3. Each markdown file is parsed for YAML frontmatter, and `createPluginCommand()` builds a `Command` with `source: 'plugin'`. Plugin variable substitution (`${CLAUDE_PLUGIN_ROOT}`, `${CLAUDE_PLUGIN_DATA}`, `${user_config.X}`) is performed at invocation time inside `getPromptForCommand()`.

### getPluginSkills()

Also in `loadPluginCommands.ts`. Memoized. Loads from plugin `skills/` directories:

1. Same `loadAllPluginsCacheOnly()` call.
2. For each plugin, scans `skillsPath` and `skillsPaths` for subdirectories containing `SKILL.md`.
3. Creates commands with `source: 'plugin'`, `loadedFrom: 'plugin'`, and `isSkillMode: true` (which prepends `Base directory for this skill:` to the prompt).

### loadAllCommands() in commands.ts

The `loadAllCommands(cwd)` function is the memoized aggregator. It runs in parallel:

```
const [
  { skillDirCommands, pluginSkills, bundledSkills, builtinPluginSkills },
  pluginCommands,
  workflowCommands,
] = await Promise.all([
  getSkills(cwd),          // skill dirs + plugin skills + bundled + builtin plugin skills
  getPluginCommands(),     // marketplace plugin commands/
  getWorkflowCommands?.(), // workflow scripts (feature-gated)
])
```

The result is combined in this order:
1. `bundledSkills` (from `getBundledSkills()`)
2. `builtinPluginSkills` (from `getBuiltinPluginSkillCommands()`)
3. `skillDirCommands` (from `getSkillDirCommands()`)
4. `workflowCommands`
5. `pluginCommands`
6. `pluginSkills`
7. Built-in CLI commands (`COMMANDS()`)

`getCommands(cwd)` wraps `loadAllCommands()` with fresh `meetsAvailabilityRequirement()` and `isCommandEnabled()` checks (not memoized, so auth state changes take effect immediately), plus deduplication of dynamic skills.

---

## Skill System

### Skill discovery

Skills are discovered from multiple sources and directories. The `getSkillDirCommands(cwd)` function in `loadSkillsDir.ts` is the primary loader for filesystem-based skills. It loads from these directories in parallel:

1. **Managed (policy)**: `{managedFilePath}/.claude/skills/` -- policy-controlled, can be disabled via `CLAUDE_CODE_DISABLE_POLICY_SKILLS`.
2. **User**: `~/.claude/skills/` -- user's personal skills.
3. **Project**: `.claude/skills/` in every directory from `cwd` up to `$HOME` -- project-specific skills.
4. **Additional dirs**: Paths from `--add-dir` flag, looking in `{dir}/.claude/skills/`.
5. **Legacy commands**: `.claude/commands/` directories (deprecated, loaded via `loadSkillsFromCommandsDir()`).

In `--bare` mode, only explicit `--add-dir` paths are loaded.

### loadSkillsDir.ts

This file is the backbone of skill loading. Key exports:

- **`getSkillDirCommands(cwd)`** -- memoized main entry point. Loads all filesystem skills, deduplicates by resolved path (using `realpath()` to handle symlinks), and separates conditional skills (those with `paths` frontmatter) from unconditional ones. Conditional skills are stored in a `Map` and only activated when the model touches matching files.

- **`createSkillCommand({...})`** -- constructs a `Command` from parsed skill data. Sets up `getPromptForCommand()` which: prepends the base directory, substitutes `$ARGUMENTS` and named arguments, replaces `${CLAUDE_SKILL_DIR}` and `${CLAUDE_SESSION_ID}`, and executes inline shell commands (`!command` syntax) via `executeShellCommandsInPrompt()`. MCP-sourced skills skip shell execution for security.

- **`parseSkillFrontmatterFields(frontmatter, markdownContent, resolvedName)`** -- extracts all frontmatter fields shared between file-based and MCP skill loading: description, allowedTools, argumentHint, whenToUse, model, disableModelInvocation, userInvocable, hooks, context (fork/inline), agent, effort, shell, paths, etc.

- **`discoverSkillDirsForPaths(filePaths, cwd)`** -- dynamic skill discovery. When the model reads/writes/edits files, this function walks up from each file path looking for `.claude/skills/` directories below `cwd`. Newly discovered directories are loaded and added as "dynamic skills." Gitignored directories are skipped.

- **`clearSkillCaches()`** -- clears the `getSkillDirCommands` memoization cache, the `loadMarkdownFilesForSubdir` cache, and the conditional/activated skill sets.

- **`getDynamicSkills()`** -- returns skills that were discovered during file operations (not at startup).

### Bundled skills

Bundled skills are TypeScript modules in `source/src/skills/bundled/`. Each module exports a `register*Skill()` function that calls `registerBundledSkill(definition)` from `bundledSkills.ts`. The definition provides a `getPromptForCommand(args, context)` function that returns `ContentBlockParam[]`.

The `registerBundledSkill()` function converts a `BundledSkillDefinition` into a `Command` with `source: 'bundled'`, `loadedFrom: 'bundled'`, and pushes it to an internal array. `getBundledSkills()` returns a copy of this array.

### mcpSkillBuilders.ts

This module solves a circular dependency problem. MCP skill discovery code (in `mcpSkills.ts`) needs `createSkillCommand` and `parseSkillFrontmatterFields` from `loadSkillsDir.ts`, but the reverse dependency chain (`client.ts -> mcpSkills.ts -> loadSkillsDir.ts -> ... -> client.ts`) would create a cycle.

The solution is a write-once registry:

1. `mcpSkillBuilders.ts` defines a `MCPSkillBuilders` type holding references to `createSkillCommand` and `parseSkillFrontmatterFields`.
2. `loadSkillsDir.ts` calls `registerMCPSkillBuilders()` at module initialization time (early in startup).
3. MCP code calls `getMCPSkillBuilders()` to retrieve the functions without importing `loadSkillsDir.ts` directly.

This pattern avoids both static import cycles and the fragility of dynamic imports in bundled (Bun) binaries.

---

## SkillTool

The `Skill` tool (`source/src/tools/SkillTool/SkillTool.ts`) is how the model invokes skills at runtime. It accepts `{ skill: string, args?: string }` as input.

### Execution flow

1. **Validation** (`validateInput`): Normalizes the skill name (strips leading `/`), looks it up via `getAllCommands(context)` (which merges local commands with MCP skills from `AppState.mcp.commands`), and checks that the command exists, is `type: 'prompt'`, and does not have `disableModelInvocation` set.

2. **Permission check** (`checkPermissions`): Checks deny rules, then allow rules (both support exact match and prefix wildcards like `review:*`). Skills with only "safe properties" (no hooks, no allowedTools, no fork context, no paths, no model override, etc.) are auto-allowed. Otherwise, the user is prompted.

3. **Execution** (`call`): Two execution modes:
   - **Inline** (default): Calls `processPromptSlashCommand()` to expand the skill's `getPromptForCommand()` into message content. The expanded content is injected into the conversation as new messages. The tool result also carries a `contextModifier` that can override `allowedTools`, `model`, and `effort` for subsequent turns.
   - **Forked** (`context: 'fork'`): Runs the skill in an isolated sub-agent via `runAgent()`. The sub-agent has its own token budget and agent definition. Results are extracted and returned as a string in the tool result.

### getPromptForCommand() expansion

When a skill's `getPromptForCommand(args, context)` is called, the typical sequence is:

1. Prepend `Base directory for this skill: {baseDir}` if the skill has a base directory.
2. Substitute `$ARGUMENTS` and named arguments (e.g., `$1`, `$2`, or `${argName}`).
3. Replace `${CLAUDE_SKILL_DIR}` with the skill's directory path.
4. Replace `${CLAUDE_SESSION_ID}` with the current session ID.
5. For plugin commands: replace `${CLAUDE_PLUGIN_ROOT}` and `${CLAUDE_PLUGIN_DATA}` with plugin paths, and `${user_config.X}` with saved option values.
6. Execute inline shell commands (`!command` blocks) via `executeShellCommandsInPrompt()` -- skipped for MCP-sourced skills for security.
7. Return `[{ type: 'text', text: finalContent }]`.

### Skill listing in the system prompt

The `prompt.ts` module builds the Skill tool's prompt, which includes a listing of available skills. The listing is budget-constrained to 1% of the context window (in characters). Bundled skills always get full descriptions. Non-bundled skill descriptions are truncated proportionally to fit within budget. In extreme cases, non-bundled skills are listed by name only.

`getSkillToolCommands(cwd)` filters commands to include only `type: 'prompt'` commands that are not `disableModelInvocation`, not `source: 'builtin'` (which refers to hardcoded CLI commands like `/help` and `/clear`), and have either a user-specified description, a `whenToUse`, or are loaded from `bundled`, `skills`, or `commands_DEPRECATED`.

---

## Skill Types

Skills are `Command` objects with `type: 'prompt'`. The `PromptCommand` type (from `source/src/types/command.ts`) has these key fields:

| Field | Type | Description |
|-------|------|-------------|
| `source` | `SettingSource \| 'builtin' \| 'mcp' \| 'plugin' \| 'bundled'` | Origin of the command. `'builtin'` = hardcoded CLI commands; `'bundled'` = compiled-in skills; `'plugin'` = marketplace plugin; `'mcp'` = MCP server; `SettingSource` = filesystem settings (user/project/policy). |
| `pluginInfo` | `{ pluginManifest: PluginManifest, repository: string }` | Present only for plugin-sourced commands. Contains the plugin's manifest and repository identifier. |
| `skillRoot` | `string \| undefined` | Base directory for skill resources. Used to set `CLAUDE_PLUGIN_ROOT` for skill hooks. Set by `createSkillCommand()` for directory-based skills and by `registerBundledSkill()` for bundled skills with extracted files. |
| `context` | `'inline' \| 'fork' \| undefined` | Execution mode. `'inline'` (default): skill content expands into current conversation. `'fork'`: skill runs as an isolated sub-agent with separate context and token budget. |
| `agent` | `string \| undefined` | Agent type to use when forked (e.g., `'Bash'`, `'general-purpose'`). Only meaningful when `context` is `'fork'`. |
| `hooks` | `HooksSettings \| undefined` | Hooks to register when the skill is invoked. Validated against `HooksSchema` at parse time. |
| `paths` | `string[] \| undefined` | Glob patterns for file paths this skill applies to. When set, the skill starts as conditional and only becomes visible after the model touches matching files. |
| `allowedTools` | `string[] \| undefined` | Tools that are auto-allowed when this skill is active. Injected into `toolPermissionContext.alwaysAllowRules.command`. |
| `model` | `string \| undefined` | Model override for the skill. Can be an alias like `'haiku'`, `'sonnet'`, `'opus'`, or a full model ID. |
| `effort` | `EffortValue \| undefined` | Effort level override for the skill. |
| `contentLength` | `number` | Length of command content in characters (for token estimation). |
| `disableModelInvocation` | `boolean` | If true, the model cannot invoke this skill through the Skill tool. |
| `userInvocable` | `boolean` | If true (default), users can invoke the skill by typing `/skill-name`. |
| `getPromptForCommand` | `(args: string, context: ToolUseContext) => Promise<ContentBlockParam[]>` | The function that produces the skill's prompt content when invoked. |
| `loadedFrom` | `'commands_DEPRECATED' \| 'skills' \| 'plugin' \| 'managed' \| 'bundled' \| 'mcp'` | Distinguishes the loading mechanism. Affects filtering in `getSkillToolCommands()` and `getSlashCommandToolSkills()`. |

---

## Cache Invalidation

The plugin and skill system uses multiple layers of memoization. Each layer has a dedicated clear function:

### clearPluginCommandCache()

Defined in `source/src/utils/plugins/loadPluginCommands.ts`. Clears the memoization cache for `getPluginCommands()`. After clearing, the next call will re-walk all enabled plugin `commands/` directories.

### clearPluginSkillsCache()

Also in `loadPluginCommands.ts`. Clears the memoization cache for `getPluginSkills()`. After clearing, the next call will re-scan all enabled plugin `skills/` directories.

### clearCommandsCache()

Defined in `source/src/commands.ts`. This is the comprehensive clear function. It calls:

1. `clearCommandMemoizationCaches()` -- clears `loadAllCommands.cache`, `getSkillToolCommands.cache`, `getSlashCommandToolSkills.cache`, and `clearSkillIndexCache()` (if the experimental skill search feature is enabled).
2. `clearPluginCommandCache()` -- see above.
3. `clearPluginSkillsCache()` -- see above.
4. `clearSkillCaches()` -- clears `getSkillDirCommands.cache`, `loadMarkdownFilesForSubdir.cache`, and the conditional/activated skill name sets.

### clearAllPluginCaches()

Defined in `source/src/utils/plugins/cacheUtils.ts`. Broader than command caches -- also clears `clearPluginCache()` (the plugin loader itself), `clearPluginAgentCache()`, `clearPluginHookCache()` (with async pruning of hooks from removed plugins), `clearPluginOptionsCache()`, `clearPluginOutputStyleCache()`, and `clearAllOutputStylesCache()`.

### clearAllCaches()

Also in `cacheUtils.ts`. Calls `clearAllPluginCaches()`, then `clearCommandsCache()`, `clearAgentDefinitionsCache()`, `clearPromptCache()` (the Skill tool's system prompt), and `resetSentSkillNames()`. This is the nuclear option -- used by `/reload-plugins`.

### When invalidation happens

- **`/reload-plugins` command**: Triggers `clearAllCaches()`.
- **Dynamic skill discovery**: When `discoverSkillDirsForPaths()` finds new skill directories, it calls `clearCommandMemoizationCaches()` (not the full clear) so the new skills appear in subsequent `getCommands()` calls.
- **Plugin install/uninstall/update**: The plugin management UI triggers cache clears through `clearAllPluginCaches()`.
