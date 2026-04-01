# Doc 10 — Simplified MVP Architecture: No-Kubernetes Approach

> **Status:** MVP Design
> **Replaces:** K8s-based MVP from [doc 09: Implementation Plan](09-implementation-plan.md) (for initial deployment)
> **Prereqs:** [doc 06: System Design](06-system-design.md), [doc 07: Context Architecture](07-hierarchical-context-architecture.md), [doc 08: Unified Platform](08-unified-platform-architecture.md)

---

## 1. Motivation

The full architecture defined in docs 06–09 targets long-term scale and uses DigitalOcean Kubernetes Service (DOKS) as the compute backbone. A 3-node DOKS cluster costs ~$144/mo before databases, and Kubernetes adds significant operational overhead: Helm chart management, kubectl knowledge, rolling update complexity, and debugging across pod boundaries.

For an MVP serving early clients with predictable, low-to-moderate traffic, this overhead is unnecessary. The goal of this document is to:

1. Preserve **all core architectural principles** (MCP-first, 4-tier context, RLS isolation, guardrails pipeline, autonomous improvement loop, Claude Code SDK)
2. **Eliminate Kubernetes** entirely for the MVP deployment phase
3. **Reduce baseline infrastructure cost** from ~$216/mo to ~$161/mo
4. **Reduce operational complexity** so a single developer can run, debug, and iterate rapidly
5. Define a **clean migration path** back to Kubernetes when the MVP outgrows this approach

This is not a permanent simplification — it is a deliberate starting point optimized for speed of iteration and cost control.

---

## 2. Core Principles Preserved

Every principle from the full architecture carries forward unchanged:

| Principle | Full K8s | Simplified MVP | Notes |
|-----------|----------|----------------|-------|
| MCP-First capabilities | ✅ K8s pods | ✅ Docker Compose services | Same MCP servers, same protocol |
| 4-Tier context model | ✅ | ✅ | PostgreSQL + pgvector unchanged |
| RLS client isolation | ✅ | ✅ | Enforced at DB layer, not infra layer |
| Layered guardrails | ✅ | ✅ | Presidio → LLM Guard → Llama Guard 3 |
| Claude Code SDK | ✅ | ✅ | Flat-rate, no change |
| asyncio concurrency | ✅ | ✅ | Same WorkerPool implementation |
| Langfuse observability | ✅ | ✅ | Self-hosted, same container |
| Autonomous improvement loop | ✅ | ✅ | Prefect + DSPy on Modal unchanged |
| Temporal knowledge graph | ✅ | ✅ | Graphiti on PostgreSQL unchanged |
| Skills registry | ✅ | ✅ | PostgreSQL table unchanged |

---

## 3. What Changes

### 3.1 Compute Layer

| Component | Full K8s MVP | Simplified MVP |
|-----------|-------------|----------------|
| Cluster | 3× s-2vcpu-4gb DOKS nodes ($144/mo) | 1× s-8vcpu-16gb Droplet ($96/mo) |
| Orchestration | Kubernetes + Helm | Docker Compose |
| Ingress | nginx-ingress + cert-manager | Caddy (auto-TLS built-in) |
| Service discovery | kube-dns | Docker Compose internal DNS |
| Scaling | HPA (horizontal pod autoscaler) | Manual: `docker compose up --scale` |
| Rolling deploys | Built-in | Blue-green via Compose profiles |
| Health checks | Liveness/readiness probes | Docker healthcheck + Caddy upstreams |

### 3.2 Task Queue

| Component | Full K8s | Simplified MVP |
|-----------|---------|----------------|
| Worker system | Celery + Redis task queue | asyncio background tasks (in-process) |
| Task durability | Redis-backed, survives restarts | In-memory queue; tasks lost on crash |
| Scaling workers | `kubectl scale deployment celery-worker` | Not supported (single process) |
| Monitoring | Flower UI | Langfuse traces cover LLM tasks |
| Retry semantics | Celery `autoretry_for`, exponential backoff | `asyncio.shield` + manual retry loop |

**Implication:** For MVP, task durability is traded for simplicity. If the process crashes mid-task, the Discord user receives an error and can re-trigger. This is acceptable for MVP traffic. Celery is reintroduced at the Growth stage.

### 3.3 CI/CD

| Component | Full K8s | Simplified MVP |
|-----------|---------|----------------|
| Deploy mechanism | GitHub Actions → Helm upgrade | GitHub Actions → SSH + `docker compose up` |
| Secrets management | SOPS + age + Helm secrets | GitHub Actions secrets → `.env` via SSH |
| Rollback | `helm rollback` | `docker compose up` previous image tag |
| Zero-downtime deploy | Kubernetes rolling update | Brief downtime (~5s container restart) |

---

## 4. Infrastructure Topology

