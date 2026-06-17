---
name: mcp-server-development
description: MCP (Model Context Protocol) server development — invoke when building, debugging, or extending an MCP server with tools, resources, or prompts in any language
---

# MCP Server Development

## Architecture Overview

```
┌─────────────────────────────────────────────────────────────────┐
│  HOST  (Claude Desktop / Claude Code / IDE / Custom App)        │
│  • Manages server lifecycle                                     │
│  • Routes tool calls and resource reads                         │
│  • Presents tools to the model                                  │
└────────────────────────┬────────────────────────────────────────┘
                         │  MCP Protocol (JSON-RPC 2.0)
                         │
            ┌────────────┴────────────┐
            │         CLIENT          │
            │  (embedded in host)     │
            └────────────┬────────────┘
                         │  Transport (stdio / SSE / HTTP)
                         │
            ┌────────────┴────────────┐
            │         SERVER          │
            │  Your MCP server code   │
            └─────┬──────────┬────────┘
                  │          │
         ┌────────┘    ┌─────┘
         v             v
    ┌─────────┐   ┌──────────┐   ┌──────────┐
    │  Tools  │   │Resources │   │ Prompts  │
    │ actions │   │   data   │   │templates │
    └─────────┘   └──────────┘   └──────────┘
```

## Protocol Fundamentals

- **JSON-RPC 2.0**: all messages are JSON-RPC requests, responses, and notifications
- **Transports**:
  - `stdio`: server is a subprocess; host writes to stdin, reads from stdout; best for local tools
  - `SSE (HTTP)`: server listens on HTTP, client uses Server-Sent Events for streaming; supports multiple clients
  - `Streamable HTTP` (spec 2025-03-26): single bidirectional endpoint; replaces SSE in newer implementations
- **Initialization**: client sends `initialize` → server responds with capabilities → client sends `initialized` notification
- **Capabilities declared at init**: `tools`, `resources`, `prompts`, `logging`, `experimental`

---

## Server Capabilities

### Tools (Actions)

Tools let the model take actions — run code, call APIs, write files, query databases.

```
Tool lifecycle:
  1. tools/list  → client fetches available tools and their schemas
  2. tools/call  → model decides to call a tool; client sends request
  3. [your handler runs]
  4. Response returned to model (success or isError: true)
```

**Schema fields**: `name`, `description`, `inputSchema` (JSON Schema object)

### Resources (Data)

Resources expose data the model can read — files, database records, API responses.

```
Resource types:
  Static    → single URI; resource/read returns content
  Template  → URI template with {param} placeholders; listed via resources/list
              Example: "file:///logs/{date}/access.log"
```

**Schema fields**: `uri`, `name`, `description`, `mimeType`

### Prompts (Templates)

Reusable prompt templates the host can inject into conversations. Accept typed arguments.

```
Usage: host calls prompts/get with argument values → server returns rendered message list
```

---

## Python SDK

### Installation

```bash
pip install mcp
```

### Minimal Server

```python
import asyncio
from mcp.server import Server
from mcp.server.stdio import stdio_server
from mcp import types

server = Server("my-server")

@server.list_tools()
async def list_tools() -> list[types.Tool]:
    return [
        types.Tool(
            name="get_weather",
            description=(
                "Get current weather conditions for a city. "
                "Returns temperature, humidity, and conditions. "
                "Use when the user asks about weather, temperature, or conditions in a location."
            ),
            inputSchema={
                "type": "object",
                "properties": {
                    "city": {
                        "type": "string",
                        "description": "City name, optionally with country code (e.g., 'London' or 'London,UK')"
                    },
                    "units": {
                        "type": "string",
                        "enum": ["celsius", "fahrenheit"],
                        "description": "Temperature unit (default: celsius)",
                        "default": "celsius"
                    }
                },
                "required": ["city"]
            }
        )
    ]

@server.call_tool()
async def call_tool(name: str, arguments: dict) -> list[types.TextContent]:
    if name == "get_weather":
        city = arguments["city"]
        units = arguments.get("units", "celsius")

        try:
            weather_data = await fetch_weather(city, units)
            return [types.TextContent(
                type="text",
                text=f"Weather in {city}: {weather_data['temp']}°{'C' if units == 'celsius' else 'F'}, "
                     f"{weather_data['conditions']}, humidity {weather_data['humidity']}%"
            )]
        except WeatherAPIError as e:
            return [types.TextContent(
                type="text",
                text=f"Error fetching weather for {city}: {str(e)}"
            )]

    # Unknown tool
    return [types.TextContent(type="text", text=f"Unknown tool: {name}")]

async def main():
    async with stdio_server() as (read_stream, write_stream):
        await server.run(
            read_stream,
            write_stream,
            server.create_initialization_options()
        )

if __name__ == "__main__":
    asyncio.run(main())
```

