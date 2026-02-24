User types question in InsideAgent chat
        │
        ▼
[1] UI sends POST /chat/stream  { message: "Tell me about Apple", agent: "inside_agent" }
        │
        ▼
[2] FastAPI route receives request
    → ao.launch("insight_chain", { user_query: "Tell me about Apple" })
        │
        ▼
[3] AgentOrchestrator resolves the DAG from @produces/@consumes:
    check_coverage → search_content → generate_answer
        │
        ▼
[4] STEP 1: check_coverage
    → MCPConnector calls InsideAgent MCP server (auto-discovered tools)
    → MCP returns: { in_coverage: true, matched_entities: [{name: "Apple Inc", ...}] }
    → Writes to state: is_in_coverage=True, matched_entities=[...]
    → Middleware: CacheMiddleware caches this (5 min TTL for repeat queries)
    → Middleware: LoggerMiddleware logs step start/complete + duration
    → Middleware: MetricsMiddleware records step_duration_ms
        │
        ▼
[5] STEP 2: search_content
    → Reads from state: is_in_coverage, matched_entities (via @consumes)
    → MCPConnector calls InsideAgent MCP's search tool
    → MCP returns: { results: [{content: "Apple Q4 earnings...", source: "doc_123"}, ...] }
    → Writes to state: search_results, content_chunks
        │
        ▼
[6] STEP 3: generate_answer
    → Reads from state: content_chunks, is_in_coverage (via @consumes)
    → LLMGatewayClient calls Claude Sonnet with context (OAuth auto-managed)
    → NO system prompt needed — context is passed directly from step 2
    → LLM returns generated answer
    → Writes to state: generated_answer, citations
        │
        ▼
[7] Route handler reads result.context.generated_answer
    → Streams back as SSE chunks
    → UI renders progressively (same as before)