```
┌─────────────────────────────────────────────────────────────┐
│                    DigitalOcean Region                       │
│                                                             │
│  ┌──────────────────────────────────────────────────────┐   │
│  │          s-8vcpu-16gb Droplet  ($96/mo)              │   │
│  │                                                      │   │
│  │  ┌──────────┐  ┌──────────────────────────────────┐ │   │
│  │  │  Caddy   │  │     Docker Compose Network        │ │   │
│  │  │ :80/:443 │  │                                   │ │   │
│  │  │ auto-TLS │  │  nanoclaw      openclaw-api       │ │   │
│  │  └────┬─────┘  │  mcp-context   mcp-memory         │ │   │
│  │       │        │  mcp-skills    mcp-guardrails      │ │   │
│  │       │        │  mcp-hitl      mcp-exec            │ │   │
│  │       │        │  mcp-db        mcp-github          │ │   │
│  │       │        │  langfuse      prometheus          │ │   │
│  │       │        │  grafana       prefect-server      │ │   │
│  │       └───────►│  prefect-agent                    │ │   │
│  │                └──────────────────────────────────┘ │   │
│  └──────────────────────────────────────────────────────┘   │
│                                                             │
│  ┌─────────────────────┐  ┌─────────────────────────────┐  │
│  │  Managed PostgreSQL  │  │     Managed Redis           │  │
│  │  + pgvector          │  │     (sessions, cache)       │  │
│  │  ($50/mo)            │  │     ($15/mo)                │  │
│  └─────────────────────┘  └─────────────────────────────┘  │
│                                                             │
│  ┌──────────────────────────────────────────────────────┐   │
│  │  DigitalOcean Spaces (S3-compatible)  ($5/mo)        │   │
│  │  skill artifacts, session exports, pg backups         │   │
│  └──────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────┘

External Services (unchanged from full architecture):
  Modal       — DSPy nightly optimization (serverless, ~$5–10/mo)
  Anthropic   — Claude API (Haiku) + Claude Code SDK ($200/seat/mo)
  Discord     — NanoClaw bot interface
  GitHub      — Source, Actions CI/CD
```

### Monthly Cost Breakdown

| Resource | Simplified MVP | Full K8s MVP | Δ |
|----------|---------------|-------------|---|
| Compute | $96 (1 Droplet) | $144 (3-node DOKS) | **−$48** |
| PostgreSQL | $50 (managed) | $50 (managed) | $0 |
| Redis | $15 (managed) | $15 (managed) | $0 |
| Spaces (S3) | $5 | $5 | $0 |
| Modal (DSPy) | ~$5–10 | ~$5–10 | $0 |
| **Total infra** | **~$171–176/mo** | **~$219–224/mo** | **−$48/mo** |
| Claude Code SDK | $200/seat | $200/seat | $0 |
| Claude API (Haiku) | ~$20 | ~$20 | $0 |
| **Total platform** | **~$391–396/mo** | **~$439–444/mo** | **−$48/mo** |

> The simplified approach saves ~$576/year in infrastructure cost at MVP scale.

---

## 5. Docker Compose Configuration

### 5.1 Directory Structure

```
openclaw/
├── docker-compose.yml          # All services
├── docker-compose.override.yml # Local dev overrides
├── .env.example                # Template (committed)
├── .env                        # Secrets (gitignored)
├── Caddyfile                   # Reverse proxy config
├── prometheus/
│   └── prometheus.yml
├── grafana/
│   └── provisioning/
├── prefect/
│   └── flows/
└── services/
    ├── nanoclaw/
    ├── openclaw-api/
    └── mcp-*/
```

### 5.2 Full docker-compose.yml

