# Volume 00 — Mercato MVP Implementation Cut

> **Read this before anything else in `docs_v2/`.**
>
> Document owner: CPO + CTO + Head of SRE
> Status: **Implementation Baseline v2.0 — Approved for MVP planning; ADRs pending**
> Version: 1.0.0
> Cross-references: Vol 02 PRD §9; Vol 01 §16; Implementation Backlog

---

## 0. Why this document exists

The documentation suite in volumes 01–15 specifies the **target architecture** for Mercato — a fully enterprise-grade marketplace with 99.99% uptime, AI Copilot, BYOK, multi-region failover, 27 plugins, double-entry accounting, and SOC-2 Type II.

If engineering treats every line of those documents as MVP-required, **the platform will never ship.** Both external reviewers (GPT and Grok, see `docs_v2/deliverables/Readiness_Scorecard.md`) flagged this as the single biggest risk in the documentation suite.

This document is the contract that says **exactly** what must be built, stubbed, deferred, or mocked to declare MVP done. Everything outside this cut is post-MVP unless an explicit ADR overrides.

---

## 1. MVP Definition

**MVP success criterion:** one beta tenant operates a live marketplace with ≥10 vendors, taking real buyer orders that split correctly across vendors, with commissions computed, payouts disbursed weekly via Stripe Connect, and a published reconciliation report that agrees with Stripe Treasury within ±$0.01.

**Target duration:** 12 weeks from kickoff.

**Target tenant tier:** Starter only (Pooled multisite).

**Target geo:** US only (`us-east-1`).

**Target buyer geo:** US, single currency (USD).

---

## 2. The Cut — Module by Module

Legend:
- ✅ **BUILD** — fully implemented per spec for MVP.
- 🟡 **STUB** — minimal viable version implemented; advanced behavior deferred.
- 🟦 **MOCK** — fake/dev-only implementation; real integration deferred.
- ⛔ **DEFER** — out of MVP scope entirely; spec exists for later phases.

### 2.1 Core Framework (`mercato-core`)

| Capability | Status | Notes |
|---|---|---|
| Plugin manifest registry + topological boot | ✅ BUILD | Vol 04 §4.2 |
| DI container | ✅ BUILD | |
| DB migrator (up/down/verify) | ✅ BUILD | Vol 06 §8 |
| RBAC engine (9 roles, capability check API) | ✅ BUILD | Vol 09 §5 |
| Capability JWT validator | 🟡 STUB | Use static feature flags in `wp_options` for MVP; Control Plane JWT issuance deferred to P2 |
| Outbox publisher | ✅ BUILD | Vol 01 §7.5 |
| Outbox relay daemon | ✅ BUILD | Go binary per ADR-006 |
| Idempotency-Key store | ✅ BUILD | |
| WC hook adapter | ✅ BUILD | Vol 04 §3.6 — only the hooks MVP modules need |
| Settings registry | ✅ BUILD | |
| Deprecation log | 🟡 STUB | Logging only; admin UI deferred |
| Telemetry buffer | ⛔ DEFER | No metered billing yet |

### 2.2 Vendors (`mercato-vendors`)

| Capability | Status |
|---|---|
| Registration | ✅ BUILD |
| KYC via Stripe Identity | ✅ BUILD |
| Tenant approval workflow (manual mode only) | ✅ BUILD |
| Suspension with reason | ✅ BUILD |
| Storefront settings (name, slug, logo, return policy) | ✅ BUILD |
| Vendor staff roles | 🟡 STUB — owner role only at MVP; multi-role at P2 |
| Vendor data export | ⛔ DEFER P2 |
| Closure path | ⛔ DEFER P2 |
| Reputation scoring | ⛔ DEFER P2 |
| Multi-tier KYC providers (Persona, Trulioo) | ⛔ DEFER — Stripe Identity only |

### 2.3 Products (`mercato-products`)

