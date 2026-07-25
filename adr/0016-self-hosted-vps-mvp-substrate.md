# ADR-0016: MVP compute substrate - self-hosted single-node VPS, not a PaaS

This ADR replaces **Railway/Render** with a **self-hosted single-node VPS running Docker Compose** as the substrate for the 36-week MVP build. The whole backend (the FastAPI modular monolith, the outbox relay, the Temporal worker) and every stateful store (PostgreSQL, Redis, Redpanda, Temporal) run on that one box behind Caddy for TLS. Kubernetes (EKS/GKE) + Terraform remains the production target, unchanged, and is still reached only through Plan 14.

## Related documents

- [ADR-0014: Compute platform - Kubernetes (EKS/GKE) + Terraform IaC](0014-kubernetes-compute-platform.md) - the decision this one partially supersedes. Its Kubernetes production substrate stands; only its "Railway/Render early-stage fast path" is replaced.
- [Deployment & Infrastructure](../16-deployment-and-infrastructure.md) - the environment matrix and the MVP-to-Kubernetes migration this ADR re-bases.
- [Tech & Architecture Decisions](../_canon/06-tech-decisions.md) - the locked `Compute platform` row (section 2.1) that this ADR is the record behind.
- [ADR-0002: Multi-tenancy model](0002-multi-tenancy-model.md) - pooled + PostgreSQL RLS, which is what makes a single-node MVP tenable at all.
- [Reliability & DR](../14-reliability-and-dr.md) - the availability posture the single-node substrate constrains.
- [Cost & FinOps](../17-cost-and-finops.md) - the fixed-cost profile this decision produces.
- [plan/plan-14-scale-out.md](../plan/plan-14-scale-out.md) - the demand-driven extraction to Kubernetes, whose starting point this ADR changes.

---

## Status

**Accepted.** Supersedes [ADR-0014](0014-kubernetes-compute-platform.md) **in part**: specifically the clause preserving "Railway/Render as the sanctioned early-stage modular-monolith fast path". Every other decision in ADR-0014 - Kubernetes (EKS/GKE) as the production substrate, Terraform IaC, one cluster per region, the four family-aligned node pools, the HPA/KEDA/Ray Serve autoscalers, and the frozen Wave 1-4 extraction order - is untouched and still binding.

Railway and Render are struck entirely. They are not the fast path, and they are not a fallback. A document that still presents them as either is a bug against this ADR.

---

## Context

### What the PDF and ADR-0014 assumed

The NEXUS Execution Plan v2.1 specified Railway/Render for services. [ADR-0014](0014-kubernetes-compute-platform.md) accepted that for the early build and locked Kubernetes for production, on the reasoning that a PaaS is git-push-to-deploy with no infrastructure code and no cluster to operate: the correct trade for a small team with one application, one environment, and no tenants yet.

That reasoning was sound but rested on an assumption that no longer holds: that the alternative to a PaaS is *provisioning and paying for* infrastructure.

### What changed

A suitable VPS is already owned and available to the team at no marginal cost: a Hostinger KVM instance, 8 vCPU / 32 GB RAM / 400 GB disk, Ubuntu 24.04. The operational address and credentials live with the deploy runbook in `mytheus-platform`, not in this document.

That inverts the PaaS trade. The question is no longer "pay a platform to avoid operating a box" but "use a box we already have, or pay a platform to avoid using it". At MVP scale the second option buys convenience that the first already provides.

### The forces

1. **The MVP workload fits one node comfortably.** The MVP is a modular monolith serving the tenant-isolated CRUD spine: identity, meetings, teams, audit. Pre-revenue tenant counts are single digits. 8 vCPU / 32 GB carries the FastAPI process, the relay, the Temporal worker, and all four stores with substantial headroom. The capacity anchors that force Kubernetes (`A5` = 20,000 peak meetings, `A6` = 100,000 peak participants, `EM2` ~ 525 embeddings/sec, per [Capacity](../_canon/05-capacity-math.md)) are targets for the scaled estate, not the MVP, and none of them is approached before Plan 14.

2. **Marginal cost is zero, and it is fixed.** Railway and Render bill per service and per resource, and the MVP stack is 9 containers including four stateful services. A PaaS also pushes managed Postgres, Redis, and Kafka as separate paid add-ons, since it is a poor host for stateful workloads. On the VPS the cost is already sunk and does not scale with the number of containers or with traffic. For a pre-revenue build, a fixed known cost beats a variable one.

3. **Self-hosting the stateful tier was already the direction.** Redpanda and Temporal are awkward and expensive as PaaS add-ons. `docker-compose.yml` already ran Postgres, Redis, Redpanda, and Temporal locally for dev. The VPS makes production a deployment of the same topology rather than a different one, which removes a class of works-locally-fails-in-prod divergence.