### Resources in Python

```python
from mcp.server import Server
from mcp import types

server = Server("file-server")

@server.list_resources()
async def list_resources() -> list[types.Resource]:
    return [
        types.Resource(
            uri="file:///config/app.json",
            name="Application Config",
            description="Current application configuration",
            mimeType="application/json"
        ),
        # Template resource — listed with template URI
        types.Resource(
            uri="log:///access/{date}",
            name="Access Log",
            description="Access logs for a specific date (YYYY-MM-DD format)",
            mimeType="text/plain"
        )
    ]

@server.read_resource()
async def read_resource(uri: str) -> str:
    if uri == "file:///config/app.json":
        with open("/config/app.json") as f:
            return f.read()

    # Handle template URI
    if uri.startswith("log:///access/"):
        date = uri.split("/")[-1]
        # Validate to prevent path traversal
        if not re.match(r"^\d{4}-\d{2}-\d{2}$", date):
            raise ValueError(f"Invalid date format: {date}")
        log_path = f"/var/log/access-{date}.log"
        with open(log_path) as f:
            return f.read()

    raise ValueError(f"Unknown resource: {uri}")
```

### Prompts in Python

```python
@server.list_prompts()
async def list_prompts() -> list[types.Prompt]:
    return [
        types.Prompt(
            name="code_review",
            description="Review code for correctness, security, and style issues",
            arguments=[
                types.PromptArgument(
                    name="language",
                    description="Programming language (e.g., python, typescript, java)",
                    required=True
                ),
                types.PromptArgument(
                    name="focus",
                    description="Review focus: 'security', 'performance', 'correctness', or 'all'",
                    required=False
                )
            ]
        )
    ]

@server.get_prompt()
async def get_prompt(name: str, arguments: dict) -> types.GetPromptResult:
    if name == "code_review":
        language = arguments["language"]
        focus = arguments.get("focus", "all")

        return types.GetPromptResult(
            description=f"Code review prompt for {language}",
            messages=[
                types.PromptMessage(
                    role="user",
                    content=types.TextContent(
                        type="text",
                        text=f"Please review the following {language} code, focusing on {focus} issues. "
                             f"List each issue with its severity (critical/warning/info), location, "
                             f"and a concrete suggestion to fix it."
                    )
                )
            ]
        )

    raise ValueError(f"Unknown prompt: {name}")
```

---

## TypeScript SDK

### Installation

```bash
npm install @modelcontextprotocol/sdk zod
```

### Server with Zod Validation

```typescript
import { McpServer } from "@modelcontextprotocol/sdk/server/mcp.js";
import { StdioServerTransport } from "@modelcontextprotocol/sdk/server/stdio.js";
import { z } from "zod";

const server = new McpServer({
  name: "my-server",
  version: "1.0.0"
});

// Tool with Zod schema — automatically converted to JSON Schema
server.tool(
  "search_database",
  "Search the product database by keyword or SKU. Returns matching products with name, price, and stock level. Use when the user asks about product availability or wants to find products.",
  {
    query: z.string().describe("Search keyword or SKU number"),
    category: z.enum(["electronics", "clothing", "home", "all"]).default("all")
      .describe("Product category filter"),
    limit: z.number().int().min(1).max(20).default(5)
      .describe("Maximum number of results to return")
  },
  async ({ query, category, limit }) => {
    try {
      const results = await db.searchProducts({ query, category, limit });

      if (results.length === 0) {
        return {
          content: [{ type: "text", text: `No products found matching "${query}"` }]
        };
      }

      const formatted = results.map(p =>
        `${p.name} (SKU: ${p.sku}) — $${p.price} — ${p.stock > 0 ? `${p.stock} in stock` : "OUT OF STOCK"}`
      ).join("\n");

      return {
        content: [{ type: "text", text: `Found ${results.length} products:\n${formatted}` }]
      };
    } catch (error) {
      return {
        isError: true,
        content: [{ type: "text", text: `Database error: ${(error as Error).message}` }]
      };
    }
  }
);

// Resource
server.resource(
  "app-config",
  "config://app/settings",
  { mimeType: "application/json" },
  async (uri) => {
    const config = await loadConfig();
    return {
      contents: [{
        uri: uri.toString(),
        mimeType: "application/json",
        text: JSON.stringify(config, null, 2)
      }]
    };
  }
);

// Prompt
server.prompt(
  "summarize_document",
  "Summarize a document with configurable detail level",
  {
    detail_level: z.enum(["brief", "standard", "detailed"]).default("standard")
      .describe("How detailed the summary should be")
  },
  async ({ detail_level }) => ({
    messages: [{
      role: "user",
      content: {
        type: "text",
        text: `Please provide a ${detail_level} summary of the document. ` +
              (detail_level === "brief" ? "Use 2-3 sentences." :
               detail_level === "standard" ? "Use one paragraph per main section." :
               "Include all key points, sub-arguments, and data.")
      }
    }]
  })
);

// Start server
const transport = new StdioServerTransport();
await server.connect(transport);
```