| Capability | Status |
|---|---|
| Simple product CRUD via SPA | ✅ BUILD |
| Shadow projection to WC HPOS | ✅ BUILD |
| Archive (soft-delete) | ✅ BUILD |
| Image upload via S3 presign + virus scan | ✅ BUILD |
| Bulk CSV import | ⛔ DEFER P2 |
| Variable products | ⛔ DEFER P2 — `product_type=simple` only at MVP |
| Grouped / digital / service / subscription products | ⛔ DEFER |
| Moderation modes | 🟡 STUB — `off` only at MVP |
| OpenSearch indexing | ⛔ DEFER P2 — MySQL catalog browse only |
| AI description generation | ⛔ DEFER P3 |
| Product attributes / variations | ⛔ DEFER P2 |

### 2.4 Orders (`mercato-orders`)

| Capability | Status |
|---|---|
| Multi-vendor cart split | ✅ BUILD |
| Per-vendor shipping line | ✅ BUILD |
| Sub-order lifecycle (created → acknowledged → shipped → delivered → completed) | ✅ BUILD |
| Tracking number capture + carrier autodetect | ✅ BUILD (carrier picker UI; autodetect P2) |
| Order item snapshot | ✅ BUILD |
| Auto-cancellation cron (72h non-ack) | ✅ BUILD |
| Buyer cancel sub-order pre-ship | 🟡 STUB — admin-only at MVP |
| Multi-currency orders | ⛔ DEFER P3 |
| Sub-order partial refund | ✅ BUILD |

### 2.5 Commissions (`mercato-commissions`)

| Capability | Status |
|---|---|
| Global default rate | ✅ BUILD |
| Per-vendor override | ✅ BUILD |
| Per-category override | 🟡 STUB — table exists, UI deferred to P2 |
| Per-product override | ⛔ DEFER P2 |
| Tiered rules | ⛔ DEFER P2 |
| Hold period | ✅ BUILD |
| Refund reversal (append-only) | ✅ BUILD |
| Preview API | ⛔ DEFER P2 |
| CSV/XLSX export | ✅ BUILD (CSV); XLSX P2 |

### 2.6 Payouts (`mercato-payouts`)

| Capability | Status |
|---|---|
| Vendor balance (available / pending) | ✅ BUILD |
| Weekly batch schedule | ✅ BUILD |
| Daily / biweekly / monthly schedules | ⛔ DEFER P2 |
| Manual trigger from admin | ✅ BUILD |
| Stripe Connect Destination Charge | ✅ BUILD |
| PayPal payout | ⛔ DEFER P2 |
| Bank wire (Plaid/Wise) | ⛔ DEFER P4 |
| Daily reconciliation ±$0.01 | ✅ BUILD |
| Monthly PDF statement | 🟡 STUB — CSV at MVP; PDF P2 |
| Failed-payout retry policy | ✅ BUILD |
| Failed-payout manual review queue | ✅ BUILD |

### 2.7 Reviews (`mercato-reviews`)

| All capabilities | ⛔ **DEFER ENTIRELY to P2** |

### 2.8 Disputes (`mercato-disputes`)

| Capability | Status |
|---|---|
| Buyer opens dispute | 🟡 STUB — basic form; admin email-only at MVP |
| Vendor response 72h | ⛔ DEFER P2 |
| Tenant adjudication UI | ⛔ DEFER P2 |
| Auto-escalation | ⛔ DEFER P2 |
| Appeal flow | ⛔ DEFER P3 |

MVP path: dispute creates a record + emails tenant admin who handles manually via email & manual refund.

### 2.9 Messaging (`mercato-messaging`)

| Capability | Status |
|---|---|
| Buyer↔Vendor threads (text-only) | ✅ BUILD |
| SSE real-time push | 🟡 STUB — 30s polling at MVP; SSE P2 |
| Attachments | ⛔ DEFER P2 |
| Admin read with audit | ⛔ DEFER P2 |
| Off-platform detection | ⛔ DEFER P3 |
| AI reply suggestions | ⛔ DEFER P3 |

### 2.10 Notifications (`mercato-notifications`)

