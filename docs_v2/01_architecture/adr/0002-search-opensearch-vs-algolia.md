# ADR-002: Search — OpenSearch self-hosted vs. Algolia

- **Status:** Proposed (pre-Phase 2 decision required)
- **Date:** 2026-05-21
- **Deciders:** CTO, Head of SRE, VP Engineering
- **Tags:** search, infrastructure, multi-tenancy, cost

## Context

Vol 01 §10.2 requires per-tenant search with sub-100ms P95 query latency at the Enterprise tier and >1M SKUs per tenant ceiling. Search is deferred from MVP (Vol 00 §2.12) but must be ready for Phase 2.

The candidates are Amazon OpenSearch Service (managed Elasticsearch fork) self-deployed per region, and Algolia (managed SaaS).

Key requirements:
1. Strict per-tenant isolation (no cross-tenant query leakage).
2. Per-tenant index sizing flexibility (a tenant with 5k SKUs shouldn't pay for capacity sized to a 5M SKU tenant).
3. Predictable cost at scale (>100 tenants).
4. Synonyms, boost rules, multilingual analyzers.
5. Faceting + filtering + geo + numeric range.
6. Operational control (we want to debug slow queries directly).
7. Fallback path: degraded mode when search is unavailable.

## Options

### Option A — OpenSearch (Amazon OpenSearch Service)
- Per-tenant index pattern; ISM lifecycle for hot/warm/cold tiers.
- Full operational control; query slow log accessible.
- Multilingual analyzers built-in.
- Pricing predictable: instance hours + storage; no per-query fee.
- Self-tuning shard sizing is non-trivial.
- Recovery from cluster red state requires runbook discipline.

### Option B — Algolia
- Excellent developer ergonomics; fast time-to-value.
- Per-record pricing creates unpredictable cost at scale.
- Multi-tenant isolation via API key + filter (per-tenant API key feasible).
- Synonym + ranking management UI strong.
- No operational visibility into the indexing pipeline.
- Vendor lock-in significant.

## Decision

**Adopt Amazon OpenSearch Service.** The per-query and per-record pricing of Algolia scales unfavourably against marketplace catalogs in the 100k–10M SKU range, and operational visibility matters for our SLO commitments. Tenant isolation via dedicated indices is cleaner than relying on Algolia API keys for isolation guarantee.

Deferred from MVP per Vol 00 §2.12 (MySQL `LIKE` fallback for MVP cap).

## Consequences

**Positive:**
- Predictable cost scaling.
- Operational visibility.
- Per-tenant index allows custom analyzers / boost rules per tenant.

**Negative:**
- Higher operational burden — needs DevOps capacity.
- Cluster red recovery procedures must be drilled.
- Index template management overhead.

**Follow-up:**
- Define index naming, ILM, snapshot policy in Vol 11.
- Test fallback to MySQL works under Phase 2 load.
- Re-evaluate at Phase 4 against new managed search offerings.

## References
- Vol 01 §10.2
- Vol 04 §15 (search functional spec)
- Vol 11 §6.5 (OpenSearch sizing)