```yaml
# docker-compose.yml
version: "3.9"

x-common-env: &common-env
  DATABASE_URL: ${DATABASE_URL}
  REDIS_URL: ${REDIS_URL}
  ANTHROPIC_API_KEY: ${ANTHROPIC_API_KEY}
  LANGFUSE_PUBLIC_KEY: ${LANGFUSE_PUBLIC_KEY}
  LANGFUSE_SECRET_KEY: ${LANGFUSE_SECRET_KEY}
  LANGFUSE_HOST: http://langfuse:3000
  LOG_LEVEL: ${LOG_LEVEL:-info}

x-restart: &restart
  restart: unless-stopped

x-healthcheck-defaults: &hc-defaults
  interval: 30s
  timeout: 10s
  retries: 3
  start_period: 15s

networks:
  openclaw:
    driver: bridge

volumes:
  prometheus_data:
  grafana_data:
  langfuse_data:
  prefect_data:

services:
  # ──────────────────────────────────────────
  # Reverse Proxy (replaces nginx-ingress)
  # ──────────────────────────────────────────
  caddy:
    image: caddy:2-alpine
    <<: *restart
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - ./Caddyfile:/etc/caddy/Caddyfile:ro
      - caddy_data:/data
      - caddy_certs:/etc/caddy/certs
    networks:
      - openclaw
    healthcheck:
      <<: *hc-defaults
      test: ["CMD", "caddy", "version"]

  # ──────────────────────────────────────────
  # Discord Bot (NanoClaw)
  # ──────────────────────────────────────────
  nanoclaw:
    image: ghcr.io/your-org/nanoclaw:${IMAGE_TAG:-latest}
    <<: *restart
    environment:
      <<: *common-env
      DISCORD_TOKEN: ${DISCORD_TOKEN}
      DISCORD_APPLICATION_ID: ${DISCORD_APPLICATION_ID}
      OPENCLAW_API_URL: http://openclaw-api:8000
      MCP_CONTEXT_URL: http://mcp-context:8001
      MCP_MEMORY_URL: http://mcp-memory:8002
      MCP_SKILLS_URL: http://mcp-skills:8003
      MCP_GUARDRAILS_URL: http://mcp-guardrails:8004
      MCP_HITL_URL: http://mcp-hitl:8005
      MCP_EXEC_URL: http://mcp-exec:8006
      MCP_DB_URL: http://mcp-db:8007
      MCP_GITHUB_URL: http://mcp-github:8008
    depends_on:
      openclaw-api:
        condition: service_healthy
      mcp-guardrails:
        condition: service_healthy
    networks:
      - openclaw
    healthcheck:
      <<: *hc-defaults
      test: ["CMD", "python", "-c", "import urllib.request; urllib.request.urlopen('http://localhost:9090/health')"]

  # ──────────────────────────────────────────
  # OpenClaw REST API
  # ──────────────────────────────────────────
  openclaw-api:
    image: ghcr.io/your-org/openclaw-api:${IMAGE_TAG:-latest}
    <<: *restart
    environment:
      <<: *common-env
      API_SECRET_KEY: ${API_SECRET_KEY}
      CORS_ORIGINS: ${CORS_ORIGINS:-https://app.openclaw.ai}
    depends_on:
      mcp-context:
        condition: service_healthy
      mcp-memory:
        condition: service_healthy
    networks:
      - openclaw
    healthcheck:
      <<: *hc-defaults
      test: ["CMD", "curl", "-f", "http://localhost:8000/health"]

  # ──────────────────────────────────────────
  # MCP Servers
  # ──────────────────────────────────────────
  mcp-context:
    image: ghcr.io/your-org/mcp-context:${IMAGE_TAG:-latest}
    <<: *restart
    environment:
      <<: *common-env
      MCP_PORT: 8001
    networks:
      - openclaw
    healthcheck:
      <<: *hc-defaults
      test: ["CMD", "curl", "-f", "http://localhost:8001/health"]

  mcp-memory:
    image: ghcr.io/your-org/mcp-memory:${IMAGE_TAG:-latest}
    <<: *restart
    environment:
      <<: *common-env
      MCP_PORT: 8002
      VECTOR_DIMENSIONS: 1536
    networks:
      - openclaw
    healthcheck:
      <<: *hc-defaults
      test: ["CMD", "curl", "-f", "http://localhost:8002/health"]

  mcp-skills:
    image: ghcr.io/your-org/mcp-skills:${IMAGE_TAG:-latest}
    <<: *restart
    environment:
      <<: *common-env
      MCP_PORT: 8003
    networks:
      - openclaw
    healthcheck:
      <<: *hc-defaults
      test: ["CMD", "curl", "-f", "http://localhost:8003/health"]

  mcp-guardrails:
    image: ghcr.io/your-org/mcp-guardrails:${IMAGE_TAG:-latest}
    <<: *restart
    environment:
      <<: *common-env
      MCP_PORT: 8004
      PRESIDIO_ENABLED: "true"
      LLM_GUARD_ENABLED: "true"
      LLAMA_GUARD_MODEL: ${LLAMA_GUARD_MODEL:-meta-llama/Llama-Guard-3-8B}
    # Higher memory for guardrail models
    deploy:
      resources:
        limits:
          memory: 4G
    networks:
      - openclaw
    healthcheck:
      <<: *hc-defaults
      test: ["CMD", "curl", "-f", "http://localhost:8004/health"]

  mcp-hitl:
    image: ghcr.io/your-org/mcp-hitl:${IMAGE_TAG:-latest}
    <<: *restart
    environment:
      <<: *common-env
      MCP_PORT: 8005
      DISCORD_WEBHOOK_URL: ${HITL_DISCORD_WEBHOOK_URL}
      APPROVAL_TIMEOUT_SECONDS: 300
    networks:
      - openclaw
    healthcheck:
      <<: *hc-defaults
      test: ["CMD", "curl", "-f", "http://localhost:8005/health"]

  mcp-exec:
    image: ghcr.io/your-org/mcp-exec:${IMAGE_TAG:-latest}
    <<: *restart
    environment:
      <<: *common-env
      MCP_PORT: 8006
      SANDBOX_ENABLED: "true"
      # Execution sandboxed via gVisor at container level
    security_opt:
      - no-new-privileges:true
    cap_drop:
      - ALL
    networks:
      - openclaw
    healthcheck:
      <<: *hc-defaults
      test: ["CMD", "curl", "-f", "http://localhost:8006/health"]

  mcp-db:
    image: ghcr.io/your-org/mcp-db:${IMAGE_TAG:-latest}
    <<: *restart
    environment:
      <<: *common-env
      MCP_PORT: 8007
    networks:
      - openclaw
    healthcheck:
      <<: *hc-defaults
      test: ["CMD", "curl", "-f", "http://localhost:8007/health"]

  mcp-github:
    image: ghcr.io/your-org/mcp-github:${IMAGE_TAG:-latest}
    <<: *restart
    environment:
      <<: *common-env
      MCP_PORT: 8008
      GITHUB_APP_ID: ${GITHUB_APP_ID}
      GITHUB_APP_PRIVATE_KEY: ${GITHUB_APP_PRIVATE_KEY}
    networks:
      - openclaw
    healthcheck:
      <<: *hc-defaults
      test: ["CMD", "curl", "-f", "http://localhost:8008/health"]

  # ──────────────────────────────────────────
  # Observability
  # ──────────────────────────────────────────
  langfuse:
    image: langfuse/langfuse:latest
    <<: *restart
    environment:
      DATABASE_URL: ${LANGFUSE_DATABASE_URL}
      NEXTAUTH_URL: https://langfuse.${BASE_DOMAIN}
      NEXTAUTH_SECRET: ${LANGFUSE_NEXTAUTH_SECRET}
      SALT: ${LANGFUSE_SALT}
      LANGFUSE_ENABLE_EXPERIMENTAL_FEATURES: "true"
    volumes:
      - langfuse_data:/app/.next/cache
    networks:
      - openclaw
    healthcheck:
      <<: *hc-defaults
      test: ["CMD", "curl", "-f", "http://localhost:3000/api/public/health"]

  prometheus:
    image: prom/prometheus:latest
    <<: *restart
    volumes:
      - ./prometheus/prometheus.yml:/etc/prometheus/prometheus.yml:ro
      - prometheus_data:/prometheus
    command:
      - '--config.file=/etc/prometheus/prometheus.yml'
      - '--storage.tsdb.path=/prometheus'
      - '--storage.tsdb.retention.time=15d'
      - '--web.enable-lifecycle'
    networks:
      - openclaw

  grafana:
    image: grafana/grafana:latest
    <<: *restart
    environment:
      GF_SECURITY_ADMIN_PASSWORD: ${GRAFANA_ADMIN_PASSWORD}
      GF_SERVER_ROOT_URL: https://grafana.${BASE_DOMAIN}
      GF_AUTH_ANONYMOUS_ENABLED: "false"
    volumes:
      - grafana_data:/var/lib/grafana
      - ./grafana/provisioning:/etc/grafana/provisioning:ro
    depends_on:
      - prometheus
    networks:
      - openclaw

  # ──────────────────────────────────────────
  # Workflow Orchestration (Prefect)
  # ──────────────────────────────────────────
  prefect-server:
    image: prefecthq/prefect:3-latest
    <<: *restart
    command: prefect server start --host 0.0.0.0
    environment:
      PREFECT_API_DATABASE_CONNECTION_URL: ${PREFECT_DATABASE_URL}
      PREFECT_SERVER_API_HOST: 0.0.0.0
    volumes:
      - prefect_data:/root/.prefect
    networks:
      - openclaw
    healthcheck:
      <<: *hc-defaults
      test: ["CMD", "curl", "-f", "http://localhost:4200/api/health"]

  prefect-agent:
    image: ghcr.io/your-org/prefect-agent:${IMAGE_TAG:-latest}
    <<: *restart
    command: prefect worker start --pool default-process-pool
    environment:
      <<: *common-env
      PREFECT_API_URL: http://prefect-server:4200/api
      MODAL_TOKEN_ID: ${MODAL_TOKEN_ID}
      MODAL_TOKEN_SECRET: ${MODAL_TOKEN_SECRET}
    depends_on:
      prefect-server:
        condition: service_healthy
    networks:
      - openclaw
```

