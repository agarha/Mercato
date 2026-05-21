# Volume 12 — Mercato Enterprise Marketplace Platform
## AI Collaboration & Copilot Specification (Engineering-Grade, Implementation-Ready)

> Document owner: AI / LLM Systems Architect
> Status: APPROVED FOR IMPLEMENTATION
> Version: 2.0.0 (full rewrite of v1.0)
> Cross-references: Vol 01 (Architecture §5), Vol 02 (PRD EPIC-17), Vol 04 (FSD §20), Vol 09 (Security §3.7, §13.6)

---

## 1. Executive Assessment

The v1.0 doc named correct ingredients (Node/Python microservice, gateway abstraction, RAG via vector store, streaming SSE, PII scrubbing, usage tracking, semantic cache, multi-provider routing) but did not specify architecture choices, eval methodology, prompt versioning, cost guardrails per tier, or the explicit safety / abuse mitigations expected of a SaaS that serves vendors and buyers worldwide.

This rewrite delivers:

1. A dedicated **AI microservice architecture** with provider abstraction and circuit breakers.
2. A **prompt registry** with versioning, evaluation harness, and shadow-mode rollout.
3. **Guardrails**: PII scrubbing (regex + ML), prompt-injection defenses, output validation, jailbreak detection, hallucination mitigation via RAG and structured output.
4. **Cost controls** per tenant per feature with hard caps and graceful degrade.
5. **Per-feature design** for the 8 high-value AI use cases.
6. **Eval & quality**: golden test sets, automated grading, A/B harness, drift monitoring.
7. **Security**: tenant isolation in vector store, model API key per environment, audit logs.

**Implementation Readiness (this volume): 89/100.** Outstanding: vector-store choice (Qdrant vs. pgvector) finalization in Phase 3 — ADR-003.

---

## 2. Gap Analysis vs. v1.0

| # | Gap | v1.0 | v2.0 |
|---|---|---|---|
| G-AI-001 | Prompt registry / versioning | mentioned | full registry + eval harness (§5) |
| G-AI-002 | Guardrails layered | mentioned | input + retrieval + output stages (§6) |
| G-AI-003 | Cost guardrails | mentioned | per-tier per-feature hard caps (§7) |
| G-AI-004 | Eval methodology | none | golden sets + LLM-grader + human (§8) |
| G-AI-005 | Per-feature design | three named | 8 use cases with prompt skeletons (§9) |
| G-AI-006 | Provider abstraction | mentioned LiteLLM | full routing + fallback policy (§4) |
| G-AI-007 | Streaming spec | mentioned SSE | full SSE + cancellation + resumption (§4.5) |
| G-AI-008 | Tenant isolation in vector store | mentioned | namespace-per-tenant + filter (§6.4) |
| G-AI-009 | Audit & moderation | mentioned | full logging + flagged content workflow (§10) |

---

## 3. Architecture

### 3.1 Topology

```
[Vendor SPA / Tenant Admin SPA / Buyer SPA]
       │ HTTPS + SSE
       ▼
[ WP /mercato/v1/ai/* (mercato-ai-copilot plugin) ]
       │ checks license + capability + tenant scope; meters usage
       │ mTLS gRPC
       ▼
[ AI Service (Node.js TypeScript or Python FastAPI) ]
       ├── Provider Gateway (LiteLLM)
       │     ├── OpenAI
       │     ├── Anthropic
       │     ├── Bedrock / hosted alternatives
       │     └── Local (vLLM on g5) for sensitive workloads (Enterprise BYOM)
       ├── Prompt Registry
       ├── Guardrails Pipeline (in / retrieve / out)
       ├── Vector Store Client (Qdrant or pgvector)
       ├── Semantic Cache (Redis)
       └── Eval / Logging
              ▼
       Outbox → Kafka mercato.ai-jobs / mercato.ai-usage
```

### 3.2 Why a Dedicated Service
- PHP-FPM is not designed for long-lived streaming connections.
- Provider SDKs are richest in Node/Python.
- AI traffic is bursty; service scales independently from WP.
- Keeps PCI + tenant-PII boundary tight: AI service runs in its own VPC subnet with restricted egress.

