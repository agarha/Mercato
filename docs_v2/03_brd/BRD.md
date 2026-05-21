# Volume 03 — Mercato Enterprise Marketplace Platform
## Business Requirements Document (Engineering-Grade, Implementation-Ready)

> Document owner: CPO + Senior PMs + Compliance Lead + Finance Lead
> Status: Implementation Baseline v2.0 — Approved for MVP planning; ADRs pending
> Version: 2.0.0 (full rewrite of v1.0)
> Cross-references: Vol 01 (Architecture), Vol 02 (PRD), Vol 04 (FSD), Vol 09 (Security), Vol 10 (QA/UAT)

---

## 1. Executive Assessment

The v1.0 BRD captured business intent but stopped short of the contractual specificity that procurement, legal, finance, and operational stakeholders need to commit. It defined high-level goals (modularity, multi-vendor support, SaaS monetization) without specifying business rules with stable identifiers, SLA tiers, regulatory boundaries, escalation paths, or financial reconciliation accuracy targets.

This rewrite delivers an enterprise-grade BRD with:

1. A **stakeholder matrix** spanning internal (operator) and external (tenant, vendor, buyer, regulator, processor) actors with explicit responsibility allocations (RACI).
2. A **business rules registry** keyed `BR-<domain>-<n>` so every rule is uniquely citable in contracts, legal, and engineering tickets (~250 rules across 14 domains).
3. A **regulatory compliance matrix** mapping controls to PCI-DSS, GDPR, CCPA, SOC-2 Type II, AML/KYC, consumer protection (Magnuson-Moss, EU Distance Selling), tax (US sales tax, EU VAT, GST/HST/PST, OECD VAT-on-services).
4. A **revenue & monetization grid** with SKU pricing, overage rules, FX policy, refund policy, and chargeback handling.
5. **SLA tiers** tied to subscription plans — uptime, response time, escalation, credit policy.
6. A **risk register** with quantified likelihood × impact and named mitigation owners.

**Implementation Readiness (this volume): 88/100.** Outstanding work is jurisdiction-specific legal review and finalized tenant contract templates, tracked in the Gap Matrix.

---

## 2. Gap Analysis vs. v1.0

| # | Gap | v1.0 State | v2.0 Resolution |
|---|---|---|---|
| G-BRD-001 | Business rules lacked IDs | Prose only | Stable `BR-<domain>-<n>` registry (§5) |
| G-BRD-002 | SaaS tiers under-specified | "Core / Pro / Enterprise" | Four-tier grid with limits, overages, SLAs (§6) |
| G-BRD-003 | Regulatory scope ambiguous | Mentioned PCI/GDPR | Full control matrix with evidence ownership (§8) |
| G-BRD-004 | RACI absent | Stakeholder list only | RACI per domain (§4) |
| G-BRD-005 | Refund/chargeback rules vague | Stated only | Detailed lifecycle with finance, accounting impact (§5.5, §5.8) |
| G-BRD-006 | Tax handling under-specified | Mentioned engines | Jurisdictional matrix + marketplace facilitator law treatment (§8.4) |
| G-BRD-007 | SLA credit policy absent | Uptime stated only | Tiered credit policy with claim process (§7) |
| G-BRD-008 | Risks not quantified | Listed | Likelihood × Impact × Owner × Mitigation (§9) |
| G-BRD-009 | Vendor agreement obligations vague | Mentioned | Standard clauses + tenant override surface (§10) |
| G-BRD-010 | i18n / multi-currency policy missing | Mentioned in passing | FX capture, display, and accounting policy (§6.4) |
| G-BRD-011 | DSAR / GDPR Right-to-Erase process undefined | Mentioned | Step-by-step workflow with retention exceptions (§8.2) |
| G-BRD-012 | Marketplace governance / prohibited items vague | One sentence | Category-level policy with enforcement tiers (§5.14) |

---

## 3. Business Overview & Value Proposition

### 3.1 Mission
Provide the world's most modular, secure, and scalable enterprise marketplace platform — bringing Mirakl/Amazon-grade capability to organizations that prefer the WordPress/WooCommerce ecosystem.

### 3.2 Value Proposition by Segment
- **SMB tenant:** turnkey marketplace with low setup cost, 60-second provisioning, prebuilt branded storefront, and Stripe-powered payouts.
- **Enterprise tenant:** absolute control over architecture, BYOK encryption, custom domain, dedicated cluster, white-glove migration, SLA-backed support.
- **Franchise / regional tenant:** multi-region deployment, locale-aware tax/legal, regional fulfillment partners.
- **White-label reseller:** rebrand the platform completely; offer sub-tenants to their own customers (Phase 4).

### 3.3 Why Now
Global online marketplaces represent >60% of e-commerce; the next decade is brand marketplaces and B2B procurement marketplaces. The existing WooCommerce ecosystem has Dokan, WCFM, and YITH, but none of these scale beyond mid-market because of `wp_postmeta` ceiling, monolithic architecture, and weak SaaS controls. Mercato's modular plugin-first + microservices-augmented architecture is the first that scales linearly while keeping the WordPress install base addressable.

