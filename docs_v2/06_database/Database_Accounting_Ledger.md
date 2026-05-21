# Volume 06 Extension — Double-Entry Accounting Ledger

> Document owner: Principal DBA + Head of Finance + Compliance
> Status: Implementation Baseline v2.0 — REQUIRED for any tenant with chargebacks, FX, or multi-vendor refunds
> Version: 1.0.0
> Cross-references: Vol 06 §3.11 (commissions table), §3.12 (payouts table); Vol 03 §5.5 (financial business rules); Vol 04 §8 & §9; Vol 09 §6

---

## 1. Why this extension exists

External review (GPT) correctly observed that the v2.0 ledger — `wp_mercato_commissions` and `wp_mercato_payouts` as append-only delta logs — is **single-entry**. Single-entry ledgers cannot produce a trial balance, cannot detect drift, cannot represent the accounting reality of a marketplace clearing platform funds, paying out to vendors, withholding tax, and reversing on chargebacks.

For a marketplace with:
- Tenant funds held in escrow
- Vendor payable balances
- Commission income
- Tax liabilities
- Refund liabilities
- Chargeback reserves
- Gateway fee expenses
- FX gains/losses

a proper **double-entry general ledger** is required. This document specifies the chart of accounts, the journal entry schema, the posting rules, and the reconciliation procedures that extend Vol 06.

This extension is **mandatory before** the first dollar of tenant chargeback, multi-currency settlement, or refund-across-payout occurs. Implementation must precede those features.

---

## 2. Accounting Posture

### 2.1 Principles
- **Double-entry:** every transaction posts debits = credits, totalling zero.
- **Append-only:** journal entries are immutable; reversals are new entries.
- **Per-tenant ledger:** each tenant has its own books; never co-mingled.
- **Multi-currency:** all entries carry currency_code + a base-currency equivalent computed at post time.
- **Reconciliation-first:** ledger must balance against Stripe Treasury daily within ±$0.01 (BR-PAY-007).
- **Audit:** every journal entry references a domain event (order, refund, payout, dispute, etc.) for traceability.

### 2.2 Standards Alignment
- US GAAP — accrual basis.
- Mapped to common chart-of-accounts conventions; can be exported to QuickBooks IIF, NetSuite SuiteTalk, Xero CSV, Sage formats.

---

## 3. Chart of Accounts (per Tenant)

| # | Code | Name | Type | Normal Balance | Purpose |
|---|---|---|---|---|---|
| 1010 | CASH-STRIPE | Cash — Stripe Treasury | Asset | Dr | Funds held at Stripe |
| 1015 | CASH-BANK | Cash — Operating Bank | Asset | Dr | Tenant's own bank (off-platform, optional sync) |
| 1110 | RECV-BUYER | Receivable — Buyer | Asset | Dr | Authorized but uncaptured |
| 1120 | RECV-CHARGEBACK | Receivable — Chargeback Recovery | Asset | Dr | Won chargebacks awaiting funds |
| 1200 | TAX-WITHHELD | Tax Withholding Receivable | Asset | Dr | If platform withholds from vendor |
| 2010 | PAYABLE-VENDOR | Payable — Vendors | Liability | Cr | Net amount owed to vendors |
| 2020 | PAYABLE-TAX | Tax Liability — Marketplace Facilitator | Liability | Cr | Sales tax collected, remit due |
| 2030 | RESERVE-CHARGEBACK | Chargeback Reserve | Liability | Cr | Hold funds vs anticipated chargebacks |
| 2040 | REFUND-LIABILITY | Refund Liability (Outstanding) | Liability | Cr | Refunds owed but not yet sent |
| 2900 | DEFERRED-REVENUE | Deferred Revenue (Pre-Delivery) | Liability | Cr | Commission unrecognized during hold period |
| 3000 | EQUITY-RETAINED | Retained Earnings | Equity | Cr | Carried forward |
| 4010 | REV-COMMISSION | Commission Revenue | Revenue | Cr | Platform/tenant commission income |
| 4020 | REV-SUBSCRIPTION | Subscription Revenue | Revenue | Cr | Tenant SaaS billing |
| 4030 | REV-OVERAGE | Usage Overage Revenue | Revenue | Cr | AI / vendor / API overages |
| 4040 | REV-LATE-FEE | Late Fees / Penalty Revenue | Revenue | Cr | (Phase 3) |
| 4900 | REV-CONTRA-REFUND | Revenue — Contra Refunds | Revenue (Contra) | Dr | Tracks refund reductions |
| 5010 | EXP-GATEWAY | Stripe Processing Fees | Expense | Dr | Per-transaction fees |
| 5020 | EXP-PAYOUT | Stripe Payout Fees | Expense | Dr | Connect transfer fees |
| 5030 | EXP-CHARGEBACK-FEE | Chargeback Fees | Expense | Dr | $15 per Stripe Dispute |
| 5040 | EXP-REFUND-FEE | Refund Fees | Expense | Dr | Stripe-imposed refund fees (where applicable) |
| 5100 | EXP-AI | AI Provider Cost | Expense | Dr | OpenAI/Anthropic spend |
| 5200 | EXP-KYC | KYC Verification Cost | Expense | Dr | Stripe Identity per verification |
| 5300 | EXP-TAX-ENGINE | Tax Engine Cost | Expense | Dr | TaxJar/Avalara API |
| 6010 | FX-GAIN | FX Gain | Revenue | Cr | Currency gain on settlement |
| 6020 | FX-LOSS | FX Loss | Expense | Dr | Currency loss on settlement |
| 7000 | SUSPENSE | Suspense (Reconciliation) | Asset | Dr | Unmatched amounts — must clear ≤24h |