---

## Java / Kotlin (Spring AI MCP)

### Dependency (Maven)

```xml
<dependency>
    <groupId>org.springframework.ai</groupId>
    <artifactId>spring-ai-mcp-server-spring-boot-starter</artifactId>
    <version>1.0.0</version>
</dependency>
```

### Kotlin Server

```kotlin
import org.springframework.ai.mcp.server.annotation.McpTool
import org.springframework.ai.mcp.server.annotation.McpResource
import org.springframework.ai.mcp.server.annotation.McpPrompt
import org.springframework.stereotype.Component
import org.springframework.ai.mcp.spec.McpSchema

@Component
class ProductMcpServer(private val productService: ProductService) {

    @McpTool(
        name = "search_products",
        description = "Search the product catalog by keyword. Returns matching products " +
                     "with name, SKU, price, and availability. Use when the user asks " +
                     "about products, pricing, or availability."
    )
    fun searchProducts(
        query: String,
        category: String = "all",
        limit: Int = 5
    ): String {
        return try {
            val products = productService.search(query, category, limit)
            if (products.isEmpty()) {
                "No products found matching '$query'"
            } else {
                products.joinToString("\n") { p ->
                    "${p.name} (${p.sku}) — \$${p.price} — ${if (p.inStock) "In stock" else "Out of stock"}"
                }
            }
        } catch (e: Exception) {
            "Error searching products: ${e.message}"
        }
    }

    @McpResource(
        uri = "catalog://products/categories",
        name = "Product Categories",
        description = "List of all available product categories",
        mimeType = "application/json"
    )
    fun getCategories(): String {
        return productService.getCategories().let { categories ->
            """{"categories": ${categories.map { "\"$it\"" }}}"""
        }
    }

    @McpPrompt(
        name = "product_recommendation",
        description = "Generate a product recommendation prompt based on user preferences"
    )
    fun productRecommendationPrompt(
        budget: String,
        use_case: String
    ): List<McpSchema.PromptMessage> {
        return listOf(
            McpSchema.PromptMessage(
                role = McpSchema.Role.USER,
                content = McpSchema.TextContent(
                    "Please recommend products for a customer with budget $${budget} " +
                    "for the use case: ${use_case}. Include product name, price, and " +
                    "why it fits their needs."
                )
            )
        )
    }
}
```

### Spring Boot Configuration

```yaml
# application.yml
spring:
  ai:
    mcp:
      server:
        name: product-catalog-server
        version: 1.0.0
        transport: STDIO   # or SSE for remote
        # For SSE transport:
        # sse-endpoint: /mcp/sse
        # message-endpoint: /mcp/messages
```

---

## Transport Implementations

### stdio (Local Tools)

```
Use when: tool runs on the same machine as the host (Claude Desktop, Claude Code)
Lifecycle: host spawns server as subprocess; manages start/stop
Auth: not needed (same machine, same user)
Clients: single host only
```

**claude_desktop_config.json**
```json
{
  "mcpServers": {
    "my-tool": {
      "command": "python",
      "args": ["/path/to/server.py"],
      "env": {
        "DATABASE_URL": "postgresql://localhost/mydb"
      }
    }
  }
}
```

### SSE (Remote / Multi-Client)

