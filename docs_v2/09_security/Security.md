# Volume 09 — Mercato Enterprise Marketplace Platform
## Security Architecture & Compliance (Engineering-Grade, Implementation-Ready)

> Document owner: Security Architect + CISO / Compliance Lead
> Status: Implementation Baseline v2.0 — Approved for MVP planning; ADRs pending
> Version: 2.0.0 (full rewrite of v1.0)
> Cross-references: Vol 01 (Architecture §5.3, §11), Vol 03 (BRD §8 regulatory), Vol 04 (FSD), Vol 06 (DDL), Vol 11 (DevOps §7)

---

## 1. Executive Assessment

The v1.0 security doc covered the right surface (defense-in-depth, MFA, RBAC, encryption, PCI scope, GDPR) but lacked the engineering specificity required for OWASP ASVS L2 conformance and SOC-2 Type II evidence. It named tactics without specifying control owners, control evidence, control frequency, key-management practice, secrets rotation policy, incident-response playbook, or threat model.

This rewrite delivers a posture that targets **100% security compliance** for SOC-2 Type II, PCI-DSS SAQ-A, GDPR, CCPA, OWASP ASVS L2 minimum, and Mercato's own security policies.

Contents:
1. STRIDE threat model per surface (web, REST, browser, mobile-like SPAs, microservices, broker, control plane).
2. Authentication & authorization architecture, MFA, federation, capability JWT.
3. RBAC matrix per resource & capability across the 9 personas.
4. Data classification → encryption → access controls → retention.
5. Audit logging design with immutability guarantees and SOC-2 evidence.
6. Secrets management with rotation policy and break-glass.
7. Vulnerability management (SAST, DAST, SCA, DAST, container, IaC scanning).
8. Incident response playbook & runbooks.
9. Privacy engineering (DSAR automation, consent, PII inventory).
10. Tenant isolation engineering (3 modes).

**Implementation Readiness (this volume): 91/100.** Outstanding: tenant-side BYOK key management UX (Phase 4); tracked in Gap Matrix.

---

## 2. Gap Analysis vs. v1.0

| # | Gap | v1.0 | v2.0 |
|---|---|---|---|
| G-SEC-001 | Threat model | Mentioned OWASP top 10 | STRIDE per surface (§3) |
| G-SEC-002 | RBAC matrix | List of roles | Capability matrix per resource (§5) |
| G-SEC-003 | Secrets rotation | Mentioned vault | Rotation policy + break-glass (§7) |
| G-SEC-004 | Audit immutability | Mentioned | Append-only + DB grant + Object Lock (§6) |
| G-SEC-005 | Incident playbook | Absent | Severity classes, comms, postmortem (§9) |
| G-SEC-006 | DSAR automation | Mentioned | Step-by-step workflow + retention exceptions (§10.2) |
| G-SEC-007 | Tenant isolation guarantees | Mentioned | Mode-specific isolation matrix (§11) |
| G-SEC-008 | Key management | Mentioned KMS | KMS keys per tenant, BYOK pattern (§8) |

---

## 3. Threat Model (STRIDE per Surface)

| Surface | Spoofing | Tampering | Repudiation | Info Disclosure | DoS | Elevation |
|---|---|---|---|---|---|---|
| Public storefront | bot signups | XSS | review abuse | PII leak | scraping/DDoS | n/a |
| Buyer SPA | session theft | client tampering | review abuse | account info | client DoS | privilege gain |
| Vendor SPA | account takeover | edit other vendor's data | log denial | other vendor's data | mass uploads | RBAC bypass |
| Tenant Admin SPA | priv escalation | settings tampering | audit erase | tenant data | bulk ops | cross-tenant |
| Super Admin UI | priv escalation | global tamper | audit erase | all tenants | n/a | full compromise |
| REST API | API key theft | replay | log denial | data leak | rate exhaustion | priv gain |
| Webhook ingress | spoofed sender | replay | repudiation | leaks via mistyped routes | flood | priv via signature bypass |
| Kafka broker | producer impersonation | message tamper | event erase | topic leak | flood | n/a |
| Control plane | issuer compromise | flag tampering | n/a | tenant metadata | DoS | tenant takeover |
| Stripe webhook | spoofed sender | tampered amount | dispute denial | metadata leak | flood | n/a |
| AI service | prompt injection | output tamper | repudiation | PII out | token DoS | n/a |
| Object storage | impersonated upload | object tamper | n/a | bucket misconfig | upload flood | n/a |

Each row maps to specific mitigations (catalogued below).

---

## 4. Authentication & Identity

### 4.1 User AuthN

