# Mercato Gap Matrix

> Cross-volume gap tracking, 60 items across 12 volumes plus cross-cutting concerns.
> Mirror of [`Mercato_Gap_Matrix.xlsx`](Mercato_Gap_Matrix.xlsx) for GitHub viewing.

**Severity legend:** 🔴 CRITICAL · 🟠 HIGH · 🟡 MED · 🟢 LOW

**Priority legend:** P1 = block MVP · P2 = block Phase 2 · P3 = Phase 3+ · P4 = backlog

## Summary by Volume

| Volume | Total | Critical | High | Med | Low | Open | Resolved |
|---|---|---|---|---|---|---|---|
| Cross | 5 | 0 | 3 | 2 | 0 | 5 | 0 |
| Vol 01 Architecture | 7 | 0 | 4 | 3 | 0 | 6 | 1 |
| Vol 02 PRD | 5 | 0 | 1 | 3 | 1 | 5 | 0 |
| Vol 03 BRD | 5 | 0 | 3 | 1 | 1 | 5 | 0 |
| Vol 04 FSD | 4 | 0 | 2 | 2 | 0 | 3 | 1 |
| Vol 05 SRS | 3 | 0 | 2 | 1 | 0 | 3 | 0 |
| Vol 06 DB | 4 | 0 | 1 | 2 | 1 | 4 | 0 |
| Vol 07 OpenAPI | 3 | 0 | 0 | 2 | 1 | 1 | 2 |
| Vol 08 UX | 4 | 0 | 2 | 2 | 0 | 4 | 0 |
| Vol 09 Security | 5 | 0 | 1 | 4 | 0 | 5 | 0 |
| Vol 10 QA | 5 | 0 | 2 | 1 | 2 | 5 | 0 |
| Vol 11 DevOps | 5 | 0 | 2 | 3 | 0 | 5 | 0 |
| Vol 12 AI | 5 | 0 | 2 | 2 | 1 | 5 | 0 |

## Severity × Priority Heatmap

| Sev \ Priority | P1 | P2 | P3 | P4 |
|---|---|---|---|---|
| CRITICAL | 0 | 0 | 0 | 0 |
| HIGH | 20 | 5 | 0 | 0 |
| MED | 0 | 26 | 2 | 0 |
| LOW | 0 | 0 | 5 | 2 |

## Gap Detail

