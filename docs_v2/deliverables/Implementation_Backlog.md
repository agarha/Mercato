# Mercato Implementation Backlog

> Engineering execution plan, 98 stories across 20 epics + cross-cutting.
> Mirror of [`Mercato_Implementation_Backlog.xlsx`](Mercato_Implementation_Backlog.xlsx) for GitHub viewing.

**Phase legend:** 🟢 P1 MVP · 🟡 P2 · 🟠 P3 · ⚪ P4

## Epic Summary

| Epic | Stories | MVP | P2 | P3 | P4 | Pts |
|---|---|---|---|---|---|---|
| EP-01 Core Framework | 7 | 7 | 0 | 0 | 0 | 52 |
| EP-02 Vendor Lifecycle | 7 | 4 | 2 | 1 | 0 | 58 |
| EP-03 Catalog | 8 | 3 | 5 | 0 | 0 | 83 |
| EP-04 Orders | 6 | 5 | 1 | 0 | 0 | 63 |
| EP-05 Commissions | 6 | 3 | 3 | 0 | 0 | 54 |
| EP-06 Payouts | 6 | 5 | 1 | 0 | 0 | 60 |
| EP-07 Reviews | 4 | 0 | 4 | 0 | 0 | 26 |
| EP-08 Disputes | 4 | 0 | 4 | 0 | 0 | 36 |
| EP-09 Messaging | 4 | 1 | 2 | 1 | 0 | 39 |
| EP-10 Notifications | 4 | 1 | 3 | 0 | 0 | 34 |
| EP-11 Reports | 3 | 0 | 3 | 0 | 0 | 26 |
| EP-12 Search | 3 | 0 | 2 | 1 | 0 | 29 |
| EP-13 Subscriptions | 2 | 0 | 0 | 2 | 0 | 29 |
| EP-14 Tax | 2 | 0 | 1 | 1 | 0 | 18 |
| EP-15 KYC | 3 | 2 | 1 | 0 | 0 | 26 |
| EP-16 Fraud | 3 | 0 | 0 | 3 | 0 | 34 |
| EP-17 AI | 6 | 0 | 0 | 6 | 0 | 73 |
| EP-18 Collab | 2 | 0 | 0 | 0 | 2 | 21 |
| EP-19 Enterprise | 5 | 3 | 0 | 2 | 0 | 63 |
| EP-20 Migration | 3 | 0 | 0 | 0 | 3 | 55 |
| EP-PLAT Compliance | 1 | 0 | 1 | 0 | 0 | 13 |
| EP-PLAT DR | 2 | 1 | 1 | 0 | 0 | 13 |
| EP-PLAT Observability | 2 | 2 | 0 | 0 | 0 | 21 |
| EP-PLAT Security | 3 | 3 | 0 | 0 | 0 | 29 |
| EP-PLAT a11y | 1 | 1 | 0 | 0 | 0 | 8 |
| EP-PLAT i18n | 1 | 0 | 1 | 0 | 0 | 13 |
| TOTAL | 98 | 41 | 35 | 17 | 5 | 976 |

## Phase Roadmap

| Phase | Window | Stories | Pts | Description |
|---|---|---|---|---|
| P1 MVP | week 0-12 | 41 | 395 | Core platform, vendor onboarding, multi-vendor checkout, commissions, weekly payouts, basic moderation. |
| P2 | week 12-22 | 35 | 304 | Reviews, disputes, reports, search, tax, bulk import, variations. |
| P3 | week 22-34 | 17 | 201 | Subscriptions, fraud, AI Copilot, collaboration, multi-currency, white-label. |
| P4 | week 34-44 | 5 | 76 | Migration importers, multi-region, BYOK, SSO, advanced AI. |

## Stories


### EP-01 Core Framework

| ID | Story | Phase | Pts | Plugin | FR | Status |
|---|---|---|---|---|---|---|
| US-CORE-001 | As a plugin developer, I declare my plugin via plugin.json | 🟢 P1 MVP | 8 | mercato-core | FR-CORE-001..003 | Ready |
| US-CORE-002 | As a domain developer, I emit events via outbox | 🟢 P1 MVP | 13 | mercato-core | FR-CORE-004 | Ready |
| US-CORE-003 | Capability check API | 🟢 P1 MVP | 8 | mercato-core | FR-CORE-006..007 | Ready |
| US-CORE-004 | Tenant JWT refresh | 🟢 P1 MVP | 8 | mercato-core, mercato-enterprise | FR-ENT-001..002 | Ready |
| US-CORE-005 | DB migration runner | 🟢 P1 MVP | 5 | mercato-core | FR-CORE-005 | Ready |
| US-CORE-006 | Idempotency-Key handling | 🟢 P1 MVP | 5 | mercato-core | FR-CORE-009 | Ready |
| US-CORE-007 | WC hook adapter | 🟢 P1 MVP | 5 | mercato-core | FR-CORE-001 | Ready |

