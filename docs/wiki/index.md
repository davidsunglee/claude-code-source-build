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
