# Volume 13 — WooCommerce / HPOS Compatibility & Mapping

> Document owner: WordPress/WooCommerce Platform Architect
> Status: Implementation Baseline v2.0
> Version: 1.0.0
> Cross-references: Vol 01 §6, Vol 04 §3.6 (hook map) & §7, Vol 06 §3.9 (orders DDL), Vol 09 §12

---

## 1. Why this document exists

External review (GPT) flagged that the architecture's heavy modification of WooCommerce — shadow catalog, custom sub-orders, custom ledgers, event outbox, REST overlays — creates real compatibility risk that isn't explicitly addressed. This document is the contract that bounds where Mercato touches WooCommerce, how it maps to WC's native object model, and what the operational rules are for keeping Mercato compatible across WC LTS upgrades.

It is meant to be read by anyone working on `mercato-orders`, `mercato-products`, `mercato-payouts`, or any plugin that reads or writes WooCommerce data.

---

## 2. WooCommerce Compatibility Posture

### 2.1 Pinned Versions

- WordPress: 6.4+ LTS (auto security patches; minor upgrades quarterly under change control).
- WooCommerce: 8.x LTS — pinned to a single LTS minor; major version upgrades only after the WC LTS cycle and regression suite green.
- **HPOS (High-Performance Order Storage) is MANDATORY.** Mercato refuses to activate against legacy `wp_posts`-backed orders.

### 2.2 Compatibility Posture Statement

> Mercato treats WooCommerce as a black-box dependency through documented hooks and CRUD APIs. We never edit WC core files, never modify WC tables directly, and never bypass WC's order lifecycle.

The single exception is the **shadow projection** in §5 — a tightly-scoped, well-tested write path into a small set of WC tables for cart compatibility only.

### 2.3 Plugin-Conflict Strategy

| Type of third-party plugin | Mercato position |
|---|---|
| Shipping plugins (Shippo, EasyPost, custom carriers) | **Compatible** if they read `WC_Order` and `WC_Order_Item`. Mercato sub-orders are visible to these plugins via shadow projection. |
| Tax plugins (TaxJar, Avalara, native) | **Compatible** with caveat: Mercato's tax engine (Vol 04 §17) overrides at sub-order level; tenant must disable conflicting tax plugins. |
| Payment plugins (alternative gateways) | **Conflict-prone.** Only Stripe Connect (`mercato-stripe-connect`) supported at MVP. Other gateways may break payout splitting. |
| Caching plugins (W3 Total Cache, WP Rocket, LiteSpeed) | **Compatible** with documented cache bust rules. Mercato emits cache-invalidation hooks on price/stock changes. |
| Backup plugins (UpdraftPlus, BlogVault) | **Compatible.** Must include `wp_mercato_*` tables. |
| Multilingual plugins (WPML, Polylang) | **Conflict-prone.** Phase 3+ — disable for MVP. |
| Page builders (Elementor, Divi) | **Compatible** for content pages. Vendor SPA and admin SPA are excluded — page builders cannot edit them. |
| Other multivendor plugins (Dokan, WCFM, WC Vendors, Marketplace) | **Hard conflict.** Mercato cannot coexist. Migration tool will be provided at P4 (`mercato-migration`). |
| Subscription plugins (WC Subscriptions) | **Conflict-prone.** Phase 3+ — `mercato-subscriptions` is the supported path. |

A compatibility test matrix runs in CI against the top-50 WooCommerce plugins (by install count) to detect conflicts early.

### 2.4 WC Upgrade Discipline

- Subscribe to WC LTS release notes; review breaking changes before each minor.
- Pin the WC version in `composer.json` or container image at production exactly.
- Run full regression suite (Vol 10 QA) against new WC version in staging for 7 days before promoting.
- The `Mercato\Core\WooCommerce\HookAdapter` is the single point of WC integration; version-specific changes are localized to this class.

---

## 3. The Mercato ↔ WooCommerce Object Mapping

This is the canonical mapping. Implementation MUST not introduce ad-hoc mappings outside this table.

### 3.1 Order Mapping

| WC Object | Mercato Object | Relationship | Notes |
|---|---|---|---|
| `WC_Order` (HPOS `wp_wc_orders.id`) | `wp_mercato_suborders.parent_order_id` | 1 : N | Parent order is canonical for buyer-facing state, payment, totals |
| `WC_Order_Item` (HPOS `wp_wc_order_items.order_item_id`) | `wp_mercato_suborder_items.item_id` (separately keyed) | 1 : 1 | Mercato item has a `wc_order_item_id` column |
| `WC_Order_Refund` (HPOS) | `wp_mercato_refunds.refund_id` | 1 : 1 | Mercato refund stores `wc_refund_id` |
| `WC_Customer` (user) | `wp_mercato_suborders.buyer_user_id` (wp_users.ID) | 1 : N | |
| (no WC concept) | `wp_mercato_suborders.suborder_id` (vendor-keyed) | N per parent | Pure Mercato construct |
| WC order status | `wp_mercato_suborders.status` | independent state machines | See §3.3 |
| Payment intent (Stripe-side) | `wp_mercato_suborders.parent_order_id` → meta `_stripe_payment_intent` | 1 : 1 | Stripe is canonical for payment lifecycle |