### EP-02 Vendor Lifecycle

| ID | Story | Phase | Pts | Plugin | FR | Status |
|---|---|---|---|---|---|---|
| US-VEN-001 | Vendor public sign-up form | 🟢 P1 MVP | 8 | mercato-vendors | FR-VEN-001..003 | Ready |
| US-VEN-002 | Stripe Identity onboarding | 🟢 P1 MVP | 13 | mercato-vendors, mercato-kyc-kyb, mercato-stripe-connect | FR-VEN-004..006, FR-KYC-001..003 | Ready |
| US-VEN-003 | Tenant admin approve/reject/suspend | 🟢 P1 MVP | 8 | mercato-vendors | FR-VEN-005..007 | Ready |
| US-VEN-004 | Vendor configures storefront | 🟢 P1 MVP | 8 | mercato-vendors | FR-VEN-001 | Ready |
| US-VEN-005 | Vendor invites staff | 🟡 P2 | 8 | mercato-vendors | FR-VEN-010 | Backlog |
| US-VEN-006 | Vendor downloads data export | 🟡 P2 | 8 | mercato-vendors | FR-VEN-008 | Backlog |
| US-VEN-007 | Vendor closes store | 🟠 P3 | 5 | mercato-vendors | FR-VEN-009 | Backlog |

### EP-03 Catalog

| ID | Story | Phase | Pts | Plugin | FR | Status |
|---|---|---|---|---|---|---|
| US-PROD-001 | Vendor creates product via SPA | 🟢 P1 MVP | 13 | mercato-products | FR-PROD-001..002 | Ready |
| US-PROD-002 | CSV import 10k rows | 🟡 P2 | 13 | mercato-products | FR-PROD-005 | Backlog |
| US-PROD-003 | Edit price/stock without form | 🟡 P2 | 5 | mercato-products | FR-PROD-002 | Backlog |
| US-PROD-004 | Variable products up to 250 | 🟡 P2 | 13 | mercato-products | FR-PROD-004 | Backlog |
| US-PROD-005 | Soft-delete product | 🟢 P1 MVP | 5 | mercato-products | FR-PROD-003 | Ready |
| US-PROD-006 | Tenant moderation mode | 🟡 P2 | 13 | mercato-products, mercato-ai-copilot | FR-PROD-007 | Backlog |
| US-PROD-007 | Faceted catalog browse | 🟡 P2 | 13 | mercato-products, mercato-search | FR-PROD-007..008 | Backlog |
| US-PROD-008 | PDP shows vendor + ratings + ship-to | 🟢 P1 MVP | 8 | mercato-products | FR-PROD-008 | Ready |

### EP-04 Orders

| ID | Story | Phase | Pts | Plugin | FR | Status |
|---|---|---|---|---|---|---|
| US-ORD-001 | Multi-vendor checkout splits | 🟢 P1 MVP | 21 | mercato-orders | FR-ORD-001..002 | Ready |
| US-ORD-002 | Shipping rates per vendor | 🟢 P1 MVP | 13 | mercato-orders | FR-ORD-002 | Ready |
| US-ORD-003 | Vendor sees only own | 🟢 P1 MVP | 5 | mercato-orders | FR-ORD-004 | Ready |
| US-ORD-004 | Vendor updates status + tracking | 🟢 P1 MVP | 8 | mercato-orders | FR-ORD-005..006 | Ready |
| US-ORD-005 | Buyer consolidated tracking | 🟢 P1 MVP | 8 | mercato-orders | FR-ORD-006 | Ready |
| US-ORD-006 | Buyer cancels sub-order | 🟡 P2 | 8 | mercato-orders | FR-ORD-008 | Backlog |

### EP-05 Commissions

