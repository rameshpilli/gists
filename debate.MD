# Multi-Debate Agent: Project Overview

**Version:** 0.1 — Initial Spec  
**Status:** Pre-build  
**Purpose:** Feed to coding agent as authoritative project context

---

## 1. What This System Does

The Multi-Debate Agent is a multi-agent pipeline that takes 1–3 reference GitHub repositories and 1 target (your) repository, then produces a prioritized, deduplicated list of features your repo should adopt to stay competitive or complete. The agents debate with each other across structured rounds before a Moderator synthesizes a final recommendation list. Output is scoped to features that are both architecturally compatible with your target repo and not already present in your existing feature backlog (`feature_map.json`).

**Primary use case:** Engineers or teams working in isolated/corporate environments who want to keep pace with fast-moving open-source equivalents (e.g., your internal agent framework vs. LangGraph + CrewAI) without manually auditing large codebases.

---

## 2. Inputs

| Input | Type | Required | Notes |
|---|---|---|---|
| Reference repos | 1–3 GitHub URLs or local paths | Yes | The "competition" or reference projects |
| Target repo | 1 GitHub URL or local path | Yes | Your project — what we are improving |
| `feature_map.json` | JSON file | Optional but recommended | Existing feature backlog; prevents duplicate suggestions |
| User intent hint | Short string | Optional | e.g., "focus on streaming and observability" — steers debate priorities |

The system auto-scales agent count: 1 Analyst Agent per reference repo + 1 Target Repo Agent + 1 Moderator Agent.

---

## 3. Agent Roles

### 3.1 Analyst Agents (one per reference repo)
- Each agent is assigned exactly one reference repo.
- Responsible for producing a structured feature manifest from that repo.
- During the debate round, they argue *for* features from their repo being relevant to the target.

### 3.2 Target Repo Agent
- Deeply reads the target repo — architecture, patterns, dependencies, existing capabilities.
- Acts as the "defense": contextualizes whether a proposed feature is feasible or redundant.
- Produces the initial target feature manifest.
- Flags architectural mismatches (e.g., "that feature requires a Rust runtime, we are Python-only").

### 3.3 Moderator Agent
- Does **not** just tally votes. It must understand the target repo's architecture before scoring anything.
- Runs a pre-synthesis step: reads Target Repo Agent's architectural summary first.
- Scores features on: relevance to target architecture, estimated effort signal, community signal (stars/issues in reference repo), and gap criticality.
- Deduplicates against `feature_map.json` before final output.
- Produces the final structured output.

---

## 4. The Three Debate Rounds

### Round 1 — Feature Extraction (Parallel)
All Analyst Agents and the Target Repo Agent run in parallel. Each produces a structured feature manifest in this schema:

```json
{
  "repo": "langchain",
  "features": [
    {
      "id": "feat_001",
      "name": "Streaming token callbacks",
      "category": "streaming",
      "description": "Per-token streaming via callback hooks on chain execution",
      "evidence": ["langchain/callbacks/streaming_stdout.py", "docs/streaming.md"],
      "maturity": "stable"
    }
  ]
}
```

Categories to extract (non-exhaustive, agents may add their own):
`memory`, `tool-use`, `streaming`, `observability`, `multi-agent coordination`, `error handling / retries`, `persistence`, `human-in-the-loop`, `evaluation / testing`, `security`, `deployment`, `extensibility / plugins`

### Round 2 — Gap Debate (Sequential, structured)
- Target Repo Agent shares its feature manifest and a short architectural summary (tech stack, key patterns, constraints).
- Each Analyst Agent proposes its top N features as "gaps" against the target, with a brief argument for relevance.
- Target Repo Agent responds to each proposal: `COMPATIBLE`, `INCOMPATIBLE` (with reason), or `ALREADY_PRESENT`.
- Analyst Agents may rebut once. No infinite back-and-forth.
- All debate outputs are logged as structured JSON for the Moderator.

Debate message schema:
```json
{
  "from": "analyst_langchain",
  "round": 2,
  "proposals": [
    {
      "feature_id": "feat_001",
      "argument": "Target repo has no streaming support; LangChain shows this is expected baseline behavior",
      "priority_signal": "high"
    }
  ]
}
```

### Round 3 — Moderator Synthesis
The Moderator Agent:
1. Reads the Target Repo Agent's architectural summary (pre-step, before scoring anything).
2. Ingests all Round 2 debate outputs.
3. Loads `feature_map.json` if provided.
4. For each proposed feature, checks: (a) already in `feature_map.json`? → drop; (b) Target Repo Agent flagged INCOMPATIBLE? → drop or flag as "future/stretch"; (c) Compute a composite score.
5. Outputs final ranked list.

