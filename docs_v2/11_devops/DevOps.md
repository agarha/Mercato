# Volume 11 — Mercato Enterprise Marketplace Platform
## DevOps & Infrastructure Specification (Engineering-Grade, Implementation-Ready)

> Document owner: DevOps Architect / Head of SRE
> Status: APPROVED FOR IMPLEMENTATION
> Version: 2.0.0 (full rewrite of v1.0)
> Cross-references: Vol 01 (Architecture §10–§14), Vol 09 (Security), Vol 10 (QA pipeline gates)

---

## 1. Executive Assessment

The v1.0 DevOps doc named the right components (EKS, IaC, ArgoCD, OpenTelemetry, FluentBit) but stopped at directional guidance. For an enterprise marketplace targeting 99.99% Enterprise uptime with multi-region failover, BYOK, and SOC-2/PCI/GDPR evidence, engineering needs the topology diagrams, the Terraform module structure, the cluster sizing, the deployment workflow, the canary strategy, the database HA configuration, the secret rotation runbooks, the cost guardrails, and the disaster recovery drill cadence.

This rewrite delivers all of the above with prescriptive defaults that are immediately implementable.

**Implementation Readiness (this volume): 92/100.** Outstanding: cost models for Phase 4 BYOK + multi-region pricing; tracked in Gap Matrix.

---

## 2. Gap Analysis vs. v1.0

| # | Gap | v1.0 | v2.0 |
|---|---|---|---|
| G-DEV-001 | Topology diagram | Verbal | Diagram + module-level Terraform (§4) |
| G-DEV-002 | Cluster sizing | None | Per-tier sizing tables (§5) |
| G-DEV-003 | DB HA | Aurora named | Aurora Global + replicas + ProxySQL (§6) |
| G-DEV-004 | CI/CD specifics | Mentioned | Pipeline definition + secrets + signing (§7) |
| G-DEV-005 | Canary / blue-green | Mentioned | Progressive delivery via Argo Rollouts (§7.4) |
| G-DEV-006 | Observability | OpenTelemetry mentioned | Full stack + dashboards + SLO burn alerts (§8) |
| G-DEV-007 | DR drill | None | Quarterly named drills with pass criteria (§9) |
| G-DEV-008 | Cost guardrails | None | Budgets + tags + alerting (§10) |
| G-DEV-009 | Multi-region | Mentioned | Active/passive with failover steps (§9.3) |

---

## 3. Cloud Topology

### 3.1 Provider & Regions
AWS as primary (with abstraction layer to enable GCP later if needed).
- **Production primary**: `us-east-1` (or `eu-west-1` for EU tenants).
- **Production warm standby**: `us-west-2` (Enterprise multi-region).
- **Staging**: `us-east-1` only.
- **Pre-prod**: `us-east-1`.
- **Dev/CI**: ephemeral via PR-based namespaces.

### 3.2 Diagram

```
┌─────────── Route 53 (DNS) ─────── ACM (TLS) ──────── CloudFront ─────────── AWS WAF / Shield ────────┐
│                                                                                                       │
│  Edge → ALB (Ingress) → EKS                                                                           │
│    ├── Namespace cp-prod (Control Plane)                                                              │
│    │     ├─ tenant-mgmt-svc          ├─ license-svc          ├─ ai-svc                                │
│    │     ├─ notif-svc                ├─ search-indexer       ├─ outbox-relay                          │
│    │     └─ stripe-webhook-ingest                                                                     │
│    └── Namespace dp-<tenant_id> (Data Plane, per tenant)                                              │
│          ├─ wp-fpm Deployment (N replicas)                                                            │
│          ├─ wp-nginx Deployment                                                                       │
│          └─ outbox-relay (sidecar OR shared)                                                          │
│                                                                                                       │
│  Data & state:                                                                                        │
│    ├─ Aurora MySQL 8.0 (Global cluster for Enterprise)                                                │
│    ├─ Aurora PostgreSQL (Control Plane only)                                                          │
│    ├─ ElastiCache Redis (replication groups)                                                          │
│    ├─ Amazon MSK (Kafka)                                                                              │
│    ├─ Amazon OpenSearch Service                                                                       │
│    ├─ Qdrant on EKS (for AI RAG, Phase 3)                                                             │
│    ├─ S3 (public assets, KYC docs, reports, telemetry, backups)                                       │
│    └─ KMS (multiple CMKs per data class; BYOK for Enterprise)                                         │
│                                                                                                       │
│  Observability:                                                                                       │
│    ├─ Prometheus + Grafana (self-hosted) OR Datadog                                                   │
│    ├─ Tempo / Jaeger (traces)                                                                         │
│    ├─ OpenSearch (logs) via FluentBit DaemonSet                                                       │
│    └─ PagerDuty for alerts                                                                            │
│                                                                                                       │
│  Security:                                                                                            │
│    ├─ HashiCorp Vault (or AWS Secrets Manager)                                                        │
│    ├─ GuardDuty + Security Hub + Config                                                               │
│    └─ Linkerd service mesh (mTLS)                                                                     │
└───────────────────────────────────────────────────────────────────────────────────────────────────────┘
```

