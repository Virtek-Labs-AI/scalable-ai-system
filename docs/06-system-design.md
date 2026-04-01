# End-to-End System Design

This document brings together all the research from the previous five sections into a concrete, actionable system design. It covers the recommended architecture, technology stack, implementation roadmap, and decision frameworks for building Virtek Labs' scalable autonomous AI system.

---

## Table of Contents

1. [System Vision & Goals](#1-system-vision--goals)
2. [Full Architecture Diagram](#2-full-architecture-diagram)
3. [Component Breakdown](#3-component-breakdown)
4. [Technology Stack by Maturity Level](#4-technology-stack-by-maturity-level)
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
2. **Scale horizontally:** Adding new projects, agents, or tasks should require zero changes to the core infrastructure.
3. **Improve autonomously:** The system monitors its own performance and iterates on prompts, skills, and agent configurations without requiring human intervention for every improvement.
4. **Stay safe:** Every agent operates within defined boundaries with layered guardrails, sandboxed execution, and human oversight available at all levels.
5. **Be observable:** Every agent action, decision, and output is traced, scored, and stored for analysis.

---

## 2. Full Architecture Diagram

```
┌───────────────────────────────────────────────────────────────────────┐
│                         EXTERNAL INTERFACE                              │
│   ┌─────────────┐  ┌─────────────┐  ┌─────────────┐             │
│   │  Dashboard  │  │  REST / WS   │  │  MCP Client  │             │
│   │  (Dify/     │  │  API (FastAPI)│  │  (IDE / Bot)│             │
│   │  Appsmith)  │  └─────────────┘  └─────────────┘             │
│   └─────────────┘                                                   │
└───────────────────────────────┬───────────────────────────────────────┘
                               │
                  ┌─────────▼─────────┐
                  │  PERIMETER LAYER      │
                  │  LiteLLM Proxy        │
                  │  Rate Limiting        │
                  │  Lakera Guard         │
                  └─────────┬─────────┘
                               │
┌─────────────────────────────▼───────────────────────────────────┐
│                    ORCHESTRATION LAYER                              │
│  ┌──────────────────────────────────────────────────────────────┐  │
│  │ SUPERVISOR AGENT (LangGraph)                                  │  │
│  │ • Receives tasks, plans decomposition                         │  │
│  │ • Routes to specialist agents                                 │  │
│  │ • Aggregates and synthesizes results                          │  │
│  └──────────────────────────────────────────────────────────────┘  │
│           │ Delegate via MCP / A2A Protocol                          │
│  ┌────────────▼─────────────┐  ┌───────────────────┐  ┌────────────┐  │
│  │ SPECIALIST AGENTS  │  │  OPTIMIZER AGENT     │  │ EVALUATOR │  │
│  │ (Ray Actors)       │  │  (DSPy / ADAS)       │  │ AGENT     │  │
│  │ • Research Agent   │  │  • Prompt optimizer   │  │ (RAGAS /  │  │
│  │ • Code Agent       │  │  • Skill evolver      │  │  DeepEval) │  │
│  │ • Analysis Agent   │  │  • Architecture search│  └────────────┘  │
│  └────────────────────┘  └───────────────────┘                    │
└────────────────────────────────┬───────────────────────────────────┘
                               │
┌─────────────────────────────▼───────────────────────────────────┐
│                    SHARED CONTEXT & MEMORY                          │
│  ┌─────────────────┐ ┌────────────────┐ ┌────────────────┐ │
│  │ SKILLS REGISTRY   │ │ EPISODIC MEMORY  │ │ KNOWLEDGE RAG   │ │
│  │ (Directus + MCP)  │ │ (Mem0 / Letta)   │ │ (Qdrant +        │ │
│  │ Skills, Contexts, │ │ Per-user/session │ │  GraphRAG)        │ │
│  │ Best Practices    │ │ memories         │ │ Domain knowledge │ │
│  └─────────────────┘ └────────────────┘ └────────────────┘ │
│  ┌──────────────────────────────────────────────────┐  │
│  │ TEMPORAL GRAPH (Graphiti / Zep)                            │  │
│  │ Evolving relationships, historical context, entity graph  │  │
│  └──────────────────────────────────────────────────┘  │
└────────────────────────────────┬───────────────────────────────────┘
                               │
┌─────────────────────────────▼───────────────────────────────────┐
│                    GUARDRAILS LAYER                                 │
│  Input: LLM Guard • NeMo Rails • Llama Guard                      │
│  Exec:  Sandbox (E2B) • HITL Gates • Circuit Breakers             │
│  Output: Guardrails AI • RAGAS Faithfulness • Hallucination Check  │
└────────────────────────────────┬───────────────────────────────────┘
                               │
┌─────────────────────────────▼───────────────────────────────────┐
│                    OBSERVABILITY LAYER                              │
│  Traces: Langfuse • AgentOps • OpenTelemetry Collector             │
│  Metrics: Prometheus • Grafana • LiteLLM Dashboards               │
│  Evals:  RAGAS nightly • DeepEval CI • Arize Phoenix drift        │
│  Alerts: Slack • PagerDuty • Grafana Alerting                     │
└─────────────────────────────────────────────────────────────────┘
```

---

## 3. Component Breakdown

### A. Supervisor Agent
- **Framework:** LangGraph (stateful graph with supervisor node)
- **Responsibilities:** Task planning, agent routing, result aggregation, final output synthesis
- **Model:** Claude Sonnet 3.5 or GPT-4o (high reasoning, low latency at reasonable cost)
- **State:** Persisted in PostgreSQL via LangGraph's checkpoint system

### B. Specialist Agents (Ray Actor Pool)
- **Framework:** Ray Actors (each agent is a stateful actor)
- **Types:** Research, Code, Analysis, Writing, Data, API Integration agents
- **Model:** Task-dependent — routed via LiteLLM (cheap tasks → Haiku/Flash, complex → Sonnet/GPT-4o)
- **Parallelism:** Ray autoscaler handles scaling specialist agents up/down based on queue depth

### C. Optimizer Agent
- **Framework:** DSPy for prompt optimization + custom ADAS-inspired architecture search
- **Trigger:** Runs nightly or when eval metrics drop below threshold
- **Output:** Updated skill prompts committed to the Skills Registry
- **Safety:** Optimizer changes go to staging environment first; promoted only after A/B eval shows improvement

### D. Evaluator Agent
- **Framework:** Custom LangGraph node + RAGAS + DeepEval
- **Trigger:** After every agent task completion
- **Metrics:** Faithfulness, relevance, goal achievement, latency, cost
- **Output:** Scores stored in Langfuse; anomalies trigger alerts

---

## 4. Technology Stack by Maturity Level

### Phase 1: Prototype (Weeks 1-4)

| Component | Technology | Why |
|-----------|------------|-----|
| Agent Framework | LangGraph | Stateful, production-grade, great docs |
| Memory | Mem0 cloud | Zero ops, instant setup |
| Vector DB | Chroma (local) | In-process, no server needed |
| LLM Routing | LiteLLM SDK | No proxy needed yet |
| Observability | Langfuse cloud | Free tier, zero setup |
| Guardrails | Guardrails AI | Simple output validation |
| Context UI | Directus cloud | Free tier, MCP ready |
| Deployment | Local / Docker Compose | Fast iteration |

### Phase 2: Production (Months 2-4)

| Component | Technology | Why |
|-----------|------------|-----|
| Agent Framework | LangGraph + Ray | Add parallelism |
| Memory | Mem0 self-hosted + Graphiti | Data control, graph memory |
| Vector DB | Qdrant self-hosted | Production performance |
| LLM Routing | LiteLLM Proxy | Budget enforcement, routing |
| Observability | Langfuse self-hosted + AgentOps | Full data control |
| Guardrails | NeMo + LLM Guard + Llama Guard | Layered defense |
| Context UI | Directus self-hosted + Appsmith | Custom admin panels |
| Deployment | Kubernetes (GKE/EKS) | Auto-scaling |
| CI/CD | GitHub Actions + DeepEval | Quality gates |

### Phase 3: Enterprise (Months 5+)

| Component | Technology | Why |
|-----------|------------|-----|
| Agent Framework | LangGraph + Ray + Message Queue | Full async, massive scale |
| Memory | Letta + Graphiti + pgvector | Tiered memory, SQL integration |
| Vector DB | Qdrant cluster | Horizontal scaling |
| LLM Routing | Kong AI Gateway + LiteLLM | Enterprise auth, compliance |
| Observability | Datadog LLM Observability | Unified enterprise observability |
| Guardrails | Full 5-layer stack | Defense in depth |
| Context UI | Custom React + Directus | Full custom admin experience |
| Deployment | Kubernetes + HPA + KEDA | Event-driven autoscaling |
| Self-Optimization | DSPy + ADAS + Nightly eval loops | Autonomous improvement |

---

## 5. Shared Context Registry Design

The Skills Registry is the core innovation that makes skills defined once available everywhere.

### Data Model

```sql
-- Skills Registry (PostgreSQL via Directus)
CREATE TABLE agent_skills (
    id          UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    slug        VARCHAR(100) UNIQUE NOT NULL,  -- e.g., 'security-code-review'
    name        VARCHAR(200) NOT NULL,
    description TEXT,
    version     VARCHAR(20) NOT NULL,           -- semver: '1.2.0'
    instructions TEXT NOT NULL,                 -- Full skill prompt/instructions
    tags        TEXT[],                          -- ['security', 'code-review']
    frameworks  TEXT[],                          -- ['python', 'any']
    metadata    JSONB,                           -- Arbitrary metadata
    is_active   BOOLEAN DEFAULT true,
    eval_score  FLOAT,                           -- Latest evaluation score
    created_at  TIMESTAMPTZ DEFAULT NOW(),
    updated_at  TIMESTAMPTZ DEFAULT NOW()
);

CREATE TABLE skill_versions (
    id          UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    skill_id    UUID REFERENCES agent_skills(id),
    version     VARCHAR(20) NOT NULL,
    instructions TEXT NOT NULL,
    eval_score  FLOAT,
    promoted_by VARCHAR(100),  -- 'human' or 'optimizer-agent'
    created_at  TIMESTAMPTZ DEFAULT NOW()
);

CREATE TABLE project_contexts (
    id          UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    project_id  VARCHAR(100) NOT NULL,
    context_key VARCHAR(100) NOT NULL,
    context_val JSONB NOT NULL,
    inherited_from UUID REFERENCES project_contexts(id),  -- Inheritance chain
    UNIQUE(project_id, context_key)
);
```

### How Skills Flow to Agents

```python
# 1. Agent starts up and loads relevant skills from registry
async def load_skills_for_agent(agent_type: str) -> list[Skill]:
    # Via Directus MCP server or REST API
    skills = await directus.get_items(
        collection="agent_skills",
        filter={"tags": {"_contains": agent_type}, "is_active": {"_eq": True}}
    )
    return skills

# 2. Skills are injected into the agent's system prompt
def build_system_prompt(base_prompt: str, skills: list[Skill]) -> str:
    skill_section = "\n\n## Available Skills\n"
    for skill in skills:
        skill_section += f"### {skill.name} (v{skill.version})\n"
        skill_section += f"{skill.instructions}\n\n"
    return base_prompt + skill_section

# 3. Any project can access skills by tag without per-project maintenance
code_agent_skills = await load_skills_for_agent("code-review")
# Returns: Security Review v1.2, Code Quality v2.0, Documentation v1.5
# These are shared across ALL projects automatically
```

### Context Inheritance Pattern

```
Global Context (organization-wide defaults)
    └── Project Context (project-specific overrides)
            └── Agent Context (agent-specific overrides)
                    └── Session Context (ephemeral, per-conversation)
```

Each level inherits from the level above and can override specific keys. This allows organization-wide best practices to propagate automatically while enabling project-specific customizations.

---

## 6. Parallel Agent Execution Design

### Ray Actor Pool Pattern

```python
import ray
from ray import serve

@ray.remote
class SpecialistAgent:
    def __init__(self, agent_type: str, skills: list):
        self.agent_type = agent_type
        self.skills = skills
        self.llm = load_litellm_client()
        
    async def execute_task(self, task: Task) -> TaskResult:
        # Load agent-specific skills from registry
        context = await load_skills_for_agent(self.agent_type)
        # Execute with ReAct loop
        result = await self.react_loop(task, context)
        # Store outcome in Mem0
        await mem0.add(result.summary, user_id=task.user_id)
        return result

# Create a pool of specialist agents
research_pool = [SpecialistAgent.remote("research", skills) for _ in range(10)]
code_pool = [SpecialistAgent.remote("code", skills) for _ in range(10)]

# Distribute tasks across the pool in parallel
futures = [agent.execute_task.remote(task) for agent, task in zip(research_pool, tasks)]
results = await asyncio.gather(*[f for f in futures])
```

### Task Queue Architecture

```
User Request
    │
    ▼
Supervisor Agent ───► Task Decomposition
    │                       │
    │              ┌────────▼────────┐
    │              │  Redis Streams    │  ←── Task Queue
    │              │  (Task Queue)     │
    │              └───┬───┬───┬───┘
    │                 │   │   │
    │              ┌──▼┐ ┌▼┐ ┌▼─┐
    │              │R│ │C│ │A │  ←── Specialist Agent Workers (Ray)
    │              └──┘ └─┘ └──┘
    │              Research Code Analysis
    │
    ▼
Result Aggregation ─► Final Response
```

---

## 7. Self-Optimization Loop Design

```
┌────────────────────────────────────────────────────────────────┐
│                   NIGHTLY OPTIMIZATION CYCLE                        │
│                                                                      │
│  1. COLLECT EVAL DATA                                                │
│     Langfuse → gather last 24h of traces with scores                │
│     Filter: skills with eval_score < 0.80 threshold                  │
│                          │                                          │
│  2. IDENTIFY IMPROVEMENT TARGETS                                     │
│     Optimizer Agent analyzes low-scoring skill/prompt pairs          │
│     Clusters failure modes from trace metadata                       │
│                          │                                          │
│  3. GENERATE CANDIDATES (DSPy)                                       │
│     DSPy MIPROv2 generates 5-10 improved prompt variants            │
│     Each variant is tested against evaluation dataset               │
│                          │                                          │
│  4. EVALUATE CANDIDATES                                              │
│     RAGAS scores each variant on faithfulness, relevance             │
│     DeepEval runs regression test suite                              │
│                          │                                          │
│  5. PROMOTE OR DISCARD                                               │
│     If best candidate > current score + 5%: promote to staging       │
│     If staging A/B test passes: update Skills Registry               │
│     Human notification sent (optional HITL approval gate)           │
│                          │                                          │
│  6. LOG & LEARN                                                      │
│     Store optimization history in skill_versions table              │
│     Reflexion: store “what worked” for optimizer agent memory       │
└────────────────────────────────────────────────────────────────┘
```

---

## 8. Dashboard & Admin Design

### Dashboard Components

```
┌────────────────────────────────────────────────────────────────┐
│  VIRTEK LABS AI PLATFORM — ADMIN DASHBOARD                          │
├────────────────────────────────────────────────────────────────┤
│  ┌────────────────┐ ┌────────────────┐ ┌────────────────┐  │
│  │ SKILLS MANAGER  │ │ AGENT MONITOR   │ │ COST TRACKER    │  │
│  │ (Directus UI)   │ │ (Grafana)       │ │ (Helicone /     │  │
│  │                 │ │                 │ │  LiteLLM)       │  │
│  │ • Browse skills  │ │ • Active agents  │ │ • Daily spend    │  │
│  │ • Edit contexts  │ │ • Task queue     │ │ • Per-project    │  │
│  │ • Version history│ │ • Error rates    │ │ • Budget alerts  │  │
│  │ • Tag management │ │ • Latency p95    │ │ • Model usage    │  │
│  └────────────────┘ └────────────────┘ └────────────────┘  │
│  ┌────────────────┐ ┌────────────────┐ ┌────────────────┐  │
│  │ TRACE EXPLORER  │ │ EVAL DASHBOARD  │ │ HITL APPROVALS  │  │
│  │ (Langfuse)      │ │ (Langfuse/RAGAS)│ │ (Appsmith)      │  │
│  │                 │ │                 │ │                 │  │
│  │ • Full traces   │ │ • Quality trends │ │ • Pending tasks  │  │
│  │ • Token usage   │ │ • Skill scores   │ │ • Approve/Reject │  │
│  │ • Debug view    │ │ • Model compare  │ │ • Action details │  │
│  │ • Prompt history │ │ • Regressions    │ │ • Audit log      │  │
│  └────────────────┘ └────────────────┘ └────────────────┘  │
└────────────────────────────────────────────────────────────────┘
```

---

## 9. Implementation Roadmap

### Sprint 1-2 (Weeks 1-4): Foundation
```
☐ Set up GitHub repository and CI/CD (GitHub Actions)
☐ Deploy LangGraph-based Supervisor Agent
☐ Implement 2-3 Specialist Agents (Research, Code)
☐ Set up Directus for Skills Registry with initial skills
☐ Connect Mem0 for agent memory
☐ Deploy LiteLLM Proxy for model routing
☐ Set up Langfuse for observability
☐ Add Guardrails AI for output validation
☐ First working end-to-end task: user → supervisor → agents → result
```

### Sprint 3-4 (Weeks 5-8): Parallelism & Memory
```
☐ Integrate Ray for parallel specialist agent execution
☐ Add Redis Streams task queue
☐ Deploy Qdrant for production vector search
☐ Implement Graphiti for temporal knowledge graph
☐ Build context inheritance system (global → project → agent → session)
☐ Add NeMo Guardrails for dialogue control
☐ Set up Grafana dashboards for system metrics
☐ Deploy AgentOps for agent session replay
```

### Sprint 5-6 (Weeks 9-12): Safety & Evaluation
```
☐ Implement full 5-layer guardrail stack
☐ Deploy Llama Guard 3 for content safety
☐ Set up RAGAS nightly evaluation pipeline
☐ Implement DeepEval CI/CD quality gates
☐ Add HITL approval gates for high-stakes actions
☐ Build sandboxed code execution (E2B integration)
☐ Run first Garak + PyRIT security assessment
☐ Document adversarial test results and mitigations
```

### Sprint 7-8 (Weeks 13-16): Self-Optimization & Dashboard
```
☐ Build Optimizer Agent using DSPy
☐ Implement nightly optimization loop
☐ Build Appsmith admin dashboard for Skills Manager + HITL
☐ Set up Helicone for cost tracking
☐ Implement context versioning with rollback
☐ A/B testing infrastructure for prompt/skill updates
☐ Build skill promotion workflow (staging → production)
☐ Team training on dashboard and context management
```

### Sprint 9-12 (Months 5-6): Scale & Harden
```
☐ Kubernetes deployment with HPA
☐ Multi-tenant isolation (project namespaces)
☐ Compliance audit (OWASP LLM Top 10 checklist)
☐ Disaster recovery and backup procedures
☐ Load testing at 10x expected volume
☐ Documentation for new team members
☐ Establish SLOs: p95 latency, uptime, eval score thresholds
```

---

## 10. Cost Estimation

### Monthly Infrastructure Costs (Production, ~1000 tasks/day)

| Component | Tool | Estimated Cost/Month |
|-----------|------|----------------------|
| LLM API calls | Mixed models via LiteLLM | $500-2000 |
| Vector Database | Qdrant Cloud (1M vectors) | $70 |
| Memory | Mem0 Pro | $50 |
| Observability | Langfuse cloud | $0-100 |
| Compute (agents) | GKE 4-node cluster | $400 |
| Context Registry | Directus cloud | $0-50 |
| Redis (queues) | Redis Cloud | $30 |
| **Total** | | **~$1,050-2,700/month** |

### Cost Optimization Strategies
1. **Semantic caching** (Portkey): reduce LLM calls by 20-40% for repeated queries
2. **Model routing** (LiteLLM): route 60% of tasks to cheap models (Haiku, Flash), reserve GPT-4o/Sonnet for complex ones
3. **Self-hosted stack**: Replace cloud services with self-hosted to cut recurring costs by 40-60%
4. **Batch processing**: Run non-urgent agent tasks in batch mode during off-peak hours at lower API cost

---

## 11. Key Decision Framework

### When to Add Human-in-the-Loop

| Risk Level | Action Type | Decision |
|------------|-------------|----------|
| Critical | Irreversible (delete data, deploy to prod, send external communications) | Always require human approval |
| High | Semi-reversible (write to production DB, external API writes) | Require approval for first N executions, then auto-approve if consistent |
| Medium | Easily reversible (draft creation, internal data writes) | Auto-execute with human notification |
| Low | Read-only, ephemeral | Auto-execute silently |

### When to Scale Out vs. Scale Up

| Situation | Strategy |
|-----------|----------|
| Many parallel independent tasks | Scale out (more agent workers, Ray pool) |
| Single complex multi-step task | Scale up (bigger context window, better model) |
| Repeated similar queries | Semantic caching (reduce scale entirely) |
| Unpredictable burst traffic | Serverless (Modal, AWS Lambda for agent execution) |

### When to Upgrade Memory Architecture

| Signal | Action |
|--------|--------|
| Context window limits hit regularly | Upgrade to Letta for infinite memory |
| Relationships between entities matter | Add Graphiti knowledge graph |
| Cross-session personalization needed | Add Mem0 for user-level memory |
| Domain knowledge frequently outdated | Implement agentic RAG with refresh triggers |

---

*Last updated: April 2026 | [← GitHub Resources](05-github-resources.md) | [↑ Back to README](../README.md)*