The account code is a 4-digit number; tenants may extend with sub-accounts (e.g., `2010-VENDOR-7281` for a vendor sub-ledger). Default platform CoA seeded at tenant provisioning.

### 3.1 Vendor Sub-Ledger

Each vendor has a sub-account under `2010 PAYABLE-VENDOR`. Vendor balance = sum of all journal entry lines crediting `2010-VENDOR-{vendor_id}` minus debits. Vendor payout debits this sub-account; refund reversals credit it back.

---

## 4. DDL Additions

```sql
-- The chart of accounts (seeded; tenants may add custom accounts)
CREATE TABLE wp_mercato_ledger_accounts (
  account_id      BIGINT UNSIGNED NOT NULL AUTO_INCREMENT,
  tenant_id       BIGINT UNSIGNED NOT NULL,
  account_code    VARCHAR(32) NOT NULL,    -- e.g., '2010-VENDOR-7281'
  parent_code     VARCHAR(32) DEFAULT NULL,
  name            VARCHAR(128) NOT NULL,
  account_type    ENUM('asset','liability','equity','revenue','expense','contra_revenue','contra_expense') NOT NULL,
  normal_balance  ENUM('debit','credit') NOT NULL,
  is_system       TINYINT(1) NOT NULL DEFAULT 0,
  active          TINYINT(1) NOT NULL DEFAULT 1,
  currency_code   CHAR(3) DEFAULT NULL,    -- NULL = multi-currency, else restricted to one
  created_at      DATETIME(3) NOT NULL DEFAULT CURRENT_TIMESTAMP(3),
  PRIMARY KEY (account_id),
  UNIQUE KEY uk_tenant_code (tenant_id, account_code),
  KEY idx_parent (tenant_id, parent_code)
) ENGINE=InnoDB;

-- Journal entries — the immutable double-entry record
CREATE TABLE wp_mercato_ledger_entries (
  entry_id        BIGINT UNSIGNED NOT NULL AUTO_INCREMENT,
  tenant_id       BIGINT UNSIGNED NOT NULL,
  journal_id      CHAR(26) NOT NULL,           -- ULID; one ID per multi-line transaction
  entry_seq       SMALLINT UNSIGNED NOT NULL,  -- 1..N within journal_id
  account_code    VARCHAR(32) NOT NULL,
  direction       ENUM('debit','credit') NOT NULL,
  amount_minor    BIGINT UNSIGNED NOT NULL,    -- always positive
  currency_code   CHAR(3) NOT NULL,
  base_amount_minor BIGINT NOT NULL,           -- in tenant's base currency, signed (debits +, credits +)
  fx_rate         DECIMAL(18,8) NOT NULL DEFAULT 1.0,
  -- Causation linking
  event_type      VARCHAR(96) NOT NULL,        -- e.g., mercato.commission.recorded.v1
  source_table    VARCHAR(64) DEFAULT NULL,    -- e.g., wp_mercato_commissions
  source_id       BIGINT UNSIGNED DEFAULT NULL,
  description     VARCHAR(255) NOT NULL,
  posted_at       DATETIME(3) NOT NULL DEFAULT CURRENT_TIMESTAMP(3),
  posted_by       BIGINT UNSIGNED DEFAULT NULL, -- user_id or NULL=system
  PRIMARY KEY (entry_id),
  KEY idx_tenant_journal (tenant_id, journal_id),
  KEY idx_tenant_account_posted (tenant_id, account_code, posted_at),
  KEY idx_source (source_table, source_id)
) ENGINE=InnoDB
PARTITION BY RANGE COLUMNS (posted_at) (
  PARTITION p2026m01 VALUES LESS THAN ('2026-02-01'),
  PARTITION p2026m02 VALUES LESS THAN ('2026-03-01'),
  -- maintained monthly
  PARTITION pmax    VALUES LESS THAN MAXVALUE
);

-- Journal envelopes (one row per logical transaction)
CREATE TABLE wp_mercato_ledger_journals (
  journal_id      CHAR(26) NOT NULL,
  tenant_id       BIGINT UNSIGNED NOT NULL,
  journal_type    VARCHAR(48) NOT NULL,        -- 'order_capture' | 'commission' | 'payout' | ...
  description     VARCHAR(255) NOT NULL,
  posted_at       DATETIME(3) NOT NULL DEFAULT CURRENT_TIMESTAMP(3),
  posted_by       BIGINT UNSIGNED DEFAULT NULL,
  reversed_by     CHAR(26) DEFAULT NULL,       -- if this journal was reversed
  reverses        CHAR(26) DEFAULT NULL,       -- if this journal is itself a reversal
  correlation_id  CHAR(26) DEFAULT NULL,       -- causation chain
  PRIMARY KEY (journal_id),
  KEY idx_tenant_type_posted (tenant_id, journal_type, posted_at),
  KEY idx_reverses (reverses)
) ENGINE=InnoDB;

-- Account balance snapshot (materialized for performance; refreshed nightly + on-demand)
CREATE TABLE wp_mercato_ledger_balances (
  tenant_id       BIGINT UNSIGNED NOT NULL,
  account_code    VARCHAR(32) NOT NULL,
  as_of_date      DATE NOT NULL,
  base_currency   CHAR(3) NOT NULL,
  debit_minor     BIGINT UNSIGNED NOT NULL DEFAULT 0,
  credit_minor    BIGINT UNSIGNED NOT NULL DEFAULT 0,
  balance_minor   BIGINT NOT NULL,             -- signed
  updated_at      DATETIME(3) NOT NULL DEFAULT CURRENT_TIMESTAMP(3) ON UPDATE CURRENT_TIMESTAMP(3),
  PRIMARY KEY (tenant_id, account_code, as_of_date)
) ENGINE=InnoDB;
```

