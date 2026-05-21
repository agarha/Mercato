# Volume 14 — Plugin Packaging Strategy

> Document owner: WordPress/WooCommerce Platform Architect + Principal Platform Eng
> Status: Implementation Baseline v1.0
> Version: 1.0.0
> Cross-references: Vol 01 §4 (plugin topology), Vol 04 §4 (mercato-core), Vol 11 §7 (CI/CD)

---

## 1. Why this document exists

External review flagged that one `mercato-core` + 18 domain plugins + 9 integration adapters = **27 separately-installable WordPress plugins** is operationally painful:

- Tenants install/upgrade 27 things instead of 1.
- Version drift between dependent plugins becomes likely.
- Plugin activation order matters and is error-prone.
- WP plugin update screen becomes noisy.
- Dependency conflicts (a tenant disables `mercato-vendors` without realizing it breaks `mercato-orders`) cause hard-to-diagnose failures.

This document specifies the packaging strategy that resolves the tension between **architectural modularity** (good) and **deployment fragmentation** (bad).

---

## 2. The Strategy: Modular Internally, Bundled Externally

### 2.1 Core rule

> Mercato is developed as 27 independent modules in a monorepo, and shipped as **one** WordPress plugin bundle ("Mercato Suite") that activates the modules per the tenant's plan.

This is the same pattern Sensei, ACF, WooCommerce itself (with extensions), Yoast, and Jetpack use successfully at scale.

### 2.2 Visualisation

```
WordPress plugin slot:
  mercato-suite/          ← ONE plugin installed in wp-content/plugins/
    mercato-suite.php     ← main plugin file (registers loader)
    composer.json
    modules/
      mercato-core/       ← module (= conceptually a plugin in Vol 01)
      mercato-vendors/
      mercato-products/
      mercato-orders/
      …
      mercato-stripe-connect/
      mercato-sendgrid/
      mercato-aws-s3/
    bundled-assets/
      js/, css/, etc.
    languages/
    vendor/               ← composer deps
```

### 2.3 What "module" means inside the suite

Each module:
- Has its own folder, `composer.json`, `module.json` manifest (per Vol 04 §4.2).
- Has its own tests, migrations, REST routes, capabilities, events.
- Is bootable independently by `mercato-core` (DI container).
- Is enabled/disabled per **tenant**, not per WordPress site.
- Versions in lockstep with the suite (all modules ship as a single suite version, e.g. v2.0.1).

### 2.4 Why not really 27 plugins?

| Concern | If 27 plugins | If 1 bundle |
|---|---|---|
| WP admin clarity | 27 rows in Plugins screen | 1 row — Mercato Suite |
| Dependency enforcement | hard; each plugin checks others | trivial; modules controlled by core |
| Update coordination | tenants update 27 things | tenants update 1 thing |
| Activation order | error-prone (must activate core first, then deps in order) | suite controls boot order internally |
| Version skew | possible (mercato-orders v1.2 with mercato-commissions v1.1) | impossible (suite ships one set) |
| Marketplace distribution | 27 listings to maintain | 1 listing |
| WordPress.org rules | each plugin needs separate review | one review |

---

## 3. Module Activation Per Tenant (not per site)

Even though all 27 modules ship inside the suite, **only the modules a tenant's plan/feature flags allow are bootstrapped at runtime.**

### 3.1 Boot sequence
1. WP loads `mercato-suite/mercato-suite.php` at `plugins_loaded` p=1.
2. `Mercato\Core\Bootstrap` initializes the DI container.
3. Core reads the tenant's capability JWT (cached in `wp_options`).
4. For each module in `modules/`, check if the JWT's `features` array enables it.
5. Only enabled modules are loaded (autoloader + `register()`).
6. Topological sort guarantees dependency order (Vol 01 §4.4).

A disabled module's database tables still exist (so re-enabling is hot), but its routes, hooks, events, and admin UI do not load.

### 3.2 Tenant tier examples

| Plan | Active modules |
|---|---|
| Starter | core, vendors, products, orders, commissions, payouts, messaging, notifications, kyc-kyb, stripe-connect, sendgrid, aws-s3, enterprise (subset) |
| Pro | + reviews, disputes, reports, search, tax-engine, taxjar, postmark |
| Business | + subscriptions, twilio, fraud-risk |
| Enterprise | + ai-copilot, collaboration, paypal-marketplace, avalara, shippo, migration |

These map directly to PRD plan tiers (Vol 02 §6.1) and the JWT `features` array (Vol 01 §11.1).

---

## 4. Repository / Code Organisation

### 4.1 Monorepo
- Single Git repository: `agarha/mercato-suite`.
- Composer monorepo via `wikimedia/composer-merge-plugin` or `symplify/monorepo-builder` to keep per-module dependencies isolated.
- Lerna-like discipline: every module has its own `composer.json` listing intra-monorepo deps explicitly.

