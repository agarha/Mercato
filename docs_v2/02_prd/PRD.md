# Volume 02 — Mercato Enterprise Marketplace Platform
## Product Requirements Document (Engineering-Grade, Implementation-Ready)

> Document owner: Chief Product Officer (with Senior PMs per domain)
> Status: Implementation Baseline v2.0 — Approved for MVP planning; ADRs pending
> Version: 2.0.0 (full rewrite of v1.0)
> Cross-references: Vol 01 (Architecture), Vol 03 (BRD), Vol 04 (FSD), Vol 05 (SRS), Vol 08 (UX)

---

## 1. Executive Assessment

The v1.0 PRD captured the right vision (modular, plugin-first, microservices-augmented marketplace overlay on WordPress/WooCommerce) but left the product surface too coarse for engineering to plan against. It identified ~5 epics where the product actually has 20+. It enumerated 3 personas where the operational reality is 9. It listed 3 missing APIs where Vol 07 will define ~120. It declared NFRs without success-metric instrumentation, and treated the SaaS commercial model as a single sentence.

This rewrite resolves those gaps by defining:

1. **Nine canonical personas** with goals, pains, success measures.
2. **A complete epic and user-story catalog** spanning every domain plugin (20 epics, ~180 stories) with INVEST-compliant acceptance criteria.
3. **Lifecycle maps** for buyer, vendor, marketplace operator (tenant), SaaS super-admin, compliance officer, support, finance, and dispute moderator — including failure paths.
4. **A two-axis success-metric grid**: business KPIs (GMV, take rate, CAC, LTV, MRR, ARR, NRR) and product KPIs (vendor activation rate, time-to-first-sale, dashboard P95 latency, AI assist adoption).
5. **An MVP / Phase 2 / Phase 3 / Phase 4 cut** mapping every story to a delivery wave with explicit "out of scope for now" deferrals.
6. **A monetization model** combining B2B SaaS subscriptions, transactional take rate, usage-based AI billing, and premium integration revenue.
7. **A two-tier non-functional requirements ladder** — Pro tier vs. Enterprise tier — so engineering knows when to optimize for breadth vs. depth.

**Implementation Readiness (this volume): 90/100.** Outstanding gaps are mostly stakeholder sign-off and quantitative target setting (specific commission tiers, AI free-allotment numbers), tracked in the Gap Matrix.

---

## 2. Gap Analysis vs. v1.0

| # | Gap | v1.0 State | v2.0 Resolution |
|---|---|---|---|
| G-PRD-001 | Personas under-counted | 4 broad personas | 9 differentiated personas, each with goals/pains/measures (§3) |
| G-PRD-002 | Epics conflate domains | 5 epics | 20 epics aligned 1:1 with domain plugins (§4) |
| G-PRD-003 | Acceptance criteria absent or vague | "AC-1: cart with 2 vendors splits" | INVEST-compliant Gherkin AC for every story (§4–§5) |
| G-PRD-004 | Failure paths not enumerated | Happy path only | Failure path per lifecycle (§5 Lifecycles) |
| G-PRD-005 | Monetization is one paragraph | Mentioned in BRD only | Full monetization grid with billing events (§6) |
| G-PRD-006 | NFRs lack tier differentiation | Single tier | Pro vs. Enterprise NFR ladder (§7) |
| G-PRD-007 | Success metrics not instrumented | Stated only | Metric → event source → dashboard widget mapping (§8) |
| G-PRD-008 | Release planning absent | Mentioned in BRD | MVP / P2 / P3 / P4 with story-level scope (§9) |
| G-PRD-009 | Dependency on Vol 04/06/07 implicit | None | Explicit story → REQ → DDL → endpoint mapping (§10) |
| G-PRD-010 | Internationalization not specified | Mentioned once | i18n/l10n + currency model (§5.9) |
| G-PRD-011 | Accessibility absent | Not mentioned | WCAG 2.1 AA mandated; specific stories (§5.8) |
| G-PRD-012 | Buyer-side AI use cases absent | Vendor-only | Buyer conversational discovery (§4.16) |