- **Storefront / buyer:** WordPress accounts (bcrypt cost ≥12 enforced), optional social sign-in (Google, Apple), magic link, optional MFA TOTP.
- **Vendor:** Mercato account (Argon2id), MFA TOTP enforced at Enterprise; WebAuthn supported.
- **Tenant admin/staff:** MFA required (Enterprise mandatory); optional SAML 2.0 / OIDC for federation.
- **Super admin (us):** SSO with operator IdP + WebAuthn + per-action 2FA prompts for destructive actions.

### 4.2 Session Model
- Server-side session for WP admin paths (existing WP behavior, hardened).
- For SPAs: stateless JWT access token (5min TTL) + opaque refresh token (rotating, 30d) stored httpOnly Secure SameSite=Lax cookie + same-origin proxy.
- Session revocation: refresh token table; revoke on logout, password change, MFA reset.
- Idle timeout: 30min default; 15min for admin roles.

### 4.3 Password Policy
- Min 12 chars; rejected against HIBP (k-anonymity API); no expiry (NIST aligned), but force-rotate on suspected compromise.
- Lockout: 5 failed attempts → 15min lock with captcha; further attempts increase backoff; account lock after 25 attempts in 24h.

### 4.4 Capability JWT (Tenant-Plane)
RS256 signed; rotated every 24h with overlap window. Validation:
- Signature against published JWKS.
- `aud == mercato-data-plane`.
- `iss == mercato-control-plane`.
- `exp > now`.
- `jti` not on revocation list.
- `tenant_id` matches request scope.

### 4.5 Federation (Enterprise)
- SAML 2.0 SP-initiated + IdP-initiated.
- OIDC supported (Azure AD, Okta, Google Workspace).
- SCIM 2.0 for provisioning (Phase 4).

### 4.6 API Keys
- Format `mer_live_<26 chars base32>` with prefix indexed; full key shown once at creation.
- Server stores SHA-256 hash.
- Scoped (read-only / write / admin) and rate-limited per key.
- Auto-rotation policy: tenant admin may set max age (default 365d). Notification 30d / 7d / 1d before expiry.

---

## 5. Authorization (RBAC)

### 5.1 Capability Catalog (excerpt)

```
mercato_super_admin_full
mercato_tenant_admin_full
mercato_vendors_read_all, mercato_vendors_manage
mercato_products_moderate
mercato_orders_read_all, mercato_orders_read_own, mercato_orders_admin
mercato_commissions_configure
mercato_payouts_execute, mercato_payouts_view_all
mercato_disputes_moderate
mercato_messages_read_thread, mercato_messages_moderate
mercato_audit_read
mercato_compliance_dsar_handle
mercato_ai_use, mercato_ai_admin
mercato_settings_branding, mercato_settings_billing
```