### 4.2 Versioning
- **Single suite version** (semver). v2.0.1 means every module ships at version `2.0.1`.
- A module's internal `module.json` records its semver of public surface (REST routes, hooks, events, classes); breaking changes obey the SemVer + deprecation policy in Vol 01 §4.5.
- Suite minor version bumps on additive changes; major bumps on breaking changes to any public surface.

### 4.3 CI build
- Branch protection + 2 approvals + CODEOWNERS per module.
- CI matrix: PHP 8.2 + PHP 8.3 × WP 6.4 + WP 6.5 × WC 8.0 + WC 8.1.
- For each PR, the suite tarball builds: zip of all modules with vendor/ prepopulated.
- Tagged releases publish:
  - `mercato-suite-v2.0.1.zip` (the WP-installable plugin)
  - `mercato-sdk-v2.0.1.tar.gz` (third-party plugin authors)
  - Signed cosign attestations for both
  - GitHub Release with changelog

---

## 5. Distribution

### 5.1 Where the suite is installed
- Tenant servers receive the suite via:
  - Automated install during tenant provisioning (Control Plane drops the suite into the WP container image).
  - For self-hosted Enterprise, signed download from a release CDN with download token.

### 5.2 Update path
- Suite updates roll via the same WP update mechanism but with a custom updater that:
  - Validates capability JWT.
  - Pulls signed package from Mercato's release server (not WP.org).
  - Verifies SHA-256 + cosign signature.
  - Runs migrations as Kubernetes Job (Vol 11 §7.5) before swapping the running pod.
- Rollback: previous suite tarball retained in `wp-content/upgrade/`; one command restores.

### 5.3 Not on WordPress.org
Mercato Suite is **not** distributed via WordPress.org. The plan-tier gating, license validation, and SaaS-controlled module activation are not compatible with WP.org's freemium rules. Suite ships via Mercato's own release infrastructure under proprietary license.

### 5.4 Third-party plugin authors
A separate `mercato-sdk` package (Composer + npm) is published for third-party developers building integrations on top of Mercato. The SDK is open-source (MIT) so third parties can build adapters; the **runtime** suite remains proprietary.

---

## 6. When Modules Can Be Disabled

A tenant admin (or operator) may disable a module via tenant settings (Vol 04 §22 enterprise). Rules:

| Module | Can be disabled by tenant? |
|---|---|
| `mercato-core` | NO — required |
| `mercato-vendors`, `mercato-products`, `mercato-orders`, `mercato-commissions`, `mercato-payouts` | NO — required for any marketplace |
| `mercato-stripe-connect`, `mercato-aws-s3`, `mercato-sendgrid` | NO at MVP — required adapters |
| `mercato-messaging` | YES (with banner: vendor communication off) |
| `mercato-reviews`, `mercato-disputes`, `mercato-reports`, `mercato-search`, `mercato-tax-engine`, `mercato-fraud-risk` | YES (per plan + tenant choice) |
| `mercato-ai-copilot`, `mercato-collaboration` | YES (per plan + tenant choice) |
| `mercato-subscriptions` | YES |
| Other integration adapters | YES |

Disabling a module fires `mercato.module.disabled.v1` and the module's routes/hooks immediately unload (no restart needed).

---

## 7. Forbidden Patterns

- ❌ Modules calling each other through direct PHP class references (must use DI or events).
- ❌ Modules adding their own WP plugin headers (would make them visible as separate plugins).
- ❌ Modules writing their own activation/deactivation hooks (lifecycle managed by suite).
- ❌ Modules running migrations directly (must go through `Mercato\Core\DB\Migrator`).
- ❌ Modules registering anything during file load (must use `register()` method called at boot).
- ❌ Modules versioned independently from suite.

CI lints enforce all of the above.

---

## 8. Migration Path (Phase 4)

The architecture allows splitting popular modules into separately-installable plugins later (e.g., if a third party wants `mercato-fraud-risk` without the rest of the suite). Path:
1. Extract module to its own repo.
2. Publish as a standalone WP plugin.
3. Replace internal module loading with WP plugin dependency check.
4. Keep the suite's bundled variant as a thin proxy that delegates to the separately-installed plugin if present.

This is **out of scope** until Phase 4 and only if business case emerges.

---

## 9. Operational Impact

| Concern | How packaging addresses |
|---|---|
| Plugin update fatigue | One update per tenant per release |
| Version mismatch bugs | Impossible — atomic suite version |
| Activation order | Internal — module dependency graph |
| Visibility/debugging | Module status in admin: enabled/disabled, version, last migration |
| Forking a single module | Hard; this is intentional — feature flags + tenant overrides preferred |

---

## 10. Cross-References

| Topic | See |
|---|---|
| Plugin topology | Vol 01 §4 |
| Manifest schema | Vol 04 §4.2 |
| SDK semver policy | Vol 01 §4.5 |
| Tenant feature flag JWT | Vol 01 §11.1 |
| CI/CD | Vol 11 §7 |
| Activation in MVP | Vol 00 §2 |

---

*End of Volume 14 — Plugin Packaging Strategy v1.0.*