### 5.3 Caddyfile (TLS Reverse Proxy)

```caddyfile
# Caddyfile
{
  email {$ACME_EMAIL}
  # Uncomment for staging TLS during setup:
  # acme_ca https://acme-staging-v02.api.letsencrypt.org/directory
}

# Discord bot webhook / API (public)
api.{$BASE_DOMAIN} {
  reverse_proxy openclaw-api:8000
  header {
    Strict-Transport-Security "max-age=31536000; includeSubDomains"
    X-Content-Type-Options nosniff
    X-Frame-Options DENY
  }
}

# Observability (private — restrict to your IP or VPN)
langfuse.{$BASE_DOMAIN} {
  @allowed remote_ip {$ALLOWED_CIDR}
  handle @allowed {
    reverse_proxy langfuse:3000
  }
  handle {
    respond 403
  }
}

grafana.{$BASE_DOMAIN} {
  @allowed remote_ip {$ALLOWED_CIDR}
  handle @allowed {
    reverse_proxy grafana:3000
  }
  handle {
    respond 403
  }
}

prefect.{$BASE_DOMAIN} {
  @allowed remote_ip {$ALLOWED_CIDR}
  handle @allowed {
    reverse_proxy prefect-server:4200
  }
  handle {
    respond 403
  }
}
```

---

## 6. asyncio Background Task Pattern (Replacing Celery)

For MVP, long-running AI tasks are handled via asyncio without a separate worker process. This trades durability for simplicity.

