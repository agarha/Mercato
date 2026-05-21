# Volume 15 — Operational Runbooks

## RB-15.1 Tenant Offboarding

> Document owner: Head of CS + Compliance Lead + Head of SRE
> Status: Implementation Baseline v1.0
> Version: 1.0.0
> Cross-references: Vol 03 §5.11 (subscription cancellation), Vol 09 §10 (DSAR), Vol 06 §10 (retention)

---

### 0. Purpose

Defines the operational process for ending a tenant's relationship with Mercato. This is an enterprise-sales requirement (procurement asks "what happens to our data?"), a GDPR Art. 17 obligation, and a SOC-2 termination control.

The runbook covers:
- voluntary cancellation by the tenant,
- non-payment termination,
- material-breach termination,
- emergency takedown (legal / law enforcement).

---

### 1. Phases of Offboarding

```
ANNOUNCED → READ-ONLY → DATA EXPORT → DELETION SCHEDULED → CERTIFIED DELETED
     T0          T+0          T+0..30d         T+30d                T+30..90d
```

| Phase | Tenant state | Reversible? |
|---|---|---|
| ANNOUNCED | full operations; banner in admin | YES — restore on request |
| READ-ONLY | new orders blocked; existing fulfillment continues | YES |
| DATA EXPORT | export prepared and shipped to tenant | YES (until DELETION SCHEDULED) |
| DELETION SCHEDULED | export shipped; soft-delete + anonymization started | NO — escalation required |
| CERTIFIED DELETED | hard delete completed; certificate of deletion issued | NO |

---

### 2. Trigger Sources & Cadence

| Trigger | Detection | Action |
|---|---|---|
| Tenant clicks "Cancel subscription" | UI event → `mercato.subscription.canceled.v1` | Operator CS confirms via email; ANNOUNCED phase enters |
| Subscription past_due >3 cycles | dunning job | ANNOUNCED, banner shown 14 days, then READ-ONLY |
| Material breach (TOS violation) | Compliance review | Immediate notification; legal review; can skip directly to READ-ONLY |
| Law-enforcement / sanctions list match | Compliance | Emergency takedown — straight to READ-ONLY + freeze export |
| Bankruptcy / insolvency notice | Legal | Mediated by counsel; data retention may be subpoena-driven |

---

### 3. Phase Details

#### 3.1 ANNOUNCED (T0 → end of billing period)

**Actions:**
1. Tenant admin receives email confirming cancellation + expected timeline.
2. Admin UI shows persistent banner: "Marketplace will move to read-only on YYYY-MM-DD. [Cancel cancellation] [Download data export]".
3. Existing operations continue; new vendor registrations disabled; new product publish disabled (configurable per Compliance).
4. Storefront unaffected unless tenant requests early read-only.
5. CS schedules offboarding kickoff call (Enterprise tier mandatory; others optional).

**Data:** no changes.

**Audit:** `mercato.tenant.offboarding.announced.v1` emitted.

#### 3.2 READ-ONLY (start of grace period — typically billing period end + 30 days)

**Actions:**
1. Storefront: shows "This marketplace is no longer accepting new orders" banner; existing orders complete.
2. Vendor SPA: still accessible for fulfillment (mark shipped, respond to disputes, view payouts).
3. Tenant Admin SPA: read-only except for: marketplace settings (closing), data export request, refund issuance, dispute resolution.
4. Payouts continue — vendors must be paid for completed work.
5. Subscriptions auto-renewal disabled; final invoice issued.
6. Webhooks continue firing for in-flight orders.
7. New API key creation disabled; existing keys read-only.

**Data:** no destructive operations.

**Audit:** `mercato.tenant.readonly.v1` emitted with effective date.

#### 3.3 DATA EXPORT (parallel to READ-ONLY; complete before DELETION SCHEDULED)

**Actions:**
1. CS triggers `POST /tenant/{id}/data-export` (super-admin capability).
2. Job assembles per-entity dumps:
   - Vendors (profile, KYC status, payout history) — CSV + JSON.
   - Products (with snapshots) — CSV + JSON.
   - Orders + sub-orders + items + refunds — CSV + JSON.
   - Commissions + payouts ledger — CSV + JSON.
   - Messages — text export.
   - Reviews — CSV.
   - Audit log — JSON.
   - Tenant settings — JSON.
   - Per-vendor 1099-K data (US) — IRS-format.
3. Each dump signed with HMAC; bundled into a single zip per tenant.
4. Zip encrypted with tenant-provided public key OR randomly-generated passphrase delivered out-of-band (Signal/encrypted email).
5. Stored in `mercato-tenant-exports-{region}` bucket with Object Lock until §3.5 completion.
6. Tenant gets signed S3 URL valid 30 days; CS confirms receipt before proceeding.

**SLA:** export complete within 30 calendar days of trigger.

**Audit:** `mercato.tenant.export.requested.v1` and `mercato.tenant.export.completed.v1` with manifest checksum.

#### 3.4 DELETION SCHEDULED (after T+30d unless extended by CS)

**Actions:**
1. CS triggers scheduled deletion via super-admin UI; two-person approval required.
2. PII anonymization sweep:
   - `wp_users` tied to tenant → name/email replaced with `Anonymized-{user_id}@deleted.invalid`.
   - `wp_mercato_vendors` PII fields nulled or pseudonymized.
   - `wp_mercato_suborders.ship_to` redacted to country + city only.
   - `wp_mercato_messages` body replaced with `[anonymized]`.
   - KYC document filenames re-keyed (data already encrypted; key destruction scheduled).
