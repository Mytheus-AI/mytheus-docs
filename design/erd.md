# Entity Relationship Diagram

P1 design artifact (Plan 1-2 section 3.1 step 2). Derived from [_canon/02-data-model.md](../_canon/02-data-model.md) section 2 and [05-data-architecture.md](../05-data-architecture.md) section 2.

This is the **whole** target model, not the Plan 1-2 subset, so migrations never have to re-cut a table. Section 4 marks what this band actually builds.

## 1. Reading rules

- Every tenant-scoped table carries `org_id UUID NOT NULL` referencing `organizations(id)`, and carries a `tenant_isolation` RLS policy. This is exceptionless and asserted in CI.
- **Two tables are deliberately un-tenanted.** `organizations` is the tenant root: its `id` **is** the `org_id`, so it has no `org_id` column and no `tenant_isolation` policy ([_canon/02-data-model.md](../_canon/02-data-model.md) section 7 rule 7). `plans` is a global catalog. Everything else is tenant-scoped.
- Soft delete is `deleted_at timestamptz NULL`. Hard purge is the right-to-be-forgotten workflow; see [rtbf-contract.md](rtbf-contract.md).
- `org_id` is never a path or query parameter. It is derived from the verified JWT.

## 2. Core tenancy and identity

```mermaid
erDiagram
  organizations ||--o{ users : "has members"
  organizations ||--|| tenant_settings : "configured by"
  organizations ||--|| data_residency : "pinned by"
  organizations ||--o{ rbac_grants : "authorizes via"
  organizations ||--o{ departments : "structured as"
  organizations ||--o{ teams : "groups into"
  users ||--o{ rbac_grants : "granted"
  users ||--o{ team_members : "joins"
  teams ||--o{ team_members : "contains"
  departments ||--o{ teams : "owns"
  departments ||--o{ departments : "parent of"

  organizations {
    uuid id PK "IS the org_id"
    text clerk_org_id UK "Clerk org mapping, immutable"
    text name
    text slug UK
    uuid plan_id FK
    text status "active or suspended"
    timestamptz created_at
    timestamptz deleted_at
  }
  users {
    uuid id PK
    uuid org_id FK
    text clerk_user_id "UNIQUE with org_id"
    text email
    text role "domain attribute, NOT a permission tier"
    text_array skills
    text bio
    text avatar_url
    timestamptz deleted_at
  }
  rbac_grants {
    uuid id PK
    uuid org_id FK
    uuid user_id FK
    text tier "owner, admin, member"
    uuid granted_by FK
  }
  tenant_settings {
    uuid org_id PK
    text default_locale
    int retention_days_transcripts
    int retention_days_worklogs
    jsonb agent_confidence_overrides
    boolean pii_redaction_enabled
    jsonb sso_config
  }
  data_residency {
    uuid org_id PK
    text region "us, eu, ap"
    text db_cell
    text qdrant_region
    text object_bucket_region
    timestamptz locked_at
  }
```

**The two role concepts are separate stores by design.** `rbac_grants.tier` is the only thing authorization reads. `users.role` plus `users.skills` is job identity feeding agent relevance and `get_user_context`, and is never consulted for permissions. Overloading one column would make a profile field a security boundary. See [ADR-0010](../adr/0010-auth-clerk-enterprise-sso.md) D5.

## 3. Meetings, agent output, and the rest of the target model

```mermaid
erDiagram
  organizations ||--o{ meetings : "owns"
  meetings ||--o{ meeting_participants : "attended by"
  meetings ||--o{ transcript_chunks : "transcribed into"
  meetings ||--o{ meeting_speakers : "diarized as"
  meetings ||--o{ worklogs : "produces"
  meetings ||--o{ decisions : "produces"
  meetings ||--o{ action_items : "produces"
  meetings ||--o{ agent_suggestions : "prompts"
  meetings ||--o{ chat_messages : "carries"
  users ||--o{ meeting_participants : "participates"
  users ||--o{ worklogs : "owns"
  users ||--o{ action_items : "assigned"
  organizations ||--o{ audit_logs : "records"
  organizations ||--|| subscriptions : "billed by"
  plans ||--o{ subscriptions : "prices"
  organizations ||--o{ usage_events : "meters"
  organizations ||--o{ invoices : "invoiced"
  organizations ||--o{ agent_budgets : "budgeted"
  organizations ||--o{ integration_connections : "installs"
  integration_connections ||--o{ oauth_tokens : "holds"
  organizations ||--o{ outbox : "emits"
  organizations ||--o{ inbox : "consumes"

  meetings {
    uuid id PK
    uuid org_id FK
    text title
    timestamptz scheduled_start "PARTITION CANDIDATE"
    timestamptz scheduled_end
    text livekit_room
    text status
    uuid created_by FK
    timestamptz deleted_at
  }
  meeting_participants {
    uuid meeting_id PK
    uuid user_id PK
    uuid org_id FK "denormalized for the RLS predicate"
    text role_in_meeting
  }
  audit_logs {
    uuid id PK
    timestamptz occurred_at PK "RANGE PARTITIONED monthly"
    uuid org_id FK
    uuid actor_id
    text actor_type "user, agent, system"
    text action
    text target_type
    text target_id
    jsonb metadata
    inet ip
  }
  outbox {
    bigserial id PK
    uuid org_id
    text topic
    text partition_key
    text idempotency_key
    text schema_subject
    jsonb payload
    timestamptz published_at
    int attempts
  }
  inbox {
    text consumer_group PK
    uuid org_id PK
    text idempotency_key PK
    timestamptz processed_at
  }
```