```
Use when: server is remote, multi-tenant, or shared across hosts
Lifecycle: server runs independently; clients connect over HTTP
Auth: required (OAuth 2.1 or API key)
Clients: multiple simultaneous clients supported
```

```python
# SSE server with FastAPI
from fastapi import FastAPI
from mcp.server.sse import SseServerTransport
from mcp.server import Server

app = FastAPI()
server = Server("remote-server")
sse_transport = SseServerTransport("/messages")

@app.get("/sse")
async def sse_endpoint(request):
    async with sse_transport.connect_sse(request.scope, request.receive, request._send) as streams:
        await server.run(*streams, server.create_initialization_options())

@app.post("/messages")
async def messages_endpoint(request):
    await sse_transport.handle_post_message(request.scope, request.receive, request._send)
```

### Authentication for Remote MCP (OAuth 2.1)

```typescript
import { McpServer } from "@modelcontextprotocol/sdk/server/mcp.js";
import { SSEServerTransport } from "@modelcontextprotocol/sdk/server/sse.js";

// The MCP spec (2025-03-26) defines OAuth 2.1 for remote server auth.
// Clients discover the authorization server via /.well-known/oauth-authorization-server

// Minimal API key auth (simpler for internal tools)
app.use("/mcp", (req, res, next) => {
  const apiKey = req.headers["x-api-key"];
  if (apiKey !== process.env.MCP_API_KEY) {
    res.status(401).json({ error: "Unauthorized" });
    return;
  }
  next();
});
```

---

## Error Handling

Never let unhandled exceptions crash the server — return structured errors instead.

```python
@server.call_tool()
async def call_tool(name: str, arguments: dict) -> list[types.TextContent]:
    try:
        result = await handle_tool(name, arguments)
        return [types.TextContent(type="text", text=result)]

    except ValueError as e:
        # Invalid input — return helpful error, not a crash
        return [types.TextContent(
            type="text",
            text=f"Invalid input: {str(e)}"
        )]

    except PermissionError as e:
        return [types.TextContent(
            type="text",
            text=f"Permission denied: {str(e)}"
        )]

    except Exception as e:
        # Log the full traceback server-side; return safe message to client
        logger.exception(f"Unexpected error in tool {name}")
        return [types.TextContent(
            type="text",
            text=f"An unexpected error occurred. Please try again or contact support."
        )]
```

**isError flag**: the MCP spec supports `isError: true` in tool responses. Use it when the tool call itself failed — the model can read the error message and decide how to proceed.

---

## Security

### Path Traversal Prevention

```python
import os
from pathlib import Path

ALLOWED_ROOT = Path("/data/user-files").resolve()

def safe_read_file(user_provided_path: str) -> str:
    # Resolve to absolute path, then verify it's inside allowed root
    requested = (ALLOWED_ROOT / user_provided_path).resolve()

    if not str(requested).startswith(str(ALLOWED_ROOT)):
        raise PermissionError(f"Access denied: path outside allowed directory")

    return requested.read_text()
```

### Input Validation

```python
import re

def validate_sql_identifier(name: str) -> str:
    """Prevent SQL injection in table/column names."""
    if not re.match(r'^[a-zA-Z_][a-zA-Z0-9_]*$', name):
        raise ValueError(f"Invalid identifier: {name!r}")
    return name

def validate_date(date_str: str) -> str:
    """Validate date format before using in file paths."""
    if not re.match(r'^\d{4}-\d{2}-\d{2}$', date_str):
        raise ValueError(f"Invalid date format (expected YYYY-MM-DD): {date_str!r}")
    return date_str
```

### Never Expose Credentials in Tool Responses

```python
# BAD — leaks internal details
return f"Connected to database at postgresql://user:password@internal-db:5432/prod"

# GOOD — surface only what the user needs
return f"Successfully connected to the database. Found {row_count} records."
```

### Rate Limiting

```python
import asyncio
from collections import defaultdict

class RateLimiter:
    def __init__(self, max_calls: int, window_seconds: int):
        self.max_calls = max_calls
        self.window = window_seconds
        self.calls: dict[str, list[float]] = defaultdict(list)

    async def check(self, client_id: str) -> None:
        now = asyncio.get_event_loop().time()
        window_start = now - self.window

        # Remove old calls
        self.calls[client_id] = [t for t in self.calls[client_id] if t > window_start]

        if len(self.calls[client_id]) >= self.max_calls:
            raise PermissionError(
                f"Rate limit exceeded: {self.max_calls} calls per {self.window}s"
            )

        self.calls[client_id].append(now)

limiter = RateLimiter(max_calls=60, window_seconds=60)

@server.call_tool()
async def call_tool(name: str, arguments: dict):
    await limiter.check("default")  # use actual client_id in multi-client setups
    ...
```

