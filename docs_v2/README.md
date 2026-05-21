# Mercato Enterprise Marketplace Platform — Documentation Suite v2.0

> **Status:** Implementation Baseline v2.0 — Approved for MVP planning; ADRs pending
> **Overall Implementation Readiness:** 91 / 100 (target; honest range 88–90 per external reviewers)
> **Owner:** Mercato Cross-Functional Delivery Organization
> **Last updated:** May 2026

---

## 📌 Read This First

**If you are an engineer about to start building, your first stop is [Volume 00 — MVP Implementation Cut](00_mvp_cut/MVP_Cut.md).** It bounds what is in scope for the first 12 weeks vs. the full target architecture.

The 12 volumes specify the **target state**. The MVP Cut specifies the **first slice**. Both are needed.

---

## 📚 Volume Index

| Vol | Document | Status | Lines | Last Reviewed |
|---|---|---|---|---|
| 🚀 [00 MVP Implementation Cut](00_mvp_cut/MVP_Cut.md) | **READ FIRST** | ✅ Baseline v2.0 | ~360 | 2026-05 |
| [01 Architecture Blueprint](01_architecture/Blueprint.md) | Target architecture | ✅ Baseline v2.0 | 728 | 2026-05 |
| └─ [ADR log](01_architecture/adr/) | Architecture decisions | 🟡 6 open | — | 2026-05 |
| [02 Product Requirements (PRD)](02_prd/PRD.md) | Personas, epics, stories | ✅ Baseline v2.0 | 446 | 2026-05 |
| [03 Business Requirements (BRD)](03_brd/BRD.md) | Rules, RACI, SLA, compliance | ✅ Baseline v2.0 | 509 | 2026-05 |
| [04 Functional Spec (FSD)](04_fsd/FSD.md) | Per-plugin behavior & state machines | ✅ Baseline v2.0 | 641 | 2026-05 |
| [05 Software Requirements (SRS)](05_srs/SRS.md) | FR + NFR catalog | ✅ Baseline v2.0 | 334 | 2026-05 |
| [06 Database & DDL](06_database/Database.md) | 45-table schema | ✅ Baseline v2.0 | 1085 | 2026-05 |
| └─ [Accounting Ledger Extension](06_database/Database_Accounting_Ledger.md) | Double-entry GL | ✅ Baseline v1.0 | ~450 | 2026-05 |
| [07 OpenAPI 3.1](07_openapi/OpenAPI.yaml) | REST API contract | ✅ Baseline v2.0 | 1507 | 2026-05 |
| [07 AsyncAPI 2.6](07_openapi/AsyncAPI.yaml) | Event catalog | ✅ Baseline v2.0 | 361 | 2026-05 |
| [08 UX & Wireframes](08_ux/Wireframes.md) | Design system + 110 screens | ✅ Baseline v2.0 | 470 | 2026-05 |
| [09 Security & Compliance](09_security/Security.md) | Threat model + controls | ✅ Baseline v2.0 | 474 | 2026-05 |
| [10 QA & UAT](10_qa/QA_Spec.md) | Test strategy + gates | ✅ Baseline v2.0 | 320 | 2026-05 |
| [11 DevOps & Infrastructure](11_devops/DevOps.md) | EKS, Aurora, observability, DR | ✅ Baseline v2.0 | 405 | 2026-05 |
| [12 AI Collaboration](12_ai/AI_Copilot.md) | AI service architecture + guardrails | ✅ Baseline v2.0 | 397 | 2026-05 |
| [13 WooCommerce/HPOS Compatibility](13_woocommerce_compat/WooCommerce_HPOS_Compat.md) | WC mapping + plugin conflicts | ✅ Baseline v1.0 | ~400 | 2026-05 |
| [14 Plugin Packaging Strategy](14_packaging/Plugin_Packaging.md) | One suite, internally modular | ✅ Baseline v1.0 | ~250 | 2026-05 |
| [15 Operational Runbooks](15_runbooks/) | Tenant offboarding, incident playbooks | ✅ Baseline v1.0 | — | 2026-05 |

## 🎯 Synthesis Deliverables

| Deliverable | Format | Purpose |
|---|---|---|
| [Gap Matrix](deliverables/Mercato_Gap_Matrix.xlsx) | .xlsx — 60 items, 4 sheets | Cross-volume gap tracking |
| [Gap Matrix (MD mirror)](deliverables/Gap_Matrix.md) | .md | GitHub-viewable mirror |
| [Implementation Backlog](deliverables/Mercato_Implementation_Backlog.xlsx) | .xlsx — 98 stories | Engineering execution plan |
| [Backlog (MD mirror)](deliverables/Implementation_Backlog.md) | .md | GitHub-viewable mirror |
| [Readiness Scorecard + Blueprint](deliverables/Mercato_Readiness_Scorecard_and_Blueprint.docx) | .docx | Executive sign-off |
| [Scorecard (MD mirror)](deliverables/Readiness_Scorecard.md) | .md | GitHub-viewable mirror |

