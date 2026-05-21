# ADR-001: Event Bus — Kafka (MSK) vs. RabbitMQ

- **Status:** Proposed (pre-MVP decision required)
- **Date:** 2026-05-21
- **Deciders:** CTO, Head of SRE, Principal Software Architect
- **Tags:** event-bus, infrastructure, scalability

## Context

Vol 01 §7 mandates an event bus to support the transactional outbox pattern, search indexing pipeline, AI job queue, audit stream, and notification dispatch. We must pick one technology before MVP because the outbox publisher and topic ACL model differ materially between options.

The candidates are Apache Kafka (delivered via Amazon MSK) and RabbitMQ.

Key requirements:
1. Per-partition ordering keyed by `tenant_id:aggregate_id` — guarantees per-aggregate event order without serializing the whole tenant.
2. At-least-once delivery with idempotent consumers.
3. Topic-level ACLs for tenant scoping (Pooled mode) and per-tier capacity.
4. Replay capability for late-arriving consumers and reindexing.
5. Multi-AZ HA with bounded recovery.
6. Compatibility with Outbox relay daemon written in Go.
7. Cost scaling acceptable from MVP through Enterprise.

## Options

### Option A — Apache Kafka via Amazon MSK
- Partition ordering is native; partitioning by `tenant_id:aggregate_id` is straightforward.
- Topic ACLs via IAM and SCRAM; per-tenant scoping fits naturally.
- Retention (1d–30d configurable) enables replay.
- Mature Go client (segmentio/kafka-go, IBM/sarama).
- MSK Serverless option reduces MVP cost.
- Higher operational complexity (zookeeper-less mode via KRaft helps).

### Option B — RabbitMQ
- Excellent fit for traditional message queues; weaker for replay/audit.
- Per-queue ordering only; sharding by tenant requires N queues.
- AMQP semantics well-understood by PHP/Node libraries.
- Lower operational complexity at small scale.
- Replay requires DLX + manual republish — clumsy.

## Decision

**Adopt Apache Kafka via Amazon MSK.** The audit stream, search-sync replay, and per-aggregate partition ordering align with Kafka's strengths. RabbitMQ would force us to fragment the event bus into many queues per tenant, complicating ACL management and replay.

**MVP cost mitigation:** start with MSK Serverless; migrate to provisioned MSK when sustained throughput justifies (likely Phase 2).

## Consequences

**Positive:**
- Native partition ordering simplifies aggregate consistency.
- Single topic per event class scales linearly with tenant count.
- Replay supports search reindex and analytics backfill.
- Kafka Connect available for analytics sinks later.

**Negative:**
- Operational overhead higher than RabbitMQ.
- Kafka client tuning (acks, idempotent producers, batch.size) requires deliberate effort.
- DLQ pattern requires explicit topic per origin.

**Follow-up:**
- Define topic naming convention and ACL scheme.
- Document producer/consumer client defaults in SDK.
- Schedule Phase 2 review on MSK Serverless vs. provisioned.

## References
- Vol 01 §7 (event-driven communication)
- Vol 06 §3.21 (outbox table)
- Vol 11 §6.4 (MSK config)