| Capability | Status |
|---|---|
| Email channel (Sendgrid) | ✅ BUILD |
| SMS / Push / Webhooks / In-app | ⛔ DEFER P2 |
| Per-locale templates | 🟡 STUB — `en-US` only |
| Suppression list (bounces) | ✅ BUILD |
| Quiet hours / preferences | ⛔ DEFER P2 |
| Subscriber webhook outbound | ⛔ DEFER P2 |

### 2.11 Reports (`mercato-reports`)

| Capability | Status |
|---|---|
| Tenant dashboard (GMV, take, vendors, orders) | ✅ BUILD — basic |
| Vendor dashboard (sales, top products) | ✅ BUILD — basic |
| CSV export | ✅ BUILD |
| XLSX/PDF export | ⛔ DEFER P2 |
| Custom report builder | ⛔ DEFER P3 |
| ClickHouse analytics store | ⛔ DEFER P3 — use Aurora replica at MVP |

### 2.12 Search (`mercato-search`)

⛔ **DEFER ENTIRELY to P2.** MVP storefront uses native WC search + MySQL `LIKE` against shadow catalog (capped at 1k results).

### 2.13 Subscriptions (`mercato-subscriptions`)

| Capability | Status |
|---|---|
| Tenant billing (Stripe Subscription) | 🟡 STUB — manual invoicing at MVP for beta tenant |
| Vendor-side recurring | ⛔ DEFER P3 |
| Buyer-side recurring | ⛔ DEFER P3 |

### 2.14 Tax (`mercato-tax-engine`)

| Capability | Status |
|---|---|
| TaxJar integration | 🟦 MOCK — fixed-rate fallback (configured per state by hand) at MVP; TaxJar adapter at P2 |
| Avalara | ⛔ DEFER P3 |
| Marketplace Facilitator rules | ⛔ DEFER P2 — vendor-collect at MVP, with disclaimer |

### 2.15 KYC (`mercato-kyc-kyb`)

| Capability | Status |
|---|---|
| Stripe Identity flow | ✅ BUILD |
| Document upload to private S3 + KMS | ✅ BUILD |
| Re-verification trigger | 🟡 STUB — manual at MVP |
| Compliance officer dashboard | ⛔ DEFER P2 |

### 2.16 Fraud (`mercato-fraud-risk`)

⛔ **DEFER ENTIRELY to P3.** Rely on Stripe Radar + manual tenant review at MVP.

### 2.17 AI Copilot (`mercato-ai-copilot`)

⛔ **DEFER ENTIRELY to P3.** No AI in MVP.

### 2.18 Collaboration (`mercato-collaboration`)

⛔ **DEFER to P4.**

### 2.19 Enterprise (`mercato-enterprise`)

| Capability | Status |
|---|---|
| Tenant provisioning (pooled multisite) | ✅ BUILD |
| Token-based branding | ✅ BUILD |
| Custom domain | ⛔ DEFER P3 |
| Feature flags | 🟡 STUB — static config at MVP; Control Plane sync P3 |
| SSO (SAML/OIDC) | ⛔ DEFER P3 |

### 2.20 Migration (`mercato-migration`)

⛔ **DEFER ENTIRELY to P4.**

### 2.21 Integration Adapters

| Adapter | Status |
|---|---|
| `mercato-stripe-connect` | ✅ BUILD |
| `mercato-sendgrid` | ✅ BUILD |
| `mercato-aws-s3` | ✅ BUILD |
| `mercato-postmark` / `mercato-twilio` / `mercato-paypal-marketplace` / `mercato-taxjar` / `mercato-avalara` / `mercato-shippo` | ⛔ DEFER P2+ |

---

## 3. Infrastructure Cut