Moderator scoring rubric (0–10 per dimension):
- **Architectural fit** (0–10): Can this be added without restructuring the target? Informed by Target Repo Agent's compatibility call.
- **Gap criticality** (0–10): How much does the absence of this feature hurt the target's competitiveness?
- **Effort signal** (0–10, inverted): Lower effort → higher score. Moderator infers effort from codebase size of the reference implementation.
- **Reference signal** (0–10): Is this feature well-adopted across multiple reference repos?

**Final score** = weighted average. Default weights: fit 0.35, criticality 0.30, effort 0.20, reference 0.15. (Configurable.)

---

## 5. Final Output Schema

```json
{
  "run_id": "debate_20240915_001",
  "target_repo": "my-company/agent-framework",
  "reference_repos": ["langchain-ai/langchain", "langgenius/dify"],
  "generated_at": "2024-09-15T10:32:00Z",
  "features": [
    {
      "rank": 1,
      "name": "Streaming token callbacks",
      "category": "streaming",
      "score": 8.7,
      "score_breakdown": { "fit": 9, "criticality": 9, "effort": 7, "reference": 8 },
      "present_in_references": ["langchain", "dify"],
      "moderator_rationale": "Target repo uses Python async throughout; this is a natural fit. Absent in target and not in backlog. Both reference repos have mature implementations to draw from.",
      "suggested_ticket_title": "Add per-token streaming via async callback hooks",
      "effort_estimate": "medium",
      "status": "new"
    }
  ],
  "dropped_features": [
    {
      "name": "Rust-native vector store",
      "reason": "incompatible — target repo is Python-only",
      "dropped_by": "target_repo_agent"
    },
    {
      "name": "SQL memory backend",
      "reason": "already in feature_map.json (feat_backlog_042)",
      "dropped_by": "moderator"
    }
  ]
}
```

Each feature in the final list can optionally be converted to a GitHub issue or a ticket in the project's issue tracker.

---

## 6. Handling Large Repositories

This is the core engineering challenge. Large repos (LangChain, for example, is 500k+ lines) cannot be fed raw into a context window.

### Strategy: Hierarchical Repo Summarization (pre-processing pipeline)

**Step 1 — Structural Skeleton**  
Use the GitHub API (or local `git`) to fetch the file tree without content. Filter to keep: `README*`, `docs/`, `*.md`, top-level `*.py`/`*.ts`/`*.go` files, `setup.py`/`pyproject.toml`/`package.json`. Exclude: `tests/`, `examples/`, `__pycache__`, `node_modules`, generated files.

**Step 2 — Tiered Content Fetching**  
Tier 1 (always fetch): README, main docs index, `pyproject.toml` / `package.json`, top-level `__init__.py` or `index.ts`.  
Tier 2 (fetch if repo < 50k lines): All source files, chunked and summarized per module.  
Tier 3 (large repo mode, > 50k lines): Fetch only file names + first 50 lines per file. Run a "feature signal extractor" LLM pass on each chunk to produce a 3-sentence feature summary per module. Aggregate summaries before the main Analyst Agent runs.

**Step 3 — Semantic Feature Chunking**  
Group files by directory/module. For each module, ask a lightweight extraction model: "What is the primary capability this module provides?" Feed module summaries (not raw code) to the Analyst Agent.

**Step 4 — On-demand Deep Fetch**  
During Round 2 debate, if the Target Repo Agent challenges a feature claim, the Analyst Agent can request a deep fetch of the specific file(s) cited as evidence. This is a scoped, lazy retrieval — not a full re-read.

**Token budget per agent (recommended defaults):**
- Analyst Agent context budget: 80k tokens (summaries + targeted deep-fetches)
- Target Repo Agent: full read up to 100k tokens (your repo should be known deeply)
- Moderator: receives structured JSON only, not raw code — stays lean

---

## 7. Tech Stack Decision: Claude SDK vs. LangChain

**Recommendation: Anthropic Claude SDK with custom orchestration.**

