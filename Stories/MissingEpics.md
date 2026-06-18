# Missing Jira Epics - Unparented Stories

This document lists the developer stories from [StoriesOverview.md](StoriesOverview.md) whose `Parent` cell is currently blank because no matching Jira epic exists in the current epic list:

| Epic | Title |
|------|-------|
| MOYO-4270 | Lead Time Matrix: Lead Time Calculation Engine - Order |
| MOYO-4128 | Lead Time Matrix: Exceptions |
| MOYO-4127 | Lead Time Matrix: Due Date Calculation & Other Rules |
| MOYO-4126 | Lead Time Matrix: Lead Time Calculation Engine - Jobcard |
| MOYO-4125 | Lead Time Matrix: Warehouse Lead Time Matrix |
| MOYO-4124 | Lead Time Matrix: Production Lead Time Matrix |
| MOYO-4123 | Lead Time Matrix: Branding Departments Manager |
| MOYO-4122 | Lead Time Matrix: Special Dates Manager |
| MOYO-3993 | Lead Time Matrix - Design & Planning |

The 10 stories below are grouped by the new epic recommended to host them.

---

## 1. Permissions & Access Control *(missing epic)*

| BRD No | Story Summary |
|--------|---------------|
| 4.1, 4.2, 4.3, 4.4, 4.5 | Lead Time Matrix - I want role-based permissions to control access to module features |

Covers BRD §4 (40 permissions across all five managers). Distinct enough that bolting it onto any one feature epic would understate its scope.

---

## 2. API Infrastructure (GraphQL + Versioning) *(missing epic)*

| BRD No | Story Summary |
|--------|---------------|
| TD/07 | Lead Time Matrix - GraphQL Endpoint Setup (Internal & External instances) |
| TD/13 | Lead Time Matrix - API versioning at /api/v1/ |

Cross-cutting API surface (TechnicalDesign §07 API Contracts + §13 Principles Compliance). Used by both Jobcard and Order calculation engines, so it doesn't belong under either MOYO-4126 or MOYO-4270.

---

## 3. Amtrack Integration & Decommission *(missing epic)*

| BRD No | Story Summary |
|--------|---------------|
| TD/08, TD/09, TD/11 | Lead Time Matrix - Amtrack Integration Changes (delegate to Moyo + circuit breaker) |

TechnicalDesign §08 (Dual-Run Synchronisation) + §09 (Risk & Impact / Decommission Plan) + §11 (Failover/Availability Circuit-Breaker). Significant cross-system work; not really feature scope.

---

## 4. Non-Functional Requirements (Observability, Caching, Performance) *(missing epic)*

| BRD No | Story Summary |
|--------|---------------|
| TD/11 | Lead Time Matrix - Calculation Input Caching with daily invalidation |
| TD/11 | Lead Time Matrix - Observability & Admin Audit Logs forwarded to Signoz |
| TD/11 | Lead Time Matrix - Performance testing & SLA validation harness |

All three are TechnicalDesign §11 NFRs (Performance / Observability / Failover). They span every feature epic and should not be hidden inside one.

---

## 5. Data Migration, Parity & Rollback (Cutover) *(missing epic)*

| BRD No | Story Summary |
|--------|---------------|
| TD/12 | Lead Time Matrix - Data Migration ETL from Amtrack legacy schema |
| TD/12 | Lead Time Matrix - Parity Run shadow-calculation comparison |
| TD/12 | Lead Time Matrix - Rollback feature toggle to revert to Amtrack legacy |

TechnicalDesign §12 (Deployment & Rollback Strategy). MOYO-3993 *Design & Planning* is upstream of these; cutover work warrants its own delivery epic.

---

## Recommendation

Create the five epics above in Jira (e.g. `MOYO-4xxx`), then update the `Parent` column for the 10 blank rows in [StoriesOverview.md](StoriesOverview.md) once the codes are issued. The other 59 rows can be imported as-is.
