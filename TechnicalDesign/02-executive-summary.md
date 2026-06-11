# Executive Summary

## Business Objective

Amrod's lead times are currently computed through legacy calculations and distributed rules. The primary goal is to establish **Moyo** as the single, authoritative, rule-driven system of record for all lead time configurations and computations. This solution governs both internal production schedules (jobcards) and external sales orders, ensuring consistent timelines across Amtrack, customer-facing websites, and internal stock calculators.

---

## Solution Overview

- **Hierarchical Configuration Engine**: Admin interfaces in Moyo for managing public holidays/shutdowns, branding departments, invoice/setup codes, and matrix thresholds.
- **Deterministic Lead Time Engine**: A rule-driven pipeline in Moyo executing a "Base + Additive" model. The engine computes both Internal and Client-facing timelines, applying exact holiday/weekend exclusion calendars.
- **API-First Architecture**: High-performance REST endpoints exposed by Moyo to serve real-time calculations to Amtrack (for sales ordering and production reporting), the external customer portal (web branding calculator), and internal Moyo stock-checking modules.

---

## Systems Impacted

1. **Moyo (Core Application)**: Storage of rules, administrative UI, core calculation service, and API endpoints.
2. **Amtrack (Legacy ERP / Core Operations)**: Modified to delegate all ordering, pricing, and scheduling due-date calculations to Moyo APIs.
3. **Pimcore (PIM System)**: Source of truth for product metadata and billing validation (validating invoice/setup codes).
4. **Amrod Customer Website**: Consumes Moyo's public API for lead-time previews.

---

## Design Complexity

High Complexity: Due to legacy dual-run requirements between Amtrack and Moyo, sub-hour time-unit calculations, strict database indexing for real-time aggregation queries, and transaction locking rules for approved/paid orders.

