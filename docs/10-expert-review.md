# Expert Review: Doc 10 — Simplified MVP Architecture

> **Reviewer:** Senior AI Systems Architect
> **Review Date:** 2026-04-01
> **Document Reviewed:** `docs/10-simplified-mvp-architecture.md`
> **Supporting Context:** docs 06–09 (referenced; docs 06–09 not present in filesystem, analysis based on doc 10 content and cross-references)
> **Verdict:** The design is directionally correct but has several production-blocking issues and one category-level security flaw that must be resolved before going live.

---

## Executive Summary

This document makes a sound strategic call: skip Kubernetes for MVP and use Docker Compose on a single Droplet. The cost and complexity savings are real, and the architecture preserves the right abstractions for future migration. However, the document contains at least one critical security flaw (the `mcp-exec` sandbox is aspirational, not implemented), two technically broken patterns (the asyncio task runner has a coroutine double-invocation bug and the `_tasks` dict is an unbounded memory leak), and significantly underestimates the operational burden of a single-node, no-durability system for a paying-client service. The risk matrix classifies the SPOF as "Low Probability" — this is directionally defensible for hardware failure but ignores the far more common software-level SPOFs (OOM kills, bad deploys, runaway containers) that will hit this architecture regularly at MVP scale. The $171/mo cost estimate is real but incomplete: it omits bandwidth overages, Langfuse storage growth, and snapshot costs. This document should be revised to fix the code bugs, harden the security section, and expand the operational runbook before any production deployment.

---

## Section-by-Section Findings

### Section 1–2: Motivation and Core Principles

**Strengths:**
- The decision to defer Kubernetes is explicitly justified with cost and complexity numbers, not vague hand-waving. The $48/mo savings framing is clear.
- The principles table (MCP-first, RLS, 4-tier context, etc.) correctly signals what is and is not changing. This prevents scope creep in implementation.
- Framing this as "deliberate starting point" rather than permanent simplification is the right narrative.

**Issues:**
- The motivation section claims setup time is "~4 hours" but the Quick-Start Checklist (§17) doesn't account for the time to build and push 11 container images for the first time, configure all managed services, troubleshoot DNS propagation, or debug initial health check failures. A first-time deployer should expect 1–2 days, not 4 hours.
- The table says "asyncio concurrency ✅ Same WorkerPool implementation" — this is misleading. The WorkerPool being the same doesn't mean the durability characteristics are the same. The footnote about Celery being removed is the important part and should be the headline, not buried in §3.2.

---

### Section 3: What Changes

**Strengths:**
- The tables clearly communicate what is being traded away. The explicit acknowledgment of task durability loss ("tasks lost on crash") is honest.
- The Celery → asyncio trade-off section is well-scoped: it correctly identifies the failure mode (crash = lost task) and states the mitigation (user retries).

**Issues:**
- The claim "asyncio.shield + manual retry loop" as the replacement for Celery's `autoretry_for` is listed in a table but the actual `asyncio.shield` usage is absent from the implementation in §6. `asyncio.shield` protects a coroutine from cancellation — it does not provide retry semantics. This is a misleading entry in the comparison table.
- "GitHub Actions secrets → `.env` via SSH" for secrets management lists as a simple entry, but it creates a specific vulnerability: the `.env` file on disk contains all secrets in plaintext. This is noted in §9 but not called out as a risk in the trade-offs table.

---

### Section 4: Infrastructure Topology

**Strengths:**
- The topology diagram is clear and accurate. External services (Modal, Anthropic, Discord) are correctly shown as external.
- Using managed PostgreSQL and Redis instead of containerized instances is the right call — it correctly pushes HA and backup responsibility to DigitalOcean.

**Issues:**

**Cost estimate is incomplete.** The $171/mo baseline omits:
- **DigitalOcean Spaces bandwidth:** Spaces includes 250GB transfer free; beyond that it's $0.01/GB. At MVP scale this is fine, but the estimate should state this assumption.
- **Droplet snapshot storage:** Automated snapshots for disaster recovery (essential given the SPOF discussion) cost $0.06/GB/mo. A 16GB root disk snapshot costs ~$1/mo, but the document recommends snapshots as SPOF mitigation and never prices them.
- **Managed database storage overages:** The $50/mo PostgreSQL plan has a storage limit. Langfuse, Prefect, and application data will all write to the same managed cluster. Storage overages are not priced.
- **Claude API costs are volatile:** The "$20/mo Claude API (Haiku)" line is speculative and will vary significantly with usage. For a document targeting a solo developer deploying to production, this should have a worked example: e.g., "100 requests/day × 500 tokens average × $0.00025/1K tokens = ~$X/mo."
- **GitHub Actions minutes:** Building 11 container images in parallel will consume substantial Actions minutes. On a free-tier org, this will exhaust minutes quickly. Not mentioned at all.

