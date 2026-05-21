# Volume 10 — Mercato Enterprise Marketplace Platform
## QA & UAT Specification (Engineering-Grade, Implementation-Ready)

> Document owner: QA / Test Architect
> Status: Implementation Baseline v2.0 — Approved for MVP planning; ADRs pending
> Version: 2.0.0 (full rewrite of v1.0)
> Cross-references: Vol 02 (PRD stories), Vol 04 (FSD FR-IDs), Vol 05 (SRS NFRs), Vol 09 (Security)

---

## 1. Executive Assessment

The v1.0 QA outlined the right pyramid (PHPUnit / Jest / Playwright / k6) and named the right tools, but did not specify coverage targets per layer, test data strategy, traceability to requirements, security/chaos/contract tests, or the UAT sign-off contract with enterprise tenants.

This rewrite delivers:

1. A **test strategy with target coverage and gating** per pyramid layer.
2. A **traceability matrix builder** ensuring every PRD story / FR / NFR has at least one named test.
3. **Test data management** with realistic seeders, PII anonymization, and shadow database isolation.
4. **End-to-end scenarios** for the 15 highest-value user flows.
5. A **performance test plan** with k6 scripts targeting NFR thresholds.
6. A **security test plan** with SAST/DAST/SCA/dependency/container/IaC scans + scheduled pentest.
7. A **chaos engineering plan** with named experiments and exit criteria.
8. A **UAT plan** with personas, scripts, sign-off workflow, and bug intake.
9. A **CI/CD gating policy**.

**Implementation Readiness (this volume): 90/100.** Outstanding: contract tests for some integration adapters (Avalara, Adyen) to be finalized in Phase 2.

---

## 2. Gap Analysis vs. v1.0

| # | Gap | v1.0 | v2.0 |
|---|---|---|---|
| G-QA-001 | Coverage targets | Stated 80% | Per-layer differentiated targets |
| G-QA-002 | Traceability | Stated | Build job + CI gate |
| G-QA-003 | Test data anonymization | Mentioned | Generators + masking rules |
| G-QA-004 | Security tests | Mentioned SAST/DAST | Full SCA + IaC + container + dependency tracks |
| G-QA-005 | Chaos | Mentioned | Catalog of experiments (§8) |
| G-QA-006 | UAT contract | Mentioned | Personas, scripts, sign-off (§10) |
| G-QA-007 | Perf scripts | None | k6 scripts mapped to NFRs (§7) |
| G-QA-008 | Defect taxonomy | None | Severity/priority matrix (§11) |

---

## 3. Test Strategy

### 3.1 Pyramid Layers, Coverage, Tool

| Layer | Tool | Scope | Target Coverage |
|---|---|---|---|
| Unit (PHP) | PHPUnit + Pest | services, value objects, rule engine, validators | ≥ 85% lines / 80% branches |
| Unit (JS/TS) | Vitest / Jest + RTL | components, hooks, utilities | ≥ 80% |
| Integration (PHP/WP) | wp-env + Pest | plugin lifecycle, hooks, query builder | every public hook + 1 negative |
| Contract (API) | Schemathesis / Pact | every endpoint vs Vol 07 | 100% endpoints |
| Contract (events) | AsyncAPI validator | every emitter & consumer | 100% events |
| E2E (web) | Playwright | top user flows multi-actor | top 30 scenarios |
| Performance | k6 | NFR thresholds | catalog browse, checkout, payout |
| Visual regression | Percy / Chromatic | top 50 screens | all Vol 08 named screens |
| Accessibility | axe-core CI | every page | WCAG 2.1 AA gate |
| Security SAST | Semgrep / Snyk Code | repo on PR | 0 Critical |
| Security SCA | Snyk OS / npm audit / composer audit | weekly + PR | 0 Critical |
| Security DAST | OWASP ZAP | nightly against staging | 0 High |
| Container scan | Trivy / ECR | every image | 0 Critical |
| IaC scan | tfsec / Checkov | Terraform PR | 0 Critical |
| Chaos | Litmus | scheduled | per §8 |
| UAT | manual scripted | Vol 02 lifecycles | per-release |

### 3.2 Test Environments

| Env | Purpose | Data |
|---|---|---|
| Local | dev | wp-env Docker; seeded factory data |
| Ephemeral PR | per-PR | seeded factory data; auto-destroyed after merge |
| Staging | pre-release | anonymized production snapshot |
| UAT | tenant review | seeded UAT data |
| Pre-prod | smoke | blue-green dark prod data |
| Prod | live | live |

Ephemeral environments spun up via ArgoCD ApplicationSet on PR open; teardown on close.

### 3.3 Test Data Management

- **Factories** (PHPUnit + Faker) for vendors, products, orders.
- **Sanitization rules** for production snapshots cloned to staging:
  - Emails → `<random>@anon.mercato.test`
  - Names → `Anon User N`
  - Phone → `+15555550000 + N`
  - Addresses → randomized real ones
  - Tax IDs → null