---

## Testing

### Using mcp-inspector

```bash
# Install the inspector
npm install -g @modelcontextprotocol/inspector

# Test a stdio server
mcp-inspector python server.py

# Test an SSE server
mcp-inspector --transport sse http://localhost:3000/sse
```

The inspector provides a web UI to:
- View all tools, resources, and prompts
- Call tools interactively and inspect input/output
- Read resources and render content
- Debug the JSON-RPC message stream

### Unit Testing Tool Handlers

```python
import pytest
import asyncio

# Test handlers directly — no MCP protocol needed
@pytest.mark.asyncio
async def test_search_products_returns_results():
    result = await handle_search_products({"query": "laptop", "category": "electronics"})
    assert len(result) > 0
    assert "laptop" in result[0].text.lower()

@pytest.mark.asyncio
async def test_search_products_handles_empty_results():
    result = await handle_search_products({"query": "xyzzy-nonexistent-12345"})
    assert "No products found" in result[0].text

@pytest.mark.asyncio
async def test_search_products_rejects_invalid_input():
    result = await call_tool("search_products", {"query": ""})
    # Should return error, not crash
    assert len(result) > 0
```

### Integration Testing

```python
from mcp.client.session import ClientSession
from mcp.client.stdio import StdioServerParameters, stdio_client

@pytest.mark.asyncio
async def test_server_integration():
    server_params = StdioServerParameters(
        command="python",
        args=["server.py"],
        env={"DATABASE_URL": "postgresql://localhost/test_db"}
    )

    async with stdio_client(server_params) as (read, write):
        async with ClientSession(read, write) as session:
            await session.initialize()

            # List tools
            tools = await session.list_tools()
            tool_names = [t.name for t in tools.tools]
            assert "search_products" in tool_names

            # Call tool
            result = await session.call_tool("search_products", {"query": "laptop"})
            assert result.content[0].text  # non-empty response
```

---

## Anti-Patterns

| Anti-Pattern | Problem | Fix |
|---|---|---|
| Tool does too many things | Model unsure when to call it; harder to test | One tool = one action; split by responsibility |
| Vague tool description | Model calls wrong tool or misses the right one | Describe what it does, when to use it, what it returns |
| No error handling | Unhandled exception crashes server; client gets protocol error | Wrap all handlers in try/except; return error text |
| Blocking I/O in async handlers | Blocks event loop; server stops responding | Use `asyncio.to_thread()` for blocking I/O |
| Path traversal not prevented | User can read arbitrary files | Always resolve and validate paths against allowed root |
| Credentials in tool responses | Leaks secrets to the model and conversation | Return only what the user needs to see |
| No input validation | Unexpected types or values cause crashes | Validate types, ranges, and formats before processing |
| Too many tools (15+) | Model selection accuracy degrades | Group into sub-agents with fewer, focused tools |
| Missing mimeType on resources | Host can't render content correctly | Always include mimeType in resource responses |
| Stateful server without cleanup | Memory leaks; stale connections | Implement proper cleanup in shutdown handlers |

---

## MCP Server Development Checklist

- [ ] Every tool has a clear `name` (verb_noun_context pattern)
- [ ] Every tool description answers: what it does, when to use it, what it returns
- [ ] All tool handlers have try/except with structured error responses
- [ ] Tool inputs are validated before processing (types, ranges, formats)
- [ ] Path operations validate against allowed root (no path traversal)
- [ ] No credentials, internal URLs, or sensitive data in tool responses
- [ ] Blocking I/O uses `asyncio.to_thread()` or equivalent
- [ ] Resources include correct `mimeType` in all responses
- [ ] Server tested with `mcp-inspector` manually
- [ ] Tool handlers unit-tested directly (without MCP protocol)
- [ ] Integration test covers initialize + list_tools + call_tool
- [ ] Rate limiting in place for remote/SSE servers
- [ ] Remote servers require authentication (OAuth 2.1 or API key)
- [ ] Server handles graceful shutdown (cleanup open connections)
- [ ] Tool count <= 10; if more, consider routing to sub-agents
