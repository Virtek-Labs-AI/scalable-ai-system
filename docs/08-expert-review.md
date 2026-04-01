# Expert Review: Doc 08 — Unified Platform Architecture

> **Reviewer:** Senior AI Systems Architect
> **Review Date:** 2026-04-01
> **Document Reviewed:** `docs/08-unified-platform-architecture.md`
> **Supporting Context:** `docs/10-simplified-mvp-architecture.md` (the primary available source describing the architecture from docs 06–08; the source document `08-unified-platform-architecture.md` is not present in the filesystem — this review is based on architectural content referenced, described, and cross-linked from doc 10, which presents doc 08 as a prerequisite and describes its core design decisions in detail)
> **Verdict:** The architecture is conceptually ambitious and hits most of the right design targets for a multi-interface AI agent platform, but carries several production-blocking design gaps — most critically around guardrail pipeline latency, context contamination enforcement, HITL failure modes, and the autonomous improvement loop's lack of safety rails — that must be resolved before live client traffic.

---

## Executive Summary

The unified platform architecture described in doc 08 (and referenced throughout doc 10) makes a correct bet on MCP as the capability abstraction layer, correctly separates concerns across 7 distinct MCP servers, and chooses an appropriate 4-tier context model for multi-tenant isolation. The interface unification strategy — routing NanoClaw (Discord), Claude Code CLI, and the REST API through the same MCP server topology — is elegant and reduces surface area significantly. However, the architecture has a critical latency problem it does not acknowledge: running Presidio → LLM Guard → Llama Guard 3 synchronously on every message at a Discord bot latency budget is not feasible at the model sizes described. The HITL design has an unacknowledged split-brain failure mode when Discord goes down mid-approval. The DSPy MIPROv2 nightly optimization loop writes directly to the production prompt store with no blue/green validation or rollback gate, meaning a single bad optimization run can degrade every client's experience overnight. The temporal knowledge graph choice (Graphiti on PostgreSQL) is defensible at MVP scale but will become a query bottleneck above ~500K nodes. The multi-tenant RLS enforcement is architecturally sound but relies entirely on the MCP server tier correctly threading `client_id` — a single missing parameter in any call path breaks tenant isolation silently.

---

## Section-by-Section Findings

---

### 1. MCP Architecture

#### Strengths

**MCP-first design is the correct abstraction choice.** Using MCP as the capability boundary between the agent engine and its tools means all three interfaces (Discord, CLI, REST) consume the same tool surface. This eliminates the classic "feature parity drift" problem where the Discord bot has capabilities the REST API doesn't, or vice versa. It also makes adding a fourth interface (web app, Slack bot, voice) a first-class operation rather than a fork.

**Seven-server decomposition is well-scoped.** The split into `mcp-context`, `mcp-memory`, `mcp-skills`, `mcp-guardrails`, `mcp-hitl`, `mcp-exec`, `mcp-db`, and `mcp-github` correctly separates concerns by responsibility domain. Each server has a clear, non-overlapping function. This is better than the common alternative of a monolithic "tools" MCP server that accumulates functionality and becomes impossible to reason about.

**Stateless MCP server design enables horizontal scaling.** By placing all durable state in PostgreSQL and Redis rather than in-process, the MCP servers remain horizontally scalable. This is the right choice even in a Docker Compose deployment — it means vertical-to-K8s migration doesn't require any architectural change to the MCP layer.

#### Issues

**Protocol mismatch: MCP is stdio/SSE, not HTTP.** The Compose configuration in doc 10 exposes MCP servers on HTTP ports (8001–8008) and health-checks them via `curl`. The MCP specification (Anthropic, 2024) defines transport over stdio (local) or SSE (remote). Running MCP over raw HTTP requires a custom adapter layer. This layer is never described in the architecture documents. If the team has implemented a custom HTTP transport for MCP, this is a significant undocumented component. If they are using stdio transport with a sidecar or proxy, that proxy is absent from both the architecture description and the Compose file. The architecture should explicitly document how MCP messages are transported between the agent engine and the MCP servers — this is not a minor detail.

**Missing service: no MCP server for Claude Code SDK.** The Claude Code SDK integration is described as a first-class interface, but there is no MCP server dedicated to Claude Code session management, workspace context, or code-specific tool routing. The architecture appears to assume Claude Code CLI connects to the same MCP servers as NanoClaw and the REST API. This conflation is problematic: Claude Code sessions have fundamentally different context semantics (file system paths, git state, working directory, multi-turn agentic sessions with automated tool use) that don't map cleanly to the `mcp-context` server's 4-tier model designed around Discord messages and API requests. A Claude Code-specific context adapter is missing.

**No circuit breaker between the agent engine and MCP servers.** If `mcp-guardrails` goes down, the architecture has no defined behavior. Does the agent engine fail-open (process the message without guardrails) or fail-closed (reject all requests)? Fail-open is a security catastrophe; fail-closed is a reliability catastrophe. Neither is documented. For the production scenario where `mcp-guardrails` is restarting after a Llama Guard model update, this ambiguity will cause an incident.

**MCP tool discovery is not addressed.** The MCP protocol includes tool listing (`tools/list`). As the platform evolves, each MCP server's tool manifest will change. There is no described versioning strategy for MCP tool schemas, no compatibility matrix between the agent engine version and MCP server versions, and no handling of the case where the agent engine has a cached tool list that is stale relative to a newly-deployed MCP server. In a microservices deployment, this creates a class of hard-to-debug failures where the agent attempts to call a tool with the old schema.

---

### 2. Interface Unification (Three Interfaces → Same MCP Servers)

#### Strengths