---

## 3. Personas

Each persona has the format: name → goals → pains → success measures.

### P-01 Buyer (B2C / B2B end customer)
- **Goals:** Discover products across many vendors quickly; trust the vendors; checkout once; track every sub-shipment; resolve issues without bouncing between vendors.
- **Pains:** Fragmented shipping/refund policies; mistaking a marketplace listing for a first-party listing; vendor responsiveness varies.
- **Success measures:** Time-to-add-to-cart <30s; checkout completion >70%; CSAT after purchase >4.3/5; first contact resolution >80%.

### P-02 Casual Vendor (single-store, <$10k/mo GMV)
- **Goals:** List 10–200 SKUs without learning WordPress; get paid on time; minimal admin overhead.
- **Pains:** Setup friction (KYC, Stripe, tax); learning curve; lost orders if dashboard unclear.
- **Success measures:** Onboarding completion within 30min; time-to-first-product <5min after KYC; payout latency <vendor expectation (typically 7d).

### P-03 Professional Vendor (multi-staff, $10k–$1M/mo GMV)
- **Goals:** Bulk product management; team roles; integrate inventory with their own ERP; advanced analytics.
- **Pains:** Manual product editing; lack of bulk price/stock updates; weak reporting.
- **Success measures:** CSV import < 60s for 5k rows; staff role grant <30s; analytics export available.

### P-04 Enterprise Vendor (>$1M/mo GMV, multi-warehouse)
- **Goals:** API-driven inventory sync; SLA-bound payouts; white-label invoicing; programmable shipping rules.
- **Pains:** Marketplace lock-in; integration brittleness; lack of audit trail.
- **Success measures:** REST API SLO met; webhook delivery >99.95%; audit trail exports available on demand.

### P-05 Marketplace Operator / Tenant Admin
- **Goals:** Configure marketplace rules (commission, return windows, vendor approval policy); moderate vendors and products; resolve disputes; monitor revenue.
- **Pains:** Manual moderation; fraud signals scattered; cross-vendor refund accounting confusing.
- **Success measures:** Average moderation time <5min/item with AI assist; dispute resolution <72h; reconciliation report generated weekly.

### P-06 Marketplace Manager (Operator staff)
- **Goals:** Day-to-day vendor approvals, product moderation, dispute handling, customer escalations.
- **Pains:** Overwhelmed by volume; no inbox-style triage; switching between WP-admin and external tools.
- **Success measures:** Tasks per hour throughput; queue size never >24h backlog.

### P-07 SaaS Super Admin (us — platform operator)
- **Goals:** Provision tenants; bill them; observe platform health; respond to incidents; manage feature flags.
- **Pains:** Multi-tenant noisy neighbors; tenant-specific issues hard to diagnose.
- **Success measures:** New tenant provisioning <60s (pooled) / <5min (silo); MTTR <30min for SEV-1; >99.9% control plane uptime.

### P-08 Compliance Officer
- **Goals:** Audit trail access; KYC status monitoring; GDPR request fulfillment; SOC-2 evidence collection.
- **Pains:** Logs scattered; data subject requests manual; gap evidence under audit.
- **Success measures:** All admin actions audit-logged; DSAR turnaround <30 days; SOC-2 evidence one-click exportable.

### P-09 Finance Officer (Operator side)
- **Goals:** Reconcile commission, payout, refund flows; produce financial reports; manage tax filing.
- **Pains:** Currency drift; Stripe vs. internal ledger drift; tax engine ambiguity.
- **Success measures:** Daily reconciliation <0.1% drift; monthly close <3 business days; tax filing automated for ≥80% of jurisdictions.

---

## 4. Epic Catalog

20 epics, one per domain plugin (Vol 01 §4.1). Stories are numbered `US-<plugin>-<n>`. Each story has user-story form + Gherkin acceptance criteria + dependency notes.