| Component | MVP | Post-MVP |
|---|---|---|
| Hosting | AWS `us-east-1` only | Add `us-west-2` warm standby (Enterprise) at P4 |
| Compute | Single EKS cluster; 3-AZ; HPA simple CPU | Add custom-metric HPA at P2; multi-cluster P4 |
| Database | Aurora MySQL **Serverless v2** (cost-friendly) ACU 1–8 | Provisioned cluster at Business+; Aurora Global at P4 |
| Cache | ElastiCache Redis, single shard + 1 replica | Cluster mode at P2 |
| Search | **Not deployed at MVP** | OpenSearch domain at P2 |
| Broker | **Single-AZ Kafka (MSK) — 2 brokers** | 3 brokers + multi-AZ at P2 |
| Vector store | Not deployed | Qdrant at P3 |
| AI providers | None | OpenAI/Anthropic at P3 |
| Service mesh | **Not deployed** | Linkerd at P3 |
| Vault | AWS Secrets Manager (simpler than HashiCorp Vault at MVP) | HashiCorp Vault optional at P3 |
| Observability | Prometheus + Grafana + FluentBit + OpenTelemetry to **Tempo lite single-node**; **Datadog deferred** | Full Datadog or self-hosted scale-out at P2 |
| WAF | **AWS WAF managed rules (free tier)** | Custom rules + Bot Management at P2 |
| BYOK | ⛔ Not at MVP | At P4 |
| CDN | CloudFront | unchanged |

---

## 4. Compliance Cut

| Requirement | MVP | Post-MVP |
|---|---|---|
| PCI-DSS SAQ-A (via Stripe Elements) | ✅ MVP | continuous |
| TLS 1.3 everywhere | ✅ MVP | |
| At-rest KMS encryption (single shared CMK) | ✅ MVP | per-tenant CMK at P3 |
| Audit log (append-only, monthly partitioned) | ✅ MVP | Object Lock at P2 |
| MFA TOTP (optional admin/vendor) | ✅ MVP | mandatory at Enterprise (P3) |
| GDPR DSAR (manual workflow, 30d SLA) | ✅ MVP — manual process | automated at P2 |
| SOC-2 Type II | observation period starts at MVP launch | first report ~12 months |
| OWASP ASVS L1 | ✅ MVP | L2 at P2 |
| External pentest | scoped on Phase-1 features | quarterly cadence at P2 |

---

## 5. Quality Cut

| Dimension | MVP target |
|---|---|
| Unit test coverage | ≥70% (relaxed from 80% target) |
| PHPStan level | Level 6 (relaxed from 8) |
| Playwright E2E | Top 8 of the 30 scenarios (TC-E2E-001..008) |
| k6 perf | TC-PERF-001 (search), 003 (checkout), 006 (orders) — relaxed thresholds (+50% headroom) |
| Visual regression | ✅ on for top 15 screens |
| Accessibility | axe-core CI gate; WCAG 2.1 AA on top 15 screens (full 110 at P2) |
| Chaos | ⛔ Deferred to P2 |
| SAST/SCA/IaC scans | ✅ ON |
| DAST nightly | ⛔ Deferred to P2 |

---

## 6. Performance Cut (relaxed NFRs for MVP)

| NFR | Target spec | MVP target |
|---|---|---|
| Catalog search P95 | <100ms (Enterprise) | <500ms (MySQL backed) |
| PDP TTFB P95 | <300ms | <800ms |
| Checkout P95 | <800ms | <2.5s |
| Vendor dashboard P95 | <1.2s | <3s |
| Uptime | 99.95% | 99.5% (Starter SLA) |
| Outbox lag P95 | <2s | <10s |
| Order ingestion sustained | 5,000/min | 500/min |

---

## 7. Documentation Cut

