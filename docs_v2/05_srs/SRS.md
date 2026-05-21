# Volume 05 — Mercato Enterprise Marketplace Platform
## Software Requirements Specification (Engineering-Grade, Implementation-Ready)

> Document owner: Principal Software Architect + QA Architect
> Status: Implementation Baseline v2.0 — Approved for MVP planning; ADRs pending
> Version: 2.0.0 (full rewrite of v1.0)
> Cross-references: Vol 01–04, Vol 06–11, ISO/IEC/IEEE 29148:2018

---

## 1. Executive Assessment

The v1.0 SRS approached enterprise-grade structure (NFR IDs, MUST/SHOULD wording) but stayed thin. It enumerated <30 requirements where the system has ~280 functional + ~70 non-functional requirements that must be testable, traceable, and unambiguous.

This rewrite produces:

1. A **complete requirements catalog** with stable IDs: `FR-<plugin>-<n>` mapped 1:1 to Vol 04 FSD; `NFR-<class>-<n>` for non-functional; `IR-<n>` for interface requirements; `DR-<n>` for data; `CR-<n>` for constraints.
2. A **traceability matrix** linking every requirement to (a) source PRD story, (b) FSD section, (c) DDL table(s), (d) API endpoint(s), (e) test case ID(s).
3. **Quantified NFRs** with measurement methodology and acceptance threshold.
4. **External interface requirements** for every third-party integration.
5. **System assumptions, constraints, dependencies** that bound implementation choices.

**Implementation Readiness (this volume): 92/100.** Outstanding: a small set of NFR thresholds for Phase 4 features awaiting capacity benchmarking. Tracked in Gap Matrix.

---

## 2. Gap Analysis vs. v1.0

| # | Gap | v1.0 | v2.0 |
|---|---|---|---|
| G-SRS-001 | Requirement IDs ad-hoc | Some FR-X | Stable `FR-<plugin>-<n>` matching FSD; ~280 FRs |
| G-SRS-002 | NFR thresholds not measurement-ready | Stated, no method | Each NFR has measurement methodology, tool, threshold |
| G-SRS-003 | Interface requirements absent | Mentioned APIs | Full IR catalog for Stripe, KYC, tax, email, AI providers |
| G-SRS-004 | Data requirements unstated | None | DR catalog covering retention, residency, classification |
| G-SRS-005 | Constraints / assumptions absent | None | Explicit C / A / D registries (§7) |
| G-SRS-006 | Traceability promised, not delivered | None | Automated CI builds traceability.xlsx (§10) |

---

## 3. Document Scope

This SRS specifies the software requirements for the Mercato Enterprise Marketplace Platform v1.0–v4.0 across the WordPress data plane, the SaaS control plane, supporting microservices, and external integrations.

### 3.1 Definitions
- **Tenant** — a marketplace operator who has subscribed to a plan.
- **Vendor** — a seller operating within a tenant marketplace.
- **Buyer** — end customer.
- **Sub-order** — vendor-specific portion of a parent WooCommerce order.
- **Ledger row** — append-only record in commission, payout, or audit tables.
- **Capability JWT** — short-lived signed token enumerating tenant feature flags & limits.

---

## 4. Functional Requirements

The full FR catalog (~280 items) lives in `docs_v2/04_fsd/FSD.md` §4–§22 with one FR per implementation behavior. This SRS section asserts the catalog as canonical and references it. The mapping is 1:1.

Sample (full catalog in FSD; reproduced here are the top-10 most safety-critical):

| ID | Statement | Verification Method |
|---|---|---|
| FR-CORE-001 | System MUST refuse to boot if PHP < 8.2 / WP < 6.4 / WC < 8.0 / HPOS disabled | Boot-time test in CI |
| FR-CORE-006 | Capability JWT MUST be validated on every `/wp-json/mercato/v1/*` request | Integration test per endpoint |
| FR-VEN-006 | Vendor cannot publish products while status not in {Active} | Unit + integration |
| FR-ORD-001 | On `woocommerce_new_order` system MUST split cart items by vendor | E2E test multi-vendor cart |
| FR-COMM-005 | Refund reversal MUST create offsetting ledger row; never edit original | Integration with seeded ledger |
| FR-COMM-006 | Commission ledger insert idempotent on event replay | Replay attack test |
| FR-PAY-007 | Daily reconciliation must agree with Stripe Treasury within ±$0.01 | Daily job + alert |
| FR-KYC-002 | Documents stored in private S3 with KMS encryption per-tenant key | Static analysis + integration test |
| FR-NOT-001 | Critical notifications cannot be opted out | Unit test on preference setting |
| FR-AI-005 | PII scrubbed before AI prompt assembly | Test with known PII corpus |

---

## 5. Non-Functional Requirements

Each NFR has: statement, target, measurement method, acceptance threshold, owner.