**The Droplet SLA claim needs qualification.** "DO Droplet SLA is 99.99%" — DigitalOcean's SLA for Droplets is actually 99.99% *monthly uptime* which translates to ~4.4 minutes downtime per month. However, the SLA excludes scheduled maintenance, and more importantly, the SLA credit mechanism only applies after the fact. This should not be used to argue SPOF is low probability — the SLA is about hardware, not about software crashes, OOM kills, or bad deploys.

---

### Section 5: Docker Compose Configuration

**Strengths:**
- YAML anchors (`&common-env`, `&restart`, `&hc-defaults`) reduce repetition correctly. This is good practice for a long Compose file.
- Separating `docker-compose.override.yml` for local dev is correct.
- `depends_on` with `condition: service_healthy` is properly used, avoiding the race conditions that plague simpler Compose configurations.
- The `mcp-guardrails` service correctly sets a 4G memory limit, acknowledging it's the largest consumer.

**Issues:**

**Critical: `caddy_certs` volume is undeclared.** The Caddy service mounts `caddy_certs:/etc/caddy/certs` but the `volumes:` section at the top of the Compose file only declares `prometheus_data`, `grafana_data`, `langfuse_data`, and `prefect_data`. The `caddy_data` and `caddy_certs` volumes are missing from the top-level `volumes:` declaration. This will cause `docker compose up` to fail with a validation error.

**Critical: The Caddy healthcheck is wrong.** The healthcheck for Caddy is:
```yaml
test: ["CMD", "caddy", "version"]
```
This checks that the `caddy` binary exists and runs, not that Caddy is actually serving traffic or that TLS certificates have been provisioned. `caddy version` exits 0 immediately regardless of whether the server is running. The correct check is:
```yaml
test: ["CMD", "wget", "-q", "--spider", "http://localhost:2019/config/"]
```
Caddy exposes an admin API on port 2019 that correctly reflects server state.

**Missing volume: Caddy TLS persistence.** Caddy stores ACME certificates in `/data` by default, not `/etc/caddy/certs`. The `caddy_certs` volume mounted at `/etc/caddy/certs` will be unused unless the Caddyfile explicitly sets `storage` to that path. Without persistent TLS storage, Caddy will re-request certificates on every container restart, and will hit Let's Encrypt rate limits (5 certificates per registered domain per week) if the container restarts frequently during development.

**The `mcp-exec` security configuration references a non-existent seccomp profile.** Section 5 (Compose file) does NOT include the seccomp profile listed in §15:
```yaml
- seccomp:./seccomp/exec-profile.json
```
The Compose file in §5 only has `no-new-privileges:true` and `cap_drop: ALL`. The `read_only: true` and `tmpfs` configuration from §15 also do not appear in §5. These are presented as two different configurations of the same service. The actual deployed configuration (§5) is significantly weaker than the security section (§15) implies.

**No resource limits on most services.** Only `mcp-guardrails` has memory limits. On a 16GB host, a single runaway container (e.g., a memory leak in mcp-memory or a Langfuse query explosion) can OOM-kill the entire host, taking all services down simultaneously. Every service should have a `deploy.resources.limits.memory` value.

**`langfuse:latest` is a stability risk.** Using `:latest` tags for third-party images (langfuse, prometheus, grafana, prefect) in production is an antipattern. A Langfuse release on a Monday morning can break your observability stack. Pin to specific versions and update on your schedule.

**The Prometheus config only scrapes 4 of 11 services.** The `prometheus.yml` in §10 scrapes `openclaw-api`, `nanoclaw`, `mcp-guardrails`, and `caddy`. The other 7 MCP servers (`mcp-context`, `mcp-memory`, `mcp-skills`, `mcp-hitl`, `mcp-exec`, `mcp-db`, `mcp-github`) have no scrape configuration. If any of these services develop issues, Prometheus will not know. All services should expose `/metrics` and be scraped.