| Criterion | Claude SDK | LangChain Agents |
|---|---|---|
| Control over debate structure | Full — you define the round logic | Partial — agent loop is abstracted |
| Debugging multi-agent turns | Easier — you own the call stack | Harder — framework magic obscures flow |
| Token/cost management | Explicit | Often implicit, can over-run |
| Structured output (JSON) | Native with `tool_use` + schemas | Possible but more boilerplate |
| Context window management | Direct | Managed by framework with less visibility |
| Dependency weight | Minimal | Heavy |
| Suitability for this specific flow | High — structured rounds map cleanly to sequential API calls | Medium — ReAct loop is not a natural fit for structured debate rounds |

LangChain is itself a reference repo here, not the tool we should build on. Building on it creates a conceptual conflict and a dependency on a fast-moving library. The Claude SDK gives you a clean separation between "what the agent thinks" and "how rounds are orchestrated."

Use `claude-3-5-sonnet` (or `claude-opus-4`) for Analyst and Moderator agents. Optionally use a smaller/faster model (e.g., `claude-haiku`) for the Tier 2/3 repo summarization pre-pass to save cost.

---

## 8. feature_map.json — Deduplication Contract

The Moderator Agent is the **only** agent that reads `feature_map.json`. It runs deduplication as the very last step before finalizing output, so it does not bias the debate.

`feature_map.json` expected schema (flexible — system adapts to what it finds):
```json
{
  "features": [
    {
      "id": "feat_042",
      "name": "SQL memory backend",
      "status": "planned | in_progress | done",
      "tags": ["memory", "persistence"]
    }
  ]
}
```

Deduplication uses semantic matching (embedding similarity), not exact string match, so "SQL memory backend" and "Add SQLite-based chat memory" are correctly identified as duplicates.

If `feature_map.json` is not provided, the system skips this step and notes it in the output header.

---

## 9. Project File Structure (Proposed)

```
multi-debate-agent/
├── main.py                     # Entry point, CLI
├── config.py                   # Weights, model choices, token budgets
├── agents/
│   ├── analyst_agent.py        # One instantiation per reference repo
│   ├── target_agent.py         # Target repo deep reader
│   └── moderator_agent.py      # Synthesis + scoring + dedup
├── pipeline/
│   ├── round1_extraction.py    # Parallel feature extraction
│   ├── round2_debate.py        # Structured debate loop
│   └── round3_synthesis.py     # Moderator consolidation
├── repo_processor/
│   ├── fetcher.py              # GitHub API + local git support
│   ├── summarizer.py           # Hierarchical summarization for large repos
│   └── deduplicator.py        # Semantic dedup against feature_map.json
├── schemas/
│   ├── feature_manifest.py     # Pydantic models for all structured outputs
│   └── debate_message.py
├── outputs/
│   └── [run_id]_results.json   # Final output per run
└── README.md
```

---

## 10. Key Open Questions / Design Decisions for Builder

1. **Ticket conversion:** Should the system auto-create GitHub Issues from the final feature list, or just output JSON? (Recommend: output JSON first, add GitHub integration as Phase 2.)
2. **Re-run behavior:** On a second run, should the system read the previous `[run_id]_results.json` to also deduplicate against prior suggestions, not just `feature_map.json`?
3. **User intent steering:** How much weight should a user-supplied focus hint (e.g., "prioritize streaming") shift the Moderator's scoring weights?
4. **Private repos:** For corporate use, the fetcher needs to support GitHub PAT or SSH auth. GitLab/Bitbucket support?
5. **Embedding model for dedup:** Use `text-embedding-3-small` (OpenAI) or Anthropic's own embeddings when available? Or a fully local model for air-gapped environments?
6. **Cost guardrail:** Add a dry-run mode that estimates token usage before actually running the debate?

---

## 11. MVP Scope (Phase 1)

Build this first, in order:

1. `repo_processor/fetcher.py` — Clone or GitHub API fetch, with large-repo summarization
2. `agents/analyst_agent.py` + `agents/target_agent.py` — Round 1 extraction only
3. `pipeline/round1_extraction.py` — Parallel extraction, structured JSON output
4. `agents/moderator_agent.py` — Basic synthesis, no dedup yet
5. End-to-end smoke test: 2 reference repos + 1 target → JSON output
6. Add `repo_processor/deduplicator.py` + `feature_map.json` integration
7. Add Round 2 debate loop
8. Polish scoring, ranking, output schema

**Phase 2:** GitHub Issue creation, re-run dedup, cost estimation, web UI.

---

---

## 12. Kubernetes Deployment Architecture

### 12.1 Design Philosophy