| ID | Story | Phase | Pts | Plugin | FR | Status |
|---|---|---|---|---|---|---|
| US-COMM-001 | Set platform default commission | 🟢 P1 MVP | 5 | mercato-commissions | FR-COMM-001..002 | Ready |
| US-COMM-002 | Category/vendor/product overrides | 🟢 P1 MVP | 13 | mercato-commissions | FR-COMM-001 | Ready |
| US-COMM-003 | Tier rules by MTD GMV | 🟡 P2 | 13 | mercato-commissions | FR-COMM-001 | Backlog |
| US-COMM-004 | Preview on sample order | 🟡 P2 | 5 | mercato-commissions | FR-COMM-007 | Backlog |
| US-COMM-005 | Export ledger CSV/XLSX | 🟡 P2 | 5 | mercato-commissions | FR-COMM-005 | Backlog |
| US-COMM-006 | Auto reversal on refund | 🟢 P1 MVP | 13 | mercato-commissions | FR-COMM-005..006 | Ready |

### EP-06 Payouts

| ID | Story | Phase | Pts | Plugin | FR | Status |
|---|---|---|---|---|---|---|
| US-PAY-001 | Show available/pending balance | 🟢 P1 MVP | 8 | mercato-payouts | FR-PAY-001 | Ready |
| US-PAY-002 | Tenant configures schedule | 🟢 P1 MVP | 5 | mercato-payouts | FR-PAY-002 | Ready |
| US-PAY-003 | Trigger batch payout | 🟢 P1 MVP | 13 | mercato-payouts | FR-PAY-002..006 | Ready |
| US-PAY-004 | Use Destination Charges | 🟢 P1 MVP | 13 | mercato-payouts | FR-PAY-004..005 | Ready |
| US-PAY-005 | Daily ±$0.01 | 🟢 P1 MVP | 13 | mercato-payouts | FR-PAY-007 | Ready |
| US-PAY-006 | Monthly vendor statement | 🟡 P2 | 8 | mercato-payouts | FR-PAY-008 | Backlog |

### EP-07 Reviews

| ID | Story | Phase | Pts | Plugin | FR | Status |
|---|---|---|---|---|---|---|
| US-REV-001 | Buyer leaves review | 🟡 P2 | 8 | mercato-reviews | FR-REV-001..002 | Backlog |
| US-REV-002 | Vendor responds once | 🟡 P2 | 5 | mercato-reviews | FR-REV-003 | Backlog |
| US-REV-003 | Admin moderates flagged | 🟡 P2 | 8 | mercato-reviews | FR-REV-004..005 | Backlog |
| US-REV-004 | Aggregate rating display | 🟡 P2 | 5 | mercato-reviews | FR-REV-005..006 | Backlog |

### EP-08 Disputes

| ID | Story | Phase | Pts | Plugin | FR | Status |
|---|---|---|---|---|---|---|
| US-DIS-001 | Buyer opens dispute | 🟡 P2 | 13 | mercato-disputes | FR-DIS-001..002 | Backlog |
| US-DIS-002 | Vendor responds 72h | 🟡 P2 | 5 | mercato-disputes | FR-DIS-003 | Backlog |
| US-DIS-003 | Tenant adjudicates | 🟡 P2 | 13 | mercato-disputes | FR-DIS-004 | Backlog |
| US-DIS-004 | Refund triggers commission reversal | 🟡 P2 | 5 | mercato-disputes | FR-DIS-006 | Backlog |

### EP-09 Messaging

| ID | Story | Phase | Pts | Plugin | FR | Status |
|---|---|---|---|---|---|---|
| US-MSG-001 | Buyer↔Vendor threads | 🟢 P1 MVP | 13 | mercato-messaging | FR-MSG-001..002 | Ready |
| US-MSG-002 | Admin read audit-logged | 🟡 P2 | 5 | mercato-messaging | FR-MSG-005 | Backlog |
| US-MSG-003 | Image attachments | 🟡 P2 | 8 | mercato-messaging | FR-MSG-004 | Backlog |
| US-MSG-004 | Detect & flag | 🟠 P3 | 13 | mercato-messaging, mercato-ai-copilot | FR-MSG-003 | Backlog |

### EP-10 Notifications

