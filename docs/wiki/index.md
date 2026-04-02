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
| [Architecture Overview](architecture-overview.md) | Narrative spine — 5 Mermaid diagrams tracing data flow from CLI entry to tool execution |
| [Query Engine](query-engine.md) | QueryEngine/query() state machine, 6 compaction strategies, stop conditions, system prompt assembly |
| [Tool System](tool-system.md) | Tool interface anatomy, ToolUseContext, StreamingToolExecutor, permission pipeline |
| [Tool Catalog](tool-catalog.md) | All 47+ built-in tools cataloged by category with parameters and feature gates |
| [Command System](command-system.md) | PromptCommand/LocalCommand/LocalJSXCommand types, registration, dispatch, skill loading |
| [Command Catalog](command-catalog.md) | All 87+ commands cataloged by category with types, aliases, and feature gates |

### Infrastructure

| Page | Description |
|------|-------------|
| [State Management](state-management.md) | Bootstrap singletons, AppStateStore (~60 fields), 9 React context providers |
| [Ink TUI](ink-tui.md) | Custom Ink fork with in-tree Yoga TS port, blit optimizer, Kitty keyboard protocol |
| [Permissions & Security](permissions-security.md) | Permission modes, deny/allow rules, hook-based overrides, bash safety classifier |

### Subsystems

| Page | Description |
|------|-------------|
| [MCP Integration](mcp-integration.md) | MCPServerConnection lifecycle, 8 transport types, tool bridging, OAuth, Claude AI proxy |
| [Task System](task-system.md) | 7 task variants, LocalShell/LocalAgent/InProcessTeammate/Dream, coordinator mode |
| [Plugin & Skill System](plugin-skill-system.md) | Plugin loading, skill discovery from directories, SkillTool invocation, cache invalidation |
| [Service Layer](service-layer.md) | API client with retries, compaction, GrowthBook analytics, OAuth PKCE, LSP integration |
| [Component Layer](component-layer.md) | REPL/Doctor screens, 31+ component directories, 85+ hooks, dialog launcher pattern |
| [Cost Tracking](cost-tracking.md) | Per-model token accounting, calculateUSDCost, session persistence, formatTotalCost |

### Peripheral Systems & Build

| Page | Description |
|------|-------------|
| [Peripheral Systems](peripheral-systems.md) | Voice, Vim mode, Bridge (32 files), Keybindings, Buddy companion, migrations |
| [Build System](build-system.md) | Source map extraction, shim generation, stub inference, feature patching, Bun bundle |
| [Reconstruction Fidelity](reconstruction-fidelity.md) | Stubbed @ant/* packages, native addons, 81-flag inventory, pinned React files |