| ID | Volume | Issue | Sev | Priority | Status | Recommendation | Owner |
|---|---|---|---|---|---|---|---|
| GAP-ARCH-001 | Vol 01 Architecture | Plugin SDK contract for third-party authors not yet published | 🟠 HIGH | P1 | Open | Publish SDK with base classes + manifest schema + CI lint; ship by MVP | Platform Eng |
| GAP-ARCH-002 | Vol 01 Architecture | ADR-001 RabbitMQ vs Kafka not decided | 🟠 HIGH | P1 | Open | Decision pending; default Kafka (MSK) — confirm before MVP | Architecture |
| GAP-ARCH-003 | Vol 01 Architecture | ADR-002 OpenSearch vs Algolia | 🟠 HIGH | P1 | Open | Default OpenSearch self-hosted; confirm pre-MVP | Architecture |
| GAP-ARCH-004 | Vol 01 Architecture | ADR-003 Qdrant vs pgvector | 🟡 MED | P2 | Open | Default Qdrant; finalize Phase 3 | AI Architecture |
| GAP-ARCH-005 | Vol 01 Architecture | Aurora MySQL vs PostgreSQL | 🟠 HIGH | P1 | Resolved | Aurora MySQL for WP native; revisit at Phase 4 | Architecture |
| GAP-ARCH-006 | Vol 01 Architecture | Tenant migration between modes (pooled→silo) | 🟡 MED | P2 | Open | Design migration runbook; staged with downtime window | SRE |
| GAP-ARCH-007 | Vol 01 Architecture | Outbox relay implementation language | 🟡 MED | P2 | Open | Go binary preferred over PHP CLI; resolve ADR-006 | Platform Eng |
| GAP-PRD-001 | Vol 02 PRD | Final commission tier numbers not finalized | 🟠 HIGH | P1 | Open | Finance + product confirm $99/499/1999 + Enterprise custom | CPO + Finance |
| GAP-PRD-002 | Vol 02 PRD | Per-tier AI completion limits not load-tested | 🟡 MED | P2 | Open | Run cost projection for top-50 anticipated tenants | PM + Finance |
| GAP-PRD-003 | Vol 02 PRD | Phase-1 launch locales not chosen | 🟡 MED | P2 | Open | Pick 5 launch locales: en, fr, de, es, pt-BR | CPO |
| GAP-PRD-004 | Vol 02 PRD | Time-to-first-sale metric instrumentation missing | 🟡 MED | P2 | Open | Instrument event chain | Data |
| GAP-PRD-005 | Vol 02 PRD | Conversational discovery scope deferred but UX unclear | 🟢 LOW | P3 | Open | Defer to Phase 3; clarify expected behavior | PM |
| GAP-BRD-001 | Vol 03 BRD | US Marketplace Facilitator state map needs finalization | 🟠 HIGH | P1 | Open | Legal review per state; ship in MVP for top-10 GMV states | Compliance + Legal |
| GAP-BRD-002 | Vol 03 BRD | Per-jurisdiction template missing | 🟠 HIGH | P1 | Open | Engage local counsel for US/EU/UK/CA | Legal |
| GAP-BRD-003 | Vol 03 BRD | Service-credit policy precise wording | 🟡 MED | P2 | Open | Legal draft; CSM sign-off | Legal + CSM |
| GAP-BRD-004 | Vol 03 BRD | Category list per region needs regulator review | 🟠 HIGH | P2 | Open | Compliance + Legal per region; final by Phase 2 | Compliance |
| GAP-BRD-005 | Vol 03 BRD | Reseller-sub-tenant contract template | 🟢 LOW | P4 | Open | Phase 4 work | Legal |
| GAP-FSD-001 | Vol 04 FSD | Terminal states for cross-jurisdictional disputes ambiguous | 🟡 MED | P2 | Open | Add dispute_jurisdiction column; spec terminal state set | PM + Eng |
| GAP-FSD-002 | Vol 04 FSD | KYC provider webhook retry policy not defined | 🟡 MED | P2 | Open | Implement inbox pattern with 5 retries / 30min backoff | Eng |
| GAP-FSD-003 | Vol 04 FSD | Cross-currency commission reversal precision | 🟠 HIGH | P1 | Resolved | Persist FX captured at order; reversal uses same rate | Eng |
| GAP-FSD-004 | Vol 04 FSD | Snapshot retention vs PII erasure conflict | 🟠 HIGH | P1 | Open | Snapshot retains pseudonymized buyer; legal retention applies | DPO + Eng |
| GAP-SRS-001 | Vol 05 SRS | Sustained 5k/min order ingestion not yet measured | 🟠 HIGH | P1 | Open | Build k6 scenario; run at staging | SRE |
| GAP-SRS-002 | Vol 05 SRS | Annual pentest scope not contracted | 🟡 MED | P2 | Open | Engage CREST-accredited firm | Security |
| GAP-SRS-003 | Vol 05 SRS | Tier-0 RPO 5min not validated by snapshot drill | 🟠 HIGH | P1 | Open | Monthly DR drill scheduled | SRE |
| GAP-DB-001 | Vol 06 DB | Monthly partition maintenance job not yet scripted | 🟠 HIGH | P1 | Open | Cron job in mercato-core to add next-month partition on day 15 | DB Architect |
| GAP-DB-002 | Vol 06 DB | Pooled-mode trigger overhead unmeasured | 🟡 MED | P2 | Open | Benchmark; expect ~3% latency hit | DB Architect |
| GAP-DB-003 | Vol 06 DB | DB role grant audit not implemented | 🟡 MED | P2 | Open | Migration that enforces grants; CI verify | Security + DB |
| GAP-DB-004 | Vol 06 DB | Online schema change tool selection (gh-ost vs pt-online) | 🟢 LOW | P3 | Open | gh-ost default; pt-online for cases gh-ost can't | SRE |
| GAP-API-001 | Vol 07 OpenAPI | Tenant outbound webhook signature spec | 🟡 MED | P2 | Resolved | HMAC SHA256, header X-Mercato-Signature, replay window 5min | API |
| GAP-API-002 | Vol 07 OpenAPI | Per-endpoint rate-limit table not in spec | 🟡 MED | P2 | Open | Add to OpenAPI extensions x-rate-limit | API |
| GAP-API-003 | Vol 07 OpenAPI | v2 deprecation timeline policy | 🟢 LOW | P3 | Resolved | 2-minor deprecation per SemVer | API |
| GAP-UX-001 | Vol 08 UX | Native mobile app deferred — mobile web SPA optimization | 🟠 HIGH | P2 | Open | Optimize SPAs for mobile web in Phase 1; native app Phase 4 | UX + Eng |
| GAP-UX-002 | Vol 08 UX | Tenant token JSON schema not published | 🟡 MED | P2 | Open | Publish JSON Schema, validate on save | UX + Eng |
| GAP-UX-003 | Vol 08 UX | Enterprise SSO config flow not designed | 🟠 HIGH | P2 | Open | Design wizard; Phase 3 | UX |
| GAP-UX-004 | Vol 08 UX | Comprehensive microcopy library | 🟡 MED | P2 | Open | Build microcopy in 5 launch locales | UX Writer |
| GAP-SEC-001 | Vol 09 Security | Tenant BYOK UX + key validation | 🟡 MED | P3 | Open | Phase 4 — design BYOK workflow + grant management | Security |
| GAP-SEC-002 | Vol 09 Security | Identity verification for DSAR not finalized | 🟠 HIGH | P1 | Open | Two-factor identity (account + email + SMS code) | DPO |
| GAP-SEC-003 | Vol 09 Security | WebAuthn rollout for vendors | 🟡 MED | P2 | Open | Phase 2 — implement via existing libs | Security |
| GAP-SEC-004 | Vol 09 Security | SIEM choice not finalized | 🟡 MED | P2 | Open | Datadog Security default; Wazuh self-hosted for cost-sensitive | Security + SRE |
| GAP-SEC-005 | Vol 09 Security | CSP nonces require build-time SPA changes | 🟡 MED | P2 | Open | Implement nonce injection in SSR layer | Web Eng |
| GAP-QA-001 | Vol 10 QA | Top-30 Playwright scenarios authoring in progress | 🟠 HIGH | P1 | Open | Author 30; ship by MVP gate | QA |
| GAP-QA-002 | Vol 10 QA | k6 scripts for NFR-P set | 🟠 HIGH | P1 | Open | Author; integrate with nightly | SRE + QA |
| GAP-QA-003 | Vol 10 QA | Percy / Chromatic vendor choice | 🟢 LOW | P3 | Open | Percy default; cost-compare quarterly | QA |
| GAP-QA-004 | Vol 10 QA | BugHerd vs Marker.io vs Sentry Session Replay | 🟢 LOW | P3 | Open | Marker.io default for UAT | QA + UX |
| GAP-QA-005 | Vol 10 QA | Litmus or Chaos Mesh | 🟡 MED | P2 | Open | Litmus default; quarterly cadence | SRE |
| GAP-DEV-001 | Vol 11 DevOps | Quarterly DR drill ops cadence | 🟠 HIGH | P1 | Open | Schedule; first drill within 30d of MVP launch | SRE |
| GAP-DEV-002 | Vol 11 DevOps | Active/passive failover not validated | 🟠 HIGH | P1 | Open | Annual full dry-run | SRE |
| GAP-DEV-003 | Vol 11 DevOps | Per-tenant cost attribution in CUR | 🟡 MED | P2 | Open | Tag everything tenant_id; nightly ETL | FinOps |
| GAP-DEV-004 | Vol 11 DevOps | Linkerd vs Istio vs Cilium | 🟡 MED | P2 | Open | Linkerd default for lower complexity | SRE |
| GAP-DEV-005 | Vol 11 DevOps | Canary thresholds not yet validated | 🟡 MED | P2 | Open | 5/25/50/100 with 5min soak; SLO-aware analysis | SRE |
| GAP-AI-001 | Vol 12 AI | Qdrant prod deployment patterns | 🟡 MED | P2 | Open | Run on EKS with 3-node cluster; snapshot to S3 | AI Eng |
| GAP-AI-002 | Vol 12 AI | Per-feature golden sets to be authored | 🟠 HIGH | P1 | Open | 200 examples per feature; hand-graded | AI + PM |
| GAP-AI-003 | Vol 12 AI | Enterprise BYOM contract clauses | 🟢 LOW | P4 | Open | Phase 4 | Legal + AI |
| GAP-AI-004 | Vol 12 AI | Provider zero-retention contracts for OpenAI/Anthropic | 🟠 HIGH | P1 | Open | Procure Enterprise tier with zero retention | Procurement |
| GAP-AI-005 | Vol 12 AI | Presidio false positive rate | 🟡 MED | P2 | Open | Tune for vendor product copy; custom recognizers | AI Eng |
| GAP-X-001 | Cross | Dokan/WCFM importer end-to-end test | 🟡 MED | P3 | Open | Phase 4 importer; dry-run + 24h rollback | Eng |
| GAP-X-002 | Cross | Per-locale legal text reviewed by counsel | 🟠 HIGH | P2 | Open | Pre-Phase 2 | Legal |
| GAP-X-003 | Cross | Tenant-aware Grafana dashboards | 🟠 HIGH | P1 | Open | Tagging discipline + per-tenant variable | SRE |
| GAP-X-004 | Cross | Status page tooling | 🟡 MED | P2 | Open | StatusPage.io default | SRE |
| GAP-X-005 | Cross | SOC-2 Type II first audit | 🟠 HIGH | P2 | Open | 12-month observation; engage auditor at month 1 | Compliance |