### 4.1 Constraints

The ledger module enforces **at the application layer** (DB triggers as backup):

1. **Balanced journal**: for any `journal_id`, sum of debit base_amount_minor = sum of credit base_amount_minor. Violation → reject the journal entirely.
2. **Append-only**: application role `mercato_app` has only INSERT/SELECT on `wp_mercato_ledger_*`. UPDATE/DELETE revoked.
3. **No edits**: corrections post a reversing journal (`reverses` column links) + a corrected journal.

```sql
GRANT INSERT, SELECT ON wp_mercato_ledger_entries TO 'mercato_app'@'%';
REVOKE UPDATE, DELETE ON wp_mercato_ledger_entries FROM 'mercato_app'@'%';
GRANT INSERT, SELECT ON wp_mercato_ledger_journals TO 'mercato_app'@'%';
REVOKE UPDATE, DELETE ON wp_mercato_ledger_journals FROM 'mercato_app'@'%';
```

---

## 5. Posting Rules — Canonical Patterns

Every domain event posts a journal. Here are the most important patterns.

### 5.1 Order Capture (buyer pays $120 incl. $10 tax + $10 shipping; 2 vendors)

Example: Vendor A items subtotal $50; Vendor B items subtotal $50; tax $5 each; shipping $5 each. Tenant facilitator state.

**Journal type: `order_capture`**
```
Dr  CASH-STRIPE                      $120.00
   Cr  PAYABLE-VENDOR (A sub)             $50.00  (subtotal, pre-commission)
   Cr  PAYABLE-VENDOR (A sub) shipping     $5.00
   Cr  PAYABLE-VENDOR (B sub)             $50.00
   Cr  PAYABLE-VENDOR (B sub) shipping     $5.00
   Cr  PAYABLE-TAX                        $10.00
```

(Note: Mercato posts the **gross** to vendor payable first; commission is recorded as a separate journal at the moment of commission accrual, debiting vendor payable + crediting commission revenue. This separation makes reversal mechanics cleaner.)