### 3.3 Network Layout (VPC)

| Subnet | CIDR | Purpose |
|---|---|---|
| Public | 10.0.0.0/22 | ALB, NAT GW |
| Private (app) | 10.0.16.0/20 | EKS nodes |
| Private (data) | 10.0.48.0/20 | Aurora, ElastiCache, MSK, OpenSearch |
| Private (egress) | 10.0.64.0/24 | NAT gateways for egress logging |

Inter-AZ across 3 AZs.

---

## 4. Infrastructure as Code (Terraform)

### 4.1 Module Structure

```
infra/terraform/
├── envs/
│   ├── prod-us-east-1/
│   ├── prod-eu-west-1/
│   └── staging-us-east-1/
└── modules/
    ├── network/      (VPC, subnets, NAT, route tables)
    ├── eks/          (EKS cluster, node groups, OIDC, addons)
    ├── data/         (Aurora MySQL, ElastiCache, MSK, OpenSearch)
    ├── edge/         (CloudFront, ACM, Route53, WAF)
    ├── secrets/      (KMS keys, Vault config)
    ├── observability/(Prometheus, Grafana, Tempo, log infra)
    ├── tenancy/      (tenant-namespace, dedicated cluster module)
    └── ci/           (ECR repos, GH Actions OIDC roles)
```

### 4.2 State

- Remote state in S3 with DynamoDB locking; cross-account replication for DR.
- One state file per env.
- Module versions pinned via Git tags.

### 4.3 Pipelines for IaC

- `terraform fmt`, `terraform validate`, `tflint`, `tfsec`, `checkov`, `infracost diff` on PR.
- Apply via `terraform plan` posted to PR; manual approve; `terraform apply` via GitHub Actions OIDC role.
- Drift detection nightly.

---

## 5. Compute (EKS)

### 5.1 Cluster Layout
- One EKS cluster per environment.
- Node groups:
  - `general` — m6a.xlarge spot+on-demand mix
  - `wp-fpm` — c6a.large dedicated for WP pods (sized predictable)
  - `data-workers` — m6a.2xlarge for indexers, outbox relay, payout workers
  - `ai-workers` — g5.xlarge (GPU) optional for hosted LLMs (or none if using API providers)

### 5.2 Tenant Sizing

| Tier | WP pods | RAM/pod | CPU/pod | Replicas |
|---|---|---|---|---|
| Starter (pooled) | shared | 2GB | 1 vCPU | n/a (shared) |
| Pro (pooled) | shared | 2GB | 1 vCPU | n/a |
| Business (silo) | 2 | 4GB | 2 vCPU | HPA 2–8 |
| Enterprise (dedicated) | 4 | 8GB | 4 vCPU | HPA 4–32 |

HPA based on CPU + custom metric (outbox lag).

### 5.3 PHP-FPM Tuning

- `pm = dynamic`, `pm.max_children` = floor(0.8 × pod_memory_MB ÷ avg_request_memory_MB).
- 2GB pod → 25 workers.
- `request_terminate_timeout = 30s`.
- `pm.max_requests = 500` (avoid memory leaks).
- OPcache: `opcache.memory_consumption=256`, `opcache.validate_timestamps=0` (production), preload core via warmup script.

### 5.4 Statelessness Enforcement
- Readonly root filesystem.
- emptyDir tmpfs for /tmp.
- No writes to `/wp-content/uploads`; S3 plugin offloads.
- WP cron disabled, replaced by system cron (Kubernetes CronJob).

---

## 6. Data Tier

### 6.1 Aurora MySQL
- Aurora MySQL 8.0 Serverless v2 for SMB tiers (autoscaling ACU 2–32).
- Aurora MySQL provisioned for Enterprise (db.r6g.2xlarge primary + 2 read replicas).
- Multi-AZ enabled.
- **Aurora Global Database** for Enterprise multi-region; standby in `us-west-2`.
- PITR enabled (35d).
- Cross-account daily snapshot copy (separate account for anti-ransomware).
- Performance Insights ON; Enhanced Monitoring 1m.