The system follows a **hybrid architecture**: the Orchestrator and Moderator run as persistent services, while Analyst Agents and the Target Repo Agent run as stateless, on-demand workers managed by a job queue. This avoids paying for always-on compute for agents that only run during active debate jobs.

A full debate run can take 3–10 minutes depending on repo size, so the Orchestrator accepts jobs asynchronously and returns a `run_id` immediately. Clients poll for status or receive a webhook callback on completion — no long-hanging HTTP connections.

### 12.2 Architecture Diagram

```
External Traffic
      │
      ▼
[ Ingress / API Gateway ]
      │
      ▼
[ Orchestrator Service ]  ←── single external endpoint (FastAPI)
  POST /run-debate
  GET  /run/{run_id}/status
  GET  /run/{run_id}/results
      │
      ├──► [ Redis Job Queue ]  ←── enqueues extraction + debate jobs
      │             │
      │    ┌─────────────────────┐
      │    ▼                     ▼
      │  [ Analyst Worker Pool ]  [ Target Repo Worker ]
      │    (KEDA-autoscaled)        (dedicated pod)
      │             │
      │    └────────┼─────────────┘
      │             ▼
      │    [ Moderator Worker ]
      │             │
      ▼             ▼
[ Shared Storage: PVC or S3-compatible ]
  - Repo clones + summaries (cached)
  - Feature manifests (Round 1 output)
  - Debate logs (Round 2 output)
  - Final results JSON
      │
      ▼
[ feature_map.json ]  ←── ConfigMap (stable) or fetched from git (dynamic)
```

### 12.3 Services and Pods

| Component | K8s Kind | Replicas | Notes |
|---|---|---|---|
| Orchestrator | Deployment | 2 (HA) | Only internet-facing service. Manages pipeline state, enqueues jobs, serves results. |
| Analyst Worker | Deployment + KEDA | 0–N (auto) | Scales on Redis queue depth. One worker handles one repo per round. |
| Target Repo Worker | Deployment | 1 | Dedicated pod; processes target repo once per run. |
| Moderator Worker | Deployment | 1 | Runs after all Round 1/2 jobs complete; triggered by Orchestrator. |
| Redis | StatefulSet | 1 (or managed) | Job queue + run state. Use Redis Sentinel or a managed service for HA. |
| Repo Cache | PVC or S3 | N/A | Shared storage for cloned/summarized repos across all workers. |

### 12.4 API Endpoints (Orchestrator Service)

```
POST /run-debate
  Body: {
    "reference_repos": ["https://github.com/org/repo1", ...],
    "target_repo": "https://github.com/your-org/your-repo",
    "feature_map_path": "optional — path or inline JSON",
    "intent_hint": "optional — e.g. focus on streaming and observability",
    "webhook_url": "optional — POST results here when complete"
  }
  Response: { "run_id": "debate_20240915_001", "status": "queued" }

GET /run/{run_id}/status
  Response: { "run_id": "...", "status": "queued|running|complete|failed", "current_round": 1, "progress": "3/3 analysts complete" }

GET /run/{run_id}/results
  Response: Final output JSON (see Section 5 schema)
```

### 12.5 Job Queue and Round Coordination

Agents do not call each other directly over HTTP. The Orchestrator acts as the conductor — it watches for round completion and gates the next round.

**Round flow:**
1. Orchestrator enqueues N Analyst jobs + 1 Target Repo job → Redis queue (Round 1 group).
2. Analyst Workers + Target Repo Worker pull from queue, run extraction, write manifests to shared storage, mark job complete in Redis.
3. Orchestrator detects all Round 1 jobs complete → enqueues Round 2 debate jobs.
4. Workers run debate, write structured debate logs to shared storage.
5. Orchestrator detects Round 2 complete → enqueues Moderator job.
6. Moderator reads all manifests + debate logs + `feature_map.json` → writes final results JSON → marks run complete.
7. Orchestrator serves results via GET endpoint or fires webhook.

This pattern is resilient: if a worker pod crashes mid-job, Redis job visibility timeout re-queues it automatically.

### 12.6 Repo Processing and Caching

Cloning large repos inside a pod on every run is expensive. The fetcher runs as a pre-job step and writes structured summaries (not raw code) to shared storage. If the same repo was summarized within the last 24 hours (configurable TTL), the cached summary is reused — the clone is skipped entirely.