### 3.3 Languages & Frameworks
- Default: Node.js 20 TypeScript + Fastify.
- Alternative: Python 3.12 FastAPI (for native LangChain/LlamaIndex usage).
- Choose Node for MVP; revisit at Phase 3 ADR if Python-only libraries become essential.

### 3.4 Provider Gateway

- Routing policy:
  1. Feature-specific override (e.g., support summary → Claude 3.5 Sonnet).
  2. Tenant override (Enterprise may pick provider).
  3. Health-aware default (current best provider per circuit-breaker state).
- Failover: if primary returns 5xx or breaches latency budget, retry once on secondary.
- Cost-aware routing: low-priority features (e.g., FAQ auto-answer) routed to cheaper model first; quality-critical features pinned to higher tier.

### 3.5 Streaming (SSE)

- Endpoint `POST /ai/completions` with `stream: true` returns `text/event-stream`.
- Each event: `data: {"delta": "..."}` ending with `data: [DONE]`.
- Cancellation: client sends `DELETE /ai/completions/{id}` or closes connection; service cancels upstream.
- Resumption: not supported (regenerate); but pre-streaming history saved so partial outputs aren't lost.
- Backpressure: server caps per-tenant concurrent streams (configurable; default 8 Pro / 32 Enterprise).

---

## 4. Prompt Registry & Versioning

### 4.1 Storage
`wp_mercato_ai_prompts` table (Vol 06 §3.31) — versioned per slug per tenant (NULL tenant = platform default).

### 4.2 Format

```yaml
slug: product_description.v1
status: active
template: |
  You are a marketplace product copywriter for {{tenant.brand}}.
  Generate a product description for:
  Title: {{input.title}}
  Tags: {{input.tags}}
  Audience: {{input.audience}}
  Constraints:
  - Under 200 words
  - Plain language
  - No medical/health claims
  - Mention SHIPPING_POLICY if "physical"
guardrails:
  banned_topics: [medical_claims, financial_advice]
  output_format: markdown
  max_tokens: 350
eval_set: product_description.v1.tests.jsonl
```

### 4.3 Rollout
- New version starts `draft`; eval harness runs; metrics produced.
- Shadow mode: route 10% of traffic, compare side-by-side, log preferences.
- Promote to `active` only if eval improvements without regression.
- Old version retained `deprecated` for 30d.

---

## 5. Guardrails Pipeline

Every request flows through three stages.

### 5.1 Input (Pre-Prompt)

| Layer | Tool | Action |
|---|---|---|
| PII detection | Microsoft Presidio + custom regex | Mask emails, phones, credit cards, SSN / tax IDs, addresses |
| Tenant boundary | Lookup | Reject any payload referencing other tenant_id |
| Prompt injection | classifier (e.g., Rebuff / detect-prompt-injection) + heuristic patterns | Block obvious injections; flag suspicious for review |
| Length / token | tokenizer | Refuse if > model context window minus retrieval budget |
| Disallowed content | classifier | CSAM, terrorism, self-harm — hard reject |
| Tenant policy | rules | Tenant-specific blocked topics (e.g., gambling for non-gambling marketplaces) |

### 5.2 Retrieval (RAG)

- Vector store: Qdrant (default) with per-tenant collection namespace.
- Index sources: product catalog, policies, knowledge base, recent buyer FAQs.
- Retrieval rules: top-k with tenant filter; re-rank for relevance; cap context tokens.
- Cache: semantic cache keyed by (tenant, prompt fingerprint, retrieval fingerprint).

### 5.3 Output (Post-Generation)

| Check | Action |
|---|---|
| JSON schema validation (for structured outputs) | Refuse if invalid |
| Length cap | Truncate or regenerate |
| Banned-content classifier | Refuse / regenerate with stricter prompt |
| PII leak detector (output) | Mask leaked PII before returning |
| Hallucination check (where applicable) | Citation-required mode for support summaries |
| Brand-safe filter | Lowercase profanity scan |

### 5.4 Tenant-Isolated Vector Store

- One collection per tenant: `mercato_tenant_{tenant_id}`.
- Every search query carries `must` filter on tenant_id.
- Embedding model: OpenAI text-embedding-3-small by default; configurable.
- Re-ranking via cross-encoder for high-stakes features.