### 5.1 Performance

| ID | Statement | Target | Method | Threshold | Owner |
|---|---|---|---|---|---|
| NFR-P-001 | Catalog search latency | P95 < 100ms (Enterprise) | OpenSearch metrics, k6 | <100ms 30d rolling | SRE |
| NFR-P-002 | PDP load TTFB | P95 < 300ms (Enterprise) | RUM (Real User Monitoring) | <300ms | Web team |
| NFR-P-003 | Checkout submission | P95 < 800ms (Enterprise) | OTel trace | <800ms | Eng |
| NFR-P-004 | Vendor dashboard initial paint | P95 < 1.2s (Enterprise) | Lighthouse CI | <1.2s | Web team |
| NFR-P-005 | Outbox relay lag | P95 < 2s (Enterprise) | Prometheus | <2s | Eng |
| NFR-P-006 | Order ingestion sustained | 5,000/min | k6 load test | sustain 30min | SRE |
| NFR-P-007 | Order ingestion burst | 10,000/min | k6 spike test | sustain 5min | SRE |
| NFR-P-008 | API P95 | < 200ms (Enterprise) | Gateway metrics | <200ms | Eng |
| NFR-P-009 | Background payout batch (10k vendors) | < 30min | Job duration metric | <30min | SRE |
| NFR-P-010 | AI completion P95 | < 8s non-streaming | OTel trace | <8s | AI team |
| NFR-P-011 | Webhook delivery latency | P95 < 5s | dispatcher metric | <5s | Eng |
| NFR-P-012 | Storefront cache hit ratio | >= 85% | CDN logs | >= 85% | SRE |

### 5.2 Scalability

| ID | Statement | Target | Method |
|---|---|---|---|
| NFR-S-001 | Concurrent buyers per tenant | 50k (Enterprise) | Soak test |
| NFR-S-002 | Concurrent vendor sessions | 5k per tenant (Enterprise) | Soak test |
| NFR-S-003 | Catalog size | 10M SKUs / tenant (Enterprise) | Functional test at scale |
| NFR-S-004 | Tenants per pooled multisite | 200 | Provisioning test |
| NFR-S-005 | Sub-orders per parent | 50 | Boundary test |
| NFR-S-006 | DB queries per request | < 50 P95 | Query logger |
| NFR-S-007 | Sustained checkout TPS | 1000/sec platform-wide | Load test |

### 5.3 Availability & Reliability

| ID | Statement | Target | Method |
|---|---|---|---|
| NFR-A-001 | Tenant uptime (Enterprise) | 99.95% rolling 30d | Uptime monitor |
| NFR-A-002 | Webhook delivery success | 99.95% | Dispatcher metric |
| NFR-A-003 | Order data durability | 11 nines | Aurora SLA + cross-AZ |
| NFR-A-004 | Outbox no-loss | 100% (zero loss) | Reconciliation test |
| NFR-A-005 | RTO Tier-0 (Enterprise) | <= 1h | DR drill |
| NFR-A-006 | RPO Tier-0 (Enterprise) | <= 5min | Aurora replica lag |
| NFR-A-007 | Graceful degradation when OpenSearch down | Storefront still returns top-100 from MySQL | Chaos test |
| NFR-A-008 | Graceful degradation when AI service down | Dashboard works; AI banner shown | Chaos test |
| NFR-A-009 | Stripe webhook idempotency | 100% (replay produces same outcome) | Replay test |

### 5.4 Security

| ID | Statement | Target | Method |
|---|---|---|---|
| NFR-Sec-001 | OWASP ASVS conformance | L2 (Enterprise) | Annual review |
| NFR-Sec-002 | MFA for admin/vendor (Enterprise) | 100% required | Policy enforcement |
| NFR-Sec-003 | Transport encryption | TLS 1.3 only | Config audit |
| NFR-Sec-004 | At-rest encryption | AES-256 | KMS audit |
| NFR-Sec-005 | Secrets in code | 0 | Secret scanning |
| NFR-Sec-006 | Critical CVE patch window | 7 days | Vuln scanner |
| NFR-Sec-007 | Audit log retention | 2y (Enterprise) | S3 retention |
| NFR-Sec-008 | Penetration test | Annual external | Engagement |
| NFR-Sec-009 | PCI scope | SAQ-A only (Stripe Elements) | QSA review |
| NFR-Sec-010 | Tenant data isolation | 100% (no cross-tenant) | RLS + lint + audit |

### 5.5 Maintainability

| ID | Statement | Target | Method |
|---|---|---|---|
| NFR-M-001 | Unit test coverage | >= 80% per plugin | CI gate |
| NFR-M-002 | Code style compliance | 100% (PHP_CodeSniffer + ESLint) | CI gate |
| NFR-M-003 | Static analysis (PHPStan) level | Level 8 | CI gate |
| NFR-M-004 | Cyclomatic complexity per method | <= 10 | Linter |
| NFR-M-005 | Public API breaking change without major bump | 0 | API diff tool in CI |
| NFR-M-006 | Documentation coverage of public APIs | 100% | Doc-comment lint |