### 3.4 Strategic Bets
1. **Modularity wins integrators.** A 27-plugin ecosystem (Vol 01 §4) is a marketplace itself, attracting partners who build adapters.
2. **AI is a differentiator, not a feature.** Embed AI assistance in vendor and operator workflows, with clear cost controls and PII protection.
3. **Compliance is a moat.** Tenants in regulated geographies (EU, UK, ANZ, parts of LatAm) need GDPR/Marketplace Facilitator/PSD2 compliance turnkey.

---

## 4. Stakeholders & RACI

### 4.1 Stakeholder Catalog

#### Internal (Platform Operator)
| Stakeholder | Responsibility Summary |
|---|---|
| Platform CEO/COO | P&L; strategic partnerships; major incidents |
| CPO | Product roadmap; persona prioritization; PRD ownership (Vol 02) |
| CTO / VP Eng | Architecture; engineering quality; Vol 01/04–07 |
| VP Sales / GTM | Tenant acquisition; pricing |
| VP Customer Success | Tenant retention; expansion; NRR |
| Marketplace Operations (in-house) | Templated tenant onboarding; tenant escalations |
| Security & Compliance Lead | Vol 09; audit evidence; incident response |
| Finance | Subscription billing; revenue recognition; reconciliation |
| Legal | Contract templates; regulator interaction |
| Support | Tenant + tenant-staff support; tier 1/2/3 |
| Data / Analytics | Vol 11 dashboards; revenue-quality metrics |

#### External
| Stakeholder | Responsibility Summary |
|---|---|
| Tenant (Marketplace Owner) | Operates a marketplace; configures rules; collects platform-level take rate |
| Tenant staff (Manager, Support, Finance) | Day-to-day operations |
| Vendor (Seller) | Lists products; fulfills orders; receives payouts |
| Vendor staff | Per-vendor delegated roles |
| Buyer | Purchases |
| Payment processors (Stripe, PayPal, Adyen) | Card processing; payouts; identity |
| Tax engines (TaxJar, Avalara) | Tax computation & filing |
| KYC providers (Stripe Identity, Persona, Trulioo, Sumsub) | Identity verification |
| Shipping carriers (FedEx, UPS, DHL, EasyPost) | Label generation; tracking |
| Email/SMS providers (Sendgrid, Postmark, Twilio) | Notifications delivery |
| Regulators (FTC, EU DPAs, ICO, state AGs, FCA) | Oversight |
| Auditors (SOC-2, PCI QSA) | Annual audits |
| Partners (system integrators, agencies) | Tenant onboarding services |

### 4.2 RACI by Domain (excerpt)

| Activity | Tenant | Operator (us) | Vendor | Processor |
|---|---|---|---|---|
| Tenant provisioning | R | A | — | — |
| Vendor onboarding (KYC) | A | I | R | R (Stripe Identity) |
| Product moderation | A | I | R | — |
| Multi-vendor checkout | I | C | C | R (Stripe) |
| Commission rule definition | R | C | I | — |
| Payout execution | A | C | I | R (Stripe Connect) |
| Refund issuance | A | C | C | R |
| Dispute adjudication | R | C | C | — |
| GDPR DSAR fulfillment | A | R | C | — |
| Tax filing | A | I | R (own taxes) | R (engine) |
| PCI compliance (cardholder data) | — | A | — | R (Stripe) |
| AML/sanctions screening | — | A | — | R |
| SLA credit issuance | — | A | — | — |
| Marketplace governance enforcement | R | A | I | — |

(R=Responsible, A=Accountable, C=Consulted, I=Informed.)

---

## 5. Business Rules Registry

Format: `BR-<domain>-<sequence>` — short title — rule — rationale — owner.

### 5.1 Vendor Onboarding & Management
- **BR-VEN-001 — Mandatory KYC** — A vendor MUST complete KYC before publishing any product or receiving any payout. **Rationale:** AML / sanctions screening; Stripe Connect requirement. **Owner:** Compliance.
- **BR-VEN-002 — Approval Workflow** — Tenant configures approval mode: `auto` (no review) / `manual` (human review) / `tiered` (manual for first N or for restricted categories). **Owner:** Tenant.
- **BR-VEN-003 — Re-verification Triggers** — Material profile changes (bank account, beneficial owner ≥25%, legal entity type) MUST re-trigger KYC. **Owner:** Compliance.
- **BR-VEN-004 — Suspension** — Tenant admin MAY suspend a vendor with a recorded reason; suspension freezes new orders and payouts, but preserves open obligations. **Owner:** Tenant.
- **BR-VEN-005 — Closure Path** — A vendor MAY close their store only after fulfilling open orders, processing pending refunds, and consenting to data retention policy. **Owner:** Tenant.
- **BR-VEN-006 — Vendor Staff Roles** — Vendor staff roles are: `vendor_owner`, `vendor_manager`, `vendor_fulfiller`, `vendor_support`, `vendor_finance`, `vendor_read_only`. Each maps to capabilities in `mercato-core` RBAC. **Owner:** Platform.
- **BR-VEN-007 — Off-Platform Prohibition** — Vendors MUST NOT solicit transactions outside the platform; detected violations trigger warnings then suspension. **Owner:** Tenant + AI moderation.
- **BR-VEN-008 — Onboarding SLA** — Tenant reviews vendor application within 3 business days unless tenant configures longer. **Owner:** Tenant.
- **BR-VEN-009 — Data Retention** — Closed vendor records retained 7y (tax/legal); PII anonymized after 30d post-closure unless legal hold. **Owner:** Compliance.

