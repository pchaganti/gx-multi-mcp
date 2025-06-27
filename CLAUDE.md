# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

Multi-MCP is a Python-based proxy server that acts as a single MCP (Model Context Protocol) server while connecting to and routing between multiple backend MCP servers. It supports both STDIO and SSE (Server-Sent Events) transports and can dynamically add/remove MCP servers at runtime.

## Architecture

### Core Components

- **MultiMCP (`src/multimcp/multi_mcp.py`)**: Main server orchestrator that handles configuration, transport modes, and HTTP endpoints
- **MCPProxyServer (`src/multimcp/mcp_proxy.py`)**: Core proxy that forwards MCP requests to backend servers, handles tool namespacing (`server_name_tool_name`), and manages capabilities aggregation
- **MCPClientManager (`src/multimcp/mcp_client.py`)**: Manages lifecycle of multiple MCP client connections (both stdio and SSE)
- **Logger (`src/utils/logger.py`)**: Centralized logging using Rich handlers with `multi_mcp.*` namespace

### Key Patterns

- **Tool Namespacing**: Tools are namespaced as `server_name_tool_name` to avoid conflicts when multiple servers expose tools with the same name
- **Transport Flexibility**: Supports both STDIO (for CLI/pipe-based) and SSE (for HTTP/network-based) communication
- **Dynamic Server Management**: Can add/remove MCP servers at runtime via HTTP API (SSE mode only)
- **Capability Aggregation**: Proxies and combines tools, prompts, and resources from all connected backend servers

## Development Commands

### Running the Server
```bash
# STDIO mode (default)
uv run main.py --transport stdio

# SSE mode with custom host/port
uv run main.py --transport sse --host 0.0.0.0 --port 8080

# With custom config file
uv run main.py --config ./examples/config/mcp_k8s.json

# Quick development run
make run
```

### Testing
```bash
# Individual test suites
make test-proxy      # Core proxy functionality  
make test-e2e        # End-to-end integration tests
make test-lifecycle  # Client lifecycle management
make test-k8s        # Kubernetes deployment tests (requires docker-build)

# All tests
make all-test

# Run specific test file directly
pytest -s tests/proxy_test.py
```

### Docker & Kubernetes
```bash
# Build and run locally
make docker-build
make docker-run

# Kubernetes with Kind
kind create cluster --name multi-mcp-test
kind load docker-image multi-mcp --name multi-mcp-test
kubectl apply -f examples/k8s/multi-mcp.yaml
```

### Docker Container Details
- **Base Image**: `ghcr.io/astral-sh/uv:python3.12-bookworm-slim` (Debian-based)
- **Node.js Runtime**: Node.js 20.x installed for MCP servers requiring Node runtime
- **Production Config**: Uses `./msc/mcp.json` with GitHub, Brave Search, and Context7 MCP servers
- **Network**: Binds to `0.0.0.0` for container accessibility

### Dependency Management
```bash
# Install dependencies
uv venv
uv pip install -r requirements.txt

# Alternative: Direct run (handles dependencies automatically)
uv run main.py
```

## Configuration

### MCP Server Configuration
Configuration is JSON-based, defining backend MCP servers to connect to:

```json
{
  "mcpServers": {
    "weather": {
      "command": "python",
      "args": ["./tools/get_weather.py"],
      "env": {"API_KEY": "value"}
    },
    "remote_service": {
      "url": "http://127.0.0.1:9080/sse"
    }
  }
}
```

### Runtime HTTP API (SSE Mode Only)
- `GET /mcp_servers` - List active servers
- `POST /mcp_servers` - Add new servers
- `DELETE /mcp_servers/{name}` - Remove server
- `GET /mcp_tools` - List all tools by server

## Code Conventions

### Error Handling
- Use Rich logging with emoji prefixes: `✅ success`, `❌ errors`, `⚠️ warnings`
- Log errors but don't raise exceptions for individual server failures
- Gracefully handle missing tools/prompts/resources with appropriate error responses

### Async Patterns
- Use `AsyncExitStack` for managing multiple client lifecycles
- Prefer `async with` for resource management
- Always await client operations and handle exceptions per-client

### Type Hints
- Use `from typing import` imports for type annotations
- Prefer `Optional[Type]` over `Type | None` for Python 3.10 compatibility
- Use `Literal` for string enums (transport modes, log levels)

## Development Notes

### Tool Namespacing Implementation
Tools are internally namespaced using `_make_key()` and `_split_key()` static methods in `MCPProxyServer`:
- `_make_key(server_name, tool_name)` creates `server_name_tool_name` identifiers
- `_split_key(key)` splits namespaced keys back into `(server, tool)` tuples using first underscore
- Namespaced tools are stored in `tool_to_server` dict mapping keys to `ToolMapping` objects
- This allows multiple servers to expose tools with identical base names without conflicts

### Transport Modes
- **STDIO**: Pipe-based communication for CLI tools and local agents
- **SSE**: HTTP Server-Sent Events for browser agents and network access

### Testing Strategy
- **Unit tests**: Focus on individual components (proxy, client manager)
- **Integration tests**: End-to-end flows with real MCP tools
- **K8s tests**: Deployment and service exposure validation

### Project Structure
- `main.py` - Entry point CLI interface using `MultiMCP` class
- `src/multimcp/multi_mcp.py` - Main orchestrator with `MCPSettings` and HTTP endpoints
- `src/multimcp/mcp_proxy.py` - Core proxy server with request forwarding and capability aggregation
- `src/multimcp/mcp_client.py` - Client lifecycle management using `AsyncExitStack`
- `src/utils/logger.py` - Centralized Rich logging with `multi_mcp.*` namespace
- `tests/` - Comprehensive test suite with mock MCP servers and fixtures
- `examples/config/` - Sample configuration files for different deployment scenarios

### Dependencies
Key dependencies include:
- `mcp>=1.4.1` - Core MCP protocol implementation
- `langchain-mcp-adapters` - LangChain integration utilities  
- `starlette` + `uvicorn` - SSE HTTP server
- `httpx-sse` - SSE client support
- `rich` - Enhanced logging and console output
- `pytest` + `pytest-asyncio` - Testing framework with async support