### 3.2 Required Columns to Add to `wp_mercato_suborders` (delta to Vol 06)

Vol 06 §3.9 defined `parent_order_id` referencing WC order ID. To make the mapping bulletproof, add:

```sql
ALTER TABLE wp_mercato_suborders
  ADD COLUMN parent_order_uuid VARCHAR(36) DEFAULT NULL COMMENT 'WC order UUID for distributed lookup',
  ADD COLUMN wc_payment_intent VARCHAR(64) DEFAULT NULL COMMENT 'Stripe PaymentIntent ID for traceability',
  ADD KEY idx_wc_payment_intent (wc_payment_intent);

ALTER TABLE wp_mercato_suborder_items
  ADD COLUMN wc_order_item_id BIGINT UNSIGNED DEFAULT NULL,
  ADD KEY idx_wc_oi (wc_order_item_id);

ALTER TABLE wp_mercato_refunds
  ADD COLUMN wc_refund_id BIGINT UNSIGNED DEFAULT NULL,
  ADD KEY idx_wc_refund (wc_refund_id);
```

### 3.3 Status Mapping

WC order status and Mercato sub-order status are **separate state machines**. They synchronize via hook adapter.

| WC parent status | Mercato sub-order status (per sub-order) | Action |
|---|---|---|
| `pending` | n/a — no Mercato sub-order yet | Wait for payment |
| `processing` (paid) | `created` → `acknowledged` | Vendor must ack within 24h |
| `on-hold` | `created` (frozen) | Vendor blocked from acknowledging until released |
| `failed` | `payment_failed` | Sub-order not actionable |
| `cancelled` | `auto_canceled` or `closed` | Per cancellation source |
| `completed` | sum of sub-order states ≥ all `completed` | Parent moves to completed only after all sub-orders settle |
| `refunded` | one or more sub-orders `refunded` | Partial refund supported; parent stays `completed` if partial |

Two-direction synchronizer (`mercato-orders/SubOrderStatusSynchronizer`) handles transitions. Race conditions resolved by per-aggregate event ordering (Vol 01 §7.3).

### 3.4 Refund Mapping

When a refund happens:
1. WC creates `WC_Order_Refund` (parent-level OR item-level).
2. `woocommerce_refund_created` hook fires.
3. Mercato hook adapter:
   - Identifies which sub-order(s) the refunded items belong to.
   - Creates `wp_mercato_refunds` row(s) — one per sub-order affected.
   - Stores `wc_refund_id` linking back.
   - Emits `mercato.refund.created.v1` event.
4. `mercato-commissions` consumer:
   - Calculates proportional commission reversal.
   - Inserts negative commission row (parent_commission_id pointing to original).

**Refund correctness rules:**
- Refund amount ≤ original sub-order total (validated pre-create).
- Partial refunds are item-level OR amount-only; both supported.
- A refund cannot cross sub-order boundaries — if buyer wants refund spanning vendors A and B, two refunds are issued.
- Refund of shipping is handled per tenant policy (BR-FIN-008).

### 3.5 Coupons & Discounts

WooCommerce coupons apply to the parent cart pre-split. Mercato must allocate discount across sub-orders.

Algorithm (in `mercato-orders/CartSplitter`):

```
For each cart item:
  subtotal_pre_coupon[item] = quantity * unit_price
For each coupon applied:
  Get coupon's discount amount: D
  total = sum(subtotal_pre_coupon)
  For each item:
    allocated_discount[item] += D * (subtotal_pre_coupon[item] / total)
For each sub-order:
  discount = sum(allocated_discount[items in this sub-order])
  subtotal = sum(subtotal_pre_coupon) - discount
```

