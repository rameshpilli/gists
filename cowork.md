What this server actually is
The open-claude-cowork backend is a plain Express.js server running on port 3001. It exposes these endpoints:

- POST /api/chat — takes a message, streams back SSE (Server-Sent Events) token by token
- POST /api/abort — cancels an in-flight request
- GET /api/providers — lists available AI providers (claude, opencode, etc.)
- GET /api/health — health check

It is not an MCP server. It doesn't speak the MCP protocol. The MCP connection lives inside the server — it calls out to Composio's MCP URL on your behalf. Regular CHAT UI cannot point at it as an MCP endpoint.
