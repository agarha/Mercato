# ADR-006: Outbox Relay — Go binary vs. PHP CLI under supervisord

- **Status:** Proposed (pre-MVP — relay is MVP-required)
- **Date:** 2026-05-21
- **Deciders:** Principal Platform Engineer, Head of SRE
- **Tags:** outbox, infrastructure, MVP

## Context

Vol 01 §7.5 mandates the transactional outbox pattern. A daemon polls `wp_mercato_event_outbox` and publishes to Kafka. This is MVP-required.

Two implementation options:
1. **Go binary** — compiled, single static binary, runs in EKS as a Kubernetes Deployment.
2. **PHP CLI** — runs as a long-lived PHP script under `supervisord` or `systemd`.

Key requirements:
1. Predictable low latency (target P95 lag <2s; <10s acceptable at MVP).
2. Stable memory footprint (long-running process).
3. Resilience to broker outages with backoff.
4. Easy to deploy in EKS.
5. Reuses MVP team's skill set without onboarding cost.

## Options

### Option A — Go binary
- Compiled, low memory ceiling (~30MB).
- Native Kafka client (segmentio/kafka-go) is fast and reliable.
- Stable long-running characteristics; predictable GC.
- Easy to ship as a Docker image.
- New language for the WP team — requires onboarding.

### Option B — PHP CLI (long-lived)
- Same language as the rest of `mercato-core` — zero onboarding cost.
- Long-running PHP has known memory-leak gotchas; needs periodic recycle.
- Kafka client (RdKafka extension) is solid but PHP performance trails Go.
- Familiar deployment + observability for WP team.

### Option C — Hybrid: PHP CLI at MVP, migrate to Go at Phase 2
- MVP simplicity; can shop for Go talent during Phase 1.
- Migration risk: outbox lag SLO at MVP may force change before staffed.

## Decision

**Adopt the Go binary** for the outbox relay. The outbox relay is the single most critical infrastructure component — every event flows through it. Predictable latency and memory characteristics are worth the onboarding cost.

Outbox-relay is the only Go component for MVP; all other plugins remain PHP. The team will hire one Go-fluent engineer for SRE rotation.

## Consequences

**Positive:**
- Predictable latency under load.
- Lower resource footprint per replica.
- Easier to reason about backpressure and circuit breakers.

**Negative:**
- One additional language in the stack.
- Builds, CI, ECR images, deployment templates need duplication.

**Follow-up:**
- Recruit Go-fluent SRE pre-MVP.
- Code style + linting + test harness in repo `mercato-outbox-relay`.
- Define observability: relay lag, publish error rate, DLQ rate, broker reconnect.
- Document recovery from total relay loss (catch-up procedure).

## References
- Vol 01 §7.5
- Vol 06 §3.21
- Vol 11 §7.5 (migrations + relay deployment)
