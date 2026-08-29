# Plan 14 — Post-W36 Scale-Out  (Scale-Out)

> **SUBSTRATE SUPERSEDED (2026-08-29).** Where this band names **Railway/Render** as the deploy
> target, read **a self-hosted single-node VPS running Docker Compose**
> ([ADR-0016](../adr/0016-self-hosted-vps-mvp-substrate.md), `_canon/06-tech-decisions.md` 2.1).
> Railway and Render are struck entirely: not the fast path, not a fallback. The band's phase
> order, acceptance checks and everything else it says are unchanged, and the PaaS name is left
> in place below only where it records what the source PDF specified.

> Evolve the W36 modular monolith on the single-node VPS into independently-scaling Kubernetes (EKS/GKE) services along the FROZEN Wave 1-4 seams, then add cells, multi-region active-passive -> active-active, and the documented sharding/graph escalation paths — all demand-driven, sized against the capacity anchors A1=5M users, A4=3,000 tenants, A5=20k peak meetings, A6=100k peak participants.

---

## 0. Band at a glance

- **Phases / weeks:** Post-W36 — this band runs AFTER P13 GA. It is **demand-driven, not a calendar band**: each sub-phase fires only when a real load signal trips its trigger (defined in §8), never on a fixed week. The internal sub-phase IDs below (P14.1 … P14.8) are an *ordering* contract, not a week schedule.
- **Goal:** Peel the monolith into the 20 canonical services on Kubernetes in the frozen Wave 1-4 order; stand up Terraform IaC + GitOps; add cell-based isolation for Enterprise; evolve single-region -> multi-region active-passive -> active-active (in-zone only); and execute the capacity escalation paths (Citus shard-by-`org_id`, AGE -> Neptune/Neo4j Aura, BYOC Redpanda, int8 Qdrant, GPU autoscaling, the C4 ~30% LLM cost reduction).
- **Repos & folders touched:**
  - `mytheus-platform` — **primary** (currently docs-only): new `terraform/`, `helm/nexus/`, `keda/`, `argo/`, `chaos/`, `runbooks/`, `loadtest/`.
  - `nexus-deploy` — **new GitOps repo** (rendered Helm values per `(env, region)`, Argo CD reconciliation source).
  - `mytheus-backend`, `mytheus-realtime`, `mytheus-intelligence`, `mytheus-events` — Dockerfiles, `grpc/` servers, per-service `loadtest/` and Helm subchart values. No domain-logic rewrites: extraction is mechanical (strangler-fig).
  - `mytheus-ui` — production image build only (Next.js 14 standalone). No feature work.
  - `mytheus-mobile` — config only (regional endpoint + cell-affinity tolerance); see §7.
- **Demoable at the end:** (a) `terraform apply` stands up a region cluster from zero; (b) Wave 1 services (`embedding-service`, `reranker-service`, `agent-runtime-service`, `transcription-service`, `realtime-gateway`, `workflow-service`) running on K8s with per-family KEDA/HPA, the monolith drained; (c) an Enterprise tenant served from a dedicated `prod-cell-{slug}` node pool; (d) a Chaos Mesh region-kill GameDay proving Pro RTO <= 30m in-zone; (e) a Citus shard-by-`org_id` cutover rehearsal; (f) a Grafana cost board showing the C4 LLM reduction realized.

---

## 1. Entry preconditions  (must be GREEN before starting)

This band assumes the **entire MVP (P1-P13 / W1-W36) is shipped and GA** as a modular monolith on a self-hosted single-node VPS running Docker Compose, single region, pooled + PostgreSQL RLS. Specifically:

1. **All 20 services exist as modules** behind the locked sync transport (gRPC for AI/real-time tiers, REST for `bff`/CRUD) and the Redpanda bus. The Wave 1-4 seams from [Service Catalog §7](../_canon/01-service-catalog.md) are real boundaries in code, not aspirations.
2. **Identity is Clerk (RESOLVED DECISION 1).** Clerk JWKS verification lives in `mytheus-backend/libs/shared`; the Svix webhook user-sync handler in `identity-tenant-service` is implemented; `CLERK_*` env is present; the Supabase Auth `custom_access_token_hook` + `auth.users -> public.users` trigger have been reverted/removed. `api-gateway` extracts `org_id` once at the edge from the Clerk JWT. **If this is not done, STOP — it is a Band-0/identity precondition, not scale-out work.**
3. **At-least-once + idempotency is live:** transactional outbox on produce, inbox/Redis-dedupe on consume, per-topic DLQ, every topic Avro + Schema Registry BACKWARD compat, partitioned by `org_id` (high-volume topics by `{org_id}:{meeting_id}`). The brief dual-running window during extraction is only safe because of this ([Service Catalog §5](../_canon/01-service-catalog.md)).
4. **RLS FORCE on every tenant-scoped table**, `app.current_org` set transaction-locally via `set_config(..., true)`, PgBouncer transaction pooling. Consumers rehydrate `org_id` from the event header before any write ([Multi-Tenancy §3.1, §4](../06-multi-tenancy.md)).
5. **Observability stack** (Prometheus + Grafana, Sentry, structured JSON logs carrying `org_id`/`family`/`version`/trace IDs) and **GitHub Actions CI/CD with staging-on-merge** are running from MVP and survive verbatim ([Deployment §13, §5](../16-deployment-and-infrastructure.md)).
6. **Capacity anchors are the production sizing target**, never the `2 orgs x 10 users x 5 meetings` seeder ([Capacity §1, §2](../_canon/05-capacity-math.md)).

> **The migration SOURCE changed, and this band was written against the old one.** It was
> drafted as a PaaS-to-Kubernetes move; the source is now a self-hosted single-node VPS
> ([ADR-0016](../adr/0016-self-hosted-vps-mvp-substrate.md)). Renaming the substrate is not
> sufficient, and the differences are real work this band does not yet cost: there are no
> build packs to replace, the box and its Docker volumes are owned rather than managed, so
> every store (PostgreSQL, Redis, Redpanda, and object storage) is a self-hosted data
> migration rather than a managed-service repoint, and the "push deploy from CI" step this
> band assumes it is replacing is already an SSH forced command. Re-cost the substrate phases
> against that before executing; the Wave 1-4 service seams and their triggers are unaffected.

> WARNING — **Do NOT start this band to "modernize the platform."** Per ADR-0014 §"Trigger conditions", Kubernetes is adopted only when one of these bites: (a) first paying tenants land contractually requiring SLA; (b) GPU inference needs a dedicated node pool; (c) an Enterprise tenant requires a cell; (d) a second residency region is sold. Until a trigger fires, Railway/Render is correct and K8s is deferred overhead.

---

## 2. Canonical names introduced in this band

This band **introduces no new service, table, topic, collection, Redis-key, or AGE-label names** — that is forbidden by the canon contract. Everything it stands up is a *placement* or *escalation* of already-frozen canon. The names it uses verbatim:

- **Services (20):** `api-gateway`, `web-app`, `bff`, `identity-tenant-service`, `core-api`, `meeting-service`, `realtime-gateway`, `transcription-service`, `ingestion-service`, `embedding-service`, `reranker-service`, `agent-runtime-service`, `rag-service`, `knowledge-graph-service`, `search-service`, `integration-service`, `workflow-service`, `billing-metering-service`, `notification-service`, `admin-ops-service`.
- **Scaling families (namespaces):** `prod-crud`, `prod-realtime`, `prod-ai`, `prod-async`, `prod-cell-{slug}`, `platform` ([Deployment §2.2](../16-deployment-and-infrastructure.md)).
- **Residency zones:** `us`, `eu`, `ap` ([Data Model `data_residency`], [ADR-0009](../adr/0009-data-residency-multi-region.md)).
- **Stores being escalated (no rename):** PostgreSQL (+ Apache AGE graph `nexus_knowledge`), Qdrant per-tenant collections `transcript_chunks__{org_id}` / `document_chunks__{org_id}` / `decisions__{org_id}`, Redis `nexus:{org_id}:...`, object buckets `nexus-recordings` / `nexus-documents` / `nexus-attachments` / `nexus-invoices` / `nexus-audit-archive` / `nexus-exports`.
- **Tenancy/escalation primitives (existing canon concepts, materialized here):** `data_residency.region` / `db_cell` / `qdrant_region` / `object_bucket_region` / `locked_at`; `app.current_org`; Citus distribution column `org_id`.
- **New *repo* (not a service name):** `nexus-deploy` (GitOps values repo, already named in [Deployment §6](../16-deployment-and-infrastructure.md)).
- **Capacity symbols cited (imported, never re-derived):** `A1`, `A4`, `A5`, `A6`, `A10`, `A18`, `E12`, `E13`, `L9`, `L10`, `EM2`, `EM4`, `V7`, `V9`, `T3`, `C3`, `C4`.