---

### Section 6: asyncio Background Task Pattern

**Strengths:**
- The pattern of accepting a `correlation_id` and `client_id` on every task is correct and essential for RLS and observability.
- Integration with Langfuse traces per task is well-designed — wrapping every attempt in a span gives excellent visibility.
- The `_running` set with `add_done_callback(self._running.discard)` correctly prevents asyncio task garbage collection (a common pitfall).
- The interface abstraction comment ("At Growth stage, swap this class for a CeleryTaskRunner with same interface") shows good forward-thinking.

**Issues:**

**Critical Bug: Coroutine is created and then re-created.** In `submit()`:
```python
coro = coro_fn(*args, **kwargs)   # coroutine created here — line 589
bg_task = BackgroundTask(
    task_id=task_id,
    coroutine=coro,               # stored but never awaited
    ...
)
```
Then in `_run_with_retry()`:
```python
result = await coro_fn(*args, **kwargs)  # coroutine created AGAIN — line 629
```
The coroutine stored in `bg_task.coroutine` is created but never awaited. It will generate a `RuntimeWarning: coroutine 'X' was never awaited` for every task. More importantly, if `coro_fn` has side effects on construction (unusual but possible), they execute twice. The `coroutine` field on `BackgroundTask` is dead code that should either be used (pass `coro` into `_run_with_retry` and await it) or removed.

**Critical Bug: `_tasks` dict is an unbounded memory leak.** The `_tasks` dict accumulates every submitted task and is never pruned. In production, each task dict entry holds a `BackgroundTask` dataclass. At MVP scale (say 1000 tasks/day), this grows indefinitely until the process crashes. There is no eviction, expiry, or cleanup. An LRU cache with a TTL, or periodic cleanup of completed/failed tasks older than N minutes, is required.

**The polling loop in `_post_result_when_ready` is an antipattern.** Polling every 1 second for up to 120 seconds from within an asyncio event loop creates 120 iterations × N concurrent users of wasted CPU cycles and event loop iterations. For a system with 20 concurrent tasks, this is 20 polling loops running simultaneously. The correct pattern is to use `asyncio.Event` or a Redis pub/sub channel to signal completion. The current polling approach will degrade event loop performance under load.

**No backpressure on submit.** If `max_concurrent=20` tasks are running and 100 more are submitted, `submit()` returns immediately with task IDs, but all 100 tasks will queue behind the semaphore. There is no way for callers to know whether their task is waiting or running, no queue depth metric, and no reject-at-threshold behavior. At 2× load, Discord users get a quick acknowledgment and then wait silently. The `pending_count` property exists but is never used to apply backpressure.

**Missing: Error notification path.** When `_run_with_retry` permanently fails and raises, the exception propagates into the asyncio task but is caught by the `_running.discard` callback. The Discord user is never notified of the failure. The `_post_result_when_ready` function in `nanoclaw/bot.py` only checks for `completed`/`failed` status and looks for a result in Redis — but `_run_with_retry` never writes to Redis. The failure notification path is broken.

**`asyncio.sleep(2 ** attempt)` without jitter.** At `max_retries=2`, the retries sleep for 1s, then 2s. With 20 concurrent tasks all failing simultaneously (e.g., a downstream MCP server goes down), all 20 tasks will retry at exactly the same time, creating a thundering herd against the recovering service. Add jitter: `asyncio.sleep(2 ** attempt + random.uniform(0, 1))`.

---

### Section 7: CI/CD Pipeline

**Strengths:**
- Building all 11 images in parallel via `strategy.matrix` is correct and efficient.
- Using `cache-from: type=registry` for layer caching is good practice.
- The health-check verification step after deploy (parsing `docker compose ps --format json`) is a good safety net.
- Discord deploy notifications are a nice operational touch.

**Issues:**

**The deploy script has a race condition.** The deploy script does:
```bash
docker compose pull
docker compose up -d --remove-orphans
sleep 15
# check health
```
The `sleep 15` is a fixed wait that does not guarantee services are healthy. `mcp-guardrails` with its 4GB model load time will likely not be healthy in 15 seconds. The health check after `up -d` should use a retry loop against the actual health endpoints, not a fixed sleep.

