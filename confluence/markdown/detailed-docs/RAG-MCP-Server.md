# RAG Service - MCP Server Integration

> This guide explains the Model Context Protocol (MCP) server integration for the RAG service, enabling AI assistants like Claude and Cursor to access RAG functionality through a standard protocol.

---

## Overview

### What is MCP?

Model Context Protocol (MCP) is a standard for connecting AI systems with external tools and data sources. It provides a unified, JSON-RPC–based interface for LLMs to call tools.

### MCP Architecture

```text
┌─────────────────────────────────────────────────────────────┐
│                    MCP Architecture                         │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  AI Assistant (Claude, Cursor, etc.)                        │
│       ↓                                                     │
│  MCP Client                                                 │
│       ↓                                                     │
│  ┌─────────────────────────────────────┐                   │
│  │         MCP Protocol Layer          │                   │
│  │  • JSON-RPC 2.0                     │                   │
│  │  • HTTP/SSE/WebSocket/stdio         │                   │
│  └─────────────────────────────────────┘                   │
│       ↓                                                     │
│  MCP Server (RAG Service)                                   │
│       ↓                                                     │
│  ┌─────────────────────────────────────┐                   │
│  │         Available Tools             │                   │
│  │  • retrieve_documents               │                   │
│  │  • search_documents                 │                   │
│  │  • list_documents                   │                   │
│  │  • query_knowledge_base             │                   │
│  └─────────────────────────────────────┘                   │
│       ↓                                                     │
│  RAG Configurations & Vector Stores                         │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

---

## MCP Server Configuration

### Configuration in RAG

```json
// mcp-server-config.json
{
  "configuration_name": "company-knowledge-mcp",
  "mcp_server": {
    "enabled": true,
    "startup_enabled": true,
    "name": "Company Knowledge Base MCP",
    "description": "Access company documentation and knowledge base",
    "version": "1.0.0",
    "protocols": ["http", "sse", "stdio"],
    "http_host": "0.0.0.0",
    "http_port": 8080,
    "base_path": "/company-kb/mcp",
    "tools": [
      {
        "type": "retrieve",
        "enabled": true,
        "name": "retrieve_documents",
        "description": "Retrieve relevant documents from the knowledge base",
        "max_results": 10,
        "include_metadata": true,
        "include_scores": true
      },
      {
        "type": "search",
        "enabled": true,
        "name": "search_documents",
        "description": "Search for specific information in documents",
        "max_results": 20,
        "search_type": "semantic"
      },
      {
        "type": "list",
        "enabled": true,
        "name": "list_documents",
        "description": "List all available documents",
        "include_stats": true
      },
      {
        "type": "query",
        "enabled": true,
        "name": "query_knowledge_base",
        "description": "Ask questions and get AI-generated answers",
        "include_sources": true,
        "max_context_length": 4000
      }
    ],
    "inherit_security": true,
    "rate_limiting": {
      "enabled": true,
      "requests_per_minute": 60,
      "requests_per_hour": 1000
    },
    "allowed_clients": ["claude-desktop", "cursor", "custom-client"],
    "metadata": {
      "department": "engineering",
      "owner": "ai-team",
      "classification": "internal"
    }
  }
}
```

---

## MCP Server Implementation

### Server Core (Pattern)

```python
# mcp_server.py
from typing import Dict, Any
from fastapi import FastAPI, Request, HTTPException

class MCPServer:
    def __init__(self, config: Dict[str, Any], rag_service):
        self.config = config
        self.rag_service = rag_service
        self.app = FastAPI()
        self.tools = self._initialize_tools()
        self._setup_routes()

    def _initialize_tools(self) -> Dict[str, Any]:
        tools: Dict[str, Any] = {}
        for tool_cfg in self.config.get("tools", []):
            if not tool_cfg.get("enabled"):
                continue
            ttype = tool_cfg["type"]
            name = tool_cfg["name"]
            if ttype == "retrieve":
                tools[name] = self._create_retrieve_tool(tool_cfg)
            elif ttype == "search":
                tools[name] = self._create_search_tool(tool_cfg)
            elif ttype == "list":
                tools[name] = self._create_list_tool(tool_cfg)
            elif ttype == "query":
                tools[name] = self._create_query_tool(tool_cfg)
        return tools

    def _setup_routes(self) -> None:
        @self.app.post("/")
        async def handle_json_rpc(request: Request):
            body = await request.json()
            if body.get("jsonrpc") != "2.0":
                return self._error_response(-32600, "Invalid Request", body.get("id"))

            method = body.get("method")
            params = body.get("params", {})
            request_id = body.get("id")

            if method == "tools/list":
                return self._success_response(request_id, self._list_tools())

            if method.startswith("tools/execute"):
                tool_name = params.get("name")
                tool = self.tools.get(tool_name)
                if not tool:
                    return self._error_response(-32601, "Tool not found", request_id)
                result = await tool(params.get("arguments", {}))
                return self._success_response(request_id, result)

            return self._error_response(-32601, "Method not found", request_id)

    def _list_tools(self) -> Dict[str, Any]:
        return {
            "tools": [
                {"name": name, "description": cfg.get("description")}
                for name, cfg in self.config.get("tools", [])
            ]
        }

    def _success_response(self, request_id: Any, result: Any) -> Dict[str, Any]:
        return {"jsonrpc": "2.0", "id": request_id, "result": result}

    def _error_response(self, code: int, message: str, request_id: Any) -> Dict[str, Any]:
        return {"jsonrpc": "2.0", "id": request_id, "error": {"code": code, "message": message}}
