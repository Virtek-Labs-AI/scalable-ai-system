# Curated GitHub Resources

A curated directory of the most valuable open-source repositories for building a scalable, autonomous AI system. Organized by category with use-case guidance.

---

## Table of Contents

1. [Multi-Agent Frameworks](#1-multi-agent-frameworks)
2. [Memory & Context Management](#2-memory--context-management)
3. [Guardrails & Safety](#3-guardrails--safety)
4. [Observability & Evaluation](#4-observability--evaluation)
5. [Red-Teaming & Adversarial Testing](#5-red-teaming--adversarial-testing)
6. [API Gateways & Infrastructure](#6-api-gateways--infrastructure)
7. [Dashboard & UI Frameworks](#7-dashboard--ui-frameworks)
8. [RAG & Knowledge Retrieval](#8-rag--knowledge-retrieval)
9. [Self-Optimization & Meta-Learning](#9-self-optimization--meta-learning)
10. [Curated Lists & Meta-Resources](#10-curated-lists--meta-resources)

---

## 1. Multi-Agent Frameworks

### LangGraph
**Repo:** https://github.com/langchain-ai/langgraph  
**Stars:** 10k+  
**Use Case:** Build stateful, cyclical multi-agent workflows with persistent memory and human-in-the-loop support.

**Best For:**
- Complex workflows where agents need to loop, branch, and retry
- Systems requiring persistent state across multi-turn sessions
- Supervisor/subgraph patterns (one orchestrator managing many specialists)
- Production-grade agent pipelines with full observability via LangSmith

**Key Examples in Repo:**
- `examples/multi_agent_collaboration.ipynb` — multiple agents working together
- `examples/agent_supervisor.ipynb` — supervisor routing to specialist agents
- `examples/plan_and_execute.ipynb` — planning agent + execution agent pattern

---

### AutoGen / AG2
**Repo:** https://github.com/ag2ai/ag2  
**Stars:** 35k+  
**Use Case:** Conversational multi-agent systems where agents communicate asynchronously via messages.

**Best For:**
- Code-writing and execution workflows (AssistantAgent + UserProxyAgent pattern)
- Research automation (agents debating and refining answers)
- Long-running autonomous tasks with human approval checkpoints

---

### CrewAI
**Repo:** https://github.com/crewAIInc/crewAI  
**Stars:** 25k+  
**Use Case:** Role-based agent teams with defined processes (sequential or hierarchical).

**Best For:**
- Business process automation with clearly defined roles
- Content production pipelines (researcher → writer → editor → publisher)
- When you want fast setup with minimal boilerplate
- Non-engineers who want to define agents in YAML

---

### OpenAI Agents SDK
**Repo:** https://github.com/openai/openai-agents-python  
**Stars:** 8k+  
**Use Case:** Lightweight, production-ready agents built on OpenAI's primitives.

**Best For:**
- Handoffs between specialized agents (triage → specialist)
- Built-in tracing without additional instrumentation
- MCP tool integration out of the box
- Teams already deep in the OpenAI ecosystem

---

### Agency Swarm
**Repo:** https://github.com/VRSEN/agency-swarm  
**Stars:** 5k+  
**Use Case:** Structured agency pattern where agents communicate only through defined "communication flows".

**Best For:**
- Enforcing communication boundaries between agents
- Building agency-style organizations of AI (CEO → managers → workers)
- Rapid custom tool creation with Pydantic schemas

---

### Google Agent Development Kit (ADK)
**Repo:** https://github.com/google/adk-python  
**Stars:** 4k+  
**Use Case:** Google's framework for multi-agent systems with A2A protocol support.

**Best For:**
- Integrating with Google Cloud (Vertex AI, BigQuery, Cloud Storage)
- A2A (Agent-to-Agent) protocol for cross-framework agent communication
- Artifact management for passing data between agents

---

### Kyegomez/Swarms
**Repo:** https://github.com/kyegomez/swarms  
**Stars:** 4k+  
**Use Case:** Enterprise-grade framework focused on massive parallelism and swarm intelligence patterns.

**Best For:**
- Running hundreds of agents in parallel across a task
- Financial analysis, medical research, scientific computing use cases
- Swarm architectures (MixtureOfAgents, AgentRearrange, SpreadsheetSwarm)

---

### MetaGPT
**Repo:** https://github.com/geekan/MetaGPT  
**Stars:** 45k+  
**Use Case:** Software company simulation with agents playing defined roles (PM, Architect, Engineer, QA).

**Best For:**
- End-to-end software development automation from requirements to code
- Understanding how role-based agent collaboration works in practice
- Generating documentation and test cases alongside code

---

## 2. Memory & Context Management

### Mem0
**Repo:** https://github.com/mem0ai/mem0  
**Stars:** 25k+  
**Use Case:** Intelligent memory layer that extracts and stores facts from conversations for personalized AI.

**Best For:**
- Per-user personalization across sessions
- Automatically extracting facts, preferences, and relationships from conversations
- Cross-project shared memory (same user's preferences available to all agents)

**Key Integration:**
```python
from mem0 import MemoryClient
client = MemoryClient(api_key="...")
client.add(messages, user_id="ryan")  # Extracts and stores key facts
results = client.search("user preferences", user_id="ryan")  # Semantic search
```

---

### Zep / Graphiti
**Zep Repo:** https://github.com/getzep/zep  
**Graphiti Repo:** https://github.com/getzep/graphiti  
**Stars:** 3k+ / 2k+  
**Use Case:** Temporal knowledge graph memory — tracks how facts change over time.

**Best For:**
- Long-running agents where facts evolve (e.g., project requirements change)
- Multi-user environments with complex relationship graphs
- When you need to answer "what did we decide about X last month?"

---

### Letta (formerly MemGPT)
**Repo:** https://github.com/letta-ai/letta  
**Stars:** 14k+  
**Use Case:** Agents with persistent memory that survives across sessions using a tiered memory architecture.

**Best For:**
- Long-horizon tasks where context window limits are a problem
- Agents that need to "remember" information from months ago
- Shared memory blocks — multiple agents sharing the same memory object
- Built-in Agent Development Environment (ADE) for visual memory inspection

---

### Chroma
**Repo:** https://github.com/chroma-core/chroma  
**Stars:** 15k+  
**Use Case:** Lightweight, embeddable vector database for RAG and semantic search.

**Best For:**
- Prototyping and development (runs in-process, no server needed)
- Small to medium knowledge bases (< 10M vectors)
- Python-native applications needing fast setup

---

### Qdrant
**Repo:** https://github.com/qdrant/qdrant  
**Stars:** 21k+  
**Use Case:** Production-grade vector search engine written in Rust.

**Best For:**
- Production vector search with high performance requirements
- Multi-vector support (store multiple embeddings per document)
- Payload filtering combined with vector search
- Hybrid search (dense + sparse vectors)

---

### pgvector
**Repo:** https://github.com/pgvector/pgvector  
**Stars:** 13k+  
**Use Case:** PostgreSQL extension for vector similarity search.

**Best For:**
- Teams already using PostgreSQL who want to avoid a new database
- Combining vector search with relational queries in a single DB
- Simpler operational model (one database for everything)

---

## 3. Guardrails & Safety

### NeMo Guardrails
**Repo:** https://github.com/NVIDIA/NeMo-Guardrails  
**Stars:** 4k+  
**Use Case:** Programmable conversational guardrails using Colang DSL.

**Best For:**
- Enforcing dialogue flows and topic restrictions
- Customer-facing agents that must stay on topic
- Integrating fact-checking against a knowledge base

---

### Guardrails AI
**Repo:** https://github.com/guardrails-ai/guardrails  
**Stars:** 4k+  
**Use Case:** Output validation with automatic retry and correction.

**Best For:**
- Enforcing JSON schema on LLM outputs
- PII detection and redaction in responses
- Any workflow requiring structured, validated LLM output

---

### LLM Guard
**Repo:** https://github.com/protectai/llm-guard  
**Stars:** 1k+  
**Use Case:** Security scanner toolkit for LLM I/O.

**Best For:**
- On-premise deployments with no external API calls
- Prompt injection detection
- Compliance scanning of all agent interactions

---

### Presidio
**Repo:** https://github.com/microsoft/presidio  
**Stars:** 3.5k+  
**Use Case:** Microsoft's PII detection and anonymization library.

**Best For:**
- Enterprise-grade PII handling (50+ entity types)
- GDPR/CCPA compliance for AI systems processing user data
- Custom entity recognition for domain-specific identifiers

---

## 4. Observability & Evaluation

### Langfuse
**Repo:** https://github.com/langfuse/langfuse  
**Stars:** 8k+  
**Use Case:** Full observability platform for LLM applications.
**Best For:** Production LLM monitoring, prompt versioning, eval pipelines, cost tracking.

---

### Arize Phoenix
**Repo:** https://github.com/Arize-ai/phoenix  
**Stars:** 4k+  
**Use Case:** AI observability with embedding-native analysis.
**Best For:** Detecting embedding drift, OTel-native tracing, local debugging.

---

### RAGAS
**Repo:** https://github.com/explodinggradients/ragas  
**Stars:** 7k+  
**Use Case:** Automated evaluation metrics for RAG pipelines.
**Best For:** Continuous quality monitoring of retrieval and generation quality.

---

### DeepEval
**Repo:** https://github.com/confident-ai/deepeval  
**Stars:** 5k+  
**Use Case:** LLM unit testing framework compatible with pytest.
**Best For:** CI/CD quality gates, regression testing, multi-turn conversation testing.

---

### TruLens
**Repo:** https://github.com/truera/trulens  
**Stars:** 2k+  
**Use Case:** RAG triad evaluation (relevance, groundedness, coherence).
**Best For:** Systematic RAG quality evaluation with a visual dashboard.

---

### AgentOps
**Repo:** https://github.com/AgentOps-AI/agentops  
**Stars:** 2k+  
**Use Case:** Session-level observability designed specifically for agents.
**Best For:** Replaying agent sessions, multi-framework support, cost per session.

---

### OpenLIT
**Repo:** https://github.com/openlit/openlit  
**Stars:** 1.5k+  
**Use Case:** OTel-native LLM + GPU observability.
**Best For:** Self-hosted model deployments, vendor-neutral observability.

---

### OpenLLMetry / Traceloop
**Repo:** https://github.com/traceloop/openllmetry  
**Stars:** 2k+  
**Use Case:** OpenTelemetry instrumentation for all major LLM SDKs.
**Best For:** Vendor-neutral tracing that exports to any OTLP backend.

---

## 5. Red-Teaming & Adversarial Testing

### PyRIT (Microsoft)
**Repo:** https://github.com/Azure/PyRIT  
**Stars:** 2k+  
**Use Case:** Automated red-teaming for generative AI systems.
**Best For:** Pre-deployment safety testing, finding guardrail bypass thresholds, crescendo attacks.

---

### Garak (NVIDIA)
**Repo:** https://github.com/NVIDIA/garak  
**Stars:** 2.5k+  
**Use Case:** LLM vulnerability scanner with 100+ probe types.
**Best For:** Systematic vulnerability scanning before model deployment, CI integration.

---

### Promptfoo
**Repo:** https://github.com/promptfoo/promptfoo  
**Stars:** 5k+  
**Use Case:** Prompt testing, evaluation, and red-teaming in CI/CD.
**Best For:** Comparing prompt versions, automated red-team generation, GitHub Actions integration.

---

### ARTKIT (BCG X)
**Repo:** https://github.com/BCG-X-Official/artkit  
**Stars:** 700+  
**Use Case:** Async red-teaming toolkit for multi-turn attacks.
**Best For:** High-throughput adversarial testing, augmented attack strategies.

---

## 6. API Gateways & Infrastructure

### LiteLLM
**Repo:** https://github.com/BerriAI/litellm  
**Stars:** 15k+  
**Use Case:** Unified LLM proxy supporting 100+ models with OpenAI-compatible API.
**Best For:** Model routing, budget enforcement, provider failover, Prometheus metrics.

---

### Portkey Gateway
**Repo:** https://github.com/Portkey-AI/gateway  
**Stars:** 6k+  
**Use Case:** AI gateway with guardrails, semantic caching, and analytics.
**Best For:** Reducing costs via semantic cache, built-in request tracing, 250+ model integrations.

---

### Helicone
**Repo:** https://github.com/Helicone/helicone  
**Stars:** 2k+  
**Use Case:** LLM cost monitoring as a proxy (change one URL).
**Best For:** Zero-friction cost tracking, per-user/project attribution.

---

### Ray
**Repo:** https://github.com/ray-project/ray  
**Stars:** 34k+  
**Use Case:** Distributed computing framework for parallel agent execution.
**Best For:** Running hundreds of agents truly in parallel, actor model for stateful agents, Ray Serve for agent deployment.

---

### Celery
**Repo:** https://github.com/celery/celery  
**Stars:** 24k+  
**Use Case:** Distributed task queue for async agent execution.
**Best For:** Simple async task queues with Redis/RabbitMQ, retry logic, scheduled tasks.

---

## 7. Dashboard & UI Frameworks

### Dify
**Repo:** https://github.com/langgenius/dify  
**Stars:** 60k+  
**Use Case:** Visual LLM app builder with workflow editor and knowledge base management.
**Best For:** No-code workflow building, prototyping, non-technical team access to AI system.

---

### Flowise
**Repo:** https://github.com/FlowiseAI/Flowise  
**Stars:** 33k+  
**Use Case:** Drag-and-drop LangChain/LlamaIndex flow builder.
**Best For:** Visual prototyping of RAG and agent pipelines.

---

### Appsmith
**Repo:** https://github.com/appsmithorg/appsmith  
**Stars:** 34k+  
**Use Case:** Open-source low-code internal tool builder.
**Best For:** Building context registry dashboards, HITL approval UIs, agent monitoring panels.

---

### Directus
**Repo:** https://github.com/directus/directus  
**Stars:** 28k+  
**Use Case:** Headless CMS with MCP-native integration for content/context management.
**Best For:** Storing and managing shared agent skills/contexts via a beautiful UI with MCP server access.

---

### Grafana
**Repo:** https://github.com/grafana/grafana  
**Stars:** 65k+  
**Use Case:** Universal observability dashboards.
**Best For:** Combining system metrics, LLM metrics, and cost data in unified dashboards.

---

## 8. RAG & Knowledge Retrieval

### LlamaIndex
**Repo:** https://github.com/run-llama/llama_index  
**Stars:** 36k+  
**Use Case:** Data framework for connecting LLMs to external data sources.
**Best For:** Complex RAG pipelines, agentic RAG, multi-document reasoning, 160+ data connectors.

---

### LangChain
**Repo:** https://github.com/langchain-ai/langchain  
**Stars:** 95k+  
**Use Case:** Comprehensive framework for LLM application development.
**Best For:** Large ecosystem, many integrations, strong community, LangGraph compatibility.

---

### Haystack
**Repo:** https://github.com/deepset-ai/haystack  
**Stars:** 17k+  
**Use Case:** Production-ready NLP/RAG pipelines.
**Best For:** Enterprise document search, pipeline-based architecture, built-in evaluation.

---

### GraphRAG (Microsoft)
**Repo:** https://github.com/microsoft/graphrag  
**Stars:** 20k+  
**Use Case:** Graph-based RAG that extracts a knowledge graph from documents.
**Best For:** Complex document corpora where relationships between entities matter, multi-hop questions.

---

### Chonkie
**Repo:** https://github.com/chonkie-inc/chonkie  
**Stars:** 2k+  
**Use Case:** Fast, lightweight text chunking library for RAG.
**Best For:** Semantic chunking, late chunking, token-aware splitting with multiple strategies.

---

## 9. Self-Optimization & Meta-Learning

### DSPy
**Repo:** https://github.com/stanfordnlp/dspy  
**Stars:** 20k+  
**Use Case:** Framework for algorithmically optimizing LLM prompts and pipelines.
**Best For:** Automatically discovering optimal prompts via a training dataset; replacing hand-crafted prompts with optimized ones.

**Key Concept:** Instead of writing prompts manually, you define the input/output signature and DSPy optimizes the prompts automatically.

```python
class RAGSignature(dspy.Signature):
    """Answer questions using retrieved context."""
    context = dspy.InputField()
    question = dspy.InputField()
    answer = dspy.OutputField()

optimizer = dspy.MIPROv2(metric=ragas_score)
optimized_rag = optimizer.compile(RAG(), trainset=examples)
```

---

### Reflexion
**Repo:** https://github.com/noahshinn/reflexion  
**Stars:** 2k+  
**Use Case:** Agents that improve via verbal reinforcement over episodes.
**Best For:** Tasks where the agent can self-evaluate failure and store lessons learned.

---

### TextGrad
**Repo:** https://github.com/zou-group/textgrad  
**Stars:** 2k+  
**Use Case:** Automatic differentiation for text — backpropagate through LLM pipelines.
**Best For:** Automatically optimizing complex multi-step agent pipelines end-to-end.

---

## 10. Curated Lists & Meta-Resources

### Awesome LLM Apps
**Repo:** https://github.com/Shubhamsaboo/awesome-llm-apps  
**Stars:** 10k+  
**Description:** Curated collection of LLM apps built with RAG, agents, and multi-modal features. Excellent for discovering real-world implementation patterns.

---

### Awesome AI Agents
**Repo:** https://github.com/e2b-dev/awesome-ai-agents  
**Stars:** 12k+  
**Description:** Comprehensive list of AI agent projects, frameworks, and infrastructure tools. Updated regularly.

---

### LLM Course (Hugging Face)
**Repo:** https://github.com/mlabonne/llm-course  
**Stars:** 40k+  
**Description:** Free course covering LLM fundamentals, fine-tuning, quantization, and deployment. Great foundation for team members new to LLMs.

---

### Awesome LangChain
**Repo:** https://github.com/kyrolabs/awesome-langchain  
**Stars:** 7k+  
**Description:** Curated list of LangChain tools, integrations, and example projects.

---

### Awesome RAG
**Repo:** https://github.com/frutik/Awesome-RAG  
**Stars:** 2k+  
**Description:** Curated resources for RAG patterns, papers, tools, and implementations.

---

### OWASP Top 10 for LLMs
**Repo:** https://github.com/OWASP/www-project-top-10-for-large-language-model-applications  
**Description:** Official OWASP security guidelines for LLM applications. Essential reading before deploying any production AI system.

---

### MITRE ATLAS
**Website:** https://atlas.mitre.org  
**Description:** Adversarial threat landscape for AI systems — attack tactics, techniques, and case studies. Use to threat-model your agent system.

---

## Quick Reference: Pick by Use Case

| I need to... | Use this |
|--------------|----------|
| Build stateful multi-agent workflows | LangGraph |
| Run agents at massive scale in parallel | Ray + Kyegomez/Swarms |
| Share skills across all projects | Mem0 + Directus (MCP) |
| Add memory that persists forever | Letta (MemGPT) |
| Track knowledge graph over time | Zep/Graphiti |
| Do production vector search | Qdrant |
| Add output validation/schema enforcement | Guardrails AI |
| Detect prompt injection attacks | LLM Guard + Lakera |
| Evaluate RAG quality automatically | RAGAS + DeepEval |
| Test LLM security before deployment | Garak + PyRIT |
| Route models and enforce budgets | LiteLLM |
| View full agent traces in production | Langfuse + AgentOps |
| Detect embedding drift | Arize Phoenix |
| Build admin UI for context management | Appsmith + Directus |
| Auto-optimize my prompts | DSPy |
| Visualize and build workflows no-code | Dify |
| Monitor infrastructure + AI metrics | Grafana |

---

*Last updated: April 2026 | [← Dashboards & Observability](04-dashboards-and-observability.md) | [System Design →](06-system-design.md)*