| ID | Story | Phase | Pts | Plugin | FR | Status |
|---|---|---|---|---|---|---|
| US-NOT-001 | Email/SMS/push/in-app | 🟢 P1 MVP | 13 | mercato-notifications | FR-NOT-001 | Ready |
| US-NOT-002 | Per-locale templates | 🟡 P2 | 8 | mercato-notifications | FR-NOT-002 | Backlog |
| US-NOT-003 | Bounce/complaint suppression | 🟡 P2 | 5 | mercato-notifications | FR-NOT-003 | Backlog |
| US-NOT-004 | Webhook delivery | 🟡 P2 | 8 | mercato-notifications | FR-NOT-004 | Backlog |

### EP-11 Reports

| ID | Story | Phase | Pts | Plugin | FR | Status |
|---|---|---|---|---|---|---|
| US-RPT-001 | GMV/take/vendors/orders | 🟡 P2 | 13 | mercato-reports | FR-RPT-001..002 | Backlog |
| US-RPT-002 | Vendor metrics | 🟡 P2 | 8 | mercato-reports | FR-RPT-001 | Backlog |
| US-RPT-003 | CSV/XLSX export | 🟡 P2 | 5 | mercato-reports | FR-RPT-004 | Backlog |

### EP-12 Search

| ID | Story | Phase | Pts | Plugin | FR | Status |
|---|---|---|---|---|---|---|
| US-SCH-001 | Sub-100ms search | 🟡 P2 | 13 | mercato-search | FR-SCH-001..002 | Backlog |
| US-SCH-002 | Facets | 🟡 P2 | 8 | mercato-search | FR-SCH-002 | Backlog |
| US-SCH-003 | Synonyms + boosts | 🟠 P3 | 8 | mercato-search | FR-SCH-003..004 | Backlog |

### EP-13 Subscriptions

| ID | Story | Phase | Pts | Plugin | FR | Status |
|---|---|---|---|---|---|---|
| US-SUB-001 | Tenant billing | 🟠 P3 | 21 | mercato-subscriptions | FR-SUB-001..004 | Backlog |
| US-SUB-003 | Overage purchases | 🟠 P3 | 8 | mercato-subscriptions | FR-SUB-004 | Backlog |

### EP-14 Tax

| ID | Story | Phase | Pts | Plugin | FR | Status |
|---|---|---|---|---|---|---|
| US-TAX-001 | TaxJar/Avalara at checkout | 🟡 P2 | 13 | mercato-tax-engine | FR-TAX-001..003 | Backlog |
| US-TAX-002 | Vendor tax view | 🟠 P3 | 5 | mercato-tax-engine | FR-TAX-005 | Backlog |

### EP-15 KYC

| ID | Story | Phase | Pts | Plugin | FR | Status |
|---|---|---|---|---|---|---|
| US-KYC-001 | Doc upload S3 private | 🟢 P1 MVP | 13 | mercato-kyc-kyb | FR-KYC-002 | Ready |
| US-KYC-002 | Verification status | 🟢 P1 MVP | 8 | mercato-kyc-kyb | FR-KYC-003 | Ready |
| US-KYC-003 | Compliance officer review | 🟡 P2 | 5 | mercato-kyc-kyb | FR-KYC-005 | Backlog |

### EP-16 Fraud

| ID | Story | Phase | Pts | Plugin | FR | Status |
|---|---|---|---|---|---|---|
| US-FRD-001 | Risk score per order | 🟠 P3 | 13 | mercato-fraud-risk | FR-FRD-001..002 | Backlog |
| US-FRD-002 | Admin queue | 🟠 P3 | 8 | mercato-fraud-risk | FR-FRD-003 | Backlog |
| US-FRD-003 | Tenant rules | 🟠 P3 | 13 | mercato-fraud-risk | FR-FRD-001 | Backlog |

### EP-17 AI

| ID | Story | Phase | Pts | Plugin | FR | Status |
|---|---|---|---|---|---|---|
| US-AI-001 | Generate desc with SSE | 🟠 P3 | 13 | mercato-ai-copilot | FR-AI-001..003 | Backlog |
| US-AI-002 | Suggested replies | 🟠 P3 | 13 | mercato-ai-copilot, mercato-messaging | FR-AI-001..003 | Backlog |
| US-AI-003 | Tenant moderation | 🟠 P3 | 13 | mercato-ai-copilot, mercato-products | FR-AI-001 | Backlog |
| US-AI-004 | Buyer discovery | 🟠 P3 | 21 | mercato-ai-copilot | FR-AI-001 | Backlog |
| US-AI-005 | Tenant usage + cost | 🟠 P3 | 8 | mercato-ai-copilot | FR-AI-004 | Backlog |
| US-AI-006 | AI-assisted badge | 🟠 P3 | 5 | mercato-ai-copilot | BR-AI-002 | Backlog |