---

## 🏗 Core Architectural Principles (Non-Negotiable)

1. **Modularity over monolith** — one `mercato-core` + 18 domain plugins + 9 integration adapters, behind a versioned SDK contract.
2. **Microservices-augmented** — control plane + AI + search + notification services offload high-velocity workloads from WordPress.
3. **100% security compliance posture** — PCI-DSS SAQ-A (Stripe Elements), GDPR + CCPA with automated DSAR, SOC-2 Type II, OWASP ASVS L2, zero-trust mTLS.
4. **Optimized infrastructure** — stateless WordPress on EKS, autoscaling, Aurora Global, Redis, CloudFront, IaC, GitOps, full observability triad.

Full non-negotiable list: [Vol 01 §16](01_architecture/Blueprint.md#16-non-negotiables-architectural-constraints).

---

## 🛠 How to Navigate by Role

- **Engineer building a plugin?** → [Vol 04 FSD](04_fsd/FSD.md) → [Vol 06 DDL](06_database/Database.md) → [Vol 07 OpenAPI](07_openapi/OpenAPI.yaml). Read [Vol 00](00_mvp_cut/MVP_Cut.md) to know if your plugin is MVP.
- **Product manager?** → [Vol 02 PRD](02_prd/PRD.md) → [Vol 00 MVP Cut](00_mvp_cut/MVP_Cut.md) → [Implementation Backlog](deliverables/Mercato_Implementation_Backlog.xlsx).
- **Designer?** → [Vol 08 UX](08_ux/Wireframes.md).
- **Security / Compliance?** → [Vol 09](09_security/Security.md) → [Vol 03 §8](03_brd/BRD.md).
- **DevOps / SRE?** → [Vol 11](11_devops/DevOps.md) → [Vol 15 Runbooks](15_runbooks/).
- **QA?** → [Vol 10](10_qa/QA_Spec.md). Annotate tests with `@PRD`, `@FR`, `@NFR`.
- **Finance / Accounting?** → [Vol 03 §5.4–§5.5](03_brd/BRD.md) → [Vol 06 Accounting Ledger Extension](06_database/Database_Accounting_Ledger.md).
- **Architect / decision-maker?** → [Vol 01](01_architecture/Blueprint.md) → [ADR log](01_architecture/adr/) → [Vol 13 WC Compat](13_woocommerce_compat/WooCommerce_HPOS_Compat.md).

---

## 🗓 Delivery Phases

| Phase | Window | Goal | Detail |
|---|---|---|---|
| MVP | weeks 0–12 | Single-tenant marketplace, multi-vendor checkout, weekly payouts | [Vol 00 MVP Cut](00_mvp_cut/MVP_Cut.md) |
| Phase 2 | weeks 12–22 | Pro tier features (reviews, disputes, search, tax, bulk) | [Vol 02 §9](02_prd/PRD.md) |
| Phase 3 | weeks 22–34 | Enterprise tier + AI Copilot + white-label | [Vol 02 §9](02_prd/PRD.md) |
| Phase 4 | weeks 34–44 | Multi-region, BYOK, SSO, migration importers | [Vol 02 §9](02_prd/PRD.md) |

---

## 🔁 Changelog

| Version | Date | Changes |
|---|---|---|
| 2.0.0 | 2026-05-21 | Complete documentation suite rewrite across 12 volumes; added Gap Matrix, Backlog, Scorecard deliverables |
| 2.0.1 | 2026-05-21 | Added Vol 00 MVP Cut, Vol 13 WC Compat, Vol 14 Packaging, Vol 15 Runbooks; added accounting ledger extension; status downgrade from "Approved" to "Baseline v2.0"; ADR log created |

---

## 📋 External Review

Two independent reviewers (GPT and Grok) provided in-depth feedback in May 2026. Convergent findings drove the v2.0.1 additions (MVP Cut, WC Compat doc, accounting ledger, ADR log, status downgrade). See [Readiness Scorecard](deliverables/Mercato_Readiness_Scorecard_and_Blueprint.docx) for details.

## 📜 License

Proprietary. © 2026 Mercato Enterprise Marketplace Platform. See [LICENSE](../LICENSE).

---

*This suite represents months of architecture, product, security, compliance, design, and engineering work, condensed into 12+ engineering-grade volumes ready for implementation. Treat it as authoritative.*