---

## 3. Work breakdown (ordered, grouped by phase then by service/component)

Ordering is a precondition chain. **Substrate before services** (you cannot peel a service onto a cluster that does not exist). **Wave 1 before Wave 2-4** because Wave 1 services *cannot run on Railway/Render at all* (no GPU pool, no KEDA-on-connections, no Temporal worker fleet) — the migration is forced, not optional ([ADR-0014 §Migration](../adr/0014-kubernetes-compute-platform.md)). **Cells and multi-region after the pooled estate is on K8s.** **Capacity escalations last and individually demand-driven.**

### Phase P14.1 — Kubernetes + Terraform IaC substrate (mytheus-platform)

#### P14.1.a — Repo scaffold

**Repo / folder:** `mytheus-platform/` (currently CLAUDE.md/README.md/system.md/.gitignore only).

**Steps:**

1. Create the directory skeleton:
   ```bash
   cd mytheus-platform
   mkdir -p terraform/modules/{network,cluster,data,edge,secrets,observability} \
            terraform/environments/prod/{us,eu,ap} \
            terraform/environments/{staging,dev} \
            helm/nexus/templates helm/nexus/charts \
            keda argo chaos runbooks loadtest scripts
   ```
2. Pin tool versions in `terraform/.tool-versions` (asdf) and `mytheus-platform/.mise.toml`:
   ```toml
   # .mise.toml
   [tools]
   terraform = "1.9.5"
   kubectl    = "1.30.3"
   helm       = "3.15.3"
   kustomize  = "5.4.2"
   argocd     = "2.12.3"
   ```
