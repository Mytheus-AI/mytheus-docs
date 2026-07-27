# STRIDE threat model

P1 design artifact (Plan 1-2 section 3.1 step 7). Derived from [11-security-architecture.md](../11-security-architecture.md) section 8.

> [SaaS-scale adjustment] The original plan batched a security audit into Phase 13. This model is authored in P1, before the code, which is the shift-left replacement. A threat found on paper in week 2 costs a paragraph; the same threat found in week 34 costs a redesign.

## 1. Scope and completeness

Six surfaces by six STRIDE categories. Plan 1-2 requires the **Auth** and **APIs** rows to be complete; the other four are stubbed to their canonical controls and completed by their owning bands.

| Surface | Row status | Completed by |
|---|---|---|
| Auth (Clerk, sessions, SSO) | **Complete** | This band |
| APIs (`/api/v1`, `/bff/*`) | **Complete** | This band |
| Transcripts (T4 content) | Stubbed to canon controls | Plan 3-4 |
| Multi-agent context (A2A) | Stubbed to canon controls | Plan 10-13 |
| Integrations (OAuth, webhooks) | Stubbed to canon controls | Plan 10-13 |
| Subprocessors | Stubbed; register executed this band | This band for the register, later bands per integration |

## 2. Surface 1: Auth (COMPLETE)

| STRIDE | Threat | Control | Verified by |
|---|---|---|---|
| **S**poofing | Forged or replayed session JWT | Clerk RS256 verified against cached JWKS at `api-gateway` and RE-VERIFIED in the service tier. `exp`, `nbf`, `iss` checked. The gateway is a DDoS shield, not a trust delegation. | Unit test: tampered and expired tokens raise. |
| Spoofing | Forged internal `x-mytheus-org` header from inside the network | The gateway strips any inbound copy and signs its own. The receiving service verifies the signature before trusting it. | Test: a hand-crafted header at the service tier is rejected. |
| Spoofing | Forged Clerk webhook | Svix HMAC verified **before** the body is parsed and before any organization resolution. | Test: bad signature returns `401` and performs zero resolutions. |
| **T**ampering | Client asserts its own tenancy | `org_id` is never accepted from a path, query, body, or client header. It is derived only from the verified token. | Spectral rules `org-id-never-in-path` and `org-id-never-in-query`. |
| Tampering | Privilege escalation by editing a profile field | Authorization reads `rbac_grants.tier` only. `users.role` is a job attribute with no permission meaning, so renaming a job title cannot change access. | Schema: two tables, two columns. Code review. |
| **R**epudiation | User denies an action | `audit_logs` append-only, live from the moment RLS is. Sign-in, `authz.deny`, and webhook applies are recorded. | Test: `UPDATE` and `DELETE` as `mytheus_app` raise. |
| **I**nformation disclosure | JWKS outage leaks internals or fails open | Keys cached for the TTL so a Clerk blip serves cached keys. On unreachable JWKS the verifier fails **closed** with `401`, never `500`. **Known defect:** `PyJWKClientError` does not derive from `InvalidTokenError`, so the current handler misses it and returns `500`. Fix in section 3.2. | Test: JWKS unreachable returns `401`. |
| Information disclosure | The edge becomes an existence oracle over the customer base | Unknown org, suspended org, and soft-deleted org return one byte-identical `401`. Unique-constraint violations on `slug` and `clerk_org_id` map to a generic `409` echoing neither constraint name nor value. | Test: the three negative cases are byte-identical. |
| **D**enial of service | Credential stuffing, token brute force | Cloudflare WAF, global L7 rate limit, per-tenant rate limit at the gateway, Clerk bot detection. Signature verification happens at the edge so forged tokens never consume a worker. | Load test at the gateway. |
| **E**levation of privilege | Authorization fails open | The middleware fails closed: unset `org_id`, unknown route, or missing tier raises `403`. Paired with RLS, where an unset `app.current_org` matches zero rows, so an authz bug degrades to "user sees nothing", never "user sees everything". **Known defect:** `grpc_server.py` catches every exception and synthesizes a valid context with a default tier, which is fail-OPEN. Fix in section 3.3. | Test: database unreachable must abort, not grant. |

## 3. Surface 2: APIs (COMPLETE)

