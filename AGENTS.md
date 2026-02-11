# Crush Development Guide

## Build/Test/Lint Commands

- **Build**: `go build .` or `go run .`
- **Test**: `task test` or `go test ./...` (run single test: `go test ./internal/llm/prompt -run TestGetContextFromPaths`)
- **Update Golden Files**: `go test ./... -update` (regenerates .golden files when test output changes)
  - Update specific package: `go test ./internal/tui/components/core -update` (in this case, we're updating "core")
- **Lint**: `task lint:fix`
- **Format**: `task fmt` (`gofumpt -w .`)
- **Modernize**: `task modernize` (runs `modernize` which make code simplifications)
- **Dev**: `task dev` (runs with profiling enabled)

## Code Style Guidelines

- **Imports**: Use `goimports` formatting, group stdlib, external, internal packages
- **Formatting**: Use gofumpt (stricter than gofmt), enabled in golangci-lint
- **Naming**: Standard Go conventions - PascalCase for exported, camelCase for unexported
- **Types**: Prefer explicit types, use type aliases for clarity (e.g., `type AgentName string`)
- **Error handling**: Return errors explicitly, use `fmt.Errorf` for wrapping
- **Context**: Always pass `context.Context` as first parameter for operations
- **Interfaces**: Define interfaces in consuming packages, keep them small and focused
- **Structs**: Use struct embedding for composition, group related fields
- **Constants**: Use typed constants with iota for enums, group in const blocks
- **Testing**: Use testify's `require` package, parallel tests with `t.Parallel()`,
  `t.SetEnv()` to set environment variables. Always use `t.Tempdir()` when in
  need of a temporary directory. This directory does not need to be removed.
- **JSON tags**: Use snake_case for JSON field names
- **File permissions**: Use octal notation (0o755, 0o644) for file permissions
- **Log messages**: Log messages must start with a capital letter (e.g., "Failed to save session" not "failed to save session")
  - This is enforced by `task lint:log` which runs as part of `task lint`
- **Comments**: End comments in periods unless comments are at the end of the line.

## Testing with Mock Providers

When writing tests that involve provider configurations, use the mock providers to avoid API calls:

```go
func TestYourFunction(t *testing.T) {
    // Enable mock providers for testing
    originalUseMock := config.UseMockProviders
    config.UseMockProviders = true
    defer func() {
        config.UseMockProviders = originalUseMock
        config.ResetProviders()
    }()

    // Reset providers to ensure fresh mock data
    config.ResetProviders()

    // Your test code here - providers will now return mock data
    providers := config.Providers()
    // ... test logic
}
```

## Formatting

- ALWAYS format any Go code you write.
  - First, try `gofumpt -w .`.
  - If `gofumpt` is not available, use `goimports`.
  - If `goimports` is not available, use `gofmt`.
  - You can also use `task fmt` to run `gofumpt -w .` on the entire project,
    as long as `gofumpt` is on the `PATH`.

## Comments

- Comments that live on their own lines should start with capital letters and
  end with periods. Wrap comments at 78 columns.

## Committing

- ALWAYS use semantic commits (`fix:`, `feat:`, `chore:`, `refactor:`, `docs:`, `sec:`, etc).
- Try to keep commits to one line, not including your attribution. Only use
  multi-line commits when additional context is truly necessary.

## Working on the TUI (UI)
Anytime you need to work on the tui before starting work read the internal/ui/AGENTS.md file

## Project Architecture

Crush is a CLI-based AI coding assistant built with Go and Bubbletea.

### Key Components

- **`internal/agent/`** - Core AI agent orchestration
  - `agent.go` - Main session agent implementation
  - `coordinator.go` - Agent coordination and tool management
  - `tools/` - Tool implementations (bash, edit, view, multiedit, etc.)
  - `prompts.go` - Prompt template functions for coder/task/initialize agents
  - Two agent types: `coder` (main agent) and `task` (sub-agent for tool use)

- **`internal/ui/`** - Bubbletea-based terminal UI
  - `model/ui.go` - Main UI model with message routing
  - `model/chat.go` - Chat message display and interaction
  - `diffview/` - Unified/split diff viewing with syntax highlighting
  - `dialog/` - Dialog implementations (models, sessions, permissions, etc.)

- **`internal/config/`** - Configuration management
  - Uses Catwalk (charm.land/catwalk) for provider/model definitions
  - Provider auto-update from remote Catwalk service
  - Embedded providers as fallback
  - Environment variable expansion: `$(echo $VAR)` syntax

- **`internal/lsp/`** - Language Server Protocol client management
  - Manages LSP server lifecycles (start/stop/restart)
  - Diagnostic caching and change notifications
  - Used for code context and error information

- **`internal/shell/`** - Cross-platform shell execution
  - Uses `mvdan.cc/sh/v3` for POSIX shell emulation on all platforms
  - **IMPORTANT**: Use forward slashes `/` for paths on all platforms including Windows
  - Background job management for long-running commands
  - Command blocking for security (e.g., `curl`, `rm`, etc.)

- **`internal/permission/`** - Permission system
  - Users must approve tool executions (bash, edit, etc.)
  - `--yolo` flag auto-approves all permissions (dangerous)
  - `permissions.allowed_tools` config for pre-approved tools

- **`internal/agent/tools/mcp/`** - Model Context Protocol support
  - Supports `stdio`, `http`, and `sse` transport types
  - MCP servers can provide additional tools and resources
  - Tools from MCPs are integrated into the agent tool set

- **`internal/db/`** - SQLite database
  - Uses sqlc for type-safe SQL queries
  - Migrations in `internal/db/migrations/`
  - Stores sessions, messages, todos, read files

- **`internal/session/`** - Session management
  - Session-based conversations with parent-child relationships
  - Title generation sessions for naming
  - Task sessions for sub-agent tool calls
  - Token usage and cost tracking

- **`internal/message/`** - Message storage
  - Multi-part messages (text, images, tool calls, etc.)
  - Summary message support for long conversations

### Configuration Files

Priority order:
1. `.crush.json` (project root)
2. `crush.json` (project root)
3. `$HOME/.config/crush/crush.json` (global)

Default context paths searched:
- `.github/copilot-instructions.md`
- `.cursorrules`, `.cursor/rules/`
- `CLAUDE.md`, `CLAUDE.local.md`
- `AGENTS.md`, `agents.md`, `Agents.md`
- etc.

### Agent Types

- **Coder Agent** (`AgentCoder`): Main agent for coding tasks
  - Reads files, edits code, runs tests, searches code
  - Uses large model for reasoning, small model for summaries

- **Task Agent** (`AgentTask`): Sub-agent for delegated tasks
  - Spawned via the `agent` tool
  - Simplified prompt for concise responses

- **Initialize Agent**: Creates project context files (AGENTS.md)
  - Analyzes codebase structure
  - Documents build commands, patterns, conventions

### Tools System

Tools are defined in `internal/agent/tools/`:
- `bash.go` - Execute shell commands (with security blocking)
- `edit.go` - Find/replace in files (exact match required)
- `multiedit.go` - Multiple sequential edits in one operation
- `write.go` - Create/overwrite files
- `view.go` - Read file contents with line numbers
- `grep.go` - Search file contents
- `glob.go` - Find files by pattern
- `ls.go` - List directory contents
- `web_search.go` - Search the web via DuckDuckGo
- `fetch.go` - Fetch raw content from URLs
- `agentic_fetch_tool.go` - Web search + AI analysis
- `mcp/` - MCP server integration for external tools

### VCR Testing

Tests use `charm.land/x/vcr` for HTTP recording:
- Cassettes stored in `internal/agent/testdata/`
- Update cassettes: `task test:record` or `go test -v -count=1 ./internal/agent`
- Each model provider has its own cassette directory

### LSP Integration

LSP clients are managed per-language:
- Defined in config `lsp` section
- Example: `gopls`, `typescript-language-server`, `nil`
- Can be restarted via `lsp_restart` tool
- Diagnostics integrated into editor view

### Cross-Platform Notes

- Shell execution is POSIX-emulated on all platforms
- Path handling: always use forward slashes `/`
- File permissions use octal notation (0o755, 0o644)
- Tests may skip on Windows (check for `runtime.GOOS == "windows"`)
