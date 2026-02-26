# Multi-Debate Agent — Project Overview

## What It Does

The Multi-Debate Agent is an AI-powered competitive analysis tool that takes in a set of **reference GitHub repositories** (e.g., open-source competitors) and one **target repository** (your own codebase), then runs a structured multi-agent debate to surface feature gaps — outputting a prioritized list of features the target repo should adopt or build.

The final output is a deduplicated, non-redundant list of actionable feature tickets (GitHub Issues) that get written directly to the target repo, with full awareness of what's already been planned or tracked.

---

## Core Concepts

### Inputs

| Input | Description |
|---|---|
| `reference_repos[]` | 2–4 public GitHub repo URLs (e.g., LangChain, LangGraph) |
| `target_repo` | Your private or internal GitHub repo |
| `feature_map.json` | A persisted JSON file tracking features already identified or planned |

**Auto-scaling rule:** If 2 reference repos are given, run 2 debating agents. If 3–4, spawn accordingly. Max 4 reference repos / 4 agents.

---

### The Agents

Each reference repo gets its own **Analyst Agent** that deeply understands that repo's features, architecture, and roadmap signals (from README, issues, changelogs, PR titles, and directory structure).

A separate **Target Repo Agent** does the same for your codebase.

A **Moderator/Judge Agent** orchestrates the debate, resolves conflicts, deduplicates, and produces the final output.

#### Debate Flow

```
[Analyst Agent 1 — LangChain]  ─┐
[Analyst Agent 2 — LangGraph]  ─┤─→ [Moderator Agent] ─→ [Target Repo Agent] ─→ Output
[Analyst Agent N — ...]        ─┘
```

1. **Round 1 — Feature Extraction:** Each Analyst Agent reads its assigned repo and produces a structured feature list with categories (e.g., memory, tool-use, streaming, observability).
2. **Round 2 — Gap Debate:** Agents cross-examine each other and the Target Repo Agent's feature list. They argue for which gaps are most critical or most feasible given the target repo's architecture.
3. **Round 3 — Synthesis:** The Moderator Agent consolidates debate outputs, scores features by relevance and effort signal, and removes any features already in `feature_map.json`.
4. **Final Output:** A clean, deduplicated list of new feature proposals formatted as GitHub Issues.

---

## Deduplication & feature_map.json

Every time the pipeline runs, it reads `feature_map.json` from the target repo (or a configurable path) before generating final output. This file tracks:

- Features already **implemented**
- Features already **planned / in backlog**
- Features **previously suggested** by this tool (with run timestamp)

The Moderator Agent uses this as a blocklist — any feature already present is excluded from the output. After generating new features, the tool appends them to `feature_map.json` so the next run stays clean.

### feature_map.json Schema

```json
{
  "last_updated": "2025-02-26T10:00:00Z",
  "features": [
    {
      "id": "feat-001",
      "title": "Streaming token callbacks",
      "status": "implemented",
      "source": "manual"
    },
    {
      "id": "feat-042",
      "title": "Async tool execution",
      "status": "planned",
      "source": "multi-debate-agent",
      "run_id": "run-2025-02-20",
      "github_issue": 87
    }
  ]
}
```

**Statuses:** `implemented` | `planned` | `in_progress` | `suggested` | `rejected`

---

## Output: GitHub Issues

Each accepted feature proposal is created as a GitHub Issue on the target repo with:

- **Title:** Feature name
- **Body:** Why it's relevant (which reference repos have it, what gap it fills)
- **Labels:** `multi-debate-agent`, `feature-request`, plus auto-detected category
- **Milestone** (optional): Current sprint or next release

---

## Tech Stack Decision

### Recommended: **Claude Agent SDK (Anthropic)**

Use the Claude Agent SDK as the primary framework. Here's why:

- The debate pattern (multiple agents with distinct system prompts + a moderator) maps cleanly to Claude's multi-turn and tool-use architecture
- GitHub repo reading can be handled via tool calls (fetch README, list files, read changelogs, search issues)
- The Moderator Agent can be implemented as a structured output agent with JSON schema enforcement for `feature_map.json`
- No added LangChain dependency overhead — simpler, more portable

