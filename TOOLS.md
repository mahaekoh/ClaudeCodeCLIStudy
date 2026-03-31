# Claude Code — Tools Reference

A comprehensive reference for every tool available in Claude Code. Tools are the primary mechanism through which Claude interacts with the filesystem, shell, web, and external services.

---

## Table of Contents

- [Overview](#overview)
- [Tool Interface](#tool-interface)
- [File Operations](#file-operations)
  - [FileRead](#fileread)
  - [FileEdit](#fileedit)
  - [FileWrite](#filewrite)
  - [NotebookEdit](#notebookedit)
  - [Glob](#glob)
  - [Grep](#grep)
- [Execution](#execution)
  - [Bash](#bash)
  - [PowerShell](#powershell)
- [Web](#web)
  - [WebFetch](#webfetch)
  - [WebSearch](#websearch)
- [Agent & Multi-Agent](#agent--multi-agent)
  - [Agent](#agent)
  - [SendMessage](#sendmessage)
  - [TeamCreate](#teamcreate)
  - [TeamDelete](#teamdelete)
- [Task Management](#task-management)
  - [TaskCreate](#taskcreate)
  - [TaskGet](#taskget)
  - [TaskList](#tasklist)
  - [TaskUpdate](#taskupdate)
  - [TaskStop](#taskstop)
  - [TaskOutput](#taskoutput)
  - [TodoWrite (Legacy)](#todowrite-legacy)
- [Planning & Workspace](#planning--workspace)
  - [EnterPlanMode](#enterplanmode)
  - [ExitPlanMode](#exitplanmode)
  - [EnterWorktree](#enterworktree)
  - [ExitWorktree](#exitworktree)
- [MCP (Model Context Protocol)](#mcp-model-context-protocol)
  - [MCPTool](#mcptool)
  - [McpAuth](#mcpauth)
  - [ListMcpResources](#listmcpresources)
  - [ReadMcpResource](#readmcpresource)
- [Code Intelligence](#code-intelligence)
  - [LSP](#lsp)
- [Scheduling](#scheduling)
  - [CronCreate](#croncreate)
  - [CronDelete](#crondelete)
  - [CronList](#cronlist)
  - [RemoteTrigger](#remotetrigger)
- [User Interaction](#user-interaction)
  - [AskUserQuestion](#askuserquestion)
  - [SendUserMessage (Brief)](#sendusermessage-brief)
- [Utility](#utility)
  - [ToolSearch](#toolsearch)
  - [SkillTool](#skilltool)
  - [Config](#config)
  - [StructuredOutput](#structuredoutput)

---

## Overview

Claude Code ships with **38+ built-in tools** organized into categories. Each tool:

- Has a defined **input schema** (validated with Zod)
- Declares whether it is **read-only** or mutating
- Specifies whether it **requires user permission** before execution
- Can be **enabled/disabled** based on feature flags, user type, or permission mode
- Produces structured **results** returned to the model

Tools are assembled into an active pool at runtime by the tool registry (`tools.ts`). The pool varies based on:

| Factor | Effect |
|---|---|
| **Feature flags** | Compile-time dead code elimination via `bun:bundle` |
| **User type** | Internal (`ant`) vs external users |
| **Permission mode** | Plan mode restricts to read-only tools |
| **MCP servers** | Connected servers contribute additional tools |
| **Plugins** | Plugins can register custom tools |

---

## Tool Interface

Every tool implements the following interface (defined in `Tool.ts`):

```typescript
interface Tool {
  name: string                              // Unique identifier
  description: string                       // Shown to the model
  inputSchema: ToolInputJSONSchema          // JSON Schema for parameters
  isEnabled(context): boolean               // Whether this tool is available
  isReadOnly(): boolean                     // Safe for plan mode?
  needsPermissions(input): boolean          // Requires user approval?
  prompt(context): string                   // Dynamic instructions for the model
  call(input, context): Promise<ToolResult> // Execution logic
}
```

---

## File Operations

### FileRead

Read files from the local filesystem with support for text, images, PDFs, and Jupyter notebooks.

| Property | Value |
|---|---|
| **Tool Name** | `Read` |
| **Read-Only** | Yes |
| **Permissions** | No |

**Parameters:**

| Parameter | Type | Required | Description |
|---|---|---|---|
| `file_path` | `string` | Yes | Absolute path to the file |
| `offset` | `number` | No | Starting line number (0-based) |
| `limit` | `number` | No | Number of lines to read (default: 2000) |
| `pages` | `string` | No | Page range for PDFs (e.g., `"1-5"`, `"3"`, `"10-20"`) |

**Notable behavior:**
- Returns content with `cat -n` style line numbers
- Image files (PNG, JPG, etc.) are returned as visual content blocks
- PDF files support page ranges; max 20 pages per request; large PDFs require the `pages` parameter
- Jupyter notebooks (`.ipynb`) return all cells with outputs
- 1 GiB file size limit
- Blocks reads of device files (`/dev/*`)
- Detects and flags memory files from the `memdir` system

---

### FileEdit

Perform surgical string replacement edits within existing files.

| Property | Value |
|---|---|
| **Tool Name** | `Edit` |
| **Read-Only** | No |
| **Permissions** | Yes |

**Parameters:**

| Parameter | Type | Required | Description |
|---|---|---|---|
| `file_path` | `string` | Yes | Absolute path to the file |
| `old_string` | `string` | Yes | Exact text to find (must be unique in file) |
| `new_string` | `string` | Yes | Replacement text (must differ from `old_string`) |
| `replace_all` | `boolean` | No | Replace all occurrences (default: `false`) |

**Notable behavior:**
- Fails if `old_string` is not unique in the file (unless `replace_all` is set)
- Generates a git-style diff patch for review
- Tracks changes in file history for `/rewind` support
- 1 GiB file size limit
- Preserves file encoding and line endings

---

### FileWrite

Create new files or completely overwrite existing ones.

| Property | Value |
|---|---|
| **Tool Name** | `Write` |
| **Read-Only** | No |
| **Permissions** | Yes |

**Parameters:**

| Parameter | Type | Required | Description |
|---|---|---|---|
| `file_path` | `string` | Yes | Absolute path (must be absolute, not relative) |
| `content` | `string` | Yes | Full file content to write |

**Notable behavior:**
- Overwrites existing files entirely — prefer `FileEdit` for partial modifications
- Generates diff patches for existing files
- Tracks changes in file history
- Clears LSP diagnostics for the file

---

### NotebookEdit

Edit cells within Jupyter notebooks (`.ipynb` files).

| Property | Value |
|---|---|
| **Tool Name** | `NotebookEdit` |
| **Read-Only** | No |
| **Permissions** | Yes |

**Parameters:**

| Parameter | Type | Required | Description |
|---|---|---|---|
| `notebook_path` | `string` | Yes | Absolute path to the `.ipynb` file |
| `cell_id` | `string` | No | Target cell ID (for edit/insert-after) |
| `new_source` | `string` | Yes | New cell source content |
| `cell_type` | `"code"` \| `"markdown"` | No | Cell type (default: `"code"`) |
| `edit_mode` | `"replace"` \| `"insert"` \| `"delete"` | No | Operation mode (default: `"replace"`) |

**Notable behavior:**
- Parses notebook JSON to locate cells by ID
- Supports replace, insert-after, and delete operations
- Tracks file history for undo support
- Detects notebook language for syntax context

---

### Glob

Find files by name using glob patterns.

| Property | Value |
|---|---|
| **Tool Name** | `Glob` |
| **Read-Only** | Yes |
| **Permissions** | No |

**Parameters:**

| Parameter | Type | Required | Description |
|---|---|---|---|
| `pattern` | `string` | Yes | Glob pattern (e.g., `"**/*.ts"`, `"src/components/**/*.tsx"`) |
| `path` | `string` | No | Directory to search in (defaults to CWD) |

**Notable behavior:**
- Results capped at 100 files
- Returns file paths sorted by modification time (newest first)
- Validates that the search directory exists

---

### Grep

Search file contents using regular expressions, powered by an embedded ripgrep binary.

| Property | Value |
|---|---|
| **Tool Name** | `Grep` |
| **Read-Only** | Yes |
| **Permissions** | No |

**Parameters:**

| Parameter | Type | Required | Description |
|---|---|---|---|
| `pattern` | `string` | Yes | Regex pattern (ripgrep syntax) |
| `path` | `string` | No | File or directory to search (defaults to CWD) |
| `glob` | `string` | No | File filter pattern (e.g., `"*.js"`, `"*.{ts,tsx}"`) |
| `type` | `string` | No | File type filter (e.g., `"js"`, `"py"`, `"rust"`) |
| `output_mode` | `"content"` \| `"files_with_matches"` \| `"count"` | No | Output format (default: `"files_with_matches"`) |
| `-B` | `number` | No | Lines of context before each match |
| `-A` | `number` | No | Lines of context after each match |
| `-C` | `number` | No | Lines of context before and after each match |
| `-n` | `boolean` | No | Show line numbers (default: `true`) |
| `-i` | `boolean` | No | Case-insensitive search |
| `head_limit` | `number` | No | Max output lines/entries (default: 250; pass 0 for unlimited) |
| `offset` | `number` | No | Skip first N results before applying `head_limit` |
| `multiline` | `boolean` | No | Enable multiline mode where `.` matches newlines |

**Notable behavior:**
- Automatically excludes `.git/`, `.svn/`, and other VCS directories
- Respects file-read permission boundaries
- Pattern uses ripgrep syntax — literal braces need escaping (e.g., `interface\{\}`)
- `head_limit` defaults to 250 to prevent context window flooding

---

## Execution

### Bash

Execute shell commands with comprehensive security checks, sandboxing, and permission validation.

| Property | Value |
|---|---|
| **Tool Name** | `Bash` |
| **Read-Only** | Conditional (validated per command) |
| **Permissions** | Yes (for non-read-only commands) |

**Parameters:**

| Parameter | Type | Required | Description |
|---|---|---|---|
| `command` | `string` | Yes | The shell command to execute |
| `description` | `string` | No | Human-readable description of what the command does |
| `timeout` | `number` | No | Timeout in milliseconds (default varies; max 600,000) |
| `run_in_background` | `boolean` | No | Run as a background task |

**Notable behavior:**
- **Security analysis**: Commands are parsed through an AST-based security checker (`bashSecurity.ts`) that flags dangerous patterns
- **Read-only validation**: Determines if a command is read-only (e.g., `ls`, `git status`) or mutating (e.g., `rm`, `git push`) via `readOnlyValidation.ts`
- **Path validation**: Ensures file operations stay within allowed directories
- **Sandboxing**: Optionally runs commands in a sandbox via `SandboxManager`
- **Git operation tracking**: Detects and tracks git operations for attribution
- **File history**: Tracks file modifications for `/rewind` support
- **Background tasks**: Commands can run in the background with `run_in_background`; results retrieved via `TaskOutput`
- **sed interception**: Parses `sed` edit commands and suggests using `FileEdit` instead
- **Image output**: Detects and resizes image output from commands like `screenshot`
- **Auto-background**: In assistant mode, long-running blocking commands auto-background after 15 seconds

---

### PowerShell

Execute PowerShell commands on Windows systems.

| Property | Value |
|---|---|
| **Tool Name** | `PowerShell` |
| **Read-Only** | Conditional |
| **Permissions** | Yes |

**Parameters:** Similar to Bash but adapted for PowerShell syntax.

**Notable behavior:**
- Windows-only tool (not available on macOS/Linux)
- Applies PowerShell-specific security checks
- Supports background task execution
- Read-only validation adapted for PowerShell cmdlets

---

## Web

### WebFetch

Fetch content from a URL and extract it as markdown.

| Property | Value |
|---|---|
| **Tool Name** | `WebFetch` |
| **Read-Only** | Yes |
| **Permissions** | Yes (unless host is pre-approved) |

**Parameters:**

| Parameter | Type | Required | Description |
|---|---|---|---|
| `url` | `string` | Yes | Valid URL to fetch |
| `prompt` | `string` | Yes | Prompt to apply against the fetched content |

**Notable behavior:**
- Fetches the URL and converts HTML to markdown
- Applies the `prompt` to filter/extract relevant content
- Results capped at 100,000 characters
- Responses cached for 15 minutes to avoid redundant fetches
- Pre-approved hosts skip the permission check

---

### WebSearch

Search the web and return processed results.

| Property | Value |
|---|---|
| **Tool Name** | `WebSearch` |
| **Read-Only** | Yes |
| **Permissions** | Yes |

**Parameters:**

| Parameter | Type | Required | Description |
|---|---|---|---|
| `query` | `string` | Yes | Search query (minimum 2 characters) |
| `allowed_domains` | `string[]` | No | Only include results from these domains |
| `blocked_domains` | `string[]` | No | Exclude results from these domains |

**Notable behavior:**
- Uses Anthropic's `web_search_20250305` beta API
- Maximum 8 search queries per invocation
- Results are streamed and processed through Claude
- Domain filtering allows scoping results to trusted sources

---

## Agent & Multi-Agent

### Agent

Launch autonomous sub-agents that execute complex tasks with their own tool pool, context window, and token budget.

| Property | Value |
|---|---|
| **Tool Name** | `Agent` |
| **Read-Only** | No |
| **Permissions** | Yes |

**Parameters:**

| Parameter | Type | Required | Description |
|---|---|---|---|
| `prompt` | `string` | Yes | Task description for the agent |
| `description` | `string` | Yes | Short (3-5 word) summary of the task |
| `subagent_type` | `string` | No | Specialized agent type (e.g., `"Explore"`, `"Plan"`) |
| `model` | `string` | No | Model override (`"sonnet"`, `"opus"`, `"haiku"`) |
| `run_in_background` | `boolean` | No | Run the agent as a background task |
| `isolation` | `"worktree"` | No | Run in an isolated git worktree |

**Notable behavior:**
- Each agent runs as an independent query loop with its own conversation
- Agents can be run in the foreground (blocking) or background
- Background agents notify the parent when they complete
- Worktree isolation creates a temporary git branch for the agent's changes
- Agents inherit the parent's permission mode but have their own tool pool
- Supports specialized built-in agent types:
  - `"general-purpose"` — default, full tool access
  - `"Explore"` — optimized for codebase exploration (read-only tools)
  - `"Plan"` — architectural planning (read-only tools)
- Custom agent definitions can be loaded from `.claude/agents/` directories
- Inter-agent communication via `SendMessage`
- Auto-backgrounds after 120 seconds when enabled
- Agents can be denied by permission rules matching agent names

---

### SendMessage

Send messages between agents, teammates, and peers in multi-agent configurations.

| Property | Value |
|---|---|
| **Tool Name** | `SendMessage` |
| **Read-Only** | No |
| **Permissions** | No |

**Parameters:**

| Parameter | Type | Required | Description |
|---|---|---|---|
| `to` | `string` | Yes | Recipient: teammate name, `"*"` for broadcast, or `"uds:<socket>"` for peers |
| `summary` | `string` | No | 5-10 word message preview |
| `message` | `string` \| `object` | Yes | Plain text or structured message (`shutdown_request`, `shutdown_response`, `plan_approval_response`) |

**Notable behavior:**
- Routes messages through the agent swarm's mailbox system
- Supports structured message types for coordination protocols
- Broadcasting (`"*"`) sends to all active teammates
- UDS (Unix Domain Socket) messaging for inter-process peer communication

---

### TeamCreate

Create a multi-agent swarm team for coordinated parallel work.

| Property | Value |
|---|---|
| **Tool Name** | `TeamCreate` |
| **Read-Only** | No |
| **Permissions** | Yes |

**Parameters:**

| Parameter | Type | Required | Description |
|---|---|---|---|
| `team_name` | `string` | Yes | Name for the new team |
| `description` | `string` | No | Team purpose description |
| `agent_type` | `string` | No | Type/role of the team lead |

**Notable behavior:**
- Requires agent swarms to be enabled
- Generates unique team names to avoid collisions
- Sets up a shared task list and assigns teammate colors
- The creating agent becomes the team lead

---

### TeamDelete

Disband a swarm team and clean up all resources.

| Property | Value |
|---|---|
| **Tool Name** | `TeamDelete` |
| **Read-Only** | No |
| **Permissions** | Yes |

**Parameters:** None.

**Notable behavior:**
- Verifies all team members are inactive before cleanup
- Only available when agent swarms are enabled
- Cleans up team state, task lists, and communication channels

---

## Task Management

### TaskCreate

Create a tracked task in the session task list.

| Property | Value |
|---|---|
| **Tool Name** | `TaskCreate` |
| **Read-Only** | No |
| **Permissions** | No |

**Parameters:**

| Parameter | Type | Required | Description |
|---|---|---|---|
| `subject` | `string` | Yes | Brief task title |
| `description` | `string` | Yes | What needs to be done |
| `activeForm` | `string` | No | Present-continuous verb for spinner display (e.g., `"analyzing"`) |
| `metadata` | `object` | No | Arbitrary key-value metadata |

**Notable behavior:**
- Only enabled when the TaskV2 system is active
- Executes task creation hooks if configured
- Tasks appear in the session's progress tracking UI

---

### TaskGet

Retrieve details of a specific task by ID.

| Property | Value |
|---|---|
| **Tool Name** | `TaskGet` |
| **Read-Only** | Yes |
| **Permissions** | No |

**Parameters:**

| Parameter | Type | Required | Description |
|---|---|---|---|
| `taskId` | `string` | Yes | Task ID to retrieve |

**Returns:** Task object with `subject`, `description`, `status`, `blocks`, `blockedBy`, and `metadata`.

---

### TaskList

List all tasks in the current session.

| Property | Value |
|---|---|
| **Tool Name** | `TaskList` |
| **Read-Only** | Yes |
| **Permissions** | No |

**Parameters:** None.

**Notable behavior:**
- Filters out internal metadata tasks
- Resolves task dependency chains (removes completed tasks from `blockedBy`)

---

### TaskUpdate

Update an existing task's status, description, or dependencies.

| Property | Value |
|---|---|
| **Tool Name** | `TaskUpdate` |
| **Read-Only** | No |
| **Permissions** | No |

**Parameters:**

| Parameter | Type | Required | Description |
|---|---|---|---|
| `taskId` | `string` | Yes | Task ID to update |
| `subject` | `string` | No | Updated title |
| `description` | `string` | No | Updated description |
| `activeForm` | `string` | No | Updated spinner verb |
| `status` | `string` | No | New status (`"in_progress"`, `"completed"`, `"deleted"`, etc.) |
| `addBlocks` | `string[]` | No | Task IDs this task now blocks |
| `addBlockedBy` | `string[]` | No | Task IDs that block this task |
| `owner` | `string` | No | Task owner/assignee |
| `metadata` | `object` | No | Metadata updates (set `null` to delete a key) |

**Notable behavior:**
- `"deleted"` status permanently removes the task
- Executes task completion hooks when status becomes `"completed"`
- Supports dependency tracking with `addBlocks`/`addBlockedBy`

---

### TaskStop

Terminate a running background task.

| Property | Value |
|---|---|
| **Tool Name** | `TaskStop` |
| **Read-Only** | No |
| **Permissions** | No |

**Parameters:**

| Parameter | Type | Required | Description |
|---|---|---|---|
| `task_id` | `string` | No | Task ID to stop |

**Notable behavior:**
- Validates the task exists and is currently running
- Backward compatible with the deprecated `shell_id` parameter

---

### TaskOutput

Retrieve output from a running or completed background task (bash, agent, or remote).

| Property | Value |
|---|---|
| **Tool Name** | `TaskOutput` |
| **Read-Only** | Yes |
| **Permissions** | No |

**Parameters:**

| Parameter | Type | Required | Description |
|---|---|---|---|
| `task_id` | `string` | Yes | Task ID to get output from |
| `block` | `boolean` | No | Wait for task completion (default: `true`) |
| `timeout` | `number` | No | Max wait time in ms (default: 30,000) |

**Notable behavior:**
- Supports three task types: local bash, local agent, and remote agent
- For agent tasks, extracts clean results from the agent's conversation memory
- Non-blocking mode returns partial output immediately

---

### TodoWrite (Legacy)

Legacy task management tool — replaced by `TaskCreate`/`TaskUpdate` in newer versions.

| Property | Value |
|---|---|
| **Tool Name** | `TodoWrite` |
| **Read-Only** | No |
| **Permissions** | No |

**Parameters:**

| Parameter | Type | Required | Description |
|---|---|---|---|
| `todos` | `array` | Yes | Complete updated todo list |

**Notable behavior:**
- Only enabled when the TaskV2 system is **not** active
- Replaces the entire task list on each call
- Deprecated in favor of the granular Task* tools

---

## Planning & Workspace

### EnterPlanMode

Switch the session to plan mode, restricting tools to read-only operations for safe exploration and design.

| Property | Value |
|---|---|
| **Tool Name** | `EnterPlanMode` |
| **Read-Only** | Yes |
| **Permissions** | No |

**Parameters:** None.

**Notable behavior:**
- Disables all mutating tools (file writes, bash commands that modify state, etc.)
- Useful for complex tasks that require research before implementation
- Disabled when `--channels` flag is active
- Triggers permission classifier activation for `"auto"` permission mode

---

### ExitPlanMode

Present a plan for user approval and transition back to implementation mode.

| Property | Value |
|---|---|
| **Tool Name** | `ExitPlanMode` |
| **Read-Only** | No |
| **Permissions** | No |

**Parameters:**

| Parameter | Type | Required | Description |
|---|---|---|---|
| `allowedPrompts` | `string[]` | No | Permission prompts needed for the plan |

**Notable behavior:**
- Only available when currently in plan mode
- Presents the plan for user review and approval
- Supports plan editing before approval
- In swarm mode, notifies teammates of plan approval
- Integration with verification agents for plan validation

---

### EnterWorktree

Create an isolated git worktree and switch the session into it.

| Property | Value |
|---|---|
| **Tool Name** | `EnterWorktree` |
| **Read-Only** | No |
| **Permissions** | Yes |

**Parameters:**

| Parameter | Type | Required | Description |
|---|---|---|---|
| `name` | `string` | No | Worktree name (alphanumeric, dots, underscores, dashes; max 64 chars) |

**Notable behavior:**
- Creates a new git worktree branched from the current state
- Provides full filesystem isolation — changes don't affect the main tree
- Clears cached system prompt sections and file state for the new context
- Worktree can be created via git or configured hooks

---

### ExitWorktree

Leave the current worktree and return to the original working directory.

| Property | Value |
|---|---|
| **Tool Name** | `ExitWorktree` |
| **Read-Only** | No |
| **Permissions** | Yes |

**Parameters:**

| Parameter | Type | Required | Description |
|---|---|---|---|
| `action` | `"keep"` \| `"remove"` | Yes | Keep the worktree branch or remove it |
| `discard_changes` | `boolean` | No | Required when `action="remove"` and worktree has uncommitted changes |

**Notable behavior:**
- Counts uncommitted changes before removal to prevent accidental data loss
- Restores the session state to the original directory
- Kills any tmux sessions associated with the worktree
- Cleans up file state caches

---

## MCP (Model Context Protocol)

### MCPTool

Proxy tool that bridges to tools provided by connected MCP servers.

| Property | Value |
|---|---|
| **Tool Name** | `mcp__<server>__<tool>` (dynamic) |
| **Read-Only** | Depends on the specific MCP tool |
| **Permissions** | Yes |

**Parameters:** Passthrough — defined by the MCP server's tool schema.

**Notable behavior:**
- Not a single tool but a template; one instance is created per MCP server tool
- Tool name follows the pattern `mcp__<server_name>__<tool_name>`
- Input schema is forwarded from the MCP server's tool definition
- Supports progress tracking during execution
- Overridden at runtime by `mcpClient.ts` with server-specific implementations

---

### McpAuth

Start an OAuth authentication flow for MCP servers that require it.

| Property | Value |
|---|---|
| **Tool Name** | `mcp__<server>__authenticate` (dynamic) |
| **Read-Only** | No |
| **Permissions** | Yes |

**Parameters:** None (empty object).

**Notable behavior:**
- Created dynamically for each MCP server that needs authentication
- Initiates the OAuth flow without opening a browser (`skipBrowserOpen`)
- Automatically replaced by real server tools once authentication completes
- Temporary tool that exists only until the server is fully connected

---

### ListMcpResources

List resources available from connected MCP servers.

| Property | Value |
|---|---|
| **Tool Name** | `ListMcpResources` |
| **Read-Only** | Yes |
| **Permissions** | No |

**Parameters:**

| Parameter | Type | Required | Description |
|---|---|---|---|
| `server` | `string` | No | Filter by specific server name |

**Notable behavior:**
- LRU-cached per server for performance
- Auto-refreshes when a server reconnects
- Returns resource URIs, names, and descriptions

---

### ReadMcpResource

Read a specific resource from an MCP server by URI.

| Property | Value |
|---|---|
| **Tool Name** | `ReadMcpResource` |
| **Read-Only** | Yes |
| **Permissions** | No |

**Parameters:**

| Parameter | Type | Required | Description |
|---|---|---|---|
| `server` | `string` | Yes | MCP server name |
| `uri` | `string` | Yes | Resource URI |

**Notable behavior:**
- Validates that the server is connected and supports resource reading
- Persists binary content to disk and returns a reference
- Text content is returned inline

---

## Code Intelligence

### LSP

Interact with Language Server Protocol servers for code intelligence features.

| Property | Value |
|---|---|
| **Tool Name** | `LSP` |
| **Read-Only** | Yes |
| **Permissions** | No |

**Parameters:**

| Parameter | Type | Required | Description |
|---|---|---|---|
| `operation` | `enum` | Yes | One of: `"goToDefinition"`, `"findReferences"`, `"hover"`, `"documentSymbol"`, `"workspaceSymbol"`, `"goToImplementation"`, `"prepareCallHierarchy"`, `"incomingCalls"`, `"outgoingCalls"` |
| `filePath` | `string` | Yes | Absolute path to the file |
| `line` | `number` | Yes | 1-based line number |
| `character` | `number` | Yes | 1-based character offset |

**Notable behavior:**
- Requires an LSP server to be running and initialized
- 10 MB file size limit
- Waits for LSP initialization before executing
- Results are formatted per operation type (e.g., location lists for definitions, hover documentation for hover)
- Supports the full call hierarchy protocol (prepare, incoming, outgoing)

---

## Scheduling

### CronCreate

Schedule a recurring or one-shot prompt to execute on a cron schedule.

| Property | Value |
|---|---|
| **Tool Name** | `CronCreate` |
| **Read-Only** | No |
| **Permissions** | Yes |
| **Feature Flag** | `AGENT_TRIGGERS` |

**Parameters:**

| Parameter | Type | Required | Description |
|---|---|---|---|
| `cron` | `string` | Yes | 5-field cron expression (e.g., `"*/5 * * * *"`) |
| `prompt` | `string` | Yes | Prompt to execute on each trigger |
| `recurring` | `boolean` | No | Repeat on schedule (default: `true`) |
| `durable` | `boolean` | No | Persist to disk across restarts (default: `false`) |

**Notable behavior:**
- Maximum 50 active cron jobs
- Validates cron expression syntax and calendar feasibility
- Non-recurring jobs auto-delete after first execution
- Jobs auto-expire after a configured maximum age

---

### CronDelete

Cancel and remove a scheduled cron job.

| Property | Value |
|---|---|
| **Tool Name** | `CronDelete` |
| **Read-Only** | No |
| **Permissions** | Yes |
| **Feature Flag** | `AGENT_TRIGGERS` |

**Parameters:**

| Parameter | Type | Required | Description |
|---|---|---|---|
| `id` | `string` | Yes | Job ID (returned by `CronCreate`) |

---

### CronList

List all active cron jobs in the current session.

| Property | Value |
|---|---|
| **Tool Name** | `CronList` |
| **Read-Only** | Yes |
| **Permissions** | No |
| **Feature Flag** | `AGENT_TRIGGERS` |

**Parameters:** None.

**Returns:** Array of jobs with `id`, `cron`, `humanSchedule`, `prompt`, `recurring`, and `durable` fields.

---

### RemoteTrigger

Manage scheduled remote agent triggers that run on Anthropic's infrastructure.

| Property | Value |
|---|---|
| **Tool Name** | `RemoteTrigger` |
| **Read-Only** | Conditional (`list`/`get` are read-only) |
| **Permissions** | Yes |
| **Feature Flag** | `AGENT_TRIGGERS_REMOTE` |

**Parameters:**

| Parameter | Type | Required | Description |
|---|---|---|---|
| `action` | `"list"` \| `"get"` \| `"create"` \| `"update"` \| `"run"` | Yes | Operation to perform |
| `trigger_id` | `string` | Conditional | Required for `get`, `update`, `run` |
| `body` | `object` | No | JSON body for `create`/`update` operations |

**Notable behavior:**
- Requires OAuth authentication
- Uses the organization's UUID for API scoping
- Communicates with the CCR triggers API

---

## User Interaction

### AskUserQuestion

Present the user with structured multiple-choice or multi-select questions.

| Property | Value |
|---|---|
| **Tool Name** | `AskUserQuestion` |
| **Read-Only** | Yes |
| **Permissions** | No |

**Parameters:**

| Parameter | Type | Required | Description |
|---|---|---|---|
| `questions` | `array` | Yes | 1-4 questions, each with 2-4 options |
| `multiSelect` | `boolean` | No | Allow selecting multiple options |
| `metadata` | `object` | No | Tracking metadata |

Each question has:
- `question` — the question text
- `options` — array of `{ label, description?, preview? }` objects

**Notable behavior:**
- Supports markdown in option descriptions
- Options can include preview content (shown when focused)
- Users can add free-text notes alongside their selections
- Designed for structured decision-making interactions

---

### SendUserMessage (Brief)

Send a visible message to the user with optional file attachments. Primary output channel in assistant mode.

| Property | Value |
|---|---|
| **Tool Name** | `SendUserMessage` |
| **Read-Only** | Yes |
| **Permissions** | No |
| **Feature Flag** | `KAIROS` or `KAIROS_BRIEF` |

**Parameters:**

| Parameter | Type | Required | Description |
|---|---|---|---|
| `message` | `string` | Yes | Message content (supports markdown) |
| `attachments` | `string[]` | No | File paths to attach |
| `status` | `"normal"` \| `"proactive"` | No | Message type for analytics |

**Notable behavior:**
- Feature-gated for assistant/proactive modes
- Resolves attachment paths and validates file existence
- Tracks proactive vs. reactive messaging for analytics

---

## Utility

### ToolSearch

Search for and fetch schemas of deferred tools that haven't been loaded yet.

| Property | Value |
|---|---|
| **Tool Name** | `ToolSearch` |
| **Read-Only** | Yes |
| **Permissions** | No |

**Parameters:**

| Parameter | Type | Required | Description |
|---|---|---|---|
| `query` | `string` | Yes | Search query or `"select:<tool_name>"` for direct lookup |
| `max_results` | `number` | No | Maximum results to return (default: 5) |

**Notable behavior:**
- Searches by keyword or exact tool name selection
- Returns full JSON Schema definitions for matched tools, making them callable
- Memoized description fetching for performance
- Cache invalidates when the set of deferred tools changes
- Essential for tools that are deferred at startup to reduce initial load

---

### SkillTool

Invoke a registered skill (slash command or prompt template).

| Property | Value |
|---|---|
| **Tool Name** | `Skill` |
| **Read-Only** | Depends on the skill |
| **Permissions** | Depends on the skill |

**Parameters:** Dynamic — varies per skill definition. Typically includes:

| Parameter | Type | Required | Description |
|---|---|---|---|
| `skill` | `string` | Yes | Skill name (e.g., `"commit"`, `"review-pr"`) |
| `args` | `string` | No | Arguments for the skill |

**Notable behavior:**
- Integrates with bundled skills, custom skills, MCP skills, and plugin skills
- Can fork an agent for skill execution
- Manages command context and skill lifecycle
- Tracks skill usage for analytics

---

### Config

Read or modify Claude Code settings at runtime.

| Property | Value |
|---|---|
| **Tool Name** | `Config` |
| **Read-Only** | Conditional (read-only for GET, mutating for SET) |
| **Permissions** | No |

**Parameters:**

| Parameter | Type | Required | Description |
|---|---|---|---|
| `setting` | `string` | Yes | Setting key to read or write |
| `value` | `any` | No | New value (omit for GET operation) |

**Notable behavior:**
- Supports both global and project-scoped settings
- Validates setting values (e.g., model names, voice mode availability)
- Syncs changes to the application state store
- Handles async validation for certain settings (e.g., model verification)

---

### StructuredOutput

Return the final response as validated structured JSON. Used for programmatic/SDK integrations.

| Property | Value |
|---|---|
| **Tool Name** | `StructuredOutput` |
| **Read-Only** | Yes |
| **Permissions** | No |

**Parameters:** Any JSON object — validated against a dynamically-provided schema.

**Notable behavior:**
- Only available in non-interactive (SDK/headless) sessions
- The output schema is provided at session creation time
- Must be called exactly once per session
- Validates the output against the provided JSON Schema before returning
- Enables type-safe programmatic responses from Claude Code

---

## Tool Availability Summary

### Always Available (Core Tools)

| Tool | Read-Only | Needs Permission |
|---|---|---|
| `Read` (FileRead) | Yes | No |
| `Edit` (FileEdit) | No | Yes |
| `Write` (FileWrite) | No | Yes |
| `Glob` | Yes | No |
| `Grep` | Yes | No |
| `Bash` | Conditional | Conditional |
| `Agent` | No | Yes |
| `WebFetch` | Yes | Yes |
| `WebSearch` | Yes | Yes |
| `LSP` | Yes | No |
| `NotebookEdit` | No | Yes |
| `ToolSearch` | Yes | No |
| `Skill` | Varies | Varies |
| `EnterPlanMode` | Yes | No |
| `ExitPlanMode` | No | No |
| `EnterWorktree` | No | Yes |
| `ExitWorktree` | No | Yes |
| `AskUserQuestion` | Yes | No |
| `TaskCreate` | No | No |
| `TaskGet` | Yes | No |
| `TaskList` | Yes | No |
| `TaskUpdate` | No | No |
| `TaskStop` | No | No |
| `TaskOutput` | Yes | No |
| `SendMessage` | No | No |

### Feature-Gated Tools

| Tool | Feature Flag | Read-Only |
|---|---|---|
| `CronCreate` | `AGENT_TRIGGERS` | No |
| `CronDelete` | `AGENT_TRIGGERS` | No |
| `CronList` | `AGENT_TRIGGERS` | Yes |
| `RemoteTrigger` | `AGENT_TRIGGERS_REMOTE` | Conditional |
| `SendUserMessage` | `KAIROS` / `KAIROS_BRIEF` | Yes |
| `PowerShell` | Windows only | Conditional |
| `TeamCreate` | Agent Swarms enabled | No |
| `TeamDelete` | Agent Swarms enabled | No |

### Dynamic Tools

| Tool | Source |
|---|---|
| `mcp__<server>__<tool>` | MCP server connections |
| `mcp__<server>__authenticate` | MCP servers needing auth |
| `ListMcpResources` | Any MCP server with resources |
| `ReadMcpResource` | Any MCP server with resources |
| `StructuredOutput` | SDK/headless sessions with output schema |