4. **Full control of the box.** Root access permits tuning PostgreSQL for RLS-heavy workloads, sizing Redpanda partitions, and inspecting a live system directly. On a PaaS the same work is mediated by a dashboard and a support ticket.

5. **Nothing about the architecture is being bent to fit.** The seams from [Service Catalog section 7](../_canon/01-service-catalog.md) are unchanged. The services are still modules in one deployable, drawn along their final boundaries. The substrate under the monolith does not touch the extraction order or the domain boundaries. This is a hosting decision, not an architecture decision.

### What is deliberately not being claimed

The VPS is **not** presented as capable of production SaaS scale. It is a single node: no horizontal scale, no GPU pool, no KEDA-on-connections, no node-pool tenant placement, no multi-region. Every force in [ADR-0014](0014-kubernetes-compute-platform.md) that mandates Kubernetes still mandates it. This ADR changes only where the MVP is built, not where the product ends up.

---

## Options considered

| Option | Verdict |
|---|---|
| **Railway/Render** (the ADR-0014 fast path) | Rejected. Pays a per-service premium for convenience already available for free, and hosts the four stateful services badly or not at all. |
| **Self-hosted VPS, Docker Compose** (chosen) | Accepted. Zero marginal cost, full control, production mirrors the dev compose topology, and the MVP workload fits the node with headroom. |
| **Kubernetes immediately** | Rejected, and this remains ADR-0014's own position: at MVP load a cluster is pure operational overhead with no autoscaling, GPU, or residency need to justify it. |
| **Managed Postgres (Supabase) plus VPS compute** | Available as a per-store variant, not the default. `DATABASE_URL` selects it by changing one line, so a move to managed Postgres for backups and PITR is a config change rather than a migration. |

---

## Decision

The 36-week MVP backend runs on a **single self-hosted VPS under Docker Compose**:

- One image, built multi-stage, running three roles selected by the compose `command`: `web` (uvicorn, the modular monolith), `relay` (outbox drain), `workflow` (Temporal worker).
- Self-hosted `postgres`, `redis`, `redpanda` (Kafka API plus Schema Registry), and `temporal`.
- A one-shot `migrate` service running `alembic upgrade head`, which `web` waits on.
- `caddy` as the reverse proxy, terminating TLS with automatic HTTPS.

The deploy artifacts and the first-time runbook live in `mytheus-platform` under `deploy/`. Kubernetes, Terraform, cells, and multi-region stay out of scope until [Plan 14](../plan/plan-14-scale-out.md), and remain demand-driven rather than scheduled.

The **per-store split is preserved**: each backing store is selected by its own environment variable, so any single store can move to a managed provider without touching the others or the application code.

---

## Consequences

### Accepted costs

- **Operational burden transfers to the team.** PostgreSQL backups and PITR, Redpanda operation, OS patching, disk monitoring, and certificate renewal are now owned work. Caddy automates TLS; nothing automates the rest. A backup and restore drill is required before the first real tenant, not after.
- **No high availability.** One node is one failure domain. A host failure is a full outage, and there is no failover until Plan 14. The per-plan availability SLOs in [Reliability & DR](../14-reliability-and-dr.md) cannot be committed publicly while the MVP runs on this substrate.
- **A hard capacity ceiling.** 8 vCPU / 32 GB is ample for MVP and cannot reach the capacity anchors. Sustained resource pressure is a Plan 14 trigger signal, so the box must be monitored against that, not merely for liveness.
- **No preview environments for free.** A PaaS gives per-PR deploys cheaply. Staging on the VPS is another compose project or another box, and preview deploys are not solved by this decision.

### Benefits realized

- Zero marginal infrastructure cost for the whole MVP build.
- Production topology is the dev topology, which closes the environment-divergence gap.
- Root-level control of PostgreSQL, Redis, and Redpanda tuning.
- No vendor lock-in at the substrate layer: the migration target is Kubernetes, and the source is already plain containers.

### What is unchanged

Kubernetes + Terraform for production, one cluster per region, the four node pools, the family autoscalers, the Wave 1-4 extraction order, pooled + RLS tenancy, and every capacity anchor. Plan 14 is unchanged in content: only its starting point moves from Railway/Render to the VPS.

### Follow-ups this ADR creates

1. Back up PostgreSQL off-box, and rehearse a restore, before any real tenant data lands.
2. Monitor CPU, memory, disk, and Redpanda lag as Plan 14 trigger signals rather than as mere health checks.
3. Decide where the `web-app` frontend is hosted. This ADR covers the backend only, and the frontend host is a separate open decision.
4. Sweep the reader docs and the remaining plan bands for stale Railway/Render references, which now contradict canon.