### 5.2 Commission Accrual (10% of subtotal, post-capture, after hold)

For Vendor A's $50 subtotal:
```
Dr  PAYABLE-VENDOR (A sub)           $5.00
   Cr  REV-COMMISSION                     $5.00
```

(Same for Vendor B.)

### 5.3 Gateway Fee Recognition

When Stripe charge fee comes back (typically 2.9% + $0.30 → ~$3.78 on $120):
```
Dr  EXP-GATEWAY                      $3.78
   Cr  CASH-STRIPE                        $3.78
```

### 5.4 Payout to Vendor (weekly batch)

Vendor A is owed $50 + $5 shipping − $5 commission = $50. Stripe payout fee $0.25:
```
Dr  PAYABLE-VENDOR (A sub)           $50.00
   Cr  CASH-STRIPE                        $49.75
   Cr  EXP-PAYOUT                          $0.25
```
(Actually the payout fee is borne by the tenant typically; CoA placement is tenant-policy configurable.)

### 5.5 Refund (Partial — Vendor A item, $25)

Tax on the refunded portion = $2.50, shipping retained per tenant policy:
```
Dr  REV-CONTRA-REFUND                $25.00
Dr  PAYABLE-TAX                       $2.50    (tax remitted back to buyer)
   Cr  CASH-STRIPE                        $27.50
-- + commission reversal:
Dr  REV-COMMISSION                    $2.50
   Cr  PAYABLE-VENDOR (A sub)              $2.50  (vendor balance credited back)
```

If Vendor A has already been paid out, the credit to `PAYABLE-VENDOR (A sub)` creates a positive balance that absorbs from next payout, OR the system creates a recovery receivable per BR-COMM-006.

### 5.6 Chargeback (Buyer wins; Stripe debits $50 + $15 fee)

```
Dr  REV-CONTRA-REFUND                $50.00
Dr  EXP-CHARGEBACK-FEE               $15.00
   Cr  CASH-STRIPE                        $65.00
-- + vendor liability reversal:
Dr  PAYABLE-VENDOR (A sub)           $5.00 (commission already taken)
   Cr  REV-COMMISSION                     $5.00
-- + vendor debt:
Dr  RECV-CHARGEBACK                  $45.00
   Cr  PAYABLE-VENDOR (A sub)              $45.00 (debt against vendor's future earnings)
```

### 5.7 FX Settlement (Vendor in EUR; tenant base USD)

When EUR payout settles:
```
Dr  PAYABLE-VENDOR (A sub)           €100.00 (base $112.00 captured at order)
   Cr  CASH-STRIPE                        €99.50 (base $111.50 at settlement)
   Cr  FX-GAIN                            base $0.50
```

If EUR weakened, FX-LOSS is debited instead.

### 5.8 Subscription Revenue Recognition (Tenant SaaS)

Each invoice settled:
```
Dr  CASH-STRIPE                      $499.00
   Cr  REV-SUBSCRIPTION                   $499.00
```

For annual prepay with monthly recognition, Mercato uses:
```
-- At billing:
Dr  CASH-STRIPE                       $4990.00
   Cr  DEFERRED-REVENUE                   $4990.00
-- Each month:
Dr  DEFERRED-REVENUE                   $415.83
   Cr  REV-SUBSCRIPTION                   $415.83
```

---

## 6. The Posting Engine (Application)

A `mercato-core/Ledger\PostingEngine` provides:

```php
$engine->open($journalType, $tenantId, $description, $correlationId);
$engine->debit($accountCode, $amountMinor, $currency, $eventType, $sourceRef);
$engine->credit($accountCode, $amountMinor, $currency, $eventType, $sourceRef);
$engine->commit(); // validates balanced, persists, emits mercato.ledger.posted.v1
```

The commit operation:
1. Validates debits = credits.
2. Inserts in one DB transaction.
3. Refreshes affected balance snapshots.
4. Emits an outbox event.

Failure to balance → `LedgerImbalanceException`; nothing persists.

### 6.1 Adapters per Domain

| Domain event | Posting adapter | Journal type |
|---|---|---|
| `mercato.payment.complete.v1` | `OrderCapturePoster` | `order_capture` |
| `mercato.commission.recorded.v1` | `CommissionPoster` | `commission` |
| `mercato.refund.created.v1` | `RefundPoster` | `refund` |
| `mercato.payout.succeeded.v1` | `PayoutPoster` | `payout` |
| Stripe `charge.refunded.failed` webhook | `PayoutFailurePoster` | `payout_reverse` |
| Stripe `charge.dispute.closed.lost` | `ChargebackPoster` | `chargeback` |
| `mercato.invoice.paid.v1` (subscription) | `SubscriptionPoster` | `subscription_revenue` |
| `mercato.ai.usage.v1` (aggregate hourly) | `AIUsagePoster` | `ai_expense_accrual` |

