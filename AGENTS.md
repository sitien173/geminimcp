# AGENTS.md

This file provides guidance to Qoder (qoder.com) when working with code in this repository.

## Project Overview

This is a Python MCP (Model Context Protocol) server that wraps the Gemini CLI. It exposes a `gemini` tool that allows MCP clients to invoke Gemini CLI sessions with conversation continuity via session IDs.

## Architecture

### Core Components

- **src/geminimcp/server.py**: FastMCP server implementation. Defines the `gemini` tool that:
  - Invokes the external `gemini` CLI process via subprocess
  - Streams JSON output from the CLI using a background thread with queue-based communication
  - Parses structured JSON events (message types, session IDs, turn completion)
  - Handles Windows-specific string escaping for prompts
  - Returns structured results with `success`, `SESSION_ID`, and `agent_messages`

- **src/geminimcp/cli.py**: Console entry point that calls `server.run()`

- **src/geminimcp/__init__.py**: Package version definition

### Key Design Patterns

1. **Subprocess Streaming**: The server spawns the Gemini CLI as a subprocess and reads output line-by-line using a background thread feeding a `queue.Queue`. This allows non-blocking streaming while detecting `turn.completed` events to terminate gracefully.

2. **JSON Event Parsing**: The CLI outputs JSON lines with event types. The parser extracts:
   - `type: "message"` + `role: "assistant"` → content for `agent_messages`
   - `session_id` → returned as `SESSION_ID` for conversation continuity
   - `type: "turn.completed"` → triggers graceful shutdown with 0.3s delay

3. **Windows Escape Handling**: Prompts are escaped using `windows_escape()` on Windows to handle special characters (newlines, quotes, backslashes) for command-line safety.

### External Dependencies

- **mcp[cli]**: FastMCP framework for MCP server implementation
- **pydantic**: Type validation with `BeforeValidator` and `Field`
- **gemini CLI**: Must be installed separately and available in PATH. The server shells out to this binary.

## Development Commands

This project uses `uv` for Python package management.

```bash
# Install dependencies
uv sync

# Run the MCP server (stdio transport)
uv run geminimcp

# Or run via module
uv run python -m geminimcp

# Build the package
uv build
```

## Project Structure

```
.
├── pyproject.toml          # Project metadata, dependencies, entry point
├── uv.lock                 # Locked dependency versions
├── .python-version         # Python 3.12
└── src/
    └── geminimcp/
        ├── __init__.py     # Version info
        ├── cli.py          # Entry point
        └── server.py       # MCP server and tool definitions
```

## Important Notes

- The `gemini` CLI binary must be installed and in PATH. This is an external dependency not managed by uv.
- The server uses `--approval-mode=yolo` and `--allowed-mcp-server-names=[*]` flags when invoking Gemini.
- Session continuity is achieved by passing `SESSION_ID` between calls; empty string starts a new session.
- The server runs on stdio transport for MCP communication.
