# ADR-005: Service Mesh — Linkerd vs. Istio vs. Cilium

- **Status:** Proposed (Phase 3 decision)
- **Date:** 2026-05-21
- **Deciders:** Head of SRE, Security Lead, CTO
- **Tags:** mesh, security, observability

## Context

Vol 09 §4 requires mTLS for all service-to-service communication in the Control Plane (tenant-mgmt-svc, license-svc, ai-svc, notif-svc, search-indexer, outbox-relay, stripe-webhook-ingest). Vol 11 §3 mentions service mesh; the exact choice was deferred.

Mesh is **not deployed at MVP** (Vol 00 §3) — service-to-service mTLS will be done via cert-manager + manual sidecar for MVP. Mesh comes online when service count > 8 (likely Phase 3).

## Options

### Option A — Linkerd 2.x
- Smallest operational surface; "just works" defaults.
- Native mTLS, automatic identity issuance.
- Lower memory/CPU overhead than Istio.
- Tap + Viz dashboards.
- Fewer advanced features (no L7 retries/circuits — those done in app).

### Option B — Istio
- Most feature-rich (L7 routing, circuit breakers, fault injection, advanced auth policies).
- Complex; high learning curve.
- Heavier on resources.
- Strong ecosystem (Anthos, EKS Distro support).

### Option C — Cilium Service Mesh
- eBPF-based; sidecar-less mode option.
- Strong network policy + observability via Hubble.
- mTLS recently added.
- Younger mesh capability; some advanced features still maturing.

## Decision

**Adopt Linkerd** as default for Phase 3 when mesh is introduced. Lowest complexity matches our service count (~8 in Phase 3) and operational maturity. App-layer retries/circuit breakers already exist (Vol 01 §10.4), so Istio's L7 features are over-spec.

Reconsider at Phase 4+ if service count >25 or if advanced policy management becomes a bottleneck.

## Consequences

**Positive:**
- Minimal config; rapid time-to-mTLS.
- Lower resource footprint.
- Strong defaults reduce mistakes.

**Negative:**
- If we later need Istio's policy depth, migration is non-trivial.
- Hubble-style network observability requires separate tool.

**Follow-up:**
- Phase 3 spike: validate Linkerd against Mercato traffic patterns.
- Document mTLS rotation and observability hooks.
- Reassess Cilium maturity at Phase 4.

## References
- Vol 09 §4 (auth)
- Vol 11 §3 (topology)
- Vol 01 §5 (two-plane model)