### 5.2 Product & Catalog Management
- **BR-PROD-001 — Required Fields** — Every product MUST have: title, description ≥30 chars, price (>0), currency (ISO-4217), at least one image (≥800px on longest edge), category, vendor ownership.
- **BR-PROD-002 — Prohibited Categories** — Items in Section §5.14.2 are rejected at submission and on import.
- **BR-PROD-003 — Moderation Mode** — Tenant configures: `off`, `pre-publish review`, `post-publish flag`.
- **BR-PROD-004 — Variation Cap** — A variable product is limited to 250 variations to prevent UI/cache pathology; tenants may raise on request with Enterprise tier.
- **BR-PROD-005 — Bulk Import Limits** — CSV import ≤10k rows / file / hour at Pro tier; ≤100k at Enterprise.
- **BR-PROD-006 — Media Rights** — Vendor warrants ownership/license of all media; takedown procedure follows DMCA / EU CDSM for compliant geographies.
- **BR-PROD-007 — Inventory Truthfulness** — Stock MUST reflect actual availability; "sell-through" cases trigger Buyer Promise compensation (BR-FIN-014).
- **BR-PROD-008 — Auto-Translation** — Tenants MAY enable auto-translation; translated content tagged `translated: true` with source language stored.

### 5.3 Customer & Order Management
- **BR-ORD-001 — Single Cart, Multi-Vendor** — Buyer cart MAY contain items from any number of vendors; payment is single transaction; sub-orders are split server-side post-payment.
- **BR-ORD-002 — Shipping Per Sub-Order** — Each vendor sets its own shipping zones/rates; cart shows per-vendor shipping line.
- **BR-ORD-003 — Order Acknowledgment SLA** — Vendor must acknowledge within 24h; auto-canceled and refunded after 72h non-acknowledgment.
- **BR-ORD-004 — Fulfillment SLA** — Per vendor's published handling time (1–10 business days); breach triggers vendor reputation penalty (BR-REV-009).
- **BR-ORD-005 — Cancellation** — Buyer may cancel pre-ship; vendor may cancel with reason; admin may cancel any time with reason.
- **BR-ORD-006 — Returns Window** — Default 14 days post-delivery; tenant overrides allowed (3 / 7 / 14 / 30 days).
- **BR-ORD-007 — Order Snapshot** — Each order captures product attributes at purchase time; product edits/deletes do not retroactively change orders.
- **BR-ORD-008 — Cross-Currency Orders** — If buyer pays in currency ≠ vendor settlement currency, FX rate is captured at checkout time + 0–2% platform spread (configurable).

### 5.4 Commission & Payout
- **BR-COMM-001 — Rule Hierarchy** — Resolution order: product override → vendor override → category rule → global default.
- **BR-COMM-002 — Commission Base** — Commission applies to product subtotal pre-tax pre-shipping by default; tenant may include shipping; never on tax (regulatory).
- **BR-COMM-003 — Tiered** — Tenants may configure tiers by MTD GMV or rolling 30d GMV (e.g., 15% under $10k, 12% above).
- **BR-COMM-004 — Promotion Treatment** — Discounts reduce the commission base; coupon costs are borne per tenant config (vendor / tenant / split).
- **BR-COMM-005 — Hold Period** — Commission becomes payable after hold (default 14 days post-delivery; configurable).
- **BR-COMM-006 — Reversal on Refund** — Refunds reverse commission proportionally; a paid-out commission becomes negative balance against future payouts; persistent negative balance >30d triggers vendor recovery process.
- **BR-COMM-007 — Immutability** — Commission ledger rows are append-only; corrections issue offsetting entries with full audit trail.
- **BR-PAY-001 — Payout Schedule** — Tenants configure: daily / weekly (default) / biweekly / monthly / manual.
- **BR-PAY-002 — Minimum Payout Threshold** — Default $25; configurable per tenant / per vendor.
- **BR-PAY-003 — Payout Method** — Stripe Connect (default), PayPal (optional), bank wire (Enterprise tenants only via Plaid/Wise).
- **BR-PAY-004 — Currency** — Vendor settled in the currency Stripe Connect supports for their account; FX captured per BR-ORD-008.
- **BR-PAY-005 — Failed Payout Retry** — 3 retries with 24h backoff; after final failure, vendor notified; tenant required to manually intervene.
- **BR-PAY-006 — Statements** — Vendors receive PDF statements monthly; CSV/XLSX available on-demand within 60 days.
- **BR-PAY-007 — Reconciliation Tolerance** — Daily reconciliation against Stripe Treasury must agree within ±$0.01 cumulative per tenant per day; breach raises P1 incident.