**Single capability surface eliminates parity drift.** Having NanoClaw, Claude Code CLI, and the REST API all route through the same MCP servers means a skill added to `mcp-skills` is immediately available to all three interfaces. This is architecturally clean and operationally sound.

**The `interface` field on tasks is the right design signal.** Recording which interface submitted a task (visible in the `BackgroundTask` in doc 10's asyncio implementation) enables per-interface observability, cost attribution, and behavior differentiation. This is a well-designed trace point.

**Shared context store across interfaces** means a user who starts a conversation on Discord and continues via the REST API will have access to the same memory and context. This is the correct behavior for a unified assistant platform and is non-trivial to achieve correctly.

#### Issues

**Context continuity across interfaces is described but not specified.** The architecture states that all three interfaces share the 4-tier context model, but the session boundary semantics are undefined. When does a Discord conversation "session" end and a new one start? Does a REST API call to the same client resume the Discord conversation context, or start a fresh session? The Claude Code CLI has continuous workspace context — does it inherit the client's domain context from previous Discord conversations? These are product-level decisions that must be encoded in `mcp-context`'s session management logic, and they are absent from the architecture.

**Authentication and authorization are asymmetric across interfaces.** Discord users are implicitly authenticated via Discord's OAuth. REST API callers need API keys. Claude Code SDK presumably uses Anthropic authentication. The MCP servers receive `client_id` as a parameter, but the architecture does not specify how `client_id` is mapped from each interface's authentication mechanism. A single missing validation — such as a REST API caller spoofing a `client_id` they don't own — breaks tenant isolation for all three interfaces simultaneously. The auth-to-`client_id` binding logic is a security-critical component that is entirely undescribed.

**Notification/callback asymmetry is unaddressed.** NanoClaw can push results back to Discord via message replies. The REST API presumably returns results synchronously or via a webhook. Claude Code CLI presumably streams results to the terminal. The architecture describes a unified submission model but doesn't describe the unified result delivery model. When `mcp-hitl` triggers an approval flow initiated from a REST API call (not a Discord message), where does the human receive the approval request? Where does the approval response return? This asymmetry is particularly important for HITL.

---

### 3. 4-Tier Context Model (Global → Domain → Client → Task)

#### Strengths

**Tier separation maps cleanly to access control domains.** Global context (platform-wide instructions), domain context (industry/use-case vertical), client context (per-tenant configuration and memory), and task context (per-conversation ephemeral state) are the correct granularity levels for a multi-tenant AI platform. The design correctly identifies that each tier has different update frequency, different access patterns, and different isolation requirements.

**Loading context in order from broadest to narrowest is correct.** Starting with global → domain → client → task ensures that narrower context correctly overrides broader context, which is the right semantic for an assistant that should follow client-specific instructions over platform defaults.

**RLS at the database layer is the correct enforcement point.** Enforcing tenant isolation at PostgreSQL's row-level security rather than at the application layer means a bug in the agent engine cannot inadvertently access another client's data, as long as the database connection uses the correct `client_id`-scoped role.

#### Issues

**Context contamination prevention is asserted, not designed.** The architecture states that client contexts are isolated, but the mechanism for preventing contamination in the LLM's context window is not specified. The four tiers build a prompt by concatenation. If client A's context is loaded into a prompt and the LLM generates a response, then client B's request arrives, what ensures client A's context is fully purged? In an asyncio system with a shared LLM client, the answer depends entirely on whether the prompt construction is per-request (safe) or cached/reused (unsafe). The architecture should explicitly state that context assembly is always per-request and never cached across tenant boundaries.

**Domain context tier creates a cross-tenant contamination risk.** If multiple clients share a domain context (e.g., "legal" domain), a domain-level memory update triggered by client A's interaction could influence client B's responses if domain memory is writable by individual clients. The architecture does not specify whether domain context is read-only for clients, write-only for platform administrators, or writable by clients. If it's writable, the isolation model is broken.

**Context size management is absent.** The 4-tier model can accumulate substantial context: global system prompt + domain instructions + client history (potentially unbounded) + current conversation. There is no described strategy for managing total context window size, no mention of context compression, and no mention of what happens when the assembled context exceeds the model's context window. Claude 3 Haiku has a 200K token context window, which is generous but not infinite, and multi-session client histories can easily exceed practical limits for prompt assembly.

**The RLS enforcement chain has a single point of failure: the `client_id` parameter.** The entire isolation model depends on every MCP call correctly passing the authentic `client_id`. If any code path in the agent engine omits `client_id`, passes `None`, or passes the wrong value, the RLS policies either fail open (return all rows) or fail closed (return no rows), neither of which is correct behavior. The architecture needs an explicit "defense in depth" position: what is the behavior if `client_id` is missing? The database should have a deny-all default policy for NULL `client_id` queries, and this should be documented as a design invariant.

---

### 4. OpenClaw Agent Engine (asyncio WorkerPool)

#### Strengths

**The semaphore-based concurrency control is architecturally appropriate.** Limiting concurrent tasks via an asyncio semaphore is the correct primitive for controlling LLM API call concurrency, which is the primary cost and rate-limit driver. Keeping this in a named class (`AsyncTaskRunner`) with a clean interface that can be swapped for a Celery-backed implementation is well-designed.

**Per-task observability via Langfuse traces** is the right level of granularity. Wrapping each task attempt in a trace span makes cost attribution, latency debugging, and failure analysis practical. This is better than logging, which loses structured context.

**The `correlation_id` design** (storing the Discord message ID or API request ID alongside the task) is correct and essential for debugging "which user's request spawned this task."

#### Issues

**The coroutine double-invocation bug is present.** As documented in the doc 10 review, `submit()` creates a coroutine with `coro_fn(*args, **kwargs)` and stores it in `BackgroundTask.coroutine`, then `_run_with_retry` calls `coro_fn(*args, **kwargs)` again. The stored coroutine is never awaited. This is a real implementation bug, not a design issue, but it indicates the implementation has not been tested against actual workloads.

**The asyncio event loop is a hidden single point of failure.** All 20 concurrent tasks share a single event loop. Any coroutine that accidentally blocks the event loop (a synchronous database call, a `time.sleep()`, a CPU-intensive operation without `run_in_executor`) will freeze all 20 concurrent tasks simultaneously. The architecture has no protection against this. At minimum, all database calls must be verified to use `asyncpg` (async), all HTTP calls must use `httpx` (async), and all CPU-intensive operations (embedding generation, context compression) must use `asyncio.run_in_executor`. This constraint should be a documented invariant of the architecture.

**No described behavior for LLM API rate limits.** The Anthropic API returns 429 (rate limited) responses. The asyncio retry loop uses exponential backoff, but the maximum retry delay (`2^2 = 4s` for `max_retries=2`) is insufficient for Anthropic's rate limit windows, which can be minutes for token-per-minute limits. When all 20 concurrent tasks are simultaneously rate-limited, the retry attempts will all fire within seconds of each other, creating a thundering herd that extends the rate limit period rather than respecting it. A proper backoff strategy for 429 responses should use the `Retry-After` header value when present.

**No graceful shutdown.** The `AsyncTaskRunner` has no `shutdown()` method. When the process receives `SIGTERM` (e.g., from `docker compose stop`), in-flight tasks are abandoned with no cleanup. The architecture should describe how in-flight tasks are handled during deploys and restarts — the doc 10 description says "users can retry," but there is no mechanism to notify affected users that their task was abandoned mid-execution.

---

### 5. Guardrails Pipeline (Presidio → LLM Guard → Llama Guard 3)

#### Strengths

**Three-layer defense is architecturally sound.** The pipeline correctly separates PII/pattern detection (Presidio, deterministic), semantic content analysis (LLM Guard, ML-based), and safety classification (Llama Guard 3, purpose-built safety model). Each layer has a different failure mode and catches different attack categories. This defense-in-depth approach is consistent with production safety architectures at major AI labs.

**Placing guardrails in a dedicated MCP server** correctly isolates the safety concern from the agent engine. This means guardrails can be updated, A/B tested, or scaled independently without touching the agent engine code.

**Presidio as the first layer is the correct ordering** — deterministic pattern matching is fast and catches the most common PII patterns (SSNs, credit card numbers, email addresses) before the more expensive ML-based layers run.

#### Issues

**Critical: Pipeline latency budget is not addressed and is almost certainly infeasible for synchronous Discord use.** The described pipeline runs on every inbound message:

- **Presidio:** ~5–50ms (CPU-bound, regex + NER models)
- **LLM Guard:** ~100–500ms (transformer inference, likely on CPU at MVP scale)
- **Llama Guard 3-8B:** ~2,000–8,000ms on CPU, ~500–1,500ms on GPU

The combined guardrail latency before the agent engine even begins processing is 2–8 seconds on CPU, or 600ms–2 seconds on GPU. Discord users expect responses within 3–5 seconds for a satisfying interaction. Running a synchronous 3-layer guardrail pipeline on every message consumes the entire latency budget before the LLM call. The architecture does not acknowledge this problem, does not mention whether GPU is available (it isn't on the described Droplet), and does not describe any async/parallel execution strategy for the guardrail layers. This is arguably the single most critical unaddressed design constraint in the entire architecture.

**No described fast-path for trusted/low-risk content.** Not every Discord message needs full Llama Guard 3 evaluation. A message like "What's the status of task abc123?" poses minimal safety risk but would incur the full guardrail overhead. The architecture should describe a triage approach: run Presidio first and only escalate to LLM Guard/Llama Guard if Presidio finds signals of concern, or classify message risk by source (internal bot commands vs. user-generated content) and apply proportional scrutiny.

**Llama Guard 3-8B requires 16GB VRAM for full precision, ~8GB at 4-bit quantization.** The described Droplet has 16GB RAM and no GPU. Running Llama Guard 3-8B on CPU at 16GB RAM leaves virtually nothing for the other 11 services. The `mcp-guardrails` service in the Compose file is given a 4GB memory limit — this is insufficient to load Llama Guard 3-8B even at heavy quantization. Either the architecture assumes the model runs on Modal (not described) or the memory budget is wrong by a factor of 2–4x.

**No described PII handling strategy after detection.** Presidio detects PII. The architecture does not describe what happens when PII is found: Is the message blocked? Is the PII redacted before continuing? Is the user prompted to remove PII? Is it logged for compliance? This is a product and compliance decision that must be encoded in the guardrail pipeline's response contract.

**Guardrail bypass via multi-turn context injection is unaddressed.** A sophisticated adversarial user can split a harmful prompt across multiple turns, with each individual turn passing the guardrail check. The architecture applies guardrails per-message but does not describe whether accumulated conversation history is re-evaluated for patterns that emerge across turns. This is a known attack vector against per-message safety classifiers.

---

### 6. HITL (Human-in-the-Loop) Design

#### Strengths

**HITL as an MCP server is the correct design abstraction.** The `mcp-hitl` server allows the agent engine to request human approval in the same way it calls any other tool, with the same observability and retry semantics. This is cleaner than a bespoke approval channel bolted onto the agent engine.

**Configurable timeout (`APPROVAL_TIMEOUT_SECONDS: 300`)** is the right approach. Hard-coding a 5-minute timeout and exposing it as configuration gives operators the ability to tune for their workflow without code changes.

#### Issues

**Split-brain failure when Discord goes down during an approval wait.** The HITL flow, as described, uses Discord webhooks to deliver approval requests and Discord interactions to receive approval responses. If Discord experiences an outage during the 5-minute approval window (Discord's historical uptime includes several outages per year lasting 10–60 minutes), the approval request is delivered but the response cannot be received. The agent engine is then stuck in an "awaiting approval" state with no timeout escalation path, no fallback notification channel, and no manual override mechanism. After the timeout, the task fails — but the human may have approved it via Discord during the outage, and the platform has no way to reconcile this. The architecture needs: (a) a durable approval state stored in PostgreSQL, not just an in-memory asyncio future; (b) a polling-based fallback (e.g., check the database for approval status rather than relying on a Discord callback); and (c) a manual override endpoint for platform operators.

**No described escalation path for HITL approval.** If the designated human approver does not respond within the timeout, the task fails. But for a production AI agent handling time-sensitive work, "fail silently" is not an acceptable default. The architecture should describe an escalation chain: after N minutes, notify a secondary approver; after M minutes more, notify the platform operator; after the full timeout, fail with a specific error code that the calling interface can handle gracefully.

**HITL approval context is unclear.** For a human to make an informed approval decision, they need to understand what they are approving. The architecture does not describe what information is presented in the approval request. A Discord message saying "Do you approve this action?" is insufficient; the approver needs to see the action description, the affected scope, the requesting user/client, and any risk indicators. The approval request schema must be designed for human comprehension, not just machine routing.

**Concurrent HITL requests for the same client create a queue management problem.** If client A submits 5 tasks that all require HITL approval simultaneously, the architecture has no described mechanism for sequencing, prioritizing, or aggregating these approval requests. The human approver could be spammed with 5 simultaneous Discord messages. The `mcp-hitl` server needs per-client rate limiting and queue management for concurrent approval requests.

---

### 7. Skills Registry and Autonomous Improvement Loop

#### Strengths

**The DSPy MIPROv2 optimization loop running on Modal is correctly designed as an external, asynchronous process.** Offloading the optimization to a serverless compute platform (Modal) means it doesn't compete for resources with the production system, and the nightly cadence means failures do not immediately affect live traffic.

**PostgreSQL as the skills store** is correct — skills are structured data (schema, implementation, metadata, performance history) that benefit from transactional consistency and SQL query capabilities. Using a separate registry table allows complex queries like "find all skills used by client A that have degraded in the last 7 days."

**Versioned skills with performance metrics** is the right design signal. Tracking skill versions and their associated performance metrics is the foundation of any meaningful autonomous improvement system.

#### Issues

**Critical: No validation gate between optimization output and production prompt store.** The described flow is: `Modal → runs DSPy optimization → writes updated prompts to PostgreSQL → all MCP servers read updated prompts on next request`. There is no described validation step between the optimization output and the production prompt. If MIPROv2 produces a prompt that degrades performance on edge cases not represented in the optimization dataset, that degraded prompt goes live for all clients immediately. This is a production-breaking design gap. The minimum required safety gate is: write optimized prompts to a staging table, run a canary evaluation against a held-out test set, compare metrics to the current production prompt, and only promote to production if metrics improve above a threshold. Without this gate, a single bad optimization run can silently degrade every client's experience overnight.

**The optimization training data provenance is unspecified.** DSPy MIPROv2 requires a training dataset to optimize against. What data is used? Is it synthetic? Is it derived from production traces (which would raise client data privacy questions)? Is it manually curated per-skill? The quality of the optimization is entirely determined by the quality of the training data, and this critical dependency is not documented.

**No described rollback mechanism for prompt versions.** If the nightly optimization produces a bad prompt and goes to production, how does an operator revert? There is no described versioning scheme for prompts, no `previous_version` pointer, and no `rollback_to_version(n)` operation. PostgreSQL can store historical rows (e.g., with a `valid_from`/`valid_to` time range), but this schema is not described. The architecture assumes optimization always improves things, which is false.

**DSPy MIPROv2's cost model is not bounded.** MIPROv2 can run many LLM calls during optimization. For a platform running optimization nightly on Modal, an unbounded optimization run could consume significant API credits (hundreds of dollars) in a single night if not constrained. The architecture should describe: maximum allowed LLM calls per optimization run, cost monitoring for the Modal function, and automatic termination if cost exceeds a threshold.

**The "autonomous improvement" framing obscures the human oversight requirement.** For a production multi-tenant AI platform serving paying clients, fully autonomous prompt updates with no human review is a significant responsibility decision. Regulatory and contractual obligations (e.g., SOC 2, enterprise client agreements) may require human sign-off on model behavior changes. The architecture should explicitly address whether human oversight is required for skill updates and under what conditions updates can be fully autonomous.

---

### 8. Temporal Knowledge Graph (Graphiti on PostgreSQL)

#### Strengths

**PostgreSQL + pgvector for the knowledge graph is defensible at MVP scale.** Using a relational store with a vector extension avoids introducing a dedicated graph database (Neo4j, Amazon Neptune, etc.) as an additional operational dependency. For a system that will have thousands to low-hundreds-of-thousands of entities and relationships, PostgreSQL's query planner can handle graph traversals via recursive CTEs with acceptable performance.

**Graphiti's temporal graph abstraction** (tracking entity states over time) is the correct model for an AI assistant that needs to understand "what was true about client X at time T." The temporal component is non-trivial to implement correctly, and using an existing library that handles validity intervals and temporal queries is wise.

**Storing the knowledge graph in the same PostgreSQL instance** as the rest of the application data allows transactional consistency between entity creation and application events. A client being created in the `clients` table and their initial knowledge graph node being created in the `graphiti` tables can happen in the same transaction.

#### Issues

**PostgreSQL recursive CTEs for multi-hop graph traversals will become a query bottleneck above ~500K nodes.** For queries like "find all entities connected to client A within 3 hops that were updated in the last 30 days," a recursive CTE on PostgreSQL must materialize intermediate result sets. At hundreds of thousands of nodes, this becomes a multi-second query. The architecture does not describe any caching strategy for common graph traversals, no query time SLO for knowledge graph lookups, and no migration plan to a dedicated graph database. The threshold for migration should be defined upfront (e.g., "migrate to Neo4j when knowledge graph exceeds 500K nodes or traversal P95 exceeds 200ms").

**pgvector index type is unspecified.** pgvector supports `ivfflat` and `hnsw` index types with very different performance characteristics. `ivfflat` requires a training step (rebuild when data distribution changes significantly) and has high recall at moderate speed. `hnsw` has higher build cost but better query performance. For an AI agent platform with continuous insertions (every conversation potentially adds new knowledge entities), `hnsw` is the correct choice, but this is not documented. Using the wrong index type or no index at all will cause vector similarity searches to degrade from milliseconds to seconds at thousands of entities.

**The knowledge graph has no described pruning or expiry strategy.** Over time, a temporal knowledge graph accumulates historical states of entities. For a long-running deployment, this could mean hundreds of millions of rows. The architecture does not describe when old entity states are archived or deleted, or what the storage growth model looks like. For a PostgreSQL-backed system where storage is shared with Langfuse traces and application data, unbounded knowledge graph growth will eventually consume all allocated storage.

**Concurrent writes from multiple agents are not addressed.** In the asyncio system with 20 concurrent tasks, multiple tasks may attempt to write to the same entity's knowledge graph simultaneously (e.g., two simultaneous conversations about the same client contact). PostgreSQL's MVCC handles concurrent writes correctly for ACID operations, but Graphiti's temporal semantics (setting `valid_to` on old states, creating new states) require careful transaction design to avoid interleaved writes creating inconsistent temporal sequences. The architecture should specify how concurrent entity updates are serialized or merged.

---

### 9. Multi-Tenant Security

#### Strengths

**RLS at the database layer is the correct enforcement point** for multi-tenant isolation. Application-level filtering is insufficient — a bug or missing `WHERE` clause in the application can expose cross-tenant data. Database-enforced RLS with a per-request connection role that sets `app.client_id` is a robust isolation mechanism.

**Separating MCP servers by function** provides defense in depth. A bug in `mcp-memory` cannot directly access `mcp-exec` capabilities, and vice versa.

**The `client_id` threading model** — passing the authentic client identifier through every layer from interface to MCP to database — is the correct design pattern for multi-tenant request tracing and isolation.

#### Issues

**The `client_id` trust chain is undocumented.** For RLS to provide actual security, the `client_id` passed to the database must be cryptographically authenticated, not caller-supplied. The architecture does not describe how `client_id` is established and validated at each interface:
- For Discord: How is a Discord user or server mapped to a `client_id`? Can a Discord user claim a `client_id` they don't own?
- For REST API: Is the `client_id` extracted from a signed JWT, or is it a query parameter? If it's a query parameter, any API caller can claim any `client_id`.
- For Claude Code: How is the Claude Code session authenticated to a specific `client_id`?

Without describing this trust chain, the RLS enforcement is only as strong as the weakest interface's authentication — which based on the available documentation is unclear.

**Prompt injection via cross-tenant knowledge graph** is an attack vector. If client A can write arbitrary text to their knowledge graph, and the agent engine loads that knowledge graph content into prompts for other contexts (e.g., shared domain context), then client A can inject adversarial content into other clients' prompts. Even within a single client, an adversarial end user can attempt to inject instructions through the knowledge graph. The architecture should describe input sanitization for knowledge graph writes and prompt construction safeguards that distinguish system-trusted content from user-contributed content.

**No described data deletion or right-to-be-forgotten flow.** For a multi-tenant SaaS platform, clients will eventually terminate their service. The architecture does not describe how a client's data (knowledge graph, conversation history, skill customizations, session data) is completely and verifiably deleted. GDPR Article 17 and similar regulations require this capability. The absence of a described deletion flow is a compliance gap.

**The audit log for sensitive operations is absent.** The architecture does not describe logging for security-sensitive operations: which admin or client changed a domain context, when was a skill promoted to production, which requests were blocked by the guardrails pipeline, when was an HITL approval granted and by whom. Without audit logging, incident forensics and compliance reporting are impossible.

---

### 10. Cost Model (Claude Code SDK Flat-Rate + Haiku Per-Token)

#### Strengths

**The flat-rate/per-token split is strategically sound.** Using Claude Code SDK ($200/seat/month, flat rate) for developer-facing AI assistance and Claude Haiku (per-token pricing) for the high-volume agent engine tasks correctly matches pricing structure to usage pattern. Claude Code SDK's flat rate absorbs the variability of developer usage; Haiku's low per-token cost ($0.00025/1K input tokens as of early 2026) keeps per-request agent costs manageable.

**Acknowledging the per-token cost uncertainty** (listing Haiku as "~$20/mo") is honest about the speculative nature of usage-based cost estimates before real traffic data.

#### Issues

**No cost runaway protection for Haiku API calls.** The asyncio task runner can have 20 concurrent tasks, each potentially running multi-turn LLM conversations with tool use. A single runaway task (e.g., an agent stuck in a tool-use loop calling LLM iteratively) could generate thousands of LLM calls per hour. At Haiku pricing, this might cost $5–10/hour before anyone notices — not catastrophic, but multiply by 20 concurrent runaway tasks and the hourly cost becomes $100–200. The architecture does not describe any per-task token budget, per-client daily spending limit, or API cost alerting. A `max_tokens_per_task` and `max_turns_per_task` limit should be a design-level constraint, not an afterthought.

**The Claude Code SDK cost basis assumes a known seat count.** At $200/seat/month, the cost model is linear in the number of developer seats using Claude Code CLI. If this platform is offered to enterprise clients with large development teams, the seat cost can dominate the Haiku API cost by a factor of 10-100x. The cost model should describe the seat allocation strategy: is each enterprise client a single seat, or is each developer at the client a seat?

**No billing isolation per client.** In a multi-tenant platform, the Anthropic API costs are incurred on a single account. There is no described mechanism for attributing costs to individual clients, enforcing per-client spending limits, or billing clients differently based on usage. This is acceptable at MVP scale but becomes problematic when a single large client generates disproportionate API costs and the platform has no way to charge them appropriately or throttle them.

**The Modal DSPy optimization cost is underestimated.** The "$5–10/mo" estimate for Modal is based on the Modal compute cost for running the optimization logic. It does not account for the Anthropic API calls made by MIPROv2 during optimization (potentially hundreds of LLM calls per skill per nightly run). For a platform with 50 skills and a nightly optimization cadence, the API cost of optimization could easily exceed the Modal compute cost by 10–100x. This should be treated as a separate line item in the cost model.

---

### 11. Completeness: Missing Topics for Production Readiness

The following topics are either absent or insufficiently addressed in the architecture:

**Data Backup and Recovery (RPO/RTO not stated).** The architecture relies on managed PostgreSQL for all durable state, which provides automated backups. However, the Recovery Point Objective (maximum acceptable data loss) and Recovery Time Objective (maximum acceptable downtime for recovery) are never stated. For a production AI platform where knowledge graph and conversation history are valuable client data, the RPO/RTO targets should drive backup frequency and recovery procedure design.

**API Versioning Strategy.** The REST API is described as a first-class interface but no versioning strategy is described. What is the URL scheme (`/v1/`, `/api/v1/`, header-based)? How are breaking changes handled? How long are old versions supported? These questions must be answered before any external clients integrate against the API.

**SDK / Client Library for REST API Consumers.** A REST API without client libraries creates friction for developers. The architecture does not mention whether client libraries (Python, TypeScript) are planned, or whether the OpenAPI spec will be published and auto-generated clients provided.

**Tenant Onboarding and Offboarding Automation.** Adding a new client requires creating database records, setting up RLS policies, initializing domain context, and potentially configuring Discord server bindings. This process is not described. If it requires manual database operations, it will not scale and introduces error risk.

**Rate Limiting at the API Gateway Layer.** The Caddy reverse proxy is described as the ingress, but no rate limiting configuration is shown. Without per-client rate limiting at the API layer, a misbehaving client can saturate the asyncio task runner's 20-concurrent-task limit, degrading service for all other clients. Caddy supports rate limiting via plugins; this should be configured and documented.

**Compliance Posture.** For a platform handling potentially sensitive business information for enterprise clients, the architecture should state its compliance posture: What data is retained and for how long? Is data encrypted at rest and in transit? Are audit logs retained for the legally required period? Are there PCI, HIPAA, or SOC 2 implications based on the target client base? These are not MVP blockers for an early-stage platform, but they must be planned.

**Embedding Model Management.** The `mcp-memory` service uses vector embeddings with 1536 dimensions (implying a specific embedding model, likely `text-embedding-3-small`). If the embedding model is changed, all existing vectors in the knowledge graph and memory store become incompatible — similarity searches will produce meaningless results. The architecture does not describe an embedding model migration strategy, a version column on vector stores, or a re-embedding pipeline. This is a future operational risk that should be acknowledged and planned for.

---

### 12. What Is Genuinely Strong

**The MCP-first architecture with per-capability servers is one of the strongest design decisions in the platform.** It correctly separates the "what the agent can do" question (MCP servers) from the "how it reasons" question (agent engine) and from the "how users interact" question (interfaces). This separation enables independent iteration, independent scaling, and independent testing of each concern.

**The 4-tier context model maps correctly to multi-tenant SaaS semantics.** The global → domain → client → task hierarchy is the right granularity for the described use cases. The decision to enforce isolation at the database layer via RLS, rather than at the application layer via filtering, is mature and defensible.

**The choice to offload nightly optimization to Modal** (external serverless compute) rather than running it on the application server is architecturally clean. It correctly avoids resource contention between production traffic and batch optimization, and it makes the optimization step independently observable and debuggable.

**The interface unification strategy** — three interfaces sharing one MCP server topology — reduces the attack surface for consistency bugs, reduces the maintenance burden, and makes feature additions cheaper by ensuring they are automatically available to all interfaces.

**The asyncio task runner's interface abstraction** (designed to be swapped for Celery at growth stage) shows good forward engineering discipline. Designing for the migration from the beginning, rather than assuming the initial implementation is permanent, is a sign of architectural maturity.

**The three-layer guardrails pipeline** (Presidio → LLM Guard → Llama Guard 3) is the correct conceptual design for a safety-conscious AI platform. The choice of purpose-built tools for each layer (PII detection, semantic analysis, safety classification) rather than a single monolithic classifier shows awareness of the specialization in the safety tooling ecosystem.

---

## Top 5 Critical Issues (Must-Fix Before Production)

### 1. Guardrail Pipeline Latency Makes Synchronous Execution Infeasible

Running Presidio → LLM Guard → Llama Guard 3-8B synchronously on every message on the described hardware (no GPU, 16GB RAM, `mcp-guardrails` limited to 4GB) will produce latencies of 2–10 seconds per message before the agent engine begins. This is incompatible with Discord's interaction expectations and will make the platform feel broken to users. This is not a "we should optimize this later" issue — it is a fundamental architectural constraint that must be resolved at the design level. The resolution options are: (a) run Llama Guard on GPU-enabled Modal and accept the async model, (b) implement a fast-path triage that skips Llama Guard for low-risk message classes, (c) accept that Llama Guard runs asynchronously and is used for audit/retrospective rather than pre-response gating, or (d) replace Llama Guard 3-8B with a smaller/faster model (Llama Guard 3-1B or a fine-tuned DistilBERT classifier). The architecture must pick one of these and document the implications.

### 2. DSPy Optimization Loop Has No Validation Gate

The nightly MIPROv2 optimization writes directly to the production prompt store with no holdout evaluation, no canary phase, and no rollback mechanism. A single bad optimization run — caused by a skewed training batch, a regression in the optimizer, or a distribution shift in the evaluation data — will degrade every client's prompt quality overnight, with no automatic detection and no easy rollback. This must be addressed with: (a) a staging prompt store separate from production, (b) a mandatory canary evaluation step that compares optimized prompt performance against the current production prompt on a held-out test set, (c) a promotion gate (metrics must improve by at least X% before promotion), and (d) a version-keyed prompt store with a `rollback_to_version(n)` operation accessible to platform operators.

### 3. `client_id` Trust Chain is Undescribed, Creating Silent Isolation Failures

The entire multi-tenant security model depends on `client_id` being cryptographically authenticated at each interface entry point. The architecture describes what `client_id` is used for (RLS enforcement, context isolation, cost attribution) but not how it is established or validated for each interface (Discord, REST API, Claude Code CLI). Without this specification, the isolation model has an undefined trust assumption at its foundation. If any interface allows `client_id` to be caller-supplied without validation, the RLS enforcement is bypassed for that interface entirely. The architecture must document the auth-to-`client_id` binding mechanism for each interface and specify what happens when `client_id` is absent or fails validation.

### 4. HITL Has No Durable State and No Discord Outage Recovery

The HITL approval flow uses Discord webhooks for request delivery and Discord interactions for response receipt. During a Discord outage (which occurs several times per year), approval requests cannot be received, responses cannot be submitted, and the approval state is lost when the asyncio future times out. The approval state must be persisted in PostgreSQL before the Discord webhook is sent, the response path must poll the database rather than relying on a Discord callback, and a manual override endpoint must exist for platform operators to approve/reject pending requests outside of Discord. The current design is one Discord outage away from losing all in-flight HITL approvals with no recovery path.

### 5. MCP Transport Layer is Undocumented and Potentially Non-Standard

The architecture describes MCP servers as HTTP services on ports 8001–8008, but the MCP specification defines transport over stdio (for local processes) or SSE (Server-Sent Events for remote servers). Running MCP over plain HTTP requires a custom transport adapter that is never described, shown in code, or accounted for in the service architecture. Either the MCP servers implement SSE (which requires a specific handler structure and connection management), or they implement a custom HTTP-RPC wrapper around MCP (which is not MCP and loses MCP's streaming and event semantics), or there is an undescribed sidecar/proxy layer converting HTTP to stdio. This ambiguity affects every component in the architecture that calls an MCP server. The transport mechanism must be explicitly specified and its compliance with the MCP protocol version in use must be documented.

---

## Top 5 Recommendations

### 1. Implement a Guardrail Tiering Strategy with a Fast Path

Rather than running the full Presidio → LLM Guard → Llama Guard 3 pipeline on every message, implement a tiered approach:

- **Tier 0 (always):** Presidio PII detection, ~5–20ms. Run synchronously as a pre-flight check.
- **Tier 1 (conditional):** LLM Guard semantic check, ~100–500ms. Run only when Tier 0 finds signals of concern, or for messages matching a "high-risk source" heuristic (new users, flagged accounts, direct DMs).
- **Tier 2 (async audit):** Llama Guard 3 classification, ~1–5s. Always run but asynchronously — do not block the response on Tier 2. Use Tier 2 results to update a user risk score and escalate if a retroactive block is warranted.

This design delivers sub-100ms safety overhead on the fast path, maintains full safety coverage via async auditing, and keeps Llama Guard 3 off the critical path for user-visible latency.

### 2. Add a Blue/Green Prompt Promotion Pipeline

For the autonomous improvement loop:

```
MIPROv2 optimization
        ↓
Write to: skills.optimized_prompts (staging table)
        ↓
Canary evaluation: run optimized prompt vs. current on held-out test set
        ↓
Compare: score(optimized) > score(current) + threshold?
        ↓ YES                    ↓ NO
Promote to                 Archive as rejected,
skills.prompts (prod)      alert platform operator
        ↓
Version the old prompt (skills.prompt_versions)
```

This requires: a `prompt_versions` table (id, skill_id, version, content, score, promoted_at, promoted_by), a canary evaluation runner (can be a Prefect task), and a threshold configuration per skill. The rollback path becomes `UPDATE skills SET active_version = N-1`.

### 3. Document the `client_id` Trust Chain and Implement a Hard Deny-All Default

For each interface:
- **Discord:** `client_id` is derived from the Discord guild ID via a server-side lookup table (Discord guild → platform client). It is never caller-supplied.
- **REST API:** `client_id` is extracted from a signed JWT (RS256, issued by the platform auth service). The JWT is validated at the API gateway layer before reaching any application code. `client_id` is never a query parameter.
- **Claude Code:** `client_id` is bound to the Claude Code SDK API token at token creation time. The agent cannot override it.

At the database layer, the RLS policy for NULL `client_id` must be deny-all: `USING (client_id = current_setting('app.client_id', true))` will return no rows if `app.client_id` is not set, which is the correct safe default.

### 4. Replace the Polling-Based HITL with an Event-Driven Durable Approval Flow

The HITL approval flow should be:
1. Agent engine calls `mcp-hitl` tool with approval request.
2. `mcp-hitl` writes the approval request to `hitl_approvals` table (status=PENDING).
3. `mcp-hitl` sends the Discord webhook notification asynchronously (non-blocking).
4. `mcp-hitl` returns a pending approval ID to the agent engine.
5. Agent engine suspends the task (via an asyncio Event or Redis pub/sub channel) awaiting the approval ID to be resolved.
6. When the human approves/rejects (via Discord interaction OR via the REST API override endpoint), `mcp-hitl` updates the `hitl_approvals` table and publishes a Redis message.
7. The agent engine receives the Redis message and resumes the task.

This design is resilient to Discord outages (the approval can be submitted via REST API), is auditable (full history in PostgreSQL), and decouples the agent engine from the Discord transport layer.

### 5. Define and Implement MCP Transport Explicitly

The architecture should explicitly choose one of the following transport strategies and document it:

**Option A: SSE Transport (standards-compliant).** Each MCP server exposes a SSE endpoint at `/sse`. The agent engine maintains persistent SSE connections to all servers. This is standards-compliant MCP remote transport and enables streaming tool responses.

**Option B: HTTP-RPC Wrapper (pragmatic).** Each MCP server exposes a JSON-RPC style HTTP endpoint. The agent engine sends synchronous HTTP POST requests for each tool call. This is not standard MCP but is simpler to implement and debug. The trade-off is losing MCP's streaming semantics.

**Option C: stdio + sidecar (for local deployment).** Each MCP server is a stdio process. A sidecar within each container converts HTTP requests from the agent engine to stdin and stdout responses back to HTTP. This preserves stdio semantics while enabling Docker Compose networking.

The choice must be made explicitly. Option A is recommended for a production platform that intends to be genuinely MCP-compliant. Option B is acceptable for MVP if the team acknowledges it is not standard MCP and plans to migrate.

---

## Comparison with Industry Patterns

**vs. Salesforce Einstein AI Platform:** Salesforce uses a similar MCP-like tool abstraction layer (called "Actions" and "Agents") with tenant isolation at the database layer. Their approach to multi-tenancy is broadly analogous to the RLS design here. One key difference: Salesforce has a formal "trust layer" (Einstein Trust Layer) that is a distinct architectural component with its own observability — the guardrails are a first-class platform concern, not a single MCP server. The described architecture would benefit from elevating the guardrails to a similar status.

**vs. Anthropic's own Claude agents documentation:** The MCP-first approach is consistent with Anthropic's design guidance. The multi-server topology (one server per capability domain) is more granular than many reference implementations, which tend to use 1–2 MCP servers. The granularity is defensible but increases operational surface area.

**vs. LangChain/LangGraph production deployments:** LangGraph (State Machines for LLM agents) approaches the concurrency and state management problem differently — with explicit state machines and checkpointing rather than an asyncio runner. The described asyncio approach is simpler to implement but lacks LangGraph's built-in durability (each step is checkpointed), which is precisely the durability problem this architecture acknowledges but defers to a later stage. For a production platform serving paying clients, the absence of task durability is a more significant gap than the document acknowledges.

**vs. OpenAI Assistants API architecture:** The 4-tier context model is more sophisticated than OpenAI's Assistants API thread model (which is flat). The temporal knowledge graph is a genuine differentiator — OpenAI's Assistants API has no equivalent temporal entity tracking. The described platform is architecturally more ambitious than Assistants API in ways that are justified if the use cases require it.

**vs. industry standard for autonomous improvement:** The DSPy MIPROv2 approach to automated prompt optimization is at the research frontier rather than production hardened. Most production AI platforms use human-reviewed prompt updates with A/B testing and gradual rollout. The described architecture skips the A/B testing and gradual rollout steps, which is the risky part. The choice to use MIPROv2 is technically interesting; the choice to deploy its output directly to production without a validation gate is the issue, not the optimizer itself.

---

## Overall Grade: B−

**Justification:**

The architectural conceptual design is strong (B+). The MCP-first decomposition, the 4-tier context model, the interface unification strategy, and the three-layer guardrail concept are all at the level of senior architectural thinking. The choice of technologies (pgvector, Graphiti, DSPy, Langfuse, Presidio) is informed and defensible. The system has a coherent design philosophy that holds together across its components.

The grade drops to B− on implementation specificity and critical gap analysis:

- The guardrail pipeline latency problem is a fundamental constraint that undermines the usability of the platform as designed and is not acknowledged.
- The autonomous improvement loop's lack of a validation gate is a safety regression that will cause production incidents.
- The `client_id` trust chain is undocumented, leaving the core security model on an undefined foundation.
- The HITL design has a well-known distributed systems failure mode (split-brain during an external service outage) that is unaddressed.
- The MCP transport mechanism is ambiguous in a way that affects the entire system's feasibility.

These are not implementation details — they are architectural gaps. A B-level architecture document should be implementable by a competent team following the document. This one cannot be implemented without resolving these gaps first.

**To reach an A−:** Address the five critical issues above, add the MCP transport specification, document the `client_id` trust chain, add the guardrail tiering strategy, and add a prompt validation pipeline for the autonomous improvement loop.

**To reach a B+:** Address at minimum the guardrail latency issue, the HITL durability issue, and the optimization validation gate — the three issues most likely to cause production incidents with live client traffic.