- **Seed scenarios**: a "10k vendor / 1M product / 10M order" dataset for perf; a "200-vendor demo tenant" for UAT; a "10-vendor smoke" for E2E.
- **State manipulation endpoints** (only in test envs): `POST /qa/seed`, `POST /qa/reset`, `POST /qa/mock-webhook/stripe`, `POST /qa/timewarp/{seconds}`.

---

## 4. Traceability Matrix

CI job builds `docs_v2/deliverables/traceability.xlsx` from:
- PRD stories (Vol 02 §4).
- FRs (Vol 04).
- NFRs (Vol 05).
- Test annotations in test files (`@PRD US-ORD-001`, `@FR FR-ORD-001`, `@NFR NFR-P-006`).

Gate: CI fails if any PRD story or FR has zero `@PRD` / `@FR` annotated test.

---

## 5. Test Naming Convention

```
class CheckoutMultiVendorTest extends TestCase
{
    /**
     * @PRD US-ORD-001
     * @FR  FR-ORD-001 FR-ORD-002
     * @TC  TC-E2E-031
     */
    public function test_cart_with_two_vendors_creates_two_suborders(): void
    {
        ...
    }
}
```

`TC-<layer>-<n>` is the public test ID surfaced to UAT.

---

## 6. End-to-End Scenarios (Playwright)

Top 30 scripted; 15 highlighted here:

| TC ID | Scenario | Actors |
|---|---|---|
| TC-E2E-001 | Vendor signs up + KYC + first product | Vendor, Tenant Admin |
| TC-E2E-002 | Buyer browses, adds multi-vendor cart, checks out | Buyer |
| TC-E2E-003 | Vendor acknowledges + ships sub-order; buyer sees tracking | Vendor, Buyer |
| TC-E2E-004 | Refund issued by admin; commission reverses | Admin |
| TC-E2E-005 | Dispute lifecycle full → admin resolution | Buyer, Vendor, Admin |
| TC-E2E-006 | Stripe payout batch produces correct transfers | Admin, Vendor |
| TC-E2E-007 | Reconciliation report ±0.01 against Stripe Treasury | Finance |
| TC-E2E-008 | Vendor suspended; products hidden; existing orders unaffected | Admin |
| TC-E2E-009 | Bulk import 10k products with 50 invalid rows | Vendor |
| TC-E2E-010 | Buyer DSAR — Access + Erasure | Buyer, Compliance |
| TC-E2E-011 | Tenant white-label branding propagates | Tenant Admin |
| TC-E2E-012 | Custom domain provisioning end-to-end | Tenant Admin |
| TC-E2E-013 | AI Copilot generates description, vendor accepts | Vendor |
| TC-E2E-014 | Search filter set produces expected facets | Buyer |
| TC-E2E-015 | Multi-currency: buyer USD, vendor EUR settlement | Buyer, Vendor |

All scenarios run multi-browser (Chromium, Firefox, WebKit) on mobile + desktop viewports.

---

## 7. Performance Test Plan (k6)

### 7.1 Mapping NFR → Test
| NFR | Test ID | Description |
|---|---|---|
| NFR-P-001 search P95 <100ms | TC-PERF-001 | 1k VUs sustained search |
| NFR-P-002 PDP TTFB <300ms | TC-PERF-002 | 5k VUs distributed across 100 PDPs |
| NFR-P-003 checkout P95 <800ms | TC-PERF-003 | 200 VU checkout, ramp |
| NFR-P-006 5k/min orders | TC-PERF-006 | sustained for 30min |
| NFR-P-009 payout 10k/30min | TC-PERF-009 | batch job under load |
| NFR-S-001 50k buyer sessions | TC-PERF-010 | distributed soak |

### 7.2 k6 Snippet

```javascript
import http from 'k6/http'; import { check, sleep } from 'k6';
export const options = {
  scenarios: {
    search: {
      executor: 'ramping-arrival-rate',
      preAllocatedVUs: 200, maxVUs: 1000,
      stages: [
        { duration: '2m', target: 200 },
        { duration: '10m', target: 1000 },
        { duration: '2m', target: 0 },
      ],
    },
  },
  thresholds: {
    'http_req_duration{type:search}': ['p(95)<100'],
    'http_req_failed': ['rate<0.001'],
  },
};
export default function () {
  const r = http.get('https://api-staging.mercato.com/v1/search/products?q=widget', {
    tags: { type: 'search' },
    headers: { Authorization: `Bearer ${__ENV.TOKEN}` },
  });
  check(r, { status: 200 });
  sleep(1);
}
```

### 7.3 Reporting
- Run nightly against staging seeded with 100k vendors / 1M products / 10M orders.
- Trend stored in Grafana; regression alert if P95 > NFR threshold for 3 consecutive nights.