### 5.6 Portability & Compatibility

| ID | Statement | Target |
|---|---|---|
| NFR-Port-001 | Browser support | Chrome/Edge/Firefox/Safari last 2 majors |
| NFR-Port-002 | Mobile breakpoints | iOS Safari, Chrome Android |
| NFR-Port-003 | PHP versions | 8.2, 8.3 |
| NFR-Port-004 | MySQL versions | 8.0 LTS |
| NFR-Port-005 | WP versions | 6.4, 6.5 LTS |
| NFR-Port-006 | WC versions | 8.x LTS |

### 5.7 Usability & Accessibility

| ID | Statement | Target | Method |
|---|---|---|---|
| NFR-Use-001 | WCAG conformance | 2.1 AA | axe-core CI + manual |
| NFR-Use-002 | Locales supported | 30+ (Enterprise) | i18n test |
| NFR-Use-003 | Mobile viewport breakpoint | 360px min | Visual regression |
| NFR-Use-004 | Keyboard navigation | 100% interactive elements | Manual + axe |

### 5.8 Observability

| ID | Statement | Target |
|---|---|---|
| NFR-Obs-001 | Trace sampling (Enterprise) | 10% |
| NFR-Obs-002 | Log retention | 30d hot, 1y cold |
| NFR-Obs-003 | Metric scrape interval | 15s |
| NFR-Obs-004 | Alert ack SLA (SEV-1) | < 5min (24/7) |
| NFR-Obs-005 | MTTR SEV-1 | < 1h (Enterprise) |
| NFR-Obs-006 | Trace correlation across services | 100% (W3C traceparent) |

### 5.9 Data Quality

| ID | Statement |
|---|---|
| NFR-DQ-001 | Money fields stored as INT minor units + ISO-4217 |
| NFR-DQ-002 | Datetimes stored as UTC timestamp + IANA tz where user-facing |
| NFR-DQ-003 | All `wp_mercato_*` tables include `tenant_id`, `created_at`, `updated_at` |
| NFR-DQ-004 | Append-only tables (commission, payout, audit) MUST NOT be UPDATEd |

---

## 6. External Interface Requirements

### 6.1 Stripe Connect
- **IR-STR-001** Mercato MUST use Stripe API version pinned (`stripe-version` header); update flow tested.
- **IR-STR-002** Stripe webhook verified via signing secret per endpoint (per tenant).
- **IR-STR-003** Idempotency keys MUST be sent on every Stripe mutation.
- **IR-STR-004** Account onboarding uses Express or Custom per tenant choice.
- **IR-STR-005** Destination Charge transfer uses `source_transaction` linking.

### 6.2 OpenSearch
- **IR-OS-001** REST API v2.x.
- **IR-OS-002** Per-tenant index template.
- **IR-OS-003** Bulk indexing batch size 1000 documents.
- **IR-OS-004** ISM policy for hot/warm/cold lifecycle.

### 6.3 KYC Providers (Stripe Identity default)
- **IR-KYC-001** Document upload via provider SDK only; never direct.
- **IR-KYC-002** Webhook validated; idempotency via event ID.
- **IR-KYC-003** Re-verification trigger 365 days.

### 6.4 Tax Engines (TaxJar, Avalara)
- **IR-TAX-001** Cart calculation latency 2s; cached 60s.
- **IR-TAX-002** Filing export monthly via engine API or CSV.

### 6.5 Email (Sendgrid / Postmark)
- **IR-EM-001** SMTP relay via API; raw SMTP not used.
- **IR-EM-002** Webhook on bounce / complaint → suppression list.

### 6.6 SMS (Twilio)
- **IR-SM-001** TCPA / GDPR consent enforced.
- **IR-SM-002** Status callbacks tracked.

### 6.7 AI Providers (OpenAI / Anthropic / local)
- **IR-AI-001** API key per provider stored in Vault.
- **IR-AI-002** Streaming via SSE.
- **IR-AI-003** Multi-provider routing via LiteLLM gateway.

### 6.8 Object Storage (AWS S3)
- **IR-S3-001** Bucket per data classification (public, kyc, reports, telemetry).
- **IR-S3-002** Presigned URL upload only.
- **IR-S3-003** SSE-KMS with per-tenant key for sensitive buckets.

### 6.9 CDN (CloudFront)
- **IR-CDN-001** All static assets cached at edge.
- **IR-CDN-002** Signed URLs for private content.

### 6.10 Search/Analytics Sinks
- **IR-AN-001** Kafka publish to analytics topic; ETL via dbt nightly.
- **IR-AN-002** BI export to Looker/PowerBI via secure connector.

