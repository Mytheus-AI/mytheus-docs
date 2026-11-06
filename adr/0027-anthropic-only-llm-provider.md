# ADR-0027: Anthropic is the only LLM provider; OpenAI is dropped; LiteLLM stays as the routing layer

The locked LLM routing decision in [_canon/06-tech-decisions.md](../_canon/06-tech-decisions.md) read
"LiteLLM (OpenAI + Anthropic)". The user has decided that Mytheus drops OpenAI entirely and uses
Anthropic only for LLM inference. This ADR records that decision and supersedes the dual-provider
wording wherever it appears.

## Related documents

- [_canon/06-tech-decisions.md](../_canon/06-tech-decisions.md) - the "LLM routing" and "Compliance" rows, updated in the same change.
- [ADR-0004](0004-llm-cost-control-pipeline.md) - tiered routing (Haiku, Sonnet, Opus) over LiteLLM. The tiering design is unchanged; only the provider set narrows.
- [ADR-0008](0008-dedicated-inference-serving.md) - embeddings and reranker are self-hosted BGE. Unaffected.
- [ADR-0009](0009-data-residency-multi-region.md) - subprocessor placement; the LLM subprocessor list is now Anthropic alone.
- [Compliance and Privacy](../12-compliance-and-privacy.md) - the subprocessor register.

---

## Status

**Accepted** (decided by the user 2026-11-06). Supersedes the "OpenAI + Anthropic" provider set in the
LLM routing row of canon. ADR-0004 and ADR-0009 keep their original context text as the record of what was
true when they were written; each carries a one-line "superseded by ADR-0027" note for the provider list.

## Context

The original plan routed LLM traffic through LiteLLM to both OpenAI and Anthropic. In practice the
tiered pipeline (pre-gate, extract, high-stakes) and the cost model in
[_canon/05-capacity-math.md](../_canon/05-capacity-math.md) section 9 are already expressed entirely in
Anthropic tiers (Haiku, Sonnet, Opus) with Anthropic prompt caching. OpenAI appeared only as an
alternative route and as a listed subprocessor.

## Options

| Option | Assessment |
|---|---|
| **Keep dual provider (OpenAI + Anthropic)** | Provider redundancy and a second price point, but two DPAs and SCC sets, two zero-retention configurations, two eval baselines for agent quality, and a cost model that was never built on OpenAI tiers. |
| **Anthropic only, LiteLLM retained (chosen)** | One vendor, one DPA, one eval baseline, and the cost model already matches. LiteLLM stays provider-agnostic, so adding a provider later is configuration, not a redesign. |
| **Anthropic only, drop LiteLLM and call the SDK directly** | Removes a hop, but loses the per-user rate limits, token tracking, and quota hooks the budget pipeline relies on ([ADR-0004](0004-llm-cost-control-pipeline.md)). Rejected. |

## Decision

**D1. Anthropic is the sole LLM provider.** OpenAI is removed from routing, the subprocessor list, secrets, and diagrams.

**D2. The fast-response tier is Haiku 5.5, model id `claude-haiku-5-5`.** This is the model behind the triage and pre-gating tier and any other latency-sensitive call.

**D3. Other tiers: Anthropic models, mapping to be set.** The user gave no mapping for the extraction and high-stakes tiers. This ADR deliberately does not assign one; the Sonnet and Opus tier names in canon and the cost model remain the working labels until the user sets the mapping.

**D4. LiteLLM stays as the routing layer**, now fronting Anthropic only, with its per-user rate limits and token tracking unchanged.

**D5. Embeddings and reranker are unaffected.** They remain self-hosted BGE ([ADR-0008](0008-dedicated-inference-serving.md)).

## Consequences

- **Negative, accepted.** Single vendor risk: an Anthropic outage, rate-limit change, price change, or policy change hits every LLM path. Mitigation is LiteLLM, which keeps a second provider a configuration change.
- **Positive.** One fewer subprocessor and DPA: the LLM subprocessor set is Anthropic alone, shrinking the compliance surface and the SCC and zero-retention paperwork.
- **Positive.** One agent-quality eval baseline and one prompt-caching behavior to tune.
- **Follow-up for the user.** Pricing in canon 05 section 9 was stated for earlier Haiku, Sonnet, and Opus models. Re-costing against Haiku 5.5 and the eventual tier mapping is needed; this ADR changes no number.
- **Follow-up for code repos.** `OPENAI_API_KEY` and the `openai` dependency should be removed from `mytheus-intelligence` configuration when that work is scheduled.

> [SaaS-scale adjustment] The PDF named a dual-provider LiteLLM setup; production narrows it to Anthropic only to reduce vendor and compliance surface while LiteLLM keeps the layer provider-agnostic.
