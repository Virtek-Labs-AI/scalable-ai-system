# 10 — Implementation Plan: Hosting, Infrastructure & Cost Analysis

> **Related docs:** [06 System Design](06-system-design.md) · [08 Unified Platform Architecture](08-unified-platform-architecture.md) · [07 Hierarchical Context Architecture](07-hierarchical-context-architecture.md)

This document covers the concrete path from code to production: where to host, what to provision, how much it costs at each growth stage, and what to spin up first.

---

## Table of Contents

1. [Hosting Platform Comparison](#1-hosting-platform-comparison)
2. [Infrastructure Topology](#2-infrastructure-topology)
3. [Stage 1 — MVP (Months 1–3)](#3-stage-1--mvp-months-13)
4. [Stage 2 — Growth (Months 4–9)](#4-stage-2--growth-months-49)
5. [Stage 3 — Scale (Months 10+)](#5-stage-3--scale-months-10)
6. [Detailed Cost Analysis](#6-detailed-cost-analysis)
7. [Provision This First](#7-provision-this-first)
8. [CI/CD Pipeline](#8-cicd-pipeline)
9. [Database Strategy](#9-database-strategy)
10. [Secrets & Configuration Management](#10-secrets--configuration-management)
11. [Disaster Recovery & Backups](#11-disaster-recovery--backups)
12. [Migration Path: DO → AWS](#12-migration-path-do--aws)

---

## 1. Hosting Platform Comparison

### Decision Matrix

| Criterion | DigitalOcean | AWS | Winner |
|-----------|-------------|-----|--------|
| **Operational simplicity** | Excellent — fewer moving parts | Complex — IAM, VPC, subnets, SGs all separate | ✅ DO |
| **Managed PostgreSQL** | DO Managed Postgres (simple, solid) | RDS Postgres (more config, more power) | Tie |
| **pgvector support** | ✅ Supported | ✅ Supported (RDS/Aurora) | Tie |
| **Kubernetes** | DOKS — straightforward, affordable | EKS — powerful but expensive ($0.10/hr cluster fee + nodes) | ✅ DO |
| **Container registry** | DO Container Registry (included in some plans) | ECR ($0.10/GB-month, free under 50GB with free tier) | ✅ DO (simpler) |
| **Serverless compute** | Functions (limited, not great for Python) | Lambda (excellent, especially with container images) | ✅ AWS |
| **GPU instances** | GPU Droplets (H100, available) | EC2 P/G instances (wider selection) | ✅ AWS |
| **Modal.ai compatibility** | Agnostic — Modal is external SaaS | Agnostic — Modal is external SaaS | Tie |
| **Pricing predictability** | Very predictable flat-rate Droplets | Complex — egress, requests, NAT GW add up | ✅ DO |
| **Egress costs** | Free between DO resources same region | $0.09/GB out to internet, VPC free within region | ✅ DO |
| **Enterprise features** | Limited (no WAF, basic compliance) | Comprehensive (SOC2, HIPAA, FedRAMP, WAF) | ✅ AWS |
| **Support cost** | $50/mo Developer plan | $100/mo Developer, $300/mo Business | ✅ DO |
| **Learning curve** | Low — straightforward UI + API | High — hundreds of services, complex IAM | ✅ DO |
| **Uptime SLA** | 99.99% for Managed DBs | 99.99% for RDS Multi-AZ | Tie |

### Recommendation by Stage

```
MVP / Early Growth:    DigitalOcean
  → Faster to ship, predictable costs, simpler operations
  → Managed Postgres + DOKS covers all needs
  → Avoid AWS complexity until justified

Growth / Enterprise:   AWS (or hybrid)
  → When clients require AWS residency or compliance certs
  → Lambda for DSPy batch jobs instead of Modal (optional)
  → EKS when team has dedicated DevOps

Current Recommendation: DigitalOcean for all stages until either:
  (a) Monthly infra cost > $3,000/mo (AWS reserved instances save money at scale), or
  (b) Enterprise client requires AWS-hosted deployment
```

### Why Not GCP / Azure?

| Platform | Reason to Skip for Now |
|----------|----------------------|
| **GCP** | Strong AI/ML but operational complexity similar to AWS; Vertex AI not in our stack |
| **Azure** | Best for Microsoft shops; no native advantage for our Python/PostgreSQL stack |
| **Fly.io** | Great DX but immature managed databases; limited to small teams |
| **Render** | Good for simple apps; managed Postgres lacks pgvector maturity |

---

## 2. Infrastructure Topology

### MVP Topology (DigitalOcean)

```
┌─────────────────────────────────────────────────────────────────┐
│                    DigitalOcean — NYC3 Region                    │
│                                                                   │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │                   DOKS Cluster (3 nodes)                 │    │
│  │                                                           │    │
│  │  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐  │    │
│  │  │  NanoClaw    │  │  OpenClaw    │  │  MCP Servers │  │    │
│  │  │  (Discord    │  │  Workers     │  │  (context,   │  │    │
│  │  │   bot)       │  │  (Celery)    │  │   memory,    │  │    │
│  │  │              │  │              │  │   skills,    │  │    │
│  │  │  2 replicas  │  │  3 workers   │  │   guardrails)│  │    │
│  │  └──────────────┘  └──────────────┘  └──────────────┘  │    │
│  │                                                           │    │
│  │  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐  │    │
│  │  │  Prefect     │  │  Langfuse    │  │  Prometheus  │  │    │
│  │  │  (scheduler) │  │  (LLM obs)   │  │  + Grafana   │  │    │
│  │  └──────────────┘  └──────────────┘  └──────────────┘  │    │
│  └─────────────────────────────────────────────────────────┘    │
│                                                                   │
│  ┌──────────────────────┐   ┌──────────────────────────────┐    │
│  │  DO Managed Postgres  │   │  DO Managed Redis            │    │
│  │  (Primary + 1 standby)│   │  (Celery broker + cache)     │    │
│  │  pgvector enabled     │   │  1GB RAM                     │    │
│  │  4GB RAM, 2 vCPU      │   └──────────────────────────────┘    │
│  └──────────────────────┘                                        │
│                                                                   │
│  ┌──────────────────────┐   ┌──────────────────────────────┐    │
│  │  DO Spaces (S3-compat)│   │  DO Container Registry       │    │
│  │  (artifacts, exports) │   │  (Docker images)             │    │
│  └──────────────────────┘   └──────────────────────────────┘    │
└─────────────────────────────────────────────────────────────────┘

External Services (SaaS — not hosted by us):
  ├── Anthropic Claude API (direct API calls for simple tasks)
  ├── Claude Code SDK (subscription seats — not an infrastructure component)
  ├── Modal.ai (serverless DSPy optimization runs)
  └── Discord (NanoClaw interface)
```

### Growth Topology Additions

```
Additional components at growth stage:
  ├── DOKS cluster: scale to 5–7 nodes
  ├── DO Managed Postgres: upgrade to 8GB RAM, add read replica
  ├── Redis: upgrade to 4GB RAM cluster mode
  ├── Add: DO Load Balancer (if not using DOKS ingress)
  ├── Add: Separate DOKS node pool for memory-intensive MCP servers
  └── Add: Second Claude Code SDK seat ($200/mo)
```

---

## 3. Stage 1 — MVP (Months 1–3)

### What Gets Built

Corresponds to Phases 1–2 of the 18-week roadmap (Foundations + Memory/Parallelism).

```
Week 1–3:   Core infrastructure up, NanoClaw → OpenClaw bridge working,
            MCP servers deployed, PostgreSQL + pgvector live,
            Langfuse + Prometheus + Grafana from day 1

Week 4–6:   Memory layer working (pgvector search, Graphiti knowledge graph),
            asyncio WorkerPool handling parallel agent execution,
            Celery + Redis task queue operational
```

### DOKS Node Pool — MVP

```yaml
# MVP node pool configuration
cluster:
  name: openclaw-mvp
  region: nyc3
  version: latest-stable

node_pools:
  - name: general
    size: s-4vcpu-8gb      # $48/mo per node
    count: 3               # min 3 for HA
    auto_scale: false      # keep costs predictable at MVP
    labels:
      pool: general
```

**Node sizing rationale:**
- `s-4vcpu-8gb` ($48/mo): Sufficient for NanoClaw + MCP servers + Celery workers
- 3 nodes = 12 vCPU + 24GB RAM total cluster capacity
- Kubernetes overhead ~15%: usable ~10 vCPU + 20GB RAM

### Kubernetes Resource Allocations — MVP

| Service | Replicas | CPU Request | Memory Request | CPU Limit | Memory Limit |
|---------|----------|-------------|----------------|-----------|--------------|
| nanoclaw | 2 | 100m | 256Mi | 500m | 512Mi |
| openclaw-api | 2 | 200m | 512Mi | 1000m | 1Gi |
| celery-worker | 3 | 500m | 1Gi | 2000m | 2Gi |
| mcp-context | 2 | 200m | 512Mi | 500m | 1Gi |
| mcp-memory | 2 | 200m | 512Mi | 500m | 1Gi |
| mcp-skills | 1 | 100m | 256Mi | 500m | 512Mi |
| mcp-guardrails | 2 | 300m | 512Mi | 1000m | 1Gi |
| mcp-hitl | 1 | 100m | 256Mi | 300m | 512Mi |
| prefect-agent | 1 | 100m | 256Mi | 500m | 512Mi |
| langfuse | 1 | 200m | 512Mi | 500m | 1Gi |
| prometheus | 1 | 100m | 512Mi | 500m | 1Gi |
| grafana | 1 | 100m | 256Mi | 300m | 512Mi |

**Total requests:** ~2.4 vCPU, ~6.5Gi RAM — fits comfortably in 3-node cluster.

### Managed Services — MVP

| Service | DO Plan | vCPU | RAM | Storage | Monthly |
|---------|---------|------|-----|---------|---------|
| PostgreSQL | `db-s-2vcpu-4gb` | 2 | 4GB | 38GB SSD | $50 |
| Redis | `db-s-1vcpu-1gb` | 1 | 1GB | 10GB SSD | $15 |

**PostgreSQL extensions to enable at provision time:**
```sql
CREATE EXTENSION IF NOT EXISTS vector;        -- pgvector for embeddings
CREATE EXTENSION IF NOT EXISTS "uuid-ossp";   -- UUID generation
CREATE EXTENSION IF NOT EXISTS pg_trgm;       -- trigram search
CREATE EXTENSION IF NOT EXISTS btree_gin;     -- composite index support
```

---

## 4. Stage 2 — Growth (Months 4–9)

### Triggers for Upgrading

Upgrade from MVP → Growth when ANY of:
- Celery queue depth consistently > 50 tasks
- Average task latency > 30s (p95)
- PostgreSQL CPU > 60% sustained
- Memory usage > 70% on any node
- Onboarding 3rd paying client
- Monthly LLM cost approaching $1,000

### DOKS Node Pool — Growth

```yaml
node_pools:
  - name: general
    size: s-4vcpu-8gb
    count: 3
    auto_scale: true
    min_nodes: 3
    max_nodes: 6

  - name: memory-intensive
    size: s-8vcpu-16gb     # $96/mo per node
    count: 1               # for MCP memory servers + Langfuse
    labels:
      pool: memory-intensive
```

### Managed Services — Growth

| Service | DO Plan | vCPU | RAM | Storage | Monthly |
|---------|---------|------|-----|---------|---------|
| PostgreSQL | `db-s-4vcpu-8gb` | 4 | 8GB | 115GB SSD | $100 |
| PostgreSQL Read Replica | `db-s-2vcpu-4gb` | 2 | 4GB | 38GB SSD | $50 |
| Redis | `db-s-2vcpu-4gb` | 2 | 4GB | 38GB SSD | $50 |

---

## 5. Stage 3 — Scale (Months 10+)

### Architecture Changes at Scale

```
Scale-stage additions:
  ├── Separate DOKS clusters per environment (prod / staging)
  ├── PostgreSQL: Consider Crunchy Data or Neon for serverless scaling
  ├── Redis: Upgrade to Redis Cluster (3 primary + 3 replica shards)
  ├── Add: CDN for static assets (DO CDN or Cloudflare free tier)
  ├── Add: Dedicated ingress nodes (nginx-ingress or Traefik)
  └── Consider: AWS migration if enterprise clients require it
```

### When to Evaluate AWS Migration

| Trigger | Action |
|---------|--------|
| Monthly infra > $3,000 | Compare reserved instances vs DO Droplets |
| Enterprise client requires AWS | Deploy client-specific environment on AWS |
| Need Lambda-class auto-scale | Add AWS Lambda for DSPy runs (replace Modal) |
| Compliance requirement (SOC2 Type II) | AWS has more mature compliance tooling |
| Team grows to 3+ DevOps engineers | AWS complexity becomes manageable |

---

## 6. Detailed Cost Analysis

### MVP Monthly Cost (~$590–650/month)

```
Infrastructure:
  DOKS cluster (3x s-4vcpu-8gb nodes)        $144/mo
  DO Managed PostgreSQL (db-s-2vcpu-4gb)      $ 50/mo
  DO Managed Redis (db-s-1vcpu-1gb)           $ 15/mo
  DO Spaces (250GB + egress)                  $  7/mo
  DO Container Registry (starter)             $  0/mo (free tier)
  DO Load Balancer (if needed outside DOKS)   $  0/mo (use DOKS ingress)
  ─────────────────────────────────────────   ───────
  Infrastructure subtotal                     $216/mo

AI / LLM:
  Claude Code SDK (1 seat)                    $200/mo
  Anthropic API — Claude Haiku (simple tasks) $ 30/mo  (est. ~15M tokens)
  Anthropic API — Embeddings (text-embedding-3-small) $ 5/mo
  Modal.ai (DSPy nightly runs, burst)         $ 20/mo  (est. 5h GPU)
  ─────────────────────────────────────────   ───────
  AI/LLM subtotal                             $255/mo

Observability / External:
  Langfuse (self-hosted in cluster)           $  0/mo
  Prometheus + Grafana (self-hosted)          $  0/mo
  GitHub (Virtek-Labs-AI org)                 $ 4/mo   (Team plan per seat)
  ─────────────────────────────────────────   ───────
  Observability subtotal                      $  4/mo

Support / Misc:
  DigitalOcean support (Developer plan)       $ 50/mo
  Domain / DNS                                $  2/mo
  Misc (backups, snapshots)                   $ 10/mo
  ─────────────────────────────────────────   ───────
  Misc subtotal                               $ 62/mo

────────────────────────────────────────────────────
TOTAL MVP                                    ~$537/mo
  + buffer for spikes / overages             + $60/mo
  ─────────────────────────────────────────   ───────
  Realistic MVP budget                       ~$597/mo
```

### Growth Monthly Cost (~$1,050–1,200/month)

```
Infrastructure:
  DOKS cluster (3–6x s-4vcpu-8gb + 1x s-8vcpu-16gb)  $240–$384/mo
  DO Managed PostgreSQL (db-s-4vcpu-8gb)               $100/mo
  DO Managed PostgreSQL Read Replica                   $ 50/mo
  DO Managed Redis (db-s-2vcpu-4gb)                    $ 50/mo
  DO Spaces (500GB + egress)                           $ 12/mo
  ─────────────────────────────────────────────────    ────────
  Infrastructure subtotal                              $452–$596/mo

AI / LLM:
  Claude Code SDK (2 seats)                            $400/mo
  Anthropic API — Haiku                                $ 60/mo
  Anthropic API — Embeddings                           $ 10/mo
  Modal.ai (DSPy + expanded jobs)                      $ 40/mo
  ─────────────────────────────────────────            ────────
  AI/LLM subtotal                                      $510/mo

Support / Misc:
  DigitalOcean Developer support                       $ 50/mo
  Misc                                                 $ 20/mo
  ─────────────────────────────────────────            ────────
  Misc subtotal                                        $ 70/mo

────────────────────────────────────────────────────
TOTAL GROWTH                                         ~$1,032–$1,176/mo
  + buffer                                           + $75/mo
  ─────────────────────────────────────────────────   ────────
  Realistic growth budget                            ~$1,107–$1,251/mo
```

### Cost Per Client (Revenue Model Reference)

```
Assuming 3 active clients at Growth stage:

Platform cost:           ~$1,150/mo
Per-client attribution:  ~$383/mo (infra + AI share)

Suggested client pricing:
  Starter tier:    $500/mo  (margin: ~$117/mo or ~30%)
  Professional:    $1,500/mo (margin: ~$1,117/mo or ~74%)
  Enterprise:      $3,000+/mo (custom)

Break-even: 3 Starter clients or 1 Professional client
Target: 2 Professional + 1–2 Starter = ~$3,500–$4,500/mo revenue
```

### AWS Equivalent Costs (for comparison)

If migrating to AWS at Growth stage:

```
EKS cluster fee:                  $ 72/mo  (2 clusters × $0.10/hr)
EC2 nodes (6x m6a.xlarge RI 1yr): $486/mo  ($81/mo each)
RDS PostgreSQL (db.r6g.large):    $175/mo
RDS Read Replica:                 $175/mo
ElastiCache Redis (cache.r6g.large): $140/mo
S3 (500GB + requests):            $ 15/mo
ECR:                              $  5/mo
CloudWatch Logs + metrics:        $ 30/mo
NAT Gateway:                      $ 45/mo
────────────────────────────────  ────────
AWS Infrastructure subtotal:     ~$1,143/mo  (vs DO ~$520/mo)

AI/LLM costs: Same (external services)

AWS premium over DO at Growth:   +$623/mo (+120%)
```

> **Conclusion:** DigitalOcean is significantly cheaper at MVP and Growth stages. The AWS premium is only justified if enterprise compliance, Lambda-scale compute, or client requirements demand it.

---

## 7. Provision This First

A sequenced guide to spinning up infrastructure, ordered by dependency.

### Week 1: Core Infrastructure (Day 1–3)

```bash
# 1. Create DOKS cluster
doctl kubernetes cluster create openclaw-mvp \
  --region nyc3 \
  --node-pool "name=general;size=s-4vcpu-8gb;count=3" \
  --version latest

# 2. Set kubectl context
doctl kubernetes cluster kubeconfig save openclaw-mvp

# 3. Install cert-manager (TLS for MCP server endpoints)
kubectl apply -f https://github.com/cert-manager/cert-manager/releases/latest/download/cert-manager.yaml

# 4. Install nginx ingress
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
helm install ingress-nginx ingress-nginx/ingress-nginx \
  --set controller.service.type=LoadBalancer

# 5. Create namespaces
kubectl create namespace openclaw
kubectl create namespace monitoring
kubectl create namespace celery
```

### Week 1: Managed Databases (Day 1–2, parallel with cluster)

```bash
# 1. Create PostgreSQL cluster
doctl databases create openclaw-pg \
  --engine pg \
  --version 16 \
  --size db-s-2vcpu-4gb \
  --region nyc3 \
  --num-nodes 2  # primary + standby

# 2. Wait for it to be ready, then get connection string
doctl databases get <id> --format ConnectionURI

# 3. Enable pgvector and other extensions (via psql)
psql $CONNECTION_URI -c "CREATE EXTENSION IF NOT EXISTS vector;"
psql $CONNECTION_URI -c "CREATE EXTENSION IF NOT EXISTS \"uuid-ossp\";"

# 4. Create Redis cluster
doctl databases create openclaw-redis \
  --engine redis \
  --version 7 \
  --size db-s-1vcpu-1gb \
  --region nyc3

# 5. Restrict both databases to DOKS cluster IPs only
doctl databases firewalls append <pg-id> \
  --rule k8s:<cluster-uuid>
doctl databases firewalls append <redis-id> \
  --rule k8s:<cluster-uuid>
```

### Week 1: Observability Stack (Day 2–3)

```bash
# Deploy observability first — must be running before any agents

# 1. Prometheus + Grafana via kube-prometheus-stack
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm install kube-prom prometheus-community/kube-prometheus-stack \
  --namespace monitoring \
  --set grafana.adminPassword=<secure-password>

# 2. Langfuse (self-hosted)
# Use the official Langfuse Helm chart or Docker Compose values
helm repo add langfuse https://langfuse.github.io/langfuse-k8s
helm install langfuse langfuse/langfuse \
  --namespace monitoring \
  --set postgresql.enabled=false \
  --set externalPostgresql.connectionString=$LANGFUSE_DB_URI

# 3. Verify dashboards are accessible
kubectl port-forward svc/kube-prom-grafana 3000:80 -n monitoring
```

### Week 1: Container Registry & CI/CD (Day 3)

```bash
# 1. Create DO Container Registry
doctl registry create openclaw-registry --subscription-tier basic

# 2. Authenticate DOKS to pull from registry
doctl registry kubernetes-manifest | kubectl apply -f -

# 3. Create GitHub Actions secret for registry push
# Add to GitHub repo secrets:
#   DIGITALOCEAN_ACCESS_TOKEN: <token>
#   REGISTRY_NAME: registry.digitalocean.com/openclaw-registry
```

### Week 2: Core Services

**Deploy in this order (each depends on the previous):**

```
1. mcp-db         (database schema migrations, Row-Level Security policies)
2. mcp-context    (depends on: mcp-db for context storage)
3. mcp-memory     (depends on: mcp-db + pgvector for embeddings)
4. mcp-guardrails (depends on: nothing — stateless screening)
5. mcp-skills     (depends on: mcp-db + mcp-context)
6. mcp-hitl       (depends on: mcp-context + notification service)
7. mcp-exec       (depends on: mcp-guardrails)
8. mcp-github     (depends on: nothing — thin wrapper over GitHub API)
9. celery-worker  (depends on: Redis + all MCP servers)
10. openclaw-api  (depends on: celery + all MCP servers)
11. nanoclaw      (depends on: openclaw-api bridge endpoint)
```

### Week 2: Secrets Management

```bash
# Store all secrets in Kubernetes secrets, sourced from environment
# For production, use SOPS + age for secrets encryption in git

# Pattern for creating service secrets
kubectl create secret generic mcp-context-secrets \
  --namespace openclaw \
  --from-literal=DATABASE_URL="$PG_CONNECTION_URI" \
  --from-literal=LANGFUSE_SECRET_KEY="$LANGFUSE_KEY" \
  --from-literal=ANTHROPIC_API_KEY="$ANTHROPIC_KEY"

# Validate secrets are mounted correctly before deploying services
kubectl get secret mcp-context-secrets -n openclaw -o jsonpath='{.data}' | base64 -d
```

---

## 8. CI/CD Pipeline

### GitHub Actions Workflow

```yaml
# .github/workflows/deploy.yml
name: Deploy to DigitalOcean

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

env:
  REGISTRY: registry.digitalocean.com/openclaw-registry
  CLUSTER_NAME: openclaw-mvp

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-python@v5
        with:
          python-version: '3.12'
      - run: pip install -e ".[dev]"
      - run: pytest tests/ --cov --cov-report=xml
      - uses: codecov/codecov-action@v4

  build-push:
    needs: test
    runs-on: ubuntu-latest
    if: github.ref == 'refs/heads/main'
    strategy:
      matrix:
        service: [nanoclaw, openclaw-api, mcp-context, mcp-memory,
                  mcp-skills, mcp-guardrails, mcp-hitl, mcp-exec,
                  mcp-github, celery-worker]
    steps:
      - uses: actions/checkout@v4
      - name: Install doctl
        uses: digitalocean/action-doctl@v2
        with:
          token: ${{ secrets.DIGITALOCEAN_ACCESS_TOKEN }}
      - name: Log in to registry
        run: doctl registry login --expiry-seconds 600
      - name: Build and push
        uses: docker/build-push-action@v5
        with:
          context: services/${{ matrix.service }}
          push: true
          tags: |
            ${{ env.REGISTRY }}/${{ matrix.service }}:${{ github.sha }}
            ${{ env.REGISTRY }}/${{ matrix.service }}:latest

  deploy:
    needs: build-push
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Install doctl
        uses: digitalocean/action-doctl@v2
        with:
          token: ${{ secrets.DIGITALOCEAN_ACCESS_TOKEN }}
      - name: Set kubectl context
        run: doctl kubernetes cluster kubeconfig save ${{ env.CLUSTER_NAME }}
      - name: Deploy with Helm
        run: |
          helm upgrade --install openclaw ./helm/openclaw \
            --namespace openclaw \
            --set image.tag=${{ github.sha }} \
            --set image.registry=${{ env.REGISTRY }} \
            --atomic \
            --timeout 5m
      - name: Verify rollout
        run: |
          kubectl rollout status deployment/openclaw-api -n openclaw
          kubectl rollout status deployment/nanoclaw -n openclaw
```

### Helm Chart Structure

```
helm/
└── openclaw/
    ├── Chart.yaml
    ├── values.yaml              # defaults
    ├── values-production.yaml  # production overrides
    └── templates/
        ├── _helpers.tpl
        ├── deployments/
        │   ├── nanoclaw.yaml
        │   ├── openclaw-api.yaml
        │   ├── celery-worker.yaml
        │   └── mcp-*.yaml
        ├── services/
        ├── ingress.yaml
        ├── hpa.yaml             # HorizontalPodAutoscaler
        └── secrets/             # external-secrets or sealed-secrets
```

---

## 9. Database Strategy

### Schema Initialization

```sql
-- Run once at first deploy via mcp-db migration job

-- Enable extensions
CREATE EXTENSION IF NOT EXISTS vector;
CREATE EXTENSION IF NOT EXISTS "uuid-ossp";

-- Row-Level Security: enforced at DB level
ALTER TABLE client_contexts ENABLE ROW LEVEL SECURITY;
ALTER TABLE agent_memories ENABLE ROW LEVEL SECURITY;
ALTER TABLE skills ENABLE ROW LEVEL SECURITY;

-- RLS policies (example for client_contexts)
CREATE POLICY client_isolation ON client_contexts
  USING (client_id = current_setting('app.current_client_id')::uuid
         OR current_setting('app.role') = 'platform_admin');

-- pgvector index for semantic search
CREATE INDEX ON agent_memories USING hnsw (embedding vector_cosine_ops)
  WITH (m = 16, ef_construction = 64);

-- Partitioning for large tables
CREATE TABLE agent_memories (
  id UUID DEFAULT uuid_generate_v4(),
  client_id UUID NOT NULL,
  content TEXT NOT NULL,
  embedding vector(1536),  -- text-embedding-3-small dimensions
  created_at TIMESTAMPTZ DEFAULT NOW(),
  expires_at TIMESTAMPTZ,
  trust_level FLOAT DEFAULT 0.8
) PARTITION BY RANGE (created_at);

-- Monthly partitions (auto-created by Prefect job)
CREATE TABLE agent_memories_2026_04 PARTITION OF agent_memories
  FOR VALUES FROM ('2026-04-01') TO ('2026-05-01');
```

### Migration Strategy

```python
# Use Alembic for schema migrations
# Run as a Kubernetes Job before each deployment

# alembic.ini points to $DATABASE_URL
# migrations/ contains versioned migration files

# Deployment sequence:
# 1. Run migration Job (kubectl apply -f k8s/jobs/migration.yaml)
# 2. Wait for Job completion
# 3. Deploy new service versions
```

### Backup Configuration

```bash
# DO Managed PostgreSQL: automatic daily backups included
# Retention: 7 days (default), extend to 30 days in settings

# Additionally: weekly logical backup to DO Spaces
# Configured as a Prefect scheduled flow:

# prefect/flows/backup.py
@flow(name="weekly-postgres-backup")
async def backup_to_spaces():
    timestamp = datetime.now().strftime("%Y%m%d")
    subprocess.run([
        "pg_dump", os.environ["DATABASE_URL"],
        "-F", "c",  # custom format (compressed)
        "-f", f"/tmp/backup-{timestamp}.dump"
    ])
    # Upload to DO Spaces
    s3.upload_file(f"/tmp/backup-{timestamp}.dump",
                   "openclaw-backups",
                   f"pg-backups/backup-{timestamp}.dump")
```

---

## 10. Secrets & Configuration Management

### Secret Hierarchy

```
Tier 1 — Build-time (CI/CD only):
  DIGITALOCEAN_ACCESS_TOKEN       (GitHub Actions secret)
  REGISTRY_NAME                   (GitHub Actions secret)

Tier 2 — Cluster-level (Kubernetes secrets):
  ANTHROPIC_API_KEY               (used by all services)
  DATABASE_URL                    (per-namespace)
  REDIS_URL                       (per-namespace)
  LANGFUSE_SECRET_KEY             (monitoring namespace)

Tier 3 — Service-level (mounted per deployment):
  DISCORD_BOT_TOKEN               (nanoclaw only)
  GITHUB_APP_PRIVATE_KEY          (mcp-github only)
  MODAL_TOKEN_ID / SECRET         (prefect worker only)
```

### SOPS + age for GitOps (recommended)

```bash
# Encrypt secrets for safe git storage
brew install sops age

# Generate age key pair
age-keygen -o ~/.config/sops/age/keys.txt

# .sops.yaml at repo root
creation_rules:
  - path_regex: k8s/secrets/.*\.yaml
    age: age1xxxxx...  # your public key

# Encrypt a secrets file
sops --encrypt k8s/secrets/anthropic.yaml > k8s/secrets/anthropic.enc.yaml

# Decrypt at deploy time (key in CI secret)
sops --decrypt k8s/secrets/anthropic.enc.yaml | kubectl apply -f -
```

---

## 11. Disaster Recovery & Backups

### Recovery Time Objectives

| Scenario | RTO Target | RPO Target |
|----------|-----------|-----------|
| Single pod crash | < 30s (K8s restarts) | 0 (stateless pods) |
| Node failure | < 5min (K8s reschedules) | 0 (stateless pods) |
| PostgreSQL primary failure | < 60s (DO auto-failover to standby) | < 1min |
| Redis failure | < 60s (DO auto-failover) | < 1min |
| Full cluster failure | < 30min (redeploy via Helm) | Last backup (up to 24h) |
| Region failure | < 2h (manual failover to second region) | Last backup |

### Runbook: Full Cluster Recovery

```bash
# 1. Provision new cluster in backup region (SFO3)
doctl kubernetes cluster create openclaw-recovery \
  --region sfo3 \
  --node-pool "name=general;size=s-4vcpu-8gb;count=3"

# 2. Restore PostgreSQL from latest backup
doctl databases create openclaw-pg-recovery \
  --engine pg \
  --restore-from-cluster-name openclaw-pg \
  --restore-created-at <timestamp>

# 3. Deploy via Helm with last known good image tag
helm upgrade --install openclaw ./helm/openclaw \
  --set image.tag=<last-good-sha> \
  --set postgresql.host=<new-pg-host>

# 4. Update DNS to point to new cluster LB IP
doctl compute domain records update ...

# Total estimated time: 20–35 minutes
```

---

## 12. Migration Path: DO → AWS

If and when the decision to migrate to AWS is made:

### Phase 1: Parallel Run (2 weeks)

```
1. Provision EKS cluster in us-east-1 alongside DO cluster
2. Set up RDS PostgreSQL with pgvector (pg_vector extension on RDS)
3. Migrate read-only traffic first (Langfuse queries, metrics)
4. Validate performance parity
```

### Phase 2: Data Migration (1 week)

```sql
-- Use pg_dump / pg_restore for full data migration
-- Schedule during low-traffic window (2–4 AM)
-- RLS policies are schema-level — migrate cleanly

pg_dump $DO_PG_URI \
  --format=custom \
  --no-owner \
  --no-acl \
  > full_backup.dump

pg_restore \
  --dbname=$AWS_RDS_URI \
  --format=custom \
  --no-owner \
  full_backup.dump
```

### Phase 3: Cutover (1 day)

```
1. Enable maintenance mode (reject new Discord messages)
2. Final pg_dump + restore for delta
3. Update all DATABASE_URL secrets to RDS endpoint
4. Rolling restart of all services
5. Validate all MCP servers healthy
6. Disable maintenance mode
7. Monitor for 24h before decommissioning DO resources
```

### AWS Service Mapping

| Current (DO) | AWS Equivalent | Notes |
|-------------|----------------|-------|
| DOKS | EKS | Add $0.10/hr cluster fee |
| DO Managed Postgres | RDS PostgreSQL | pgvector supported on db.r6g+ |
| DO Managed Redis | ElastiCache | Redis Cluster mode available |
| DO Spaces | S3 | S3-compatible API — minimal code changes |
| DO Container Registry | ECR | Update image pull secrets |
| DO Load Balancer | ALB or NLB | ALB for HTTP/HTTPS, NLB for TCP |

---

## Summary

| Stage | Monthly Cost | Clients Supported | Key Threshold |
|-------|-------------|-------------------|---------------|
| MVP | ~$597/mo | 1–2 | Ship it |
| Growth | ~$1,150/mo | 3–5 | 3rd paying client |
| Scale | ~$2,000–3,000/mo | 6–10+ | $3k/mo triggers AWS eval |

**Start with DigitalOcean.** It's faster to provision, simpler to operate, and significantly cheaper at early stages. AWS becomes relevant only when client requirements or scale demand it — and this document's migration path makes that transition straightforward.

**Provision order:** DOKS cluster → PostgreSQL + Redis → observability (Langfuse + Prometheus) → MCP servers (bottom-up: mcp-db first) → Celery workers → API → NanoClaw.

> See [08 Unified Platform Architecture](08-unified-platform-architecture.md) for the full system design and [06 System Design](06-system-design.md) for architecture diagrams and the 18-week build roadmap.