Each adapter is single-purpose and unit-tested against fixtures of the event payload.

---

## 7. Reconciliation

### 7.1 Daily Reconciliation Job

Runs nightly. For each tenant:
1. Pull all Stripe Treasury transactions for the date.
2. Pull all ledger entries posting to `CASH-STRIPE` for the date.
3. Compute net Stripe-side total and net ledger-side total in base currency.
4. **PASS:** |drift| ≤ $0.01. Persist `reconciliation_status=ok`.
5. **FAIL:** drift > $0.01. Persist `reconciliation_status=drift`; route to `SUSPENSE` account; raise P1 alert.

Suspense entries must clear within 24h or escalate.

### 7.2 Trial Balance

A view (or scheduled report) computes per tenant:
```sql
SELECT account_code, SUM(CASE WHEN direction='debit'  THEN base_amount_minor ELSE 0 END) AS debits,
                     SUM(CASE WHEN direction='credit' THEN base_amount_minor ELSE 0 END) AS credits
FROM wp_mercato_ledger_entries
WHERE tenant_id = ? AND posted_at < ?
GROUP BY account_code;
```

Sum of debits across all accounts = sum of credits. Anything else = corruption.

### 7.3 Vendor Statement

Per vendor monthly, generated by:
```sql
SELECT *
FROM wp_mercato_ledger_entries
WHERE tenant_id = ? AND account_code = ? -- e.g. 2010-VENDOR-7281
  AND posted_at BETWEEN ? AND ?
ORDER BY posted_at;
```

### 7.4 Financial Exports

Mercato exports to:
- QuickBooks IIF
- NetSuite SuiteTalk
- Xero CSV
- Sage 50 CSV
- Generic GL CSV

Mapping config per tenant in `wp_mercato_tenant_settings.accounting.export_mapping`.

---

## 8. Migration from v2.0 Single-Entry Ledger

For tenants who launched before this extension:

1. Read all `wp_mercato_commissions` and `wp_mercato_payouts` rows.
2. Reconstruct journals per pattern §5 for historical events.
3. Post into `wp_mercato_ledger_entries` with `posted_at` matching original event time.
4. Verify reconstructed trial balance matches Stripe Treasury historical balance.
5. Switch live posting engine on.
6. Leave `wp_mercato_commissions` and `wp_mercato_payouts` as the operational ledgers; the new `wp_mercato_ledger_*` becomes the accounting ledger.

The two ledger systems coexist:
- **Operational ledger** (`wp_mercato_commissions`, `wp_mercato_payouts`) = source of truth for *vendor-facing* statements + payout eligibility.
- **Accounting ledger** (`wp_mercato_ledger_*`) = source of truth for *finance*, accounting exports, trial balance, tax filing.

They must agree daily; reconciliation job verifies.

---

## 9. Compliance Implications

- **Audit trail**: ledger entries reference event_type + source_table/source_id, providing forensic traceability.
- **SOC-2**: CC7.2 evidence — ledger immutability + reconciliation runs.
- **Tax filing**: facilitator tax exports straight from `2020 PAYABLE-TAX` activity.
- **Vendor 1099-K**: total of `4010 REV-COMMISSION` (negative for vendor's share calculation) + payout totals on vendor sub-account → 1099-K box 1a.

---

## 10. Phasing

- **MVP**: do NOT implement double-entry. Keep simple `wp_mercato_commissions` + `wp_mercato_payouts`.
- **Phase 2**: implement ledger module before first chargeback or multi-currency tenant.
- **Phase 3**: backfill historical activity from operational ledger.
- **Phase 4**: BYOK + per-tenant export connectors.

This phasing aligns with §11 of Vol 00 MVP Cut.

---

## 11. Cross-References

| Topic | See |
|---|---|
| Operational ledger | Vol 06 §3.11–§3.14 |
| Business rules for fees | Vol 03 §5.5 |
| FX policy | Vol 03 §6.4 |
| Stripe webhooks | Vol 04 §13 |
| Tax facilitator | Vol 03 §8.4 |
| Compliance | Vol 09 §15 |

---

*End of Volume 06 Extension — Double-Entry Accounting Ledger v1.0.*
