# Mercato — Enterprise Marketplace Platform

This repository contains the implementation-ready documentation suite for the **Mercato Enterprise Marketplace Platform** — a highly modular, microservices-augmented, security-first multivendor marketplace built on top of WordPress / WooCommerce.

**📚 Start here:** [docs_v2/README.md](docs_v2/README.md) — full navigation, status badges, and changelog.

**🚀 If you are an engineer about to build, read first:** [docs_v2/00_mvp_cut/MVP_Cut.md](docs_v2/00_mvp_cut/MVP_Cut.md)

## Documentation Suite (v2.0.1)

| Vol | Document |
|---|---|
| 🚀 00 | [MVP Implementation Cut](docs_v2/00_mvp_cut/MVP_Cut.md) |
| 01 | [Architecture Blueprint](docs_v2/01_architecture/Blueprint.md) + [ADR log](docs_v2/01_architecture/adr/) |
| 02 | [PRD](docs_v2/02_prd/PRD.md) |
| 03 | [BRD](docs_v2/03_brd/BRD.md) |
| 04 | [FSD](docs_v2/04_fsd/FSD.md) |
| 05 | [SRS](docs_v2/05_srs/SRS.md) |
| 06 | [Database & DDL](docs_v2/06_database/Database.md) + [Accounting Ledger Extension](docs_v2/06_database/Database_Accounting_Ledger.md) |
| 07 | [OpenAPI](docs_v2/07_openapi/OpenAPI.yaml) + [AsyncAPI](docs_v2/07_openapi/AsyncAPI.yaml) |
| 08 | [UX & Wireframes](docs_v2/08_ux/Wireframes.md) |
| 09 | [Security & Compliance](docs_v2/09_security/Security.md) |
| 10 | [QA & UAT](docs_v2/10_qa/QA_Spec.md) |
| 11 | [DevOps & Infrastructure](docs_v2/11_devops/DevOps.md) |
| 12 | [AI Collaboration](docs_v2/12_ai/AI_Copilot.md) |
| 13 | [WooCommerce/HPOS Compatibility](docs_v2/13_woocommerce_compat/WooCommerce_HPOS_Compat.md) |
| 14 | [Plugin Packaging Strategy](docs_v2/14_packaging/Plugin_Packaging.md) |
| 15 | [Operational Runbooks](docs_v2/15_runbooks/) |

## Synthesis Deliverables

- 📊 [Gap Matrix](docs_v2/deliverables/Gap_Matrix.md) (also [.xlsx](docs_v2/deliverables/Mercato_Gap_Matrix.xlsx))
- 📋 [Implementation Backlog](docs_v2/deliverables/Implementation_Backlog.md) (also [.xlsx](docs_v2/deliverables/Mercato_Implementation_Backlog.xlsx))
- 📄 [Readiness Scorecard & Blueprint](docs_v2/deliverables/Readiness_Scorecard.md) (also [.docx](docs_v2/deliverables/Mercato_Readiness_Scorecard_and_Blueprint.docx))

## Architectural Principles

1. **Modularity over monolith** — `mercato-core` + 18 domain plugins + 9 integration adapters, behind a versioned SDK contract.
2. **Microservices-augmented** — control plane + AI + search + notification services offload high-velocity workloads.
3. **100% security compliance posture** — PCI-DSS SAQ-A, GDPR + CCPA with automated DSAR, SOC-2 Type II, OWASP ASVS L2, zero-trust mTLS.
4. **Optimized infrastructure** — stateless WordPress on EKS, autoscaling, Aurora Global, Redis, CloudFront, IaC, GitOps, full observability.

## Implementation Readiness

**Overall score: 91 / 100** — ready for engineering execution. Open items in [Gap Matrix](docs_v2/deliverables/Gap_Matrix.md).

## License

Proprietary. © 2026 Mercato.