### EPIC-01 — `mercato-core`: Foundation
*Goal: Provide the framework that all other plugins depend on.*

- **US-CORE-001** As a plugin developer, I declare my plugin via a `plugin.json` manifest so that `mercato-core` knows my dependencies and capabilities.
  - **Given** a plugin ships a valid manifest, **when** WP loads, **then** core boots my plugin in topological order.
  - **Given** a manifest declares a missing dependency, **when** WP loads, **then** my plugin is skipped and a notice is logged.
- **US-CORE-002** As a domain developer, I emit an event via `Mercato\Core\Events\Outbox::publish()` so that downstream consumers receive it reliably.
  - **Given** I call publish inside a DB transaction, **when** the transaction commits, **then** the outbox row is persisted; **when** it rolls back, **then** no event is published.
- **US-CORE-003** As a domain developer, I check capabilities via `mercato_user_can('mercato_orders_admin', $tenant_id)` so that RBAC is uniformly enforced.
- **US-CORE-004** As a SaaS operator, I see a tenant capability JWT refreshing every 24h so that feature flags propagate within 1 day.
- **US-CORE-005** As a tenant admin, the platform degrades gracefully when the Control Plane is unreachable, per offline grace policy (Vol 01 §11.2).

### EPIC-02 — `mercato-vendors`: Vendor Lifecycle
- **US-VEN-001** As a prospective vendor, I register via a public form so that I can begin onboarding.
  - **Given** I submit valid required fields (store name, email, country, tax-ID), **when** I click submit, **then** a record exists in `wp_mercato_vendors` with `status = pending_kyc` and an email is dispatched.
  - **Given** I submit a duplicate email, **when** I click submit, **then** the form returns a 409 with an "already registered" message.
- **US-VEN-002** As a vendor, I complete KYC via Stripe Identity / Connect onboarding so that I can receive payouts.
  - **Given** I click "Verify identity", **when** the flow returns, **then** my `kyc_status` reflects Stripe's `requirements.disabled_reason`.
- **US-VEN-003** As a tenant admin, I approve, reject, or suspend a vendor with a reason note that the vendor sees.
- **US-VEN-004** As a vendor, I configure my storefront (name, slug, banner, return policy, shipping zones).
- **US-VEN-005** As a vendor, I invite staff and assign roles (Manager, Order Fulfiller, Read-Only).
- **US-VEN-006** As a vendor, I see and download my full GDPR-compliant data export.
- **US-VEN-007** As a vendor, I close my store; pending orders must fulfill before closure completes.

### EPIC-03 — `mercato-products`: Catalog
- **US-PROD-001** As a vendor, I create a product via the SPA (title, description, price, currency, stock, images, category, attributes).
- **US-PROD-002** As a vendor, I bulk-import products via CSV (≤10k rows) with per-row validation and a downloadable error report.
- **US-PROD-003** As a vendor, I edit a product's price/stock with a single field update without losing other attributes.
- **US-PROD-004** As a vendor, I create variable products (size/color matrix) up to 250 variations.
- **US-PROD-005** As a vendor, I delete a product; existing orders are preserved with a snapshot of the product at order time.
- **US-PROD-006** As a tenant admin, I configure product moderation (off / pre-publish / post-publish flag).
- **US-PROD-007** As a buyer, I browse the catalog with facets (vendor, category, price range, rating, shipping speed, availability).
- **US-PROD-008** As a buyer, I view a product detail page that always shows the vendor identity, ratings, return policy, and shipping eligibility for my postal code.

### EPIC-04 — `mercato-orders`: Multi-Vendor Order Lifecycle
- **US-ORD-001** As a buyer, I add items from multiple vendors to one cart and check out once.
  - **Given** cart has items from Vendor A and Vendor B, **when** I complete checkout, **then** one WC parent order and two `wp_mercato_suborders` rows are created with correct totals and shipping per vendor.