**Use LangChain/LangGraph only if:** You need to plug into an existing LangGraph pipeline, or you need LangChain's built-in GitHub toolkit for repo traversal. In that case, use LangGraph for orchestration with Claude as the LLM backbone.

**Default recommendation:** Claude Agent SDK + GitHub REST API (via tool calls) + `feature_map.json` for state.

---

## System Architecture

```
┌─────────────────────────────────────────────────────────┐
│                      CLI / API Entry                     │
│  --refs repo1,repo2 --target myrepo --map feature_map   │
└───────────────────┬─────────────────────────────────────┘
                    │
          ┌─────────▼──────────┐
          │   Orchestrator      │  (reads feature_map.json, spawns agents)
          └──┬──────────┬──────┘
             │          │
    ┌────────▼──┐  ┌────▼────────────┐
    │ Analyst   │  │ Analyst Agent N │  (one per reference repo)
    │ Agent 1   │  └─────────────────┘
    └────────┬──┘
             │  (structured feature lists)
    ┌────────▼──────────────┐
    │   Target Repo Agent   │  (analyzes your codebase)
    └────────┬──────────────┘
             │
    ┌────────▼──────────────┐
    │   Moderator Agent     │  (debate, dedup, score, filter via feature_map)
    └────────┬──────────────┘
             │
    ┌────────▼──────────────┐
    │   GitHub Issue Writer │  (creates issues, updates feature_map.json)
    └───────────────────────┘
```

---

## File Structure (Proposed)

```
multi-debate-agent/
├── agents/
│   ├── analyst_agent.py        # Per-reference-repo agent
│   ├── target_agent.py         # Target repo analyzer
│   └── moderator_agent.py      # Debate orchestrator + dedup logic
├── tools/
│   ├── github_tools.py         # Fetch README, files, issues, PRs
│   └── feature_map_tools.py    # Read/write feature_map.json
├── output/
│   └── issue_writer.py         # GitHub Issue creation
├── config.py                   # API keys, repo URLs, settings
├── main.py                     # CLI entry point
├── feature_map.json            # Persisted feature state (lives in target repo or local)
└── README.md
```

---

## CLI Usage (Target Interface)

```bash
python main.py \
  --refs https://github.com/langchain-ai/langchain,https://github.com/langchain-ai/langgraph \
  --target https://github.com/myorg/myrepo \
  --map ./feature_map.json \
  --create-issues true \
  --max-features 10
```

---

## Open Questions / Decisions for Later

1. **Repo ingestion depth:** Should agents read only README + top-level structure, or go deeper into source files? (Deeper = more accurate, slower + more tokens)
2. **Scoring model:** How do we weight "gap importance"? By recency of feature in reference repos? By how many reference repos share it?
3. **LangGraph fallback:** If the target repo already uses LangChain, should we offer a LangGraph-native mode as a plugin?
4. **Authentication:** For private target repos, handle GitHub PAT or GitHub App auth.
5. **Feature categories:** Should we auto-categorize features (memory, routing, observability, etc.) using a taxonomy, or let the agents decide?
6. **Human-in-the-loop:** After the debate, should there be an approval step before issues are created, or is fully automated OK?

---

## Phased Build Plan

### Phase 1 — Core Pipeline (MVP)
- [ ] GitHub repo reader tool (README, file tree, changelog)
- [ ] Single Analyst Agent with structured output
- [ ] Target Repo Agent
- [ ] Basic Moderator (no debate, just merge + dedup)
- [ ] `feature_map.json` read/write
- [ ] Console output (no GitHub Issue creation yet)

### Phase 2 — Debate Layer
- [ ] Multi-agent debate rounds (Round 1 extraction → Round 2 cross-examination → Round 3 synthesis)
- [ ] Scoring/ranking of features by Moderator
- [ ] GitHub Issue creation

### Phase 3 — Polish
- [ ] Dynamic agent count based on number of repos
- [ ] Config file support
- [ ] Web UI or Slack integration (optional)
- [ ] LangGraph adapter (optional)