```python
# openclaw/core/task_runner.py
import asyncio
import logging
from dataclasses import dataclass, field
from datetime import datetime
from enum import Enum
from typing import Any, Callable, Coroutine
from langfuse import Langfuse
import uuid

logger = logging.getLogger(__name__)
langfuse = Langfuse()


class TaskStatus(Enum):
    PENDING = "pending"
    RUNNING = "running"
    COMPLETED = "completed"
    FAILED = "failed"


@dataclass
class BackgroundTask:
    task_id: str
    coroutine: Coroutine
    correlation_id: str          # Discord message ID, API request ID, etc.
    client_id: str
    interface: str               # "nanoclaw" | "api" | "claude-code"
    max_retries: int = 2
    attempt: int = 0
    status: TaskStatus = TaskStatus.PENDING
    created_at: datetime = field(default_factory=datetime.utcnow)


class AsyncTaskRunner:
    """
    MVP replacement for Celery workers.
    Runs tasks in asyncio background without cross-process durability.
    At Growth stage, swap this class for a CeleryTaskRunner with same interface.
    """

    def __init__(self, max_concurrent: int = 20):
        self._semaphore = asyncio.Semaphore(max_concurrent)
        self._tasks: dict[str, BackgroundTask] = {}
        self._running: set[asyncio.Task] = set()

    async def submit(
        self,
        coro_fn: Callable[..., Coroutine],
        *args,
        correlation_id: str,
        client_id: str,
        interface: str,
        max_retries: int = 2,
        **kwargs,
    ) -> str:
        """Submit a task and return its task_id immediately."""
        task_id = str(uuid.uuid4())

        # Wrap coroutine with retry + observability
        coro = coro_fn(*args, **kwargs)
        bg_task = BackgroundTask(
            task_id=task_id,
            coroutine=coro,
            correlation_id=correlation_id,
            client_id=client_id,
            interface=interface,
            max_retries=max_retries,
        )
        self._tasks[task_id] = bg_task

        asyncio_task = asyncio.create_task(
            self._run_with_retry(bg_task, coro_fn, args, kwargs)
        )
        self._running.add(asyncio_task)
        asyncio_task.add_done_callback(self._running.discard)

        return task_id

    async def _run_with_retry(
        self, bg_task: BackgroundTask, coro_fn, args, kwargs
    ) -> Any:
        trace = langfuse.trace(
            id=bg_task.task_id,
            name=coro_fn.__name__,
            metadata={
                "client_id": bg_task.client_id,
                "interface": bg_task.interface,
                "correlation_id": bg_task.correlation_id,
            },
        )

        async with self._semaphore:
            bg_task.status = TaskStatus.RUNNING
            last_error = None

            for attempt in range(bg_task.max_retries + 1):
                bg_task.attempt = attempt
                span = trace.span(name=f"attempt_{attempt}")
                try:
                    result = await coro_fn(*args, **kwargs)
                    bg_task.status = TaskStatus.COMPLETED
                    span.end(output={"status": "success"})
                    trace.update(output={"status": "completed"})
                    return result
                except Exception as e:
                    last_error = e
                    span.end(
                        output={"error": str(e)},
                        level="ERROR",
                    )
                    logger.warning(
                        "Task %s attempt %d failed: %s",
                        bg_task.task_id, attempt, e,
                    )
                    if attempt < bg_task.max_retries:
                        await asyncio.sleep(2 ** attempt)  # exponential backoff

            bg_task.status = TaskStatus.FAILED
            trace.update(output={"status": "failed", "error": str(last_error)})
            logger.error("Task %s permanently failed: %s", bg_task.task_id, last_error)
            raise last_error

    def get_status(self, task_id: str) -> TaskStatus | None:
        task = self._tasks.get(task_id)
        return task.status if task else None

    @property
    def pending_count(self) -> int:
        return sum(
            1 for t in self._tasks.values()
            if t.status in (TaskStatus.PENDING, TaskStatus.RUNNING)
        )


# Global singleton — injected into NanoClaw and OpenClaw API
task_runner = AsyncTaskRunner(max_concurrent=20)
```

**NanoClaw integration:**

```python
# nanoclaw/bot.py (simplified)
from openclaw.core.task_runner import task_runner
from openclaw.core.agents import run_openclaw_agent

async def handle_complex_task(discord_message, client_id: str):
    """Submit long-running agent task without blocking the Discord event loop."""
    task_id = await task_runner.submit(
        run_openclaw_agent,
        message=discord_message.content,
        context=await build_context(discord_message),
        correlation_id=str(discord_message.id),
        client_id=client_id,
        interface="nanoclaw",
    )

    # Acknowledge immediately
    await discord_message.reply(
        f"⏳ Working on it... (task `{task_id[:8]}`)",
        mention_author=False,
    )

    # Poll for completion and post result
    asyncio.create_task(
        _post_result_when_ready(task_id, discord_message)
    )

async def _post_result_when_ready(task_id: str, discord_message):
    """Poll task status and reply when done."""
    for _ in range(120):  # max 120 × 1s = 2 minutes
        status = task_runner.get_status(task_id)
        if status and status.value in ("completed", "failed"):
            # Result is stored in Redis by the agent
            result = await redis.get(f"task_result:{task_id}")
            if result:
                await discord_message.reply(result.decode())
            return
        await asyncio.sleep(1)

    await discord_message.reply("⚠️ Task timed out. Please try again.")
```

---

## 7. CI/CD Pipeline (SSH + Docker Compose)

This replaces the Helm-based GitHub Actions workflow from doc 09.

```yaml
# .github/workflows/deploy.yml
name: Deploy to Production

on:
  push:
    branches: [main]
  workflow_dispatch:
    inputs:
      image_tag:
        description: 'Image tag to deploy'
        required: false
        default: 'latest'

env:
  REGISTRY: ghcr.io
  ORG: your-org
  IMAGE_TAG: ${{ github.sha }}

jobs:
  build:
    runs-on: ubuntu-latest
    permissions:
      contents: read
      packages: write
    strategy:
      matrix:
        service:
          - nanoclaw
          - openclaw-api
          - mcp-context
          - mcp-memory
          - mcp-skills
          - mcp-guardrails
          - mcp-hitl
          - mcp-exec
          - mcp-db
          - mcp-github
          - prefect-agent
    steps:
      - uses: actions/checkout@v4

      - name: Log in to GHCR
        uses: docker/login-action@v3
        with:
          registry: ${{ env.REGISTRY }}
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}

      - name: Build and push ${{ matrix.service }}
        uses: docker/build-push-action@v5
        with:
          context: ./services/${{ matrix.service }}
          push: true
          tags: |
            ${{ env.REGISTRY }}/${{ env.ORG }}/${{ matrix.service }}:${{ env.IMAGE_TAG }}
            ${{ env.REGISTRY }}/${{ env.ORG }}/${{ matrix.service }}:latest
          cache-from: type=registry,ref=${{ env.REGISTRY }}/${{ env.ORG }}/${{ matrix.service }}:latest
          cache-to: type=inline

  deploy:
    needs: build
    runs-on: ubuntu-latest
    environment: production
    steps:
      - uses: actions/checkout@v4

      - name: Deploy via SSH
        uses: appleboy/ssh-action@v1
        with:
          host: ${{ secrets.DROPLET_IP }}
          username: deploy
          key: ${{ secrets.DEPLOY_SSH_KEY }}
          script: |
            cd /opt/openclaw

            # Pull latest compose file from repo
            git fetch origin main
            git checkout origin/main -- docker-compose.yml Caddyfile prometheus/ grafana/

            # Update image tag and pull new images
            export IMAGE_TAG=${{ env.IMAGE_TAG }}
            docker compose pull

            # Rolling restart: stop old, start new
            # ~5s downtime per service (acceptable for MVP)
            docker compose up -d --remove-orphans

            # Verify health
            sleep 15
            docker compose ps --format json | python3 -c "
            import sys, json
            services = [json.loads(l) for l in sys.stdin]
            unhealthy = [s['Name'] for s in services if s.get('Health') == 'unhealthy']
            if unhealthy:
                print(f'UNHEALTHY: {unhealthy}')
                sys.exit(1)
            print('All services healthy')
            "

            # Prune old images (keep last 3)
            docker image prune -f --filter "until=72h"

  notify:
    needs: deploy
    runs-on: ubuntu-latest
    if: always()
    steps:
      - name: Discord notification
        uses: sarisia/actions-status-discord@v1
        with:
          webhook: ${{ secrets.DISCORD_DEPLOY_WEBHOOK }}
          status: ${{ needs.deploy.result }}
          title: "OpenClaw Deploy"
          description: "Tag: `${{ env.IMAGE_TAG }}`"
```