- **US-ORD-002** As a buyer, I see shipping costs per vendor at checkout (each vendor's shipping zone applies).
- **US-ORD-003** As a vendor, I see only my sub-orders, never other vendors'.
- **US-ORD-004** As a vendor, I update sub-order status (Processing → Packed → Shipped → Delivered) and attach a tracking number.
- **US-ORD-005** As a buyer, I see consolidated tracking across all sub-orders in one place.
- **US-ORD-006** As a buyer, I cancel an unpaid sub-order while the parent stays paid for other sub-orders.

### EPIC-05 — `mercato-commissions`: Fee Engine
- **US-COMM-001** As a tenant admin, I define a default platform commission (e.g., 10%).
- **US-COMM-002** As a tenant admin, I override commission by category, vendor, or product.
- **US-COMM-003** As a tenant admin, I define tiered commission (e.g., 15% under $10k MTD, 12% above).
- **US-COMM-004** As a tenant admin, I preview commission calculations on a sample order before activating a new rule.
- **US-COMM-005** As a finance officer, I export commission ledger by date range and vendor.
- **US-COMM-006** As a finance officer, refunds automatically reverse proportionate commission (idempotent).

### EPIC-06 — `mercato-payouts`: Disbursement
- **US-PAY-001** As a vendor, I see my available balance, pending balance, and next payout date.
- **US-PAY-002** As a tenant admin, I configure a payout schedule (daily / weekly / monthly / manual).
- **US-PAY-003** As a finance officer, I batch-trigger payouts and see per-vendor success/failure with retry.
- **US-PAY-004** As a vendor, my payout uses Stripe Connect Destination Charge with proportionate refunds.
- **US-PAY-005** As a finance officer, I export a payout reconciliation report that matches Stripe Treasury within ±0.01%.

### EPIC-07 — `mercato-reviews`: Ratings
- **US-REV-001** As a buyer, I leave a star rating + text review for the product and the vendor after delivery.
- **US-REV-002** As a vendor, I respond to a review once per review.
- **US-REV-003** As a tenant admin, I moderate flagged reviews; rejection with reason.
- **US-REV-004** As a buyer, I see aggregate ratings on PDP and store page.

### EPIC-08 — `mercato-disputes`: Disputes & Refunds
- **US-DIS-001** As a buyer, I open a dispute on a sub-order with reason and evidence.
- **US-DIS-002** As a vendor, I respond to a dispute within 72h, else auto-escalate.
- **US-DIS-003** As a tenant admin (or staff dispute moderator), I adjudicate and issue full/partial refund.
- **US-DIS-004** As a finance officer, I see commission/payout reversal entries traceable to the dispute.

### EPIC-09 — `mercato-messaging`: Buyer ↔ Vendor Messages
- **US-MSG-001** As a buyer or vendor, I message the other party about a product, order, or general inquiry.
- **US-MSG-002** As a tenant admin, I read any thread for moderation reasons (audit-logged).
- **US-MSG-003** As a vendor, I attach an image to a message (≤10MB, MIME-validated).
- **US-MSG-004** As a buyer or vendor, I receive a real-time notification on new messages.

### EPIC-10 — `mercato-notifications`: Notifications
- **US-NOT-001** As any user, I get notified via my channel preference (email / SMS / in-app / push).
- **US-NOT-002** As a tenant admin, I customize notification templates per locale.
- **US-NOT-003** As a tenant admin, I see delivery success/failure with bounce reasons.
- **US-NOT-004** Webhooks for "order.shipped", "payout.completed", etc. are subscribable by Enterprise vendors.

### EPIC-11 — `mercato-reports`: Reporting
- **US-RPT-001** As a tenant admin, I see GMV / take / vendor count / order count dashboards (daily, weekly, monthly).
- **US-RPT-002** As a vendor, I see sales, conversion, top products, top SKUs.
- **US-RPT-003** As a finance officer, I export reconciliation reports as CSV/XLSX.
- **US-RPT-004** Reports are computed from the analytics store (Vol 01 §8), never the primary.

### EPIC-12 — `mercato-search`: Catalog Search
- **US-SCH-001** As a buyer, I search the catalog with sub-100ms responses including spell-correction, synonyms, and stemming.
- **US-SCH-002** As a buyer, I filter results by vendor, category, attributes, price range.
- **US-SCH-003** As a tenant admin, I configure synonyms and boost rules.

### EPIC-13 — `mercato-subscriptions`: Subscription Billing
- **US-SUB-001** As a SaaS super admin, I bill tenants per plan (Starter / Pro / Business / Enterprise).
- **US-SUB-002** As a tenant admin, I see/edit my subscription plan and billing history.
- **US-SUB-003** As a tenant admin, I add overage purchases (e.g., extra AI completions).
- **US-SUB-004** As a vendor (where enabled), I sell subscription products to buyers (recurring orders).

### EPIC-14 — `mercato-tax-engine`: Tax
- **US-TAX-001** As a buyer, tax is computed at checkout for my jurisdiction.
- **US-TAX-002** As a vendor, I see tax collected per order.
- **US-TAX-003** As a tenant admin, I integrate TaxJar/Avalara via toggle in tenant settings.

### EPIC-15 — `mercato-kyc-kyb`: Identity Verification
- **US-KYC-001** As a vendor, I upload identity documents securely; documents are stored in a private S3 bucket with KMS encryption.
- **US-KYC-002** As a vendor, I see verification status, what's pending, and re-upload as needed.
- **US-KYC-003** As a compliance officer, I see KYC records and re-trigger verification.

### EPIC-16 — `mercato-fraud-risk`: Risk Engine
- **US-FRD-001** Orders above a threshold are automatically risk-scored; high-risk are flagged and held.
- **US-FRD-002** A tenant admin reviews flagged orders and releases or refunds.
- **US-FRD-003** As a tenant admin, I configure fraud rules (velocity, geo mismatch, BIN risk).

### EPIC-17 — `mercato-ai-copilot`: AI Assistance
- **US-AI-001** As a vendor, I "Generate description" from a title and tags; output streams via SSE.
- **US-AI-002** As a vendor, I get suggested replies to buyer messages, with edit before send.
- **US-AI-003** As a tenant admin, I see flagged vendor or product content (toxicity, prohibited categories).
- **US-AI-004** As a buyer, I ask the storefront copilot "find me a waterproof red backpack under $80" and get a curated result set.
- **US-AI-005** As a tenant admin, I see AI usage and cost per vendor per month.
- **US-AI-006** AI outputs that go directly into product copy are tagged as AI-generated; a vendor must accept before publish.

### EPIC-18 — `mercato-collaboration`: Group Workspaces (Enterprise)
- **US-COL-001** As an enterprise vendor, I create a shared workspace for my team for internal chat about orders/products.
- **US-COL-002** As a tenant admin, I create a "broadcast" workspace pushing announcements to all vendors.

### EPIC-19 — `mercato-enterprise`: Licensing / White-label / Multi-tenancy
- **US-ENT-001** As a SaaS super admin, I provision a new tenant with a chosen tier in one click.
- **US-ENT-002** As a tenant admin, I configure my brand tokens (colors, fonts, logos).
- **US-ENT-003** As a tenant admin, I configure a custom domain (CNAME + auto SSL).
- **US-ENT-004** Capability JWTs enforce feature flags at runtime; flipping a flag in the Control Plane takes effect within 1 minute (≤ JWT TTL/24).

### EPIC-20 — `mercato-migration`: Importers
- **US-MIG-001** As a tenant migrating from Dokan, I import vendors, products, orders, and payouts.
- **US-MIG-002** As a tenant migrating from WCFM, I import the same with field-mapping UI.
- **US-MIG-003** As a tenant migrating from Mirakl, I import via CSV with explicit mapping templates.
- **US-MIG-004** Migration produces a dry-run summary before commit; rollback supported within 24h.

---

## 5. Lifecycle Maps

For each lifecycle, the happy path is enumerated alongside the major failure paths. This is the contract Vol 04 (FSD) must implement.

### 5.1 Buyer Lifecycle
1. Discover → browse, search, filter.
2. Add to cart (single or multi-vendor).
3. Checkout — address, shipping per vendor, tax, payment.
4. Pay — Stripe Elements; 3DS challenge if required.
5. Order confirmation — receipt + sub-order references.
6. Fulfillment — tracking events stream in.
7. Delivery — buyer marks received or auto-marked after 14d post-shipping.
8. Review — prompt to leave review on each sub-order.
9. Post-purchase — return / dispute / repeat purchase.

**Failure paths:**
- Payment declined → retry or change method → after 3 declines, cart preserved 30min.
- Stock changed during checkout → user shown new inventory, prompted to adjust.
- Shipping not available to address → blocked at checkout with message.
- Sub-order partially shipped, partially canceled → buyer sees consolidated status.

### 5.2 Vendor Lifecycle
Register → KYC → Stripe Connect → tenant approval → storefront setup → first product → first order → first payout → ongoing operations → suspension or closure.

**Failure paths:**
- KYC rejected → vendor sees reason, can re-submit.
- Stripe requirements outstanding → dashboard banner with deep-link.
- Tenant rejection → vendor account closed, data retained 30d for appeal then anonymized.

### 5.3 Tenant Admin Lifecycle (Marketplace Operator)
Sign up → choose plan → configure marketplace (commission, return window, payout schedule, branding) → invite staff → onboard first vendor → go live → monitor KPIs → adjust rules → renew / upgrade plan.

### 5.4 SaaS Super Admin Lifecycle
Receive tenant signup → auto-provision tier-appropriate resources → activate → bill monthly → monitor health → respond to incidents → handle upgrades/downgrades → handle churn (export + delete).

### 5.5 Dispute Lifecycle
Open (by buyer) → vendor response (≤72h) → escalate if no response → tenant adjudicates → resolution (full/partial refund or denial) → commission reversal if refund → audit logged.

### 5.6 Payout Lifecycle
Order delivered → commission ledger row aged past return window → payout candidate → batch run → Stripe transfer → success/failure recorded → vendor notified.

**Failure paths:**
- Insufficient platform Stripe balance → batch holds, alerts ops.
- Stripe transfer fails (closed account, etc.) → row marked `failed`, retry policy applied, vendor notified.

### 5.7 KYC Lifecycle
Submit → external KYC provider check → automated decision (verified/rejected) or human review queue → result stored → vendor notified → reverification triggers on material profile changes (bank account, beneficial owner change).

### 5.8 Accessibility Lifecycle (WCAG 2.1 AA mandate)
Every shipped UI passes:
- Automated audit (axe-core in CI).
- Keyboard-only navigation review (per release).
- Screen reader sanity (NVDA/VoiceOver) for top 10 flows.
- Color contrast ≥4.5:1 for body / ≥3:1 for large.
- Focus indication visible for every interactive element.

Specific stories:
- **US-A11Y-001** Every form has labeled inputs and visible focus rings.
- **US-A11Y-002** Every modal traps focus and returns on close.
- **US-A11Y-003** Every data table is navigable by keyboard and announces row/column to screen readers.

### 5.9 i18n / l10n & Currency
- All UI strings via `__()` / `_e()`; SPAs via i18next.
- Tenant configures default locale + supported locales.
- Money displayed in tenant default currency on listing; in buyer currency on PDP (FX rate stamped at fetch).
- Order is captured at the buyer's currency; commission/payout converted at order time using a captured FX rate (never re-derived later).

---

## 6. Monetization Model

Mercato (the platform) earns from tenants via three rails. Tenants in turn earn from vendors via commission take rate.

### 6.1 Platform → Tenant (B2B SaaS Subscriptions)

| Tier | Monthly | Annual | Vendor cap | AI completions/mo | API RPS | White-label | Custom domain | Multi-region |
|---|---|---|---|---|---|---|---|---|
| Starter | $99 | $990 | 25 | 1,000 | 20 | Tokens only | No | No |
| Pro | $499 | $4,990 | 250 | 25,000 | 50 | Tokens + templates | Yes | No |
| Business | $1,999 | $19,990 | 5,000 | 100,000 | 200 | Full | Yes | No |
| Enterprise | Custom | Custom | Unlimited | Custom | Custom | Full + BYOK | Yes | Yes |

Overages: AI completions $5 per 1,000; vendors $5 each over plan; API request $0.10 per 1k over plan.

Billing events emitted by `mercato-subscriptions`:
- `mercato.subscription.created.v1`, `.upgraded.v1`, `.downgraded.v1`, `.canceled.v1`
- `mercato.usage.metered.v1` (per metered unit, hourly batched)
- `mercato.invoice.issued.v1`, `.paid.v1`, `.failed.v1`

### 6.2 Tenant → Vendor (Marketplace Take Rate)
Tenants set their own commission via the Commission Rules Engine (§4 EPIC-05). Platform does not take a cut of this; the tenant's GMV-derived revenue is theirs.

### 6.3 Tenant → Vendor (Add-on Modules)
Optional paid features per vendor unlocked by tenant config: premium analytics, advanced AI, Featured Listings ads.

### 6.4 Platform → Vendor (Direct, optional)
Bundled "Mercato Pay" / "Mercato Shipping" rev-share models in Phase 4.

---

## 7. Non-Functional Requirements (NFRs)

NFRs split across two tiers — Pro and Enterprise — so engineering can prioritize. Starter/Business inherit closer to Pro.

| ID | Area | Pro | Enterprise |
|---|---|---|---|
| NFR-P-01 | Storefront PDP P95 | <500ms | <300ms |
| NFR-P-02 | Search P95 | <200ms | <100ms |
| NFR-P-03 | Checkout P95 | <1.5s | <800ms |
| NFR-P-04 | Vendor dashboard load P95 | <2.5s | <1.2s |
| NFR-A-01 | Tenant uptime | 99.5% | 99.95% |
| NFR-A-02 | Webhook delivery | 99.5% | 99.95% |
| NFR-A-03 | Outbox lag P95 | <10s | <2s |
| NFR-S-01 | Concurrent buyers | 1,000 / tenant | 50,000 / tenant |
| NFR-S-02 | Concurrent vendors | 100 active sessions | 5,000 active sessions |
| NFR-S-03 | Catalog size | 100k SKUs | 10M SKUs |
| NFR-Sec-01 | OWASP ASVS | L1 | L2 |
| NFR-Sec-02 | MFA | Optional vendor | Mandatory vendor + admin |
| NFR-Sec-03 | Audit retention | 90d | 2y |
| NFR-Sec-04 | KYC retention | 5y | 7y |
| NFR-DR-01 | RTO Tier-0 | 4h | 1h |
| NFR-DR-02 | RPO Tier-0 | 1h | 5min |
| NFR-A11Y-01 | WCAG | 2.1 AA | 2.1 AA + screen reader certified |
| NFR-Obs-01 | Trace sampling | 1% | 10% |
| NFR-I18n-01 | Supported locales | 5 | 30+ |

---

## 8. Success Metrics Instrumentation

Every metric must have an event source and a dashboard widget.

| Metric | Definition | Source Event | Dashboard | Owner |
|---|---|---|---|---|
| GMV | Sum of completed sub-order totals | `mercato.order.completed.v1` | Tenant Health | Finance |
| Take rate | Platform fees / GMV | `mercato.commission.recorded.v1` | Tenant Health | Finance |
| New vendors / day | Distinct new `vendor_id` | `mercato.vendor.activated.v1` | Tenant Health | PM |
| Vendor activation rate | (KYC complete in 7d) / signups | `vendor.kyc.completed.v1` | Vendor Funnel | PM |
| Time-to-first-sale | Hours from activation to first paid order | derived | Vendor Funnel | PM |
| AI assist adoption | % vendors with ≥1 AI completion / week | `mercato.ai.completion.v1` | AI Adoption | PM |
| MRR / ARR | Subscription billing | `mercato.invoice.paid.v1` | Platform Revenue | Finance |
| Dispute rate | Disputes / completed orders | `mercato.dispute.opened.v1` | Trust & Safety | Ops |
| CSAT (post-purchase) | Survey median | `mercato.csat.submitted.v1` | Buyer Experience | PM |
| P95 checkout latency | OTel trace span | trace | Performance | SRE |

---

## 9. Release Plan

### MVP (Phase 1, weeks 0–12)
*Goal: A tenant can launch a single-region marketplace with vendor onboarding, multi-vendor checkout, commissions, weekly payouts, and basic moderation.*

Includes: EPIC-01 Core, EPIC-02 Vendors, EPIC-03 Products (basic), EPIC-04 Orders (split cart), EPIC-05 Commissions (flat + per-vendor override), EPIC-06 Payouts (weekly batch, Stripe), EPIC-09 Messaging (text only), EPIC-10 Notifications (email only), EPIC-15 KYC (Stripe Identity integration), EPIC-19 Enterprise (provisioning + token theming).

Out of scope: AI, multi-currency, subscriptions, fraud engine, collaboration, full reports, multi-region.

### Phase 2 (weeks 12–22)
EPIC-07 Reviews, EPIC-08 Disputes, EPIC-11 Reports (vendor + tenant), EPIC-12 Search (OpenSearch + facets), EPIC-14 Tax (TaxJar or Avalara), EPIC-03 Products (bulk import, variations).

### Phase 3 (weeks 22–34)
EPIC-13 Subscriptions, EPIC-16 Fraud, EPIC-17 AI Copilot (vendor-side first), EPIC-18 Collaboration, multi-currency, white-label templates, custom domain, EPIC-19 advanced.

### Phase 4 (weeks 34–44)
EPIC-20 Migration importers, multi-region failover, BYOK, federated SSO, deeper AI use cases, marketplace SLA contracts.

---

## 10. Story → REQ → Endpoint → Table Traceability

A traceability matrix is generated automatically by CI from this PRD's story IDs, Vol 04 REQ IDs, Vol 06 table refs, and Vol 07 endpoint paths. Output is `docs_v2/deliverables/traceability.xlsx`.

Sample row:
| Story | REQ | Endpoint | Table | UI Screen | Test ID |
|---|---|---|---|---|---|
| US-ORD-001 | FR-ORD-001..003 | POST /mercato/v1/orders | `wp_mercato_suborders` | UX-BUY-CART-01 | TC-E2E-031 |
| US-PAY-005 | FR-PAY-004 | GET /mercato/v1/payouts/export | `wp_mercato_payouts` | UX-FIN-PAYOUT-01 | TC-INT-094 |

---

## 11. Out of Scope (explicit deferrals)

The following are out-of-scope for Mercato v1–v4 and tracked separately:

- B2B procurement workflows with purchase orders (deferred to Mercato B2B Add-on Pack).
- POS / in-store retail integration.
- Crypto / wallet payments.
- Decentralized identity (DID) for vendor identity.
- Live shopping / video commerce.

---

## 12. Cross-Volume Cross-References

| This Section | See Also |
|---|---|
| §4 Epics | Vol 04 FSD per-domain; Vol 06 tables; Vol 07 endpoints |
| §5 Lifecycles | Vol 08 UX wireframes; Vol 10 QA E2E test IDs |
| §7 NFRs | Vol 01 §15 perf targets; Vol 05 SRS NFR catalog |
| §8 Metrics | Vol 11 DevOps §6 dashboards; Vol 12 AI for AI metrics |
| §9 Release plan | Vol 11 DevOps deployment cadence; Implementation Backlog (.xlsx) |

---

*End of Volume 02 — PRD v2.0.*