3. Add `terraform/versions.tf` (provider pins — Terraform's provider maturity is the locked reason over Pulumi/CDK, [ADR-0014 Alternatives]):
   ```hcl
   terraform {
     required_version = ">= 1.9.0"
     required_providers {
       aws        = { source = "hashicorp/aws",        version = "~> 5.60" }
       google     = { source = "hashicorp/google",     version = "~> 5.40" }
       kubernetes = { source = "hashicorp/kubernetes", version = "~> 2.31" }
       helm       = { source = "hashicorp/helm",       version = "~> 2.14" }
       cloudflare = { source = "cloudflare/cloudflare", version = "~> 4.40" }
     }
     backend "remote" {            # Terraform Cloud is the single control plane
       organization = "mytheus-nexus"
       workspaces { prefix = "nexus-" }   # nexus-prod-us-cluster, etc.
     }
   }
   ```

**Canonical names:** `mytheus-platform`, the four family namespaces, the `cluster`/`data`/`edge`/`secrets`/`observability` Terraform layers ([Deployment §4.1](../16-deployment-and-infrastructure.md)).

**Acceptance checks:**
- `terraform -chdir=terraform/environments/prod/us init` succeeds against Terraform Cloud.
- `mise install` resolves all pinned tools.

#### P14.1.b — Terraform layers (state split per `(account, region, layer)`)

**Repo / folder:** `mytheus-platform/terraform/`.

The blast radius is bounded by splitting state per `(account, region, layer)` — a botched `prod/eu/data` apply cannot touch `prod/us/cluster` ([Deployment §4.2](../16-deployment-and-infrastructure.md)).

**Steps:**

1. `terraform/modules/cluster/variables.tf` — the four family node pools + per-cell tainted pools. Node-pool shapes are engineering specifics derived from the anchors, NOT new capacity numbers ([Deployment §4.2](../16-deployment-and-infrastructure.md)):
   ```hcl
   variable "node_pools" {
     type = map(object({ shape = string, min = number, max = number, spot = bool }))
     default = {
       crud     = { shape = "m6i.xlarge",  min = 3, max = 40, spot = true  }
       realtime = { shape = "c6in.xlarge", min = 6, max = 60, spot = false } # long-lived sockets: NEVER spot
       ai_cpu   = { shape = "c6i.2xlarge", min = 4, max = 80, spot = false }
       ai_gpu   = { shape = "g6.xlarge",   min = 2, max = 24, spot = false } # L4 GPU; EM2/EM4 floor
       async    = { shape = "m6i.large",   min = 2, max = 20, spot = true  }
     }
   }
   variable "enterprise_cells" {            # taint+affinity node pools per cell tenant
     type = map(object({ slug = string, shape = string, min = number, max = number }))
     default = {}
   }
   variable "region" { type = string }      # us-east-1 | eu-west-1 | ap-southeast-1 ...
   ```
   The `realtime` pool is `spot = false` deliberately: spot reclamation would sever live WebSocket / Deepgram-proxy connections held for `A9 ~= 45 min`, which the live-caption < 2s budget cannot tolerate ([Deployment §4.2, §7.2](../16-deployment-and-infrastructure.md)).
2. `terraform/modules/cluster/main.tf` — EKS or GKE per the cloud the region lives in (workload Helm is cloud-agnostic; only the `cluster` layer diverges per cloud, [ADR-0014 §"What we accept"]). For EKS, create the control plane, the five managed node groups, and IRSA bindings:
   ```hcl
   module "eks" {
     source  = "terraform-aws-modules/eks/aws"
     version = "~> 20.0"
     cluster_name    = "nexus-${var.environment}-${var.region}"
     cluster_version = "1.30"
     eks_managed_node_groups = {
       for k, p in var.node_pools : k => {
         instance_types = [p.shape]
         min_size = p.min, max_size = p.max, desired_size = p.min
         capacity_type = p.spot ? "SPOT" : "ON_DEMAND"
         labels = { family = k }
         taints = k == "realtime" ? [] : []     # cells get taints below
       }
     }
   }
   # Per-cell tainted pools (taint: cell={slug}) for Enterprise — Multi-Tenancy §3.6
   resource "aws_eks_node_group" "cell" {
     for_each = var.enterprise_cells
     node_group_name = "cell-${each.value.slug}"
     # ... taint { key = "cell", value = each.value.slug, effect = "NO_SCHEDULE" }
   }
   ```
3. `terraform/modules/data/main.tf` — managed Postgres (primary + read replica + PITR), Qdrant Cloud cluster(s), Redis (Upstash), **Redpanda BYOC** cluster, Temporal Cloud namespace. Sized at the anchors: Redpanda ~`E12`=17,900 ev/s peak, ~`E13`=13.9 TB/mo; Qdrant `V7`=4.42B vectors / ~`V9`=17 TB ([Deployment §4.1](../16-deployment-and-infrastructure.md), [Capacity §6, §5](../_canon/05-capacity-math.md)).
4. `terraform/modules/edge/main.tf` — Cloudflare CDN + WAF, DNS, custom domain, **health-checked failover policy** (the multi-region swing in §P14.6).
5. `terraform/modules/secrets/main.tf` — Cloud KMS keys including **per-tenant KMS data keys** for `oauth_tokens` envelope encryption (referenced by `oauth_tokens.token_version`, consumed only by `integration-service`), Secrets Manager entries, IAM roles for External Secrets Operator ([Deployment §9.2](../16-deployment-and-infrastructure.md)).
6. `terraform/environments/prod/us/main.tf` — instantiate all modules for `us`; `eu`/`ap` are the *same* modules with `region` changed, not copy-paste ([Deployment §4.2](../16-deployment-and-infrastructure.md)).
7. Wire CI: `plan` on every PR touching `terraform/`, `apply` review-gated from Terraform Cloud (never a laptop); `tfsec`/`checkov` gate IaC ([Deployment §5.2](../16-deployment-and-infrastructure.md)).

**Canonical names:** `cluster`/`data`/`edge`/`secrets`/`observability` layers; node pools `crud`/`realtime`/`ai_cpu`/`ai_gpu`/`async`; per-tenant KMS data keys; Redpanda BYOC.

**Acceptance checks:**
- `terraform -chdir=terraform/environments/prod/us plan` shows the 5 node pools, Postgres primary+replica+PITR, Qdrant, Redis, Redpanda BYOC, Temporal namespace, Cloudflare edge.
- `tfsec terraform/` and `checkov -d terraform/` return zero high-severity findings (CI gate).
- After `apply`: `kubectl get nodes -L family` shows nodes labeled `crud|realtime|ai_cpu|ai_gpu|async`.

#### P14.1.c — Platform operators (the `platform` namespace)

**Repo / folder:** `mytheus-platform/helm/nexus/charts/` + `argo/`.

**Steps:**

1. Install cluster-wide operators into the `platform` namespace ([Deployment §2.2](../16-deployment-and-infrastructure.md)): External Secrets Operator (ESO), **KEDA**, ingress controller (NGINX or cloud LB controller), cert-manager, Prometheus + Grafana, optional mesh control plane (Linkerd preferred; **optional, not locked** — [Deployment §8.2]).
   ```bash
   helm repo add kedacore https://kedacore.github.io/charts
   helm repo add external-secrets https://charts.external-secrets.io
   helm install keda kedacore/keda -n platform --create-namespace
   helm install eso external-secrets/external-secrets -n platform
   ```
2. ESO `ClusterSecretStore` reconciling from AWS Secrets Manager / GCP Secret Manager with Cloud KMS as the encryption root. **No secret in git, image, or plaintext manifest** ([Deployment §9.1](../16-deployment-and-infrastructure.md)); enforced by the `gitleaks` gate from day one.
3. `NetworkPolicy` default-deny per namespace: only `api-gateway` may ingress `prod-crud`; only the bus + authorized internal callers reach `prod-ai`/`prod-realtime` ([Deployment §8.1](../16-deployment-and-infrastructure.md)).

**Canonical names:** `platform` namespace, External Secrets Operator, KEDA, per-tenant KMS data keys.

**Acceptance checks:**
- `kubectl get pods -n platform` shows keda, eso, ingress, cert-manager, prometheus, grafana Running.
- An `ExternalSecret` for the Deepgram key syncs a K8s Secret consumed *only* by `transcription-service` (the server-side-key rule).
- A `NetworkPolicy` test: a pod in `prod-ai` cannot reach `prod-crud` except via the bus / authorized path.

#### P14.1.d — Helm umbrella chart + GitOps (`nexus-deploy`)

**Repo / folder:** `mytheus-platform/helm/nexus/` (chart) and the new `nexus-deploy` repo (rendered values).

**Steps:**

1. One **umbrella chart** with a subchart template per service and per-env `values-{env}.yaml`. A service differs across environments only by values (replica floors, resources, autoscaler thresholds, image digest) — **never by template** ([Deployment §3.2](../16-deployment-and-infrastructure.md)). Standard pod labels on every deployment: `app`, `family` (`crud`/`realtime`/`ai`/`async`), `version` (git SHA), `tenant-cell` (or `pooled`).
   ```yaml
   # helm/nexus/values-prod.yaml  (excerpt — replica floors are engineering specifics, Deployment §2.3)
   api-gateway:           { family: crud,     minReplicas: 3 }   # per region, HA across AZs
   realtime-gateway:      { family: realtime, minReplicas: 6 }   # HA floor; KEDA scales toward A6
   transcription-service: { family: realtime, minReplicas: 6 }   # KEDA scales toward A5 (1 Deepgram WS/meeting)
   embedding-service:     { family: ai,       minReplicas: 2, gpu: true }  # EM2/EM4 floor
   reranker-service:      { family: ai,       minReplicas: 2, gpu: true }  # <500ms cold budget
   agent-runtime-service: { family: ai,       minReplicas: 4 }   # token-scaled toward L9
   core-api:              { family: crud,     minReplicas: 3 }   # pooled + RLS CRUD heart
   workflow-service:      { family: async,    minReplicas: 2 }   # Temporal worker fleet
   # all others floor at 2 replicas across AZs for HA
   ```
2. Create the `nexus-deploy` GitOps repo holding rendered values per `(env, region)`. The CI "deploy" step is a **commit to `nexus-deploy`** bumping `image.digest`, NOT a `kubectl apply` ([Deployment §6](../16-deployment-and-infrastructure.md)).
3. Install **Argo CD** + **Argo Rollouts** in `platform`; one `Application` per `(env, region)` reconciling against `nexus-deploy`. Rollback is a `git revert` of the digest bump.
4. **Image promotion by digest:** the image built once at merge (tagged git SHA, pinned by digest) is the *same* image promoted staging -> prod; promotion changes only the Helm `image.digest` value ([Deployment §5.3](../16-deployment-and-infrastructure.md)).

**Canonical names:** `nexus-deploy`, umbrella chart `nexus`, labels `app`/`family`/`version`/`tenant-cell`, Argo CD, Argo Rollouts.

**Acceptance checks:**
- `helm template helm/nexus -f values-prod.yaml` renders all 20 services with correct `family` labels.
- An Argo CD `Application` shows `Synced/Healthy` against `nexus-deploy` for `prod/us`.
- A `git revert` of an `image.digest` bump in `nexus-deploy` reconciles the cluster back within one Argo sync interval.

### Phase P14.2 — Wave 1 extraction (different runtime from birth)

Wave 1 = `embedding-service` + `reranker-service` (GPU/Ray Serve), `agent-runtime-service` (token-scaled), `transcription-service` + `realtime-gateway` (connection-scaled), `workflow-service` (Temporal). **This wave goes first because Railway/Render cannot host it at all** ([Service Catalog §7](../_canon/01-service-catalog.md), [ADR-0014 §Migration]).

**Strangler-fig mechanics (every wave):** (1) **cut over the bus first** — point the extracted service at the same Redpanda topics; the dual-running window is safe because at-least-once + idempotency dedupes on `idempotency_key`; (2) **repoint sync calls** from in-process to gRPC/REST via the stable path prefix; (3) **run both, drain the monolith's copy, then delete the module** ([Deployment §11.1](../16-deployment-and-infrastructure.md)).

#### P14.2.a — embedding-service + reranker-service (GPU / Ray Serve / vLLM)

**Repo / folder:** `mytheus-intelligence/services/embedding-service/`, `mytheus-intelligence/services/reranker-service/`; chart values in `mytheus-platform/helm/nexus/values-prod.yaml`; KEDA in `mytheus-platform/keda/`.

**Steps:**

1. Build the GPU image: CUDA runtime base + Ray Serve + ONNX/vLLM runtime with **BGE-small-en (384-dim)** / **BGE-reranker-base** weights **baked in at build time** so cold start is a *load*, not a *fetch* ([Deployment §3.1](../16-deployment-and-infrastructure.md)).
   ```dockerfile
   # mytheus-intelligence/services/embedding-service/Dockerfile.gpu
   FROM nvidia/cuda:12.4.1-runtime-ubuntu22.04
   # ... pip install ray[serve] vllm onnxruntime-gpu
   COPY weights/bge-small-en /opt/models/bge-small-en   # baked, not fetched
   ENTRYPOINT ["tini","--","ray","start","--head","--block"]
   ```
2. KEDA `ScaledObject` for `embedding-service` — Redpanda consumer lag on `transcript-chunks` + `document-chunks`, plus GPU util; scale-to-zero permitted off-peak (cold start absorbed by the lag buffer); sized to `EM2` ~525 embeds/sec peak ([Scalability §3.2](../13-scalability-and-performance.md)):
   ```yaml
   # mytheus-platform/keda/embedding-service.yaml
   apiVersion: keda.sh/v1alpha1
   kind: ScaledObject
   metadata: { name: embedding-service, namespace: prod-ai }
   spec:
     scaleTargetRef: { name: embedding-service }
     minReplicaCount: 0
     maxReplicaCount: 40
     triggers:
       - type: kafka          # Redpanda is Kafka-API compatible
         metadata: { bootstrapServers: redpanda.internal:9092, consumerGroup: embedding-consumer,
                     topic: transcript-chunks, lagThreshold: "2000" }
       - type: prometheus
         metadata: { query: "avg(gpu_utilization{svc='embedding-service'})", threshold: "70" }
   ```
3. `reranker-service` KEDA on rerank req/s + GPU util with **>= 1 warm replica per active region** (cross-encoder cold start is the latency risk; warm floor keeps RAG p95 on the warm side, < 500 ms) ([Scalability §3.2, §6.3](../13-scalability-and-performance.md)).
4. Rollout strategy = **canary, not blue-green** (GPUs are expensive/capacity-constrained; doubling the fleet is uneconomical): shift 5% -> 25% -> 50% -> 100% via Argo Rollouts; auto-abort on SLO regression (embedding batch < 100 ms, RAG p95 warm < 500 ms) ([Deployment §7.1, §7.3](../16-deployment-and-infrastructure.md)).
5. Cut over the bus (embedding consumer points at the same `transcript-chunks`/`document-chunks` topics and the same per-tenant Qdrant collections `transcript_chunks__{org_id}` etc.), repoint `rag-service -> embedding-service` / `rag-service -> reranker-service` from in-process to gRPC, drain, delete the monolith modules.

**Canonical names:** `embedding-service`, `reranker-service`, BGE-small-en 384-dim, BGE-reranker-base, Ray Serve/vLLM, `prod-ai`, Qdrant collections `transcript_chunks__{org_id}` / `document_chunks__{org_id}` / `decisions__{org_id}`, `EM2`/`EM4`.

**Acceptance checks:**
- `kubectl get scaledobject -n prod-ai` shows `embedding-service` scaling 0->N on injected lag; pod lands on a `family=ai_gpu` node.
- Embedding batch (32) p95 < 100 ms; reranker warm p95 < 500 ms under load (LT-4 gate).
- Argo Rollouts canary auto-aborts and reverts `image.digest` when a synthetic SLO regression is injected.
- Monolith embedding/reranker modules deleted; no duplicate Qdrant upserts (idempotency holds).

#### P14.2.b — agent-runtime-service (token-scaled)

**Repo / folder:** `mytheus-intelligence/services/agent-runtime-service/`.

**Steps:**

1. KEDA on **token throughput + `transcript-chunks` window backlog**, min 4 replicas/region, sized toward `L9` ~8,930 LLM calls/sec peak ([Scalability §3.2](../13-scalability-and-performance.md), [Deployment §2.3](../16-deployment-and-infrastructure.md)). It hosts the **six LangGraph agents** (`PersonalListenerAgent`, `DecisionAgent`, `ActionAgent`, `QuestionAgent`, `ContextGapAgent`, `ProactiveAgent`) on the shared `BaseAgent` lifecycle.
2. Preserve the cost-controlled pipeline unchanged: relevance pre-gate (Haiku) -> windowed batching (`A17`=30s, per 30-second WINDOW not per raw chunk) -> tiered routing (Haiku -> Sonnet -> Opus) -> semantic + Anthropic prompt caching -> per-tenant token budgets (`agent_budgets`, hot mirror `nexus:{org_id}:budget:{period}`, deny-by-default at cap). Confidence: >0.7 surface, 0.4-0.7 silent-log to `worklogs`, <0.4 drop. **No silent commits** — agents emit candidates; `core-api` writes confirmed decisions/actions on confirmation and is the SOLE producer of the `decisions`/`actions` events.
3. Cut over the bus (`transcript-chunks`, `document-chunks`, `agent-requests`, `agent-responses` consume; `agent-results`, `agent-stream`, `decisions`(candidates via core-api), `actions`, `notifications`, `usage-events` produce); repoint `agent-runtime-service -> identity-tenant-service /context` and `-> rag-service` to gRPC.

**Canonical names:** `agent-runtime-service`, the six agent classes, `agent_budgets`, `nexus:{org_id}:budget:{period}`, `L9`, topics `agent-results`/`agent-stream`/`agent-requests`/`agent-responses`/`usage-events`.

**Acceptance checks:**
- KEDA scales `agent-runtime-service` on injected token throughput; per-tenant hard cap denies at `hard_cap` (100% enforcement SLO).
- Tier mix under LT-3 holds Haiku 45% / Sonnet 55% / Opus <1%; no DLQ growth.
- Agent end-to-end p50 < 2s / p95 < 4s under LT-1.

#### P14.2.c — transcription-service + realtime-gateway (connection-scaled)

**Repo / folder:** `mytheus-realtime/services/transcription-service/`, `mytheus-realtime/services/realtime-gateway/`.

**Steps:**

1. `realtime-gateway` KEDA on **concurrent WebSocket connections**, min 6, max 200 (~600 conns/pod -> ~167 pods at `A6`=100k peak), Redis pub/sub backbone `nexus:{org_id}:pubsub:meeting:{meeting_id}` for cross-pod fan-out ([Scalability §3.3](../13-scalability-and-performance.md)):
   ```yaml
   # mytheus-platform/keda/realtime-gateway.yaml
   apiVersion: keda.sh/v1alpha1
   kind: ScaledObject
   metadata: { name: realtime-gateway, namespace: prod-realtime }
   spec:
     scaleTargetRef: { name: realtime-gateway }
     minReplicaCount: 6
     maxReplicaCount: 200
     triggers:
       - type: prometheus
         metadata: { query: "sum(socketio_active_connections)/count(up{svc='realtime-gateway'})", threshold: "600" }
       - type: cpu
         metadata: { type: Utilization, value: "60" }
   ```
2. `transcription-service` KEDA on **concurrent live streams** (one diarized Deepgram WS session per *meeting*, anchored on `A5`=20k — the 6x cost lever, `D3` avoided), min 6.
3. Rollout strategy = **connection-drain / graceful shed**: old pods marked `NotReady` for new connections but keep serving existing meetings; `terminationGracePeriodSeconds` above the worst-case meeting tail; `preStop` hook waits for active sessions to reach zero; sticky Redis-backed routing keeps a participant on the pod that holds their socket. This is why the realtime pool is **on-demand, never spot** ([Deployment §7.2](../16-deployment-and-infrastructure.md)).
4. The passive region does **not** keep the real-time tier warm: a live meeting cannot be transparently migrated; clients reconnect to the surviving in-zone region and the LiveKit room re-establishes there ([Reliability §3.1](../14-reliability-and-dr.md)).

**Canonical names:** `transcription-service`, `realtime-gateway`, `prod-realtime`, `nexus:{org_id}:pubsub:meeting:{meeting_id}`, `A5`/`A6`, topics `agent-stream`/`agent-results`/`notifications`/`presence-events`.

**Acceptance checks:**
- LT-2: 100k WS connections, fan-out hop < 50 ms, zero dropped connections, pod count <= 200.
- A rollout during a live meeting drains old pods to zero active sessions before terminating (no caption drop).
- Deepgram WS proxy reconnect > 99.9% within 2s under a Chaos Mesh network-partition experiment.

#### P14.2.d — workflow-service (Temporal)

**Repo / folder:** `mytheus-backend/services/workflow-service/`.

**Steps:**

1. Deploy the Temporal worker fleet (min 2) into `prod-async`; KEDA on **Temporal task-queue depth**. Registers `MeetingLifecycleWorkflow`, `ActionFollowUpWorkflow` (reminders day 1/3/7, manager Slack-DM escalation day 10), the cross-store right-to-be-forgotten saga, and dunning sagas — sized to ~1.34B state transitions/mo.
2. Rollout strategy = **Temporal Worker Versioning**: running workflows complete on the build that started them (new build ID for new executions); in-flight `MeetingLifecycleWorkflow` / `ActionFollowUpWorkflow` pin to their build ([Deployment §7.1](../16-deployment-and-infrastructure.md)).

**Canonical names:** `workflow-service`, `prod-async`, `MeetingLifecycleWorkflow`, `ActionFollowUpWorkflow`, Temporal Worker Versioning.

**Acceptance checks:**
- KEDA scales workers on injected task-queue depth.
- A rollout mid-workflow: an in-flight `ActionFollowUpWorkflow` completes on its original build ID while new executions go to the new build.

#### P14.2.e — Database migrations are expand/contract (applies to ALL waves)

**Repo / folder:** `mytheus-backend/` (Alembic) — a standing rule, not a one-off.

Blue-green and canary both run two app versions against one database simultaneously. That is only safe if **Alembic** migrations follow **expand -> deploy -> contract**: (1) **expand** — additive only (new nullable column, new table, `CREATE INDEX CONCURRENTLY`); (2) **deploy** — roll the new image, both versions coexist; (3) **contract** — once 100% on the new version, drop the now-unused column/constraint ([Deployment §7.4](../16-deployment-and-infrastructure.md)). This mirrors the Avro BACKWARD compatibility already locked for events.

**Acceptance check:** a CI lint rejects any migration that drops/renames a column in the same revision that the new image needs; every destructive migration must be a separate contract revision gated on 100% rollout.

### Phase P14.3 — Wave 2 extraction (CRUD core: core-api then identity-tenant-service)

**Trigger:** CRUD req/s pressure — the relational heart and control plane outgrow the shared deployable ([Service Catalog §7](../_canon/01-service-catalog.md)).

**Repo / folder:** `mytheus-backend/services/core-api/`, `mytheus-backend/services/identity-tenant-service/`.

**Steps:**

1. Extract `core-api` **from `bff` first**, then `identity-tenant-service` (order matters: `core-api` is the relational heart; `identity-tenant-service` is the control-plane chokepoint that resolves `org_id` -> posture + cell). Both deploy to `prod-crud`.
2. HPA on req/s + CPU + a Prometheus-adapter p95 custom metric so the fleet scales out *before* the 300 ms SLO is breached ([Scalability §3.1](../13-scalability-and-performance.md)):
   ```yaml
   # mytheus-platform/helm/nexus/templates/hpa-core-api.yaml
   apiVersion: autoscaling/v2
   kind: HorizontalPodAutoscaler
   metadata: { name: core-api, namespace: prod-crud }
   spec:
     scaleTargetRef: { apiVersion: apps/v1, kind: Deployment, name: core-api }
     minReplicas: 4
     maxReplicas: 120
     metrics:
       - type: Resource
         resource: { name: cpu, target: { type: Utilization, averageUtilization: 65 } }
       - type: Pods
         pods: { metric: { name: http_requests_per_second }, target: { type: AverageValue, averageValue: "150" } }
       - type: Pods
         pods: { metric: { name: http_request_p95_seconds }, target: { type: AverageValue, averageValue: "0.25" } }
     behavior:
       scaleUp:   { stabilizationWindowSeconds: 30,  policies: [{ type: Percent, value: 100, periodSeconds: 30 }] }
       scaleDown: { stabilizationWindowSeconds: 300, policies: [{ type: Percent, value: 25,  periodSeconds: 60 }] }
   ```
3. The hard ceiling on this tier is **Postgres connections**, not pod count: every CRUD pod fronts the primary through **PgBouncer transaction pooling** (`default_pool_size` 25 per (db,user), `max_client_conn` 10,000) so the HPA fans out without exhausting `max_connections`. Read-heavy paths (dashboard reads, FTS, the `rag-service` FTS arm) route to the async read replica; only writes + read-your-write (token mint, decision confirm) touch the primary ([Scalability §3.1](../13-scalability-and-performance.md)).
4. Rollout strategy = **blue-green or rolling** (stateless, cheap to double briefly; instant cutover/rollback).
5. Preserve `core-api` as the **sole producer** of confirmed `decisions`/`actions` events (no silent commits); `identity-tenant-service` keeps the Clerk Svix webhook user-sync (outbox + reconciliation) and `get_user_context`.

**Canonical names:** `core-api`, `identity-tenant-service`, `prod-crud`, `app.current_org`, PgBouncer, `bff`, topics `decisions`/`actions`/`meeting-events`/`tenant-events`/`user-sync-events`.

**Acceptance checks:**
- LT-6 CRUD: p95 < 300 ms, p99 < 800 ms under DAU browse load; HPA scales on the p95 custom metric before SLO breach.
- PgBouncer holds connections flat while pods fan out to maxReplicas.
- Clerk webhook user-sync still lands in `public.users` via the outbox after extraction.

### Phase P14.4 — Wave 3 extraction (SaaS business)

**Trigger:** SaaS-business load — metering ingest, webhook bursts; bursty, monetization-critical; extract once paying tenants land ([Service Catalog §7](../_canon/01-service-catalog.md)).

**Services:** `billing-metering-service`, `integration-service`, `admin-ops-service` (all `prod-crud`).

**Repo / folder:** `mytheus-backend/services/{billing-metering-service,integration-service,admin-ops-service}/`.

**Steps:**

1. Extract `billing-metering-service` — Stripe billing/subscriptions/seats/dunning/tax + the `usage-events -> usage_aggregates -> agent_budgets` metering pipeline. The `usage-events` consumer dedupes on `idempotency_key` (+ `nexus:{org_id}:dedupe:usage-events:{idempotency_key}`) — billing-correctness, not just delivery nicety.
2. Extract `integration-service` — Calendar/Slack/Jira-or-Linear OAuth. **Off the meeting critical path** (R8 reliability stance): outbound syncs queue in the outbox and retry with backoff; a provider outage never degrades a live meeting. `integration-service` is the **only** decryptor of per-tenant KMS data keys for `oauth_tokens` ([Deployment §9.2](../16-deployment-and-infrastructure.md), [Reliability §5.4](../14-reliability-and-dr.md)).
3. Extract `admin-ops-service` — tenant-admin + internal-ops consoles, feature flags, consumer-lag & per-tenant cost dashboards, audit-log access. Strict RBAC + full audit.
4. Rollout strategy = blue-green/rolling.

**Canonical names:** `billing-metering-service`, `integration-service`, `admin-ops-service`, `usage-events`, `agent_budgets`, `oauth_tokens`, `billing-events`/`integration-events`/`admin-events`, per-tenant KMS data keys.

**Acceptance checks:**
- LT-3 budget enforcement 100%; usage double-counting rate ~0 (idempotent metering).
- A simulated Slack/Jira outage: outbound syncs queue + retry; no live-meeting SLO regression.
- `admin-ops-service` cost board renders per-tenant COGS from `usage_aggregates`.

### Phase P14.5 — Wave 4 extraction (retrieval stores)

**Trigger:** when AGE/FTS each warrant dedicated ops. **Extract last** because they start as libraries inside `rag-service` and share read models with `core-api` ([Service Catalog §7](../_canon/01-service-catalog.md)).

**Services:** `knowledge-graph-service`, `search-service` (split out of `rag-service`; both `prod-ai`).

**Repo / folder:** `mytheus-intelligence/services/{knowledge-graph-service,search-service}/`.

**Steps:**

1. Split `knowledge-graph-service` out of `rag-service`: it owns the Apache AGE graph `nexus_knowledge` (nodes Person/Meeting/Decision/Action/Topic; edges ATTENDED/PRODUCED/ASSIGNED_TO/RELATES_TO/FOLLOWS_FROM; **edges on confirmation only**), org-partitioned with `$org_id` on every named Cypher template. gRPC `GraphQuery(template, $org_id)` from `rag-service`. Add **read replicas** for Cypher read load.
2. Split `search-service` out: Postgres FTS over `transcript_chunks` + `chat_messages`, the FTS arm of hybrid retrieval. gRPC `Search` from `rag-service`.
3. Preserve RRF fusion in `rag-service` (k=60 over Qdrant vector + Postgres FTS + AGE topic-relations arms) and the hot-degradation path: if `knowledge-graph-service` is slow/down, RRF **drops the AGE arm** and continues on Qdrant + FTS; `ProactiveAgent` falls back from AGE `FOLLOWS_FROM` lineage to the `decisions__{org_id}` Qdrant semantic path (R1 hot degradation, [Reliability §5.3](../14-reliability-and-dr.md)).

**Canonical names:** `knowledge-graph-service`, `search-service`, `rag-service`, `nexus_knowledge`, the 5 node + 5 edge labels, `$org_id`, `decisions__{org_id}`, RRF k=60, `graph-events`.

**Acceptance checks:**
- `knowledge-graph-service` kill (Chaos Mesh pod-kill): RAG p95 stays < 1.5s on Qdrant+FTS; `ProactiveAgent` uses the Qdrant decisions path.
- Named Cypher templates only in prod (the `/graph/cypher` surface is internal-only and gated); every template carries `$org_id` on every matched node.

### Phase P14.6 — Cells + multi-region (active-passive -> active-active, in-zone only)

#### P14.6.a — Cell-based isolation for Enterprise (RLS stays ON inside a cell)

**Repo / folder:** `mytheus-platform/terraform/` (per-cell node pools + dedicated data) + `nexus-deploy` (`prod-cell-{slug}` values).

**Steps:**

1. Provision a cell as a **dedicated node pool + namespace `prod-cell-{slug}` with taints/affinity** (Terraform `enterprise_cells` var from P14.1.b), plus the cell's own Postgres instance/cluster, dedicated Qdrant cluster, and (for the largest) dedicated Redpanda capacity + inference node pools ([Multi-Tenancy §2, §3.6](../06-multi-tenancy.md), [ADR-0014 §D2]).
2. **RLS stays FORCE on inside the cell** — a cell reduces blast radius and removes noisy-neighbor, but never replaces the row-level backstop ([Multi-Tenancy §2](../06-multi-tenancy.md)).
3. Record cell membership in `data_residency.db_cell` (e.g. `cell-eu-3`); the same row pins `region`, `qdrant_region`, `object_bucket_region`, immutable after `locked_at`. `identity-tenant-service` resolves `org_id` -> posture + cell and routes Enterprise traffic to `prod-cell-{slug}`.

**Canonical names:** `prod-cell-{slug}`, `data_residency.db_cell` / `region` / `qdrant_region` / `object_bucket_region` / `locked_at`, RLS FORCE, `app.current_org`.

**Acceptance checks:**
- An Enterprise tenant's pods schedule only onto the tainted `cell-{slug}` pool; its Postgres/Qdrant are dedicated.
- RLS `FORCE` confirmed on every tenant-scoped table inside the cell DB; a missing `app.current_org` returns zero rows (fail-closed).
- A noisy-neighbor LT-9 run against the pooled estate does not touch the cell tenant's SLOs.

#### P14.6.b — Multi-region active-passive, then active-active candidate (in-zone only)

**Repo / folder:** `mytheus-platform/terraform/environments/prod/{us,eu,ap}/` + `terraform/modules/edge/` + `nexus-deploy`.

**Steps:**

1. Stand up the passive physical region **inside the tenant's zone only** (the in-zone failover rule — cross-zone replication is FORBIDDEN at every layer): `eu` = `eu-west-1` primary + `eu-central-1` passive; `us` = `us-east-1` + `us-west-2`; `ap` = `ap-southeast-1` + `ap-southeast-2` ([ADR-0009 §D3](../adr/0009-data-residency-multi-region.md)). A new region is the same `cluster`+`data`+`edge` modules with `region` changed.
2. Passive region runs CRUD + AI tiers **scaled to minimum (warm, not cold)**; holds an async Postgres streaming replica (lag < 5s = the Pro/Enterprise RPO), a Redpanda **MirrorMaker2** mirror targeting the in-zone passive only, and cross-region-replicated object buckets. The real-time tier is deliberately **not** kept warm ([Reliability §3.1, §4](../14-reliability-and-dr.md)).
3. Cloudflare health-checked DNS failover: probe `/healthz` every 10s, swing on 3 consecutive failures (30s); `identity-tenant-service`/control plane promotes the replica (`pg_ctl promote`), scales the passive CRUD+AI tiers to serving capacity, replays the Redpanda mirror from last committed offset ([Reliability §3.2](../14-reliability-and-dr.md)).
4. **Per-plan availability tiers** map onto posture:
   | Plan | Availability | Downtime/mo | Posture | RPO | RTO |
   |---|---|---|---|---|---|
   | Free | 99.5% | ~216 min | single-region, pooled + RLS | <= 5 min (in-region PITR) | <= 2h |
   | Pro | 99.9% | ~43.2 min | multi-region active-passive (in-zone) | <= 5s (async replica lag) | <= 30m |
   | Enterprise | 99.95% | ~21.6 min | cell-based active-active candidate | ~0 (active-active) / <= 5s (single-cell) | ~0 / <= 15m |
   Each RTO must leave headroom inside the monthly budget for one unplanned event ([Reliability §2, §3.3](../14-reliability-and-dr.md)).
5. **Active-active** is the Enterprise-cell evolution only: a cell already owns dedicated node pools + pinned data, so it serves both in-zone regions concurrently with conflict-free writes scoped to `org_id` ([Reliability §3.2](../14-reliability-and-dr.md), [ADR-0009 §D3]).
6. `data_residency` cells: subprocessor/inference regionality follows the zone — NEXUS-operated `embedding-service`/`reranker-service` run **in-zone unconditionally**; LiteLLM/Deepgram/LiveKit route to an in-zone endpoint where one exists, else the SCC + zero-retention backstop applies ([ADR-0009 §D4]).

**Canonical names:** zones `us`/`eu`/`ap`, in-zone failover rule, MirrorMaker2, `data_residency`, PITR, Cloudflare health-checked failover, active-passive -> active-active.

**Acceptance checks:**
- Region-kill GameDay (AWS FIS): Pro RTO <= 30m, <= 5s data loss, new meetings resume in the in-zone passive region.
- A CI invariant + tenant-router startup assertion reject any `data_residency` row whose `db_cell`/`qdrant_region`/`object_bucket_region` disagrees with `region`.
- No cross-zone replication exists at any layer (Postgres replica, MirrorMaker2 mirror, object replication all in-zone).

### Phase P14.7 — Capacity escalation paths (each demand-driven, independent)

Each path fires only on its own saturation trigger; none is a default.

#### P14.7.a — Postgres: cells -> Citus shard-by-`org_id`

**Trigger:** single-primary Postgres write ceiling at `T3`=1.728B `transcript_chunks` rows/mo + all CRUD writes, after pooling + read replicas + monthly range partitions + cells are exhausted ([Scalability §3.5, §6.2](../13-scalability-and-performance.md)).

**Steps:** escalation order is **vertical + read replicas -> monthly range partitions -> cells -> Citus sharding by `org_id`**. Because every tenant-scoped row already carries `org_id` as the natural distribution column, the shard is *mechanical, not a schema rewrite*: `SELECT create_distributed_table('transcript_chunks', 'org_id');` per tenant-scoped table; the graph does **not** distribute under Citus and follows its own Neptune/Aura path ([Scalability §3.5](../13-scalability-and-performance.md), [Data Architecture §3.7]).

**Acceptance checks:** a shard-cutover rehearsal on a staging clone distributes `transcript_chunks`/`chat_messages`/`meetings`/`usage_events` by `org_id` with RLS still FORCE; read-your-write correctness holds post-shard.

#### P14.7.b — Graph: AGE -> Neptune / Neo4j Aura exit (+ relational MV floor)

**Trigger:** Apache AGE proves structurally awkward on managed Postgres at scale (R1 cold operational fallback) — a **planned cutover over hours with dual-write + verification, NOT a live failover** ([Reliability §5.3](../14-reliability-and-dr.md), [Multi-Tenancy §3.3]).

**Steps:** migrate `nexus_knowledge` to Amazon Neptune / Neo4j Aura with a relational materialized-view fallback as the floor; every Cypher template stays parameterized by `$org_id` so the partition contract is preserved. AGE is **rebuildable from Postgres** (replay confirmed `decisions`/`action_items` rows reconstructs every edge), so the cutover is verifiable by edge-count parity ([Reliability §4.1](../14-reliability-and-dr.md)).

**Acceptance checks:** dual-write window shows edge-count parity between AGE and target; `$org_id` partitioning verified on the target; RAG AGE-arm latency within budget post-cutover.

#### P14.7.c — Redpanda BYOC/dedicated multi-AZ

**Trigger:** already sized at the anchors — `E12` ~17,900 ev/s peak, `E13` ~13.9 TB/mo. Production runs a **BYOC/dedicated multi-AZ** cluster (not the Serverless trial), tenant-partitioned, high-volume topics keyed `{org_id}:{meeting_id}` across 120-180 partitions ([Scalability §3.6](../13-scalability-and-performance.md), [Capacity §6](../_canon/05-capacity-math.md)).

**Acceptance checks:** LT-7 24h soak at `E12`: consumer lag < 5s, DLQ rate < 0.1%, duplicate rate ~0; AZ-kill (FIS) absorbs with zero data loss.

#### P14.7.d — int8 Qdrant (4.42B vectors / ~17 TB)

**Trigger:** `V7`=4.42B live vectors / ~`V9`=17 TB indexed at `A18`=13-month retention. Per-tenant collections + **scalar int8 quantization** (~4x resident-RAM cut) + hot ~90d in RAM / cold <=13mo memmap ([Scalability §3.5, §6.4](../13-scalability-and-performance.md), [Capacity §5](../_canon/05-capacity-math.md)). A tenant's entire footprint drops by deleting its `__{org_id}` collections (RTBF fast path).

**Acceptance checks:** Qdrant top-50 vector search p95 < 80 ms int8-quantized; per-tenant collection delete removes a tenant's full vector stock.

#### P14.7.e — GPU autoscaling

**Trigger:** GPU util / queue depth on `embedding-service` + `reranker-service`. Ray Serve autoscale + KEDA GPU-util triggers (P14.2.a). GPU pool is on-demand, lumpy, bin-packed; idle cost between surges is accepted and tuned by FinOps ([ADR-0014 §Consequences], [Cost §8.3]).

**Acceptance checks:** LT-8 embedding-ingest backpressure (burst 4x then drain): batch < 100 ms, lag recovers < 30s, `embedding-service` scales on lag; LT-4 GPU rerank warm/cold split holds.

#### P14.7.f — The C4 ~30% LLM cost reduction realized

**Trigger:** standing FinOps loop, not a one-time pass. LLM is `C3`=94% of variable cost; the C4 target takes LLM ~$18.29M -> ~$12.8M/mo and cost/MAU ~$9.73 -> ~$6.99 ([Cost §9](../17-cost-and-finops.md), [Capacity §10.1](../_canon/05-capacity-math.md)). Realized via: (1) higher Anthropic prompt-cache hit rate; (2) tighter relevance pre-gating (gate-pass below 35% where precision allows); (3) tier downshifting Sonnet->Haiku where confidence permits; (4) semantic-cache widening. Each is observable on the `admin-ops-service` cost-center board; a weekly canon-reconciliation guardrail flags if realized `C1` drifts > 15% above canon `$4.06`.

**Acceptance checks:** LT-3 asserts cost shape (tier mix Haiku 45% / Sonnet 55% / Opus <1%, gate-pass ~35%) as a CI gate; the cost-center board shows the realized LLM line trending to the C4 ~$12.8M/mo envelope; an alert fires if `C3` drifts below 90% or above 96%.

---

## 4. Data stores stood up in this band

Per [roadmap 22-roadmap-and-phasing.md §3.1], the MVP bands stand up the single-region pooled stores; **this band escalates them, it does not introduce new store types**:

- **PostgreSQL (+ AGE `nexus_knowledge`):** primary + read replica + PITR per region; monthly range partitions already in place; this band adds **cells** (dedicated instances) and the **Citus shard-by-`org_id`** escalation (P14.7.a) and the cross-region async streaming replica (P14.6.b).
- **Qdrant:** per-tenant collections escalated to **int8 quantization + hot/cold split** at `V7`/`A18` (P14.7.d); dedicated clusters for cell tenants; cross-region snapshot/re-derive for DR.
- **Redis (Upstash):** sharded **Redis Cluster** (hash-tagged by `{org_id}`) for keyspace caches; replicated backbone for pub/sub + presence + priority queues; `nexus:{org_id}:...` namespacing unchanged.
- **Redpanda:** **BYOC/dedicated multi-AZ** cluster, tenant-partitioned, + **MirrorMaker2** in-zone mirror (P14.7.c, P14.6.b).
- **Object buckets** (`nexus-recordings`, `nexus-documents`, `nexus-attachments`, `nexus-invoices`, `nexus-audit-archive`, `nexus-exports`): regional per `object_bucket_region`; cross-region async replication for DR; object-lock (WORM) on `nexus-invoices` (7-yr) and `nexus-audit-archive`.
- **Temporal Cloud:** namespace per region; worker fleet on K8s (`prod-async`).

The two principles hold throughout: **Postgres is the system of record; every other store is derivable (Qdrant, AGE) or ephemeral (Redis) or replayable (Redpanda)** ([Reliability §1, §4](../14-reliability-and-dr.md)). The one non-derivable exception is the **per-tenant KMS data keys** for `oauth_tokens`, backed up per the key-management DR posture.

---

## 5. Events / topics introduced

**This band introduces NO new topics and NO new Avro schemas.** All 21 topics and the 7 FROZEN schemas (`TranscriptChunk`, `AgentResult`, `Decision`, `Action`, `Notification`, `AgentRequest`, `AgentResponse`, each opening with `EventEnvelopeHeader`) are already frozen from the MVP bands. What this band changes is **placement and durability of the existing bus**:

- **BYOC/dedicated multi-AZ Redpanda** replaces the Serverless trial; subjects stay `{topic}-value`, BACKWARD compat, partitioned by `org_id` (high-volume by `{org_id}:{meeting_id}`), each paired with `<topic>.dlq`.
- **MirrorMaker2** mirrors all topics to the in-zone passive region only; consumers replay from last committed offset on failover; the idempotent inbox dedupes redelivery (`nexus:{org_id}:dedupe:{topic}:{idempotency_key}`).
- The **strangler-fig bus cut-over** (P14.2-P14.5) points each extracted service at the *same* topics it consumed/produced as a monolith module — no schema change, only deployment. Producers/consumers per [Service Catalog §5](../_canon/01-service-catalog.md) are unchanged: e.g. `transcription-service` -> `transcript-chunks` -> {`embedding-service`, `agent-runtime-service`, `search-service`}; `core-api` is the SOLE producer of confirmed `decisions`/`actions`; `agent-runtime-service` produces `agent-results`/`agent-stream`/`agent-requests`/`agent-responses`/`usage-events`/`notifications`.
- **KEDA-on-lag** (Kafka scaler) reads the same Redpanda consumer groups as the scaling signal for the AI/async tiers.

> Redpanda is NOT an RTBF deletion target — it is a replayable log whose tenant-partitioned data ages out by retention, not point-delete ([Reliability §4.2](../14-reliability-and-dr.md)).

---

## 6. SaaS overlay / preconditions landing here

- **Security shift-left (standing gate, not a phase):** `gitleaks` secret scanning from day one, `pip-audit`/`npm audit`/Dependabot, Trivy/Grype image scan, `tfsec`/`checkov` IaC scan, and the **authz check reviewed in every PR** (a custom lint asserting `org_id` injection on every new query path) all gate every PR ([Deployment §5.2](../16-deployment-and-infrastructure.md)). Secrets via ESO + Cloud KMS; **no secret in git/image/manifest**; per-tenant KMS data keys for `oauth_tokens` decrypted only in `integration-service`.
- **Compliance gate (data residency):** zone-pinned residency with in-zone failover (P14.6.b); the `data_residency` row drives every store's placement; a CI invariant + router assertion fail-closed any cross-zone misconfiguration; region change is the D5 Temporal saga, never an in-place update ([ADR-0009](../adr/0009-data-residency-multi-region.md)).
- **Billing + budgets:** the `usage-events -> usage_aggregates -> agent_budgets` pipeline (extracted in Wave 3) enforces per-tenant hard caps deny-by-default (100% SLO); the C4 LLM reduction (P14.7.f) is realized through the standing FinOps loop and the `admin-ops-service` cost boards ([Cost §6, §7, §10](../17-cost-and-finops.md)).
- **Dedicated inference:** the GPU node pool (`ai_gpu`) is a first-class hardware boundary for `embedding-service`/`reranker-service`, off the API tier, autoscaled on GPU util independent of CRUD req/s ([ADR-0014 §D3], [ADR-0008]).

---

## 7. Mobile (mytheus-mobile) work in this band

**No new mobile build work in this band.** This is a platform/infrastructure band; the mobile client realignment (remove `src/data/mock.ts`, add Clerk auth, wire to `bff`/`realtime-gateway`/LiveKit) lands in the earlier MVP bands, not here.

The only mobile-relevant requirement is **tolerance**, verified, not built:

- **Regional routing tolerance:** the mobile client must resolve the API endpoint through the Cloudflare geo + health-checked failover edge (the same `api-gateway` ingress as web), so a region failover (P14.6.b) transparently re-routes the device to the surviving in-zone region. The client must **reconnect** its `realtime-gateway` Socket.io/WSS session and re-mint its LiveKit token on a failover rather than assuming a pinned host — consistent with the "in-flight meetings reconnect, they do not migrate" rule ([Reliability §3.1](../14-reliability-and-dr.md), [ADR-0009 §"What we accept"]).
- **Cell affinity tolerance:** for an Enterprise tenant served from a `prod-cell-{slug}`, the client must follow the `identity-tenant-service` `org_id` -> posture+cell resolution and never hard-code a pooled host; cell routing is server-side, so the client only needs to honor the resolved base URL and not cache a stale region/cell.

**Acceptance check:** a region-failover GameDay with a mobile device in a live meeting shows the client reconnecting `realtime-gateway` + re-establishing the LiveKit room in the in-zone passive region within the reconnect budget; an Enterprise-cell tenant's mobile traffic lands on the cell, not the pooled estate.

---

## 8. DECISION-REQUIRED callouts

> ### DECISION-REQUIRED 1 — Per-wave extraction trigger metrics + owners (demand-driven; re-collapse is the rollback)
> Each extraction wave (P14.2-P14.5) is **demand-driven: extract a service only when its autoscaler signal diverges from the CRUD tier under real load.** Re-collapsing the service back into the monolith module is the rollback (the dual-run window is the safety net until the module is deleted). **Decision needed:** define and ratify the exact per-wave trigger metric and the owning role for each, e.g.
> - **Wave 1** (`embedding-service`/`reranker-service`/`agent-runtime-service`/`transcription-service`/`realtime-gateway`/`workflow-service`): forced — extract at first GPU need / first KEDA-on-connections need / first Temporal worker need (binary trigger). Owner: **Platform Eng**.
> - **Wave 2** (`core-api` then `identity-tenant-service`): trigger on CRUD p95 approaching 300 ms at HPA max, or Postgres connection pressure. Owner: **Backend Eng**.
> - **Wave 3** (`billing-metering-service`/`integration-service`/`admin-ops-service`): trigger on metering-ingest lag or webhook-burst saturation diverging from CRUD. Owner: **SaaS/Billing Eng**.
> - **Wave 4** (`knowledge-graph-service`/`search-service`): trigger when AGE Cypher load or FTS index ops each warrant dedicated ops. Owner: **Intelligence Eng**.
>
> **Who must decide:** Platform Eng lead + each service-owning lead, ratified by the architecture owner. **What it blocks:** starting any wave prematurely (wasted ops complexity) or too late (SLO breach). Until the trigger + owner table is signed off, **no wave may begin**.

> ### DECISION-REQUIRED 2 — At-scale graph store (keep AGE vs Neptune/Neo4j Aura) + the sharding trigger
> Two coupled at-scale calls:
> - **Graph store:** keep **Apache AGE on Postgres** (co-located, RTBF-cheap, rebuildable from confirmed `decisions`/`action_items`) vs migrate `nexus_knowledge` to **Amazon Neptune / Neo4j Aura** (R1 cold operational fallback, with a relational materialized-view floor). The migration is a **planned cutover with dual-write + verification, never a live failover**. **Decision criterion:** AGE proving "structurally awkward on managed Supabase" at scale (Cypher read-replica saturation, operational toil). Owner: **Data/Platform Eng**, ratified by architecture owner. **What it blocks:** Wave 4 split (P14.5) ops model and the P14.7.b escalation budget.
> - **Sharding trigger:** the precise saturation point at which single-primary Postgres warrants **Citus shard-by-`org_id`** — i.e. after vertical + read replicas + monthly partitions + cells are exhausted at `T3`=1.728B rows/mo. **Decision needed:** the concrete metric (e.g. sustained primary write-IOPS / WAL throughput / connection ceiling threshold) that fires the shard, and that the graph is **excluded** from Citus and follows the Neptune/Aura path instead. Owner: **Data Eng**, ratified by architecture owner. **What it blocks:** P14.7.a; firing too early adds distributed-query complexity, too late risks the write ceiling.

---

## 9. Risk checkpoints

- **R1 (graph unavailability) — hot vs cold, both checkpointed here.** *Hot:* `knowledge-graph-service` slow/down -> RRF drops the AGE topic-relations arm, continues on Qdrant + FTS; `ProactiveAgent` -> `decisions__{org_id}` Qdrant path. Validated by the Chaos Mesh `knowledge-graph-service` kill (RAG p95 < 1.5s holds). *Cold:* the AGE -> Neptune/Aura migration (P14.7.b, DECISION-REQUIRED 2) — planned cutover, not live failover ([Reliability §5.3](../14-reliability-and-dr.md)).
- **R6 (multi-agent collaboration) — hot fall-through chain preserved across extraction.** `ContextGapAgent`'s `agent-requests`/`agent-responses` chain `peer-agents -> related-meeting agents -> RAG -> silent` with the A2A round-trip p95 < 1.5s deadline as the breaker must survive the `agent-runtime-service` extraction (P14.2.b) unchanged; tenant-scoped partitions keep an `agent-request` from ever crossing `org_id`. Validated by the peer-agent-silence Chaos experiment ([Reliability §5.2](../14-reliability-and-dr.md)).
- **R8 (integrations off the critical path) — preserved by Wave 3 placement.** `integration-service` (P14.4) consumes/produces events asynchronously; a provider outage queues outbound syncs in the outbox + retries, never blocking a live meeting; `ActionFollowUpWorkflow` resumes its durable timers after the outage ([Reliability §5.4](../14-reliability-and-dr.md)).

**Chaos engineering (the band's proof):** wire `mytheus-platform/chaos/` with **Chaos Mesh** (in-cluster pod/network/IO/stress) + **AWS FIS** (AZ/region) experiments, each with a steady-state SLO hypothesis, blast radius scoped to a staging cell or one canary tenant cell, and automatic rollback on SLO breach. The catalog: region-kill, Postgres-primary-kill, Deepgram-outage, `knowledge-graph-service`-kill, LLM-503-storm, Redpanda-consumer-lag, token-budget-saturation, Redis-flush, peer-agent-silence, AZ-kill ([Reliability §6](../14-reliability-and-dr.md)). GameDays quarterly against staging cells; region-kill + Postgres-promotion before every major release.

---

## 10. Definition of Done & end-of-phase demo

A demo at the end of EVERY sub-phase (PDF cadence):

| Sub-phase | Definition of Done | End-of-phase demo |
|---|---|---|
| P14.1 substrate | `terraform apply` stands a region cluster from zero; `tfsec`/`checkov` clean; `platform` operators (ESO/KEDA/Argo) running; `nexus-deploy` GitOps reconciling | Run `terraform apply` live -> `kubectl get nodes -L family` shows 5 pools; revert an `image.digest` in `nexus-deploy` -> Argo reconciles back |
| P14.2 Wave 1 | All 6 Wave-1 services on K8s with per-family KEDA/HPA; GPU canary + connection-drain + Temporal versioning rollouts proven; monolith modules drained+deleted; expand/contract migration lint live | LT-1/LT-2/LT-4 pass at anchors; a live-meeting rollout drains without caption drop; GPU canary auto-aborts on injected SLO regression |
| P14.3 Wave 2 | `core-api` then `identity-tenant-service` extracted to `prod-crud`; HPA on req/s+p95; PgBouncer ceiling holds; Clerk webhook user-sync intact | LT-6 CRUD p95 < 300 ms while HPA fans to max; PgBouncer connections flat |
| P14.4 Wave 3 | `billing-metering-service`/`integration-service`/`admin-ops-service` extracted; metering 100% + idempotent; integrations off critical path | Simulated Slack/Jira outage: no live-meeting regression; cost board renders per-tenant COGS |
| P14.5 Wave 4 | `knowledge-graph-service`/`search-service` split out of `rag-service`; AGE-arm hot-drop works; `$org_id` templates only | Chaos KG-kill: RAG p95 < 1.5s on Qdrant+FTS |
| P14.6 cells + multi-region | Enterprise cell on tainted pool with RLS FORCE + dedicated data; in-zone active-passive with Cloudflare failover; per-plan SLO tiers wired | Region-kill GameDay: Pro RTO <= 30m, <= 5s loss, in-zone resume; cell tenant unaffected by pooled LT-9 |
| P14.7 escalations | Each fired path proven on a staging clone: Citus shard rehearsal, AGE->Neptune dual-write parity, Redpanda BYOC soak, int8 Qdrant, GPU autoscale, C4 LLM reduction realized | LT-7 24h soak (lag<5s, DLQ<0.1%); cost board shows LLM trending to C4 ~$12.8M/mo envelope |

**Band DoD:** the modular monolith is fully decomposed into the 20 canonical services on Kubernetes across the residency zones in use; per-plan availability tiers (Free 99.5% / Pro 99.9% / Enterprise 99.95%) are met and chaos-proven; the demand-driven escalation paths are documented, rehearsed, and individually trigger-gated; and **no canon name, number, or seam was re-cut** — every extraction followed the frozen Wave 1-4 order.

---

## 11. Source docs

- [16-deployment-and-infrastructure.md](../16-deployment-and-infrastructure.md) — topology, Terraform layers, Helm/GitOps, family-aware rollout, secrets, environment matrix, Railway/Render -> K8s migration.
- [14-reliability-and-dr.md](../14-reliability-and-dr.md) — per-tier availability, RPO/RTO, per-store DR matrix, circuit breakers (R1/R6/R8), chaos catalog.
- [06-multi-tenancy.md](../06-multi-tenancy.md) — pooled+RLS / cells, six isolation layers, `app.current_org`, cell node pools, RTBF.
- [13-scalability-and-performance.md](../13-scalability-and-performance.md) — four autoscaler signals, HPA/KEDA configs, data-tier escalation ladders, SLOs, load-test scenarios LT-1..LT-9.
- [_canon/05-capacity-math.md](../_canon/05-capacity-math.md) — the frozen anchors/symbols (A1/A4/A5/A6/A10/A18/E12/E13/L9/L10/EM2/EM4/V7/V9/T3/C3/C4).
- [_canon/01-service-catalog.md](../_canon/01-service-catalog.md) — the 20 service names, scaling families, Wave 1-4 split order (§7).
- [17-cost-and-finops.md](../17-cost-and-finops.md) — cost stack, the C4 ~30% LLM reduction, per-tenant attribution, budget state machine.
- [03-container-architecture.md](../03-container-architecture.md) — C4 L2 containers, gRPC/REST internal mesh, per-family autoscaler table, store ownership.
- [adr/0014-kubernetes-compute-platform.md](../adr/0014-kubernetes-compute-platform.md) — the compute-substrate decision, node pools, Terraform, strangler-fig migration, trigger conditions.
- [adr/0009-data-residency-multi-region.md](../adr/0009-data-residency-multi-region.md) — zones, in-zone failover rule, zone->store resolution, region-change saga.