### Droplet Initial Setup Script

```bash
#!/usr/bin/env bash
# scripts/provision-droplet.sh — run once on fresh Droplet
set -euo pipefail

# System
apt-get update && apt-get upgrade -y
apt-get install -y curl git ufw fail2ban

# Docker
curl -fsSL https://get.docker.com | sh
systemctl enable docker

# Deploy user (no sudo, only docker group)
useradd -m -s /bin/bash deploy
usermod -aG docker deploy
mkdir -p /home/deploy/.ssh
# Add your GitHub Actions deploy key:
# echo "ssh-ed25519 AAAA..." >> /home/deploy/.ssh/authorized_keys
chmod 700 /home/deploy/.ssh
chmod 600 /home/deploy/.ssh/authorized_keys
chown -R deploy:deploy /home/deploy/.ssh

# App directory
mkdir -p /opt/openclaw
chown deploy:deploy /opt/openclaw

# Firewall: only 22, 80, 443
ufw default deny incoming
ufw default allow outgoing
ufw allow ssh
ufw allow http
ufw allow https
ufw enable

# Clone repo
su - deploy -c "git clone https://github.com/your-org/openclaw /opt/openclaw"

# Copy .env (do this manually or via secrets manager)
# scp .env deploy@<DROPLET_IP>:/opt/openclaw/.env

echo "Droplet provisioned. Copy .env and run: docker compose up -d"
```

---

## 8. Database Schema & Migrations

PostgreSQL schema initialization is identical to the full architecture. The only difference is how migrations are triggered:

```bash
# K8s: helm hook job runs migrations
# Docker Compose: one-off migration container

docker compose run --rm openclaw-api python -m alembic upgrade head
```

Add to CI/CD before the `docker compose up` step:

```yaml
# In deploy job, after pull, before up:
- name: Run DB migrations
  uses: appleboy/ssh-action@v1
  with:
    script: |
      cd /opt/openclaw
      docker compose run --rm openclaw-api python -m alembic upgrade head
```

---

## 9. Secrets Management

Without Kubernetes secrets or Vault, secrets are managed via a `.env` file on the Droplet plus GitHub Actions secrets.

### .env.example

```bash
# .env.example — commit this, not .env

# Database
DATABASE_URL=postgresql+asyncpg://user:pass@db-host:25060/openclaw?sslmode=require
LANGFUSE_DATABASE_URL=postgresql+asyncpg://user:pass@db-host:25060/langfuse?sslmode=require
PREFECT_DATABASE_URL=postgresql+asyncpg://user:pass@db-host:25060/prefect?sslmode=require
REDIS_URL=rediss://default:pass@redis-host:25061/0

# Anthropic
ANTHROPIC_API_KEY=sk-ant-...

# Discord
DISCORD_TOKEN=...
DISCORD_APPLICATION_ID=...
HITL_DISCORD_WEBHOOK_URL=https://discord.com/api/webhooks/...

# GitHub App
GITHUB_APP_ID=...
GITHUB_APP_PRIVATE_KEY=...

# Platform
API_SECRET_KEY=...
BASE_DOMAIN=openclaw.ai
ACME_EMAIL=admin@openclaw.ai
ALLOWED_CIDR=YOUR.IP.ADDRESS/32

# Observability
GRAFANA_ADMIN_PASSWORD=...
LANGFUSE_NEXTAUTH_SECRET=...
LANGFUSE_SALT=...

# Modal (DSPy optimization)
MODAL_TOKEN_ID=...
MODAL_TOKEN_SECRET=...
```

### Secret Rotation Procedure

```bash
# 1. Update .env on Droplet
ssh deploy@DROPLET_IP "nano /opt/openclaw/.env"

# 2. Restart affected services only
ssh deploy@DROPLET_IP "cd /opt/openclaw && docker compose restart openclaw-api nanoclaw"
```

> **Future:** When migrating to K8s, secrets move to SOPS + age encrypted secrets in the repo, loaded by Helm.

---

## 10. Observability (Unchanged)

The observability stack is identical to the full architecture — Langfuse, Prometheus, Grafana all run as Docker Compose services rather than K8s pods. No functional difference.

