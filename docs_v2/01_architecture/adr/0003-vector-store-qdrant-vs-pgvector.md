# ADR-003: Vector Store — Qdrant vs. pgvector

- **Status:** Proposed (Phase 3 decision required)
- **Date:** 2026-05-21
- **Deciders:** AI Architect, Head of SRE, VP Engineering
- **Tags:** ai, vector-store, multi-tenancy

## Context

Vol 12 §6 requires a vector database for RAG-backed AI features (product description grounding, support summaries, conversational discovery). Tenant isolation is a hard requirement (Vol 09 §11).

Candidates: Qdrant (purpose-built vector DB) and PostgreSQL with pgvector extension.

Key requirements:
1. Per-tenant namespace isolation enforced by the engine.
2. Metadata filtering during ANN search (filter by tenant + category + visibility).
3. Snapshot/restore.
4. gRPC client for performance.
5. Operational maturity at multi-million-vector scale per tenant.

## Options

### Option A — Qdrant on EKS
- Purpose-built; HNSW indexing; metadata filtering native.
- Per-collection isolation; one collection per tenant.
- gRPC + REST clients.
- Self-host on EKS; snapshots to S3.
- Operational complexity moderate.

### Option B — pgvector in Aurora PostgreSQL
- Reuses existing Aurora PostgreSQL infrastructure (Control Plane).
- Native SQL filtering on metadata.
- Tenant isolation via row-level + schema.
- ANN performance lags Qdrant at >1M vectors per tenant.
- Easier ops (no new database tier).

## Decision

**Adopt Qdrant** for tenant-facing AI RAG (catalog grounding, support, discovery). Use **pgvector** for Control-Plane internal corpora (internal documentation, internal evaluation sets).

Rationale: filter-rich ANN at >1M vectors per tenant is Qdrant's design center; tenant-isolated collections fit the security model in Vol 09 §11 better than schema-level isolation in pgvector.

Deferred from MVP per Vol 00 §2.17 (no AI at MVP).

## Consequences

**Positive:**
- Best-in-class ANN performance at scale.
- Clean per-tenant boundary.
- Snapshots for backup independent of operational DB.

**Negative:**
- New infrastructure component to operate.
- Skills gap — needs runbook + on-call training.
- Migration path between vector versions requires deliberate rebuilds.

**Follow-up:**
- Phase 3 spike to validate at 5M vectors per tenant.
- Choose embedding model (text-embedding-3-small default; multimodal Phase 4).
- Plan recovery from corrupted collection (rebuild from authoritative source).

## References
- Vol 12 §6, §13
- Vol 09 §11 (tenant isolation)