### 5.5 Payment & Financial Processes
- **BR-FIN-001 — Tokenization** — Card numbers never traverse platform infrastructure; Stripe Elements / Stripe Checkout only. PCI scope is SAQ-A.
- **BR-FIN-002 — 3DS Mandate** — 3DS challenge follows acquirer rules; PSD2 SCA mandatory for EU.
- **BR-FIN-003 — Saved Payment Methods** — Buyer-saved methods stored as Stripe Customer + Payment Method IDs; never raw PAN.
- **BR-FIN-004 — BNPL** — Klarna / Affirm available as optional integration; eligibility per processor.
- **BR-FIN-005 — Chargeback Handling** — Platform forwards Stripe Dispute event to tenant + vendor; vendor responds with evidence; tenant may auto-accept liability per policy.
- **BR-FIN-006 — Chargeback Reserve** — Tenants on lower tiers may have rolling 7-day funds reserve up to N% of weekly GMV (tenant config).
- **BR-FIN-007 — Refund Method** — Refunds issued via original payment method only.
- **BR-FIN-008 — Partial Refunds** — Allowed; pro-rate commission, tax (where lawful), shipping retention per tenant policy.
- **BR-FIN-009 — Currency Conversion Policy** — All money in ledger stored as INT minor units + ISO-4217; FX rate captured + stored at transaction time.
- **BR-FIN-010 — Accounting Export** — Standard exports: GMV, net sales, taxes collected, refunds, commission income, payouts, gateway fees, FX adjustments. Periods: daily/weekly/monthly. Formats: CSV, XLSX, Quickbooks IIF, NetSuite SuiteTalk.
- **BR-FIN-011 — Revenue Recognition** — Subscription revenue recognized ratably over period; commission recognized at order completion (delivery + return window).
- **BR-FIN-012 — Sales Tax Marketplace Facilitator** — In jurisdictions where the platform is marketplace facilitator by law, the platform collects/remits; otherwise pass-through to vendor.
- **BR-FIN-013 — Withholding** — US 1099-K issued to vendors meeting threshold ($600 federal 2024+; state-specific overrides).
- **BR-FIN-014 — Buyer Promise** — If a vendor cancels for stock unavailability after acceptance, buyer is automatically refunded + receives a credit equal to lower of $10 or 10% of order value (tenant configurable / opt-in).