```yaml
# prometheus/prometheus.yml
global:
  scrape_interval: 15s
  evaluation_interval: 15s

scrape_configs:
  - job_name: openclaw-api
    static_configs:
      - targets: ['openclaw-api:8000']
    metrics_path: /metrics

  - job_name: nanoclaw
    static_configs:
      - targets: ['nanoclaw:9090']
    metrics_path: /metrics

  - job_name: mcp-guardrails
    static_configs:
      - targets: ['mcp-guardrails:8004']
    metrics_path: /metrics

  - job_name: caddy
    static_configs:
      - targets: ['caddy:2019']
    metrics_path: /metrics
```

Key metrics to alert on (same as full architecture):
- `langfuse_task_latency_p95 > 30s` → page on-call
- `openclaw_guardrail_blocks_total` rate spike → security review
- `openclaw_concurrent_tasks > 15` → approaching asyncio limit
- `container_memory_usage_bytes > 14G` → Droplet approaching OOM

---

## 11. Autonomous Improvement Loop (Unchanged)

The nightly improvement loop is entirely external to the Droplet:

```
Prefect-agent (Compose) → triggers Modal serverless function
Modal → runs DSPy MIPROv2 optimization
Modal → writes updated prompts back to PostgreSQL
All MCP servers → read updated prompts on next request
```

No Kubernetes required. The `prefect-agent` container connects to the Prefect server and dispatches Modal tasks. This is identical to the K8s approach — only the container runtime changes.

---

## 12. Scaling Strategy

### Vertical Scaling (First Response)

The single Droplet approach scales vertically before requiring any architectural change:

| Traffic Level | Droplet | Cost | Notes |
|--------------|---------|------|-------|
| MVP (0–10 concurrent users) | s-8vcpu-16gb | $96/mo | Current spec |
| Early Growth (10–30 concurrent) | s-16vcpu-32gb | $192/mo | Double resources |
| Late Growth (30–50 concurrent) | m-16vcpu-128gb | $336/mo | Memory-optimized |
| Scale threshold | → Migrate to K8s | — | See §13 |

Vertical scaling is a 5-minute operation with ~2 minutes of downtime:

```bash
# DigitalOcean Droplet resize
doctl compute droplet-action resize DROPLET_ID --size s-16vcpu-32gb --resize-disk=false
```

### Horizontal Scaling Without K8s (Partial)

Before migrating to K8s, stateless services can be scaled horizontally with a DigitalOcean Load Balancer ($12/mo):

```yaml
# Multiple Droplets + DO Load Balancer
# Stateless services: openclaw-api, all MCP servers
# State remains in managed PostgreSQL + Redis (unaffected)

docker compose up -d --scale openclaw-api=2 --scale mcp-context=2
```

This requires updating Caddy to upstream to multiple backends or using a DO Load Balancer in front of multiple Droplets.

---

## 13. Migration Path: Docker Compose → Kubernetes

When any of these triggers are hit, migrate to the K8s architecture from doc 09:

| Trigger | Reason |
|---------|--------|
| >40 concurrent users | asyncio semaphore saturated, need Celery |
| Revenue >$5,000/mo MRR | K8s overhead is justified |
| Task loss on crash is unacceptable | Need Celery durable queues |
| Need zero-downtime deploys | Rolling updates require K8s |
| Need multi-region | K8s is the right tool |
| Security audit requires pod isolation | K8s network policies |

### Migration Steps

```bash
# Step 1: Convert Compose to K8s manifests
brew install kompose
kompose convert -f docker-compose.yml -o k8s/

# Step 2: Review and adjust generated manifests
# kompose output is a starting point, not production-ready
# Key additions needed:
# - Resource requests/limits
# - Liveness/readiness probes
# - PodDisruptionBudgets
# - HorizontalPodAutoscalers
# - NetworkPolicies

# Step 3: Provision DOKS cluster (from doc 09)
doctl kubernetes cluster create openclaw-prod \
  --region nyc3 --node-pool "name=main;size=s-2vcpu-4gb;count=3"

# Step 4: Apply manifests
kubectl apply -f k8s/

# Step 5: Run both environments in parallel for 48h
# Validate correctness, then cut over DNS

# Step 6: Decommission Droplet
```

The entire migration is estimated at 2–3 days of engineering time.

---

## 14. Trade-Off Analysis

### 14.1 What You Gain With This Approach

| Benefit | Impact |
|---------|--------|
| **Faster iteration** | `docker compose up -d` vs `helm upgrade`; debug with `docker logs` vs `kubectl logs -n openclaw pod/...` |
| **Lower cost** | ~$48/mo savings, ~$576/year |
| **Simpler mental model** | All services on one machine; no cluster networking to debug |
| **Easier local dev** | Same `docker-compose.yml` runs locally |
| **No cluster management** | No etcd, no node upgrades, no control plane concerns |
| **Faster cold starts** | Container restarted in <5s vs K8s pod scheduling |

### 14.2 What You Lose

| Trade-off | Severity | Mitigation |
|-----------|----------|-----------|
| **Single point of failure** | HIGH | DO Droplet SLA is 99.99%; managed DB/Redis are HA; most downtime is planned maintenance |
| **No task durability** | MEDIUM | Acceptable at MVP — users can retry. Reintroduce Celery at Growth |
| **No auto-scaling** | MEDIUM | Vertical scale is fast (5 min); volume predictable at MVP |
| **~5s deploy downtime** | LOW | MVP users tolerate this; fix at Growth with blue-green |
| **No pod isolation** | LOW | Security is at application + DB layer (RLS), not infra |
| **No rolling deploys** | LOW | Ship smaller, ship often; schedule deploys during low-traffic windows |
| **Memory contention** | MEDIUM | 16GB shared across all services; mcp-guardrails is the largest consumer at ~4GB |
| **No built-in service mesh** | LOW | Docker Compose bridge network is sufficient for trusted internal services |

