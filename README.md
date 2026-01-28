# Cursor-To-OpenAI

OpenAI-compatible API proxy for Cursor Editor with **full agent mode and tool calling support**.

> **Compatible with Cursor 2.3.41** - Protocol implementation based on reverse-engineered protobuf schemas.

## Features

- **OpenAI API compatibility** - Works with any OpenAI client (Python, Node.js, curl, etc.)
- **Agent mode with tool calling** - Execute local tools via bidirectional HTTP/2 streaming
- **Supported tools**: `list_dir`, `read_file`, `edit_file`, `run_terminal_cmd`, `grep_search`, `file_search`, `glob_search`, `delete_file`
- **Streaming responses** - SSE streaming for real-time output
- **Multiple models** - Access Claude, GPT-4, and other models available in Cursor

## Quick Start

```bash
# Install
npm install

# Get auth token (opens browser for Cursor login)
npm run login

# Start server
npm start
# Server runs on http://localhost:3010
```

## Usage

### With OpenAI Python client

```python
from openai import OpenAI

client = OpenAI(
    api_key="YOUR_CURSOR_TOKEN",  # From `npm run login`
    base_url="http://localhost:3010/v1"
)

# Simple chat
response = client.chat.completions.create(
    model="claude-3.5-sonnet",
    messages=[{"role": "user", "content": "Hello!"}]
)

# Agent mode with tools
response = client.chat.completions.create(
    model="claude-3.5-sonnet",
    messages=[{"role": "user", "content": "List files in /tmp"}],
    tools=[{"type": "function", "function": {"name": "run_terminal_cmd"}}]
)
```

### With curl

```bash
# Chat completion
curl http://localhost:3010/v1/chat/completions \
  -H "Authorization: Bearer YOUR_CURSOR_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"model": "claude-3.5-sonnet", "messages": [{"role": "user", "content": "Hello"}]}'

# With streaming
curl http://localhost:3010/v1/chat/completions \
  -H "Authorization: Bearer YOUR_CURSOR_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"model": "claude-3.5-sonnet", "messages": [{"role": "user", "content": "Hello"}], "stream": true}'
```

### With Crush CLI

[Crush](https://github.com/charmbracelet/crush) is a terminal-based AI assistant. Configure it to use this proxy:

**1. Install Crush**
```bash
# macOS/Linux
brew install charmbracelet/tap/crush

# Or download from GitHub releases
```

**2. Configure provider** in `~/.local/share/crush/crush.json`:
```json
{
  "default_provider": "cursor-bridge",
  "default_model": "claude-4.5-opus-high-thinking",
  "providers": {
    "cursor-bridge": {
      "kind": "openai",
      "api_key": "YOUR_CURSOR_TOKEN",
      "url": "http://localhost:3010/v1"
    }
  }
}
```

**3. Start the proxy** (in a separate terminal):
```bash
cd cursor-to-openai
npm start
```

**4. Run Crush**:
```bash
# Interactive TUI
crush

# One-shot query
crush run "list files in current directory"

# With specific model
crush --model claude-3.5-sonnet run "explain this code"
```

Crush will automatically use agent mode with tool calling when appropriate, allowing the AI to execute commands, read files, and perform other tasks locally.

## API Endpoints

| Endpoint | Method | Description |
|----------|--------|-------------|
| `/v1/models` | GET | List available models |
| `/v1/chat/completions` | POST | Chat completion (streaming supported) |
| `/cursor/loginDeepControl` | GET | Get auth token via browser login |

## Agent Mode

When `tools` array is provided in the request, the proxy enables **bidirectional HTTP/2 streaming** to execute tools locally and return results to the model.

Supported tools (mapped to Cursor's `ClientSideToolV2`):
- `list_dir` - List directory contents
- `read_file` - Read file contents
- `edit_file` - Edit/create files
- `run_terminal_cmd` - Execute shell commands
- `grep_search` - Search with ripgrep
- `file_search` - Search files by name
- `glob_search` - Search files by glob pattern
- `delete_file` - Delete files

## Authentication

Get your Cursor token using one of these methods:

### Method 1: CLI login
```bash
npm run login
# Opens browser, returns token after login
```

### Method 2: From Cursor IDE
Extract token from Cursor's IndexedDB or use the auth reader script.

### Method 3: API endpoint
```bash
curl http://localhost:3010/cursor/loginDeepControl \
  -H "Authorization: Bearer YOUR_WORKOS_SESSION_TOKEN"
```

## Architecture

```
Client (OpenAI SDK) 
    ↓ HTTP/1.1
cursor-to-openai proxy (localhost:3010)
    ↓ HTTP/2 bidirectional streaming
Cursor API (api2.cursor.sh)
    ↓
Claude/GPT models
```

For agent mode, the proxy:
1. Encodes request with `isAgentic=true` and `supportedTools` (protobuf)
2. Opens bidirectional HTTP/2 stream to Cursor API
3. Receives tool calls, executes locally, sends results back
4. Streams final response to client

## Development

```bash
# Run with auto-reload
npm run dev

# Regenerate protobuf JS
npm run proto
```

## Compatibility

**Tested with Cursor 2.3.41**

The protobuf schemas and protocol details were derived from reverse engineering Cursor's `workbench.desktop.main.js`. Key discoveries:
- `StreamUnifiedChatWithTools` RPC for bidirectional streaming
- `ClientSideToolV2` enum with 44 tool types
- `isAgentic` (field 27) and `supportedTools` (field 29) for agent mode
- Tool call/result message formats

## Reverse Engineering

The protocol analysis and standalone proof-of-concept implementations are available at:

**[eisbaw/cursor_api_demo](https://github.com/eisbaw/cursor_api_demo)** - Python PoC with:
- Protobuf wire format encoder/decoder
- HTTP/2 bidirectional streaming client (h2 library)
- Tool call detection and result encoding
- Analysis documents (TASK-7, TASK-26, TASK-110)

## Credits

- Fork of [JiuZ-Chn/Cursor-To-OpenAI](https://github.com/JiuZ-Chn/Cursor-To-OpenAI)
- Based on [zhx47/cursor-api](https://github.com/zhx47/cursor-api)
- Protocol analysis from [eisbaw/cursor_api_demo](https://github.com/eisbaw/cursor_api_demo)

## License

MIT
