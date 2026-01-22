# MS-365 MCP Server Architecture

## Overview
This is a **"Thin Server"** or **"Declarative API Bridge"** MCP implementation. It automatically generates MCP tools from Microsoft Graph API endpoints with minimal custom code.

## Architecture Pattern: Thin Server / API Bridge

**Yes, this is a standard MCP pattern**, especially for large APIs. There are two main approaches:

### 1. **Fat Server** (Custom Logic)
- Each tool has custom TypeScript implementation
- Example: `async function sendEmail(params) { /* custom logic */ }`
- **Pros**: Full control, can add complex business logic
- **Cons**: High maintenance (100+ tools = 100+ functions), hard to keep in sync with API changes

### 2. **Thin Server** (This Implementation)
- Tools defined declaratively in configuration (`endpoints.json`)
- Generic execution engine handles all tools
- **Pros**: Easy to maintain, scales to 100+ tools, self-documenting
- **Cons**: Limited custom logic per tool (must be LLM-driven or in generic middleware)

**This server uses Pattern #2** - a lightweight bridge between MCP protocol and Microsoft Graph API.

---

## System Flow

```
┌─────────────────────────────────────────────────────────────────┐
│ 1. BUILD TIME (npm run generate)                                │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌──────────────────────────────────────────────────────────────────┐
│ bin/generate-graph-client.mjs                                     │
│ ├─ Step 1: Download full Microsoft Graph OpenAPI spec            │
│ │          (https://aka.ms/graph/openapi/v1.0)                   │
│ │          → openapi/openapi.yaml (~30MB, 1000+ endpoints)       │
│ │                                                                 │
│ ├─ Step 2: Filter to only endpoints in endpoints.json            │
│ │          → openapi/openapi-trimmed.yaml (~500KB, 60 endpoints) │
│ │                                                                 │
│ └─ Step 3: Generate TypeScript client with Zod schemas           │
│            → src/generated/client.ts (Type-safe API client)      │
└──────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│ 2. RUNTIME (Server Startup)                                      │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌──────────────────────────────────────────────────────────────────┐
│ src/server.ts → registerGraphTools()                              │
│                                                                   │
│ For each endpoint in endpoints.json:                             │
│   1. Load OpenAPI-generated schema from src/generated/client.ts  │
│   2. Merge with custom config (llmTip, headers, scopes, etc.)    │
│   3. Register as MCP tool with auto-generated description        │
│                                                                   │
│ Result: 60+ MCP tools available to Claude                        │
└──────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│ 3. REQUEST HANDLING (Tool Invocation)                            │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌──────────────────────────────────────────────────────────────────┐
│ Claude calls tool: send-chat-message({ chat-id: "...", body: ...})│
└──────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌──────────────────────────────────────────────────────────────────┐
│ src/graph-tools.ts → executeGraphTool()                           │
│                                                                   │
│ Generic execution engine:                                         │
│   1. Validate params against Zod schema (from OpenAPI)           │
│   2. Build HTTP request:                                          │
│      - Path params: /chats/{chat-id}/messages                    │
│      - Query params: $filter, $select, $expand                   │
│      - Body: JSON payload                                        │
│      - Headers: Authorization, Content-Type, Custom (from config)│
│   3. Apply config overlays (llmTip, headers, returnDownloadUrl)  │
│   4. Handle pagination if fetchAllPages=true                     │
│   5. Call GraphClient.graphRequest()                             │
└──────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌──────────────────────────────────────────────────────────────────┐
│ src/graph-client.ts                                               │
│                                                                   │
│   1. Get OAuth access token (MSAL)                               │
│   2. Make HTTP request to Microsoft Graph API                    │
│      https://graph.microsoft.com/v1.0/chats/.../messages         │
│   3. Handle token refresh on 401                                 │
│   4. Return response as MCP content                              │
└──────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│ Response returned to Claude                                      │
└─────────────────────────────────────────────────────────────────┘
```

---

## Key Files & Their Roles

### Configuration Layer
| File | Purpose |
|------|---------|
| **`src/endpoints.json`** | **Master configuration** - defines which Graph API endpoints become MCP tools |
| `openapi/openapi.yaml` | Full Microsoft Graph OpenAPI spec (downloaded from Microsoft) |
| `openapi/openapi-trimmed.yaml` | Filtered spec (only endpoints in endpoints.json) |

### Code Generation Layer
| File | Purpose |
|------|---------|
| `bin/generate-graph-client.mjs` | Orchestrates the build process |
| `bin/modules/simplified-openapi.mjs` | Filters OpenAPI spec based on endpoints.json |
| `bin/modules/generate-mcp-tools.mjs` | Runs `openapi-zod-client` to generate TypeScript client |
| **`src/generated/client.ts`** | **Auto-generated** - Type-safe API client with Zod schemas |

### Runtime Layer
| File | Purpose |
|------|---------|
| `src/server.ts` | MCP server setup, registers tools |
| **`src/graph-tools.ts`** | **Generic execution engine** - handles all tool invocations |
| `src/graph-client.ts` | HTTP client for Microsoft Graph API |
| `src/auth.ts` | OAuth 2.0 / MSAL authentication |

---

## Data Flow: Adding a New Tool