**The `IMAGE_TAG` variable is not passed into the SSH environment correctly.** The SSH script uses:
```bash
export IMAGE_TAG=${{ env.IMAGE_TAG }}
```
This is a shell injection vector. The `${{ env.IMAGE_TAG }}` is a GitHub SHA (40 hex characters, so low risk in practice), but the pattern of directly interpolating GitHub Actions expressions into shell commands via appleboy/ssh-action is documented as a security concern. The correct approach is to pass it as an environment variable through the action's `envs:` parameter, not inline.

**No rollback automation.** The document mentions "rollback via previous image tag" but the CI/CD pipeline has no automated rollback step. If the post-deploy health check fails (`sys.exit(1)`), the GitHub Actions job fails, but the Droplet is left in a partially-updated broken state. There is no `docker compose up -d` with the previous tag as a recovery step. For a solo developer, a broken 2am deploy with no automated rollback is a serious operational risk.

**All 11 images build sequentially fail the whole deploy.** With `strategy.matrix`, if building `mcp-exec` fails, all other builds succeed but the deploy job never runs. However, the issue is the inverse: if `mcp-context` builds successfully but `mcp-exec` fails, the matrix is marked failed and nothing deploys. That's correct behavior, but there's no partial-deploy protection in the script itself — if image pulls succeed for 10/11 services but fail for 1, `docker compose up -d` will start the 10 successfully-pulled services with old images for the remaining 1 without any warning.

**The provision script adds `deploy` user to the `docker` group.** This is equivalent to giving the deploy user root access. Anyone who compromises the GitHub Actions deploy key can run arbitrary containers on the host (e.g., `docker run --privileged -v /:/mnt alpine chroot /mnt`). The operational simplicity is real, but this risk should be explicitly called out. The correct mitigation is to use Docker's rootless mode or constrain the deploy user to only run `docker compose` commands via sudoers.

---

### Section 9: Secrets Management

**Strengths:**
- The `.env.example` with all required variables is correct practice.
- The explicit gitignore instruction for `.env` is necessary.
- Simple rotation procedure is appropriate for MVP.

**Issues:**

**The `.env` file on the Droplet is the blast radius for a full compromise.** The file contains every secret: database credentials, Anthropic API key (can spend unlimited money), Discord token (can impersonate the bot), GitHub App private key (can read/write any repo the app has access to), and Modal tokens (can run arbitrary serverless compute). There is no secret tiering, no envelope encryption, and no audit log of access. For an MVP serving paying clients, this is acceptable if the Droplet is well-secured, but the document should state this explicitly as a risk and provide a clear upgrade path (e.g., "At Growth, migrate to DigitalOcean Secrets Manager or HashiCorp Vault").

**The `GITHUB_APP_PRIVATE_KEY` in the `.env` file** will be a multi-line PEM key. Loading a multi-line value from a `.env` file reliably requires quoting (wrapping in single quotes with no newlines, or using base64 encoding). This is a common operational pitfall that will cause subtle authentication failures and is not addressed.

**The secret rotation procedure uses `nano` over SSH.** Using `nano` interactively over SSH to edit secrets is not auditable, not scriptable, and error-prone (fat-finger a character and the service breaks). A documented procedure using `sed -i` or a small rotation script that validates the new value before writing is significantly safer.

---

### Section 10: Observability

**Strengths:**
- The alert thresholds are specific and actionable: `langfuse_task_latency_p95 > 30s`, `container_memory_usage_bytes > 14G`.
- Co-locating observability with the application on the same Droplet means observability works even when external networking has issues.

**Issues:**

**Observability stack on the same Droplet is a reliability antipattern.** If the application causes an OOM kill, Prometheus and Grafana are killed at the same time. The alert that should fire at 2am fires after it's already too late, and the dashboards go dark when you most need them. At minimum, Prometheus's alertmanager should push to an external channel (PagerDuty, Opsgenie, or even a Discord webhook) before alerting relies on Grafana being up. This is partially addressed by the Discord notification in the CI/CD but is not in the operational runbook.

**`container_memory_usage_bytes > 14G` alert threshold is too late.** On a 16GB host, an alert at 14GB leaves 2GB before OOM. The Linux OOM killer is non-deterministic about which process it kills. The alert should fire at 12GB (75%), giving time to diagnose and act before the OOM killer activates.

**No alerting configuration is shown.** The document defines metrics to alert on but never shows the Alertmanager config, the alert routing, or how to receive pages. An alert rule that exists only in a metrics store but has no notification destination is useless at 2am.