| Volume | MVP-required as written | Notes |
|---|---|---|
| 00 MVP Cut (this) | ✅ | Authoritative for MVP scope |
| 01 Architecture | ✅ partially — only sections noted as "P1" |
| 02 PRD | ✅ §9 Phase 1 only |
| 03 BRD | ✅ §5 Vendor + Orders + Commissions + Payouts; §8 PCI-DSS + GDPR baseline |
| 04 FSD | ✅ FRs for plugins marked ✅ BUILD above |
| 05 SRS | ✅ NFRs revised per §6 above |
| 06 Database | ✅ tables for ✅ BUILD plugins only; others created later |
| 07 OpenAPI | ✅ endpoints for ✅ BUILD plugins only; tag others `x-mvp: deferred` |
| 08 UX | ✅ Tenant Admin + Vendor SPA + Storefront screens for MVP scope (~35 of 110 screens) |
| 09 Security | ✅ baseline controls only |
| 10 QA | ✅ relaxed gates per §5 above |
| 11 DevOps | ✅ Single-region, simplified stack per §3 above |
| 12 AI | ⛔ DEFER ENTIRELY |
| 13 WC Compat | ✅ |
| 14 Packaging | ✅ |
| 15 Runbooks | ✅ |

---

## 8. Exit Criteria (Definition of Done for MVP)

All must be true to declare MVP shipped:

1. ✅ One real tenant in production with ≥10 vendors and >$10k cumulative GMV processed.
2. ✅ Multi-vendor checkout works end-to-end (real Stripe charges, real refunds).
3. ✅ Weekly payout batch ran successfully for 4 consecutive weeks with zero manual interventions.
4. ✅ Daily Stripe reconciliation drift ≤ $0.01 cumulative for 14 consecutive days.
5. ✅ Tenant dashboard + vendor dashboard usable on mobile + desktop.
6. ✅ Refund flow tested end-to-end with commission reversal verified.
7. ✅ KYC happy path + rejection path tested with real Stripe Identity submissions.
8. ✅ Top 8 Playwright E2E scenarios green on main for 7 consecutive days.
9. ✅ k6 perf on relaxed thresholds passing.
10. ✅ External Phase-1 pentest complete; 0 Critical findings.
11. ✅ Backup + restore drill executed once (real restore in staging).
12. ✅ On-call rotation live with PagerDuty hooked up.
13. ✅ Status page live at `status.mercato.com`.
14. ✅ Vendor agreement template (US) signed by counsel.
15. ✅ Privacy policy + ToS published.
16. ✅ Audit log captures all admin actions (verify manual sample of 50).

---

## 9. Things Explicitly NOT in MVP (for clarity)

- AI of any kind
- Multi-currency
- Multi-region
- BYOK
- White-label custom domain
- SSO
- Reviews
- Search via OpenSearch
- Reports beyond basic dashboards
- Bulk import
- Vendor staff roles beyond owner
- Subscriptions billing automation
- Fraud engine
- Migration importers
- Service mesh / Linkerd
- Vault
- Tax engine integration (TaxJar/Avalara)
- Sub-tenants / resellers
- Mobile native app

If a stakeholder asks "is X in MVP?" and X is on this list — the answer is no.

---

## 10. Open MVP Risks

| Risk | Mitigation |
|---|---|
| Beta tenant requests a non-MVP feature | Polite firm "no" until Phase 2; capture in product backlog |
| Multi-currency request from international beta vendor | Restrict beta to US vendors only |
| Stripe Connect onboarding rejection | Have manual support process; tenant admin assists |
| Single-AZ Kafka outage | Outbox table will buffer; accept temporary lag; document RTO |
| MySQL `LIKE` search too slow at >5k SKUs | Beta tenant cap: 5k SKUs across all vendors; upgrade to OpenSearch at P2 trigger |

---

## 11. Phase 2 Trigger Criteria

We start Phase 2 when:
- MVP exit criteria met for ≥30 days, AND
- At least 3 paying tenants live, OR beta tenant requests a Phase 2 feature with willingness to pay, AND
- Engineering capacity available (no MVP firefighting consuming >30% of team).

---

## 12. Cross-References

| Topic | See |
|---|---|
| Detailed Phase 1 story scope | Implementation Backlog `Phase = P1 MVP` filter |
| Per-plugin spec | Vol 04 FSD §4–§22 |
| Architectural patterns | Vol 01 |
| Compliance baseline | Vol 09 |
| WC compatibility | Vol 13 |
| Packaging | Vol 14 |
| Runbooks for MVP | Vol 15 |

---

*End of Volume 00 — MVP Implementation Cut v1.0.*
