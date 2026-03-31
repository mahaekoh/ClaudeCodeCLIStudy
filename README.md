# Claude Code — Source

The source code for **Claude Code**, Anthropic's official AI-powered CLI and development tool. Claude Code provides an interactive terminal environment for working with Claude models, enabling file operations, shell execution, code analysis, git workflows, and multi-agent coordination — all from the command line.

**Product URL:** [https://claude.com/claude-code](https://claude.com/claude-code)

---

## Table of Contents

- [Overview](#overview)
- [Tech Stack](#tech-stack)
- [Directory Structure](#directory-structure)
- [Architecture](#architecture)
  - [Entry Points](#entry-points)
  - [Query Engine](#query-engine)
  - [Tool System](#tool-system)
  - [Command System](#command-system)
  - [State Management](#state-management)
  - [Terminal UI (Ink)](#terminal-ui-ink)
  - [Services Layer](#services-layer)
  - [Plugin & Skill System](#plugin--skill-system)
  - [Model Context Protocol (MCP)](#model-context-protocol-mcp)
- [Key Modules](#key-modules)
- [Feature Flags](#feature-flags)
- [Configuration](#configuration)
- [Security & Permissions](#security--permissions)
- [Notable Patterns](#notable-patterns)

---

## Overview

Claude Code is a feature-rich CLI that turns the terminal into an AI-powered development environment. It supports:

- **Interactive REPL** — conversational coding with Claude in the terminal
- **File operations** — reading, editing, writing, and searching files
- **Shell execution** — running commands with output capture and streaming
- **Git integration** — commits, PRs, branch management, code review
- **Multi-agent coordination** — spawning sub-agents for parallel tasks
- **MCP servers** — extending capabilities via Model Context Protocol
- **IDE integration** — bridging with VS Code, JetBrains, and other editors
- **Session management** — persistence, restore, teleportation across machines
- **Voice input/output** — speech-to-text and text-to-speech interfaces
- **Enterprise features** — MDM policies, OAuth, remote managed settings

---

## Tech Stack

| Layer | Technology |
|---|---|
| **Runtime** | [Bun](https://bun.sh) |
| **Language** | TypeScript |
| **UI Framework** | React + custom Ink terminal renderer |
| **AI Backend** | Anthropic Claude API via `@anthropic-ai/sdk` |
| **CLI Parsing** | `@commander-js/extra-typings` |
| **Schema Validation** | Zod |
| **Styling** | Chalk (terminal colors) |
| **Search** | Embedded ripgrep binary |
| **MCP** | `@modelcontextprotocol/sdk` |
| **Utilities** | lodash-es |

---

## Directory Structure

```
src/
├── main.tsx                 # Primary entry point — CLI setup, arg parsing, REPL launch
├── QueryEngine.ts           # Query execution orchestrator — turns, token budgets, retries
├── query.ts                 # Query lifecycle — message processing, tool orchestration
├── Tool.ts                  # Tool interface, permissions, execution context types
├── tools.ts                 # Tool registry — assembles the active tool pool
├── commands.ts              # Command registry — 100+ slash commands
├── context.ts               # System/user context builders (git state, env, date)
├── cost-tracker.ts          # Token and cost tracking across sessions
├── history.ts               # Session history persistence
│
├── entrypoints/             # Application entry points
│   ├── cli.tsx              #   CLI bootstrap with fast-path handling
│   ├── init.ts              #   Initialization (auth, settings, setup screens)
│   ├── mcp.ts               #   MCP server mode entry
│   └── sdk/                 #   SDK mode for programmatic access
│
├── tools/                   # Tool implementations (45+)
│   ├── AgentTool/           #   Sub-agent spawning and coordination
│   ├── BashTool/            #   Shell command execution
│   ├── FileReadTool/        #   File reading with line ranges
│   ├── FileEditTool/        #   Surgical string replacement edits
│   ├── FileWriteTool/       #   Full file writes
│   ├── GlobTool/            #   File pattern matching
│   ├── GrepTool/            #   Content search via ripgrep
│   ├── WebFetchTool/        #   HTTP fetch
│   ├── WebSearchTool/       #   Web search
│   ├── LSPTool/             #   Language Server Protocol integration
│   ├── MCPTool/             #   MCP server tool proxy
│   ├── SkillTool/           #   Skill invocation
│   ├── NotebookEditTool/    #   Jupyter notebook editing
│   ├── EnterPlanModeTool/   #   Plan mode entry
│   ├── EnterWorktreeTool/   #   Git worktree isolation
│   ├── TaskCreateTool/      #   Task management (create)
│   ├── TaskUpdateTool/      #   Task management (update)
│   ├── SendMessageTool/     #   Inter-agent messaging
│   ├── TeamCreateTool/      #   Team/swarm creation
│   ├── ScheduleCronTool/    #   Scheduled agent triggers
│   └── ...                  #   And many more
│
├── commands/                # Slash command implementations (100+)
│   ├── commit.js            #   /commit — AI-powered git commits
│   ├── review.js            #   /review — PR code review
│   ├── commit-push-pr.js    #   /commit-push-pr — full PR workflow
│   ├── compact/             #   /compact — context compaction
│   ├── config/              #   /config — settings management
│   ├── doctor/              #   /doctor — diagnostics
│   ├── mcp/                 #   /mcp — MCP server management
│   ├── teleport/            #   /teleport — remote development
│   ├── memory/              #   /memory — persistent memory management
│   ├── skills/              #   /skills — skill management
│   ├── plugin/              #   /plugin — plugin management
│   ├── agents/              #   /agents — agent definitions
│   ├── desktop/             #   /desktop — desktop app handoff
│   ├── mobile/              #   /mobile — mobile integration
│   ├── chrome/              #   /chrome — Chrome extension
│   ├── theme/               #   /theme — UI theming
│   └── ...                  #   And many more
│
├── components/              # React terminal UI components (146 files)
│   ├── App.tsx              #   Root application component
│   ├── diff/                #   Diff visualization components
│   ├── design-system/       #   Reusable UI primitives
│   ├── agents/              #   Agent status display
│   └── ...                  #   Dialogs, inputs, status bars, etc.
│
├── hooks/                   # Custom React hooks (87 files)
│   ├── useCanUseTool.tsx    #   Tool permission checking
│   ├── useGlobalKeybindings.tsx  # Keyboard input handling
│   ├── useIDEIntegration.tsx     # IDE bridge hooks
│   └── ...                  #   Cancellation, history, clipboard, etc.
│
├── services/                # Service layer
│   ├── api/                 #   Claude API client, retries, logging
│   ├── mcp/                 #   MCP client/server, auth, config
│   ├── analytics/           #   Telemetry, GrowthBook feature flags
│   ├── compact/             #   Conversation compaction
│   ├── lsp/                 #   Language Server Protocol
│   ├── oauth/               #   OAuth 2.0 flows
│   ├── plugins/             #   Plugin lifecycle management
│   ├── policyLimits/        #   Enterprise policy enforcement
│   ├── remoteManagedSettings/  # Remote config management
│   └── ...                  #   Diagnostics, tips, voice, etc.
│
├── state/                   # Application state management
│   ├── AppState.tsx         #   React context provider
│   ├── AppStateStore.ts     #   Central store definition
│   ├── store.ts             #   Zustand-based store implementation
│   └── selectors.ts         #   State selectors
│
├── constants/               # Configuration constants
│   ├── prompts.ts           #   System prompts
│   ├── oauth.ts             #   OAuth configuration
│   ├── product.ts           #   Product URLs and identifiers
│   ├── betas.ts             #   Beta feature flags
│   ├── tools.ts             #   Tool-related constants
│   └── ...                  #   Error IDs, limits, XML tags, etc.
│
├── skills/                  # Skill system
│   ├── bundled/             #   Built-in skills
│   ├── bundledSkills.ts     #   Bundled skill registry
│   └── loadSkillsDir.ts    #   Custom skill loading
│
├── plugins/                 # Plugin infrastructure
│   └── bundled/             #   Built-in plugins
│
├── ink/                     # Custom terminal rendering engine
│   ├── ink.tsx              #   Main Ink implementation (251KB)
│   └── components/          #   Base components (Box, Text, etc.)
│
├── types/                   # TypeScript type definitions
│   ├── message.ts           #   Message types
│   ├── permissions.ts       #   Permission types
│   ├── tools.ts             #   Tool progress types
│   └── command.ts           #   Command types
│
├── utils/                   # Utilities (331+ files)
│   ├── auth.ts              #   Authentication helpers
│   ├── config.ts            #   Configuration management
│   ├── git.ts               #   Git operations
│   ├── tokens.ts            #   Token counting/estimation
│   ├── messages.ts          #   Message construction
│   ├── claudemd.ts          #   CLAUDE.md file loading
│   ├── fileHistory.ts       #   File snapshots for undo
│   ├── settings/            #   Settings management (MDM, etc.)
│   ├── secureStorage/       #   Keychain integration
│   ├── permissions/         #   Permission evaluation
│   ├── plugins/             #   Plugin loading utilities
│   ├── swarm/               #   Multi-agent swarm utilities
│   └── ...                  #   Hundreds more helpers
│
├── context/                 # React context providers
├── migrations/              # Data/config migration scripts
├── schemas/                 # JSON schemas
├── keybindings/             # Terminal keyboard binding system
├── memdir/                  # Persistent memory file handling
├── remote/                  # Remote session management
├── coordinator/             # Multi-agent coordinator mode
├── assistant/               # Assistant/proactive mode (Kairos)
├── voice/                   # Voice input/output
├── vim/                     # Vim mode integration
├── bootstrap/               # Bootstrap state (session, model, args)
├── query/                   # Query processing internals
└── tasks/                   # Task management system
```

---

## Architecture

### Entry Points

The application has several entry points depending on the execution mode:

| Entry Point | File | Purpose |
|---|---|---|
| **CLI** | `entrypoints/cli.tsx` | Bootstrap, fast-path detection, delegates to `main.tsx` |
| **Main** | `main.tsx` | Full CLI setup — arg parsing, auth, config, REPL launch |
| **SDK** | `entrypoints/sdk/` | Programmatic API for embedding Claude Code |
| **MCP Server** | `entrypoints/mcp.ts` | Runs Claude Code as an MCP server |

`main.tsx` is the primary orchestrator. It handles:
1. **Startup profiling** — `profileCheckpoint` markers for performance analysis
2. **Prefetching** — MDM reads, keychain access, and MCP URLs in parallel
3. **CLI parsing** — Commander.js with typed options
4. **Authentication** — OAuth, API keys, keychain retrieval
5. **Feature flags** — GrowthBook initialization
6. **REPL launch** — Interactive session or one-shot execution

### Query Engine

The `QueryEngine` (`QueryEngine.ts`) is the core execution loop:

1. Receives user messages (text, tool results, attachments)
2. Constructs the API request with system prompts, messages, and tool definitions
3. Streams responses from the Claude API
4. Handles tool use blocks — dispatches to the tool system
5. Manages token budgets and triggers compaction when needed
6. Supports retries with exponential backoff on transient errors
7. Tracks costs and usage across turns

The lower-level `query()` function (`query.ts`) handles a single API call lifecycle — streaming, tool execution, stop conditions, and error classification.

### Tool System

Tools are the primary way Claude interacts with the outside world. Each tool implements a standard interface defined in `Tool.ts`:

```typescript
interface Tool {
  name: string
  description: string
  inputSchema: ToolInputJSONSchema
  isEnabled: (context) => boolean
  isReadOnly: () => boolean
  needsPermissions: (input) => boolean
  prompt: (context) => string   // Dynamic tool instructions
  call: (input, context) => Promise<ToolResult>
}
```

The tool registry (`tools.ts`) assembles the active tool pool based on:
- Feature flags (compile-time dead code elimination via `bun:bundle`)
- User type (`ant` internal vs external)
- Permission mode (plan mode, worktree mode, etc.)
- MCP-provided tools

**Built-in tools include:**

| Category | Tools |
|---|---|
| **File I/O** | FileRead, FileEdit, FileWrite, Glob, Grep, NotebookEdit |
| **Execution** | Bash, REPL (internal) |
| **Search** | Glob, Grep, WebSearch, WebFetch, ToolSearch |
| **Agents** | Agent (sub-agents), SendMessage, TeamCreate, TeamDelete |
| **Tasks** | TaskCreate, TaskGet, TaskList, TaskUpdate, TaskStop, TaskOutput |
| **Planning** | EnterPlanMode, ExitPlanMode |
| **Workspace** | EnterWorktree, ExitWorktree |
| **MCP** | MCPTool (proxy), ListMcpResources, ReadMcpResource |
| **Skills** | SkillTool |
| **Scheduling** | CronCreate, CronDelete, CronList, RemoteTrigger |
| **Other** | AskUserQuestion, LSP, Brief, SyntheticOutput |

### Command System

Slash commands (`/command`) provide user-invocable actions. The command registry (`commands.ts`) supports 100+ commands:

**Core workflow:**
- `/commit` — AI-powered git commit with message generation
- `/review` — Automated PR code review
- `/commit-push-pr` — Full commit-push-PR workflow
- `/branch` — Branch management
- `/diff` — File diffing

**Configuration:**
- `/config` — Settings management
- `/mcp` — MCP server configuration
- `/skills` — Skill management
- `/plugin` — Plugin management
- `/theme` — UI theme selection
- `/keybindings` — Keyboard shortcut customization

**Session:**
- `/compact` — Compact conversation history
- `/resume` — Resume a previous session
- `/session` — Session management
- `/memory` — Persistent memory operations
- `/rewind` — Undo to a previous state

**Information:**
- `/help` — Command help
- `/status` — Session status
- `/cost` — Token cost breakdown
- `/usage` — API usage statistics
- `/doctor` — Diagnostic checks
- `/context` — Context window analysis
- `/insights` — Session analytics report

**Navigation:**
- `/teleport` — Remote development across machines
- `/desktop` — Desktop app handoff
- `/mobile` — Mobile integration
- `/chrome` — Chrome extension

Commands can be of type `local` (execute immediately), `prompt` (inject a prompt into the conversation), or `local-jsx` (render a React component).

### State Management

Application state is managed through a centralized store:

- **`AppState.tsx`** — React context provider that wraps the app
- **`AppStateStore.ts`** — Store shape definition
- **`store.ts`** — Zustand-based store with subscriptions
- **`selectors.ts`** — Derived state selectors
- **`bootstrap/state.ts`** — Bootstrap-time state (session ID, model, CWD, etc.)

State flows through React context to components and hooks, enabling reactive UI updates when messages arrive, tools execute, or settings change.

### Terminal UI (Ink)

The terminal UI is built on a **custom fork of Ink** (`ink/ink.tsx`, 251KB), a React-based terminal rendering library. This enables:

- Declarative UI composition with React components
- Layout with flexbox-like `<Box>` and styled `<Text>`
- Interactive elements (inputs, selects, buttons)
- Streaming output with live updates
- Diff visualization with syntax highlighting
- Progress spinners and status bars
- Dialog overlays for confirmations and settings

The `components/` directory contains 146 React components covering everything from the main `App.tsx` shell to specialized displays like `CompactSummary`, `ContextVisualization`, `DiagnosticsDisplay`, and diff renderers.

### Services Layer

Services encapsulate external integrations and complex subsystems:

| Service | Directory | Purpose |
|---|---|---|
| **API** | `services/api/` | Claude API client, streaming, retry logic, request logging |
| **MCP** | `services/mcp/` | MCP client connections, auth, config, tool/resource proxy |
| **Analytics** | `services/analytics/` | Telemetry events, GrowthBook feature flag evaluation |
| **Compact** | `services/compact/` | Conversation history compaction strategies |
| **LSP** | `services/lsp/` | Language Server Protocol for code intelligence |
| **OAuth** | `services/oauth/` | OAuth 2.0 authorization flows |
| **Plugins** | `services/plugins/` | Plugin discovery, loading, lifecycle |
| **Policy** | `services/policyLimits/` | Enterprise policy limit enforcement |
| **Remote Settings** | `services/remoteManagedSettings/` | Centrally-managed configuration |
| **Voice** | `services/voice.ts` | Speech-to-text and text-to-speech |

### Plugin & Skill System

**Skills** are reusable prompt templates that can be invoked via `/skill-name` or the `SkillTool`:
- Bundled skills ship with the CLI (`skills/bundled/`)
- Custom skills can be added to a `.claude/skills/` directory
- Skills from MCP servers are also supported

**Plugins** extend Claude Code with additional commands, tools, and skills:
- Built-in plugins (`plugins/bundled/`)
- Third-party plugins loaded from configured directories
- Plugin commands appear as slash commands
- Plugins can provide MCP server configurations

### Model Context Protocol (MCP)

MCP support (`services/mcp/`) enables Claude Code to connect to external tool servers:

- **Client** (`client.ts`) — connects to MCP servers, discovers tools/resources
- **Config** (`config.ts`) — server configuration from settings files
- **Auth** (`auth.ts`) — OAuth-based MCP server authentication
- **Tool proxy** — MCP tools appear as native tools in the tool pool
- **Resource access** — MCP resources available via `ListMcpResources` / `ReadMcpResource`

---

## Key Modules

### Token & Cost Tracking (`cost-tracker.ts`)

Tracks token usage (input, output, cache reads/writes) and cost across all API calls in a session. Supports per-model breakdowns and cumulative totals.

### Context Construction (`context.ts`)

Builds the system and user context injected into every API call:
- Git status (branch, recent commits, working tree state)
- Current date and platform info
- CLAUDE.md instructions (project-level, user-level, directory-level)
- Persistent memory files

### File History (`utils/fileHistory.ts`)

Takes snapshots of files before tool modifications, enabling `/rewind` to undo changes to any prior state.

### Session Management (`history.ts`, `utils/sessionStorage.ts`)

Persists conversation history to disk for `/resume` across CLI restarts. Sessions are stored with metadata for browsing and selection.

### Compaction (`services/compact/`)

When the conversation approaches the context window limit, the compaction system summarizes older messages to free space while preserving critical context.

---

## Feature Flags

The build system uses `bun:bundle`'s `feature()` for compile-time dead code elimination. Key flags:

| Flag | Purpose |
|---|---|
| `COORDINATOR_MODE` | Multi-agent coordination |
| `KAIROS` | Assistant/proactive mode |
| `BRIDGE_MODE` | IDE bridge communication |
| `DAEMON` | Background daemon process |
| `PROACTIVE` | Proactive suggestions |
| `VOICE_MODE` | Voice input/output |
| `AGENT_TRIGGERS` | Scheduled cron agents |
| `AGENT_TRIGGERS_REMOTE` | Remote agent triggers |
| `WORKFLOW_SCRIPTS` | Workflow automation |
| `HISTORY_SNIP` | History snipping |
| `ULTRAPLAN` | Advanced planning mode |
| `BUDDY` | Buddy/pair programming mode |
| `FORK_SUBAGENT` | Forked sub-agent processes |
| `UDS_INBOX` | Unix domain socket peer communication |
| `MONITOR_TOOL` | System monitoring |
| `EXPERIMENTAL_SKILL_SEARCH` | Skill discovery/search |

Runtime feature flags are additionally managed via **GrowthBook** for gradual rollouts and A/B testing.

---

## Configuration

Configuration is loaded from multiple sources, in order of precedence:

1. **CLI arguments** — command-line flags and options
2. **Environment variables** — `ANTHROPIC_API_KEY`, `CLAUDE_*` vars, etc.
3. **User settings** — `~/.claude/settings.json`
4. **Project settings** — `.claude/settings.json` in the working directory
5. **CLAUDE.md files** — project, user, and directory-level instructions
6. **MDM policies** — Enterprise Mobile Device Management (macOS plist, Windows registry)
7. **Remote managed settings** — Centrally-pushed configuration

Key configuration areas:
- **API keys** — stored in macOS Keychain or environment
- **Permission mode** — `default`, `plan`, `bypass`, with per-tool overrides
- **MCP servers** — server definitions, auth tokens
- **Keybindings** — customizable keyboard shortcuts (`~/.claude/keybindings.json`)
- **Themes** — terminal color schemes
- **Hooks** — shell commands triggered by tool events

---

## Security & Permissions

Claude Code implements a layered permission system:

- **Permission modes** — `default` (ask for dangerous ops), `plan` (read-only tools only), `bypass` (skip checks with warning)
- **Per-tool permissions** — each tool declares whether it needs permission; rules can allow/deny specific tools or patterns
- **Filesystem sandboxing** — restricts file operations to the working directory and configured paths
- **Tool approval** — interactive prompts for operations like file writes, shell commands, and network access
- **Secure storage** — API keys and OAuth tokens stored in the OS keychain
- **Policy limits** — enterprise-managed constraints on tool use and API access
- **Denial tracking** — tracks user denials to avoid repeating blocked actions

---

## Notable Patterns

### Startup Performance
The entry point (`main.tsx`) is heavily optimized for startup latency:
- Profiling checkpoints track every phase
- MDM reads, keychain access, and MCP prefetch run in parallel before imports complete
- Lazy `require()` breaks circular dependencies and defers heavy modules
- Feature-flag-gated `require()` enables dead code elimination at build time

### Parallel Prefetching
Multiple expensive operations are kicked off concurrently at startup:
```
startMdmRawRead()         // MDM policy reads (plutil/reg query)
startKeychainPrefetch()   // OAuth + API key from keychain
prefetchOfficialMcpUrls() // MCP server registry
prefetchFastModeStatus()  // Fast mode eligibility
```

### Compile-Time Feature Gating
```typescript
const SleepTool = feature('PROACTIVE') ? require('./tools/SleepTool').SleepTool : null
```
Bun's bundler eliminates the `require()` entirely when the flag is off, producing zero runtime cost.

### React in the Terminal
The entire UI is React components rendered to the terminal via a custom Ink engine, enabling:
- Component composition and reuse
- Hook-based state and effects
- Declarative layout
- Streaming updates without flickering

### Multi-Agent Architecture
The `AgentTool` spawns sub-agents that run as independent query loops, each with their own tool pool, context, and token budget. Agents can communicate via `SendMessage` and coordinate through the `TeamCreate`/`TeamDelete` system.
