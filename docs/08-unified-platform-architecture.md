# OpenClaw: Unified Platform Architecture
## Self-Improving Multi-Client AI Operating System

This is the authoritative architecture document for the OpenClaw platform. It consolidates the original autonomous agent platform design (formerly doc 08) and the expert-reviewed enhanced design (formerly doc 09) into a single cohesive reference. It also introduces the **AI Agency Platform** — the meta-use case where this system is used to design and deliver custom AI systems for clients.

> **Supersedes:** `08-autonomous-agent-platform.md` and `09-enhanced-system-design.md`

---

## Table of Contents

1. [What This System Is](#1-what-this-system-is)
2. [System Relationship: NanoClaw & OpenClaw](#2-system-relationship-nanoclaw--openclaw)
3. [Is Anyone Doing This? Industry Evidence](#3-is-anyone-doing-this-industry-evidence)
4. [End-to-End Walkthrough](#4-end-to-end-walkthrough)
5. [Full System Architecture](#5-full-system-architecture)
6. [Layer 1: Interface & Intake](#6-layer-1-interface--intake)
7. [Interface Unification: Three Ways to Reach the Platform](#7--interface-unification-three-ways-to-reach-the-platform)
8. [Layer 2: Task Intelligence & LLM Routing](#8-layer-2-task-intelligence--llm-routing)
9. [Layer 3: Orchestration & Execution](#9-layer-3-orchestration--execution)
10. [Layer 4: Hierarchical Context](#10-layer-4-hierarchical-context)
11. [Layer 5: Memory & Knowledge](#11-layer-5-memory--knowledge)
12. [Layer 6: Self-Improvement Engine](#12-layer-6-self-improvement-engine)
13. [Layer 7: Observability & Control](#13-layer-7-observability--control)
14. [Layer 8: Safety & Guardrails](#14-layer-8-safety--guardrails)
15. [AI Agency Platform: Building AI Systems for Clients](#15-ai-agency-platform-building-ai-systems-for-clients)
16. [MCP-First Architecture](#16-mcp-first-architecture)
17. [Complete Technology Stack](#17-complete-technology-stack)
18. [Data Flow: Request to Completion](#18-data-flow-request-to-completion)
19. [Self-Improvement Loop Detail](#19-self-improvement-loop-detail)
20. [Human Oversight Patterns](#20-human-oversight-patterns)
21. [Deployment Architecture](#21-deployment-architecture)
22. [Implementation Roadmap](#22-implementation-roadmap)
23. [What Makes This Different](#23-what-makes-this-different)

---

## 1. What This System Is

OpenClaw is a **Personal AI Operating System** — a backend platform that:

| Capability | What It Means |
|------------|---------------|
| **Conversational intake** | Describe tasks in Discord; no forms, no dashboards required |
| **Hierarchical context** | Automatically applies the right skills + client context per task |
| **Parallel autonomous execution** | Multiple specialist agents work simultaneously on complex tasks |
| **Strict multi-client isolation** | Client A's data is architecturally invisible to Client B's agents |
| **Self-improving** | The system gets better at your work patterns over time, without manual updates |
| **Human oversight** | You can monitor, pause, approve, reject, or redirect any task at any time |
| **Observable** | Full trace of every decision, tool call, and cost is available on demand |
| **AI agency capable** | Can design and deliver complete AI systems for clients using itself as the platform |

**What it is NOT:**
- Not a chatbot (it takes autonomous action, not just answers questions)
- Not a simple automation tool (it reasons, decides, and improves)
- Not a SaaS product (it's a platform you operate for your clients)
- Not a replacement for human judgment (it escalates when uncertain)

---

## 2. System Relationship: NanoClaw & OpenClaw

```
┌─────────────────────────────────────────────────────────┐
│                    USER / CLIENT                         │
│              (Discord messages, requests)                │
└────────────────────────┬────────────────────────────────┘
                         │
                         ▼
┌─────────────────────────────────────────────────────────┐
│                    NANOCLAW                              │
│           (Discord Bot — User Interface)                 │
│                                                         │
│  • Receives Discord messages                            │
│  • Simple queries → answers directly (Haiku, cheap)     │
│  • Complex tasks → delegates to OpenClaw                │
│  • Displays results back in Discord                     │
│  • HITL: posts approval requests, handles reactions     │
└────────────────────────┬────────────────────────────────┘
                         │ Task Bridge (HTTP/Queue)
                         ▼
┌─────────────────────────────────────────────────────────┐
│                    OPENCLAW                              │
│        (Autonomous Agent Platform — Backend)             │
│                                                         │
│  • Receives task from NanoClaw (or REST API)            │
│  • Routes to specialist agents                          │
│  • Manages multi-client context isolation               │
│  • Executes tools, writes code, calls APIs              │
│  • Self-improves from outcomes                          │
│  • Returns results to NanoClaw                          │
└─────────────────────────────────────────────────────────┘
```

### Why Two Systems?

| Concern | NanoClaw | OpenClaw |
|---------|----------|----------|
| **Purpose** | User interface layer | Execution engine |
| **Primary input** | Discord messages | Task API / internal |
| **LLM usage** | Haiku (fast, cheap) for routing | Sonnet/Opus for complex work |
| **State** | Discord session state | Persistent task state |
| **Scaling** | Scales with Discord guilds | Scales with task volume |
| **Upgrade path** | Can swap UI (Slack, web) without touching OpenClaw | Can upgrade execution without touching Discord layer |

**The Task Bridge** is the interface between them: NanoClaw classifies incoming messages, routes simple ones locally, and submits complex ones to OpenClaw's task queue with a client_id, correlation_id, and full context snapshot.

---

## 3. Is Anyone Doing This? Industry Evidence

Before investing in this architecture, it's worth asking: is this validated? The answer is yes — but at the enterprise scale, not the indie/startup scale we're building at.

### Companies Running This Pattern Today

| Company | What They're Doing | Source |
|---------|-------------------|--------|
| **Cognition (Devin)** | Autonomous software engineer; persistent context per project; HITL on architectural decisions | Public demos, 2024 |
| **Character.AI** | Per-user persistent memory across sessions; multi-agent coordination for complex personas | Engineering blog |
| **Salesforce Einstein** | Client-scoped agent context; multi-agent task routing; HITL approval flows | Dreamforce 2024 |
| **Microsoft Copilot Studio** | Tenant-isolated agents; skill/plugin registry; human escalation patterns | Microsoft docs |
| **Dust.tt** | Multi-agent workflows; hierarchical context (workspace → team → user); MCP-style tool registry | Public product |
| **LangChain/LangGraph** | Framework for exactly this pattern; hierarchical agent graphs; persistent memory | LangGraph docs |

### What's Different at Our Scale

The enterprise versions cost $10M+ to build. We're building the same pattern at 1/100th the cost because:

1. **MCP standardizes the tool layer** — no custom agent-tool contracts needed
2. **Claude Code SDK handles orchestration** — no custom agent runtime needed
3. **pgvector + Graphiti handles memory** — no custom memory store needed
4. **Existing infrastructure** — NanoClaw already works; OpenClaw extends it

**The insight:** The architecture is validated. The innovation is delivering it at small-team economics.

---

## 4. End-to-End Walkthrough

A client sends a message in Discord:

> "Can you analyze our Q3 marketing spend and tell me where we're wasting money?"

Here's exactly what happens:

```
1. DISCORD MESSAGE RECEIVED
   NanoClaw receives the message in #task-intake
   
2. COMPLEXITY CLASSIFICATION (NanoClaw, ~200ms)
   Task classifier: "This requires data analysis, judgment, and synthesis"
   → Classify as COMPLEX → route to OpenClaw
   NanoClaw posts: "⏳ On it! I'll follow up in this thread. (task-a3b7)"

3. TASK SUBMITTED TO OPENCLAW (NanoClaw → OpenClaw, ~100ms)
   POST /tasks {
     content: "Analyze Q3 marketing spend...",
     client_id: "acme-corp",
     interface: "nanoclaw",
     correlation_id: "discord-msg-id-1234"
   }

4. CONTEXT LOADING (OpenClaw, ~500ms)
   MCP Context Server loads for client_id=acme-corp:
   - Client profile: "B2B SaaS, 3 campaigns, Google Ads + LinkedIn"
   - Prior work: "Last analysis found LinkedIn CPC 3x higher than Google"
   - Active constraints: "Budget: $15k/mo, Focus: enterprise prospects"
   - Relevant skills: [marketing-analyst-v2, spend-optimizer-v1]

5. TASK PLANNING (OpenClaw Orchestrator, ~2s)
   Planner agent creates sub-tasks:
   a. Pull Q3 spend data (tool: sheets_reader)
   b. Pull Q3 conversion data (tool: crm_reader)  
   c. Calculate ROAS per channel (agent: math-analyst)
   d. Compare to Q2 baseline (agent: trend-analyst)
   e. Identify waste patterns (agent: marketing-expert)
   f. Write recommendations (agent: writer)

6. PARALLEL EXECUTION (OpenClaw Workers, ~45s)
   Sub-agents a, b run in parallel (data collection)
   Sub-agents c, d run after a, b complete (analysis)
   Sub-agent e synthesizes c+d
   Sub-agent f writes the final report

7. HITL CHECK (if needed)
   If any agent hits uncertainty threshold:
   → Post in Discord: "@ryan: Found anomaly in LinkedIn data — $4k charge with 
     no attributed conversions. Proceed with analysis or investigate first?"
   → Wait for reaction (✅ proceed / ❌ investigate)

8. RESULT RETURNED TO NANOCLAW
   OpenClaw → NanoClaw: {
     task_id: "task-a3b7",
     status: "completed",
     result: { report: "...", key_findings: [...], recommendations: [...] }
   }

9. DISCORD DELIVERY (NanoClaw)
   Posts in thread: 
   "✅ Analysis complete for Acme Q3:
    
   **Key findings:**
   • LinkedIn is underperforming: $2,847 spent, 0.3% CTR vs Google's 1.8%
   • Remarketing campaign has $1,200 with zero conversions this quarter
   • Top performer: Google Brand keywords, 4.2x ROAS
   
   **Recommendation:** Pause LinkedIn and remarketing (~$4k/mo savings).
   Full report → [link]"

10. MEMORY UPDATED (async, post-delivery)
    Writes to mcp-memory:
    - "Acme Q3 analysis: LinkedIn underperforming, see task-a3b7"
    - "Pattern: B2B clients often over-invest in LinkedIn vs Google"
    - Quality score: calculated from outcome feedback
```

Total time: ~60 seconds for a task that would take a human analyst 2-4 hours.

---

## 5. Full System Architecture

```
┌─────────────────────────────────────────────────────────────────────────┐
│                         EXTERNAL INTERFACES                              │
│                                                                         │
│   Discord (NanoClaw) ──────┐                                           │
│   Claude Code CLI ─────────┤──→ Task Bridge / REST API                 │
│   REST API (webhooks) ─────┘                                           │
└──────────────────────────────────┬──────────────────────────────────────┘
                                   │
┌──────────────────────────────────▼──────────────────────────────────────┐
│                      LAYER 1: INTERFACE & INTAKE                         │
│                                                                         │
│   Task Classifier ──→ Complexity Router ──→ Client ID Resolver          │
│   Context Injector ──→ Correlation ID Generator ──→ Queue Submitter     │
└──────────────────────────────────┬──────────────────────────────────────┘
                                   │
┌──────────────────────────────────▼──────────────────────────────────────┐
│                   LAYER 2: TASK INTELLIGENCE & LLM ROUTING               │
│                                                                         │
│   Task Planner (Sonnet) ──→ Sub-task Generator ──→ Agent Selector       │
│   LLM Router: Haiku/Sonnet/Opus by complexity + cost optimization       │
└──────────────────────────────────┬──────────────────────────────────────┘
                                   │
┌──────────────────────────────────▼──────────────────────────────────────┐
│                   LAYER 3: ORCHESTRATION & EXECUTION                     │
│                                                                         │
│   Claude Code SDK ──→ Specialist Agents (parallel) ──→ Tool Executor    │
│   Celery Task Queue ──→ Redis ──→ Worker Pool                           │
│   HITL Manager ──→ Approval Gate ──→ Resume Handler                    │
└──────────────────────────────────┬──────────────────────────────────────┘
                                   │
┌──────────────────────────────────▼──────────────────────────────────────┐
│                    LAYER 4: HIERARCHICAL CONTEXT (MCP)                   │
│                                                                         │
│   mcp-context ──→ Global / Agency / Client / Project / Session layers   │
│   mcp-memory ──→ Graphiti knowledge graph + pgvector semantic search    │
│   mcp-skills ──→ Skill registry + versioned prompt library              │
│   mcp-github ──→ Code context, PR history, repo knowledge               │
└──────────────────────────────────┬──────────────────────────────────────┘
                                   │
┌──────────────────────────────────▼──────────────────────────────────────┐
│                      LAYER 5: MEMORY & KNOWLEDGE                         │
│                                                                         │
│   PostgreSQL + pgvector ──→ Semantic search across all stored knowledge  │
│   Graphiti ──→ Temporal knowledge graph (who said what, when, trust)    │
│   Redis ──→ Hot cache for active session context                        │
└──────────────────────────────────┬──────────────────────────────────────┘
                                   │
┌──────────────────────────────────▼──────────────────────────────────────┐
│                    LAYER 6: SELF-IMPROVEMENT ENGINE                      │
│                                                                         │
│   Outcome Tracker ──→ Quality Scorer ──→ Pattern Extractor              │
│   Skill Proposer ──→ Human Review ──→ Skill Deployer                    │
│   A/B Testing ──→ Prompt Optimizer ──→ Cost Analyzer                   │
└──────────────────────────────────┬──────────────────────────────────────┘
                                   │
┌──────────────────────────────────▼──────────────────────────────────────┐
│                   LAYER 7: OBSERVABILITY & CONTROL                       │
│                                                                         │
│   Langfuse ──→ Full trace of every LLM call, tool use, decision         │
│   Prometheus + Grafana ──→ System metrics, cost per client              │
│   Sentry ──→ Error tracking with full context                           │
└──────────────────────────────────┬──────────────────────────────────────┘
                                   │
┌──────────────────────────────────▼──────────────────────────────────────┐
│                      LAYER 8: SAFETY & GUARDRAILS                        │
│                                                                         │
│   Autonomy Level Config ──→ Action Classifier ──→ HITL Gate             │
│   Budget Limiter ──→ Rate Limiter ──→ Client Isolation Enforcer         │
│   Prompt Injection Detector ──→ Output Validator                        │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## 6. Layer 1: Interface & Intake

### NanoClaw Discord Bot

The Discord bot (NanoClaw) remains the primary human interaction surface. The Task Bridge component routes complex work to OpenClaw.

```python
# discord_bot.py — NanoClaw with Task Bridge integration
import discord
from discord.ext import commands

bot = commands.Bot(command_prefix="/")
bridge = OpenClawBridge(openclaw_url=OPENCLAW_API_URL)

@bot.event
async def on_message(message):
    if message.author.bot:
        return
    if bot.user.mentioned_in(message) or is_task_channel(message.channel):
        task_id = await bridge.handle_message(message, get_group_config(message))
        if task_id:
            thread = await message.create_thread(name=f"Task: {await get_short_title(message.content)}")
            await thread.send(f"⏳ Working on it... (task `{task_id}`)")

@bot.slash_command(name="status")
async def status(ctx, task_id: str = None):
    if task_id:
        info = await bridge.get_task_status(task_id)
        await ctx.respond(format_task_status(info))
    else:
        active = await bridge.get_active_tasks(user=ctx.author.id)
        await ctx.respond(format_active_tasks(active))

@bot.slash_command(name="pause")
async def pause(ctx, task_id: str):
    await bridge.pause_task(task_id)
    await ctx.respond(f"⏸️ Task {task_id} paused. Use /resume to continue.")

@bot.slash_command(name="approve")
async def approve(ctx, task_id: str):
    await bridge.approve_hitl(task_id, approver=ctx.author.id)
    await ctx.respond(f"✅ Approved. Resuming task {task_id}.")
```

### Discord Channel Structure

```
Discord Server: Virtek Labs AI
├── #task-intake        ← Drop tasks here (or @mention the bot anywhere)
├── #ai-alerts          ← Anomalies, HITL requests, budget warnings
├── #daily-digest       ← Automated morning summary
├── #skill-proposals    ← New/updated skills awaiting approval
└── Threads (auto)      ← #task-acme-csv-export, #task-globex-api-v2 ...
```

### Discord UX Patterns

| Pattern | How It Works |
|---------|-------------|
| **Thread per task** | Every OpenClaw task gets its own Discord thread for clean updates |
| **Reaction-based approval** | ✅ = approve, ❌ = reject, ⏸️ = pause (no commands needed) |
| **Progress updates** | Agent posts step-complete messages in thread as work progresses |
| **Mention-based HITL** | `@ryan` tag in thread when human decision needed |
| **Daily digest** | Morning summary of overnight tasks, cost, quality scores |
| **Alert channel** | Separate #ai-alerts for anomalies, budget alerts, errors |

---

## 7 — Interface Unification: Three Ways to Reach the Platform

> All three interfaces connect to the **same MCP servers** and access the **same context, memory, and skills**. There is no "local mode" vs "cloud mode" — only one platform.

### Why Interface Unification Matters

In most multi-agent platforms, local development uses mocks or stubs, then you deploy to a different runtime with different behavior. OpenClaw avoids this by making the MCP servers the single source of truth accessible from any interface:

```
Interface Layer (NanoClaw / Claude Code CLI / REST API)
          ↓ identical MCP calls ↓
MCP Server Layer (context, memory, skills, guardrails)
          ↓ same DB queries, same RLS ↓
Data Layer (PostgreSQL + pgvector + Graphiti)
```

This means:
- A task started in Discord can be continued in a local Claude Code CLI session
- A skill proven in production is immediately available to local development
- HITL interventions work regardless of which interface triggered the task
- Observability (Langfuse traces, Prometheus metrics) captures all interfaces equally

---

### Interface 1: NanoClaw (Discord)

The production-facing interface. NanoClaw handles natural language from Discord messages and routes to OpenClaw via the Task Bridge.

```python
# nanoclaw/bridge.py
class OpenClawBridge:
    """
    Sits inside NanoClaw. Classifies every Discord message.
    Simple → NanoClaw handles directly (fast, cheap Haiku).
    Complex → delegates to OpenClaw via task queue.
    """
    async def handle_message(self, discord_message, group_context):
        complexity = await self.task_classifier.assess(
            discord_message.content,
            history=group_context.recent_messages[-5:]
        )
        
        if complexity.is_simple:
            return None  # NanoClaw handles it, no OpenClaw involvement
        
        # Inject correlation ID for end-to-end tracing
        task_id = await self.openclaw_client.submit(
            content=discord_message.content,
            client_id=group_context.client_id,
            interface="nanoclaw",
            correlation_id=str(discord_message.id),  # Discord msg ID as root
            context={
                "channel_id": discord_message.channel_id,
                "user_id": discord_message.author.id,
                "group_context": group_context.to_dict()
            }
        )
        return task_id
```

**Characteristics:**
| Property | Value |
|----------|-------|
| Interface type | Discord messages |
| Session scoping | Per-Discord-group mapped to client_id |
| Auth | Discord bot token (no user auth needed) |
| Task complexity | Simple (Haiku) or complex (delegates to OpenClaw) |
| Async behavior | Posts "Working on it..." then follows up when done |
| Observability | Discord message ID → correlation_id → Langfuse trace |

---

### Interface 2: Claude Code CLI (Local Development)

The developer-facing interface. A developer runs `claude` locally and connects to the live platform MCP servers via `~/.claude/mcp.json`. They get full access to the same context, memory, and skills as production agents.

```json
// ~/.claude/mcp.json — connects local Claude Code CLI to OpenClaw MCP servers
{
  "mcpServers": {
    "openclaw-context": {
      "command": "npx",
      "args": ["-y", "@openclaw/mcp-context-client"],
      "env": {
        "MCP_SERVER_URL": "https://mcp-context.openclaw.internal",
        "OPENCLAW_API_KEY": "${OPENCLAW_DEV_API_KEY}",
        "CLIENT_ID": "${CLIENT_ID}",
        "SESSION_SCOPE": "development"
      }
    },
    "openclaw-memory": {
      "command": "npx",
      "args": ["-y", "@openclaw/mcp-memory-client"],
      "env": {
        "MCP_SERVER_URL": "https://mcp-memory.openclaw.internal",
        "OPENCLAW_API_KEY": "${OPENCLAW_DEV_API_KEY}",
        "CLIENT_ID": "${CLIENT_ID}"
      }
    },
    "openclaw-skills": {
      "command": "npx",
      "args": ["-y", "@openclaw/mcp-skills-client"],
      "env": {
        "MCP_SERVER_URL": "https://mcp-skills.openclaw.internal",
        "OPENCLAW_API_KEY": "${OPENCLAW_DEV_API_KEY}"
      }
    },
    "openclaw-github": {
      "command": "npx",
      "args": ["-y", "@openclaw/mcp-github-client"],
      "env": {
        "MCP_SERVER_URL": "https://mcp-github.openclaw.internal",
        "OPENCLAW_API_KEY": "${OPENCLAW_DEV_API_KEY}"
      }
    }
  }
}
```

**Developer workflow example:**

```bash
# 1. Set environment variables
export OPENCLAW_DEV_API_KEY="dev-key-from-platform"
export CLIENT_ID="acme-corp"

# 2. Start Claude Code with platform MCP context
claude

# 3. Inside Claude Code — same context as production agents
> "What AI architecture patterns do we have for e-commerce clients?"
# → Queries mcp-memory with CLIENT_ID=acme-corp
# → Returns patterns from prior Acme Corp work + global patterns
# → Same result as if a NanoClaw agent asked the same question

# 4. Continue a task that started in Discord
> "The Acme team asked us in Discord to analyze their checkout flow latency.
   Let me pull up their context and draft the analysis."
# → Reads client context via mcp-context
# → Has access to all prior work, constraints, preferences
# → Can write findings back to memory (tagged agent_generated, trust: 0.8)
```

**Characteristics:**
| Property | Value |
|----------|-------|
| Interface type | Local `claude` CLI session |
| Session scoping | CLIENT_ID env var → mcp-context enforces RLS |
| Auth | OPENCLAW_DEV_API_KEY (dev key with write access) |
| Task complexity | All tasks handled by Claude Code SDK directly |
| Async behavior | Synchronous — developer sees responses in real time |
| Observability | SESSION_SCOPE=development tag in Langfuse; same traces |

**Security note:** Dev API keys are scoped to specific client IDs and cannot access other clients' data — RLS at the database level enforces this regardless of the key used.

---

### Interface 3: REST API

The programmatic interface for webhooks, integrations, and external systems that need to trigger OpenClaw tasks.

```python
# Example: Submit a task via REST API
import httpx

async def submit_task(content: str, client_id: str):
    async with httpx.AsyncClient() as client:
        response = await client.post(
            "https://api.openclaw.internal/v1/tasks",
            headers={
                "Authorization": f"Bearer {OPENCLAW_API_KEY}",
                "X-Client-ID": client_id,
                "X-Correlation-ID": str(uuid4())  # caller-provided trace root
            },
            json={
                "content": content,
                "interface": "rest_api",
                "priority": "normal",
                "callback_url": "https://your-service.com/webhooks/openclaw"
            }
        )
        return response.json()  # {"task_id": "...", "status": "queued"}

# Webhook callback format (sent to callback_url when complete)
{
    "task_id": "task-uuid",
    "status": "completed",
    "result": { ... },
    "correlation_id": "caller-provided-id",
    "duration_ms": 4832,
    "langfuse_trace_url": "https://langfuse.openclaw.internal/trace/..."
}
```

**Characteristics:**
| Property | Value |
|----------|-------|
| Interface type | HTTP REST |
| Session scoping | X-Client-ID header → mcp-context enforces RLS |
| Auth | Bearer token (client-specific API key) |
| Task complexity | All tasks queued to Celery → OpenClaw workers |
| Async behavior | Returns task_id immediately, result via webhook or polling |
| Observability | X-Correlation-ID → correlation_id → Langfuse trace |

---

### Context Continuity Across Interfaces

Because all interfaces write to the same memory store, context accumulates regardless of how a task was submitted:

```
Monday:   Discord (NanoClaw)
          "Analyze our Q1 marketing performance"
          → Agent creates memory: "Acme Q1 analysis: CTR down 12%, CPL up 8%"
          → Tagged: agent_generated, trust: 0.8, client_id: acme-corp

Tuesday:  Claude Code CLI (developer)
          "What did we find about Acme's marketing last quarter?"
          → mcp-memory returns Monday's finding
          → Developer can build on it, add nuance, write back

Wednesday: REST API (automated weekly report system)
           POST /v1/tasks { "content": "Generate weekly summary for Acme" }
           → Agent finds both Monday's analysis AND Tuesday's dev notes
           → Generates report incorporating all prior context

Thursday: Discord (NanoClaw)
          Acme team: "Can you explain the CPL increase?"
          → Agent has full chain of context from all three prior interactions
```

**Session state is NOT shared** (each interface session is independent), but **memory and context ARE shared** (persisted between sessions). This is the right tradeoff: agents don't bleed state between conversations, but they accumulate knowledge over time.

---

### Interface Comparison Summary

| | NanoClaw (Discord) | Claude Code CLI | REST API |
|---|---|---|---|
| **Primary user** | End clients, team | Developers | External systems |
| **Auth** | Discord bot token | Dev API key | Client API key |
| **Task routing** | Simple/complex split | Always Claude Code SDK | Always OpenClaw workers |
| **Response mode** | Async (follow-up message) | Sync (real-time) | Async (webhook/poll) |
| **Context access** | Full (client-scoped) | Full (dev-scoped) | Full (client-scoped) |
| **HITL triggers** | ✅ Yes | ✅ Yes (shown inline) | ✅ Yes (via webhook) |
| **Observability** | Discord msg ID as root | SESSION_SCOPE=dev tag | X-Correlation-ID as root |
| **Cost routing** | Haiku (simple) / SDK (complex) | Claude Code SDK only | Haiku or SDK (by complexity) |

> **See also:** [Doc 07 Quick Start, Step 3](07-hierarchical-context-architecture.md) for the exact `~/.claude/mcp.json` configuration and [Doc 10 Implementation Plan](10-implementation-plan.md) for how to provision the MCP server endpoints that all three interfaces connect to.

---

## 8. Layer 2: Task Intelligence & LLM Routing

### Task Classification

Every incoming request is first classified to determine complexity and routing:

```python
# task_classifier.py
class TaskClassifier:
    COMPLEXITY_LEVELS = {
        "trivial": {
            "description": "Single-step, factual, no tools needed",
            "model": "haiku",
            "examples": ["What time is it?", "Define 'CAC'", "Translate this word"]
        },
        "simple": {
            "description": "1-3 steps, single tool, clear outcome",
            "model": "haiku",
            "examples": ["Summarize this doc", "Format this CSV", "Check this URL"]
        },
        "moderate": {
            "description": "Multi-step, 1-2 tools, some reasoning",
            "model": "sonnet",
            "examples": ["Write a LinkedIn post about X", "Analyze this data", "Draft email response"]
        },
        "complex": {
            "description": "Multi-agent, multiple tools, significant reasoning",
            "model": "sonnet",
            "examples": ["Research competitors and write report", "Audit codebase for X", "Plan Q4 strategy"]
        },
        "critical": {
            "description": "High-stakes, requires best reasoning, HITL likely",
            "model": "opus",
            "examples": ["Architecture decision for client system", "Legal/financial analysis", "Sensitive client communication"]
        }
    }
    
    async def classify(self, task_content: str, client_context: dict) -> TaskClassification:
        # Fast classification using Haiku (cheap, ~100ms)
        classification = await self.haiku_client.classify(
            task=task_content,
            context=client_context,
            schema=TaskClassification
        )
        return classification
```

### LLM Routing Logic

```python
# llm_router.py
class LLMRouter:
    def route(self, task: Task) -> LLMConfig:
        # Cost-optimized routing
        if task.complexity in ["trivial", "simple"]:
            return LLMConfig(model="claude-haiku-3", max_tokens=1000)
        
        elif task.complexity == "moderate":
            return LLMConfig(model="claude-sonnet-4-5", max_tokens=4000)
        
        elif task.complexity == "complex":
            # Check if client has budget headroom
            if self.budget_tracker.has_headroom(task.client_id, estimate="$0.05-0.20"):
                return LLMConfig(model="claude-sonnet-4-5", max_tokens=8000)
            else:
                # Degrade gracefully, notify client
                self.notifier.warn(task.client_id, "Budget near limit, using Haiku for this task")
                return LLMConfig(model="claude-haiku-3", max_tokens=4000)
        
        elif task.complexity == "critical":
            # Always use best model, always HITL
            return LLMConfig(
                model="claude-opus-4",
                max_tokens=16000,
                require_hitl=True,
                hitl_reason="Critical task requires human approval before execution"
            )
```

### Cost Tracking Per Client

```python
# budget_tracker.py  
class BudgetTracker:
    async def record_usage(self, client_id: str, task_id: str, usage: TokenUsage):
        cost = self.calculate_cost(usage)
        await self.db.execute("""
            INSERT INTO client_usage (client_id, task_id, tokens_in, tokens_out, cost_usd, timestamp)
            VALUES ($1, $2, $3, $4, $5, NOW())
        """, client_id, task_id, usage.input_tokens, usage.output_tokens, cost)
        
        # Check against client budget limits
        monthly_total = await self.get_monthly_total(client_id)
        limits = await self.get_client_limits(client_id)
        
        if monthly_total > limits.warning_threshold:
            await self.notifier.budget_warning(client_id, monthly_total, limits.hard_limit)
        
        if monthly_total > limits.hard_limit:
            await self.pause_client_tasks(client_id)
            await self.notifier.budget_exceeded(client_id)
```

---

## 9. Layer 3: Orchestration & Execution

### Claude Code SDK Integration

OpenClaw uses the Claude Code SDK as its primary agent execution runtime. This gives us:
- Built-in tool use (file system, bash, web fetch)
- Streaming responses for long-running tasks
- Automatic retry and error handling
- Session management with persistent context

```python
# openclaw_agent.py
from anthropic import Anthropic
from claude_code_sdk import ClaudeCode, Tool, Session

class OpenClawAgent:
    def __init__(self, task: Task, mcp_context: MCPContext):
        self.task = task
        self.mcp = mcp_context
        self.sdk = ClaudeCode(
            model=task.llm_config.model,
            tools=self.get_tools_for_task(task),
            system_prompt=self.build_system_prompt(task, mcp_context)
        )
    
    def build_system_prompt(self, task: Task, ctx: MCPContext) -> str:
        return f"""You are a specialist agent for {ctx.client.name}.

CURRENT CONTEXT:
{ctx.client_profile}

RELEVANT PRIOR WORK:
{ctx.relevant_memories}

ACTIVE SKILLS:
{ctx.applicable_skills}

CONSTRAINTS:
{ctx.active_constraints}

TASK:
{task.content}

Execute this task. Use tools as needed. If you encounter a decision point 
requiring human judgment, call the request_hitl tool with a clear question.
"""
    
    async def execute(self) -> TaskResult:
        async with self.sdk.session() as session:
            result = await session.run(
                prompt=self.task.content,
                on_tool_use=self.handle_tool_use,
                on_hitl_request=self.handle_hitl
            )
            return TaskResult(
                task_id=self.task.id,
                output=result.content,
                tool_calls=result.tool_calls,
                tokens_used=result.usage
            )
```

### Parallel Agent Execution

For complex tasks, multiple specialist agents run in parallel:

```python
# orchestrator.py
class TaskOrchestrator:
    async def execute_plan(self, plan: TaskPlan) -> PlanResult:
        # Group sub-tasks by dependency level
        execution_levels = plan.get_execution_levels()
        
        results = {}
        for level in execution_levels:
            # All tasks at this level can run in parallel
            level_tasks = [
                self.run_subtask(subtask, prior_results=results)
                for subtask in level
            ]
            level_results = await asyncio.gather(*level_tasks)
            results.update(dict(zip([t.id for t in level], level_results)))
        
        return PlanResult(subtask_results=results)
    
    async def run_subtask(self, subtask: SubTask, prior_results: dict) -> SubTaskResult:
        # Inject results from prior levels as context
        agent = self.agent_factory.create(
            subtask,
            prior_context=self.format_prior_results(prior_results, subtask.dependencies)
        )
        return await agent.execute()
```

### HITL (Human-in-the-Loop) Manager

```python
# hitl_manager.py
class HITLManager:
    async def request_approval(self, task_id: str, question: str, options: list[str]) -> HITLResponse:
        # Post to Discord thread
        message = await self.discord.post_hitl_request(
            task_id=task_id,
            question=question,
            options=options  # Rendered as reaction buttons
        )
        
        # Pause task execution
        await self.task_queue.pause(task_id)
        
        # Wait for human response (with timeout)
        try:
            response = await asyncio.wait_for(
                self.wait_for_reaction(message.id, options),
                timeout=self.get_timeout(task_id)  # 1hr default, configurable
            )
            await self.task_queue.resume(task_id, context={"hitl_response": response})
            return response
        except asyncio.TimeoutError:
            await self.handle_timeout(task_id, question)
            raise HITLTimeoutError(f"No response within timeout for task {task_id}")
```

### Celery Task Queue

```python
# celery_config.py
from celery import Celery

app = Celery('openclaw',
    broker='redis://localhost:6379/0',
    backend='redis://localhost:6379/1'
)

app.conf.task_routes = {
    'openclaw.tasks.execute_agent': {'queue': 'agents'},
    'openclaw.tasks.run_improvement': {'queue': 'improvement'},
    'openclaw.tasks.send_notification': {'queue': 'notifications'},
}

app.conf.task_soft_time_limit = 300   # 5 min warning
app.conf.task_time_limit = 600        # 10 min hard limit

@app.task(bind=True, max_retries=3)
def execute_agent_task(self, task_id: str):
    try:
        task = TaskRepository.get(task_id)
        agent = AgentFactory.create(task)
        result = asyncio.run(agent.execute())
        TaskRepository.update(task_id, result)
        return result.to_dict()
    except Exception as exc:
        raise self.retry(exc=exc, countdown=60)
```

---

## 10. Layer 4: Hierarchical Context

The context system is the core intelligence layer. Every agent execution loads the right context automatically based on the task's client_id and task type.

### Context Hierarchy

```
┌─────────────────────────────────────────────────┐
│  GLOBAL LAYER (all agents, all clients)         │
│  • Platform capabilities and constraints        │
│  • Universal best practices                     │
│  • Tool documentation                          │
│  • Safety rules                                │
└─────────────────────┬───────────────────────────┘
                      │ inherited by
┌─────────────────────▼───────────────────────────┐
│  AGENCY LAYER (Virtek Labs specific)            │
│  • Your methodologies and approaches            │
│  • Your client communication style             │
│  • Your pricing and service tiers              │
│  • Cross-client patterns and learnings         │
└─────────────────────┬───────────────────────────┘
                      │ inherited by
┌─────────────────────▼───────────────────────────┐
│  CLIENT LAYER (per-client, strict isolation)    │
│  • Client industry, size, tech stack            │
│  • Active projects and their status             │
│  • Client preferences and constraints          │
│  • Historical work and outcomes                │
└─────────────────────┬───────────────────────────┘
                      │ inherited by
┌─────────────────────▼───────────────────────────┐
│  PROJECT LAYER (per-project context)            │
│  • Project goals and success criteria           │
│  • Technical decisions made                    │
│  • Current blockers and risks                  │
└─────────────────────┬───────────────────────────┘
                      │ inherited by
┌─────────────────────▼───────────────────────────┐
│  SESSION LAYER (current conversation only)      │
│  • This conversation's messages                 │
│  • Working notes and intermediate results       │
│  • Cleared at conversation end                 │
└─────────────────────────────────────────────────┘
```

### MCP Context Server Implementation

```python
# mcp_context_server.py
class MCPContextServer:
    """
    Exposed as MCP tools to all agents.
    Agents call these tools; they don't access the DB directly.
    """
    
    async def get_context(self, client_id: str, task_type: str, query: str) -> Context:
        """Build a context snapshot for an agent execution."""
        
        # Load each layer (with RLS enforced at DB level)
        global_ctx = await self.load_global_context(task_type)
        agency_ctx = await self.load_agency_context(task_type)
        client_ctx = await self.load_client_context(client_id)
        
        # Semantic search for relevant memories
        relevant_memories = await self.memory_store.search(
            query=query,
            client_id=client_id,
            limit=10,
            min_trust=0.5
        )
        
        # Find applicable skills
        skills = await self.skill_registry.find_applicable(
            task_type=task_type,
            client_context=client_ctx
        )
        
        return Context(
            global_layer=global_ctx,
            agency_layer=agency_ctx,
            client_layer=client_ctx,
            memories=relevant_memories,
            skills=skills
        )
    
    async def write_memory(self, client_id: str, content: str, 
                          trust: float, tags: list[str]) -> str:
        """Store a new memory from agent output."""
        memory = Memory(
            content=content,
            client_id=client_id,
            trust_score=trust,
            tags=tags,
            source="agent_generated",
            embedding=await self.embedder.embed(content)
        )
        return await self.memory_store.insert(memory)
```

### Row-Level Security for Client Isolation

```sql
-- PostgreSQL RLS: clients can only see their own data
ALTER TABLE memories ENABLE ROW LEVEL SECURITY;

CREATE POLICY client_isolation ON memories
    USING (client_id = current_setting('app.current_client_id'));

-- Applied at connection setup
SET app.current_client_id = 'acme-corp';
-- Now all queries on `memories` automatically filter to acme-corp only
```

### Context Loading Performance

| Layer | Storage | Latency | Size |
|-------|---------|---------|------|
| Global | Redis (hot cache) | < 5ms | ~2KB |
| Agency | Redis (hot cache) | < 5ms | ~3KB |
| Client | PostgreSQL + Redis | 20-50ms | ~5KB |
| Memories (semantic) | pgvector | 50-100ms | top-10 results |
| Skills | PostgreSQL + Redis | < 10ms | applicable set |
| **Total** | | **~150ms** | **~15-20KB** |

---

## 11. Layer 5: Memory & Knowledge

### Memory Architecture

```
                    MEMORY SYSTEM
                         │
          ┌──────────────┼──────────────┐
          │              │              │
    SHORT-TERM      LONG-TERM      STRUCTURAL
    (Session)       (Graphiti)     (pgvector)
          │              │              │
    Redis cache    Knowledge graph  Semantic search
    Cleared on     Temporal facts   Embedding-based
    session end    with trust       retrieval
                   scores
```

### Graphiti Knowledge Graph

Graphiti gives us a temporal knowledge graph where facts have:
- **Source**: who created this knowledge (human, agent, imported)
- **Trust score**: how reliable is this (0.0-1.0)
- **Timestamp**: when was this true
- **Relationships**: how does this connect to other facts

```python
# memory_manager.py
from graphiti_core import Graphiti

class MemoryManager:
    def __init__(self):
        self.graphiti = Graphiti(
            neo4j_uri=NEO4J_URI,
            neo4j_user=NEO4J_USER,
            neo4j_password=NEO4J_PASSWORD
        )
    
    async def store_agent_finding(self, client_id: str, finding: str, 
                                   task_id: str, confidence: float):
        """Store a fact discovered by an agent."""
        await self.graphiti.add_episode(
            name=f"agent_finding_{task_id}",
            episode_body=finding,
            source_description=f"Agent analysis (task {task_id})",
            reference_time=datetime.now(),
            source=EpisodeType.text
        )
        
        # Also store as pgvector for fast semantic search
        await self.vector_store.upsert(
            content=finding,
            metadata={
                "client_id": client_id,
                "task_id": task_id,
                "trust": confidence,
                "source": "agent_generated",
                "timestamp": datetime.now().isoformat()
            }
        )
    
    async def search_relevant(self, query: str, client_id: str, 
                              min_trust: float = 0.5) -> list[Memory]:
        """Find memories relevant to current task."""
        # Fast semantic search via pgvector
        results = await self.vector_store.search(
            query=query,
            filter={"client_id": client_id, "trust": {"$gte": min_trust}},
            limit=10
        )
        return results
```

### Memory Trust Model

| Source | Trust Score | Rationale |
|--------|-------------|-----------|
| Human-authored (direct input) | 1.0 | Human explicitly stated this |
| Human-approved (reviewed agent output) | 0.9 | Human verified agent work |
| Agent-generated (high-confidence task) | 0.8 | Agent completed successfully |
| Agent-inferred (reasoning step) | 0.6 | Agent deduction, not confirmed |
| Imported (external data) | 0.5-0.7 | Depends on source reliability |
| Unverified agent speculation | 0.3 | Low-confidence output |

### Memory Lifecycle

```python
# memory_lifecycle.py
class MemoryLifecycle:
    async def decay_stale_memories(self):
        """Reduce trust of old unconfirmed memories."""
        stale_threshold = datetime.now() - timedelta(days=90)
        await self.db.execute("""
            UPDATE memories
            SET trust_score = trust_score * 0.9
            WHERE updated_at < $1
              AND source = 'agent_generated'
              AND trust_score > 0.3
        """, stale_threshold)
    
    async def promote_confirmed_memory(self, memory_id: str, confirmer: str):
        """Boost trust when human confirms a memory."""
        await self.db.execute("""
            UPDATE memories
            SET trust_score = LEAST(trust_score + 0.2, 1.0),
                confirmed_by = $2,
                confirmed_at = NOW()
            WHERE id = $1
        """, memory_id, confirmer)
    
    async def archive_obsolete(self, memory_id: str, reason: str):
        """Mark a memory as superseded."""
        await self.db.execute("""
            UPDATE memories
            SET status = 'archived',
                archive_reason = $2,
                archived_at = NOW()
            WHERE id = $1
        """, memory_id, reason)
```

---

## 12. Layer 6: Self-Improvement Engine

This is the most distinctive layer. The system observes its own performance and proposes improvements to its skills, prompts, and approaches — with human approval before deployment.

### Improvement Loop Architecture

```
Task Execution
      │
      ▼
Outcome Collection
(was the result good? did the client use it? were there errors?)
      │
      ▼
Pattern Extraction
(what patterns correlate with good/bad outcomes?)
      │
      ▼
Improvement Proposals
(new skill? updated prompt? better approach?)
      │
      ▼
Human Review (Discord #skill-proposals)
(approve/reject/modify)
      │
      ▼
Deployment
(A/B test → production)
      │
      ▼
Validation
(did the improvement actually improve things?)
```

### Outcome Tracking

```python
# outcome_tracker.py
class OutcomeTracker:
    async def record_outcome(self, task_id: str, outcome: TaskOutcome):
        """
        Called when:
        - Task completes (success/failure)
        - Human reacts to output (👍/👎)
        - Follow-up task references this task (implies it was useful)
        - Human explicitly rates the work
        """
        await self.db.execute("""
            INSERT INTO task_outcomes (
                task_id, status, quality_score, human_feedback,
                tokens_used, duration_ms, error_type, retry_count
            ) VALUES ($1, $2, $3, $4, $5, $6, $7, $8)
        """, task_id, outcome.status, outcome.quality_score,
            outcome.human_feedback, outcome.tokens_used,
            outcome.duration_ms, outcome.error_type, outcome.retry_count)
    
    async def calculate_quality_score(self, task_id: str) -> float:
        """Multi-signal quality score (0.0-1.0)."""
        task = await self.get_task(task_id)
        signals = []
        
        if task.completed_without_error:
            signals.append(0.5)  # Base: completed successfully
        
        if task.human_approved:
            signals.append(0.3)  # Human reviewed and approved
        
        if task.referenced_by_followup:
            signals.append(0.1)  # Output was useful enough to build on
        
        if task.explicit_rating:
            signals.append(task.explicit_rating * 0.1)  # Direct rating
        
        return sum(signals)
```

### Pattern Extraction

```python
# pattern_extractor.py
class PatternExtractor:
    async def extract_improvement_opportunities(self) -> list[ImprovementOpportunity]:
        opportunities = []
        
        # Pattern 1: Tasks that consistently fail in a specific way
        failing_patterns = await self.db.fetch("""
            SELECT task_type, error_type, skill_used, COUNT(*) as failure_count
            FROM task_outcomes
            WHERE status = 'failed'
              AND created_at > NOW() - INTERVAL '30 days'
            GROUP BY task_type, error_type, skill_used
            HAVING COUNT(*) >= 3
            ORDER BY failure_count DESC
        """)
        
        for pattern in failing_patterns:
            opportunities.append(ImprovementOpportunity(
                type="fix_failing_skill",
                description=f"Skill '{pattern['skill_used']}' fails with "
                           f"'{pattern['error_type']}' on '{pattern['task_type']}' tasks",
                evidence=f"{pattern['failure_count']} failures in last 30 days",
                priority="high"
            ))
        
        # Pattern 2: Tasks that are slow and expensive
        slow_tasks = await self.db.fetch("""
            SELECT task_type, AVG(duration_ms) as avg_duration, 
                   AVG(tokens_used) as avg_tokens, COUNT(*) as count
            FROM task_outcomes
            WHERE created_at > NOW() - INTERVAL '30 days'
            GROUP BY task_type
            HAVING AVG(duration_ms) > 60000  -- slower than 1 minute
               AND COUNT(*) >= 5
        """)
        
        for task in slow_tasks:
            opportunities.append(ImprovementOpportunity(
                type="optimize_slow_task",
                description=f"Task type '{task['task_type']}' averages "
                           f"{task['avg_duration']/1000:.0f}s and "
                           f"{task['avg_tokens']:,.0f} tokens",
                priority="medium"
            ))
        
        return opportunities
```

### Skill Proposal System

```python
# skill_proposer.py
class SkillProposer:
    async def generate_skill_proposal(self, 
                                       opportunity: ImprovementOpportunity) -> SkillProposal:
        """Use Sonnet to generate an improved skill based on observed patterns."""
        
        # Gather evidence
        example_tasks = await self.get_example_tasks(opportunity)
        current_skill = await self.get_current_skill(opportunity.skill_id)
        failure_contexts = await self.get_failure_contexts(opportunity)
        
        # Generate improved skill
        proposal = await self.claude.generate(
            system="""You are an expert at writing AI agent skills (system prompts).
            Analyze the failure patterns and generate an improved skill that addresses them.
            Return a JSON object with: name, description, prompt, expected_improvement.""",
            
            user=f"""Current skill:
{current_skill.prompt}

Failure patterns observed:
{json.dumps(failure_contexts, indent=2)}

Example tasks that failed:
{json.dumps(example_tasks, indent=2)}

Generate an improved version of this skill that addresses these failure patterns."""
        )
        
        return SkillProposal(
            opportunity=opportunity,
            current_skill=current_skill,
            proposed_skill=Skill(**proposal),
            evidence=failure_contexts
        )
    
    async def post_to_discord_for_review(self, proposal: SkillProposal):
        """Post skill proposal to #skill-proposals channel."""
        embed = self.build_proposal_embed(proposal)
        message = await self.discord.post(
            channel="#skill-proposals",
            content=f"📊 New skill improvement proposal",
            embed=embed
        )
        # Add review reactions
        await message.add_reaction("✅")  # Approve as-is
        await message.add_reaction("🔧")  # Approve with modifications
        await message.add_reaction("❌")  # Reject
```

### A/B Testing for Skills

```python
# ab_tester.py
class SkillABTester:
    async def run_ab_test(self, control_skill: Skill, 
                          test_skill: Skill, 
                          duration_hours: int = 24) -> ABTestResult:
        """Run incoming tasks against both skill versions, compare outcomes."""
        
        test_id = str(uuid4())
        end_time = datetime.now() + timedelta(hours=duration_hours)
        
        # Route 20% of eligible tasks to test version
        await self.db.execute("""
            INSERT INTO ab_tests (id, control_skill_id, test_skill_id, 
                                  traffic_split, end_time, status)
            VALUES ($1, $2, $3, 0.2, $4, 'running')
        """, test_id, control_skill.id, test_skill.id, end_time)
        
        # Results collected automatically by OutcomeTracker
        # After duration, analyze results
        await asyncio.sleep(duration_hours * 3600)
        return await self.analyze_results(test_id)
```

---

## 13. Layer 7: Observability & Control

### Langfuse Integration

Every LLM call and agent execution is traced through Langfuse:

```python
# langfuse_integration.py
from langfuse import Langfuse
from langfuse.decorators import observe, langfuse_context

langfuse = Langfuse(
    public_key=LANGFUSE_PUBLIC_KEY,
    secret_key=LANGFUSE_SECRET_KEY,
    host=LANGFUSE_HOST
)

class ObservableAgent:
    @observe(name="agent_execution")
    async def execute(self, task: Task) -> TaskResult:
        # Automatically captures: input, output, latency, cost
        
        langfuse_context.update_current_trace(
            name=f"task_{task.id}",
            user_id=task.client_id,
            session_id=task.correlation_id,
            tags=[task.task_type, task.complexity, task.interface],
            metadata={
                "client_id": task.client_id,
                "skill_used": task.skill_id,
                "model": task.llm_config.model
            }
        )
        
        result = await self._execute_inner(task)
        
        langfuse_context.update_current_observation(
            output=result.output,
            level="DEFAULT" if result.success else "ERROR"
        )
        
        return result
```

### Prometheus Metrics

```python
# metrics.py
from prometheus_client import Counter, Histogram, Gauge

# Task metrics
tasks_total = Counter('openclaw_tasks_total', 
                      'Total tasks processed', 
                      ['client_id', 'task_type', 'status'])

task_duration = Histogram('openclaw_task_duration_seconds',
                          'Task execution duration',
                          ['task_type', 'complexity'],
                          buckets=[1, 5, 15, 30, 60, 120, 300])

# Cost metrics  
tokens_used = Counter('openclaw_tokens_total',
                      'Total tokens used',
                      ['client_id', 'model', 'direction'])

cost_usd = Counter('openclaw_cost_usd_total',
                   'Total cost in USD',
                   ['client_id', 'model'])

# Queue metrics
queue_depth = Gauge('openclaw_queue_depth',
                    'Current task queue depth',
                    ['queue_name'])

# Quality metrics
quality_score = Histogram('openclaw_quality_score',
                          'Task quality scores',
                          ['task_type', 'client_id'])
```

### Grafana Dashboard Layout

```
┌─────────────────────────────────────────────────────────────┐
│  OpenClaw Platform Dashboard                                │
├─────────────────┬───────────────────┬───────────────────────┤
│  Tasks/hour     │  Active workers   │  Queue depth          │
│  [time series]  │  [gauge: 0-20]    │  [gauge: 0-100]       │
├─────────────────┴───────────────────┴───────────────────────┤
│  Cost by client (last 30 days)      │  Quality scores trend │
│  [bar chart: client vs $]           │  [time series: 0-1]   │
├─────────────────────────────────────┴───────────────────────┤
│  Error rate by task type            │  HITL requests/day    │
│  [time series: % errors]            │  [time series: count] │
├─────────────────────────────────────┴───────────────────────┤
│  P50/P95/P99 task latency by complexity level               │
│  [time series with multiple lines]                          │
└─────────────────────────────────────────────────────────────┘
```

### Control Plane: Task Management API

```python
# control_api.py — Internal API for task management
@app.get("/tasks/{task_id}")
async def get_task_status(task_id: str):
    task = await TaskRepository.get(task_id)
    return {
        "task_id": task_id,
        "status": task.status,
        "progress": task.progress_pct,
        "current_step": task.current_step,
        "cost_so_far": task.cost_usd,
        "started_at": task.started_at,
        "estimated_completion": task.estimated_completion
    }

@app.post("/tasks/{task_id}/pause")
async def pause_task(task_id: str, reason: str = "Manual pause"):
    await task_queue.pause(task_id)
    await AuditLog.record(action="pause", task_id=task_id, reason=reason)
    return {"status": "paused"}

@app.post("/tasks/{task_id}/resume")  
async def resume_task(task_id: str):
    await task_queue.resume(task_id)
    return {"status": "resumed"}

@app.post("/tasks/{task_id}/cancel")
async def cancel_task(task_id: str, reason: str):
    await task_queue.cancel(task_id)
    await AuditLog.record(action="cancel", task_id=task_id, reason=reason)
    return {"status": "cancelled"}
```

---

## 14. Layer 8: Safety & Guardrails

### Autonomy Level Configuration

```yaml
# config/autonomy.yaml
autonomy_levels:
  level_1_supervised:
    description: "Draft only — all outputs require human approval"
    auto_execute_tools: false
    require_approval_for: ["all_outputs"]
    suitable_for: "New clients, sensitive domains"
  
  level_2_assisted:
    description: "Executes reversible actions, drafts irreversible"
    auto_execute_tools: ["read_only", "draft_creation"]
    require_approval_for: ["send_email", "post_social", "code_deploy"]
    suitable_for: "Established clients, moderate trust"
  
  level_3_autonomous:
    description: "Executes most actions, escalates high-stakes only"
    auto_execute_tools: ["most_tools"]
    require_approval_for: ["financial_transactions", "external_comms_to_clients"]
    suitable_for: "High-trust clients, proven track record"
  
  level_4_full_autonomy:
    description: "Full execution authority within defined scope"
    auto_execute_tools: ["all_within_scope"]
    require_approval_for: ["out_of_scope_actions", "budget_overrun"]
    suitable_for: "Mature client relationships, clear scope definition"

client_overrides:
  acme-corp:
    level: level_2_assisted
    exceptions:
      always_require_approval: ["anything_touching_salesforce"]
      never_require_approval: ["internal_doc_creation"]
```

### Action Classifier

```python
# action_classifier.py
class ActionClassifier:
    RISK_LEVELS = {
        "read_only": 0,       # Read files, query databases, fetch URLs
        "draft_creation": 1,  # Create files, write drafts (not published)
        "internal_action": 2, # Internal tool calls, test executions
        "external_read": 3,   # Call external APIs to read data
        "external_write": 4,  # Post to APIs, send webhooks
        "communication": 5,   # Send emails, post to social, message humans
        "financial": 6,       # Any action involving money
        "irreversible": 7,    # Delete data, deploy code, domain changes
    }
    
    async def classify(self, action: ToolCall) -> ActionClassification:
        risk_level = self.get_risk_level(action.tool_name, action.parameters)
        client_level = await self.get_client_autonomy_level(action.client_id)
        
        return ActionClassification(
            action=action,
            risk_level=risk_level,
            requires_approval=risk_level > client_level.auto_execute_threshold,
            reason=self.get_approval_reason(action, risk_level, client_level)
        )
```

### Prompt Injection Detection

```python
# injection_detector.py
class PromptInjectionDetector:
    INJECTION_PATTERNS = [
        r"ignore previous instructions",
        r"disregard your system prompt",
        r"you are now",
        r"new instruction:",
        r"OVERRIDE:",
        r"admin mode",
        r"\[INST\].*forget",
    ]
    
    async def check(self, content: str, source: str) -> InjectionCheckResult:
        # Pattern matching (fast, catches obvious attempts)
        for pattern in self.INJECTION_PATTERNS:
            if re.search(pattern, content, re.IGNORECASE):
                return InjectionCheckResult(
                    is_injection=True,
                    confidence=0.95,
                    pattern=pattern,
                    action="block"
                )
        
        # LLM-based check for subtle injections (slower, for high-risk inputs)
        if source in ["external_api", "web_fetch", "user_file_upload"]:
            result = await self.haiku_client.check_injection(content)
            if result.is_injection:
                return InjectionCheckResult(
                    is_injection=True,
                    confidence=result.confidence,
                    action="block" if result.confidence > 0.8 else "flag"
                )
        
        return InjectionCheckResult(is_injection=False, confidence=0.0)
```

### Budget Circuit Breaker

```python
# budget_circuit_breaker.py
class BudgetCircuitBreaker:
    async def check_before_execution(self, task: Task) -> CircuitBreakerResult:
        client_usage = await self.budget_tracker.get_current_period(task.client_id)
        client_limits = await self.get_limits(task.client_id)
        
        utilization = client_usage.cost_usd / client_limits.monthly_budget_usd
        
        if utilization >= 1.0:
            # Hard stop
            return CircuitBreakerResult(
                allow=False,
                reason="monthly_budget_exceeded",
                action="block_and_notify"
            )
        
        elif utilization >= 0.9:
            # Warning + human approval for new tasks
            return CircuitBreakerResult(
                allow=True,
                require_approval=True,
                reason=f"90% of budget used ({utilization:.0%})",
                action="warn_and_require_approval"
            )
        
        elif utilization >= 0.75:
            # Warning only
            return CircuitBreakerResult(
                allow=True,
                require_approval=False,
                reason=f"75% of budget used ({utilization:.0%})",
                action="warn_only"
            )
        
        return CircuitBreakerResult(allow=True)
```

---

## 15. AI Agency Platform: Building AI Systems for Clients

This is the meta-use case: we use OpenClaw to design and deliver custom AI systems for clients. The platform turns a human-hours-intensive consulting project into a systematic, scalable offering.

### The Agency Service Model

```
CLIENT REQUEST: "We want AI to help our support team handle tickets faster"

TRADITIONAL APPROACH:
- 2-4 weeks requirements gathering
- 4-8 weeks development
- 2-4 weeks testing/deployment
- $50-150k project cost
- Ongoing maintenance headaches

OPENCLAW AGENCY APPROACH:
- 1-2 days: Requirements + architecture via OpenClaw analysis
- 1-2 weeks: Build using OpenClaw as delivery platform
- 1-2 days: Deploy and configure
- $8-25k project cost + monthly retainer
- OpenClaw monitors and self-improves the delivered system
```

### Client Discovery Agent

```python
# agency/discovery_agent.py
class ClientDiscoveryAgent:
    """
    Runs during initial client engagement.
    Builds the client's context layer in OpenClaw.
    """
    
    DISCOVERY_PROMPT = """You are conducting an AI systems discovery session.
    
Your goal: Build a comprehensive context profile for this new client that will
enable our platform to serve them intelligently.

Gather:
1. Business overview (industry, size, revenue stage, team structure)
2. Current pain points that AI could address
3. Tech stack (what do they use? what integrations exist?)
4. Data assets (what data do they have? where does it live?)
5. Constraints (compliance, budget, timeline, existing tools they're locked into)
6. Success criteria (how will they know the AI system is working?)
7. Quick wins (what can we demonstrate value with in week 1?)

After gathering this information, generate:
- A structured client profile JSON
- A prioritized list of AI use cases
- A 90-day roadmap
- A risk assessment
"""
    
    async def run_discovery(self, client_id: str, 
                            conversation_transcript: str) -> ClientProfile:
        profile = await self.claude.extract(
            system=self.DISCOVERY_PROMPT,
            user=conversation_transcript,
            output_schema=ClientProfile
        )
        
        # Store in OpenClaw context layer
        await self.mcp_context.create_client_profile(client_id, profile)
        
        # Create initial memories
        for use_case in profile.use_cases:
            await self.mcp_memory.store(
                client_id=client_id,
                content=f"Identified use case: {use_case.description}",
                trust=0.8,
                tags=["discovery", "use_case", use_case.priority]
            )
        
        return profile
```

### AI System Architecture Generator

```python
# agency/architect_agent.py
class ArchitectAgent:
    """
    Given a client brief, generates a complete AI system architecture.
    This is one of the highest-value deliverables in the agency model.
    """
    
    async def generate_architecture(self, 
                                     client_profile: ClientProfile,
                                     use_case: UseCase) -> SystemArchitecture:
        
        # Pull relevant patterns from global memory
        similar_architectures = await self.mcp_memory.search(
            query=f"{use_case.category} AI system architecture",
            client_id="global",  # Pull from global/agency layer
            min_trust=0.7
        )
        
        architecture = await self.claude.generate(
            system="""You are a senior AI systems architect.
            Generate a detailed, production-ready architecture for the given use case.
            Base it on proven patterns from similar implementations.
            Output a structured architecture document.""",
            
            user=f"""Client: {client_profile.company_name}
Use case: {use_case.description}
Tech stack: {client_profile.tech_stack}
Constraints: {client_profile.constraints}

Relevant prior architectures:
{similar_architectures}

Generate a complete system architecture including:
1. Component diagram
2. Data flow
3. Integration points
4. Implementation sequence
5. Risk mitigations
6. Cost estimates
7. Success metrics"""
        )
        
        # Store this architecture for future clients with similar use cases
        await self.mcp_memory.store(
            client_id="agency",
            content=f"Architecture for {use_case.category}: {architecture.summary}",
            trust=0.8,
            tags=["architecture", use_case.category, client_profile.industry]
        )
        
        return architecture
```

### Delivery Tracking

```python
# agency/delivery_tracker.py
class DeliveryTracker:
    """
    Tracks AI system delivery projects.
    Uses OpenClaw to automate status updates, risk flags, and reporting.
    """
    
    async def weekly_status_update(self, project_id: str) -> StatusReport:
        project = await self.get_project(project_id)
        
        # Agent generates status update from project context
        status = await self.openclaw.run_task(
            content=f"""Generate a weekly status report for project {project.name}.
            
Current state:
{await self.get_current_state(project_id)}

Compare against plan:
{project.original_plan}

Flag any risks or blockers. Be specific about what's on track and what isn't.""",
            client_id=project.agency_client_id,
            task_type="status_report"
        )
        
        # Post to Discord and email to client
        await self.discord.post(f"📊 Weekly update: {project.name}\n{status.summary}")
        await self.email.send(project.client_email, status.full_report)
        
        return status
```

### Pricing Model

| Service | Delivery | Monthly | Notes |
|---------|----------|---------|-------|
| **Discovery & Architecture** | $3,000-8,000 | — | One-time, 1-2 day deliverable |
| **MVP Build** | $8,000-20,000 | — | 2-4 week build on OpenClaw |
| **Production System** | $15,000-40,000 | — | Full deployment + documentation |
| **Managed Service** | — | $1,500-5,000 | OpenClaw monitors + improves |
| **Platform License** | — | $800-2,000 | White-label OpenClaw for client's use |

**Unit economics (Managed Service at $2,500/mo):**
- LLM costs: ~$200-400/mo (OpenClaw's efficiency)
- Infrastructure: ~$150/mo
- Human oversight: ~2 hrs/mo (~$200)
- **Gross margin: ~65-70%**

---

## 16. MCP-First Architecture

### Why MCP Is Central

The Model Context Protocol (MCP) is not just a tool integration standard — it's the architectural backbone that makes interface unification possible:

```
Every capability is an MCP server.
Every agent is an MCP client.
The interface (Discord/CLI/API) is just a different entrypoint.
```

This means:
- **No capability lock-in**: Any MCP client can use any capability
- **No interface lock-in**: Swap Discord for Slack without touching the capability layer
- **No runtime lock-in**: Run locally, on Railway, or any cloud — same MCP endpoints
- **Composable**: New capabilities = new MCP server, available to all interfaces instantly

### MCP Server Registry

| Server | Purpose | Tools Exposed |
|--------|---------|---------------|
| `mcp-context` | Hierarchical context loading | `get_context`, `update_context`, `list_contexts` |
| `mcp-memory` | Persistent memory (read/write) | `search_memory`, `store_memory`, `update_trust`, `archive` |
| `mcp-skills` | Skill registry | `list_skills`, `get_skill`, `propose_skill`, `deploy_skill` |
| `mcp-github` | GitHub operations | `read_file`, `write_file`, `create_pr`, `run_action` |
| `mcp-sheets` | Google Sheets | `read_range`, `write_range`, `create_sheet` |
| `mcp-search` | Web research | `search`, `fetch_url`, `summarize_page` |
| `mcp-email` | Email operations | `send_email`, `read_inbox`, `reply` |
| `mcp-calendar` | Calendar management | `get_events`, `create_event`, `find_availability` |

### MCP Server Implementation Pattern

```python
# mcp_servers/mcp_memory/server.py
from mcp.server import Server
from mcp.server.models import InitializationOptions
import mcp.types as types

server = Server("openclaw-memory")

@server.list_tools()
async def handle_list_tools() -> list[types.Tool]:
    return [
        types.Tool(
            name="search_memory",
            description="Semantic search across stored memories for current client",
            inputSchema={
                "type": "object",
                "properties": {
                    "query": {"type": "string", "description": "Search query"},
                    "min_trust": {"type": "number", "description": "Minimum trust score (0-1)"},
                    "limit": {"type": "integer", "description": "Max results", "default": 10}
                },
                "required": ["query"]
            }
        ),
        types.Tool(
            name="store_memory",
            description="Store a new memory for current client",
            inputSchema={
                "type": "object", 
                "properties": {
                    "content": {"type": "string"},
                    "trust": {"type": "number"},
                    "tags": {"type": "array", "items": {"type": "string"}}
                },
                "required": ["content", "trust"]
            }
        )
    ]

@server.call_tool()
async def handle_call_tool(name: str, arguments: dict) -> list[types.TextContent]:
    # Client ID comes from the authenticated session, not the agent
    client_id = get_client_id_from_session()
    
    if name == "search_memory":
        results = await memory_store.search(
            query=arguments["query"],
            client_id=client_id,  # RLS enforced here
            min_trust=arguments.get("min_trust", 0.5),
            limit=arguments.get("limit", 10)
        )
        return [types.TextContent(type="text", text=json.dumps(results))]
    
    elif name == "store_memory":
        memory_id = await memory_store.store(
            content=arguments["content"],
            client_id=client_id,
            trust=arguments["trust"],
            tags=arguments.get("tags", [])
        )
        return [types.TextContent(type="text", text=json.dumps({"id": memory_id}))]
```

---

## 17. Complete Technology Stack

### Core Platform

| Layer | Technology | Purpose |
|-------|-----------|---------|
| **LLM** | Anthropic (Haiku/Sonnet/Opus) | All agent intelligence |
| **Agent Runtime** | Claude Code SDK | Agent execution framework |
| **Orchestration** | Celery + Redis | Task queue, parallel execution |
| **Context Protocol** | MCP (Model Context Protocol) | Tool/capability standardization |
| **API Framework** | FastAPI | REST API + webhook endpoints |
| **Discord Interface** | py-cord | NanoClaw Discord bot |

### Data Storage

| Type | Technology | Purpose |
|------|-----------|---------|
| **Primary DB** | PostgreSQL | Tasks, clients, outcomes, audit |
| **Vector Search** | pgvector | Semantic memory search |
| **Knowledge Graph** | Graphiti (Neo4j) | Temporal knowledge graph |
| **Cache** | Redis | Session cache, task queue |
| **Object Storage** | Cloudflare R2 | Files, attachments, exports |

### Observability

| Tool | Purpose |
|------|---------|
| **Langfuse** | LLM tracing (every call, cost, decision) |
| **Prometheus** | System metrics collection |
| **Grafana** | Dashboards and alerting |
| **Sentry** | Error tracking with context |
| **Structured logging** | JSON logs to Grafana Loki |

### Infrastructure

| Component | Technology | Why |
|-----------|-----------|-----|
| **Hosting** | Railway | Fast deploys, managed infra, simple scaling |
| **Containers** | Docker | Consistent environments |
| **Secrets** | Railway env vars | Simple, integrated |
| **DNS/CDN** | Cloudflare | Protection + R2 storage |
| **CI/CD** | GitHub Actions | Automated testing + deployment |

### Cost Profile (At Scale)

| Component | Monthly (10 clients) | Monthly (50 clients) |
|-----------|---------------------|---------------------|
| Anthropic API | $400-800 | $1,800-3,500 |
| Railway hosting | $100-200 | $300-600 |
| PostgreSQL (managed) | $50-100 | $150-300 |
| Neo4j (Graphiti) | $65-130 | $200-400 |
| Redis | $30-60 | $100-200 |
| Langfuse (cloud) | $50 | $150 |
| Cloudflare R2 | $10-30 | $30-80 |
| **Total** | **~$700-1,300** | **~$2,700-5,200** |

---

## 18. Data Flow: Request to Completion

### Complete Data Flow Diagram

```
Discord Message / REST API / Claude Code CLI
              │
              ▼
┌─────────────────────────────┐
│   INTAKE LAYER              │
│   - Parse message           │
│   - Extract client_id       │
│   - Classify complexity     │
│   - Generate correlation_id │
└─────────────┬───────────────┘
              │
              ▼
┌─────────────────────────────┐
│   CONTEXT LOADING           │
│   GET mcp-context:          │
│   - Global rules            │
│   - Agency patterns         │
│   - Client profile          │
│   GET mcp-memory:           │
│   - Semantic search (top-N) │
│   GET mcp-skills:           │
│   - Applicable skills       │
└─────────────┬───────────────┘
              │
              ▼
┌─────────────────────────────┐
│   TASK PLANNING             │
│   Planner agent decides:    │
│   - Sub-task breakdown      │
│   - Agent assignments       │
│   - Execution order         │
│   - HITL checkpoints        │
└─────────────┬───────────────┘
              │
              ▼
┌─────────────────────────────┐
│   PARALLEL EXECUTION        │
│   Workers run sub-agents:   │
│   - Each has isolated ctx   │
│   - Tools via MCP           │
│   - HITL if needed          │
│   - Results aggregated      │
└─────────────┬───────────────┘
              │
              ▼
┌─────────────────────────────┐
│   OUTPUT SYNTHESIS          │
│   Synthesis agent:          │
│   - Combines sub-results    │
│   - Formats for interface   │
│   - Quality self-check      │
└─────────────┬───────────────┘
              │
              ▼
┌─────────────────────────────┐
│   DELIVERY                  │
│   - Post to Discord thread  │
│   - Webhook callback        │
│   - Return to Claude Code   │
└─────────────┬───────────────┘
              │
              ▼
┌─────────────────────────────┐
│   POST-EXECUTION (async)    │
│   - Store outcome           │
│   - Update memories         │
│   - Update quality scores   │
│   - Feed improvement engine │
│   - Update cost tracking    │
└─────────────────────────────┘
```

### State Transitions

```
Task States:
RECEIVED → CLASSIFIED → CONTEXT_LOADED → PLANNED → 
EXECUTING → [HITL_PENDING] → SYNTHESIZING → DELIVERED → 
[OUTCOME_RECORDED]

Error States:
Any state → FAILED (with error type + retry count)
EXECUTING → TIMEOUT (after 10 min hard limit)
PLANNED → REJECTED (if safety check fails)
EXECUTING → BUDGET_EXCEEDED (circuit breaker)
```

---

## 19. Self-Improvement Loop Detail

```
Week 1: Baseline
→ Tasks execute with initial skills and prompts
→ Outcomes recorded, quality scores calculated

Week 2: Pattern Detection
→ Improvement engine identifies: 
   "Data analysis tasks on spreadsheets fail 30% of time 
    when sheet has >1000 rows"
→ Root cause: Current skill doesn't handle pagination

Week 2 (cont): Proposal Generation
→ Skill Proposer generates improved data-analyst-v2 skill
→ Posts to #skill-proposals: "Proposed fix for large spreadsheet failures"

Week 2 (cont): Human Review
→ You review the proposal: looks good ✅
→ Skill deployed to 20% traffic (A/B test)

Week 3: Validation
→ A/B test results: data-analyst-v2 fails 8% (vs 30% baseline)
→ Auto-promote to 100% traffic

Week 3+: Compounding
→ data-analyst-v2 creates new patterns (handles 10K+ row sheets)
→ These patterns stored in agency memory layer
→ Available to ALL clients going forward
```

---

## 20. Human Oversight Patterns

The system is designed around **calibrated autonomy**: high autonomy for low-risk, routine work; mandatory human oversight for high-stakes or novel situations.

### Oversight Touchpoints

```yaml
# config/autonomy.yaml (excerpt)
oversight_triggers:
  always_ask:
    - action_involves_money
    - sending_external_communication
    - deleting_or_overwriting_data
    - out_of_scope_for_client
    - confidence_below_threshold  # < 70% confidence
    
  sometimes_ask:
    - first_time_task_type_for_client
    - unusual_data_pattern_detected
    - cost_estimate_above_threshold  # > $5 for single task
    - HITL_requested_by_agent
    
  never_interrupt:
    - read_only_operations
    - internal_draft_creation
    - context_loading
    - quality_scoring
```

### Daily Digest

```python
# daily_digest.py
class DailyDigestGenerator:
    async def generate(self, date: date) -> DailyDigest:
        """
        Generated every morning (configurable time).
        Posted to Discord #daily-digest.
        """
        return DailyDigest(
            tasks_completed=await self.count_completed(date),
            tasks_failed=await self.count_failed(date),
            hitl_requests=await self.count_hitl(date),
            total_cost=await self.sum_cost(date),
            top_clients=await self.get_top_clients(date),
            quality_avg=await self.avg_quality(date),
            improvements_proposed=await self.count_proposals(date),
            improvements_deployed=await self.count_deployed(date),
            alerts=await self.get_active_alerts()
        )
```

---

## 21. Deployment Architecture

### Railway Service Layout

```
Railway Project: openclaw-prod
├── openclaw-api          (FastAPI — main API + webhook receiver)
├── openclaw-worker-1     (Celery worker — agent execution)
├── openclaw-worker-2     (Celery worker — agent execution)
├── openclaw-beat         (Celery beat — scheduled tasks)
├── mcp-context           (MCP context server)
├── mcp-memory            (MCP memory server)
├── mcp-skills            (MCP skills server)
├── mcp-github            (MCP GitHub tools)
├── postgresql            (Primary database + pgvector)
├── redis                 (Task queue + cache)
└── nanoclaw              (Discord bot — separate Railway project)
```

### Environment Configuration

```bash
# .env.production (Railway env vars)
# Core
ANTHROPIC_API_KEY=sk-ant-...
ENVIRONMENT=production
LOG_LEVEL=INFO

# Database
DATABASE_URL=postgresql://...
REDIS_URL=redis://...

# MCP Servers
MCP_CONTEXT_URL=https://mcp-context.openclaw.internal
MCP_MEMORY_URL=https://mcp-memory.openclaw.internal
MCP_SKILLS_URL=https://mcp-skills.openclaw.internal

# Observability
LANGFUSE_PUBLIC_KEY=pk-lf-...
LANGFUSE_SECRET_KEY=sk-lf-...
LANGFUSE_HOST=https://langfuse.openclaw.internal
SENTRY_DSN=https://...

# Integrations
GITHUB_APP_ID=...
GITHUB_APP_PRIVATE_KEY=...
GOOGLE_SERVICE_ACCOUNT_JSON=...
```

### Scaling Strategy

```
Phase 1 (0-10 clients): Single worker + shared resources
  - 1x openclaw-worker (2 CPU, 4GB RAM)
  - Shared PostgreSQL (Railway managed)
  - Redis (Railway managed)
  - Cost: ~$200-400/mo infrastructure

Phase 2 (10-50 clients): Multiple workers + dedicated DB
  - 3-5x openclaw-worker (autoscale based on queue depth)
  - Dedicated PostgreSQL with read replica
  - Redis Cluster
  - Cost: ~$600-1200/mo infrastructure

Phase 3 (50+ clients): Full horizontal scaling
  - N workers based on load (queue depth metric → autoscale)
  - PostgreSQL with connection pooling (PgBouncer)
  - Redis Cluster with separate instances per concern
  - Cost: $1500+/mo infrastructure (covered by client revenue)
```

---

## 22. Implementation Roadmap

### Phase 0: Foundation (Current → Week 2)

**Goal:** OpenClaw accepts tasks and executes via Claude Code SDK

```
Week 1:
□ Set up Railway project structure (openclaw-api, postgresql, redis)
□ Implement Task model + TaskRepository
□ Build REST API: POST /tasks, GET /tasks/:id
□ Connect NanoClaw Task Bridge to new API
□ Basic Celery worker that runs a simple Claude Code SDK session

Week 2:
□ Implement mcp-context (simple key-value, no hierarchy yet)
□ Implement mcp-memory (pgvector + basic CRUD)
□ Connect workers to MCP servers
□ End-to-end test: Discord message → task → Claude Code → Discord reply
□ Deploy to Railway
```

**Week 2 Demo:** "Send a Discord message, get an AI response that has access to client context."

---

### Phase 1: Core Intelligence (Week 3-6)

**Goal:** Task classification, LLM routing, and basic HITL

```
Week 3:
□ Task Classifier (Haiku-based, 5 complexity levels)
□ LLM Router (model selection by complexity)
□ Budget Tracker (per-client cost tracking)
□ Basic Langfuse integration

Week 4:
□ Full MCP context hierarchy (Global → Agency → Client → Project)
□ Client profile management
□ Skill registry (mcp-skills) with 5-10 starter skills

Week 5:
□ HITL Manager (pause/resume with Discord reactions)
□ Task planning (sub-task breakdown for complex tasks)
□ Parallel execution (asyncio.gather for independent sub-tasks)

Week 6:
□ Quality scoring (multi-signal)
□ Daily digest generator
□ Alert system (budget, errors, HITL timeouts)
□ Basic Grafana dashboard
```

**Week 6 Demo:** "Complex multi-step task with parallel agents, HITL checkpoint, full observability."

---

### Phase 2: Self-Improvement (Week 7-10)

**Goal:** System improves itself with human oversight

```
Week 7-8:
□ Outcome Tracker (records success/failure with context)
□ Pattern Extractor (identifies improvement opportunities)
□ Skill Proposer (generates improved skills)
□ Discord #skill-proposals channel + review flow

Week 9-10:
□ A/B testing framework for skills
□ Automatic skill promotion (if A/B test shows improvement)
□ Memory lifecycle (decay, confirmation, archival)
□ Cross-client pattern learning (agency memory layer)
```

**Week 10 Demo:** "System proposes a skill improvement, human approves, improvement deployed and validated."

---

### Phase 3: AI Agency Platform (Week 11-16)

**Goal:** Deliver AI systems to clients using OpenClaw

```
Week 11-12:
□ Client Discovery Agent
□ Architecture Generator
□ Delivery Tracker
□ Client-facing reporting (email + Discord)

Week 13-14:
□ First external client onboarding
□ Client portal (basic web UI for context management)
□ Multi-tenant stress testing
□ Security audit (RLS verification, injection testing)

Week 15-16:
□ Pricing model implementation (usage tracking → invoicing)
□ Onboarding documentation
□ Marketing materials (built with OpenClaw, naturally)
□ Second and third external clients
```

**Week 16 Goal:** 3 paying clients, $5k+ MRR, system running with < 2 hrs/week human oversight.

---

### Key Metrics to Track

| Metric | Week 4 Target | Week 8 Target | Week 16 Target |
|--------|---------------|---------------|----------------|
| Tasks/day | 20 | 100 | 500+ |
| Task success rate | 70% | 85% | 92% |
| Avg task cost | $0.15 | $0.10 | $0.07 |
| HITL rate | 30% | 15% | 8% |
| Skills in registry | 10 | 30 | 75+ |
| Active clients | 1 | 3 | 10+ |
| Monthly revenue | $0 | $2k | $10k+ |

---

## 23. What Makes This Different

### vs. Generic LLM Wrappers (ChatGPT plugins, Zapier AI)

| | Generic LLM Wrappers | OpenClaw |
|---|---|---|
| Context | Stateless (each call independent) | Persistent hierarchical context |
| Client isolation | None | Strict RLS + architectural isolation |
| Self-improvement | Manual prompt tweaking | Automated pattern detection + human review |
| Observability | None or basic | Full Langfuse traces + Prometheus + cost/client |
| Cost optimization | Fixed model | Dynamic routing (Haiku → Sonnet → Opus by complexity) |
| HITL | Not designed for it | First-class autonomy level system |

### vs. Enterprise Agent Platforms (Salesforce Einstein, Microsoft Copilot Studio)

| | Enterprise Platforms | OpenClaw |
|---|---|---|
| Cost to build | $5-50M | $50-200k |
| Time to deploy | 12-18 months | 4-8 weeks |
| Customization | Limited to platform primitives | Full code control |
| Pricing | $50-500k/year | $800-5k/month |
| Improvement | Vendor roadmap | Continuous self-improvement |
| White-labeling | No | Yes |

### vs. Raw Claude Code SDK

| | Raw Claude Code SDK | OpenClaw |
|---|---|---|
| Multi-client | Not built-in | Architectural isolation + RLS |
| Persistent memory | Not built-in | Graphiti + pgvector |
| Task queue | Not built-in | Celery + Redis |
| Observability | Not built-in | Langfuse + Prometheus |
| Self-improvement | Not built-in | Full improvement engine |
| Discord interface | Not built-in | NanoClaw integration |
| Agency use case | Not designed for | Core use case |

### The Core Differentiation

OpenClaw is the first AI operating system designed specifically for **small AI agency economics**:

1. **It uses itself** — We build client AI systems using OpenClaw, not for OpenClaw
2. **It improves continuously** — Each task makes the next one better
3. **It's economically honest** — Built to make money at $1,500-5,000/client/month, not $500k/year enterprise contracts
4. **It's observable by design** — Every decision is traceable; nothing is a black box
5. **It respects human judgment** — Calibrated autonomy, not full automation

The goal is not to replace human AI consultants. The goal is to make one human AI consultant as productive as a team of ten.

---

*Last updated: April 2026 | Supersedes: `08-autonomous-agent-platform.md` and `09-enhanced-system-design.md` | [← Hierarchical Context](07-hierarchical-context-architecture.md) | [↑ Back to README](../README.md)*