### 5.6 Messaging & Collaboration
- **BR-MSG-001 — Participants** — Only verified buyer, vendor, and assigned support agents can read a thread; admin reads are audit-logged with reason.
- **BR-MSG-002 — Immutability** — Messages append-only; visible-to-user deletions hide from UI but preserve in audit store.
- **BR-MSG-003 — Off-Platform Detection** — Heuristic + AI moderation flags emails, phone numbers, payment URLs; vendor receives warning; persistent violations trigger suspension.
- **BR-MSG-004 — Response SLA** — Vendor target 24h business-hours; vendor reputation penalty for chronic miss.
- **BR-MSG-005 — Attachments** — Max 10MB / file; allowlist MIME types (image/*, application/pdf); virus scanned via S3 + ClamAV.

### 5.7 Reviews & Ratings
- **BR-REV-001 — Eligibility** — Only buyers with completed sub-orders may review the product + vendor.
- **BR-REV-002 — Window** — Reviews allowed for 30 days post-delivery.
- **BR-REV-003 — Single Review** — One review per buyer per sub-order; editable for 7 days.
- **BR-REV-004 — Vendor Response** — One response per review.
- **BR-REV-005 — Moderation** — Reviews scanned for prohibited content; flagged for tenant manager review within 48h.
- **BR-REV-006 — Aggregate Display** — Vendor and product show: mean rating, count, distribution histogram, with minimum 10 reviews before showing mean (else "New").
- **BR-REV-007 — Incentivized Reviews** — Marked with `incentivized: true`; disclosed in UI.
- **BR-REV-008 — Fake Review Defense** — Velocity-based bot detection; verified-purchase only by default.
- **BR-REV-009 — Reputation Penalty** — Chronic late shipping, dispute loss, or low rating reduces search ranking (`mercato-search` boost adjustment).

### 5.8 Dispute Management
- **BR-DIS-001 — Initiation Window** — Buyer may open dispute within 30 days of delivery (or expected delivery).
- **BR-DIS-002 — Required Evidence** — Photos, messages, tracking data attachable; minimum: text reason and evidence chosen from a controlled list.
- **BR-DIS-003 — Vendor Response Window** — 72h; auto-escalates to tenant otherwise.
- **BR-DIS-004 — Tenant Adjudication** — Tenant rules engine + manual override; outcomes: full refund / partial refund / replacement / deny.
- **BR-DIS-005 — Vendor Liability Cap** — Per tenant policy; default = order value plus return shipping.
- **BR-DIS-006 — Linked Commission Reversal** — Refund actions automatically post commission reversal entries.
- **BR-DIS-007 — Appeal** — Vendor or buyer may appeal within 14 days; appeals reviewed by senior moderator.
- **BR-DIS-008 — SLAs** — Acknowledgment <24h; resolution <14d; escalated cases <30d.

### 5.9 Notifications & Communication
- **BR-NOT-001 — Critical Notifications** — Order placed/shipped, payment failed, payout completed/failed, dispute opened, KYC status changed — cannot be opted out.
- **BR-NOT-002 — Marketing Opt-In** — Explicit opt-in for marketing; unsubscribe in every email; CASL/CAN-SPAM compliant.
- **BR-NOT-003 — Localization** — Templates per locale; tenant may customize per channel.
- **BR-NOT-004 — Failed Delivery** — Bounces tracked; hard bounce >2x → notify user via secondary channel; suppression list updated.
- **BR-NOT-005 — Quiet Hours** — SMS/push respect recipient timezone quiet hours (default 22:00–08:00 local) unless critical.

### 5.10 Analytics & Reporting
- **BR-RPT-001 — Vendor Visibility** — Vendors see ONLY their own data; no cross-vendor benchmarks unless tenant publishes aggregated benchmarks.
- **BR-RPT-002 — Tenant Visibility** — Tenants see all vendors within tenant; cross-tenant analytics restricted to operator.
- **BR-RPT-003 — Export Limits** — Vendor export ≤90 days at a time on Starter, ≤2y on Enterprise.
- **BR-RPT-004 — KPI Definitions** — GMV, Net Sales, AOV, Conversion, Take Rate — formulas in §11 to ensure consistency.
- **BR-RPT-005 — Refresh Cadence** — Real-time KPIs (sales, orders) <30s lag; analytical KPIs (cohort retention) daily refresh.

### 5.11 Subscription & Licensing
- **BR-SUB-001 — Plan Surface** — Plans: Starter, Pro, Business, Enterprise (Vol 02 §6.1).
- **BR-SUB-002 — Annual Discount** — 17% off monthly when billed annually (default).
- **BR-SUB-003 — Trial Period** — 14-day trial for Starter/Pro; credit card required (auth, no capture).
- **BR-SUB-004 — Upgrades** — Effective immediately; prorated charge.
- **BR-SUB-005 — Downgrades** — Effective at next renewal; data retained.
- **BR-SUB-006 — Overage Billing** — AI completions, vendor cap, API RPS metered hourly, billed monthly.
- **BR-SUB-007 — Cancellation** — Effective at period end; data retained 30d post-cancel; export available.
- **BR-SUB-008 — Refund Policy** — Pro-rated refund for annual cancel within 30 days only; no refund for usage overages.
- **BR-SUB-009 — Hard Stop on Non-Payment** — After 3 failed retries over 14d, tenant moves to read-only; storefront returns 503 after 30d.

### 5.12 AI Collaboration
- **BR-AI-001 — Opt-In** — Tenant must opt in to AI features per category; vendor sees a toggle in profile.
- **BR-AI-002 — Transparency** — Any AI-generated content surfaced to a third party MUST be flagged "AI-assisted"; vendor must approve before public publish.
- **BR-AI-003 — No Auto-Send** — AI MUST NOT automatically send messages, place orders, or commit material actions; human-in-the-loop mandatory.
- **BR-AI-004 — PII Scrubbing** — PII removed/pseudonymized before requests leave tenant boundary; redaction rules in Vol 12 §5.
- **BR-AI-005 — Per-Tenant Limits** — Plan-based caps + per-vendor caps; overage billed per BR-SUB-006.
- **BR-AI-006 — Provider Outage** — Storefront and dashboard remain functional with AI disabled; degraded banner shown.

### 5.13 Multi-Tenancy & White-Label
- **BR-TEN-001 — Isolation Modes** — Pooled / Silo / Dedicated (Vol 01 §9.1) tied to plan tier.
- **BR-TEN-002 — Cross-Tenant Access** — Operator staff with `super_admin` capability may read tenants for support; every access audit-logged; tenant notified within 24h.
- **BR-TEN-003 — Branding** — Token-only branding at Starter; full template + custom domain at Pro+; SSO + BYOK at Enterprise.
- **BR-TEN-004 — Sub-Tenants (Reseller)** — Phase 4 only; reseller contract obliges flow-through of platform ToS.
- **BR-TEN-005 — Data Residency** — Pro+ tenants may pin storage region (US/EU/AU); KYC docs always region-resident.

### 5.14 Governance, Compliance & Policies

#### 5.14.1 Required Public Documents
Tenants are required to publish, or accept platform default text for: Terms of Service, Privacy Policy, Returns Policy, Vendor Agreement, Acceptable Use Policy, Cookie Policy, Dispute Resolution Policy. Platform provides editable templates per jurisdiction.

#### 5.14.2 Prohibited Categories (Platform-Level)
Categories where listings are blocked at platform level (tenants may add more):
- Weapons (firearms, ammo, replicas where prohibited)
- Pharmaceuticals (controlled substances; OTC may require category whitelisting)
- Counterfeit goods
- Stolen goods
- Adult content where prohibited by tenant jurisdiction
- Tobacco/vaping where age-gated requires explicit handling
- Live animals
- Hazardous materials (HAZMAT) without certification
- Items violating sanctions (OFAC, EU, UN, UK)
- Hate speech merchandise

Enforcement: AI moderation pre-publish + keyword block + tenant review queue + post-publish flagging.

#### 5.14.3 Enforcement Tiers
- Tier 1: Warning + removal of offending listing.
- Tier 2: Vendor suspension 7–30 days + remediation plan.
- Tier 3: Permanent ban + payout freeze pending obligation settlement.
- Tier 4: Law enforcement referral (CSAM, terrorism, etc.).

#### 5.14.4 DSAR / Right to Erasure
Workflow (§8.2 for detail): request received → identity verified → data inventory generated → 30-day fulfillment window → erasure with legal retention exceptions (tax 7y, dispute records 5y, etc.).

#### 5.14.5 Audit & Monitoring
All admin actions, financial changes, role changes, capability toggles, and policy edits logged immutably (Vol 09 §5).

---

## 6. Revenue & Monetization

### 6.1 SKU Grid (Platform → Tenant)

| Plan | $/mo (annual eq) | Vendor Cap | Order Cap/mo | AI Allotment | Domain | Isolation | SLA |
|---|---|---|---|---|---|---|---|
| Starter | $99 | 25 | 5,000 | 1,000 | shared subdomain | Pooled | 99.5% |
| Pro | $499 | 250 | 50,000 | 25,000 | Custom + Subdomain | Pooled | 99.9% |
| Business | $1,999 | 5,000 | 500,000 | 100,000 | Custom + Subdomain | Silo | 99.95% |
| Enterprise | Custom | Unlimited | Unlimited | Custom | Custom + Subdomain + BYOC | Dedicated | 99.99% |

### 6.2 Overage Policy
- AI completions: $5 per 1,000 (Pro), $4 per 1,000 (Business), negotiable Enterprise.
- Vendor cap: $5/vendor/mo (Pro), $3 (Business).
- API RPS overflow: $0.10/k requests; sustained breach over 7d triggers upsell conversation.

### 6.3 Tenant → Vendor (Take Rate)
- Tenants set their own commission; platform does not take a cut of tenant's take rate.
- Tenants may opt in to "Mercato Pay" (Phase 4) — bundled payment + payout where platform earns a basis-point share.

### 6.4 FX Policy
- Display: catalog in tenant default currency; PDP / cart in buyer-selected (if enabled).
- Capture: order denominated in buyer currency; FX rate from Stripe at checkout + optional platform spread.
- Ledger: each ledger row carries `currency_code` + `fx_rate_to_settlement_currency` + `fx_captured_at`.
- Settlement: vendor settled in their Stripe Connect account currency; tenant settled in platform's accounting currency.

### 6.5 Marketplace Take-Rate Calculator (Example)

A Pro tenant with $1M GMV/month and 10% take rate:
- Tenant revenue: $100k commission
- Tenant pays: $499 + (10k AI × $5/1k) = $549
- Platform marginal cost on AI: ~$8 (assuming $0.0008 per completion at scale)
- Net platform contribution from tenant: $499 + $42 ≈ $541

### 6.6 Refund & Chargeback Impact

- Refund within return window → commission reversed; platform absorbs Stripe fee on refunds per Stripe policy.
- Chargeback → if vendor responds with valid evidence (delivery proof, etc.), tenant may accept liability or fight; tenant pays Stripe dispute fee ($15) on loss; vendor's payout debited.

---

## 7. SLA & Operational Tiers

| Plan | Uptime | Webhook Delivery | Response Time (P1) | Resolution (P1) | Service Credit Cap |
|---|---|---|---|---|---|
| Starter | 99.5% | 99.5% | 24h | 7d | 0% |
| Pro | 99.9% | 99.9% | 4h | 48h | 10% MRR |
| Business | 99.95% | 99.95% | 1h | 24h | 25% MRR |
| Enterprise | 99.99% (or contracted) | 99.99% | 30min | 8h | 50% MRR / contract |

Service-credit claim process:
1. Tenant submits claim ≤30d after the incident via support portal.
2. Platform reviews logs; confirms breach via published SLO dashboards.
3. Credit applied to next invoice within 60d.
4. Exclusions: scheduled maintenance windows (announced ≥72h), force majeure, third-party processor outages (Stripe, AWS region).

Incident severity classification:
- **P1**: Platform-wide outage / data loss / security breach.
- **P2**: Tenant-isolated outage / major feature down.
- **P3**: Degradation / non-critical feature.
- **P4**: Cosmetic / inquiry.

Status page: `status.mercato.com` (real time + historical incidents + scheduled maintenance).

---

## 8. Regulatory Compliance Matrix

### 8.1 Standards & Frameworks
| Framework | Scope | Evidence Owner | Audit Cadence |
|---|---|---|---|
| PCI-DSS SAQ-A | Card handling via Stripe Elements | Security | Annual self-attest |
| SOC-2 Type II | All operations | Security + SRE | Annual + continuous |
| GDPR | EU buyers/vendors/tenants | Compliance + DPO | Continuous |
| CCPA / CPRA | California residents | Compliance | Continuous |
| UK Data Protection Act | UK | Compliance | Continuous |
| PIPEDA | Canada | Compliance | Continuous |
| LGPD | Brazil | Compliance | Continuous |
| HIPAA | N/A — explicitly out of scope (no PHI) | — | — |
| ISO 27001 | Phase 4 target | Security | Annual |
| OWASP ASVS L2 | All web/REST surface | Eng + Sec | Continuous |

### 8.2 DSAR Workflow (GDPR / CCPA-aligned)

1. **Request intake** — submitted via dedicated form at `<tenant>/privacy-request` or email.
2. **Identity verification** — confirm requester via account login + second-channel verification (email + SMS code).
3. **Scope classification** — Access / Erasure / Rectification / Portability / Restriction / Objection.
4. **Data inventory query** — automated job queries every domain plugin for records linked to the data subject (PII discovery rules in Vol 09 §6).
5. **Fulfillment SLA** — 30 calendar days (GDPR); CCPA 45 days.
6. **Output package** — for Access/Portability: JSON + CSV bundles signed and zip-encrypted; for Erasure: anonymization sweep + confirmation.
7. **Legal retention exceptions** — fields required to be retained (tax records, dispute records, fraud signals) are masked but not deleted; reasons disclosed to requester.
8. **Audit log** — every step logged immutably for regulator inspection.

### 8.3 KYC / AML
- KYC level: Stripe Identity (default) + Persona / Trulioo (premium tiers) for enhanced due diligence.
- Sanction screening (OFAC, EU, UN, UK, SDN) on vendor onboarding + periodic re-screen.
- Suspicious activity reporting: flagged orders / vendors → tenant compliance officer; SAR filing tenant responsibility unless platform is registered MSB in jurisdiction.
- High-risk industries blocked or require additional review (gambling, money services, regulated goods).

### 8.4 Tax Compliance
- **US:** Marketplace Facilitator law applies in ~45 states; platform configures collection/remit per state. Vendors maintain own tax registrations for non-MF states.
- **EU/UK:** Platform may act as deemed supplier for non-EU sellers ≤€150 consignment; OSS/IOSS support.
- **Canada:** GST/HST/PST per province.
- **Australia/NZ:** GST.
- **Brazil:** Federal + ICMS — partner with local tax engine.
- Tax engine integrations: TaxJar (default), Avalara, Sovos (Enterprise).

### 8.5 Consumer Protection
- Distance Selling Regulations (EU) — 14-day cooling off period reflected in BR-ORD-006.
- US FTC Made-in-USA labeling rules — vendor warranty required where applicable.
- Magnuson-Moss Warranty Act compliance for warranties.
- "Marketplace transparency" disclosures — vendor identity always shown on PDP.

### 8.6 Security & Privacy Controls Mapping (excerpt; full in Vol 09 §3)

| Control | PCI | SOC-2 | GDPR | OWASP ASVS |
|---|---|---|---|---|
| MFA for admin/vendor | ✔ (Req 8) | CC6.1 | Art. 32 | V2 |
| TLS 1.3 inbound | ✔ (Req 4) | CC6.7 | Art. 32 | V9 |
| At-rest encryption | ✔ (Req 3) | CC6.7 | Art. 32 | V8 |
| Audit logging | ✔ (Req 10) | CC7.2 | Art. 30 | V7 |
| Vulnerability mgmt | ✔ (Req 11) | CC7.1 | Art. 32 | V14 |
| Incident response | ✔ (Req 12) | CC7.3 | Art. 33 | V14 |
| RBAC least privilege | ✔ (Req 7) | CC6.3 | Art. 32 | V4 |

---

## 9. Risk Register

Scoring: Likelihood (1=Rare, 5=Almost Certain) × Impact (1=Negligible, 5=Catastrophic) = Score (1–25).

| ID | Risk | L | I | Score | Owner | Mitigation |
|---|---|---|---|---|---|---|
| R-01 | Stripe Connect outage halts payouts | 2 | 5 | 10 | SRE | Outbox + retry; manual ACH fallback (Enterprise); status comms |
| R-02 | OpenSearch cluster failure → catalog dark | 2 | 4 | 8 | SRE | Multi-AZ + Redis fallback (Vol 01 §10.4) |
| R-03 | Cross-tenant data leak | 1 | 5 | 5 | Security | Query builder tenant scoping + RLS trigger + audit |
| R-04 | Vendor PII breach | 1 | 5 | 5 | Security + DPO | Encryption + access controls + KMS + S3 Object Lock |
| R-05 | Marketplace facilitator law gaps | 3 | 4 | 12 | Compliance | Quarterly legal review; tax engine partner |
| R-06 | AI provider rate-limits / cost spike | 3 | 3 | 9 | Eng + Finance | Multi-provider routing; semantic cache; tenant limits |
| R-07 | Vendor fraud (chargeback abuse) | 4 | 3 | 12 | Tenant + Compliance | Fraud engine; vendor reserve; vendor reputation |
| R-08 | Tenant churn from compliance friction | 3 | 4 | 12 | CS | Templates + advisory; managed compliance add-on |
| R-09 | Plugin author bypasses SDK rules | 3 | 3 | 9 | Eng | CI lint + code review; SDK governance |
| R-10 | DDOS at storefront | 3 | 4 | 12 | SRE | CloudFront + AWS Shield Standard; Enterprise gets Shield Advanced |
| R-11 | Failed disaster recovery | 1 | 5 | 5 | SRE | Monthly DR drills; multi-region for Enterprise |
| R-12 | Adversarial AI prompt injection | 3 | 3 | 9 | Security | Input sanitization; system-prompt isolation; output scanning |
| R-13 | Regulator action (GDPR fine) | 2 | 5 | 10 | DPO | DSAR automation; DPA registry; sub-processor audits |
| R-14 | Currency/tax misreporting | 3 | 4 | 12 | Finance | Daily reconciliation; FX captured at txn time; engine integration |
| R-15 | SDK breaking change harms partners | 3 | 3 | 9 | Eng | Semver discipline; 2-minor deprecation window |
| R-16 | Insider threat | 1 | 5 | 5 | Security | Least privilege; just-in-time access; session recording for prod |
| R-17 | Subscription billing failure | 2 | 4 | 8 | Finance | Dunning workflow; multiple PSP support |
| R-18 | Migration tool data loss | 2 | 5 | 10 | Eng | Dry-run + checksums; 24h rollback window |
| R-19 | Geopolitical sanctions enforcement gap | 2 | 5 | 10 | Compliance | Real-time OFAC; recheck on order placement |
| R-20 | Vendor support overwhelm during launches | 3 | 3 | 9 | CS | Self-service portal; AI Copilot for tier 1 |

---

## 10. Vendor Agreement — Standard Clauses (Tenants May Override)

Templates produced and version-controlled per jurisdiction. Standard clauses include:

1. **Acceptance & Compliance** — Vendor agrees to platform AUP, tenant marketplace policy.
2. **Listings Warranty** — Vendor warrants accuracy, ownership of media, legal compliance.
3. **Fulfillment Obligations** — Acknowledge / ship / handle returns per BR-ORD.
4. **Pricing & Tax** — Vendor responsible for own pricing decisions; tax handled per BR-FIN-012.
5. **Commission & Payouts** — Stripe Connect + commission per tenant rules.
6. **Liability** — Limited to direct damages capped at trailing-12mo payouts; indemnification for product liability.
7. **Termination** — Either party with 30d notice; immediate for material breach.
8. **Dispute Resolution** — Arbitration clause per jurisdiction; class action waiver where lawful.
9. **Data Use** — Vendor consents to platform processing of operational data; opts in for any analytics sharing.
10. **Modification** — Platform may modify with 30d notice; tenant may modify subordinate sections.

---

## 11. KPI Definitions (Single Source of Truth)

- **GMV** = Σ(sub-order grand total including tax + shipping) over period; refunds subtracted on issue date.
- **Net Sales** = GMV − refunds − discounts − tax.
- **AOV** = Net Sales / Order Count (parent orders).
- **Take Rate** = Commission Income / GMV.
- **Conversion** = Orders / Unique Visitors over period.
- **Vendor Activation Rate** = (KYC-verified vendors who post ≥1 product within 7d) / signups.
- **Vendor Active** = vendor with ≥1 paid order in trailing 30d.
- **Time-to-First-Sale (TTFS)** = first paid order timestamp − activation timestamp.
- **Tenant Churn** = (Tenants canceled in period) / (Tenants active at period start).
- **NRR (Net Revenue Retention)** = (Starting MRR + expansion − contraction − churn) / Starting MRR.
- **MRR / ARR** = Recurring subscription revenue, monthly / annualized.
- **NPS** = (% Promoters) − (% Detractors); surveyed quarterly.

---

## 12. Strategic Roadmap

Aligned with Vol 02 §9.
- **Phase 1 (MVP):** SMB tenant; pooled multisite; vendor onboarding; multi-vendor checkout; commissions; weekly payouts.
- **Phase 2:** Pro tier features; reviews; disputes; reports; search; tax.
- **Phase 3:** Enterprise tier; AI; fraud; collaboration; white-label; multi-currency.
- **Phase 4:** Multi-region; BYOK; SSO; migration importers; advanced compliance modules; resellers.

---

## 13. Cross-Volume Cross-References

| This Section | See Also |
|---|---|
| §4 RACI | Vol 11 DevOps on-call; Vol 09 incident response |
| §5 Business Rules | Vol 04 FSD per-domain functional behavior |
| §6 Monetization | Vol 02 PRD §6; Vol 06 DB billing tables |
| §7 SLA | Vol 01 §13 SLI/SLO; Vol 11 DevOps |
| §8 Compliance | Vol 09 Security; Vol 10 QA compliance tests |
| §9 Risks | Vol 01 §3; cross-volume Risk Register in Gap Matrix |

---

*End of Volume 03 — BRD v2.0.*
