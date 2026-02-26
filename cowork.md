What this server actually is
The open-claude-cowork backend is a plain Express.js server running on port 3001. It exposes these endpoints:

- POST /api/chat — takes a message, streams back SSE (Server-Sent Events) token by token
- POST /api/abort — cancels an in-flight request
- GET /api/providers — lists available AI providers (claude, opencode, etc.)
- GET /api/health — health check

It is not an MCP server. It doesn't speak the MCP protocol. The MCP connection lives inside the server — it calls out to Composio's MCP URL on your behalf. Regular CHAT UI cannot point at it as an MCP endpoint.


The handshake is simple REST + SSE. UI just needs to call POST /api/chat and listen to the streaming response. Here's what the full flow looks like:

Your UI subscribes to that SSE stream and renders chunks as they arrive. That's it — no MCP handshake needed on the UI side.


And to add S&P as a data connector, you add it to server/opencode.json alongside the existing Composio entry:
```
{
  "mcp": {
    "composio": {
      "type": "remote",
      "url": "<composio-mcp-url>",
      "headers": { ... }
    },
    "sp-global": {
      "type": "remote",
      "url": "https://kfinance.kensho.com/integrations/mcp",
      "enabled": true,
      "headers": {
        "Authorization": "Bearer YOUR_SP_API_KEY"
      }
    }
  }
}
```

That `opencode.json` is what the underlying Claude Code / Opencode provider reads to know which MCP tools are available. The provider then passes those tools to Claude when it calls `provider.query(...)`.

---

## The actual architecture end-to-end
```
 UI
  │  POST /api/chat  (REST + SSE)
  ↓
open-claude-cowork server (K8s, port 3001)
  │  Node.js + Express
  │  Loads .claude/skills/*.md as Skill tool context
  ↓
Claude Code / Opencode provider
  │  Calls Anthropic API with skills as system prompt
  │  Has access to MCP tools via opencode.json
  ├→ Composio MCP (500+ SaaS tools)
  └→ S&P Kensho MCP (financial data)
  ```

UI only needs to speak plain HTTP + SSE. There is no MCP handshake between the UI and the cowork server. MCP only lives between the cowork server and the data providers (S&P, Composio). You just point your UI's chat handler at POST /api/chat on the Kubernetes service URL and consume the SSE stream.