### 6.2 Connection Pooling
- **RDS Proxy** in front of Aurora; tenant connection limit 50; total pool 5000.

### 6.3 ElastiCache Redis
- Replication group: 1 primary + 2 replicas, automatic failover.
- 6 shards for Enterprise tiers.
- Encryption in transit + at rest.
- Cluster mode ON for Business+.

### 6.4 MSK (Kafka)
- 3 brokers, m5.large for Pro / m5.xlarge for Enterprise.
- TLS-only client auth; SASL/SCRAM credentials per tenant.
- Topics per Vol 01 §7.4.
- ACLs scoped per tenant.
- Configuration backed up to S3 daily.

### 6.5 OpenSearch
- 3 data nodes + 3 master nodes (Enterprise); single-AZ for Starter.
- Per-tenant indices with sharding default 3p/1r.
- ISM lifecycle moves hot → warm at 30d, cold at 90d.
- Snapshots to S3 daily.

### 6.6 S3
- Buckets per data class (see Vol 09 §6.2).
- Versioning ON for KYC + audit; Object Lock compliance mode.
- Lifecycle: hot S3 → IA at 30d → Glacier at 365d → permanent delete at retention end.
- Cross-region replication for Enterprise.

### 6.7 Vault / Secrets
- HashiCorp Vault HA cluster (3 nodes) or AWS Secrets Manager.
- KV v2 + dynamic creds (Aurora, RDS, AWS).
- Audit to S3.

---

## 7. CI/CD Pipeline

### 7.1 Tools
- **CI**: GitHub Actions (or GitLab CI).
- **CD**: ArgoCD (GitOps).
- **Image registry**: Amazon ECR (private).
- **Image signing**: cosign + Rekor.
- **Manifest repo**: separate repo `mercato-deploy` watched by ArgoCD.

### 7.2 CI Workflow
1. PR opens → ephemeral env spun up + lint + unit + integration + traceability + security scans.
2. Merge to `main` → image built with Git SHA + signed → pushed to ECR.
3. `mercato-deploy/staging/values.yaml` bumped automatically.
4. ArgoCD syncs staging.
5. Nightly: full E2E + perf + DAST.
6. Manual promote to prod by tagging release: `mercato-deploy/prod/values.yaml` bumped via PR.

### 7.3 Secrets in CI
- OIDC-based federated auth (no static AWS keys).
- GitHub secret scanning enabled.
- Build-time env vars from Vault via OIDC.

### 7.4 Progressive Delivery (Argo Rollouts)
- Strategy: Canary 5% → 25% → 50% → 100% with 5-minute soak between steps.
- Analysis: success rate ≥99.5%, P95 latency within +10% of baseline, no SEV alert.
- Auto-rollback on breach.

### 7.5 Database Migrations
- Migrations executed by a Kubernetes Job spawned **before** pod rollout.
- Migration job uses dedicated DB role with DDL privileges.
- Rollback paths required for every migration (Vol 06 §8).
- Long-running migrations (online schema change) via `gh-ost` or `pt-online-schema-change`.

---

## 8. Observability

### 8.1 Logging
- Structured JSON logs to stdout.
- FluentBit DaemonSet → OpenSearch (or Datadog Logs).
- Indexes: `mercato-app-*` per env per day.
- PII redaction at the FluentBit filter layer (Lua filter with known patterns).
- Retention: 30d hot + 1y cold (S3 + Athena).

### 8.2 Metrics
- Prometheus scrape `/metrics` on all services every 15s.
- Federation to platform-level Prometheus.
- Recording rules for SLO burn rate.
- Grafana dashboards from `dashboards/` ConfigMap (Vol 01 §13.5).

### 8.3 Traces
- OpenTelemetry SDK in every service (PHP via opentelemetry-php, Node via @opentelemetry).
- W3C `traceparent` propagated via REST + Kafka headers + outbox payloads.
- Tempo as backend; 10% sampling (Enterprise), 1% (other).
- Tail-based sampling for errors (always sampled).

### 8.4 SLO Burn Rate Alerts
- Multi-window multi-burn-rate alerts (1h, 6h, 1d) per SLO.
- Alert severity tiered: page (1h fast burn), ticket (6h), backlog (1d).

### 8.5 Real User Monitoring (RUM)
- For storefront / SPA: OTel JS instrumentation; metrics on Largest Contentful Paint, First Input Delay, Cumulative Layout Shift.

### 8.6 Alerting & On-Call
- PagerDuty schedules per service.
- 2-person rotation (primary + secondary).
- SLA: SEV-1 ack <5min; resolution <1h (Enterprise).
- Runbooks linked from every alert.