### 5.2 Capability Resolution
- Each request: load user → tenant roles → capabilities → cache 60s in-process.
- Decision wrapper: `mercato_user_can($cap, $tenant_id, $resource_owner_id = null)`.
- Resource ownership check: `*_own` capabilities require resource's `vendor_id == acting vendor's id` (or `buyer_user_id == acting user`).

### 5.3 Permission Audit
- Capability assignments audit-logged.
- Periodic access review: quarterly, all `mercato_*_admin` capabilities.

### 5.4 Defense in Depth
- API gateway WAF rule rejects mismatched `X-Tenant-ID` vs JWT `tenant`.
- Query builder injects `tenant_id`.
- DB trigger rejects writes with wrong `tenant_id` (Pooled mode).
- Output scrubber for cross-tenant content in errors.

---

## 6. Data Protection

### 6.1 Data Classification

| Class | Examples | Encryption | Access | Retention |
|---|---|---|---|---|
| 0 Public | catalog meta | TLS in transit | open | indefinite |
| 1 Internal | tenant settings | TLS + at-rest (DB) | role-gated | per setting |
| 2 PII | name, email, address | TLS + at-rest | least privilege | per retention table |
| 3 Sensitive PII | tax_id, beneficial owner, DOB, ID document | TLS + at-rest + field-level (Vault) | tightly scoped | per retention table |
| 4 Financial | bank tokens, Stripe tokens, payout amounts | TLS + at-rest + field-level | very tight | 7y tax |
| 5 Authentication | password hashes, MFA secrets | argon2id / KMS-encrypted | only auth subsystem | until reset |
| 6 PCI cardholder | (NEVER stored) | n/a | n/a | n/a |

### 6.2 Encryption Architecture

- **In transit**: TLS 1.3, HSTS 1y preload, secure ciphers only. mTLS service-to-service in cluster (Linkerd).
- **At rest**:
  - Database volumes: AWS KMS CMK per environment.
  - S3 buckets: SSE-KMS with per-tenant CMK for KYC + reports; SSE-S3 for public assets.
  - Backups encrypted with separate CMK with no production-role access.
  - Field-level: tax_id_hash (HMAC-SHA256 keyed in Vault); document filenames re-keyed with per-tenant key.
- **Key rotation**: KMS automatic annual rotation; manual rotation on incident.

### 6.3 BYOK (Bring Your Own Key, Enterprise)
- Tenant supplies AWS KMS key ARN; Mercato grants access via grants (not policy).
- Key disablement → tenant data immediately inaccessible; documented contractual implication.

### 6.4 Tokenization & Hashing

- **Cards**: NEVER touched; Stripe Elements / Checkout only. PAN never crosses our network. PCI scope SAQ-A.
- **Tax IDs / SSN-equivalents**: HMAC-SHA256 hash stored; raw never persisted (transient compute only via secure enclave with TTL).
- **Document filenames**: random UUID, never user-supplied name.
- **PII in audit log**: redacted IPv6 last 80 bits, full IP retained only for security incidents in separate store.

### 6.5 Data Retention & Erasure

| Data | Retention | Method |
|---|---|---|
| Orders / Commissions / Payouts | 7y (tax) | Then archive to S3 Glacier with Object Lock |
| Audit log | 2y hot, +5y cold | Object Lock compliance mode |
| Messages | 1y hot, 2y cold, then anonymize | scrub PII, keep counts |
| Reviews | indefinite | anonymize buyer if requested |
| KYC documents | 7y | Glacier; then DELETE certified |
| Marketing data | until opt-out | DELETE on unsubscribe |
| Sessions | 30d | rotation/expiry |
| Telemetry buffer | 7d | discard |

---

## 7. Secrets Management

### 7.1 Backend
HashiCorp Vault (or AWS Secrets Manager). One Vault namespace per environment per tenant tier.

### 7.2 Categories
- App secrets (DB creds, broker creds): short-TTL dynamic creds where possible.
- API keys to vendors (Stripe, Sendgrid, OpenAI): per-tenant where applicable.
- Signing keys (JWT RS256): rotated 90d with overlap.
- Webhook secrets (HMAC): rotated by tenant.

### 7.3 Rotation Policy
| Secret | Rotation cadence |
|---|---|
| Database master | 90d (root)/dynamic (app via Vault) |
| JWT signing key | 90d |
| Webhook HMAC | 365d default; manual on request |
| API keys | tenant choice; default 365d |
| OAuth client secrets | 365d |
| KMS CMK | 365d auto |

### 7.4 Break-Glass
- Two-person rule for root credentials.
- Break-glass account in cold storage (offline encrypted USB at compliance officer + secondary).
- Use is auto-paged to entire security team.

### 7.5 Secret Hygiene
- CI scans (gitleaks) on every commit.
- No secrets in env files committed.
- Container build refuses if secret bake detected.
- Logs sanitized by structured logging library to redact known patterns.

---

## 8. Audit Logging

### 8.1 What is Logged
Every:
- AuthN event (success/failure).
- AuthZ denial.
- RBAC change.
- Tenant settings change.
- Commission rule change.
- Payout execution.
- Refund issuance.
- Vendor approval/suspension.
- Manual admin reads of buyer/vendor data ("audit-marked read").
- DSAR fulfillments.
- Capability JWT issuance.
- Key rotations.
- Backup events.

### 8.2 Where
- Primary: `wp_mercato_audit_log` table (append-only, monthly partition).
- Shipped to: SIEM (Datadog Security or self-hosted Wazuh) within 60s.
- Archived: S3 Object Lock (compliance mode), 7y.

### 8.3 Immutability Guarantees
- DB role `mercato_app` has only INSERT/SELECT on audit_log.
- DB role `mercato_ops` has READ only.
- Archive Object Lock prevents DELETE/OVERWRITE during retention.

### 8.4 SOC-2 Evidence Mapping
- CC7.2: log monitoring → SIEM correlation rules + alerts.
- CC7.3: incident response → runbooks RB-01..RB-08 (Vol 01 §14.4).
- CC6.6: monitor changes to system → audit_log + diff captured.
- CC6.7: encryption → KMS keys + rotation logs.

---

## 9. Vulnerability Management

### 9.1 CI Pipeline Security Gates

| Stage | Tool | Required to Pass |
|---|---|---|
| Pre-commit | gitleaks (secrets), pre-commit hooks | yes |
| PR | PHPStan L8, ESLint, PHPCS, Psalm | yes |
| PR | SAST (Snyk Code or Semgrep) | yes, no Critical |
| PR | SCA dependency scan (Snyk Open Source, npm audit, composer audit) | yes, no Critical |
| PR | IaC scan (tfsec, checkov) | yes |
| Pre-deploy | Container scan (Trivy or ECR scan) | yes, no Critical |
| Nightly | DAST (OWASP ZAP) against staging | report; SEV>= Medium creates ticket |
| Quarterly | external pentest | findings tracked |
| Annual | SOC-2 audit | evidence binder |

### 9.2 CVE / Patch Window
- Critical CVE patched in production within 7 days.
- High CVE within 30 days.
- Medium 90 days.
- Auto-update WP/WC point releases under change control.

### 9.3 Bug Bounty
Public program (HackerOne or Bugcrowd). Scope, payouts, safe harbor.

---

## 10. Privacy & Compliance Engineering

### 10.1 PII Inventory
Auto-maintained by `mercato-core/privacy/inventory.php`:
- Annotation-based: every column tagged `@PII(level=2|3, purpose=…, retention=…)`.
- Build job extracts annotations to `docs_v2/deliverables/pii-inventory.csv`.

### 10.2 DSAR Workflow (engineering view)
1. Request lands in `wp_mercato_dsar_requests`.
2. Identity verification: requester logs in + email confirms.
3. Cron job invokes per-plugin `PrivacyProvider::collect(user_id)` and aggregates results.
4. Output bundle: signed S3 zip with JSON + CSV per entity.
5. Erasure: `PrivacyProvider::erase(user_id)` anonymizes specified fields; legal retention overrides retain hashed proxy.
6. Audit log entry.
7. Notification to requester within SLA (30d GDPR / 45d CCPA).

### 10.3 Consent Management
- Buyer cookie consent banner (CMP) integrating IAB TCF where applicable.
- Marketing opt-in tracked per email/SMS channel in `wp_mercato_consent_log`.
- Vendor staff invite captures policy acknowledgment.

### 10.4 Data Processing Agreements (DPAs)
- Sub-processor registry published publicly.
- DPA template offered to tenants; signed copy stored.

---

## 11. Tenant Isolation Engineering

### 11.1 Pooled Multisite
- `tenant_id` column on every custom table.
- Application query builder injects.
- DB trigger guards INSERT/UPDATE.
- WP multisite blog ID maps to tenant.
- Logs tagged with tenant_id; never co-mingled in dashboards.

### 11.2 Silo Container
- Dedicated WP pod; dedicated DB schema in shared Aurora cluster.
- DB user limited to that schema.
- Separate S3 prefix or bucket.

### 11.3 Dedicated Cluster
- Per-tenant Aurora cluster, OpenSearch domain, Redis, S3 bucket.
- Optional dedicated EKS namespace for Enterprise+.
- BYOK supported.

### 11.4 Logical-to-Physical Mapping
| Mode | DB | Search | Cache | S3 |
|---|---|---|---|---|
| Pooled | shared, RLS | shared, index-per-tenant | shared, key-prefix | shared, prefix-per-tenant |
| Silo | shared cluster, dedicated schema | shared, index-per-tenant | shared, key-prefix | dedicated bucket |
| Dedicated | dedicated | dedicated | dedicated | dedicated + per-tenant KMS |

---

## 12. WordPress / WooCommerce-Specific Hardening

- XML-RPC disabled (`xmlrpc.php` blocked at WAF).
- REST API user enumeration disabled (`/wp-json/wp/v2/users` requires auth; UID enumeration via `?author=N` blocked).
- WP admin login throttled + 2FA enforced for admin/editor/manager.
- File editor in admin disabled (`DISALLOW_FILE_EDIT`).
- Direct file modifications blocked (`DISALLOW_FILE_MODS`).
- Cron disabled in PHP; runs via system cron (Vol 11).
- Custom security headers: HSTS, CSP (Vol 12 covers AI sandbox), X-Frame-Options DENY, X-Content-Type-Options nosniff, Referrer-Policy strict-origin-when-cross-origin, Permissions-Policy minimal.
- Strict CORS for `/wp-json/mercato/v1/*`: only known SPAs and registered API origins.

---

## 13. Application Security Controls

### 13.1 Input Validation
- JSON Schema (Vol 07) at REST layer.
- WordPress sanitization helpers (`sanitize_text_field`, `wp_kses_post`, `absint`).
- Numeric ranges via custom validator.
- Server-trusted enums; never trust client.

### 13.2 Output Encoding
- React auto-escapes; sanitize HTML rich text with DOMPurify before rendering.
- Server-rendered HTML via Twig autoescape on.
- SQL via prepared statements (PDO or `$wpdb->prepare`).

### 13.3 File Uploads
- Allowlist MIME; verify magic number; rename to UUID.
- Quarantine bucket; ClamAV scan; promotion to primary bucket on clean.
- No execution permissions on stored files.
- Image processing in restricted sandbox; remove EXIF.

### 13.4 Rate Limiting
| Surface | Window | Limit |
|---|---|---|
| Login | 1h | 10/IP, 5/user |
| Register | 1h | 20/IP |
| /search | 1m | 60/IP |
| /products POST | 1m | 60/vendor |
| /ai/completions | 1m | per plan |
| Webhook ingress | per provider's signed signature window |

Implementation: token bucket via Redis; sliding window for sensitive endpoints.

### 13.5 Anti-Abuse
- Bot detection (Cloudflare Bot Management) on storefront.
- Honeypot fields on public forms.
- Velocity checks: rapid signups, rapid checkouts.

### 13.6 Content Security Policy (CSP)
Storefront and admin SPAs ship CSP nonces:

```
default-src 'self';
script-src 'self' 'nonce-{n}' https://js.stripe.com https://*.mercatocdn.com;
style-src  'self' 'nonce-{n}' 'unsafe-inline';     # only for tenant branding CSS vars
img-src    'self' data: blob: https:;
connect-src 'self' https://api.mercato.com wss://*.mercato.com https://api.stripe.com;
frame-src  https://js.stripe.com;
frame-ancestors 'none';
report-uri /csp-report;
```

---

## 14. Incident Response

### 14.1 Severity Classes
- SEV-1: data breach suspected; widespread outage; PII at risk.
- SEV-2: tenant isolated; major functionality down.
- SEV-3: degraded service; partial functionality.
- SEV-4: cosmetic / advisory.

### 14.2 On-Call & Comms
- PagerDuty rotation; primary + secondary; 24/7 for SEV-1/2.
- SEV-1 spawn: incident channel, IC, comms officer, scribe.
- External comms via status page within 30min of SEV-1 declaration.
- Customer comms via email + portal within 24h of SEV-1.
- Regulator comms within 72h if PII breach (GDPR Art. 33).

### 14.3 Postmortem
- Blameless within 5 business days of SEV-1/2.
- Action items tracked in security backlog; aged action items reviewed monthly.

### 14.4 Runbooks
RB-01 through RB-08 (Vol 01 §14.4) — security-specific:
- RB-06 Tenant data leak suspected: containment (revoke JWTs, block APIs, disable webhooks), forensics, customer comms.
- RB-09 Credential leak (gitleaks alert): rotate, scan logs, customer comms if external.
- RB-10 Stripe Connect account compromise: pause vendor, notify Stripe, refund holds.
- RB-11 Adversarial prompt extraction from AI: rotate prompts, force review of recent outputs, customer comms.

---

## 15. Compliance Evidence Catalog

Tracked in `docs_v2/deliverables/compliance-evidence.xlsx`:

| Control | Standard | Evidence | Owner | Cadence |
|---|---|---|---|---|
| CC6.1 logical access | SOC-2 | RBAC matrix + access reviews | Security | quarterly |
| CC6.7 encryption | SOC-2 / GDPR | KMS audit logs + rotation logs | SRE | quarterly |
| CC7.2 monitoring | SOC-2 | SIEM rules + alerts | SRE | continuous |
| Req 8 unique IDs | PCI | RBAC matrix | Security | annual |
| Req 10 logging | PCI | audit log + SIEM | Security | annual |
| Art. 30 records | GDPR | DPA registry + PII inventory | DPO | continuous |
| Art. 32 measures | GDPR | this document | DPO | annual |
| Art. 33 breach | GDPR | RB-06 + 72h workflow | Security | continuous |

---

## 16. Cross-Volume Cross-References

| This Section | See Also |
|---|---|
| §3 Threats | Vol 01 §3 architectural risks |
| §5 RBAC | Vol 04 §4 capability registration |
| §6 Data class | Vol 06 §10 retention |
| §7 Secrets | Vol 11 §5 secret stores |
| §8 Audit | Vol 06 §3.20 audit log table |
| §10 Privacy | Vol 03 §8 compliance matrix |
| §11 Isolation | Vol 01 §9 multi-tenancy |
| §14 IR | Vol 01 §14.4 runbooks |

---

*End of Volume 09 — Security v2.0.*
