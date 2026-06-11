# Lead Time Matrix System – Developer Overview & Quick Reference

**Document Purpose**: High-level guide for developers implementing the Lead Time Matrix feature in Moyo. Study this before diving into code.

**Last Updated**: 2026  
**Project**: Moyo Lead Time Matrix  
**Status**: Technical Design Complete

---

## Table of Contents

1. [Core Business Mission](#core-business-mission)
2. [The Calculation Engine](#the-calculation-engine)
3. [Calendar Logic (Weekends & Holidays)](#calendar-logic-weekends--holidays)
4. [Dual Environment: Blue (Staging) → Green (Live)](#dual-environment-blue-staging--green-live)
5. [Data Model Structure](#data-model-structure)
6. [API Contracts (GraphQL)](#api-contracts-graphql)
7. [Special Features](#special-features)
8. [Synchronization Direction](#synchronization-direction)
9. [Non-Functional Requirements](#non-functional-requirements)
10. [Deployment Strategy](#deployment-strategy)
11. [Key Implementation Priorities](#key-implementation-priorities)
12. [Common Pitfalls to Avoid](#common-pitfalls-to-avoid)

---

## Core Business Mission

You are building **Moyo's Lead Time Matrix system** — a centralized, rule-driven engine that will become Amrod's **single source of truth** for all lead time calculations.

### What This Replaces
- Legacy Amtrack calculations
- Distributed, inconsistent lead time rules
- Manual spreadsheet-based configurations

### What This Enables
- **Consistent timelines** across Amtrack, customer websites, and internal stock calculators
- **Real-time calculations** via high-performance GraphQL API
- **Administrative control** of lead times through hierarchical rules and overrides
- **Staged deployments** via Blue/Green dual environment

### Systems Impacted
1. **Moyo** - Storage of rules, admin UI, core calculation service, API endpoints
2. **Amtrack** (Legacy ERP) - Modified to call Moyo APIs instead of internal calculations
3. **Pimcore** (PIM System) - Product metadata source (validates invoice/setup codes)
4. **Amrod Customer Website** - Consumes Moyo public API for lead-time previews

---

## The Calculation Engine

### Formula
```
Total Lead Time = Base Lead Time + Sum(Add-On Lead Times)
```

### Precedence Hierarchy (Most Important First)

1. **Override** (if exists, use exclusively)
   - Bypasses all other calculations
   - Highest precedence

2. **Adjustment** (if no override exists)
   - Added to base lead time
   - Medium precedence

3. **Add-Ons** (Console, CMT, Personalisation, etc.)
   - Stacked on top of base
   - Lowest precedence

### Example Calculation Path
```
Base Lead Time = 5 days (from matrix)
Adjustment = +2 days (department adjustment)
Add-Ons = Console: +1 day, Personalisation: +1 day
---
Total = 5 + 2 + 1 + 1 = 9 days
```

### Configuration Hierarchy for Branded Orders

```
BrandingDepartment
  ├─ AdjustmentDays (department-level)
  ├─ OverrideDays (department-level override)
  └─ BrandingGroup (grouping within dept)
	   ├─ AdjustmentDays (group-level)
	   ├─ OverrideDays (group-level override)
	   └─ QuantityBreak (volume-based lead times)
			├─ LowerLimit
			├─ UpperLimit
			└─ LeadTimeDays (base)
```

### Order Type Classification

#### Scenario 1: Single Jobcard Order
```
Internal Order Lead Time = Jobcard Total Internal Lead Time
Client Order Lead Time = Jobcard Total Client Lead Time
```

#### Scenario 2: Multi-Jobcard Order (Same Branding Department)
```
Internal Order Lead Time = MAX(Jobcard Total Internal Lead Time)
Client Order Lead Time = MAX(Jobcard Total Client Lead Time)
```

#### Scenario 3: Multi-Jobcard Order (Different Branding Departments)
```
Internal Order Lead Time = MAX(Jobcard Total Internal Lead Time) + 1 DAY (merge penalty)
Client Order Lead Time = MAX(Jobcard Total Client Lead Time) + 1 DAY (merge penalty)
```

#### Scenario 4: Unbranded Order (Warehouse Stock Only)
```
- No jobcards involved
- Lookup directly from WarehouseQuantityBreak matrix
- Apply warehouse group hierarchy
```

### Special Case: Logo24 Service (Tight SLA)

Logo24 has **special cutoff rules** to meet quick-turnaround SLA:

| Condition | Result |
|-----------|--------|
| Mon-Thu, approved & paid **before 17:00** | Ready JHB **next business day by 17:00** |
| Mon-Thu, **after 17:00** | Add +1 business day |
| Fri, **before 12:00** | Ready **Monday 17:00** (if trading day) |
| Fri, **after 12:00** | Reverts to standard lead times |
| Weekend or Public Holiday | Reverts to standard lead times |

---

## Calendar Logic (Weekends & Holidays)

### The Process

1. **Calculate lead time days** using the engine (section above)
2. **Apply to current/order date** to get raw due date
3. **Exclude weekends** (Saturday, Sunday)
4. **Exclude public holidays** (South Africa schedule)
5. **Check SpecialDate table** for other excluded dates
6. **Adjust forward** to next valid working day if needed

### Pseudo-Code
```
calculatedDate = orderDate + leadTimeDays

while (calculatedDate is weekend OR 
	   calculatedDate is in SpecialDate table OR
	   calculatedDate is public holiday):
	calculatedDate = calculatedDate + 1 day

return calculatedDate
```

### Public Holiday Automation

**Monthly Flowgear Pipeline** (runs 1st of month):

1. Call [OpenHolidays API](https://www.openholidaysapi.com/) for South Africa
2. Fetch holidays for current year + next year
3. Parse response
4. For each holiday:
   - Check if date already exists in `SpecialDate` table
   - If **NO**: Insert with category = "Public Holiday"
   - If **YES**: Skip (no duplicates, no overwrites)

**API Call Example**:
```
GET https://openholidaysapi.org/PublicHolidays?countryIsoCode=ZA&validFrom=2026-01-01&validTo=2027-12-31
```

### Important Rules

- ✅ **Immutability**: Historic dates (before today) are **locked** — cannot modify
- ✅ **No Duplicates**: Overlapping dates are **ignored** on conflict
- ✅ **Branch Delivery Exception**:
  - If **Lead Time Days Mode**: Add transit days additively
  - If **Specific Delivery Days Mode**: Compute production date, then adjust forward to next scheduled dispatch day

---

## Dual Environment: Blue (Staging) → Green (Live)

### Why Dual Environments?

You need to stage matrix changes **overnight** and apply them **at midnight** to avoid disrupting orders in flight.

### Blue vs. Green

| Aspect | **Blue (Staging)** | **Green (Live)** |
|--------|-------------------|-----------------|
| **Purpose** | Hold pending changes | Current production |
| **Effective From** | Midnight (next day) | Immediately |
| **Who Sees It** | Internal API only (with `useStaged: true`) | External + Internal APIs |
| **Update Timing** | Staged in update tables | Applied to live tables |
| **Warehouse Updates** | ⚠️ Exception: Real-time | ✅ Real-time (not staged) |
| **Production Updates** | All updates staged | New inserts immediate; updates staged |
| **Synced By** | Flowgear (nightly 02:00) | N/A (already live) |

### How It Works

#### Admin makes change to lead time matrix at 15:00
```
BrandingDepartmentUpdate (Blue table) ← Record status = "Pending"
  ├─ DepartmentId: 5
  ├─ AdjustmentDays: 7 (changed from 5)
  └─ Status: "Pending"
```

#### Internal API can preview impact
```graphql
query productionLeadTime(
  $useStaged: true,        # Use Blue tables
  $includeBreakdown: true
) {
  # Returns calculation using staged (Blue) lead times
}
```

#### Flowgear sync runs at 02:00 (midnight)
```
BrandingDepartmentUpdate (Blue) → BrandingDepartment (Green)
Update status: "Synced"
```

#### Next morning: Calculations use new value
```graphql
query productionLeadTime(
  $useStaged: false        # Use Green tables (default)
) {
  # Returns calculation using live (Green) lead times
  # AdjustmentDays now 7
}
```

### Critical Lock Rule

⚠️ **Once an order is APPROVED and PAID, its timeline is LOCKED**

Matrix updates **must NOT** alter timelines for:
- ✅ Already-paid orders
- ✅ Already-approved orders
- ✅ Already-scheduled orders

Implementation strategy:
1. Query `Order` table for status = "Paid" or "Approved"
2. Store their **calculated SLA dates** separately (in an `OrderLockHistory` table or similar)
3. When matrix changes, skip locked orders in recalculation

---

## Data Model Structure

### Live Tables (Green)

#### Core Branding Hierarchy
```sql
BrandingDepartment
  ├─ DepartmentId (PK)
  ├─ Name
  ├─ AdjustmentDays
  ├─ AttributeName
  └─ OverrideDays

BrandingGroup
  ├─ BrandingGroupId (PK)
  ├─ DepartmentId (FK)
  ├─ Name
  ├─ Description
  ├─ AdjustmentDays
  └─ OverrideDays

QuantityBreak
  ├─ QuantityBreakId (PK)
  ├─ GroupId (FK to BrandingGroup)
  ├─ LowerLimit
  ├─ UpperLimit
  └─ LeadTimeDays

SetupCode & InvoiceCode
  ├─ SetupCodeId / InvoiceCodeId (PK)
  ├─ PrintCodeId (FK)
  ├─ Code / Name
  ├─ Description
  ├─ AdjustmentDays
  └─ OverrideDays
```

#### Warehouse Hierarchy
```sql
OrderType
  ├─ OrderTypeId (PK)
  ├─ Name
  ├─ AdjustmentDays
  └─ OverrideDays

WarehouseGroup
  ├─ WarehouseGroupId (PK)
  ├─ OrderTypeId (FK)
  ├─ Name
  ├─ AdjustmentDays
  └─ OverrideDays

WarehouseQuantityBreak
  ├─ WarehouseQbId (PK)
  ├─ WarehouseGroupId (FK)
  ├─ LowerLimit
  ├─ UpperLimit
  ├─ LeadTimeDays
  └─ DepartmentId (FK)
```

#### Special Dates
```sql
SpecialDateCategory
  ├─ CategoryId (PK)
  └─ Name (e.g., "Public Holiday", "Shutdown")

SpecialDate
  ├─ SpecialDateId (PK)
  ├─ Date (UNIQUE)
  ├─ Name
  └─ CategoryId (FK)

SpecialDateImpact
  ├─ ImpactId (PK)
  ├─ SpecialDateId (FK)
  └─ DepartmentId (FK) — which depts affected
```

#### Add-Ons
```sql
PersonalisationRule
  ├─ PersonalisationId (PK)
  ├─ DepartmentId (FK)
  ├─ LowerLimit
  ├─ UpperLimit
  └─ LeadTimeDays

-- CMT, Console rules follow similar pattern
```

### Update Tables (Blue)

Parallel structure to live tables + `Status` field:

```sql
BrandingDepartmentUpdate
  ├─ RecordId (PK)
  ├─ DepartmentId
  ├─ Name
  ├─ AdjustmentDays
  ├─ AttributeName
  ├─ OverrideDays
  └─ Status ('Pending', 'Approved', 'Synced')

QuantityBreakUpdate
  ├─ RecordId (PK)
  ├─ QuantityBreakId
  ├─ GroupId (FK)
  ├─ LowerLimit
  ├─ UpperLimit
  ├─ LeadTimeDays
  └─ Status

-- Similar for all other entities
```

### Relationships & Constraints

All tables have proper **foreign key relationships** maintaining referential integrity:

```
PrintCode --FK--> BrandingDepartment
SetupCode --FK--> PrintCode
InvoiceCode --FK--> PrintCode
BrandingGroup --FK--> BrandingDepartment
QuantityBreak --FK--> BrandingGroup
SpecialDateImpact --FK--> (SpecialDate, BrandingDepartment)
WarehouseGroup --FK--> OrderType
WarehouseQuantityBreak --FK--> WarehouseGroup
PersonalisationRule --FK--> BrandingDepartment
```

---

## API Contracts (GraphQL)

### Single Endpoint Principle

There is **ONE calculation endpoint** with **TWO instances**:
1. **Internal Instance** (authenticated, ops/admin use)
2. **External Instance** (public/partner-facing, safe subset)

### Internal Query (Authenticated)

**Purpose**: Full details for operational staff and system debugging

```graphql
query productionLeadTime(
  $orderId: ID,
  $input: ProductionLeadTimeInput,
  $useStaged: Boolean = false,
  $includeBreakdown: Boolean = false
) {
  productionLeadTime(
	orderId: $orderId,
	input: $input,
	useStaged: $useStaged
  ) {
	slaDate               # ISO-8601 string
	leadTime              # Integer (days)
	mode                  # "live" | "staged"
	breakdown @include(if: $includeBreakdown) {
	  baseDays            # Matrix lead time
	  addOns {            # All add-on components
		code              # "Console", "CMT", etc.
		days              # Lead time contribution
	  }
	  adjustmentDays      # From adjustment rules
	  overrideApplied     # Boolean
	  calendarShifts {    # Weekend/holiday skips
		date              # Shifted date
		reason            # "Weekend", "PublicHoliday", etc.
	  }
	  finalComputationPath # Debug info: calculation trace
	}
  }
}
```

**Flags Explained**:
- `useStaged: boolean` - If `true`, use Blue (staged) lead times; if `false`, use Green (live). Default: `false`
- `includeBreakdown: boolean` - If `true`, return detailed component breakdown. Default: `false`

**Inputs** (either/or):
- `orderId: ID` — Fetch existing order context from database
- `input: ProductionLeadTimeInput` — Provide calculation details directly
  - `orderType: String` (e.g., "Branded", "Unbranded")
  - `brandingDepartments: [Int]` (department IDs)
  - `approvalPaidTimestamp: DateTime` (for Logo24 cutoff logic)
  - `branchDeliveryMode: String` (e.g., "LeadTimeDays", "SpecificDays")
  - `quantity: Int`
  - etc.

### External Query (Public)

**Purpose**: Safe, limited response for customer-facing applications

```graphql
query productionLeadTime(
  $orderId: ID,
  $input: ProductionLeadTimeInput
) {
  productionLeadTime(
	orderId: $orderId,
	input: $input
  ) {
	slaDate               # ISO-8601 string
	leadTime              # Integer (days)
	mode                  # "live" (always; never "staged")
  }
}
```

**Differences from Internal**:
- ❌ NO `useStaged` flag (always uses live Green tables)
- ❌ NO `includeBreakdown` (no component details)
- ❌ NO `finalComputationPath` (no debug info)

### Response Examples

#### Internal with Breakdown
```json
{
  "slaDate": "2026-02-15",
  "leadTime": 9,
  "mode": "live",
  "breakdown": {
	"baseDays": 5,
	"addOns": [
	  { "code": "Console", "days": 1 },
	  { "code": "Personalisation", "days": 1 }
	],
	"adjustmentDays": 2,
	"overrideApplied": false,
	"calendarShifts": [
	  {
		"date": "2026-02-14",
		"reason": "Weekend (Saturday)"
	  }
	],
	"finalComputationPath": "BrandingGroup 'Premium' → QuantityBreak [100-500] → 5 days + adjustments"
  }
}
```

#### External (Simple)
```json
{
  "slaDate": "2026-02-15",
  "leadTime": 9,
  "mode": "live"
}
```

### Performance SLA

| Metric | Target |
|--------|--------|
| Response Time (99th percentile) | **<250ms** |
| Throughput | **10 requests/second (RPS)** |
| Availability | 99.9% |

### Caching Strategy

- **Cache Inputs**: Calculation inputs (matrices, adjustments, overrides)
- **TTL**: 24 hours (or until next scheduled matrix sync)
- **Cache Invalidation**: **Immediate clear** when matrix changes are approved
- **Flowgear Sync Window**: 02:00-03:00 (low-traffic window for Blue→Green sync)

---

## Special Features

### Branch Delivery Exception

Orders can have branch deliveries (e.g., Durban, Nelspruit). Two modes:

#### Mode 1: Lead Time Days (Additive)
```
Example: Durban branch = 2 transit days

Production due date = 2026-02-10
+ Durban transit = 2 days
---
Final delivery = 2026-02-12
```

#### Mode 2: Specific Delivery Days (Scheduled)
```
Example: Nelspruit branch delivers only Tuesdays & Thursdays

Production due date = 2026-02-10 (Saturday)
No Tuesday/Thursday → Adjust forward
Next valid dispatch = 2026-02-11 (Tuesday)
---
Final delivery = 2026-02-11
```

**Implementation Note**: Calculate production date first, then adjust for branch delivery constraints.

### Console Lead Time Exception

When an order includes **Console (printing)** service:

```
If Console present:
  Client Lead Time = Base + Add-Ons + 1 DAY

If Console absent:
  Client Lead Time = Base + Add-Ons
```

---

## Synchronization Direction

### Unidirectional: Moyo → Amtrack ONLY

```
┌─────────────────────────────────────┐
│  Moyo (Authoritative Source)        │
│  - Lead time configurations         │
│  - Calculation engine               │
│  - Matrix updates                   │
└──────────┬──────────────────────────┘
		   │
		   ↓ SYNC (Nightly 02:00)
	 [Production Lead Times]
		   ↓
┌──────────────────────────────────────┐
│  Amtrack (Read-Only Consumer)        │
│  - Must call Moyo API for calcs      │
│  - No internal lead time math        │
│  - No reverse sync                   │
└──────────────────────────────────────┘
```

### Why One-Way?

If Amtrack changes lead times locally:
- ❌ Data drifts between systems
- ❌ Calculations become inconsistent
- ❌ No single source of truth

### Amtrack Migration

Amtrack **must be updated** to:

1. **Remove** internal lead time calculation logic
2. **Call** Moyo Gateway API for:
   - Production lead times
   - Warehouse lead times
   - SLA date calculations
3. **Cache** results locally for performance
4. **Implement circuit breaker**:
   - If Moyo API >1,500ms latency → fail to local read-only replica
   - If Moyo API fails 3× consecutively → fail to local read-only replica

---

## Non-Functional Requirements

### Performance

| Requirement | Target |
|---|---|
| API Response Time (99th percentile) | **<250ms** |
| Throughput | **10 RPS** (high-season peak) |
| Connection Pool | Sized for concurrent demand |
| Cache Hit Ratio | >85% (via caching strategy) |

**Optimization Tips**:
- Use database indexes on `QuantityBreak.LowerLimit`, `QuantityBreak.UpperLimit` for fast lookups
- Cache calculation matrices in memory
- Use query optimization for multi-table joins
- Batch holiday inserts (Flowgear) to avoid individual row locks

### Observability (Structured Logging)

Every administrative change must log:

```json
{
  "timestamp": "2026-02-15T14:30:45Z",
  "userId": "admin-user-123",
  "action": "UpdateLeadTime",
  "entityId": "BrandingDepartment:5",
  "entityType": "BrandingDepartment",
  "changeDelta": {
	"fieldName": "AdjustmentDays",
	"valueBefore": 5,
	"valueAfter": 7
  },
  "affectedRecords": 142,
  "changeSource": "AdminUI"
}
```

**Log Destination**: Centralized monitoring stack (Signoz)

**Query Examples**:
- Find all changes by user X in last 7 days
- Identify which admin changed lead time for department Y
- Audit trail for compliance/dispute resolution

### Security

- **Permission-Driven Admin Functions**:
  - Only authorized users can edit dates, adjustments, overrides
  - Per BRD requirements (e.g., Manager can override, Supervisor cannot)
- **API Authentication**:
  - Internal queries require valid token
  - External queries may have rate limiting
- **Audit Trail**:
  - Every change logged with user ID

### Failover & Availability

**Circuit Breaker Pattern** (Amtrack → Moyo):

```
Normal: Amtrack → Moyo API ✅
		 Response: <250ms ✅

Degraded: 
  If Moyo API >1,500ms (3 checks)
  OR Moyo API fails 3× consecutively
  ---
  Circuit OPENS → Amtrack fails back to local read-only replica

Recovery:
  Half-open state → Try Moyo API again after cooldown
  If OK → Circuit CLOSES → resume normal routing
  If FAIL → Circuit remains OPEN → retry after cooldown
```

**Implementation**: Use Polly library (for .NET) or similar circuit breaker library

---

## Deployment Strategy

### Phase 1: Data Migration

1. **ETL Run**:
   - Extract legacy lead time data from Amtrack
   - Transform into Moyo schema
   - Load into Green (live) tables
   - Validate record counts match

2. **Validate Data Integrity**:
   - Check foreign key constraints
   - Verify no orphaned records
   - Confirm all departments/groups present

### Phase 2: Parity Run (5 Business Days)

**Goal**: Ensure Moyo calculations match legacy Amtrack calculations for real orders

1. Enable **shadow mode** in Amtrack:
   - Amtrack makes normal order (using legacy calcs)
   - Simultaneously calls Moyo API with same inputs
   - Logs both results for comparison

2. **Collect Results**:
   - Run for 5 full business days
   - Capture 100% of order lead time calculations
   - Flag any mismatches

3. **Investigate Mismatches**:
   - Root cause analysis
   - Fix data in Moyo or recalibrate logic
   - Rerun parity until match rate **>99.5%**

### Phase 3: Cutover

Once parity confirmed:

1. **Feature Flag**: Enable Moyo as primary source
2. **Monitor**: Watch error rates and latency closely
3. **Gradual Rollout**: If using feature flags, increase % of traffic gradually (e.g., 10% → 50% → 100%)

### Phase 4: Rollback Criteria

If during sanity testing:
- ❌ Error rate **>0.5%**, OR
- ❌ Latency **>300ms** (99th percentile)

**Action**: Flip feature flag back to Amtrack (read-only mode)

**During Rollback**:
- Lock Moyo configuration tables (read-only)
- Prevent further changes
- Investigate root cause
- Plan fixes

---

## Key Implementation Priorities

### Phase 1: Foundation (Weeks 1-3)

1. ✅ **Calculation Engine**
   - Implement Base + Add-On hierarchy
   - Implement precedence rules (Override > Adjustment > Add-On)
   - Unit test all calculation paths

2. ✅ **Calendar Logic**
   - Working-day exclusion (weekends, holidays)
   - Special date handling (immutability, no duplicates)
   - Unit test date shifting

3. ✅ **Data Model**
   - Create all Green (live) tables with FKs
   - Create all Blue (staging) tables with Status field
   - Database migrations (EF Core DbContext or SQL scripts)

### Phase 2: API & Integration (Weeks 4-6)

4. ✅ **GraphQL API Endpoint**
   - Single calculation endpoint
   - Internal instance with full features
   - External instance with limited response
   - Input validation

5. ✅ **Blue/Green Logic**
   - Implement `useStaged` flag
   - Query logic to select Green vs. Blue tables
   - Flowgear integration (nightly Blue→Green sync)

### Phase 3: Observability & Safety (Weeks 7-9)

6. ✅ **Structured Logging**
   - Audit trail for every matrix change
   - Log to Signoz
   - Queryable by user, entity, timestamp

7. ✅ **Order Lock Protection**
   - Prevent timeline changes for paid/approved orders
   - Implement lock storage mechanism
   - Unit test lock enforcement

### Phase 4: Performance & Hardening (Weeks 10-12)

8. ✅ **Caching**
   - Implement caching for calculation inputs
   - Cache invalidation on matrix changes
   - Monitor cache hit ratio

9. ✅ **Circuit Breaker** (for Amtrack integration)
   - Implement Polly circuit breaker
   - Timeout handling (>1,500ms)
   - Failure retry logic

10. ✅ **Performance Testing**
	- Load test to 10 RPS
	- Measure 99th percentile latency
	- Optimize slow queries

### Phase 5: Deployment & Safety (Weeks 13+)

11. ✅ **Data Migration**
	- ETL from Amtrack
	- Data validation

12. ✅ **Parity Run**
	- 5 business days shadow mode
	- Mismatch investigation
	- Cutover readiness

---

## Common Pitfalls to Avoid

### 🚨 Calculation Engine

| Pitfall | Impact | Solution |
|---------|--------|----------|
| Forget calendar shift **after** calculating days | Orders ship on wrong date (customer SLA miss) | Always apply calendar logic AFTER base calculation |
| Mix up "Max" vs "Sum" for multi-jobcard orders | Over/underestimate lead times | Review scenarios 1-3 carefully; test each branch |
| Forget merge penalty (+1 day for diff. depts) | Short lead times for complex orders | Check department diversity before selecting max |
| Miss Logo24 cutoff logic (17:00 Thu, 12:00 Fri) | Wrong SLA for fast-track orders | Explicit Logo24 handler; test with sample dates |

### 🚨 Calendar & Special Dates

| Pitfall | Impact | Solution |
|---------|--------|----------|
| Forget to exclude weekends | Orders due on Saturday/Sunday | Always check day-of-week after shifting |
| Allow modification of historic dates | Audit trail broken; past orders affected | Lock all dates before today (immutability rule) |
| Create duplicate holiday entries | Inconsistent calendar; confusion in logic | Check uniqueness before inserting SpecialDate |
| Forget branch delivery logic | Branch orders ship on wrong date | Separate production due date from final delivery date |

### 🚨 Blue/Green Staging

| Pitfall | Impact | Solution |
|---------|--------|----------|
| Updates go live immediately (skip Blue) | Approved orders get wrong timelines | Always stage in Blue; Flowgear syncs at 02:00 |
| External API calls Blue tables | Customers see future lead times | External API **only** Green; internal has flag |
| Modify paid order timelines | Order fulfillment chaos; customer complaints | Query Order status before updating timelines |
| Forget order lock enforcement | Locked orders get new (wrong) SLA | Store OrderSLALock table; check before sync |

### 🚨 Data Integrity

| Pitfall | Impact | Solution |
|---------|--------|----------|
| Missing foreign keys | Data corruption; orphaned records | Add explicit FK constraints to all schemas |
| NULL values in required fields | Calculations fail; API errors | Mark NOT NULL on QuantityBreak.LeadTimeDays, etc. |
| No indexing on lookup columns | Slow queries; timeout errors | Index QuantityBreak (GroupId, LowerLimit, UpperLimit) |

### 🚨 API & Performance

| Pitfall | Impact | Solution |
|---------|--------|----------|
| No caching of matrices | Repeated DB queries; miss 250ms SLA | Cache matrices; invalidate only on admin changes |
| Missing circuit breaker (Amtrack) | Cascading failures; Amtrack hangs | Implement Polly with 1.5s timeout, 3-fail threshold |
| No structured logging | Can't audit who changed what | Log every matrix change with UserId, timestamp, delta |
| Missing input validation | SQL injection; malformed responses | Validate orderId, quantity ranges, date formats |

### 🚨 Deployment

| Pitfall | Impact | Solution |
|---------|--------|----------|
| Skip parity run | Live calculation mismatches revealed in prod | Run 5 business days shadow comparison; fix > 99.5% |
| No rollback plan | Can't recover if latency spikes | Monitor error rate & latency; threshold 0.5% / 300ms |
| Hard-coded dependencies on Amtrack | Amtrack down → entire system down | Use feature flags; support circuit-breaker fallback |

---

## Implementation Checklist

- [ ] Read and understand this document
- [ ] Review TechnicalDesign files (01-13) in detail
- [ ] Design class hierarchy for calculation engine
- [ ] Design calendar/holiday logic
- [ ] Create EF Core DbContext for data model
- [ ] Implement Blue/Green table queries
- [ ] Implement GraphQL endpoint (internal + external)
- [ ] Implement order lock protection
- [ ] Implement structured logging
- [ ] Implement caching layer
- [ ] Implement circuit breaker (Amtrack integration)
- [ ] Unit test all calculation paths
- [ ] Load test to 10 RPS target
- [ ] Run parity test with legacy Amtrack
- [ ] Plan deployment & rollback
- [ ] Train support team on admin UI

---

## Quick Reference: File Locations

All TechnicalDesign documents are located at:
```
..\..\Users\Stephan\Desktop\LeadTimeMatrix\TechnicalDesign\
```

| Document | Key Content |
|----------|---|
| `02-executive-summary.md` | Business context, systems impacted |
| `04-calculation-model-and-engine.md` | Full calculation flows & logic |
| `05-special-dates-and-working-day-logic.md` | Holiday automation, immutability |
| `06-sync-behaviour.md` | Blue/Green staging & order locks |
| `07-proposed-architecture-artefacts.md` | Data model, API contracts, Flowgear |
| `11-non-functional-requirements.md` | Performance, observability, security |
| `12-deployment-and-rollback-strategy.md` | Migration, parity run, rollback |

---

## Questions to Answer Before You Code

1. **Calculation**: Can you explain the precedence order (Override > Adjustment > Add-On) with an example?
2. **Calendar**: How would you handle an order due on Friday but next working day is Tuesday (Monday is holiday)?
3. **Blue/Green**: Why can't external API see staged lead times?
4. **Lock Protection**: What happens if admin changes lead time 1 hour after an order is paid?
5. **Performance**: How would you cache 1M+ matrix combinations?
6. **Amtrack**: Why must Amtrack stop doing internal lead time calculations?

**If you can't answer all 6, review the relevant sections above.**

---

**Document Version**: 1.0  
**Last Reviewed**: 2026  
**Next Review**: Upon TechnicalDesign update  
**Owner**: Development Team