Let's trace how **`send-chat-message`** works:

### Step 1: Define in `endpoints.json`
```json
{
  "pathPattern": "/chats/{chat-id}/messages",
  "method": "post",
  "toolName": "send-chat-message",
  "workScopes": ["ChatMessage.Send"],
  "llmTip": "CRITICAL: Body must be { \"body\": { \"contentType\": \"text\", \"content\": \"...\" } }"
}
```

### Step 2: Build generates schema (auto)
When you run `npm run generate`, the system:
1. Downloads Microsoft's OpenAPI spec
2. Extracts the `/chats/{chat-id}/messages` POST endpoint definition
3. Generates Zod schema for request/response validation

### Step 3: Runtime registration (auto)
```typescript
// In src/graph-tools.ts → registerGraphTools()
server.tool({
  name: "send-chat-message",
  description: "Send a new chatMessage in the specified chat...", // From OpenAPI
  inputSchema: {
    chat_id: { type: "string", description: "..." },
    body: { type: "object", properties: { ... } } // From Zod schema
  }
});
```

### Step 4: Execution (generic engine)
```typescript
// Claude calls the tool
executeGraphTool(tool, config, graphClient, {
  "chat-id": "19:abc123...",
  "body": { "body": { "contentType": "text", "content": "Hello!" } }
});

// Generic engine:
// 1. Validates body against Zod schema ✓
// 2. Builds URL: /v1.0/chats/19:abc123.../messages
// 3. Adds headers: Authorization, Content-Type
// 4. Makes POST request via graphClient
// 5. Returns response to Claude
```

---

## Why This Pattern?

### Traditional Approach (Fat Server)
```typescript
// 60+ custom functions
async function sendChatMessage(chatId: string, body: any) { ... }
async function listCalendarEvents(params: any) { ... }
async function getMeetingTranscript(meetingId: string, transcriptId: string) { ... }
// ... 57 more functions
```
**Problem**: High maintenance burden, hard to keep in sync with API changes.

### Thin Server Approach (This Implementation)
```json
// Just add to endpoints.json
{ "pathPattern": "/...", "method": "post", "toolName": "..." }
```
**Benefits**:
- **Scalability**: Adding 10 new tools = 10 JSON entries (not 10 TypeScript functions)
- **Maintainability**: OpenAPI spec is the source of truth
- **Self-documenting**: Tool descriptions come from Microsoft's API docs
- **Type Safety**: Zod schemas auto-generated from OpenAPI

---

## When to Use Each Pattern

### Use Thin Server When:
- ✅ API has good OpenAPI spec
- ✅ Need to support many endpoints (50+)
- ✅ API structure is predictable (REST, consistent patterns)
- ✅ Custom logic can be LLM-driven (via `llmTip`)

### Use Fat Server When:
- ❌ API lacks OpenAPI spec
- ❌ Complex business logic per tool (e.g., multi-step workflows)
- ❌ API requires session management (though you can hybrid: generic engine + session middleware)
- ❌ Need to transform responses significantly

### This Server's Hybrid Approach:
- **Thin** for tool definition (endpoints.json)
- **Smart overlays** for custom behavior:
  - `llmTip`: Guide LLM without code changes
  - `headers`: Add custom headers (like `Accept: text/vtt`)
  - `returnDownloadUrl`: Transform response behavior
  - `supportsTimezone`: Add dynamic parameters

---

## Common MCP Patterns

| Pattern | Example | Use Case |
|---------|---------|----------|
| **Thin Server** | This server | Large APIs (Google Workspace, Microsoft 365) |
| **Fat Server** | Custom tools with business logic | Specialized workflows, complex state |
| **Hybrid** | Thin + middleware for sessions | Excel (thin endpoints + session tracking) |
| **SDK Wrapper** | MCP wrapping Stripe SDK | When SDK exists but no OpenAPI |
| **Database Connector** | MCP for SQL queries | Direct data access |

---

## Adding Custom Behavior

### Option 1: LLM Tips (Preferred)
```json
{
  "toolName": "get-meeting-transcript-content",
  "llmTip": "CRITICAL: Must provide onlineMeeting-id (not Chat ID). Use list-online-meetings to resolve Join URL first."
}
```
**Pro**: No code changes, LLM adapts.  
**Con**: Depends on LLM following instructions.

### Option 2: Configuration Flags
```json
{
  "toolName": "get-meeting-transcript-content",
  "headers": { "Accept": "text/vtt" }
}
```
**Pro**: Deterministic, always applied.  
**Con**: Requires TypeScript support in execution engine.

### Option 3: Custom Middleware (Last Resort)
```typescript
if (config.toolName === 'special-case') {
  // Add custom logic here
}
```
**Pro**: Full control.  
**Con**: Breaks thin server pattern, creates tech debt.

---

## Summary

This MCP server is a **declarative API bridge** that:
1. **Sources truth** from Microsoft's OpenAPI spec
2. **Filters** to desired endpoints via `endpoints.json`
3. **Generates** type-safe client code automatically
4. **Executes** all tools through a generic engine
5. **Customizes** via configuration (not code)

This pattern is **ideal for large APIs** and is commonly used in production MCP servers for Google Workspace, Slack, GitHub, and other major platforms.
