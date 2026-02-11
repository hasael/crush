
# Crush: Architecture Overview

Crush is a CLI-based AI coding assistant built with Go and Bubbletea. This document explains how the application works, tracing through a complete user request from start to finish.

## Table of Contents

1. [High-Level Architecture](#high-level-architecture)
2. [Main Components](#main-components)
3. [How the CLI Works](#how-the-cli-works)
4. [Complete Example Flow](#complete-example-flow)
5. [Tool System Details](#tool-system-details)

---

## High-Level Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                         User Input                          │
│                    "Refactor this project..."               │
└───────────────────────────┬─────────────────────────────────┘
                            │
                            ▼
┌─────────────────────────────────────────────────────────────┐
│                      CLI Layer (cmd/)                       │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────────────┐ │
│  │    main.go  │  │   root.go   │  │     run.go          │ │
│  └─────────────┘  └─────────────┘  └─────────────────────┘ │
└───────────────────────────┬─────────────────────────────────┘
                            │
                            ▼
┌─────────────────────────────────────────────────────────────┐
│                   Application Layer (app/)                  │
│  ┌──────────────────────────────────────────────────────┐   │
│  │  App orchestrates:                                   │   │
│  │  - Session & Message services (database)             │   │
│  │  - Permission service (user approvals)               │   │
│  │  - LSP Manager (code intelligence)                   │   │
│  │  - Agent Coordinator (AI orchestration)              │   │
│  └──────────────────────────────────────────────────────┘   │
└───────────────────────────┬─────────────────────────────────┘
                            │
                            ▼
┌─────────────────────────────────────────────────────────────┐
│              Agent Layer (internal/agent/)                  │
│  ┌──────────────────────────────────────────────────────┐   │
│  │  Coordinator                                          │   │
│  │  ├── buildAgent() - Creates agent with tools         │   │
│  │  └── Run() - Executes agent with prompt              │   │
│  └──────────────────────────────────────────────────────┘   │
│                            │                                 │
│                            ▼                                 │
│  ┌──────────────────────────────────────────────────────┐   │
│  │  SessionAgent (agent.go)                             │   │
│  │  ├── Manages conversation history                    │   │
│  │  ├── Handles tool calls                              │   │
│  │  ├── Coordinates with LSP                            │   │
│  │  └── Streams responses back to user                  │   │
│  └──────────────────────────────────────────────────────┘   │
└───────────────────────────┬─────────────────────────────────┘
                            │
                            ▼
┌─────────────────────────────────────────────────────────────┐
│                  Tool System (agent/tools/)                 │
│  ┌──────┐ ┌──────┐ ┌──────┐ ┌──────┐ ┌────────┐           │
│  │ bash │ │ edit │ │ view │ │ grep │ │  MCP   │  ...     │ │
│  └──────┘ └──────┘ └──────┘ └──────┘ └────────┘           │
└───────────────────────────┬─────────────────────────────────┘
                            │
                            ▼
┌─────────────────────────────────────────────────────────────┐
│                   External Services                         │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐   │
│  │   LLM    │  │   LSP    │  │    MCP   │  │  Shell   │   │
│  │ Provider │  │ Servers  │  │ Servers  │  │ Commands │   │
│  └──────────┘  └──────────┘  └──────────┘  └──────────┘   │
└─────────────────────────────────────────────────────────────┘
```

---

## Main Components

### 1. **CLI Layer** (`internal/cmd/`)

**Entry Points:**
- **`main.go`**: Application entry point, sets up pprof profiling
- **`root.go`**: Root Cobra command that sets up the app and launches TUI
- **`run.go`**: Non-interactive mode for single prompts

The CLI uses [Cobra](https://github.com/spf13/cobra) for command parsing. When you run `crush`, it:
1. Loads configuration from `.crush.json` or global config
2. Initializes the SQLite database
3. Creates all services (sessions, messages, permissions, LSP)
4. Builds the agent coordinator
5. Launches the Bubbletea TUI (or runs non-interactively)

### 2. **Application Layer** (`internal/app/`)

**`app.go`** wires together all services:

```go
type App struct {
    Sessions         session.Service     // Database-backed session storage
    Messages         message.Service     // Database-backed message storage
    History          history.Service     // File edit history
    Permissions      permission.Service  // User permission management
    FileTracker      filetracker.Service // Tracks which files were read
    AgentCoordinator agent.Coordinator   // AI agent orchestration
    LSPManager       *lsp.Manager        // LSP server management
}
```

**Key responsibilities:**
- **Service initialization**: Creates database connections, initializes all services
- **Event coordination**: Uses pubsub patterns for service communication
- **Lifecycle management**: Handles startup, shutdown, and cleanup
- **Agent initialization**: Sets up the coder agent with tools and LLM providers

### 3. **Agent Layer** (`internal/agent/`)

The agent is the brain of Crush. It consists of:

**`coordinator.go`**: Orchestrates agent creation and execution
- Creates agents with configured LLM providers
- Merges model options from multiple sources (Catwalk, config, user)
- Handles OAuth token refresh and API key rotation
- Routes prompts to the appropriate agent

**`agent.go`**: The actual AI session agent
- Manages conversation history and context
- Implements automatic summarization for long conversations
- Coordinates tool execution (bash, edit, view, etc.)
- Handles streaming responses from LLMs
- Manages a "small model" for summaries and title generation

**`prompts.go`**: Template-based prompt generation
- Builds system prompts from templates (`coder.md.tpl`, `task.md.tpl`)
- Includes project context (git status, context files)
- Injects available tools and skills

### 4. **Tool System** (`internal/agent/tools/`)

Tools are functions the AI can call to interact with your system:

| Tool | Purpose |
|------|---------|
| `bash` | Execute shell commands (with security blocking) |
| `edit` | Find/replace text in files (requires exact match) |
| `multiedit` | Multiple sequential edits in one operation |
| `write` | Create or overwrite files |
| `view` | Read file contents with line numbers |
| `grep` | Search file contents using ripgrep |
| `glob` | Find files by pattern |
| `ls` | List directory contents |
| `web_search` | Search the web via DuckDuckGo |
| `fetch` | Fetch raw content from URLs |
| `agentic_fetch` | Web search + AI analysis |
| `lsp_restart` | Restart LSP servers |
| `mcp/*` | Interface with MCP servers |

**Tool Definition Pattern:**
```go
func NewEditTool(...) fantasy.AgentTool {
    return fantasy.NewAgentTool(
        "edit",                      // Tool name
        string(editDescription),     // Usage documentation (passed to LLM)
        func(ctx context.Context, params EditParams, call fantasy.ToolCall) (fantasy.ToolResponse, error) {
            // 1. Validate parameters
            // 2. Request user permission if needed
            // 3. Perform the edit
            // 4. Trigger LSP diagnostics
            // 5. Return response
        },
    )
}
```

### 5. **Permission System** (`internal/permission/`)

Before tools execute, they may request user approval:

```
┌─────────────────────────────────────────────────┐
│  Agent wants to: run `go test ./...`            │
│  Permission request sent to TUI                 │
│  ┌─────────────────────────────────────────┐    │
│  │  Allow bash command?                    │    │
│  │  Command: go test ./...                 │    │
│  │  [y] Yes  [n] No  [a] Always allow     │    │
│  └─────────────────────────────────────────┘    │
└─────────────────────────────────────────────────┘
```

Configuration can pre-approve tools:
```json
{
  "permissions": {
    "allowed_tools": ["view", "ls", "grep", "edit"]
  }
}
```

### 6. **LSP Integration** (`internal/lsp/`)

Crush uses Language Server Protocol for code intelligence:

**Workflow:**
1. User has `gopls` configured in `crush.json`
2. App starts `gopls` as a subprocess
3. LSP client sends `textDocument/didOpen` when agent reads files
4. Agent receives diagnostics (errors, warnings) via LSP
5. Diagnostics displayed in TUI and influence agent responses

**LSP Manager:**
```go
type Manager struct {
    clients map[string]*Client  // Language name -> LSP client
    config  config.LSPConfig    // User's LSP configuration
}
```

### 7. **UI Layer** (`internal/ui/`)

Built with [Bubbletea](https://github.com/charmbracelet/bubbletea):

**`model/ui.go`**: Main UI model
- Handles all keyboard/mouse input
- Routes messages to appropriate components
- Manages layout (sidebar, chat, status bar)

**`model/chat.go`**: Chat message display
- Renders user and assistant messages
- Shows tool calls and their results
- Handles expanding/collapsing tool outputs

**`diffview/`**: Unified/split diff viewer
- Syntax-highlighted code diffs
- Supports unified and split views
- Used for displaying file edits

---

## How the CLI Works

### Startup Sequence

```
1. user runs: crush
   │
2. main.go → cmd.Execute()
   │
3. Cobra parses flags and commands
   │
4. rootCmd.RunE executes:
   │
5. ├── setupApp() creates App instance
   │        ├── Load config (.crush.json)
   │        ├── Open SQLite database (.crush/crush.db)
   │        ├── Create services (sessions, messages, permissions, LSP)
   │        ├── Init MCP servers (async)
   │        ├── Build agent coordinator
   │        └── Start background services (update checker, etc.)
   │
6. └── Launch Bubbletea TUI
          │
7. TUI Main Loop:
          ├── User types prompt in editor
          ├── Prompt sent to AgentCoordinator.Run()
          ├── Agent processes prompt (makes LLM calls, uses tools)
          ├── Responses streamed back to TUI
          └── TUI renders messages to terminal
```

### Configuration Loading

Config is loaded in priority order:
1. `.crush.json` (project root)
2. `crush.json` (project root)
3. `~/.config/crush/crush.json` (global)

**Example config:**
```json
{
  "$schema": "https://charm.land/crush.json",
  "agents": {
    "coder": {
      "large_model": {
        "provider": "openai",
        "model": "gpt-4"
      },
      "small_model": {
        "provider": "openai",
        "model": "gpt-4o-mini"
      }
    }
  },
  "lsp": {
    "go": {
      "command": "gopls"
    }
  }
}
```

---

## Complete Example Flow

### User Request: "Refactor this project to include an HTTP API endpoint, similar to existing CLI commands"

Let's trace this request through the entire system, step by step.

#### Step 1: User Input

User types the prompt in the TUI editor and presses Enter.

```
┌─────────────────────────────────────────────────────────────┐
│ Crush                                        [ Sessions ]   │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  User: Refactor this project to include an HTTP API        │
│        endpoint, similar to existing CLI commands          │
│                                                             │
│  [Assistant is thinking...]                                │
└─────────────────────────────────────────────────────────────┘
```

**Internal flow:**
```go
// TUI sends UserInputMsg to main model
type UserInputMsg struct {
    Prompt string
}

// Main model handles input
func (m *Model) Update(msg tea.Msg) tea.Cmd {
    case UserInputMsg:
        // Clear editor, show loading state
        return m.agentCoordinator.Run(sessionID, msg.Prompt)
}
```

#### Step 2: Agent Coordinator Receives Prompt

```go
// internal/agent/coordinator.go:118
func (c *coordinator) Run(ctx context.Context, sessionID string, prompt string, attachments ...message.Attachment) (*fantasy.AgentResult, error) {
    // 1. Wait for agent to be ready
    c.readyWg.Wait()

    // 2. Refresh models (in case config changed)
    c.UpdateModels(ctx)

    // 3. Get model configuration
    model := c.currentAgent.Model()  // e.g., gpt-4

    // 4. Get provider config
    providerCfg, ok := c.cfg.Providers.Get("openai")

    // 5. Call the agent
    return c.currentAgent.Run(ctx, SessionAgentCall{
        SessionID:       sessionID,
        Prompt:          prompt,
        Attachments:     attachments,
        MaxOutputTokens: 4096,
        ProviderOptions: mergedOptions,
    })
}
```

#### Step 3: Agent Builds System Prompt

```go
// internal/agent/agent.go:Run()
func (s *sessionAgent) Run(ctx context.Context, call SessionAgentCall) (*fantasy.AgentResult, error) {
    // 1. Build system prompt from template
    systemPrompt, _ := s.systemPrompt.Build(ctx, provider, model, s.cfg)

    // System prompt includes:
    // - Critical rules (read before editing, be autonomous, test after changes, etc.)
    // - Communication style (concise, no preamble)
    // - Workflow guidelines (think, act, verify)
    // - Available tools with descriptions

    // 2. Gather conversation history
    messages, _ := s.messages.List(ctx, call.SessionID)

    // 3. Gather context files
    contextFiles := s.gatherContextFiles(ctx)  // AGENTS.md, .cursorrules, etc.

    // 4. Create fantasy.Agent with tools
    agent := fantasy.NewAgent(
        s.largeModel,                    // gpt-4
        fantasy.WithSystemPrompt(systemPrompt),
        fantasy.WithTools(s.tools),      // bash, edit, view, grep, etc.
    )

    // 5. Run the agent
    return agent.Run(ctx, call.Prompt, messages...)
}
```

**System prompt excerpt:**
```
You are Crush, a powerful AI Assistant that runs in the CLI.

<critical_rules>
1. READ BEFORE EDITING: Never edit a file you haven't already read
2. BE AUTONOMOUS: Don't ask questions - search, read, think, decide, act
3. TEST AFTER CHANGES: Run tests immediately after each modification
...
</critical_rules>

<tools>
You have access to the following tools:
- bash: Execute shell commands
- edit: Edit files by finding and replacing text
- view: Read file contents
- grep: Search file contents
- glob: Find files by pattern
...
</tools>
```

#### Step 4: Agent Makes First LLM Call

The agent sends the prompt to the LLM (e.g., OpenAI's GPT-4):

```go
// fantasy library makes the HTTP request
response, err := llm.Complete(ctx, prompt, options)
```

**Request to OpenAI API:**
```json
{
  "model": "gpt-4",
  "messages": [
    {
      "role": "system",
      "content": "You are Crush, a powerful AI Assistant..."
    },
    {
      "role": "user",
      "content": "Refactor this project to include an HTTP API endpoint, similar to existing CLI commands"
    }
  ],
  "tools": [
    {"type": "function", "function": {"name": "view", "description": "..."}},
    {"type": "function", "function": {"name": "bash", "description": "..."}},
    {"type": "function", "function": {"name": "edit", "description": "..."}},
    ...
  ]
}
```

#### Step 5: LLM Responds with Tool Calls

The LLM realizes it needs to understand the project structure first:

```json
{
  "choices": [{
    "message": {
      "role": "assistant",
      "tool_calls": [
        {
          "id": "call_1",
          "function": {
            "name": "view",
            "arguments": "{\"file_path\":\"main.go\"}"
          }
        },
        {
          "id": "call_2",
          "function": {
            "name": "view",
            "arguments": "{\"file_path\":\"internal/cmd/root.go\"}"
          }
        },
        {
          "id": "call_3",
          "function": {
            "name": "ls",
            "arguments": "{\"path\":\"internal/cmd\"}"
          }
        }
      ]
    }
  }]
}
```

#### Step 6: Agent Executes Tools

**Tool 1: view main.go**

```go
// internal/agent/tools/view.go
func (ctx *editContext) viewFile(params ViewParams) (fantasy.ToolResponse, error) {
    // 1. Resolve file path
    filePath := filepathext.SmartJoin(workingDir, params.FilePath)

    // 2. Read file content
    content, _ := os.ReadFile(filePath)

    // 3. Track that file was read (for write safety)
    ctx.filetracker.MarkRead(sessionID, filePath)

    // 4. Return content to LLM
    return fantasy.NewTextResponse(content), nil
}
```

**Tool 2: ls internal/cmd**

```go
// internal/agent/tools/ls.go
func ListFiles(ctx context.Context, path string) (string, error) {
    // 1. List directory
    entries, _ := os.ReadDir(path)

    // 2. Filter ignored files
    entries = filterIgnores(entries)

    // 3. Format as tree
    return formatTree(entries), nil
}
```

**Agent sends tool results back to LLM:**
```json
{
  "role": "tool",
  "tool_call_id": "call_1",
  "content": "package main\n\nimport (\n    \"github.com/charmbracelet/crush/internal/cmd\"\n)\n\nfunc main() {\n    cmd.Execute()\n}"
}
```

#### Step 7: Agent Analyzes Project Structure

After reading the files, the LLM understands:
- `main.go` is the entry point
- Commands are defined in `internal/cmd/`
- Uses Cobra for CLI framework
- Commands include: `run`, `dirs`, `projects`, `logs`, `schema`, etc.

**LLM's next response:**
```json
{
  "tool_calls": [
    {
      "id": "call_4",
      "function": {
        "name": "view",
        "arguments": "{\"file_path\":\"internal/cmd/run.go\"}"
      }
    },
    {
      "id": "call_5",
      "function": {
        "name": "view",
        "arguments": "{\"file_path\":\"internal/app/app.go\"}"
      }
    }
  ]
}
```

The agent wants to understand how a command is structured and how the app works.

#### Step 8: Agent Designs HTTP API

After analyzing the code, the LLM determines:
- It should create a new file `internal/cmd/serve.go` for the HTTP server
- It should register a new `serve` command in `root.go`
- It should reuse existing app services
- It should use a standard Go HTTP server

**Tool calls to implement:**

```json
{
  "tool_calls": [
    {
      "id": "call_6",
      "function": {
        "name": "write",
        "arguments": "{\"file_path\":\"internal/cmd/serve.go\",\"content\":\"package cmd\\n\\nimport...\"}"
      }
    }
  ]
}
```

#### Step 9: Permission Request

Before creating the file, the agent requests permission:

```
┌─────────────────────────────────────────────────────────────┐
│  Allow write operation?                                     │
│                                                             │
│  File: internal/cmd/serve.go                               │
│  Action: Create new file (166 lines)                       │
│                                                             │
│  [y] Yes  [n] No  [v] View diff  [a] Always allow          │
└─────────────────────────────────────────────────────────────┘
```

```go
// internal/permission/permission.go
func (s *permissionService) Request(ctx context.Context, req CreatePermissionRequest) (bool, error) {
    // 1. Check if tool is pre-approved
    if s.isAllowed(req.ToolName) {
        return true, nil
    }

    // 2. Check if session is in yolo mode
    if s.isAutoApproved(req.SessionID) {
        return true, nil
    }

    // 3. Publish request to UI
    s.Publish(pubsub.CreatedEvent, req)

    // 4. Wait for user response
    resultChan := s.pendingRequests.Get(req.ID)
    select {
    case granted := <-resultChan:
        return granted, nil
    case <-ctx.Done():
        return false, ctx.Err()
    }
}
```

User approves → Tool executes → File is created

#### Step 10: Agent Registers New Command

```json
{
  "tool_calls": [
    {
      "id": "call_7",
      "function": {
        "name": "edit",
        "arguments": "{\n  \"file_path\": \"internal/cmd/root.go\",\n  \"old_string\": \"func init() {\\n\\trootCmd.AddCommand(\\n\\t\\trunCmd,\\n\\t\\tdirsCmd,\\n\\t\\tprojectsCmd,\\n\\t\\tupdateProvidersCmd,\\n\\t\\tlogsCmd,\\n\\t\\tschemaCmd,\\n\\t\\tloginCmd,\\n\\t\\tstatsCmd,\\n\\t)\\n}\",\n  \"new_string\": \"func init() {\\n\\trootCmd.AddCommand(\\n\\t\\trunCmd,\\n\\t\\tdirsCmd,\\n\\t\\tprojectsCmd,\\n\\t\\tupdateProvidersCmd,\\n\\t\\tlogsCmd,\\n\\t\\tschemaCmd,\\n\\t\\tloginCmd,\\n\\t\\tstatsCmd,\\n\\t\\tserveCmd,\\n\\t)\\n}\"\n}"
      }
    }
  ]
}
```

**Permission requested** → **User approves** → **File edited**

#### Step 11: Agent Tests the Changes

Per the critical rules, the agent must test after changes:

```json
{
  "tool_calls": [
    {
      "id": "call_8",
      "function": {
        "name": "bash",
        "arguments": "{\"command\":\"go build .\",\"description\":\"Build the project to check for errors\"}"
      }
    }
  ]
}
```

**Permission requested** → **User approves** → **Build succeeds**

```json
{
  "tool_calls": [
    {
      "id": "call_9",
      "function": {
        "name": "bash",
        "arguments": "{\"command\":\"go test ./...\",\"description\":\"Run tests to ensure nothing broke\"}"
      }
    }
  ]
}
```

**Permission requested** → **User approves** → **Tests pass**

#### Step 12: LSP Provides Diagnostics

When files are edited, the LSP is notified:

```go
// internal/lsp/client.go
func (c *Client) DidChange(filePath string, content []byte) error {
    return c.client.Notify(ctx, "textDocument/didChange", didChangeParams{
        TextDocument: textDocument{URI: toURI(filePath)},
        ContentChanges: []textDocumentContentChangeEvent{
            {Text: string(content)},
        },
    })
}
```

LSP responds with diagnostics (if any errors/warnings found).

#### Step 13: Agent Summarizes Work

After completing the task, the LLM provides a summary:

```
I've added an HTTP API endpoint to the project. Here's what was done:

1. Created `internal/cmd/serve.go` with an HTTP server that exposes
   API endpoints similar to existing CLI commands

2. Registered the new `serve` command in `internal/cmd/root.go`

3. The server implements:
   - GET /api/sessions - List all sessions
   - GET /api/sessions/:id - Get session details
   - POST /api/sessions - Create new session
   - POST /api/sessions/:id/prompt - Send prompt to session

4. Build and tests pass successfully

You can now run:
  crush serve --port 8080

To start the HTTP server.
```

#### Step 14: Response Streamed to TUI

As the agent works, its responses are streamed to the TUI in real-time:

```go
// internal/agent/agent.go
func (s *sessionAgent) Run(ctx context.Context, call SessionAgentCall) (*fantasy.AgentResult, error) {
    // Stream response to UI
    for chunk := range responseStream {
        s.eventBus.Publish(ResponseEvent{
            SessionID: call.SessionID,
            Content:   chunk.Content,
            ToolCalls: chunk.ToolCalls,
        })
    }
}
```

The TUI renders:
- Assistant messages in real-time (streaming)
- Tool calls as expandable sections
- Tool outputs with syntax highlighting
- Diagnostics inline with code

---

## Tool System Details

### Tool Lifecycle

```
┌─────────────────────────────────────────────────────────────┐
│  1. LLM decides to use a tool                              │
│     "I need to read main.go"                               │
└───────────────────────────┬─────────────────────────────────┘
                            │
                            ▼
┌─────────────────────────────────────────────────────────────┐
│  2. Agent validates tool call                              │
│     - Check parameters are valid                           │
│     - Get session context                                  │
└───────────────────────────┬─────────────────────────────────┘
                            │
                            ▼
┌─────────────────────────────────────────────────────────────┐
│  3. Permission check                                       │
│     - Is tool in allowed_tools list?                       │
│     - Is --yolo mode enabled?                              │
│     - Otherwise, request user approval                     │
└───────────────────────────┬─────────────────────────────────┘
                            │
                            ▼
┌─────────────────────────────────────────────────────────────┐
│  4. Tool execution                                         │
│     - Execute the operation                                │
│     - Track file for write safety                          │
│     - Trigger LSP diagnostics (for edits)                  │
└───────────────────────────┬─────────────────────────────────┘
                            │
                            ▼
┌─────────────────────────────────────────────────────────────┐
│  5. Result returned to LLM                                │
│     - Format response                                      │
│     - Include metadata (lines changed, errors, etc.)       │
└───────────────────────────┬─────────────────────────────────┘
                            │
                            ▼
┌─────────────────────────────────────────────────────────────┐
│  6. LLM continues with result                              │
│     - Process tool output                                  │
│     - Decide next action                                   │
└─────────────────────────────────────────────────────────────┘
```

### Tool Parameters

Each tool defines its parameters as a struct:

```go
type EditParams struct {
    FilePath   string `json:"file_path" description:"The absolute path to the file to modify"`
    OldString  string `json:"old_string" description:"The text to replace"`
    NewString  string `json:"new_string" description:"The text to replace it with"`
    ReplaceAll bool   `json:"replace_all,omitempty" description:"Replace all occurrences"`
}
```

The `description` tags are passed to the LLM so it knows how to use the tool.

### Banned Commands

The `bash` tool blocks dangerous commands:

```go
var bannedCommands = []string{
    "alias", "aria2c", "axel", "chrome", "curl", "curlie",
    "firefox", "http-prompt", "httpie", "links", "lynx",
    "nc", "safari", "scp", "ssh", "telnet", "w3m", "wget",
    "apt", "apt-get", "brew", "choco", "dnf", "emerge",
    "npm", "pacman", "pip", "pnpm", "yum", "zypper",
    "at", "batch", "crontab", "reboot", "shutdown", "systemctl",
}
```

This prevents the agent from accidentally damaging the system.

---

## Summary

Crush works by:

1. **User inputs prompt** → CLI captures it
2. **App coordinates services** → Sessions, messages, permissions, LSP
3. **Agent processes prompt** → Builds context, calls LLM
4. **LLM decides on actions** → Reads files, edits code, runs tests
5. **Tools execute operations** → With user permission
6. **LSP provides diagnostics** → Code intelligence
7. **Response streamed to UI** → Real-time feedback

The key insight is that Crush isn't just a chatbot—it's an **agent** with:
- **Memory**: Conversation history and file tracking
- **Tools**: Capabilities to interact with your system
- **Intelligence**: LLM for reasoning and planning
- **Oversight**: User permission system for safety
- **Context**: LSP diagnostics and project structure awareness

This combination enables Crush to perform complex, multi-step development tasks while keeping the user in control.
