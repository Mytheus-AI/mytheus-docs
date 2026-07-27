# Subprocessor register and DPA status

P1 design artifact (Plan 1-2 section 3.1 step 9). Derived from [02-system-context.md](../02-system-context.md) section 6.2.

## 1. Why this exists before the integrations do

Compliance-by-construction requires the subprocessor list and its signed agreements **before any transcript-class data flows**, not after the integration ships. Deepgram, LiveKit, OpenAI, and Anthropic are not called by any code in Plan 1-2, and their agreements are still executed in this band. Signing a DPA is measured in weeks of legal turnaround; discovering an unsigned one in the week you want to ship is a schedule failure with no engineering fix.

Every system marked as a subprocessor must also appear in the customer-facing subprocessor list, which is a contractual obligation to customers, not an internal artifact.

## 2. Data classification

| Class | Meaning | Examples |
|---|---|---|
| T4 | Highly sensitive. Meeting content. | Raw audio, `transcript_chunks`, agent prompts, decisions, action items |
| T3 | Sensitive personal data | Identity, email, org membership |
| T2 | Business data | Billing contacts, usage aggregates |

## 3. The register

| Processor | Receives | What exactly | Subprocessor | DPA | SCCs | Owner | Status |
|---|---|---|---|---|---|---|---|
| **Deepgram** | T4 raw | Raw meeting audio; returns transcript text | Yes | Required | Required if non-EU processing for EU tenants | Legal | **PENDING** |
| **LiveKit** | T4 raw | Audio and video media, recordings | Yes | Required | Required | Legal | **PENDING** |
| **OpenAI** | T4 raw | Transcript-derived prompts, windowed, pre-gated, redacted where configured | Yes | Required | Required | Legal | **PENDING** |
| **Anthropic** | T4 raw | Same as OpenAI, routed via LiteLLM | Yes | Required | Required | Legal | **PENDING** |
| **Qdrant Cloud** | T4 raw | Embeddings plus citation payload | Yes | Required | Required | Legal | **PENDING** |
| **Redpanda** | T4 raw | Avro event payloads in transit | Yes | Required if managed | Required if managed | Legal | **N/A at MVP, self-hosted on our own VPS. Becomes required if a managed or BYOC cluster is adopted.** |
| **Slack** | T4 derived | Meeting briefs and summaries | Yes | Required | Required | Legal | PENDING, integration band |
| **Jira or Linear** | T4 derived | Action-item content | Yes | Required | Required | Legal | PENDING, integration band. Exactly one is chosen; shipping both is out of scope by policy. |
| **Clerk** | T3 only | Identity, organization, role. No meeting content. | Yes, identity processor | **Required now** | Required | Legal | **PENDING and BLOCKING: Clerk is live in this band.** |
| **Temporal** | Metadata only | Workflow ids and metadata. No transcript bodies. | Processor, metadata | Required if managed | Required if managed | Legal | N/A at MVP, self-hosted |
| **Stripe** | T2 only | Billing contact, usage aggregates | Yes, billing processor | Required | Required | Legal | PENDING, billing band |
| **Google Calendar** | Inbound | Sends attachments into the platform; receives nothing of substance | Inbound source | Required | Required | Legal | PENDING, integration band |
| **Cloudflare** | In transit | Terminates TLS; sees metadata and headers, not decrypted bodies | Edge processor | Required | Required | Legal | PENDING |

## 4. What Plan 1-2 can and cannot close

**This is not an engineering task and cannot be completed by writing code.** Executing a DPA is a commercial and legal action requiring a signatory. What this band produces is the register, the classification, and the per-processor owner and status. The band's Definition of Done is satisfied by every row reading **signed** or **explicitly pending with a named owner**, which is the wording the plan uses.

Two rows deserve attention now rather than at their integration band:

**Clerk is the blocking one.** It is the only processor actually receiving data during Plan 1-2. Real user identities and organization memberships flow to it the moment anyone signs in. Its DPA should be executed before the first non-team user touches the system.

**Self-hosting removed two obligations.** Redpanda and Temporal run on our own VPS under [ADR-0016](../adr/0016-self-hosted-vps-mvp-substrate.md), so neither is a third party at MVP and neither needs an agreement today. This is a genuine benefit of the substrate decision that was not part of its stated rationale. It reverses if either moves to a managed offering, and this register is where that reversal must be recorded.

## 5. Standing obligations once signed

- The customer-facing subprocessor list is published and kept current. Adding a subprocessor requires customer notice under most enterprise contracts.
- Cross-border transfer is covered by SCCs, evaluated per processor and per tenant region. See [data-residency.md](data-residency.md).
- PII redaction runs before egress to LLM processors where `tenant_settings.pii_redaction_enabled` is set.
- All processor API keys stay server-side. Deepgram in particular is proxied precisely so its key never reaches a client.
- Sub-processing changes by a processor trigger a review of this register.

## 6. Related

- [02-system-context.md](../02-system-context.md) section 6.2 is the authoritative T4 flow analysis.
- [12-compliance-and-privacy.md](../12-compliance-and-privacy.md) for the wider compliance posture.
- [rtbf-contract.md](rtbf-contract.md): deletion obligations extend to processors holding tenant data.