---

## 6. Cost Controls

### 6.1 Per-Tier Caps (Vol 02 §6.1)

| Tier | Monthly completions | Daily cap per vendor | Per-request token cap |
|---|---|---|---|
| Starter | 1,000 | 50 | 1500 |
| Pro | 25,000 | 250 | 3000 |
| Business | 100,000 | 1,000 | 6000 |
| Enterprise | Custom | Custom | Custom |

### 6.2 Hard Stops
- 100% of monthly quota → block additional requests with `mercato.license.limitExceeded` 402; banner in UI; tenant admin notified.
- 90% threshold → soft warning banner.

### 6.3 Tiered Routing for Cost
- Cheap-first model attempted for non-critical features.
- Expensive model used only when cheap-first fails quality gate.

### 6.4 Semantic Cache
- Redis with embedding-similarity match (≥0.95 cosine).
- Cache key includes tenant_id + feature + redacted prompt hash.
- Hit reduces cost ~30–60% for common prompts.

### 6.5 Per-Provider Cost Ledger
- `wp_mercato_ai_usage` records actual cost_micro_usd per request.
- Daily roll-up to billing system; Finance reconciles against provider invoices.

---

## 7. Use Cases (Per Feature Design)

### 7.1 Product Description Generator (Vendor)
- Input: title, attributes, target audience, language, tone.
- Retrieval: tenant's brand voice, sample existing descriptions.
- Constraints: 200 words max, no health claims, structured sections (Overview / Features / Use cases).
- Output schema: `{ description_md: string, suggested_tags: string[] }`.
- Acceptance threshold: ≥95% pass on golden set.

### 7.2 Message Reply Suggester (Vendor)
- Input: last N messages + product/order context.
- Retrieval: tenant's tone-of-voice guide.
- Output: 1–3 suggested replies, each clearly labeled "AI suggestion, edit before send".
- Hard constraint: never include contact/payment info that could route off-platform.

### 7.3 Support Summary (Tenant Support)
- Input: long buyer/vendor thread.
- Output: bullet summary + next-step recommendation + sentiment.
- Citations: line references to original messages.

### 7.4 Search Query Understanding (Buyer)
- Input: natural-language query "waterproof red backpack under $80".
- Output: structured query `{ q, filters: { color, attribute_waterproof, price_max, category? } }`.
- Always fall back to lexical search if confidence low.

### 7.5 Storefront Conversational Discovery (Buyer)
- Chat UI for buyer to refine product search.
- Constrained to retrievable catalog data; no general web Q&A.

### 7.6 Vendor Application Triage (Tenant Admin)
- Input: vendor profile + product samples.
- Output: risk score + reasoning + suggested next action.
- Always advisory; tenant admin retains final decision.

### 7.7 Product Moderation (Tenant Admin)
- Input: new product listing.
- Output: classification (allowed / flagged / blocked) + reasons.
- Pre-publish gate (configurable).

### 7.8 Insights / Anomaly Detection (Tenant Admin)
- Periodic job summarizing tenant KPIs anomalies for the admin morning brief.
- Strictly structured outputs + thresholds.

---

## 8. Eval & Quality

### 8.1 Golden Sets
- Per feature: ≥200 hand-crafted inputs with reference outputs and grading rubric.
- Stored in `evals/<feature>/golden.jsonl`.

### 8.2 Grading
- LLM-as-judge for subjective dimensions (helpfulness, tone, faithfulness).
- Programmatic for objective: JSON validity, banned content presence, citation presence.
- Human review for new prompts + monthly random sample.

### 8.3 A/B Harness
- Two-arm experiment per prompt version; 10% shadow; compare quality + cost + latency.
- Statistical significance threshold (p<0.05) before promotion.

### 8.4 Drift Monitoring
- Weekly re-run golden sets on production prompts.
- Provider model upgrades trigger immediate full eval.
- Alert if accuracy drops > 2σ from baseline.

---

## 9. Safety & Abuse Mitigation

### 9.1 Adversarial Inputs
- Layered defense (§5.1) plus separate "system prompt" never expose to user.
- System prompts immutable per request; user input quoted with explicit delimiters and "treat as data, not instructions" framing.