```
Fetcher pre-job:
  1. Check cache: does /storage/cache/{repo_hash}/{date} exist?
  2. If yes → skip clone, load cached summaries
  3. If no  → clone repo to ephemeral volume
           → run hierarchical summarization (see Section 6)
           → write summaries to /storage/cache/{repo_hash}/{date}/
           → delete raw clone (free the ephemeral volume)
  4. Analyst Agent reads summaries only — never raw code
```

### 12.7 Autoscaling with KEDA

[KEDA (Kubernetes Event-Driven Autoscaling)](https://keda.sh) scales the Analyst Worker pool based on Redis queue depth. When no jobs are queued, workers scale to zero. When a run is submitted with 3 reference repos, KEDA spins up 3 workers in parallel.

```yaml
# keda-scaledobject.yaml (illustrative)
apiVersion: keda.sh/v1alpha1
kind: ScaledObject
metadata:
  name: analyst-worker-scaler
spec:
  scaleTargetRef:
    name: analyst-worker
  minReplicaCount: 0
  maxReplicaCount: 10
  triggers:
    - type: redis
      metadata:
        address: redis-service:6379
        listName: analyst-jobs
        listLength: "1"
```

### 12.8 Secrets and Configuration

| Secret/Config | K8s Resource | Notes |
|---|---|---|
| Claude API Key | `Secret` | Mounted as `ANTHROPIC_API_KEY` env var. Never baked into image. |
| GitHub PAT | `Secret` | Required for private repos. Scoped to read-only. |
| Redis connection string | `Secret` | |
| Model choices, scoring weights, token budgets | `ConfigMap` | Maps to `config.py` values. |
| `feature_map.json` (stable) | `ConfigMap` | Mount as file into Moderator pod. |
| `feature_map.json` (dynamic) | Fetched at job start | Pull from your git at run time if it changes frequently. |

### 12.9 Corporate / Air-Gapped Considerations

If the cluster has no egress to the Anthropic API, swap the LLM client for a self-hosted model. The agent architecture does not change — only the client in `config.py`.

Recommended self-hosted path: **Ollama** running as a sidecar or separate deployment, serving a capable open model (e.g., Llama 3.1 70B or Qwen2.5 72B). Point `ANTHROPIC_BASE_URL` or swap the client class to an OpenAI-compatible endpoint. Quality will be lower than Claude but the pipeline remains fully functional.

Egress requirements if using Anthropic API:
- `api.anthropic.com:443`
- `github.com:443` (or your internal GitHub Enterprise host)
- Your S3-compatible storage endpoint (if not using in-cluster PVC)

### 12.10 Proposed K8s File Structure

```
k8s/
├── orchestrator/
│   ├── deployment.yaml
│   ├── service.yaml
│   └── ingress.yaml
├── workers/
│   ├── analyst-worker-deployment.yaml
│   ├── target-repo-worker-deployment.yaml
│   └── moderator-worker-deployment.yaml
├── autoscaling/
│   └── keda-scaledobject.yaml
├── storage/
│   └── pvc.yaml
├── config/
│   ├── configmap-app.yaml        # model choices, weights, budgets
│   └── configmap-feature-map.yaml
├── secrets/
│   └── secrets-template.yaml     # never commit actual values
└── redis/
    └── redis-statefulset.yaml
```

### 12.11 Local Development (Docker Compose)

Engineers should be able to run the full system locally before touching K8s. A `docker-compose.yml` mirrors the K8s topology at small scale:

```yaml
# docker-compose.yml (illustrative)
services:
  orchestrator:
    build: .
    command: uvicorn main:app --reload
    ports: ["8000:8000"]
    environment:
      - ANTHROPIC_API_KEY=${ANTHROPIC_API_KEY}
      - REDIS_URL=redis://redis:6379
    volumes:
      - ./storage:/storage

  analyst-worker:
    build: .
    command: python workers/analyst_worker.py
    environment:
      - ANTHROPIC_API_KEY=${ANTHROPIC_API_KEY}
      - REDIS_URL=redis://redis:6379
    volumes:
      - ./storage:/storage

  moderator-worker:
    build: .
    command: python workers/moderator_worker.py
    environment:
      - ANTHROPIC_API_KEY=${ANTHROPIC_API_KEY}
      - REDIS_URL=redis://redis:6379
    volumes:
      - ./storage:/storage

  redis:
    image: redis:7-alpine
    ports: ["6379:6379"]
```

Run with `docker-compose up` for local end-to-end testing. The `storage/` volume replaces the PVC/S3 layer.

---

*This document is the source of truth for the initial build. All agent behaviors, schemas, and file names defined here should be treated as the contract.*