---

## 7. Constraints, Assumptions, Dependencies

### 7.1 Constraints
- **C-01** Must run on PHP 8.2+ within WordPress 6.4+ / WC 8.0+ HPOS.
- **C-02** Must remain installable as standard WP plugins (no custom Apache modules).
- **C-03** Must not modify WC core tables; only read/write to documented WC structures.
- **C-04** Must comply with WordPress.org plugin guidelines for community Tier 1 plugins.
- **C-05** Stripe is the canonical payment processor for MVP; PayPal Phase 2.
- **C-06** Hosting on AWS us-east-1 + us-west-2 for Enterprise multi-region; tenants in EU on eu-west-1.

### 7.2 Assumptions
- **A-01** Tenants will deploy `mercato-core` along with all required plugins; engineering distributes as a bundle on first install.
- **A-02** Stripe, AWS, OpenAI, TaxJar maintain published SLAs ≥ 99.9%.
- **A-03** Tenants will provide their own marketing/SEO content.
- **A-04** Vendors will provide their own product data and warranty terms.
- **A-05** Operator (us) handles base-platform compliance; tenants handle tenant-specific compliance.

### 7.3 Dependencies
- **D-01** Aurora MySQL 8.0 (Serverless v2 acceptable for SMB tier).
- **D-02** ElastiCache Redis 7.x.
- **D-03** OpenSearch 2.x (or managed equivalent).
- **D-04** Kafka (MSK) or RabbitMQ for event bus (ADR-001).
- **D-05** ArgoCD + GitHub Actions for CD.
- **D-06** Datadog or self-hosted Grafana/Prometheus/Tempo for observability.
- **D-07** PagerDuty for on-call.
- **D-08** HashiCorp Vault or AWS Secrets Manager for secrets.
- **D-09** Qdrant or pgvector for AI RAG (ADR-003).

---

## 8. Data Requirements

| ID | Statement |
|---|---|
| DR-001 | Every entity carries `tenant_id BIGINT UNSIGNED NOT NULL`, indexed. |
| DR-002 | Soft-delete fields `deleted_at` for vendor, product, message, review. |
| DR-003 | Append-only tables (commissions, payouts, audit_log): no UPDATE / DELETE permitted by application user role. |
| DR-004 | Data classification & retention table (see Vol 09 §6): PII, financial, KYC, audit, marketing. |
| DR-005 | KYC documents: stored S3, KMS, 7y retention, then deletion certified. |
| DR-006 | Audit log: 90d hot / 2y cold (Enterprise). |
| DR-007 | Order/commission: 7y (tax). |
| DR-008 | Buyer PII: anonymize on account deletion, except where retained under DR-007. |
| DR-009 | Backups encrypted, copied cross-region, restore tested monthly. |
| DR-010 | Search index data is rebuildable; never primary store. |
| DR-011 | Analytics store may contain pseudonymized PII; access role-restricted. |
| DR-012 | All money: INT minor + ISO-4217. |
| DR-013 | Timezones: storage UTC; display per user preference. |

---

## 9. Risk-Based Requirements

Selected requirements directly mitigate risks in Vol 03 §9:

| Risk | Requirements |
|---|---|
| R-03 cross-tenant leak | FR-CORE-006 + DR-001 + NFR-Sec-010 |
| R-04 PII breach | FR-KYC-002 + NFR-Sec-004 + DR-004 |
| R-06 AI cost spike | FR-AI-004 + NFR-Obs-* |
| R-13 GDPR fine | DR-008 + Vol 03 §8.2 DSAR workflow |
| R-18 migration data loss | FR-MIG-002 + FR-MIG-004 |

---

## 10. Traceability Matrix (Auto-Generated)

Output: `docs_v2/deliverables/traceability.xlsx`. Columns:

| Story ID | FR ID | NFR ID | DDL Table | Endpoint | Test ID | Status |

Builders' rules:
- Every PRD story maps to ≥ 1 FR.
- Every FR maps to ≥ 1 endpoint OR background job.
- Every endpoint maps to ≥ 1 test (E2E, integration, or contract).
- NFRs mapped to load/security/chaos tests.

CI fails if any story is orphaned.

---

## 11. Cross-Volume Cross-References

| This Section | See Also |
|---|---|
| §4 FR catalog | Vol 04 FSD §4–§22 |
| §5 NFRs | Vol 01 §13 SLOs; Vol 10 QA load/perf tests |
| §6 IRs | Vol 04 FSD per-plugin integrations; Vol 07 OpenAPI |
| §7 Constraints | Vol 01 §16 non-negotiables |
| §8 Data | Vol 06 DDL; Vol 09 §6 classification |

---

*End of Volume 05 — SRS v2.0.*