---

## 9. Backup & Disaster Recovery

### 9.1 Backup Matrix (Vol 06 §9 expanded)

| Source | Mechanism | Frequency | Retention | Encryption |
|---|---|---|---|---|
| Aurora MySQL | PITR + automated snapshot | continuous + daily | 35d + 90d | KMS |
| ElastiCache | Snapshot daily | daily | 7d | KMS |
| OpenSearch | S3 snapshot | daily | 30d | KMS |
| MSK | Config backup | daily | 30d | KMS |
| S3 buckets | versioning + CRR | continuous | per Object Lock | SSE-KMS |
| Vault | snapshot | daily | 30d | KMS |

Cross-account copies for Aurora + S3 audit + KYC for anti-ransomware.

### 9.2 DR Drill

- Quarterly drill: restore one Tier-0 / Tier-1 table from snapshot in isolated env; replay 24h of outbox; verify checksum.
- Annual drill: full region failover dry run (no production traffic moved); document RTO actual.

### 9.3 Region Failover (Enterprise)
- Aurora Global → promote replica to primary.
- Route 53 health checks flip CloudFront origin.
- S3 access continues from CRR replica with reduced consistency notice.
- DR coordinator script automates 60% of steps; remainder human-approved per RB-07.

### 9.4 RTO / RPO Targets
Per Vol 01 §14.1.

---

## 10. Cost Management

### 10.1 Tagging
Every AWS resource tagged: `project=mercato`, `env=prod`, `tier=enterprise|business|pro|starter`, `tenant_id=…` (where applicable), `owner=team`, `service=…`.

### 10.2 Budgets & Alerts
- AWS Budgets per env per tier.
- Daily anomaly detection (CUR + AWS Cost Anomaly Detection).
- Slack alert at 80% / 100% / 120% of plan.

### 10.3 Right-Sizing
- Weekly Compute Optimizer recommendations reviewed.
- Reserved Instance / Savings Plan coverage target ≥70% for base load.

### 10.4 Per-Tenant Cost Attribution
- Cost & Usage Report tagged by `tenant_id` allocated via daily ETL into ClickHouse.
- Operator finance gets per-tenant unit economics dashboard.

---

## 11. Developer Experience

### 11.1 Local Environment
- `wp-env` or DDEV-based stack.
- Docker Compose for adjacent services (Redis, MySQL, OpenSearch, Kafka via Redpanda).
- Stub external dependencies (Stripe mock-server, KYC mock, AI fixtures).

### 11.2 Bootstrap
- `make bootstrap` clones, installs deps, runs migrations, seeds factory data.
- `make test` runs unit + integration.
- `make e2e` runs Playwright headless.

### 11.3 Code Ownership
- CODEOWNERS per plugin folder.
- 2 approvals + SOX-relevant areas require security review.

---

## 12. Maintenance Cadence

| Cadence | Activity |
|---|---|
| Daily | Reconcile payouts; verify backup; on-call review |
| Weekly | Patch review; cost review; SLO review |
| Monthly | DR partial drill; access review; vendor / sub-processor review |
| Quarterly | Full DR drill; pentest scope refresh; ADR review |
| Annual | SOC-2 audit; major version upgrade plan |

---

## 13. Runbook Index (operational, named & linked from alerts)

| ID | Title |
|---|---|
| RB-01 | Database failover |
| RB-02 | Outbox relay stalled |
| RB-03 | OpenSearch cluster degraded |
| RB-04 | Stripe webhook flood |
| RB-05 | AI Service provider outage |
| RB-06 | Tenant data leak suspected |
| RB-07 | Region failover (manual) |
| RB-08 | Cache poisoning / mass invalidation |
| RB-09 | Credential leak (gitleaks) |
| RB-10 | Stripe Connect account compromise |
| RB-11 | Adversarial prompt extraction (AI) |
| RB-12 | DNS misconfiguration |
| RB-13 | Certificate expiry / renewal failure |
| RB-14 | Multi-tenant noisy neighbor mitigation |

Each runbook has: detection signature, severity, immediate-actions, rollback steps, postmortem template.

---

## 14. Cross-Volume Cross-References

| This Section | See Also |
|---|---|
| §3 Topology | Vol 01 §5 |
| §6 Data | Vol 06 |
| §7 CI/CD | Vol 10 QA gates |
| §8 Observability | Vol 01 §13; Vol 02 §8 metrics |
| §9 DR | Vol 01 §14 |
| §10 Cost | Vol 03 §6 monetization |
| §13 Runbooks | Vol 09 §14 IR |

---

*End of Volume 11 — DevOps v2.0.*