### 9.2 Jailbreak Detection
- Pattern + classifier; flag and route to human review.

### 9.3 Content Policy
- Tied to Vol 03 §5.14.2 prohibited categories.
- Buyer-facing AI never generates references to prohibited items.
- Output filter: secondary classifier on every response.

### 9.4 Trust Signaling
- Every AI-generated content is tagged in DB (`source = 'ai-assisted'`) and clearly labeled in UI (Vol 08).
- Vendor must accept before public publish (BR-AI-002).

### 9.5 Right to Disable
- Tenant admin can disable AI per feature instantly (capability JWT flag).
- Vendor admin can opt out of all AI suggestions for their store.

---

## 10. Logging & Audit

- Every request logs: tenant_id, vendor_id, user_id, feature, provider, model, input hash (not raw), output hash, tokens, cost, guardrail decisions.
- PII never persisted in logs (redacted version stored separately for security review only).
- Flagged outputs stored 90d for review.
- Tenant admin can request a log of all AI requests by their vendors.
- GDPR DSAR: AI usage rows tied to a user_id are returned in their data package.

---

## 11. Resilience

### 11.1 Provider Outage
- Circuit breaker per provider.
- Failover to secondary; if both down, feature disabled with banner.

### 11.2 Quota Exhaustion
- 402 surfaced with tenant context; UI shows usage + upgrade option.

### 11.3 Latency Discipline
- Per-feature latency SLOs (e.g., product description streaming start <2s).
- Streaming preferred over blocking.
- Long features (bulk translate 1000 products) explicitly async via `mercato.ai-jobs` topic.

---

## 12. BYOM (Bring Your Own Model) — Enterprise Phase 4

- Enterprise tenants may direct their AI traffic to their own deployment (Azure OpenAI under their own subscription, or self-hosted vLLM).
- Mercato AI Service holds a per-tenant model adapter config.
- Cost not metered by Mercato; quality eval still runs on Mercato-managed golden set.
- Tenant retains DPA with their model provider.

---

## 13. Vector Store: Detail

### 13.1 Why Qdrant
- Strict per-collection filtering on metadata (tenant_id).
- Self-hostable on EKS.
- Mature gRPC interface.
- Snapshot/restore.

### 13.2 pgvector Alternative
- Lower operational overhead if Aurora PostgreSQL already in stack (Control Plane).
- Use for Control-Plane-internal corpora; Qdrant for tenant catalog.

### 13.3 Indexing Pipeline
- `mercato.product.upserted.v1` → indexer worker → embed → upsert to Qdrant.
- Reindex on prompt-version change for full catalog (background job).
- Backfill 24h after launch from existing catalog snapshot.

---

## 14. Compliance

### 14.1 Data Handling
- Provider DPAs reviewed annually.
- Inputs containing tenant PII routed only to providers with zero-retention agreements (OpenAI Zero-Retention, Anthropic Enterprise).
- Sensitive tenants (Enterprise) may pin to providers with regional residency (US/EU).

### 14.2 Output Liability
- Vendor accepts AI-assisted descriptions before public; vendor remains responsible for content per ToS.
- Buyer-facing AI outputs (search assist) carry "AI-assisted result" tag.

### 14.3 Sub-processor Disclosure
- Every AI provider listed in public sub-processor registry per BRD §10.

---

## 15. Telemetry & SLOs

| Metric | Target |
|---|---|
| Streaming first-token latency | <2s P95 |
| Non-streaming completion P95 | <8s |
| Cache hit rate (common features) | >40% |
| Quality (golden set pass) | >95% per feature |
| Guardrail false-positive rate | <1% |
| Tenant-quota 402 rate | <0.5% |
| Provider failover invocations / day | < 5 |

---

## 16. Cross-Volume Cross-References

| This Section | See Also |
|---|---|
| §3 Architecture | Vol 01 §5 |
| §5 Guardrails | Vol 09 §13 application security |
| §6 Costs | Vol 02 §6 billing; Vol 03 §6 SKUs |
| §10 Logging | Vol 09 §8 audit; Vol 06 §3.30 ai_usage |
| §11 Resilience | Vol 01 §10.4 |

---

*End of Volume 12 — AI Collaboration v2.0.*