`meeting_participants` and `team_members` carry `org_id` denormalized so the RLS predicate is a column compare and not a subquery ([06-multi-tenancy.md](../06-multi-tenancy.md) section 3.1).

## 4. Range partitioning

Canon marks seven tables for range partitioning. They are not all equal, and the distinction is load-bearing.

| Table | Partition key | Canon wording | Decision for this program |
|---|---|---|---|
| `audit_logs` | `occurred_at` monthly | Mandated | **Partition now**, in Plan 1-2. Append-only, nothing references it, so the composite primary key costs nothing. Pre-create twelve months plus a catch-all partition so a missed maintenance window can never fail an insert. |
| `transcript_chunks` | `created_at` monthly | Mandated | Partition from the first row in Plan 3-4. At T3 volume this is not optional. |
| `chat_messages` | `created_at` monthly | Mandated | Partition when the table is created. |
| `usage_events` | `occurred_at` daily | Mandated | Partition when the table is created, in the billing band. |
| `meetings` | `scheduled_start` monthly | **"Range-partition candidate at scale"** | **Deferred.** See below. |
| `worklogs` | `created_at` monthly | Candidate at scale | Deferred to its owning band. |
| `agent_suggestions` | `created_at` monthly | Candidate at scale | Deferred to its owning band. |

**Why `meetings` is deferred.** Three reasons, and the third is decisive.

1. Canon hedges it as a candidate at scale; only the reader docs state it as done. Canon wins.
2. The partition key must be non-null to sit in the primary key, which would force every ad-hoc meeting to carry a sentinel `scheduled_start`. A meeting created on the spot has no scheduled start.
3. PostgreSQL requires every unique constraint on a partitioned table to include all partition-key columns. So `UNIQUE (org_id, id)` becomes impossible, and with it the **tenant-safe composite foreign keys** that make a cross-tenant dangling reference unrepresentable:

```sql
ALTER TABLE meetings ADD CONSTRAINT uq_meetings_org_id UNIQUE (org_id, id);
ALTER TABLE meeting_participants
  ADD CONSTRAINT fk_mp_meeting_org
  FOREIGN KEY (org_id, meeting_id) REFERENCES meetings (org_id, id);
```

Without those, a participant row can carry `org_id = B` while pointing at a meeting owned by `org_id = A`. RLS hides the row from B's reads, so the corruption is silent, and the right-to-be-forgotten sweep cannot be trusted to find everything. Partitioning an empty table would permanently trade that guarantee away. For a product whose worst bug class is a cross-tenant leak, that trade is backwards.

**Exit trigger, recorded so it is not forgotten:** partition `meetings` when it exceeds roughly 5 to 10 million rows or when a listing query's p95 crosses its SLO, whichever comes first. At that point `meeting_participants` must be re-cut too. The conversion is a shadow-table swap and the partitioning machinery is proven on `audit_logs` in this band regardless.

## 5. What Plan 1-2 builds

| Built and complete | `organizations`, `users`, `tenant_settings`, `data_residency`, `rbac_grants`, `teams`, `team_members`, `departments`, `meetings`, `meeting_participants`, `audit_logs`, `outbox`, `inbox` |
|---|---|
| **Not built** | `transcript_chunks`, `meeting_speakers`, `worklogs`, `decisions`, `action_items`, `agent_suggestions`, `agent_collaborations`, `chat_messages`, `document_chunks`, `plans`, `subscriptions`, `usage_events`, `invoices`, `agent_budgets`, `integration_connections`, `oauth_tokens` |

Eight of the thirteen built tables already exist from Plan 0 but as thin shells missing canonical columns. Plan 1-2 widens them to canon rather than creating them fresh.

## 6. Related

- [_canon/02-data-model.md](../_canon/02-data-model.md) is the naming authority. Where this diagram and canon disagree, canon wins and this file is the bug.
- [rtbf-contract.md](rtbf-contract.md) for the deletion contract every table above is designed to satisfy.
- [ADR-0002](../adr/0002-multi-tenancy-model.md) for the RLS posture and the three database roles.