**Langfuse on `:latest` with state in a Docker volume** means upgrading Langfuse (e.g., by pulling a new `:latest`) may require database migrations that are not automatically run. Langfuse schema migrations are not idempotent across all versions. This can silently corrupt observability data on restart.

---

### Section 12: Scaling Strategy

**Strengths:**
- The vertical scaling table with concrete Droplet sizes and costs is directly actionable.
- The 5-minute resize claim for DigitalOcean Droplets is accurate (it's a cold resize, but typically completes in 2–5 minutes).

**Issues:**

**The `--scale openclaw-api=2` horizontal scaling suggestion is incomplete and potentially broken.** Scaling to 2 instances of `openclaw-api` on the same Droplet with Docker Compose requires that both instances bind to different ports or use a shared load balancer. The current configuration doesn't expose host ports for `openclaw-api` (it's only reachable via Caddy), but Compose scaling adds containers without automatically configuring Caddy to upstream to both. This requires manual Caddyfile updates and is not documented.

**The `m-16vcpu-128gb` Droplet at $336/mo** is described as "memory-optimized" for the Late Growth (30–50 concurrent) tier. 128GB of RAM for 50 concurrent users of an AI agent platform with managed databases is likely over-provisioned by a factor of 4–8. The bottleneck at 50 concurrent users will be CPU (running Llama Guard 3 inference) and I/O (PostgreSQL queries), not RAM. A `c-32vcpu-64gb` CPU-optimized Droplet may be more appropriate and potentially cheaper.

**The 40-concurrent-user K8s migration trigger conflates two separate problems.** The asyncio semaphore saturating at 40 concurrent tasks is fixable without K8s (by adding Celery + Redis, which is explicitly mentioned as an option). The document conflates "add Celery" with "migrate to K8s" in a way that may push the team to over-engineer when simply adding Celery would suffice.

---

### Section 13: Migration Path

**Strengths:**
- The acknowledgment that "kompose output is a starting point, not production-ready" is honest and important.
- The list of required additions (resource limits, probes, PDBs, HPAs, NetworkPolicies) is correct.

**Issues:**

**`kompose` will not handle this Compose file well.** The Compose file uses several features that `kompose` either mishandles or ignores:
- YAML anchors (`<<: *restart`, `<<: *common-env`) are not standard YAML features natively supported by all YAML parsers; kompose may or may not expand them correctly.
- `deploy.resources.limits.memory` maps to K8s resource limits, but kompose's output requires manual validation.
- The `security_opt` fields (`no-new-privileges`, `seccomp`) translate to K8s `securityContext` with different syntax — kompose generates partial output that must be manually completed.
- The managed PostgreSQL and Redis connection strings use DigitalOcean-specific SSL parameters that will need re-validation in a DOKS context.

The "2–3 days of engineering time" for migration is almost certainly an underestimate if the team hasn't done a DOKS migration before. 1–2 weeks is more realistic for a first-time migration including networking, SOPS secret migration, Helm chart creation, and production cut-over.

