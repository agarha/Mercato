# ADR-004: RDBMS — Aurora MySQL vs. Aurora PostgreSQL

- **Status:** Accepted
- **Date:** 2026-05-21
- **Deciders:** CTO, Principal DBA, Principal Software Architect
- **Tags:** database, infrastructure

## Context

WordPress and WooCommerce are tightly coupled to MySQL — `wp_posts`, `wp_options`, WooCommerce HPOS tables (`wp_wc_orders`, `wp_wc_orders_meta`, `wp_wc_order_addresses`, `wp_wc_order_operational_data`, `wp_wc_product_meta_lookup`) are MySQL-flavoured. Mercato's `wp_mercato_*` custom tables live alongside.

The question: do we accept the MySQL flavor as authoritative, or run PostgreSQL alongside (with MySQL only for WP/WC and PostgreSQL for custom tables / Control Plane)?

## Options

### Option A — Aurora MySQL 8.0 for everything tenant-data
- Co-located with WP/WC; transactions can span Mercato + WC tables.
- Read replicas for analytics queries.
- HPOS-friendly.
- Limited support for true row-level security policies (vs PostgreSQL RLS).
- JSON column type adequate but ergonomically weaker than PostgreSQL JSONB.

### Option B — Mixed: MySQL for WP/WC, PostgreSQL for Mercato custom tables
- PostgreSQL RLS would simplify pooled-mode tenant isolation.
- JSONB more powerful for tenant_settings document.
- Cross-DB transactions impossible — outbox pattern would need a different design.
- Operational complexity doubled (two RDBMS to operate, backup, patch).
- DBA skill set divides.

### Option C — Aurora PostgreSQL for Control Plane only; MySQL for tenant data
- Control Plane (tenant management, licensing, AI usage rollup) uses PostgreSQL.
- Tenant data plane stays on MySQL.

## Decision

**Adopt Option C.** Aurora MySQL 8.0 for the tenant data plane (where it coexists with WP/WC). Aurora PostgreSQL for the Control Plane (where we have no WP dependency and can use RLS + JSONB + pgvector freely).

## Consequences

**Positive:**
- WP/WC compatibility preserved without compromise.
- Outbox transaction guarantees intact (single DB).
- Control Plane gets PostgreSQL's strengths for tenant management and AI corpora.

**Negative:**
- DBA team must operate two engines.
- Backup/restore runbooks duplicated.
- Tools like dbt require both connectors.

**Follow-up:**
- Document operational procedures for both engines in Vol 11.
- Pin Aurora MySQL 8.0 LTS minor; PostgreSQL 15 LTS.
- Revisit at Phase 4 if WP HPOS adds PostgreSQL support.

## References
- Vol 01 §6.1
- Vol 06 (entire)
- Vol 11 §6.1