---

## 8. Chaos Engineering

### 8.1 Experiments
| ID | Experiment | Hypothesis | Pass Criteria |
|---|---|---|---|
| TC-CHAOS-001 | Kill primary Aurora | Failover < 60s; no order loss | Order ingest resumed; outbox catches up |
| TC-CHAOS-002 | Pause Kafka broker | Outbox queues; resumes | Lag drains within 5min of recovery |
| TC-CHAOS-003 | Disable OpenSearch cluster | Storefront degraded but functional | Search returns MySQL fallback results |
| TC-CHAOS-004 | Stripe API 5xx 50% | Retries succeed | Payout success once provider restored |
| TC-CHAOS-005 | AI provider down | UI hides AI; no crashes | Banner shown, fallback enabled |
| TC-CHAOS-006 | CPU saturation WP pod | HPA scales out; ELB drains | Latency recovers within 2min |
| TC-CHAOS-007 | Network partition between AZs | Reads continue from healthy AZ | No write loss |
| TC-CHAOS-008 | Redis flush | Cache rebuilds; minor latency hit | < 5min impact |

Game day cadence: quarterly.

---

## 9. Security Testing

- **SAST** Semgrep + Snyk Code in PR.
- **DAST** OWASP ZAP nightly against staging; auth flow scripted.
- **SCA** Snyk OS + `composer audit` + `npm audit`.
- **Container** Trivy + ECR scan; deploy blocked on Critical.
- **IaC** tfsec + Checkov in Terraform PRs.
- **Secrets** gitleaks pre-commit + CI.
- **Pentest** external quarterly + post-major-release.
- **Bug bounty** continuous.

### 9.1 Targeted Test Cases
- IDOR: vendor A reading vendor B's `/suborders/{id}` returns 404.
- Tenant boundary: tenant A reading tenant B's resources returns 404.
- Authentication: replay of JWT after logout → 401.
- Idempotency: replayed POST returns original response.
- File upload: PHP file masquerading as image rejected.
- SSRF: webhook URL with internal host blocked.
- Prompt injection (AI): adversarial input does not exfiltrate system prompt.
- Rate limit: 100 rapid /search → 429 with Retry-After.

---

## 10. UAT Plan

### 10.1 Process
- Feature branch deployed to UAT env.
- Test scripts (per persona) provided as PDFs + Notion pages.
- Tenant admins / vendors / buyers from beta cohort execute scripts.
- Feedback via BugHerd embedded; auto-syncs to Jira/Linear.
- Final demo + sign-off per release.

### 10.2 Scripts (excerpt)
- Persona: Tenant Admin — "Approve a vendor, set commission rule, view payout reconciliation".
- Persona: Vendor — "Onboard, create 5 products, fulfill an order, request payout".
- Persona: Buyer — "Search, multi-vendor checkout, file dispute, leave review".

### 10.3 Sign-off Workflow
- UAT lead reviews bug intake.
- Critical bugs must close; SEV-3/4 allowed with mitigation note.
- Tenant CSM signs off on Pro+ release; Enterprise tenant signs own contract.

---

## 11. Defect Taxonomy

| Severity | Definition | Example |
|---|---|---|
| S1 Critical | Data loss / payment failure / security breach / outage | Lost commission entries |
| S2 Major | Feature unusable | Vendor cannot publish products |
| S3 Moderate | Feature degraded | Search filter mis-counts |
| S4 Minor | Cosmetic / non-blocking | Misaligned button |

| Priority | Action |
|---|---|
| P1 | Hotfix patch within 24h |
| P2 | Next sprint |
| P3 | Backlog |
| P4 | Icebox |

CFR / Bug-fix turnaround tracked per sprint.

---

## 12. CI/CD Gating Policy

| Stage | Gates |
|---|---|
| PR open | lint + unit + SAST + SCA + IaC + container scan + traceability |
| PR ready | integration tests + contract + a11y |
| Merge to main | nightly E2E + DAST + visual regression + perf nightly |
| Tag release | full E2E + perf full + UAT sign-off |
| Deploy prod | smoke + canary + observability healthy |

Branch protection: 2 approvals + CODEOWNERS + green CI.

---

## 13. Test Reporting

- Per-PR: Allure report attached.
- Nightly: dashboard with pass/fail trends; flaky test top-10.
- Per-release: traceability XLSX + UAT sign-off attached to release notes.

---

## 14. Cross-Volume Cross-References

| This Section | See Also |
|---|---|
| §3 strategy | Vol 11 DevOps CI/CD |
| §6 E2E | Vol 02 PRD lifecycles |
| §7 Perf | Vol 05 SRS NFRs |
| §9 Security | Vol 09 Security |
| §10 UAT | Vol 03 BRD SLA |

---

*End of Volume 10 — QA & UAT v2.0.*