```

(Actual implementations of `_create_retrieve_tool`, `_create_search_tool`, etc. will call RAG retrieval and query functions and shape the responses into MCP tool outputs.)

---

## Client Configuration Examples

### Claude Desktop Configuration

```json
{
  "mcpServers": {
    "company-knowledge": {
      "command": "http",
      "args": ["http://localhost:8080/company-kb/mcp"],
      "env": {
        "MCP_TOKEN": "your-jwt-or-api-key"
      }
    }
  }
}
```

### Cursor MCP Configuration

```json
{
  "servers": [
    {
      "name": "RAG Knowledge Base",
      "type": "http",
      "url": "http://localhost:8080/rag/mcp",
      "headers": {
        "Authorization": "Bearer ${MCP_TOKEN}"
      }
    }
  ]
}
```

---

## Security & Authentication

### MCP Security Layer

```python
# mcp_security.py
from fastapi import Request, HTTPException
from typing import Dict, Any

class MCPSecurity:
    def __init__(self, config: Dict[str, Any]):
        self.config = config
        self.inherit_security = config.get("inherit_security", True)

    async def validate_request(self, request: Request) -> bool:
        # Inherit JWT security from upstream if configured
        if self.inherit_security:
            auth_header = request.headers.get("Authorization")
            if auth_header:
                # Delegate to shared JWT validation (not shown here)
                return True

        api_key = request.headers.get("X-API-Key")
        if api_key and api_key in self.config.get("allowed_api_keys", []):
            return True

        if self.config.get("require_auth", False):
            raise HTTPException(status_code=401, detail="Authentication required")

        return True
```

---

## Monitoring & Observability

### MCP Metrics (Prometheus)

```python
# mcp_metrics.py
from prometheus_client import Counter, Histogram, Gauge
import time

mcp_requests_total = Counter(
    "mcp_requests_total",
    "Total MCP requests",
    ["method", "tool", "status"],
)

mcp_request_duration = Histogram(
    "mcp_request_duration_seconds",
    "MCP request duration",
    ["method", "tool"],
)

mcp_active_connections = Gauge(
    "mcp_active_connections",
    "Active MCP connections",
    ["protocol"],
)

class MCPMetrics:
    def __init__(self):
        self.start_times: dict[str, float] = {}

    def record_request_start(self, request_id: str):
        self.start_times[request_id] = time.time()

    def record_request_end(self, request_id: str, method: str, tool: str | None, status: str = "success"):
        mcp_requests_total.labels(method=method, tool=tool or "none", status=status).inc()
        if request_id in self.start_times:
            duration = time.time() - self.start_times[request_id]
            mcp_request_duration.labels(method=method, tool=tool or "none").observe(duration)
            del self.start_times[request_id]

    def record_connection(self, protocol: str, delta: int):
        mcp_active_connections.labels(protocol=protocol).inc(delta)
```

---

## Use Cases

### AI Assistant Integration

- Claude Desktop browsing RAG-backed documentation.
- Cursor IDE surfacing code examples and API docs.
- Custom chatbots calling MCP tools for retrieval and QA.

### Developer Productivity

- Code completion with live documentation context.
- API reference search directly from the IDE.

### Research Assistants

- Literature review on top of RAG corpora.
- Fact checking and cross-reference validation.

---

## Best Practices

MCP implementation guidelines:

- Require authentication (JWT, API key, or mTLS).
- Apply rate limiting on MCP endpoints.
- Cache frequent queries when possible.
- Monitor tool usage and error rates.
- Version your MCP API and schemas.
- Document tool parameters and outputs clearly.
- Handle errors gracefully and return structured error objects.
- Support multiple protocols where appropriate (HTTP, SSE, etc.).
- Log all tool invocations for debugging and audit.

---

## Related Documentation

- RAG Service Documentation Part 1.
- RAG-Knowledge-Graph.
- RAG Query Expansion (if present in your docs set).
