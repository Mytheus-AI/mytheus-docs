# design

The P1 design pack for [Plan 1-2 Foundation and Tenancy](../plan/plan-1-2-foundation-and-tenancy.md), section 3.1. These artifacts freeze the contract that every P2 build step is written against, and they exist before the code so the architecture does not calcify around an unreviewed decision.

This folder **records and specifies**; it never **defines a name**. Canon (`_canon/`) remains the only naming authority.

## Index

| Artifact | Plan step | What it fixes |
|---|---|---|
| [c4-l1-system-context.md](c4-l1-system-context.md) | 3.1.1 | External actors, subprocessors, the four trust zones |
| [c4-l2-container.md](c4-l2-container.md) | 3.1.1 | Containers, the three scaling families, the edge path |
| [erd.md](erd.md) | 3.1.2 | The whole target entity model and the partitioning decisions |
| [avro/README.md](avro/README.md) | 3.1.3 | Where the event schemas live, and two field defects to fix before registration |
| [openapi/](openapi/) | 3.1.4 | One OpenAPI 3.1 spec per service, plus the Spectral ruleset |
| [wireframes/README.md](wireframes/README.md) | 3.1.5 | Layout and data contracts for sign-in, the app shell, and the dashboard |
| [tech-lock.md](tech-lock.md) | 3.1.6 | The locked stack. **Sign-off gate: initialled before the first migration.** |
| [../threat-model/stride.md](../threat-model/stride.md) | 3.1.7 | STRIDE, with the Auth and API rows complete |
| [data-residency.md](data-residency.md) | 3.1.8 | The region set, the MVP default, and what single-region really means |
| [subprocessors.md](subprocessors.md) | 3.1.9 | The register and per-processor DPA status |
| [rtbf-contract.md](rtbf-contract.md) | 3.1.10 | The cross-store deletion contract every table is designed to satisfy |

## Gating

Two artifacts block P2 work rather than merely informing it:

- **[openapi/](openapi/)** gates section 3.10 step 7, which generates the TypeScript client from these specs, and section 3.11 step 5, which runs Spectral and `oasdiff` against them.
- **[tech-lock.md](tech-lock.md)** is a sign-off gate. Section 3.3, the first migration of the band, does not start until its table is initialled. It is currently **UNSIGNED**.

## Deviations from the plan text, and why

Recorded here rather than discovered later. Every one of these is a place where the plan text was written before the facts changed.

| Plan text | What was done | Why |
|---|---|---|
| 3.1.3: draft `*.avsc` files in `design/avro/` | A reference page, zero schema files | `mytheus-events` already ships all four band schemas, registered and under a BACKWARD gate. A copy here could never be validated, so it could only drift. |
| 3.1.5: design the wireframes | Annotated layout and data contracts for screens that already exist | `mytheus-ui` has built every one of them. Speculative wireframes would create a competing design authority. |
| 3.1.6: lock "Supabase, PgBouncer, Upstash" | Locked self-hosted PostgreSQL, Redis, Redpanda, and Temporal; PgBouncer explicitly deferred with an exit trigger | [ADR-0016](../adr/0016-self-hosted-vps-mvp-substrate.md) supersedes the plan text, which predates it. |
| 3.1.9: "execute the DPAs" | Register plus per-processor owner and status | Signing is a legal action, not an engineering one. The band's own DoD wording accepts pending-with-owner. |
| Implicit: `meetings` is range-partitioned | Deferred, with a recorded exit trigger | Canon says "candidate at scale". Partitioning it would make tenant-safe composite foreign keys impossible. See [erd.md](erd.md) section 4. |
| 3.6: `GET /api/v1/orgs/{id}/teams` | `GET /api/v1/teams` | The plan's own form puts a tenant identifier in the path, violating the `org_id`-never-in-path invariant it also states. |

## Open items surfaced by writing this pack

These are findings, not opinions. Each was confirmed against the source.

1. **The application connects to PostgreSQL as the superuser in every environment.** Superusers bypass RLS unconditionally, so the whole isolation foundation is currently inert.
2. **The idempotency cache is not tenant-namespaced**, which is a live cross-tenant leak of response bodies that never touches the database.
3. **The identity gRPC server fails open** on any exception.
4. **The outbox relay will silently drain nothing** once correct database roles are wired, reporting an empty queue.
5. **One poison outbox row crash-loops the relay forever**, with no `attempts` increment and no produce-side dead-letter path.
6. **Seven tenant tables have no foreign key to the tenant root**, which breaks the RTBF completeness guarantee.
7. **Tenant provisioning has no reachable path** under the resolved tenant-claim decision, and is deferred with an operator seed script standing in.
8. **`organizations.status` was missing** and is now added to canon; the edge checkpoint is specified to check suspension and previously had no column to read.

Items 1 to 5 are detailed in [../threat-model/stride.md](../threat-model/stride.md) section 5. Items 6 and 7 are in [rtbf-contract.md](rtbf-contract.md) section 5 and [openapi/identity-tenant-service.yaml](openapi/identity-tenant-service.yaml).
