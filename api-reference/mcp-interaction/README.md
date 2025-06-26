# MCP Server Interaction API

## Overview

The MCP (Model Context Protocol) Server Interaction API uses **JSON-RPC 2.0** over HTTP, not REST. All MCP communication goes through a single tenant-scoped endpoint using structured JSON-RPC requests and responses.

## Key Architecture Differences

### ❌ What We DON'T Use (Common Misconceptions)

- **REST endpoints** like `GET /tools` or `POST /tools/call`
- **Resource-based URLs** with different HTTP methods
- **HTTP status codes** for business logic (only for transport)
- **Query parameters** for method arguments

### ✅ What We DO Use (Actual Implementation)

- **Single endpoint** per tenant: `POST /{tenant}/mcp`
- **JSON-RPC 2.0** protocol with `method` field
- **Session-based** communication with initialization handshake
- **Tenant routing** like `/pipedrive/mcp`, `/salesforce/mcp`

## Protocol Flow

### 1. Initialize Session

```json
POST /pipedrive/mcp
{
  "jsonrpc": "2.0",
  "id": "init-123",
  "method": "initialize",
  "params": {
    "protocolVersion": "2025-03-26",
    "capabilities": {...},
    "clientInfo": {...}
  }
}
```

### 2. Send Initialized Notification

```json
POST /pipedrive/mcp
{
  "jsonrpc": "2.0",
  "method": "notifications/initialized",
  "params": {}
}
```

### 3. Use MCP Methods

```json
POST /pipedrive/mcp
{
  "jsonrpc": "2.0",
  "id": "tools-123",
  "method": "tools/list",
  "params": {}
}
```

## Available Methods

| Method           | Purpose                  | Returns               |
| ---------------- | ------------------------ | --------------------- |
| `initialize`     | Start MCP session        | Server capabilities   |
| `tools/list`     | List available tools     | Array of tools        |
| `tools/call`     | Execute a tool           | Tool execution result |
| `resources/list` | List available resources | Array of resources    |
| `prompts/list`   | List available prompts   | Array of prompts      |

## Request ID Usage

**The ID is NOT a resource identifier!** It's a JSON-RPC correlation ID:

```json
// Request
{
  "id": "my-unique-request-123",  // ← This matches the response
  "method": "tools/call",
  "params": {
    "name": "get_contacts"         // ← This is the actual tool name
  }
}

// Response
{
  "id": "my-unique-request-123",  // ← Same ID as request
  "result": {...}
}
```

## Authentication & Session Management

### Headers Required

```http
Authorization: Bearer your_oauth_token
Content-Type: application/json
Mcp-Session-Id: session_abc123  # After initialize
```

### Session Lifecycle

1. **Initialize** - Get session ID in response header
2. **Notify** - Send `notifications/initialized`
3. **Use** - Include `Mcp-Session-Id` in all subsequent requests

## Error Handling

JSON-RPC errors use standard error codes:

```json
{
  "jsonrpc": "2.0",
  "id": "request-123",
  "error": {
    "code": -32602,
    "message": "Invalid params",
    "data": {
      "details": "Tool 'nonexistent' not found"
    }
  }
}
```

Common codes:

- `-32602` - Invalid params
- `-32601` - Method not found
- `-32000` - Server error
- `-32603` - Internal error

## Tenant Routing

Each tenant has its own MCP endpoint:

- `/pipedrive/mcp` - Pipedrive CRM integration
- `/salesforce/mcp` - Salesforce integration
- `/hubspot/mcp` - HubSpot integration

## Tools vs Resources vs Prompts

### Tools (`tools/list`, `tools/call`)

- **Executable functions** that perform actions
- Take arguments, return results
- Examples: `get_contacts`, `create_deal`, `send_email`

### Resources (`resources/list`, `resources/read`)

- **Data objects** that can be read
- Have URIs like `crm://contacts/123`
- Examples: Contact records, documents, reports

### Prompts (`prompts/list`, `prompts/get`)

- **Templates** for AI interactions
- Have parameters for customization
- Examples: Email templates, analysis prompts

## Quick Start Example

```bash
# 1. Initialize session
curl -X POST "https://api.kambrium.com/pipedrive/mcp" \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "jsonrpc": "2.0",
    "id": "init-1",
    "method": "initialize",
    "params": {
      "protocolVersion": "2025-03-26",
      "capabilities": {"roots": {"listChanged": true}},
      "clientInfo": {"name": "my-client", "version": "1.0.0"}
    }
  }'

# 2. Send initialized notification (include Mcp-Session-Id from step 1)
curl -X POST "https://api.kambrium.com/pipedrive/mcp" \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -H "Mcp-Session-Id: $SESSION_ID" \
  -d '{
    "jsonrpc": "2.0",
    "method": "notifications/initialized",
    "params": {}
  }'

# 3. List tools
curl -X POST "https://api.kambrium.com/pipedrive/mcp" \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -H "Mcp-Session-Id: $SESSION_ID" \
  -d '{
    "jsonrpc": "2.0",
    "id": "tools-1",
    "method": "tools/list",
    "params": {}
  }'

# 4. Call a tool
curl -X POST "https://api.kambrium.com/pipedrive/mcp" \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -H "Mcp-Session-Id: $SESSION_ID" \
  -d '{
    "jsonrpc": "2.0",
    "id": "call-1",
    "method": "tools/call",
    "params": {
      "name": "get_contacts",
      "arguments": {"limit": 10}
    }
  }'
```

## Documentation Structure

- **[List Tools](./list-tools.mdx)** - Discover available tools
- **[Call Tool](./call-tool.mdx)** - Execute tools with parameters
- **[Get Resources](./get-resources.mdx)** - List available data resources
- **[Get Prompts](./get-prompts.mdx)** - List available prompt templates

All endpoints share the same `POST /{tenant}/mcp` URL but use different JSON-RPC methods.
