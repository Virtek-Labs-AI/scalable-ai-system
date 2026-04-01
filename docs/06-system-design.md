# End-to-End System Design

This document establishes the system vision, architecture principles, and technology foundations for the OpenClaw platform. For the full authoritative architecture — including implementation roadmap, all 8 layers, MCP-first design, AI Agency Platform, and Claude Code CLI integration — see [doc 08: Unified Platform Architecture](08-unified-platform-architecture.md).

---

## Table of Contents

1. [System Vision & Goals](#1-system-vision--goals)
2. [Full Architecture Diagram](#2-full-architecture-diagram)
3. [Component Breakdown](#3-component-breakdown)
4. [Technology Stack](#4-technology-stack)
5. [Shared Context Registry Design](#5-shared-context-registry-design)
6. [Parallel Agent Execution Design](#6-parallel-agent-execution-design)
7. [Self-Optimization Loop Design](#7-self-optimization-loop-design)
8. [Dashboard & Admin Design](#8-dashboard--admin-design)
9. [Implementation Roadmap](#9-implementation-roadmap)
10. [Cost Estimation](#10-cost-estimation)
11. [Key Decision Framework](#11-key-decision-framework)

---

## 1. System Vision & Goals

The system being designed here is a **platform**, not just a single AI application. It is designed to:

1. **Grow skills once, apply everywhere:** A skill defined once (e.g., "Security Code Review") is immediately available to any agent in any project without per-project maintenance.
2. **Scale horizontally:** Adding new projects, agents, or tasks requires zero changes to the core infrastructure.
3. **Improve autonomously:** The system monitors its own performance and iterates on prompts, skills, and agent configurations without requiring human intervention for every improvement.
4. **Stay safe:** Every agent operates within defined boundaries with layered guardrails, sandboxed execution, and human oversight available at all levels.
5. **Be observable:** Every agent action, decision, and output is traced, scored, and stored for analysis. Observability is live from Week 1.
6. **Be interface-agnostic:** The same context and capabilities are available whether accessed via Discord (NanoClaw), the Claude Code CLI, or REST API.

---

## 2. Full Architecture Diagram

```
┌───────────────────────────────────────────────────────────────────────┐
│                         INTERFACE LAYER                                │
│   ┌─────────────┐  ┌─────────────┐  ┌──────────────────────────┐    │
│   │  NanoClaw   │  │  REST / WS   │  │  Claude Code CLI         │    │
│   │  (Discord)  │  │  API (FastAPI)│  │  (local dev access)     │    │
│   │  Task Bridge│  └─────────────┘  │  connects via MCP servers │    │
│   └─────────────┘                   └──────────────────────────┘    │
└───────────────────────────────┬───────────────────────────────────────┘
                               │
                  ┌─────────▼─────────┐
                  │   PERIMETER LAYER  │
                  │   LiteLLM Proxy    │
                  │   Rate Limiting    │
                  │   Budget Manager   │
                  └─────────┬─────────┘
                               │
┌─────────────────────────────▼───────────────────────────────────┐
│                    ORCHESTRATION LAYER                              │
│  ┌──────────────────────────────────────────────────────────────┐  │
│  │ SUPERVISOR AGENT (Claude Code SDK + LangGraph)               │  │
│  │ • Receives tasks, plans decomposition                        │  │
│  │ • Routes to specialist agents via asyncio WorkerPool         │  │
│  │ • Aggregates and synthesizes results                         │  │
│  └──────────────────────────────────────────────────────────────┘  │
│           │ via asyncio WorkerPool (I/O-bound; no Ray needed)        │
│  ┌────────▼─────────┐  ┌───────────────────┐  ┌───────────────┐  │
│  │ SPECIALIST AGENTS │  │  OPTIMIZER AGENT   │  │  EVALUATOR   │  │
│  │ Code / Research   │  │  (DSPy on Modal)   │  │  (DeepEval + │  │
│  │ Review / Architect│  │  Nightly via Prefect│  │   RAGAS)    │  │
│  └──────────────────┘  └───────────────────┘  └───────────────┘  │
└────────────────────────────────┬───────────────────────────────────┘
                               │
┌─────────────────────────────▼───────────────────────────────────┐
│               SHARED CONTEXT & MEMORY (MCP Servers)                │
│  ┌─────────────────┐ ┌────────────────┐ ┌────────────────┐ │
│  │  mcp-context     │ │   mcp-memory   │ │   mcp-skills   │ │
│  │  Skills Registry │ │  PostgreSQL +  │ │  Versioning +  │ │
│  │  Client Context  │ │  pgvector      │ │  Conflict Res. │ │
│  │  4-Tier Model    │ │  Graphiti      │ │                │ │
│  └─────────────────┘ └────────────────┘ └────────────────┘ │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │  Qdrant — per-client namespaced semantic vector search   │  │
│  └──────────────────────────────────────────────────────────┘  │
└────────────────────────────────┬───────────────────────────────────┘
                               │
┌─────────────────────────────▼───────────────────────────────────┐
│                    GUARDRAILS LAYER                                 │
│  Presidio (PII) → LLM Guard (injection) → Llama Guard 3 (safety) │
│  E2B Sandbox • HITL Gates • Circuit Breakers • Trust Level Tags  │
└────────────────────────────────┬───────────────────────────────────┘
                               │
┌─────────────────────────────▼───────────────────────────────────┐
│                    OBSERVABILITY LAYER (live from Week 1)          │
│  Langfuse (traces + evals) • Prometheus • Grafana                 │
│  Correlation IDs end-to-end • Per-client cost attribution         │
└─────────────────────────────────────────────────────────────────┘
```

---

## 3. Component Breakdown

### A. Supervisor Agent
- **Framework:** Claude Code Agent SDK + LangGraph (stateful graph with supervisor node)
- **Billing:** Flat-rate subscription (~$200/month/seat) — complex tasks have no per-token cost
- **Responsibilities:** Task planning, agent routing, result aggregation, final output synthesis
- **State:** Persisted in PostgreSQL via LangGraph's checkpoint system

### B. Specialist Agents (asyncio WorkerPool)
- **Framework:** asyncio WorkerPool — LLM calls are I/O-bound; asyncio handles concurrency with zero ops overhead
- **Types:** Research, Code, Analysis, Security Review, System Architect, AI Delivery agents
- **Model routing:** Complex tasks → Claude Code SDK (subscription); simple ops → Haiku API (per-token)
- **Why not Ray:** Ray adds operational complexity for a problem that asyncio solves natively at this scale

### C. Optimizer Agent
- **Framework:** DSPy MIPROv2 for prompt optimization
- **Compute:** Modal (serverless) — bursty, infrequent runs; only pay when it runs
- **Trigger:** Nightly via Prefect when skill eval scores trend below threshold
- **Output:** Updated skill prompts proposed via Discord #skill-proposals channel

### D. Evaluator Agent
- **Framework:** DeepEval + RAGAS
- **Trigger:** After every agent task completion (async, non-blocking)
- **Metrics:** Goal achievement, standard compliance, completeness, correctness, efficiency
- **Output:** Scores stored in Langfuse with correlation IDs; drives all three improvement loops

---

## 4. Technology Stack

### Definitive Stack (April 2026)

| Component | Technology | Why |
|-----------|------------|-----|
| Agent Framework | Claude Code SDK + LangGraph | Subscription flat-rate for complex work; stateful checkpointing |
| Agent Concurrency | asyncio WorkerPool | LLM calls are I/O-bound; asyncio handles this natively |
| Task Queue | Celery + Redis | Durable retry semantics, dead letter queue, mature ecosystem |
| Primary Database | PostgreSQL 16 | Tasks, skills, audit logs, LangGraph checkpoints, RLS isolation |
| Episodic Memory | PostgreSQL + pgvector | Same database, no external dependency; RLS auto-enforces isolation |
| Temporal Graph | Graphiti on PostgreSQL | Open-source; runs on existing PG instance; no Neo4j needed |
| Vector Search | Qdrant (self-hosted) | Per-client namespaces, production performance, open source |
| Skills Registry | Directus + PostgreSQL | MCP-native, UI for editing skills, version tracking |
| Observability | Langfuse + Prometheus + Grafana | Deployed Week 1; traces + metrics + dashboards |
| Guardrails | Presidio + LLM Guard + Llama Guard 3 | Layered defense; designed for Claude (unlike NeMo) |
| Code Sandbox | E2B | Ephemeral VM per execution; no host escape possible |
| Optimization | DSPy on Modal | Serverless compute; only pay when nightly job runs |
| Scheduling | Prefect | Nightly DSPy, memory consolidation, weekly skill mining |
| Admin UI | Directus + Appsmith | Skills management + HITL approval interface |

### Removed Components and Replacements

| Removed | Why | Replacement |
|---------|-----|-------------|
| Ray | Overkill for I/O-bound LLM concurrency; adds ops burden | asyncio WorkerPool |
| Mem0 | External dependency; RLS not automatic | PostgreSQL + pgvector |
| Neo4j | Second graph database; ops overhead | Graphiti on PostgreSQL |
| Zep Cloud | Vendor dependency; same capability available OSS | Graphiti on PostgreSQL |
| NeMo Guardrails | Designed for Llama; poor Claude compatibility | Presidio + LLM Guard + Llama Guard 3 |
| AgentOps | Redundant with Langfuse | Langfuse (covers traces + evals) |
| Redis Streams (primary) | Custom consumer code; fragile retries | Celery + Redis |

### LLM Cost Strategy: Subscription + API Hybrid

| Task Category | Model | Billing |
|---------------|-------|---------|
| Complex: planning, code gen, research, evaluation | Claude via SDK | Flat-rate subscription |
| Simple: parsing, classification, embedding | Claude Haiku API | Per-token (~$0.001/1K tokens) |
| DSPy optimization | Claude Opus via SDK | Flat-rate subscription |
| Embeddings | text-embedding-3-small | OpenAI API per-token |

---

## 5. Shared Context Registry Design

The Skills Registry is the core innovation that makes skills defined once available everywhere. See [doc 07](07-hierarchical-context-architecture.md) for the complete 4-tier context model and implementation details.

### Data Model (Key Tables)

```sql
-- Global & Domain Skills
CREATE TABLE skills (
    id           UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    slug         VARCHAR(100) UNIQUE NOT NULL,
    name         VARCHAR(200) NOT NULL,
    tier         VARCHAR(20) NOT NULL CHECK (tier IN ('global', 'domain')),
    tags         TEXT[],
    instructions TEXT NOT NULL,
    version      VARCHAR(20) NOT NULL DEFAULT '1.0.0',
    eval_score   FLOAT,
    is_active    BOOLEAN DEFAULT true,
    created_at   TIMESTAMPTZ DEFAULT NOW(),
    updated_at   TIMESTAMPTZ DEFAULT NOW()
);

-- Client / Project Contexts (isolated by RLS)
CREATE TABLE client_contexts (
    id           UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    client_id    VARCHAR(100) NOT NULL,
    project_id   VARCHAR(100),
    context_key  VARCHAR(100) NOT NULL,
    context_val  JSONB NOT NULL,
    UNIQUE(client_id, project_id, context_key)
);

-- Row-Level Security enforces isolation at database level
ALTER TABLE client_contexts ENABLE ROW LEVEL SECURITY;
CREATE POLICY client_isolation ON client_contexts
    USING (client_id = current_setting('app.current_client_id'));
```

### How Skills Flow to Agents

Skills are exposed via the `mcp-context` and `mcp-skills` MCP servers. Any interface — NanoClaw agents, Claude Code CLI sessions, or REST API callers — connects to the same servers and receives the same context. This is what makes the system interface-agnostic.

### Context Inheritance Pattern

```
Global Context (all agents, all interfaces)
    └── Domain Context (tagged agents matching task domain)
            └── Client Context (isolated per client, RLS-enforced)
                    └── Task Context (ephemeral, single session)
```

---

## 6. Parallel Agent Execution Design

### asyncio WorkerPool Pattern

The WorkerPool manages concurrency without the operational overhead of Ray. For I/O-bound LLM calls, asyncio's event loop is the correct primitive:

- **Concurrency limit:** Semaphore at 10 concurrent agent tasks (tunable per Claude Code seat count; ~3-5 sessions per seat)
- **Partial failure handling:** `asyncio.gather(return_exceptions=True)` — if one context source fails (circuit breaker tripped), the task continues with the available context
- **Structured outputs:** All agent results are typed Pydantic models (`AgentResult`, `AgentHandoff`) — no untyped dicts flowing between agents
- **Idempotency:** Idempotency keys prevent double-execution on Celery retries

### Task Queue Architecture

```
User Request (Discord / CLI / API)
    │
    ▼
Task Intelligence → Parsed + Budget-checked task
    │
    ▼
Celery + Redis Task Queue (durable, retry with backoff, DLQ)
    │
    ▼
Supervisor Agent (Claude Code SDK + LangGraph)
    │
    ▼
asyncio WorkerPool → Specialist Agents (concurrent)
    │
    ▼
Result Aggregation → Discord / CLI / API Response
```

---

## 7. Self-Optimization Loop Design

```
NIGHTLY OPTIMIZATION CYCLE (Prefect-scheduled, Modal compute)

1. COLLECT EVAL DATA
   Langfuse → gather last 24h traces with quality scores
   Filter: skills with eval_score < 0.80 or trending down over 7 days

2. IDENTIFY TARGETS
   Optimizer Agent analyzes low-scoring skill/prompt pairs
   Clusters failure modes from trace metadata

3. GENERATE CANDIDATES (DSPy MIPROv2 on Modal)
   Generates 5-10 improved prompt variants per underperforming skill
   Each variant tested against evaluation dataset

4. EVALUATE CANDIDATES (DeepEval + RAGAS)
   RAGAS scores each variant on faithfulness, relevance
   DeepEval runs regression test suite

5. PROMOTE OR DISCARD
   If best candidate > current score + 5%: stage as canary (5% traffic)
   Canary passes quality gates → 25% → 50% → 100%
   Human notification sent via Discord #skill-proposals
   Auto-rollback if metrics regress at any canary stage

6. LOG & LEARN
   Store optimization history in skill_versions table
   Successful patterns stored in pgvector for future reference
```

---

## 8. Dashboard & Admin Design

All dashboards are accessible from the start (deployed Week 1 alongside the first agent):

| Dashboard | Tool | What It Shows |
|-----------|------|---------------|
| **Trace Explorer** | Langfuse | Full token-level agent traces, prompt versions, quality scores |
| **System Metrics** | Grafana + Prometheus | Active agents, queue depth, error rates, latency p95 |
| **Cost Tracker** | Langfuse + LiteLLM | Per-client spend, model usage breakdown, budget status |
| **Skills Manager** | Directus UI | Browse/edit skills, version history, effectiveness scores |
| **HITL Approvals** | Discord + Appsmith | Pending decisions, approve/reject interface, audit log |
| **Memory Browser** | Custom (Appsmith) | Client context explorer, memory contradiction alerts |

---

## 9. Implementation Roadmap

See [doc 08: Unified Platform Architecture — Implementation Roadmap](08-unified-platform-architecture.md#21-implementation-roadmap) for the complete 6-phase, 18-week roadmap.

See [doc 10: Implementation Plan](10-implementation-plan.md) for hosting options, infrastructure sizing, and detailed cost analysis.

**Phase summary:**
- **Weeks 1-3:** Foundations — loop closes, observability live
- **Weeks 4-6:** Memory & parallelism — multi-client, isolated, accumulating
- **Weeks 7-9:** Safety & guardrails — production-grade trust
- **Weeks 10-13:** Self-improvement — system gets better autonomously
- **Weeks 14-16:** AI Agency Platform — design and deliver AI systems for clients
- **Weeks 17-18:** Scale & harden — Kubernetes, load testing, full autonomy

---

## 10. Cost Estimation

### Monthly Operational Costs

| Category | Tool | MVP (< 5 clients) | Growth (5-15 clients) |
|----------|------|-------------------|----------------------|
| LLM — complex tasks | Claude Code SDK (subscription) | $400/month (2 seats) | $600/month (3 seats) |
| LLM — simple ops | Haiku API | ~$20/month | ~$50/month |
| Embeddings | OpenAI API | ~$10/month | ~$25/month |
| Infrastructure | DigitalOcean (see doc 10) | ~$150-200/month | ~$350-500/month |
| Vector DB | Qdrant (self-hosted on DO) | included in infra | included in infra |
| Observability | Langfuse (self-hosted) | included in infra | included in infra |
| Serverless compute | Modal (DSPy nightly) | ~$5-15/month | ~$15-30/month |
| **Total** | | **~$585-645/month** | **~$1,040-1,205/month** |

> LLM API costs drop significantly relative to Claude Code subscription as task volume grows — the subscription absorbs more and more value at flat rate.

### Cost Optimization Strategies
1. **Subscription maximization:** Route every feasible complex task through Claude Code SDK — the more you use it, the better the $/task ratio
2. **Semantic caching (Redis):** Cache identical or near-identical task results — reduces API calls 20-40%
3. **Self-hosted stack:** All key services run self-hosted on DigitalOcean — no per-seat SaaS fees
4. **Haiku for all simple ops:** Classification, parsing, extraction never touch the expensive models

---

## 11. Key Decision Framework

### When to Add Human-in-the-Loop

| Risk Level | Action Type | Decision |
|------------|-------------|----------|
| Critical | Irreversible (delete data, deploy to prod, external client comms, AI system delivery) | Always require human approval |
| High | Semi-reversible (write to production DB, external API writes) | Require approval for first N, then auto with monitoring |
| Medium | Easily reversible (draft content, internal writes) | Auto-execute with human notification |
| Low | Read-only, ephemeral | Auto-execute silently |

### When to Add More Claude Code Seats

| Signal | Action |
|--------|--------|
| Queue depth consistently > 10 tasks | Add another Claude Code seat (+$200/month) |
| Agent session wait time > 30 seconds | Add another seat |
| Peak concurrency hitting semaphore limit | Add seat + increase WorkerPool limit |

### When to Upgrade Infrastructure

| Signal | Action |
|--------|--------|
| PostgreSQL query p95 > 100ms | Upgrade to next DO managed DB tier |
| Qdrant search p95 > 500ms | Add RAM to Qdrant droplet |
| Celery worker backlog growing | Add worker replicas (scale horizontally) |
| Redis memory > 70% | Upgrade Redis plan |

---

*Last updated: April 2026 | [← GitHub Resources](05-github-resources.md) | [Unified Architecture →](08-unified-platform-architecture.md) | [↑ Back to README](../README.md)*