### EP-18 Collab

| ID | Story | Phase | Pts | Plugin | FR | Status |
|---|---|---|---|---|---|---|
| US-COL-001 | Vendor shared workspace | ⚪ P4 | 13 | mercato-collaboration | FR-COL-001 | Backlog |
| US-COL-002 | Tenant broadcast | ⚪ P4 | 8 | mercato-collaboration | FR-COL-001 | Backlog |

### EP-19 Enterprise

| ID | Story | Phase | Pts | Plugin | FR | Status |
|---|---|---|---|---|---|---|
| US-ENT-001 | Provision tenant | 🟢 P1 MVP | 13 | mercato-enterprise | FR-ENT-001 | Ready |
| US-ENT-002 | Token config | 🟢 P1 MVP | 8 | mercato-enterprise | FR-ENT-003 | Ready |
| US-ENT-003 | Custom domain | 🟠 P3 | 13 | mercato-enterprise | FR-ENT-004 | Backlog |
| US-ENT-004 | Runtime feature flag enforcement | 🟢 P1 MVP | 8 | mercato-enterprise | FR-ENT-002 | Ready |
| US-ENT-005 | SAML/OIDC | 🟠 P3 | 21 | mercato-enterprise | FR-ENT-005 | Backlog |

### EP-20 Migration

| ID | Story | Phase | Pts | Plugin | FR | Status |
|---|---|---|---|---|---|---|
| US-MIG-001 | Import from Dokan | ⚪ P4 | 21 | mercato-migration | FR-MIG-001..005 | Backlog |
| US-MIG-002 | Import WCFM | ⚪ P4 | 21 | mercato-migration | FR-MIG-001..005 | Backlog |
| US-MIG-003 | Import CSV | ⚪ P4 | 13 | mercato-migration | FR-MIG-001..005 | Backlog |

### EP-PLAT Observability

| ID | Story | Phase | Pts | Plugin | FR | Status |
|---|---|---|---|---|---|---|
| US-PLAT-001 | OTel propagation | 🟢 P1 MVP | 13 | mercato-core | NFR-Obs-006 | Ready |
| US-PLAT-002 | Burn-rate alerts | 🟢 P1 MVP | 8 | SRE | NFR-Obs-001 | Ready |

### EP-PLAT Security

| ID | Story | Phase | Pts | Plugin | FR | Status |
|---|---|---|---|---|---|---|
| US-PLAT-003 | CSP nonces | 🟢 P1 MVP | 8 | Web | NFR-Sec-001 | Ready |
| US-PLAT-004 | MFA TOTP | 🟢 P1 MVP | 8 | mercato-core | NFR-Sec-002 | Ready |
| US-PLAT-005 | Audit log immutable | 🟢 P1 MVP | 13 | mercato-core | NFR-Sec-007 | Ready |

### EP-PLAT DR

| ID | Story | Phase | Pts | Plugin | FR | Status |
|---|---|---|---|---|---|---|
| US-PLAT-006 | PITR + snapshot copy | 🟢 P1 MVP | 8 | SRE | NFR-DR-01..02 | Ready |
| US-PLAT-007 | Monthly drill | 🟡 P2 | 5 | SRE | NFR-A-005..006 | Backlog |

### EP-PLAT a11y

| ID | Story | Phase | Pts | Plugin | FR | Status |
|---|---|---|---|---|---|---|
| US-PLAT-008 | axe-core in CI | 🟢 P1 MVP | 8 | Web | NFR-Use-001 | Ready |

### EP-PLAT i18n

| ID | Story | Phase | Pts | Plugin | FR | Status |
|---|---|---|---|---|---|---|
| US-PLAT-009 | ICU i18n | 🟡 P2 | 13 | Web | NFR-I18n-01 | Backlog |

### EP-PLAT Compliance

| ID | Story | Phase | Pts | Plugin | FR | Status |
|---|---|---|---|---|---|---|
| US-PLAT-010 | DSAR automation | 🟡 P2 | 13 | Compliance | §10.2 | Backlog |