**Coupon cost responsibility** (BR-COMM-004) is tenant-configurable:
- `vendor` — vendor absorbs (commission base = post-discount).
- `tenant` — tenant absorbs (commission base = pre-discount, but commission is calculated on the post-discount item price; difference paid out of tenant's take).
- `split` — pro-rata between vendor and tenant.

### 3.6 Tax Mapping

WC computes tax at checkout. Mercato consumes the calculated tax per item and snapshots into `wp_mercato_tax_records` (Vol 06 §3.32).

Per BR-COMM-002, commission base **excludes** tax. The tax allocation per sub-order is:

```
tax_subOrder = sum(item.tax for item in sub_order.items)
```

For marketplace-facilitator jurisdictions (Vol 03 §8.4), platform collects + remits. The tax engine (Vol 04 §17) flags `facilitator=1` and the corresponding `wp_mercato_tax_records.facilitator=1`.

### 3.7 Shipping Mapping

WC has shipping zones / rates. Mercato adds **vendor-defined shipping zones** (`wp_mercato_shipping_zones`, Vol 06 §3.36) which override WC's at the sub-order level.

At cart calculation:
1. For each vendor in cart, query `wp_mercato_shipping_zones` for the buyer's ship-to country.
2. Compute per-vendor shipping cost.
3. Add per-vendor shipping line to cart display.
4. Pass through to WC as a fee on the parent order.
5. Sub-orders persist `shipping_minor` per their vendor's calculation.

Refund of shipping: per BR-ORD policy; default = refunded only if items refunded sum to full sub-order.

### 3.8 Reports Compatibility

WooCommerce reports (Analytics) query HPOS tables. Mercato sub-orders are **invisible** to native WC reports. This is acceptable because:
- Tenant admins use Mercato Reports (Vol 04 §14), not WC Analytics.
- Vendor users never access wp-admin.

If a tenant wants to keep using WC Analytics for parent-order metrics, that continues to work. Mercato Reports adds vendor-level / sub-order-level metrics WC cannot produce.

### 3.9 Inventory Mapping

WooCommerce stock is tracked on the WC product (`wp_wc_product_meta_lookup.stock_quantity`). Mercato `wp_mercato_products.stock_qty` is the source of truth; the shadow projector synchronizes.

**Race condition guard:** stock decrement during checkout uses `SELECT ... FOR UPDATE` on `wp_mercato_products`; the projection to WC is async post-commit.

For variations (P2), `wp_mercato_product_variations.stock_qty` projects to `wp_wc_product_meta_lookup` per-variation.

---

## 4. WC Hook Dependency Surface (canonical)

Same as Vol 04 §3.6 — reproduced here with added "WC LTS minor that introduced this hook" so we know what version bumps may move things.

| WC Hook | Introduced | Mercato Adapter | Critical |
|---|---|---|---|
| `plugins_loaded` (WP) | core | `Core\Bootstrap` | YES |
| `init` (WP) | core | `Core\Capabilities\Register` | YES |
| `rest_api_init` (WP) | core | `Core\REST\Router` | YES |
| `woocommerce_init` | WC 2.x | refuses if HPOS off | YES |
| `woocommerce_cart_calculate_fees` | WC 2.x | `Orders\Cart\FeeCalculator` | YES |
| `woocommerce_checkout_create_order` | WC 3.x | `Orders\Checkout\SplitValidator` | YES |
| `woocommerce_new_order` | WC 2.x | `Orders\Splitter` | YES |
| `woocommerce_order_status_changed` | WC 2.x | `Orders\Status\Synchronizer` | YES |
| `woocommerce_payment_complete` | WC 2.x | `Payments\Confirmer` | YES |
| `woocommerce_refund_created` | WC 2.5 | `Refunds\Reverser` | YES |
| `save_post_product` (WP) | core | `Products\ShadowGuard` | YES |
| `template_redirect` (WP) | core | `Storefront\VendorPageRouter` | NO |
| `woocommerce_email_classes` | WC 2.x | `Notifications\WCEmailOverride` | NO |
| `woocommerce_thankyou` | WC 2.x | `Notifications\OrderReceipt` | NO |
| `woocommerce_order_refunded` | WC 2.5 | `Refunds\PostRefundActions` | YES |
| `woocommerce_after_register_post_type` | WC 3.x | (not used; documented for awareness) | — |
| `woocommerce_admin_order_actions_end` | WC 4.x | admin UI integration | NO |

**Hook surface contract:**
- The hook adapter is the single point of contact with WC.
- A hook NOT in this list MUST NOT be subscribed to by a Mercato plugin without architectural review.
- Removing or renaming any "YES critical" hook in a WC version = blocker for upgrade.

---

## 5. The Shadow Projection Contract

`wp_mercato_products` is the source of truth. WC tables receive a tightly-scoped projection so the cart, checkout, and PDP can function without Mercato modification of WC core.

### 5.1 Projected Fields ONLY

For each Mercato product → `wp_posts` + `wp_postmeta` + `wp_wc_product_meta_lookup`:

| Mercato column | WC destination | Notes |
|---|---|---|
| `title` | `wp_posts.post_title` | |
| `slug` | `wp_posts.post_name` | |
| `description_html` | `wp_posts.post_content` | rendered from MD |
| `visibility` (`published`/`draft`/`archived`) | `wp_posts.post_status` | mapped: published→publish, draft→draft, archived→trash |
| `vendor_id` | `wp_postmeta._mercato_vendor_id` | for ownership filtering |
| `wc_product_id` | (back-ref into `wp_mercato_products.wc_product_id`) | one-time at create |
| `price_minor` (in DECIMAL form) | `wp_wc_product_meta_lookup.min_price` & `max_price` | |
| `stock_qty` | `wp_wc_product_meta_lookup.stock_quantity` | |
| `stock_policy` | `wp_wc_product_meta_lookup.stock_status` | track→`instock`/`outofstock` |
| `primary_image_id` | `_thumbnail_id` | |

**Forbidden writes to WC tables:**
- `wp_postmeta` keys for marketplace business data (anything starting `_mercato_*` except `_mercato_vendor_id`).
- Any column not listed above.

### 5.2 The Shadow Guard

`Products\ShadowGuard` hooks `save_post_product` and:
- If WC admin attempts to edit a Mercato-owned product, blocks the save and shows admin notice "This product is managed by Mercato — edit via Vendor Dashboard."
- WP-admin product edits of non-Mercato products are unaffected.

### 5.3 Projection Idempotency

The projector is idempotent. Replays must produce the same final state. Conflict resolution: Mercato wins.

---

## 6. HPOS Migration Strategy

If a tenant brings an existing WC store (with legacy orders) into Mercato:

### 6.1 Pre-flight
- Verify HPOS is **disabled** but ready to enable in their WC version.
- Verify all required Mercato tables can be created.
- Verify no conflicting multivendor plugin is active.

### 6.2 Step-by-step
1. **Snapshot.** Database backup before any change.
2. **Disable conflicting plugins** (other multivendor).
3. **Run WC HPOS sync** to populate `wp_wc_orders` from `wp_posts`.
4. **Verify HPOS data integrity** via WC's diagnostic tool.
5. **Switch to HPOS authoritative** mode in WC settings.
6. **Install Mercato** (one bundled suite per Vol 14).
7. **Activate `mercato-core`** — auto-creates `wp_mercato_*` tables and runs migrations.
8. **Activate domain plugins** in dependency order.
9. **Reconcile legacy orders** — for orders without vendor splits, treat as single-vendor and backfill `wp_mercato_suborders` with the tenant's primary vendor.
10. **Soak test** — read-only mode for 48 hours with shadow writes.

### 6.3 Rollback
- Within 24h: disable Mercato, switch HPOS back if needed.
- After 24h: restore from snapshot.

### 6.4 Failed Migration Recovery
A migration log table (`wp_mercato_migrations`) captures every step. Failed migrations leave the system in a known state with manual intervention path documented.

---

## 7. Known Compatibility Caveats

| Caveat | Impact | Mitigation |
|---|---|---|
| WC's "Order received" page may show single-vendor info | Buyer confusion | Mercato overrides the thankyou template to show sub-order references |
| WC native emails are vendor-agnostic | Vendor doesn't know to fulfill | Mercato sends sub-order-specific emails via `mercato-notifications` |
| WC reports show parent orders only | Admin missing vendor metrics | Use Mercato Reports instead |
| WC product duplication function | Could create orphan Mercato-less products | Hook intercepts and creates the Mercato record |
| Some shipping zone plugins write directly to WC shipping tables | May not see Mercato vendor zones | Disable conflicting plugins; use Mercato shipping zones only |
| Block editor (Gutenberg) blocks for products | Cannot edit Mercato products from WP admin | Vendors use Vendor SPA; admin blocked by ShadowGuard |
| WC REST API exposes products | Vendor product data may be visible | Mercato disables `/wp-json/wc/v3/products` by default for non-admin users (override via setting) |

---

## 8. Testing & Verification

Vol 10 QA includes a dedicated **WC Compatibility Test Suite**:

- WC version regression: full suite runs against each WC LTS minor under support.
- Top-50 plugin compatibility: install + smoke test with each plugin.
- HPOS sync correctness: migrate 10k legacy orders + verify Mercato sub-orders backfill correctly.
- Refund correctness: every refund pattern (parent / item-level / partial amount) verified.
- Coupon allocation: 5 coupon types × 3 vendor configurations.
- Tax allocation: 5 jurisdictions × marketplace-facilitator on/off.

---

## 9. Cross-References

| Topic | See |
|---|---|
| Hook map detail | Vol 04 §3.6 |
| Order DDL | Vol 06 §3.9 |
| Cart split functional spec | Vol 04 §7 |
| Refund flow | Vol 04 §7 + §11 |
| Migration importer (P4) | Vol 04 §23 |
| QA WC test suite | Vol 10 §3 + §6 |

---

*End of Volume 13 — WooCommerce/HPOS Compatibility & Mapping v1.0.*
