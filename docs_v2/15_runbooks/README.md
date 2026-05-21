# Volume 15 — Operational Runbooks

> Document owner: Head of SRE + CS + Compliance
> Status: Implementation Baseline v1.0
> Version: 1.0.0

This volume collects operational runbooks — step-by-step procedures for SRE, CS, and Compliance teams handling operational events.

## Index

| ID | Title | Source | Status |
|---|---|---|---|
| RB-15.1 | [Tenant Offboarding](Tenant_Offboarding.md) | enterprise sales requirement | ✅ v1.0 |
| RB-01 | Database failover | (referenced from Vol 01 §14.4) | TBD pre-MVP |
| RB-02 | Outbox relay stalled | (referenced from Vol 01 §14.4) | TBD pre-MVP |
| RB-03 | OpenSearch cluster degraded | (referenced from Vol 01 §14.4) | TBD Phase 2 |
| RB-04 | Stripe webhook flood | (referenced from Vol 01 §14.4) | TBD pre-MVP |
| RB-05 | AI Service provider outage | (referenced from Vol 01 §14.4) | TBD Phase 3 |
| RB-06 | Tenant data leak suspected | (referenced from Vol 01 §14.4) | TBD pre-MVP |
| RB-07 | Region failover (manual) | (referenced from Vol 01 §14.4) | TBD Phase 4 |
| RB-08 | Cache poisoning / mass invalidation | (referenced from Vol 01 §14.4) | TBD Phase 2 |
| RB-09 | Credential leak (gitleaks) | (referenced from Vol 09 §14.4) | TBD pre-MVP |
| RB-10 | Stripe Connect account compromise | (referenced from Vol 09 §14.4) | TBD pre-MVP |
| RB-11 | Adversarial prompt extraction (AI) | (referenced from Vol 09 §14.4) | TBD Phase 3 |
| RB-12 | DNS misconfiguration | (referenced from Vol 11 §13) | TBD pre-MVP |
| RB-13 | Certificate expiry / renewal failure | (referenced from Vol 11 §13) | TBD pre-MVP |
| RB-14 | Multi-tenant noisy neighbor mitigation | (referenced from Vol 11 §13) | TBD Phase 2 |

## Runbook Template

Every runbook follows the same structure:

```markdown
# RB-XX.Y Title

## 0. Purpose
What event this runbook covers; who runs it.

## 1. Detection
How we know this is happening (alert name, log pattern, customer report).

## 2. Severity
SEV-1/2/3 classification and on-call response time.

## 3. Immediate Actions (within 5 min)
Stop the bleeding.

## 4. Diagnosis
Discover root cause.

## 5. Mitigation
Containment + workaround.

## 6. Resolution
Permanent fix or rollback.

## 7. Rollback
If mitigation fails.

## 8. Comms
Who to notify and when (internal + status page + tenant).

## 9. Postmortem
Blameless review within 5 business days.

## 10. References
Linked dashboards, dashboards, sometimes Slack channels.
```

## Authoring guidance

- Runbooks are tested via **game days** (Vol 10 §8).
- Each runbook has a named owner (rotates annually).
- Last-tested-date on every runbook; runbooks not tested in 6 months are flagged stale.
- Runbooks are linked from PagerDuty alerts so on-call sees them immediately.