### 14.3 Risk Matrix

```
         HIGH IMPACT
              │
    SPOF      │    Task Loss
  (Droplet)   │   on Crash
              │
──────────────┼──────────────
              │
  Memory      │   No Zero-
  Contention  │   Downtime
              │   Deploy
         LOW IMPACT

LOW PROBABILITY    HIGH PROBABILITY
```

**SPOF (High Impact, Low Probability):** A Droplet failure is rare (DO 99.99% SLA) and recovery is fast (snapshot restore in <5 min). Managed databases are HA. Acceptable at MVP.

**Task Loss on Crash (High Impact, Medium Probability):** An OOM kill or bad deploy could lose in-flight tasks. Mitigated by: memory limits per container, Prometheus OOM alerts, and Discord users receiving a clear error message to retry.

**Memory Contention (Medium Impact, Medium Probability):** At 16GB, there is ~10GB for application containers after OS overhead. Monitor with Prometheus; alert at 75% usage.

### 14.4 Decision Framework

```
Start MVP here (this doc)
        │
        ├── Revenue > $5K MRR?  ──YES──► Migrate to K8s (doc 09)
        │
        ├── >40 concurrent?     ──YES──► Add Celery + consider K8s
        │
        ├── Task loss hurts?    ──YES──► Add Celery + Redis queue
        │
        ├── Need 0-downtime?    ──YES──► Blue-green Compose or K8s
        │
        └── Still fine?         ────────► Vertical scale Droplet
```

---

## 15. Security Considerations

### Container Security

The simplified approach maintains all application-level security. Infrastructure-level isolation is reduced compared to K8s but mitigated at the application layer:

```yaml
# Applied to mcp-exec (code execution container — highest risk)
security_opt:
  - no-new-privileges:true
  - seccomp:./seccomp/exec-profile.json  # restrict syscalls
cap_drop:
  - ALL
cap_add: []  # No capabilities added back
read_only: true  # Read-only root filesystem
tmpfs:
  - /tmp:size=512m,noexec  # Writable temp, no execution
```

### Network Isolation

Docker Compose bridge network provides basic service isolation:

```yaml
# Services not exposed to host — only Caddy is exposed
# Internal services only reachable within Docker network
networks:
  openclaw:
    internal: false   # Allow outbound (Anthropic API, GitHub, etc.)
    # NOTE: All services can reach each other — no K8s NetworkPolicy equivalent
    # Mitigated by: all containers are our own code, no untrusted containers
```

### Firewall

UFW rules on the Droplet:
- **Port 22:** SSH (deploy key only, no password auth)
- **Port 80:** HTTP (redirects to HTTPS via Caddy)
- **Port 443:** HTTPS (Caddy TLS termination)
- **All other ports:** BLOCKED

Internal service ports (8001–8008) are not exposed to the host.

---

## 16. Summary Comparison

| Dimension | Simplified MVP (this doc) | Full K8s (doc 09) |
|-----------|--------------------------|-------------------|
| Infra cost | ~$171/mo | ~$219/mo |
| Setup time | ~4 hours | ~2 days |
| Deploy complexity | Low (SSH + compose) | High (Helm + kubectl) |
| Debug experience | Good (docker logs) | Moderate (kubectl logs, exec) |
| Horizontal scaling | Limited | Native (HPA) |
| Task durability | Low (in-memory) | High (Celery + Redis) |
| Zero-downtime deploy | No | Yes |
| Auto-recovery | Basic (restart: unless-stopped) | Full (K8s self-healing) |
| Migration cost | — | 2–3 days engineering |
| Recommended for | MVP, <$5K MRR | Growth, >$5K MRR |

---

## 17. Quick-Start Checklist

```bash
# 1. Provision Droplet
doctl compute droplet create openclaw-prod \
  --image ubuntu-22-04-x64 \
  --size s-8vcpu-16gb \
  --region nyc3 \
  --ssh-keys YOUR_SSH_KEY_ID

# 2. Run provision script
scp scripts/provision-droplet.sh root@DROPLET_IP:/tmp/
ssh root@DROPLET_IP "bash /tmp/provision-droplet.sh"

# 3. Add GitHub Actions deploy key
# In GitHub: Settings → Deploy keys → Add key (write access)
# On Droplet: echo "..." >> /home/deploy/.ssh/authorized_keys

# 4. Create managed PostgreSQL + Redis
#    (see doc 09 §5 for exact doctl commands)

# 5. Copy .env to Droplet
scp .env deploy@DROPLET_IP:/opt/openclaw/.env

# 6. Run migrations
ssh deploy@DROPLET_IP "cd /opt/openclaw && docker compose run --rm openclaw-api python -m alembic upgrade head"

# 7. Start all services
ssh deploy@DROPLET_IP "cd /opt/openclaw && docker compose up -d"

# 8. Verify
ssh deploy@DROPLET_IP "docker compose ps"
curl https://api.YOUR_DOMAIN/health

# 9. Push to main → automatic deploy from GitHub Actions
```

**Estimated time to first deploy:** ~4 hours including DNS propagation.

---

*See [doc 09: Implementation Plan](09-implementation-plan.md) for the K8s-based Growth architecture this design migrates to. See [doc 08: Unified Platform](08-unified-platform-architecture.md) for the interface layer that sits above this infrastructure.*