**The migration plan has no data migration section.** The Langfuse, Prefect, and application data in the managed PostgreSQL instance is shared between the Docker Compose deployment and the future K8s deployment. This is actually a major advantage (the databases don't need migrating) but is not called out. The plan should explicitly state: "Database state is already in managed services — no data migration required."

---

### Section 14: Trade-Off Analysis

**Strengths:**
- The trade-off table with Severity and Mitigation columns is a clean, honest communication tool.
- The decision framework tree is easy to follow and gives clear escalation signals.

**Issues:**

**The risk matrix is inaccurate.** The matrix places SPOF at "High Impact, Low Probability." This is correct for hardware failure, but the more common failure modes that will affect this single-node architecture are:

- **Bad deploy:** Every push to main deploys to production. A bad deploy can take down all 11 services simultaneously. Probability: every few weeks in active development. Not hardware — software.
- **OOM kill:** With 16GB shared across 11+ containers, one runaway process (a large prompt, a vector similarity search explosion, Llama Guard loading a model twice) can OOM-kill the host. Probability: several times per month at MVP traffic levels. Not hardware — memory pressure.
- **Runaway asyncio task:** A task that loops infinitely or blocks the event loop (e.g., a synchronous database call accidentally awaited with `asyncio.run_in_executor` missing) will freeze the entire `openclaw-api` process. All 20 concurrent tasks hang simultaneously. Probability: high in early development.

These software-level SPOFs should appear separately in the risk matrix. The hardware SPOF (Droplet failure) is genuinely low probability. The software SPOFs are medium-to-high probability for an actively-developed MVP.

---

### Section 15: Security Considerations

**Strengths:**
- Listing `no-new-privileges`, `cap_drop: ALL` is correct minimum hardening.
- The Caddyfile correctly restricts observability endpoints to an IP allowlist.
- UFW rules blocking all non-HTTP/HTTPS/SSH ports are correct.

**Issues:**

**Critical Security Issue: `mcp-exec` runs arbitrary code with inadequate sandboxing.** The document mentions `gVisor` in a comment (`# Execution sandboxed via gVisor at container level`) but:
1. gVisor is not installed in the provision script.
2. The Docker daemon is not configured to use the `runsc` runtime.
3. The Compose file does not specify `runtime: runsc` for `mcp-exec`.
4. The seccomp profile (`./seccomp/exec-profile.json`) is referenced in §15 but not in §5, and the file itself does not exist in the described directory structure.

This means `mcp-exec` runs with the default Docker runtime (runc), `no-new-privileges:true` and `cap_drop: ALL` — which is better than nothing, but is far weaker than advertised. A container escape from `mcp-exec` gives access to the Docker socket (if mounted) or at minimum breaks the container boundary via kernel exploits. For a service that executes AI-generated code, this is a critical gap.

**The Docker socket is not explicitly excluded.** The provision script adds `deploy` to the `docker` group, and the containers run with Docker's default socket access. It's not clear whether any container has the Docker socket mounted (they shouldn't), but this is not explicitly prohibited in the Compose file.

**The Caddy IP restriction using `remote_ip {$ALLOWED_CIDR}` has a pitfall.** If the solo developer's home IP changes (dynamic IP, traveling, working from a client site), they will lock themselves out of Langfuse, Grafana, and Prefect dashboards. There is no secondary access method documented (VPN, IP override procedure, fallback). The runbook should include "how to change ALLOWED_CIDR in an emergency."

**No container image signing or verification.** The CI/CD pipeline pushes to GHCR and pulls from GHCR. There is no image signing (Cosign/Sigstore) or verification step. A compromised GHCR token or a supply chain attack on a base image (particularly `langfuse/langfuse:latest`) is not mitigated.

**The `SANDBOX_ENABLED: "true"` environment variable** implies there is application-level sandboxing logic in `mcp-exec`. This logic is not shown anywhere in the document. If this environment variable is not actually implemented in the mcp-exec service code, it is a false safety signal.

---

### Section 16–17: Summary and Quick-Start

**Strengths:**
- The summary comparison table is well-structured and honest about the trade-offs.
- The Quick-Start Checklist is a practical addition that most architecture documents omit.

**Issues:**
- Step 3 ("Add GitHub Actions deploy key") confuses two different things: a GitHub repository deploy key (read-only by default, used for `git clone`) and the SSH key used by `appleboy/ssh-action` to connect to the Droplet. These are different keys with different scopes. The checklist should clearly separate these.
- The checklist has no smoke test: after step 7, there is no verification that all services are actually healthy, only that they started. A service can be in `Up` state but not healthy.
- Step 4 says "(see doc 09 §5 for exact doctl commands)" but doc 09 doesn't exist in this filesystem. This cross-reference is broken and leaves a gap in the setup procedure.

---

## Top 5 Critical Issues (Must-Fix Before Going Live)

### 1. mcp-exec Security is Aspirational, Not Implemented

The document claims gVisor sandboxing ("Execution sandboxed via gVisor at container level") but provides no evidence of gVisor being installed, configured, or enabled. The Compose service definition in §5 has weaker security than the configuration shown in §15. The seccomp profile referenced does not exist in the directory structure. For a service whose entire purpose is executing AI-generated code, this gap is production-blocking. Before going live: install gVisor on the Droplet, add `runtime: runsc` to the `mcp-exec` service, create and validate the seccomp profile, reconcile §5 and §15 into a single authoritative configuration, and add an integration test that confirms `mcp-exec` cannot write to `/proc`, call `ptrace`, or fork beyond a limit.

### 2. Coroutine Double-Invocation Bug in AsyncTaskRunner

The `submit()` method in `task_runner.py` creates a coroutine via `coro_fn(*args, **kwargs)` and stores it in `BackgroundTask.coroutine`, then `_run_with_retry` calls `coro_fn(*args, **kwargs)` again. The stored coroutine is never awaited, generating `RuntimeWarning` on every task. If `coro_fn` has any side effect on construction (e.g., increments a counter, allocates a connection, logs a trace start), those effects fire twice. Fix: either pass the pre-created coroutine into `_run_with_retry` and await it for the first attempt (with `coro_fn` called only for retry attempts), or remove the `coroutine` field from `BackgroundTask` entirely and always call `coro_fn` fresh.

### 3. Missing Volume Declarations Break `docker compose up`

The Compose file mounts `caddy_data` and `caddy_certs` as named volumes but neither appears in the top-level `volumes:` declaration. `docker compose up` will fail at parse time with: `service "caddy" refers to undefined volume caddy_data`. This is a day-zero blocker. Additionally, Caddy stores ACME certificates under `/data` by default, not `/etc/caddy/certs`, so the mounted volume provides no persistence benefit until the Caddyfile explicitly overrides the storage path. Fix: add both volumes to the declaration and set `storage file /data/caddy` in the Caddyfile.

### 4. No Automated Rollback on Failed Deploy

The CI/CD pipeline health check (`sys.exit(1)` on unhealthy services) correctly detects a bad deploy but leaves the Droplet in a broken state. There is no automated recovery. A solo developer being paged at 2am needs to know: what was the last good image tag, and what command reverts to it. The deploy job must capture `PREVIOUS_IMAGE_TAG` before pulling new images and include a rollback step triggered on health-check failure. Without this, every bad deploy requires manual intervention under pressure.

### 5. `_tasks` Dict Memory Leak Will Crash the Process

The `AsyncTaskRunner._tasks` dict accumulates every submitted task with no eviction. At 1,000 tasks/day (a modest MVP load), this grows without bound. At 10,000 tasks it will contain megabytes of task metadata. More critically, the `BackgroundTask.coroutine` field holds a reference to a coroutine object that may in turn hold references to closures capturing large data structures (message content, context, client state). This will cause progressive memory growth until the process is OOM-killed, taking all 20 concurrent users' tasks with it. Fix: add a cleanup coroutine that runs every 5 minutes to delete tasks from `_tasks` where `status in (COMPLETED, FAILED)` and `created_at < now - 10 minutes`.

---

## Top 5 Recommendations (Significant Improvements)

### 1. Add a Minimal Operational Runbook for 2am Failures

The document has excellent design content but no operational runbook. A solo developer needs documented answers to: "Everything is down. What do I do?" The runbook should cover at minimum:
- How to check which container is unhealthy: `docker compose ps`
- How to view logs for a failed service: `docker compose logs --tail=100 <service>`
- How to restart a single service without touching others: `docker compose restart <service>`
- How to identify an OOM kill: `dmesg | grep -i oom`
- How to do an emergency rollback: explicit commands with the image tag syntax
- How to restore from a Droplet snapshot: DigitalOcean console steps
- How to unlock observability tools if ALLOWED_CIDR is stale: procedure to update and restart Caddy
This runbook can be a separate document (`docs/11-runbook.md`) but must be linked from this document before production.

### 2. Pin All Third-Party Image Tags and Add Dependabot

Replace all `:latest` tags with pinned versions:
- `langfuse/langfuse:2.x.y`
- `prom/prometheus:v2.x.y`
- `grafana/grafana:10.x.y`
- `prefecthq/prefect:3.x.y`

Add a `.github/dependabot.yml` that opens PRs when new patch versions are available. Using `:latest` in production means an upstream release on any given morning can break the observability stack or change behavior of the workflow engine silently. The cost of pinning is running a quarterly update; the cost of not pinning is unpredictable prod breakage.

### 3. Replace the Polling Loop with asyncio.Event Signaling

The `_post_result_when_ready` polling loop (120 iterations × 1s = 2 minutes max) is inefficient and will degrade under load. Replace with an `asyncio.Event` per task stored alongside the task metadata:

```python
# In BackgroundTask, add:
completion_event: asyncio.Event = field(default_factory=asyncio.Event)

# In _run_with_retry, after setting status:
bg_task.completion_event.set()

# In _post_result_when_ready:
try:
    await asyncio.wait_for(task.completion_event.wait(), timeout=120)
except asyncio.TimeoutError:
    await discord_message.reply("Task timed out.")
    return
```

This eliminates 120× the event loop iterations per task, eliminates the Redis result storage dependency in the bot, and provides immediate notification rather than up-to-1-second lag.

### 4. Add Resource Limits to Every Service and Set OOM Alert at 75%

Apply explicit memory limits to every service in the Compose file based on realistic profiling. Suggested starting limits:
- `nanoclaw`: 512M
- `openclaw-api`: 1G
- `mcp-context`, `mcp-skills`, `mcp-hitl`, `mcp-db`, `mcp-github`: 256M each
- `mcp-memory`: 512M (vector operations)
- `mcp-guardrails`: 4G (Llama Guard 3 model)
- `mcp-exec`: 512M
- `langfuse`, `grafana`, `prometheus`: 512M each
- `prefect-server`, `prefect-agent`: 256M each

Total: ~10G, within the 16GB host with ~4GB OS overhead. Move the OOM alert threshold from 14GB to 12GB (75%). Add a per-container memory alert at 80% of limit to catch runaway services before they hit the OOM killer.

### 5. Implement Structured Secret Management with a Migration Path

The current `.env` plaintext approach is acceptable at MVP but should have a documented upgrade path rather than just "Future: SOPS + age." A practical intermediate step before full K8s migration: use DigitalOcean's Secrets Manager (or a simple `pass`-based encrypted store) for the most sensitive secrets (Anthropic API key, GitHub App private key, Discord token), with the `.env` file only containing non-sensitive configuration (domain names, ports, feature flags). This reduces the blast radius of a compromised Droplet from "attacker has everything" to "attacker has some things." Add explicit instructions for base64-encoding the `GITHUB_APP_PRIVATE_KEY` to avoid the multiline `.env` parsing bug.

---

## Additional Minor Issues (Not Blocking, But Should Be Fixed)

- The Caddyfile has no `log` directive. Caddy access logs are essential for security auditing and debugging. Add `log { output file /var/log/caddy/access.log }`.
- The `prefect-agent` command uses `--pool default-process-pool` but the Prefect 3 work pool type for local execution is `process`. Verify this matches the Prefect 3 API syntax.
- The CI/CD deploy script uses `git checkout origin/main -- docker-compose.yml ...` to pull config files. If the deploy key is read-only (as configured), this `git fetch origin main` will succeed, but the checkout requires the HEAD to be updated, which may fail if `/opt/openclaw` is not properly initialized. A clean `git pull origin main` or `git reset --hard origin/main` for config-only files would be more robust.
- There is no `docker system df` or disk space monitoring. A 16GB root disk filling with old container images and logs is a common operational failure mode. Add a Prometheus node_exporter scrape target and an alert for `disk_free < 20%`.
- The `VECTOR_DIMENSIONS: 1536` on `mcp-memory` assumes OpenAI `text-embedding-3-small` dimensions. If the embedding model changes, this mismatch will cause silent incorrect similarity results rather than a hard failure. This configuration should be documented as tightly coupled to the embedding model selection.

---

## Overall Grade: C+

**Justification:**

The strategic design is sound (B+) — defer K8s, use Docker Compose, preserve the right abstractions, plan for migration. The trade-off analysis is honest and the scaling decision framework is well-reasoned. These decisions would survive an RFC review at a top-tier company.

However, the implementation has multiple correctness issues that prevent this document from being used directly for production deployment. The coroutine double-invocation bug and memory leak in the AsyncTaskRunner are real bugs in production code samples, not just design issues. The undeclared volumes break `docker compose up` at day zero. The mcp-exec security section misrepresents the actual implemented security posture. The CI/CD pipeline lacks automated rollback. The operational runbook is essentially absent.

For a document that claims to be "MVP Design" for a paying-client service, the gap between "design decisions are good" and "this is ready to ship" is substantial. The document earns its grade by being honest about trade-offs and structurally sound, but loses significant ground on implementation correctness, security accuracy, and operational completeness.

**To reach a B:** Fix the five critical issues, add resource limits to all services, pin all image tags, and add a basic operational runbook.
**To reach an A:** All of the above plus implement the asyncio.Event pattern, add proper secret management, complete the Prometheus scrape configuration, and validate the Caddyfile/volume configuration with `docker compose config`.