| STRIDE | Threat | Control | Verified by |
|---|---|---|---|
| **S**poofing | Unauthenticated route ships by accident | Every endpoint declares its auth requirement in the OpenAPI spec. `scripts/check_authz.py` AST-walks the routes and fails the build on an undeclared one. Spectral requires document-level `security`. | CI gate, required for merge. |
| **T**ampering | Cross-tenant write via a forgotten scope predicate | RLS `FORCE` with `WITH CHECK` prevents inserting or moving a row into another `org_id`. The predicate returns zero rows when the GUC is unset. | Test: tenant A cannot read or write tenant B; no-GUC returns zero rows. |
| Tampering | Double-applied mutation on retry | `Idempotency-Key` on every mutating POST. Same body replays the original response; different body returns `409 IDEMPOTENCY_KEY_REUSED`. | Test: both branches. |
| **R**epudiation | Mutation with no trail | `meeting.created`, `meeting.updated`, `org.provisioned`, `user.synced`, and `authz.deny` all write `audit_logs` rows. An operation that requires audit fails closed if the audit write fails. | Test: creating a meeting writes the row. |
| **I**nformation disclosure | API confirms a resource exists in another tenant | Cross-tenant access returns `404`, never `403`. Existence is tenant-confidential. Enforced in the shared resource-fetch helper, not per route. | Test: `404` asserted distinct from the `403` role case. |
| Information disclosure | Idempotency cache serves one tenant's response body to another | The cache key must be namespaced by `org_id` and bound to a body hash. **Known defect: it is not.** The current key is built from the client-supplied header alone, so a replayed key returns another tenant's cached body with no database access at all, defeating RLS entirely. This is the most severe open finding in the codebase. Fix in section 3.1. | Test: two tenants, same key, disjoint responses. |
| Information disclosure | Stack traces or internals in an error body | RFC 7807 only. `500` carries a `trace_id` and nothing else. **Known defect:** bare `HTTPException` responses bypass the problem+json handler. Fix in section 3.4. | Spectral `errors-use-problem-json`. |
| **D**enial of service | Expensive unbounded list query | Cursor pagination only, `limit` default 50 and max 200, rejected above with `422`. No `OFFSET`. No default `COUNT(*)`. | Spectral `list-endpoints-are-cursor-paginated`. |
| Denial of service | One poison outbox row halts all tenants' events | Per-row error handling, `attempts` increment, and a produce-side DLQ after a retry ceiling. **Known defect: none of these exist.** A single unserializable row crash-loops the relay forever, blocking every tenant. Fix in section 3.5. | Test: a poison row lands in the DLQ and the drain continues. |
| **E**levation of privilege | An operator or service reaches another tenant's rows | No role holds `BYPASSRLS`. The relay gets cross-tenant reach on `outbox` alone, through one permissive policy on that one table. | CI: `rolbypassrls` and `rolsuper` are false for every application role. |

## 4. Surfaces 3 to 6 (stubbed to canonical controls)

| Surface | Canonical controls, to be expanded in the owning band |
|---|---|
| **Transcripts (T4)** | Only finals persisted to `transcript_chunks`. Deepgram key stays server-side, proxied. PII redaction before LLM egress where `tenant_settings.pii_redaction_enabled`. Per-tenant retention drives partition drops. Transcript reads are RLS-scoped like everything else. |
| **Multi-agent context** | Deny-by-default agent-to-agent authorization. An agent inherits its principal's tenant. A request is authorized only if requester and responder share `org_id` AND the grant scope permits it. Every attempt logged to `agent_collaborations`. A foreign `org_id` on the bus goes to the DLQ, never to a handler. |
| **Integrations** | OAuth tokens envelope-encrypted with a per-tenant KMS data key, never returned by any API, never logged. Provider webhooks signature-verified. Scopes least-privilege per connection. |
| **Subprocessors** | Signed DPAs plus SCCs before any T4 data flows. Regional routing per `data_residency`. Server-side keys only. See [subprocessors.md](../design/subprocessors.md). |

## 5. Open defects this model surfaced

Five findings, all in code that already exists, all reachable today. Ordered by severity. These are not hypothetical threat-model entries: they were confirmed by reading the source.

| # | Defect | Why it matters |
|---|---|---|
| 1 | **The application connects to PostgreSQL as the superuser** in every environment including the deployed one. Superusers bypass RLS unconditionally. | The entire fail-closed foundation is inert. Every acceptance check that claims to prove isolation proves it only for a role the application never uses. |
| 2 | **The idempotency cache is not tenant-namespaced.** | A live cross-tenant leak of full response bodies, requiring no RLS bypass and no database access. |
| 3 | **The identity gRPC server fails open.** A bare `except Exception` returns a valid context for the caller's asserted `org_id` with a default tier. | Becomes "database unreachable therefore grant access" the moment it reads real permissions. |
| 4 | **The outbox relay will read zero rows and report an empty queue** once the correct roles are wired. | Silent, total event loss with no error, no log, and no metric that distinguishes it from an idle queue. |
| 5 | **One poison outbox row crash-loops the relay forever.** `attempts` exists and is never written; there is no produce-side DLQ. | Head-of-line block across every tenant. |

Defects 1 and 4 share a root cause, which is why the database-role work is the first task of the build phase rather than a naming tidy-up.

## 6. Related

- [11-security-architecture.md](../11-security-architecture.md) is the authoritative narrative.
- [ADR-0002](../adr/0002-multi-tenancy-model.md), [ADR-0010](../adr/0010-auth-clerk-enterprise-sso.md).
- [design/rtbf-contract.md](../design/rtbf-contract.md), [design/subprocessors.md](../design/subprocessors.md).
