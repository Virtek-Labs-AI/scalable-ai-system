# Hierarchical Context Architecture
## Reusable AI Skills Across Projects with Isolated Client Contexts

This document addresses one of the most practical challenges in scaling AI systems: **how do you make global best practices universally available to all agents, while keeping client/project-specific context completely isolated?**

This pattern is sometimes called **Hierarchical Context Management**, **Scoped Agent Memory**, or a **Multi-Tenant Skills Architecture**.

> **Stack note:** All implementation recommendations in this document reflect the definitive stack from [doc 08: Unified Platform Architecture](08-unified-platform-architecture.md). Where the original design referenced Mem0, Neo4j, or Zep Cloud, those have been replaced with PostgreSQL + pgvector and Graphiti on PostgreSQL. All interfaces — NanoClaw (Discord), Claude Code CLI, and REST API — share the same context layer via MCP servers.

---

## Table of Contents

1. [The Core Problem](#1-the-core-problem)
2. [Is This Being Done? (Industry Evidence)](#2-is-this-being-done-industry-evidence)
3. [The Four-Tier Context Model](#3-the-four-tier-context-model)
4. [Reference Architecture](#4-reference-architecture)
5. [Implementation: Skills Registry](#5-implementation-skills-registry)
6. [Implementation: Client Context Isolation](#6-implementation-client-context-isolation)
7. [Implementation: Agent Context Loading](#7-implementation-agent-context-loading)
8. [Preventing Context Contamination](#8-preventing-context-contamination)
9. [Scalability Improvements](#9-scalability-improvements)
10. [Productivity Patterns Beyond This System](#10-productivity-patterns-beyond-this-system)
11. [Recommended Stack for This Pattern](#11-recommended-stack-for-this-pattern)
12. [Quick Start Implementation](#12-quick-start-implementation)

---

## 1. The Core Problem

Imagine you have:
- **Global skills** you want every agent to always know: developer best practices, security standards, code review guidelines, communication templates
- **Domain skills** that apply in certain tech contexts: React conventions, Python idioms, DevOps patterns
- **Client A context**: their codebase, stack preferences, business rules, past decisions
- **Client B context**: completely different — their API contracts, naming conventions, deploy targets
- **Task context**: the specific thing being worked on right now

The challenge: An agent working on Client A's React app should know **React best practices (global) + Client A's context**, but must **never** see Client B's data. And you shouldn't have to copy your global best practices into every client project.

```
What we want:

  Agent on Client A task (whether via Discord or Claude Code CLI)
  ├── ✅ Global: Developer best practices
  ├── ✅ Domain: React conventions
  ├── ✅ Client A: Their stack, history, preferences
  └── ❌ Client B: Completely invisible

  Agent on Client B task
  ├── ✅ Global: Developer best practices
  ├── ✅ Domain: Python conventions
  ├── ✅ Client B: Their stack, history, preferences
  └── ❌ Client A: Completely invisible
```

---

## 2. Is This Being Done? (Industry Evidence)

**Yes — and it's one of the fastest-growing patterns in production AI.**

### What companies are doing:

**Anthropic (Claude's own system)**
Claude Code uses a `CLAUDE.md` file pattern — a markdown file in each project repo that injects project-specific context into every Claude session. Global preferences live in `~/.claude/CLAUDE.md`, project context in `./CLAUDE.md`. This is the same hierarchical override pattern described here, just in a file system form.

**GitHub Copilot Workspace**
Uses repository-level context files (`.github/copilot-instructions.md`) for project-specific instructions that all Copilot agents inherit. Microsoft extended this with enterprise-level instructions that propagate to all repos in an org — exactly the global → project hierarchy.

**Cursor IDE**
Implements `.cursorrules` (project-level) and global rules in settings. Rules compose: global + project-level rules are merged, with project rules taking precedence on conflicts.

**Agency Swarm (open source)**
Built the explicit concept of an "Agency Manifesto" — shared instructions visible to all agents in a crew — plus agent-specific instructions that only that agent sees. Direct implementation of this pattern.

**Letta (MemGPT)**
Introduced "shared memory blocks" — a memory object multiple agents can read/write simultaneously. Perfect for a global skills block shared across all agents, with separate per-client blocks only accessible to agents in that client's context.

**LangGraph Long-Term Memory Store**
Uses namespaced memory: `("skills", "global")`, `("client", "client-a")`, `("client", "client-b")` — agents query only the namespaces they have access to.

**Enterprise RAG systems (Elastic, Pinecone docs)**
Multi-tenant RAG is a documented pattern: one vector database, multiple isolated namespaces/collections per tenant, a shared "public" namespace all tenants can query.

---

## 3. The Four-Tier Context Model

```
┌────────────────────────────────────────────────────────────────────┐
│  TIER 1: GLOBAL SKILLS                    Scope: ALL agents        │
│  ─────────────────────────────────────────────────────────────     │
│  Developer best practices, security standards, code review         │
│  checklist, communication templates, ethical guidelines            │
│  Updated by: humans or optimizer agent                             │
│  Versioned: yes, with rollback                                     │
└───────────────────────────┬────────────────────────────────────────┘
                            │ inherited by
┌───────────────────────────▼────────────────────────────────────────┐
│  TIER 2: DOMAIN SKILLS                    Scope: Tagged agents     │
│  ─────────────────────────────────────────────────────────────     │
│  React conventions, Python idioms, DevOps patterns, API design,    │
│  database optimization, testing strategies, AI system design       │
│  Loaded: when agent's task matches domain tags                     │
└───────────────────────────┬────────────────────────────────────────┘
                            │ combined with
┌───────────────────────────▼────────────────────────────────────────┐
│  TIER 3: CLIENT / PROJECT CONTEXT         Scope: ISOLATED          │
│  ─────────────────────────────────────────────────────────────     │
│  Client A: tech stack, team preferences, past decisions,           │
│            approved libraries, deploy targets, naming conventions  │
│  Client B: entirely separate namespace — never mixed               │
│  Access control: PostgreSQL RLS enforces at database level         │
└───────────────────────────┬────────────────────────────────────────┘
                            │ added at runtime
┌───────────────────────────▼────────────────────────────────────────┐
│  TIER 4: TASK / SESSION CONTEXT           Scope: Ephemeral         │
│  ─────────────────────────────────────────────────────────────     │
│  Current conversation, active file contents, tool call results,    │
│  working memory for this specific task                             │
│  Lifetime: single session, never persisted to higher tiers         │
└────────────────────────────────────────────────────────────────────┘
```

### Inheritance & Override Rules

| Rule | Example |
|------|---------| 
| Lower tiers inherit upper tiers | Client A agents always see global skills |
| Lower tiers can override upper tiers | Client A can override a global naming convention |
| Sibling tiers are fully isolated | Client A and Client B never share context |
| Task context never promotes upward | A session conversation doesn't leak into client context |
| Safety skills always global-dominant | Security rules cannot be overridden by client or task context |

---

## 4. Reference Architecture

```
                    ┌──────────────────────────────────┐
                    │      CONTEXT REGISTRY            │
                    │  (PostgreSQL + Directus)         │
                    │                                  │
                    │  skills table (global + domain)  │
                    │  client_contexts table (RLS)     │
                    │  skill_overrides table           │
                    │  skill_versions table            │
                    └──────────────┬───────────────────┘
                                   │
                    ┌──────────────▼───────────────────┐
                    │    mcp-context MCP Server        │
                    │                                  │
                    │  Exposes context to ALL clients: │
                    │  • NanoClaw agents (Discord)     │
                    │  • Claude Code CLI sessions      │
                    │  • REST API callers              │
                    │                                  │
                    │  RLS enforces isolation:         │
                    │  session scope = client_id       │
                    └──────────────┬───────────────────┘
                                   │
           ┌───────────────────────┼───────────────────────┐
           │                       │                       │
┌──────────▼──────┐    ┌───────────▼──────┐    ┌──────────▼──────┐
│  NanoClaw Agent │    │  Claude Code CLI │    │  REST API Agent │
│  Client A task  │    │  Client A session│    │  Client A task  │
│                 │    │                  │    │                 │
│ Context:        │    │ Same context:    │    │ Same context:   │
│ ✅ Global       │    │ ✅ Global        │    │ ✅ Global       │
│ ✅ React domain │    │ ✅ React domain  │    │ ✅ React domain │
│ ✅ Client A     │    │ ✅ Client A      │    │ ✅ Client A     │
│ ❌ Client B     │    │ ❌ Client B      │    │ ❌ Client B     │
└─────────────────┘    └──────────────────┘    └─────────────────┘
```

The key insight: **the MCP server is the single source of truth for context**. Any interface that connects to it gets the same isolated, correct context for its scope. You can start a task on Discord and continue it in your local Claude Code CLI — the context is identical because it comes from the same store.

---

## 5. Implementation: Skills Registry

### Data Model

```sql
-- Global & Domain Skills
CREATE TABLE skills (
    id           UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    slug         VARCHAR(100) UNIQUE NOT NULL,
    name         VARCHAR(200) NOT NULL,
    tier         VARCHAR(20) NOT NULL CHECK (tier IN ('global', 'domain')),
    tags         TEXT[],          -- ['react', 'typescript', 'frontend']
    instructions TEXT NOT NULL,   -- The actual skill content
    version      VARCHAR(20) NOT NULL DEFAULT '1.0.0',
    eval_score   FLOAT,           -- Effectiveness score from evaluations
    is_active    BOOLEAN DEFAULT true,
    created_at   TIMESTAMPTZ DEFAULT NOW(),
    updated_at   TIMESTAMPTZ DEFAULT NOW()
);

-- Client / Project Contexts (isolated)
CREATE TABLE client_contexts (
    id           UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    client_id    VARCHAR(100) NOT NULL,
    project_id   VARCHAR(100),
    context_key  VARCHAR(100) NOT NULL,
    context_val  JSONB NOT NULL,
    created_at   TIMESTAMPTZ DEFAULT NOW(),
    updated_at   TIMESTAMPTZ DEFAULT NOW(),
    UNIQUE(client_id, project_id, context_key)
);

-- Per-client overrides of global/domain skills
CREATE TABLE skill_overrides (
    id           UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    client_id    VARCHAR(100) NOT NULL,
    skill_slug   VARCHAR(100) REFERENCES skills(slug),
    override_val TEXT NOT NULL,
    reason       TEXT,
    created_at   TIMESTAMPTZ DEFAULT NOW()
);

-- Full version history for every skill change
CREATE TABLE skill_versions (
    id           UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    skill_id     UUID REFERENCES skills(id),
    version      VARCHAR(20) NOT NULL,
    instructions TEXT NOT NULL,
    eval_score   FLOAT,
    promoted_by  VARCHAR(100),  -- 'human', 'optimizer-agent', 'rollback'
    created_at   TIMESTAMPTZ DEFAULT NOW()
);
```

### Example Global Skill (YAML representation)

```yaml
slug: developer-best-practices
name: Developer Best Practices
tier: global
tags: [all, development]
instructions: |
  ## Code Quality Standards
  - Write self-documenting code; add comments only for non-obvious logic
  - Functions should do one thing and do it well (Single Responsibility)
  - Write tests before fixing bugs (verify the bug, then fix, then verify fix)
  - Never commit secrets, API keys, or credentials to version control
  - All PRs require: passing CI, at least one reviewer approval, no TODO comments

  ## Security Baseline
  - Validate and sanitize all external inputs
  - Use parameterized queries — never string concatenation for SQL
  - Log security events; never log passwords or tokens
  - Dependencies must be pinned to exact versions in production

  ## Communication
  - Commit messages: imperative mood, < 72 chars, reference ticket ID
  - PR descriptions must include: what changed, why, how to test
  - Breaking changes require a migration guide in the PR
version: 2.1.0
```

---

## 6. Implementation: Client Context Isolation

### Populating Client Context

When onboarding a new client, populate their isolated namespace. Every agent working on that client — regardless of which interface they use — will receive this context automatically from the mcp-context server.

```python
async def setup_client_context(client_id: str, onboarding_data: dict):
    entries = [
        {
            "client_id": client_id,
            "context_key": "tech_stack",
            "context_val": {
                "frontend": onboarding_data["frontend"],
                "backend": onboarding_data["backend"],
                "database": onboarding_data["database"],
                "deployment": onboarding_data["deployment"],
            }
        },
        {
            "client_id": client_id,
            "context_key": "conventions",
            "context_val": {
                "branch_naming": "feature/TICKET-description",
                "approved_libraries": onboarding_data.get("approved_libraries", []),
                "forbidden_libraries": onboarding_data.get("forbidden_libraries", [])
            }
        }
    ]
    await db.bulk_insert("client_contexts", entries)
    # Now both NanoClaw agents AND Claude Code CLI sessions
    # will automatically receive this context for client_id
```

### Vector Store Isolation (Qdrant)

```python
# Per-client namespaces in Qdrant
# Every document tagged with tenant at write time

def search_client_knowledge(query_vector: list, client_id: str, include_global: bool = True):
    """
    Returns client-scoped results + optionally global skills.
    Client A can NEVER see Client B's results — enforced by payload filter.
    """
    filter_conditions = [
        FieldCondition(key="tenant", match=MatchValue(value=client_id))
    ]
    if include_global:
        filter_conditions.append(
            FieldCondition(key="tenant", match=MatchValue(value="global"))
        )
    return client.search(
        collection_name="knowledge_base",
        query_vector=query_vector,
        query_filter=Filter(should=filter_conditions),
        limit=10
    )
```

---

## 7. Implementation: Agent Context Loading

The context loader composes the full 4-tier context for any agent at runtime. It runs on every task start, regardless of which interface triggered the task.

```python
@dataclass
class AgentContext:
    global_skills: list[dict]
    domain_skills: list[dict]
    client_context: dict
    overrides: dict

    def to_system_prompt(self) -> str:
        """
        Compose all context tiers into a single system prompt section.
        Client overrides take precedence over global skills.
        """
        sections = ["## Organization Standards (applies to all work)"]

        for skill in self.global_skills:
            if skill["slug"] in self.overrides:
                sections.append(f"### {skill['name']} (customized for this client)")
                sections.append(self.overrides[skill["slug"]])
            else:
                sections.append(f"### {skill['name']}")
                sections.append(skill["instructions"])

        if self.domain_skills:
            sections.append("\n## Domain-Specific Standards")
            for skill in self.domain_skills:
                sections.append(f"### {skill['name']}")
                sections.append(skill["instructions"])

        if self.client_context:
            sections.append("\n## Client Context")
            for key, val in self.client_context.items():
                sections.append(f"**{key.replace('_', ' ').title()}:** {val}")

        return "\n".join(sections)


async def load_agent_context(
    agent_type: str,
    task_tags: list[str],
    client_id: Optional[str] = None
) -> AgentContext:
    """
    Central context loading function — called for every agent invocation,
    whether triggered by Discord, Claude Code CLI, or REST API.
    """
    # 1. Always load global skills
    global_skills = await db.query(
        "SELECT * FROM skills WHERE tier = 'global' AND is_active = true"
    )

    # 2. Load domain skills matching task tags
    domain_skills = await db.query(
        "SELECT * FROM skills WHERE tier = 'domain' AND is_active = true "
        "AND tags && $1",
        [task_tags]
    )

    # 3. Load client context (RLS ensures no cross-client access)
    client_context = {}
    overrides = {}
    if client_id:
        # SET app.current_client_id = client_id at connection time
        # PostgreSQL RLS blocks all cross-client access automatically
        rows = await db.query(
            "SELECT context_key, context_val FROM client_contexts WHERE client_id = $1",
            [client_id]
        )
        client_context = {row["context_key"]: row["context_val"] for row in rows}
        override_rows = await db.query(
            "SELECT skill_slug, override_val FROM skill_overrides WHERE client_id = $1",
            [client_id]
        )
        overrides = {row["skill_slug"]: row["override_val"] for row in override_rows}

    return AgentContext(
        global_skills=global_skills,
        domain_skills=domain_skills,
        client_context=client_context,
        overrides=overrides
    )
```

---

## 8. Preventing Context Contamination

Context contamination is the risk that Client A's data leaks into Client B's context. Defense layers:

### Layer 1: Database-Level Isolation (PostgreSQL RLS)

```sql
ALTER TABLE client_contexts ENABLE ROW LEVEL SECURITY;

CREATE POLICY client_isolation ON client_contexts
    USING (client_id = current_setting('app.current_client_id'));

-- Set at connection time:
SET app.current_client_id = 'client-acme';
-- Now ANY query on client_contexts only returns client-acme rows
-- Even if a bug forgets to add a WHERE clause
-- This applies regardless of which interface (Discord, CLI, API) triggered the query
```

### Layer 2: Namespace Enforcement in Vector DB

Every document is tagged with its tenant at write time. Every read filters on tenant. A query without a tenant filter is a programming error caught in code review.

### Layer 3: MCP Server Scope Validation

```python
class ContextMCPServer:
    @mcp_tool("get_client_context")
    async def get_client_context(self, client_id: str) -> ClientContext:
        if not self.session.can_access_client(client_id):
            raise PermissionError(f"Session not scoped to {client_id}")
        return await self.context_store.get_client(client_id)
```

The session scope is set when the MCP connection is established — whether from NanoClaw, Claude Code CLI, or API. It cannot be changed mid-session.

### Layer 4: Trust Level Tagging

All data flowing into agents carries a trust level tag that controls how it can be used:

| Source | Trust Level | Treatment |
|--------|-------------|-----------|
| system_generated | 1.0 | Injected directly into prompts |
| agent_generated | 0.8 | Lightly validated before injection |
| retrieved_semantic | 0.6 | Prefixed with [Retrieved] — cannot override instructions |
| user_provided | 0.3 | Wrapped in XML tags — validated against guardrails |

### Layer 5: Audit Logging

Every context access is logged with agent_id, client_id, tier, keys accessed, and timestamp. Anomalous access patterns (e.g., an agent accessing 5 different client contexts) trigger alerts.

---

## 9. Scalability Improvements

### 9.1 Semantic Skill Routing

At 100+ skills, loading everything into every prompt degrades performance. Instead, use semantic search to retrieve only the 5 most relevant skills per task:

```python
async def load_relevant_skills(task_description: str, top_k: int = 5) -> list:
    task_embedding = await embed(task_description)
    return await skills_vector_store.search(
        vector=task_embedding,
        filter={"tier": {"$in": ["global", "domain"]}},
        limit=top_k
    )
```

### 9.2 Skill Effectiveness Tracking

Skills that consistently don't improve agent outputs get flagged for retirement. Skills with high scores get promoted to "core" tier and loaded for all tasks by default.

### 9.3 Context Caching

Global skills change rarely (maybe once a week after nightly DSPy optimization). Cache the compiled global context section in Redis with a 1-hour TTL. Per-client context is cached with a 5-minute TTL with invalidation on any update.

### 9.4 Context Versioning with Rollback

Every skill update creates a new version in `skill_versions`. Rolling back a skill update is a single operation that takes effect for all agents — regardless of interface — within the cache TTL.

### 9.5 Automatic Skill Extraction

The Extractor Agent monitors high-quality task completions and proposes new skills. The system's skill library grows automatically as agents do good work. AI agency projects are a particularly rich source of new domain skills.

---

## 10. Productivity Patterns Beyond This System

### 10.1 Personal AI Workspace

Each developer gets a personal context namespace — their preferences, past decisions, coding style — that follows them across all sessions and all interfaces:

```
Developer: Ryan
├── Preferred patterns (functional over OOP, Zustand over Redux)
├── Past decisions ("chose FastAPI over Django for project X because...")
├── Expertise ("strong in systems design, working on ML ops")
└── Personal shortcuts ("always wants PR descriptions in bullet format")
```

This context is available whether Ryan is working via Discord, Claude Code CLI, or the REST API.

### 10.2 Automatic Knowledge Distillation

As agents work, the best patterns are automatically promoted through the skill extraction pipeline. AI agency projects compound this effect — each delivered AI system adds new domain skills to the library.

### 10.3 Context-Aware Tool Authorization

Different clients require different tool access. The mcp-context server enforces tool scoping so agents literally cannot call tools outside their allowed set for a given client.

### 10.4 Async Background Agents

Submit a long-running task from Claude Code CLI, close your terminal, check results from Discord later. The task persists in Celery, the LangGraph checkpoint survives restarts, and the result is waiting when you return.

### 10.5 Cross-Client Pattern Mining (Anonymous)

Patterns that repeat across multiple client projects are anonymized and promoted to global domain skills. Client-specific data never leaves the client namespace — only abstract patterns are elevated.

---

## 11. Recommended Stack for This Pattern

| Component | Recommended Tool | Why |
|-----------|-----------------|-----|
| **Skills Registry** | Directus + PostgreSQL | MCP-native, UI for editing, version tracking built in |
| **Client Context Store** | PostgreSQL + Row-Level Security | Database-enforced isolation — no app-layer bugs possible |
| **Episodic Memory** | PostgreSQL + pgvector | Same database, RLS applies automatically; replaces Mem0 |
| **Temporal Graph** | Graphiti on PostgreSQL | Open-source, runs on existing PG; replaces Neo4j + Zep Cloud |
| **Vector Search (global)** | Qdrant — `global` namespace | Fast, filtering, open source |
| **Vector Search (client)** | Qdrant — per-client namespace | Same infra, fully isolated by payload filter |
| **Context Loader** | FastAPI + MCP server | Single source of truth for context assembly; all interfaces connect here |
| **Agent Framework** | Claude Code SDK + LangGraph | Subscription flat-rate; state persistence; HITL checkpointing |
| **Agent Concurrency** | asyncio WorkerPool | I/O-bound LLM calls; asyncio is the right tool |
| **Caching** | Redis | Context cache for global skills (changes rarely) |
| **Admin Dashboard** | Directus UI + Appsmith | Edit skills, view client context, manage overrides |
| **Skill Optimization** | DSPy on Modal | Auto-improve skill prompts from evaluation data; serverless |
| **Effectiveness Eval** | DeepEval + RAGAS | Score which skills are actually working |
| **Interfaces** | NanoClaw (Discord), Claude Code CLI, REST API | All connect to the same mcp-context server |

### What Was Replaced and Why

| Old Tool | New Tool | Reason |
|----------|----------|--------|
| Mem0 | PostgreSQL + pgvector | Eliminates external dependency; RLS enforces isolation automatically |
| Neo4j | PostgreSQL (Graphiti) | Single database; no separate graph DB to operate |
| Zep Cloud | Graphiti on PostgreSQL | OSS alternative; same temporal graph capability |
| Ray Actors | asyncio WorkerPool | LLM calls are I/O-bound; asyncio handles this natively |
| NeMo Guardrails | Presidio + LLM Guard + Llama Guard 3 | NeMo designed for Llama; these work natively with Claude |

---

## 12. Quick Start Implementation

### Step 1: Deploy PostgreSQL with Skills Schema

```bash
docker compose up postgres directus redis qdrant
# Apply schema via Directus or direct SQL migration
```

### Step 2: Start the mcp-context Server

```bash
# The MCP server exposes context to all interfaces
uvicorn mcp_servers.context:app --port 8100
```

### Step 3: Configure Claude Code CLI

```bash
# Add to ~/.claude/mcp.json — your local CLI now shares context with NanoClaw
{
  "mcpServers": {
    "openclaw-context": {
      "url": "https://your-platform-host/mcp/context",
      "apiKey": "your-api-key"
    },
    "openclaw-memory": {
      "url": "https://your-platform-host/mcp/memory",
      "apiKey": "your-api-key"
    },
    "openclaw-skills": {
      "url": "https://your-platform-host/mcp/skills",
      "apiKey": "your-api-key"
    }
  }
}
```

### Step 4: Add Your First Global Skill

```python
# Via Directus UI or API
httpx.post("http://localhost:8055/items/skills", json={
    "slug": "developer-best-practices",
    "name": "Developer Best Practices",
    "tier": "global",
    "tags": ["all"],
    "instructions": "[Your organization's coding standards here]",
    "version": "1.0.0"
}, headers={"Authorization": f"Bearer {token}"})
```

### Step 5: Add a Client

```python
await setup_client_context("client-acme", {
    "frontend": "React 18 + TypeScript",
    "backend": "FastAPI + Python 3.12",
    "database": "PostgreSQL",
    "branch_naming": "feature/JIRA-description",
    "approved_libraries": ["react-query", "zustand", "zod"]
})
# Context is now available to NanoClaw agents AND Claude Code CLI sessions
```

---

*Last updated: April 2026 | [← System Design](06-system-design.md) | [Unified Architecture →](08-unified-platform-architecture.md) | [↑ Back to README](../README.md)*