3. **Legal retention preserved:**
   - `wp_mercato_commissions`, `wp_mercato_payouts`, `wp_mercato_suborders` retained per BR-FIN-013 (US 1099 = 7y); pseudonymized to `vendor_id`/`buyer_user_id` placeholders.
   - Audit log retained per Vol 09 §6 (2y hot / 5y cold).
4. Tenant subdomain disabled; CloudFront distribution returns 410 Gone.
5. Vendor accounts notified — vendors may export their own data per BR-VEN-006 within an additional 30 days.

**Audit:** `mercato.tenant.deletion.scheduled.v1`.

#### 3.5 CERTIFIED DELETED (typically T+60..90d)

**Actions:**
1. Tenant container/namespace decommissioned.
2. Aurora schema dropped (Silo/Dedicated) OR Mercato rows removed via partition drop / DELETE on Pooled mode (tenant_id-scoped).
3. KYC documents in S3: lifecycle moves to Glacier with delete pending; per BR-VEN-009 retain 7 years for tax/legal; certificate of deletion deferred to 7y mark unless tenant has provided proof of independent legal compliance.
4. Redis keys purged.
5. OpenSearch indices deleted.
6. KMS CMK (per-tenant): scheduled deletion after 30 days; once cryptographically deleted, S3-encrypted blobs are unrecoverable even if bytes remain.
7. Backup retention: existing backups continue per backup policy; new backups exclude tenant; restoration of deleted tenant from backup requires CISO + legal approval.
8. **Certificate of deletion** issued to tenant (PDF, countersigned by CS + Compliance) listing:
   - Date of phase changes.
   - Categories of data deleted vs. retained (with legal basis).
   - Public hash of the deletion manifest.

**Audit:** `mercato.tenant.deletion.certified.v1` — the audit log itself is retained for 7 years.

---

### 4. Special Cases

#### 4.1 Vendor obligations not yet settled

If at READ-ONLY there are vendors with pending payouts:
- READ-ONLY extends until all payouts complete OR tenant prepays remaining balance.
- Disputes in flight extend READ-ONLY until resolved.
- Chargebacks in dispute window (120d for cards) — funds held in reserve account; refund obligation accrues to platform if tenant won't honor.

#### 4.2 Vendor data ownership

Vendor data is the vendor's data, not the tenant's. Even after tenant deletion, vendors:
- Retain their KYC records (until they themselves close).
- Retain access to their order history via Mercato support for 30 days.
- May export their own data per BR-VEN-006.

#### 4.3 Buyer data

Buyer accounts owned by Mercato platform (not tenant) — unaffected by tenant offboarding except that the tenant's storefront no longer accepts orders.

#### 4.4 Tenant requests immediate deletion (waiver)

A tenant may sign a waiver requesting:
- Skip ANNOUNCED period.
- Skip DATA EXPORT (acknowledge data is lost).
- Skip READ-ONLY grace period.

Two-person operator + legal approval required. Waiver is recorded permanently.

#### 4.5 Tenant returns

A tenant in CERTIFIED DELETED state cannot be restored. They sign up as a new tenant. Anonymized historical records remain in our books for legal retention but are not retrievable to them.

---

### 5. Communication Templates

#### 5.1 Cancellation acknowledgement
> Hi {{tenant_admin}}, we've received your cancellation request. Your marketplace will move to read-only on {{date}}, and your data export will be prepared and shipped by {{date}}. If you change your mind in the meantime, click here: [Reactivate].

#### 5.2 Vendor notification
> Your marketplace {{tenant_name}} is closing. You can continue fulfilling orders until {{date}}, after which the marketplace will close to new orders. Please export your data from your dashboard. Outstanding payouts will be processed normally.

#### 5.3 Certificate of deletion (excerpt)
> This certifies that all tenant data for {{tenant_id}} has been removed from Mercato's primary systems on {{date}}, with the exceptions of records retained for legal/tax/regulatory obligations as listed in Appendix A. The cryptographic manifest hash of deleted records is {{hash}}.

---

### 6. Cross-Tenant Coordination

If a tenant operated a marketplace that hosted vendors who are themselves Mercato tenants (Phase 4 reseller scenario), special coordination required:
- Sub-tenant relationships flagged at offboarding.
- Sub-tenants notified of parent offboarding.
- Sub-tenants may migrate to direct Mercato customership or independent platform.

---

### 7. Compliance Mapping

| Standard | Requirement | How met |
|---|---|---|
| GDPR Art. 17 | Right to erasure | §3.4 anonymization + §3.5 certified deletion |
| GDPR Art. 20 | Right to portability | §3.3 data export |
| CCPA §1798.105 | Right to delete | Same as GDPR Art. 17 |
| SOC-2 CC6.4 | Termination procedures | This runbook |
| PCI-DSS | Data retention | We hold tokens not PAN; tokens deleted with tenant |
| Sarbanes-Oxley | Financial record retention | 7-year legal retention preserved |

---

### 8. Open Items

| Item | Owner | Target |
|---|---|---|
| Tenant-controlled BYOK destruction flow | Security | Phase 4 |
| Automated multi-tenant reseller offboarding | CS + Eng | Phase 4 |
| Self-service vendor data export portal | CS + Eng | Phase 2 |

---

*End of RB-15.1 Tenant Offboarding v1.0.*
