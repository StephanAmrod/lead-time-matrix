# Feedback & Clarification for Lead Time Matrix Technical Design

**Prepared By**: Development Team  
**For**: Software Architect  
**Date**: 2026  
**Purpose**: Highlight unclear sections, potential gaps, and critical clarification questions before implementation

---

## Executive Summary

The Technical Design is **comprehensive and well-structured overall**. However, there are **critical ambiguities** in:
1. Order lock enforcement mechanism (how we track & identify locked orders)
2. Calculation input specification (what fields map to which rules?)
3. Flowgear error handling & retry logic
4. Multi-order scenarios and jobcard merging logic
5. Ad-hoc calculation vs. persisted order calculations

**Recommendation**: Resolve clarifications in this document before development begins (estimated 2-3 design review meetings).

---

## 1. UNCLEAR POINTS & AMBIGUITIES

### 1.1 Order Lock Mechanism (🔴 CRITICAL)

**Current Text**:
> "Once an order is approved and paid, its calculated timeline is locked. Matrix updates must not alter timelines for orders that are already paid, approved, or scheduled."

**Issues**:
- ❓ How do we **identify** which orders are "paid" and "approved"?
  - Is this stored in an `Order` table in Moyo?
  - Or do we query Amtrack for order status?
  - What if Amtrack DB is unavailable during sync?

- ❓ When exactly does the "lock" happen?
  - At payment confirmation?
  - At approval workflow step?
  - At order creation?

- ❓ How do we **store** the locked SLA date?
  - New table: `OrderSLALock` with (OrderId, CalculatedSLADate, LockedTimestamp)?
  - Or denormalize into `Order.LockedSLADate`?

- ❓ What does "scheduled" mean?
  - Job scheduled for production?
  - Order scheduled for shipment?
  - Both?

**Consequence**: Without clarity, lock enforcement either fails silently or prevents legitimate matrix updates.

**Questions to Ask**:
1. Where is order status information (Paid, Approved, Scheduled) stored? Moyo or Amtrack?
2. Can you provide the exact workflow timeline: Order Created → Approved → Paid → SLA Locked?
3. Should we maintain an audit table `OrderSLALockHistory` for traceability?
4. If order status is in Amtrack, what's the sync mechanism to Moyo? Real-time or batch?

---

### 1.2 Calculation Input Specification (🔴 CRITICAL)

**Current Text**:
> "Inputs (both instances): orderId or a calculation input object (orderType, brandingDepartments, approvalPaidTimestamps, branchDeliveryMode, etc.)"

**Issues**:
- ❓ What is the **complete** list of input fields?
  - The "etc." is too vague for implementation
  - Need exact field names, types, cardinality

- ❓ For `approvalPaidTimestamps`:
  - Single timestamp or separate approval & payment timestamps?
  - Format? ISO-8601 or Unix epoch?
  - What if only approved but not paid? What if paid but not approved?

- ❓ For `brandingDepartments` (array):
  - Is this single or multiple departments?
  - If multiple, is it for multi-jobcard scenario? How do we merge them?
  - What if order has 0 departments (unbranded)?

- ❓ For `branchDeliveryMode`:
  - What are the valid enum values? ("LeadTimeDays", "SpecificDays", "None"?)
  - If "SpecificDays", how do we pass the scheduled dispatch days? (another nested object?)

- ❓ For quantity:
  - Single quantity or array (for multi-item orders)?
  - Does each quantity map to a department? If so, how?

**Consequence**: API consumers won't know what to send; implementation will need reverse-engineering from business logic.

**Questions to Ask**:
1. Can you provide the complete `ProductionLeadTimeInput` GraphQL schema (not just examples)?
2. What's the mapping between order items, quantities, departments, and jobcards?
3. How do we handle orders with mixed branded/unbranded items?
4. Is there an example order JSON/GraphQL mutation showing all fields?

---

### 1.3 Jobcard & Multi-Order Scenarios (🟡 UNCLEAR)

**Current Text** (from Calculation Model):
> "Scenario 2: Multi-Jobcard Order (Same Branding Department)"  
> "Scenario 3: Multi-Jobcard Order (Different Branding Departments - Merged Jobs)"

**Issues**:
- ❓ How does the system **determine** if jobcards are in the same department?
  - Is it a field on the jobcard? (`DepartmentId`?)
  - Or derived from the branding items?

- ❓ What defines a "merged job"?
  - Are they physically merged in production?
  - Or is it a logical calculation rule?
  - Does Moyo need to store the merge relationship?

- ❓ Scenario 2 vs. 3 logic:
  - We take MAX of jobcard times, then add 1 day for Scenario 3
  - But what if 1 jobcard takes 5 days (Dept A) and another takes 3 days (Dept B)?
  - Do we still add 1 day, or is it only for true "merges"?

- ❓ Who decides if jobcards should be merged?
  - Moyo calculation logic?
  - Amtrack workflow?
  - Manual flag on order?

**Consequence**: Calculation engine may apply wrong precedence; orders ship at wrong times.

**Questions to Ask**:
1. Is jobcard merging a logical calculation or a physical production constraint?
2. What fields on jobcards/orders determine if they can/should be merged?
3. Can you walk through a real order example with 2-3 jobcards across departments?
4. Is the +1 day merge penalty a hard rule or configurable per department?

---

### 1.4 Blue/Green Sync Mechanism (🟡 UNCLEAR)

**Current Text**:
> "Flowgear will process these updates daily at night and move them to the live tables."
> "Updates to the matrix only take effect from midnight on the day that the change is made."

**Issues**:
- ❓ What happens if Flowgear sync **fails** at 02:00?
  - Retry logic? (e.g., exponential backoff, max 3 retries?)
  - Manual intervention required?
  - What's the SLA for sync completion?

- ❓ What if an admin makes changes at 01:59 (1 minute before sync)?
  - Does sync pick them up?
  - Or does it wait for tomorrow's run?

- ❓ Partial sync failure scenario:
  - Flowgear syncs 50 of 100 QuantityBreak updates, then fails
  - How do we prevent data inconsistency?
  - Transactional guarantees?

- ❓ Rollback capability:
  - If sync completes but calculations break, can we roll back Green to previous state?
  - How long do we keep Blue records after sync?

- ❓ Audit trail:
  - How do we track which Blue records were synced when?
  - Need sync timestamp on each Blue record?

**Consequence**: Data corruption risk; no recovery path if sync fails mid-operation.

**Questions to Ask**:
1. Is Blue→Green sync a single transaction or multiple transactions per table?
2. What's the rollback/recovery procedure if sync fails partway through?
3. Should we add `SyncTimestamp` and `SyncStatus` fields to Blue tables?
4. How long do we retain Blue records after successful sync?
5. What's the alert/escalation path if Flowgear sync fails?

---

### 1.5 Warehouse vs. Production Lead Time Updates (🟡 UNCLEAR)

**Current Text**:
> "Warehouse lead times are updated immediately."  
> "Production lead times that do not exist in the live tables yet. The insert is immediate. Updates are staged."

**Issues**:
- ❓ Why the asymmetry?
  - Why are warehouse updates immediate but production updates staged?
  - Is there a business reason (e.g., warehouse stock calculates differently)?

- ❓ "Do not exist in live tables" — what triggers an insert vs. update?
  - Is it based on primary key existence?
  - Or a status flag?

- ❓ Warehouse immediate update:
  - Does it bypass Blue tables entirely?
  - Or does it write to both Blue & Green simultaneously?
  - What if admin regrets change? Can they revert?

- ❓ Order lock interaction:
  - If warehouse lead time changes immediately, can it affect locked orders?
  - Or does lock protection apply to warehouse updates too?

**Consequence**: Inconsistent behavior across matrix types; potential for locked order timelines to shift unexpectedly.

**Questions to Ask**:
1. Why do warehouse updates need to be immediate vs. staged?
2. For warehouse immediate updates, do we skip Blue tables or write to both?
3. Does order lock protection apply to warehouse updates?
4. Can an admin undo a warehouse update after it's applied?

---

### 1.6 GraphQL Input Types (🟡 UNCLEAR)

**Current Text**:
> "Inputs (both instances): orderId or a calculation input object (orderType, brandingDepartments, approvalPaidTimestamps, branchDeliveryMode, etc.)"

**Issues**:
- ❓ No GraphQL schema provided for `ProductionLeadTimeInput`
  - What fields are required vs. optional?
  - What are valid enum values?
  - Are there nested objects?

**Example ambiguity**:
```graphql
# Is it like this?
input ProductionLeadTimeInput {
  orderType: String!
  brandingDepartments: [Int!]!
  approvalPaidTimestamp: DateTime
  branchDeliveryMode: String
  quantity: Int
}

# Or like this?
input ProductionLeadTimeInput {
  orderType: OrderType!
  jobcards: [JobcardInput!]!
  items: [OrderItemInput!]!
  branchDelivery: BranchDeliveryInput
}
```

**Consequence**: Frontend/Amtrack team won't know what to send; integration fails.

**Questions to Ask**:
1. Can you provide the complete GraphQL SDL (Schema Definition Language) for all input/output types?
2. Should we version the API (e.g., `productionLeadTimeV1`, `productionLeadTimeV2`)?

---

### 1.7 Special Dates & Department Impact (🟡 UNCLEAR)

**Current Text** (from data model):
```sql
CREATE TABLE SpecialDateImpact (
	ImpactId INT NOT NULL PRIMARY KEY,
	SpecialDateId INT NOT NULL,
	DepartmentId INT NOT NULL
);
```

**Issues**:
- ❓ Does a single SpecialDate affect **all** departments or specific ones?
  - If a date is Public Holiday, does it affect all departments?
  - Or only listed ones in SpecialDateImpact?

- ❓ If a date impacts only some departments:
  - Order with multiple departments spans affected & unaffected depts — what's the final date?
  - MAX of adjusted dates? Or must all depts skip?

- ❓ Example scenario:
  - Order Due: 2026-02-15 (Monday)
  - 2026-02-15 is a special date affecting Dept A only
  - Order has jobcards in Depts A & B
  - What's the final SLA?

**Consequence**: Calculation logic ambiguous for multi-department orders with partial special-date impact.

**Questions to Ask**:
1. Is a SpecialDate always global (affects all departments) or can it be department-specific?
2. For multi-department orders, if special date affects only some depts, how do we calculate final date?
3. Should "Public Holiday" category always affect all departments, while "Shutdown" can be department-specific?

---

### 1.8 Logo24 Cutoff Rules & Time Zone (🟡 UNCLEAR)

**Current Text**:
> "Rule 1 (Monday to Thursday): If the order is approved and paid before 17:00 on a trading day..."  
> "Rule 2 (Friday): If paid before 12:00..."

**Issues**:
- ❓ What time zone?
  - South African Standard Time (SAST, UTC+2)?
  - UTC?
  - Client's local time zone?

- ❓ What is "trading day"?
  - A day that's not a weekend or public holiday?
  - Or a specific business calendar?

- ❓ How do we interpret "before 17:00"?
  - Does it mean 16:59:59 or exactly 17:00:00?
  - Or up to but not including 17:00:00?

- ❓ Logo24 revert to standard lead times:
  - If paid Friday after 12:00, do we calculate with normal lead times?
  - Or is there a fallback lead time for Logo24 overages?

**Consequence**: Off-by-one minute errors; orders miss SLA by 1 hour; customer complaints.

**Questions to Ask**:
1. Which time zone should we use for Logo24 cutoff checks?
2. Is "before 17:00" inclusive or exclusive? (i.e., 16:59 vs. 17:00)
3. What's the fallback lead time if Logo24 order misses cutoff?

---

### 1.9 Add-On Codes & Extensibility (🟡 UNCLEAR)

**Current Text**:
> "Add-ons: Console, CMT, Personalisation..."

**Issues**:
- ❓ Is this a **fixed list** or **extensible** system?
  - Can new add-on types be added in future? (e.g., Gift Wrapping)
  - If extensible, how are they stored & queried?

- ❓ Current data model:
  - `PersonalisationRule` table exists
  - But where's `CMTRule`? `ConsoleRule`?
  - Or are they all in a generic `AddOnRule(type, ...)` table?

- ❓ Can an order have multiple instances of same add-on?
  - (e.g., 2x Console services)
  - Or max 1 of each type?

**Consequence**: May need refactoring if new add-ons added mid-project.

**Questions to Ask**:
1. Is the add-on list fixed (Console, CMT, Personalisation only) or will new ones be added?
2. How should new add-on types be structured in the data model for extensibility?
3. Can an order have multiple instances of the same add-on?

---

## 2. POTENTIAL MISTAKES & INCONSISTENCIES

### 2.1 Console Lead Time Rule Contradiction (🔴 MISTAKE)

**Document 04 says**:
> "Console lead time from warehouse matrix"  
> "If Console present: Client Lead Time = Total lead time + 1 DAY"

**But**:
- ❓ Is Console a **lead time source** (warehouse matrix) or a **modifier** (+1 day)?
- ❓ If Console = warehouse matrix lead time, why add +1 day on top?
- ❓ Does Console apply to branded orders, unbranded orders, or both?

**Current understanding**:
- Base calculation: Base + Add-ons
- Then: If Console, add +1 day to **client-facing** time only (not internal)

**Issue**: Document is unclear on whether Console is:
- A) One of multiple add-ons, each with their own lead times
- B) A modifier that increases client lead time by 1 day after all calculations

**Questions to Ask**:
1. Is Console a component that contributes X days, or a flag that triggers +1 day to client time?
2. Does Console apply to branded and/or unbranded orders?

---

### 2.2 Order Type Override vs. Branding Department Override (🟡 INCONSISTENCY)

**Data Model shows**:
```sql
BrandingDepartment (AdjustmentDays, OverrideDays)
OrderType (AdjustmentDays, OverrideDays)
BrandingGroup (AdjustmentDays, OverrideDays)
```

**Precedence not specified**:
- If both OrderType Override and BrandingDepartment Override exist, which wins?
- Is there a hierarchy? (e.g., BrandingDepartment > OrderType?)

**Consequence**: Ambiguous precedence could lead to wrong lead time selection.

**Questions to Ask**:
1. For a branded order with both OrderType Override and BrandingDepartment Override, which takes precedence?
2. Is there a hierarchy? (e.g., Invoice Code > Setup Code > Branding Department > Branding Group > Quantity Break)?

---

### 2.3 Branch Delivery Mode Enum Values Missing (🟡 INCONSISTENCY)

**Document says**:
> "branchDeliveryMode: String (e.g., 'LeadTimeDays', 'SpecificDays')"

**Missing**:
- No enum definition
- No example of "SpecificDays" input format (which days?)
- No mention of "None" for direct shipment

**Consequence**: Implementation has to guess valid values.

**Questions to Ask**:
1. What are the exact enum values? (e.g., `LeadTimeDays` | `SpecificDays` | `DirectShipment`?)
2. For "SpecificDays" mode, how are scheduled dispatch days passed? (e.g., `dayOfWeek: [TUESDAY, THURSDAY]`?)

---

## 3. GAPS IN SPECIFICATION

### 3.1 Add-On Quantities & Combinations (🔴 GAP)

**Missing**:
- Can an order have **multiple instances** of same add-on? (e.g., 2x Personalisation passes)
- Or is it 0/1 per add-on type?

- Can add-ons be **mutually exclusive**? (e.g., CMT and Console can't both apply)
- Or are they always combinable?

- How do we **pass** add-on details in GraphQL input?
  ```graphql
  # Option A:
  addOns: {
	console: true,
	cmt: false,
	personalisation: true
  }

  # Option B:
  addOns: [
	{ type: "Console", quantity: 1 },
	{ type: "Personalisation", quantity: 2 }
  ]
  ```

**Questions to Ask**:
1. Can an order have multiple instances of the same add-on type?
2. Are any add-ons mutually exclusive?
3. How should the GraphQL input represent add-ons?

---

### 3.2 Calculation Caching Strategy (🟡 GAP)

**Document says**:
> "Caching Strategy: Cache calculation inputs. Clear the cache immediately on active matrix adjustments."

**Missing**:
- What are the **cache keys**?
  - Example: `cache_key = hash(orderType, brandingDepts, quantity_break, addOns, specialDates)`?

- **Cache invalidation precision**:
  - If admin changes 1 BrandingDepartment, do we clear ALL cache?
  - Or only cache entries related to that department?

- **Distributed cache**:
  - Single in-process cache (works for 1 server)?
  - Or distributed cache (Redis) for multi-server setup?

- **Cache TTL**:
  - "24 hours" — is that a hard coded TTL?
  - Or should we invalidate immediately on matrix changes AND have 24h fallback?

**Consequence**: Poor cache efficiency; either memory bloat or repeated DB queries.

**Questions to Ask**:
1. What are the specific cache key components? Can we cache at the QuantityBreak level or must we cache full order calculations?
2. If cache is distributed (Redis), are there sync concerns between instances?
3. Should cache invalidation be fine-grained (per department) or coarse-grained (all)?

---

### 3.3 Amtrack Integration Details (🟡 GAP)

**Document says**:
> "Amtrack will no longer be custodian of data. Amtrack needs to be updated to call the Moyo Gateway for the lead time calculations..."

**Missing**:
- **How** does Amtrack call Moyo?
  - Direct HTTP API? (over VPN?)
  - Message queue? (eventual consistency?)
  - Sync process?

- **Fallback behavior**:
  - If Moyo API is down, does Amtrack:
	- Use local read-only replica (copy of Green tables)?
	- Call legacy calculation engine?
	- Reject orders?

- **Local replica sync**:
  - How often is Amtrack's local replica updated?
  - Real-time via Flowgear? Or batch nightly?
  - What's SLA if replica is stale?

- **Data migration**:
  - What legacy lead time data exists in Amtrack?
  - How do we map old Setup Codes → new Setup Code table?
  - Manual data entry or script?

**Consequence**: Amtrack integration implementation may diverge from intent.

**Questions to Ask**:
1. What's the communication protocol between Amtrack and Moyo? REST API, message queue, or scheduled sync?
2. Should Amtrack maintain a local replica of Green tables, or always call Moyo API?
3. What data migration steps are needed from legacy Amtrack lead time tables?

---

### 3.4 Flowgear Holiday Sync Error Handling (🟡 GAP)

**Flowgear Holiday Sync Process**:
- Fetches SA holidays monthly
- Inserts into `SpecialDate` table

**Missing**:
- ❓ What if OpenHolidays API is down?
  - Retry logic?
  - Alert?
  - Use cached list from previous month?

- ❓ What if holiday data is malformed?
  - Validation rules?
  - Partial vs. full sync failure?

- ❓ What if a holiday is already in the table?
  - Skip (current design)
  - Or update if details changed?

- ❓ Audit trail:
  - Should we log every holiday sync with count & timestamp?
  - For compliance/debugging?

**Questions to Ask**:
1. What's the retry/alerting strategy if OpenHolidays API fails?
2. Should Flowgear log every holiday sync (count, timestamp, success/failure)?
3. Should we maintain a `HolidaySync` audit table?

---

### 3.5 Multi-Department Order Resolution (🟡 GAP)

**Scenario unclear**:
- Order contains jobcards from Dept A, Dept B, Dept C
- Each has different lead times
- How do we calculate the FINAL lead time?

**Current text**:
> "Multi-Jobcard Order (Different Branding Departments - Merged Jobs)"  
> "Internal Order Lead Time = MAX(Jobcard Total Internal Lead Time) + 1 DAY"

**Questions**:
- Is this always "take MAX + 1 day"?
- Or is there a more complex rule?
- What if departments have conflicting `SpecialDateImpact`? (only some affected)

**Example**:
```
Dept A jobcard: 5 days
Dept B jobcard: 7 days (due date: 2026-02-15)
Dept C jobcard: 4 days

Special date: 2026-02-15 affects only Dept B

Final due date?
- 7 days + 1 merge = 8 days = 2026-02-15 (Sat) 
- Skip Saturday → 2026-02-16 (Sun)
- Skip Sunday → 2026-02-17 (Mon)
- But special date also affects 2026-02-15
- Does that matter since we already skipped it?
```

**Questions to Ask**:
1. When multiple departments have special date impacts, do ALL need to be skipped or just the affected ones?
2. Can you provide a worked example of a 3-department order calculation?

---

### 3.6 Performance Testing & Load Test Criteria (🟡 GAP)

**Document says**:
> "Throughput Support: The calculation endpoints must scale up to 10 requests per second (RPS)"  
> "Response Time SLA: Calculation requests... must resolve in under 250ms (99th percentile)"

**Missing**:
- ❓ Load test matrix undefined:
  - What's the CPU/memory baseline?
  - Single server or clustered?
  - How many calculation threads?

- ❓ Cache hit ratio:
  - "Cache hit ratio >85%" — how is this measured?
  - Across what time window?

- ❓ Database indexing requirements:
  - Which columns need indexing for 10 RPS?
  - Have query execution plans been analyzed?

- ❓ Timeout handling:
  - If 1 request takes 300ms, how long do we wait before timeout?
  - Default .NET timeout is 30s — is that acceptable?

**Consequence**: Performance bottlenecks discovered in prod instead of dev.

**Questions to Ask**:
1. What's the baseline environment for the 10 RPS target? (CPU, memory, server config)
2. Which database queries are most critical for performance? (do you have execution plans?)
3. Should we implement query result caching at the database level (materialized views)?

---

### 3.7 Audit Trail Storage & Querying (🟡 GAP)

**Document says**:
> "Detailed Log Structure: Every administrative adjustment... must generate a structured log entry"

**Missing**:
- ❓ Where are logs stored?
  - Database table?
  - File system?
  - Both (database + Signoz)?

- ❓ What's the audit table schema?
  - `UserId, Timestamp, EntityId, EntityType, Action, ValueBefore, ValueAfter, ChangeSource, IPAddress, ...`?

- ❓ Queryability:
  - Can we query "all changes to Department X by User Y in last 7 days"?
  - Need indexes on (UserId, EntityId, Timestamp)?

- ❓ Retention policy:
  - How long do we keep audit logs?
  - Forever, or archival after 1-2 years?

**Questions to Ask**:
1. Should audit logs be stored in a dedicated `AdminChangeLog` table?
2. What's the retention policy for audit logs?
3. Do we need to index audit logs for fast querying by user/entity/date?

---

## 4. CRITICAL DESIGN QUESTIONS FOR ARCHITECT

### 4.1 Order Identity & Tracking

**Q**: How do we uniquely identify an order in Moyo?
- Is there an `Order` table in Moyo, or do we only reference Amtrack orders?
- If referential, how do we sync order status (Paid, Approved) from Amtrack?
- Real-time push or batch pull?

**Q**: When calculating lead time ad-hoc (without orderId), how do we know it's for an existing order vs. a quote?
- Should we persist ad-hoc calculations for audit trail?
- Or only persist calculations for actual orders?

---

### 4.2 Calculation Persistence

**Q**: For existing orders, do we **recalculate** lead times nightly (if matrix changes)?
- Or do we only use the **original calculated** SLA for locked orders?
- If we recalculate for unlocked orders, when do they see the new SLA?

**Example**:
```
Day 1 @ 10:00 - Order created, SLA calculated as 2026-02-20
Day 2 @ 15:00 - Admin changes lead time matrix
Day 3 @ 02:00 - Blue→Green sync applies new matrix
Day 3 @ 10:00 - Does same order now show SLA as 2026-02-21?
(assuming it's still unpaid/unapproved)
```

---

### 4.3 Calculation Reconciliation

**Q**: If a user queries the same order twice (same inputs), should we get identical results?
- Or could special dates, caching, or staging cause variations?
- Need idempotency guarantee?

---

### 4.4 Permission Model for Admin Functions

**Document says**:
> "Administrative functions for editing/updating dates, adjustments and overrides should be permission-driven as per the BRD."

**Q**: What are the specific permission levels?
- Can anyone update lead times?
- Only managers?
- Role-based access control (RBAC) with specific roles?
- Should permission changes also be audited?

---

### 4.5 Multi-Tenancy or Single-Tenant?

**Gap**: Is Moyo:
- Single-tenant (Amrod only)?
- Multi-tenant (multiple customers, each with own matrices)?

**If multi-tenant**:
- How do we isolate matrices per tenant?
- Can tenants see each other's calculations?
- Should we add `TenantId` to all tables?

---

### 4.6 GraphQL vs. REST API

**Gap**: Document specifies GraphQL, but:
- Is REST API also needed? (for Amtrack integration?)
- Are they both exposed, or GraphQL only?
- Should there be separate endpoints for internal vs. external?

---

## 5. IMPLEMENTATION RISK AREAS

### 5.1 🔴 Order Lock Enforcement (HIGH RISK)

**Why**: Without clear specification of what "locked" means, implementation will either:
- Silently fail to protect orders (wrong SLA dates)
- Over-protect and prevent legitimate matrix updates

**Mitigation**: Get architect to specify:
1. Order status source (Moyo vs. Amtrack)
2. Lock timing (when exactly)
3. Storage mechanism (how to query locked orders efficiently)

---

### 5.2 🔴 Multi-Department Calculation (HIGH RISK)

**Why**: Current spec is ambiguous on how to merge multiple departments' lead times.

**Consequence**: Orders with multiple departments could get wrong SLA.

**Mitigation**: Ask architect for worked example with 2-3 departments across different scenarios.

---

### 5.3 🟡 Flowgear Sync Reliability (MEDIUM RISK)

**Why**: If Blue→Green sync fails mid-operation, no clear recovery path.

**Consequence**: Stale matrices; orders use outdated lead times.

**Mitigation**: Define transaction boundaries, rollback logic, alerting.

---

### 5.4 🟡 Performance at Scale (MEDIUM RISK)

**Why**: 250ms SLA for 10 RPS on complex multi-table joins is aggressive.

**Consequence**: May miss SLA in production load; requires optimization.

**Mitigation**: Prototype complex calculation queries; measure latency early.

---

## 6. RECOMMENDED FOLLOW-UP ACTIONS

### Before Development Starts (Week 1)

- [ ] **Schedule Design Review Meeting #1** (2 hours)
  - Clarify order lock mechanism
  - Define calculation input schema (complete GraphQL schema)
  - Clarify multi-department calculation logic

- [ ] **Schedule Design Review Meeting #2** (1.5 hours)
  - Define Blue/Green sync transaction boundaries & error handling
  - Clarify warehouse vs. production update asymmetry
  - Define audit trail storage & retention

- [ ] **Architect creates**:
  - Complete GraphQL SDL for all input/output types
  - Worked example order calculations (single, multi-dept, with special dates)
  - Data migration strategy (Amtrack → Moyo)
  - Amtrack integration protocol (how it calls Moyo API)

---

### During Development (Ongoing)

- [ ] **Prototype**: Implement calculation engine; measure performance on sample data
- [ ] **Prototype**: Blue/Green sync logic; test failure scenarios
- [ ] **Clarify**: As edge cases arise during coding, document and feed back to architect

---

## 7. QUESTIONS TO ASK IN DESIGN REVIEW MEETINGS

### Meeting #1 Agenda

1. **Order Locks**:
   - Where is order status stored? Moyo or Amtrack?
   - What's the exact workflow: Created → Approved → Paid → Locked?
   - Should we maintain `OrderSLALock` table?

2. **Calculation Inputs**:
   - Can you provide the complete `ProductionLeadTimeInput` GraphQL schema?
   - How do quantities map to departments/jobcards?
   - Can an order have mixed branded/unbranded items?

3. **Multi-Department Orders**:
   - Walk through 2-3 department order calculation with concrete numbers
   - If special dates affect only some depts, what's the final date?

4. **Warehouse Updates**:
   - Why immediate vs. staged? Business reason?
   - Should lock protection apply?

---

### Meeting #2 Agenda

1. **Blue/Green Sync**:
   - Single transaction across all tables, or per-table?
   - Rollback mechanism if sync fails mid-operation?
   - Alert & SLA if sync fails?

2. **Add-Ons**:
   - Fixed list or extensible?
   - Multiple instances of same type allowed?
   - Can they be mutually exclusive?

3. **Amtrack Integration**:
   - How does it call Moyo? (REST, message queue, sync?)
   - Fallback if Moyo down?
   - Local replica sync frequency?

4. **Performance & Testing**:
   - Have complex queries been analyzed for 10 RPS?
   - Database indexing strategy?
   - Load test plan & criteria?

---

## Summary Table: Clarification Priority

| Item | Priority | Impact | Status |
|------|----------|--------|--------|
| Order lock mechanism | 🔴 CRITICAL | Calculation correctness | Needs spec |
| Calculation input schema | 🔴 CRITICAL | API integration | Needs spec |
| Multi-dept calculation | 🔴 CRITICAL | Calculation correctness | Needs clarity |
| Blue/Green sync errors | 🟡 HIGH | Data integrity | Needs spec |
| Warehouse vs. prod asymmetry | 🟡 HIGH | Implementation consistency | Needs justification |
| Add-on extensibility | 🟡 MEDIUM | Future maintenance | Needs decision |
| Amtrack integration protocol | 🟡 MEDIUM | Integration success | Needs spec |
| Performance requirements | 🟡 MEDIUM | SLA compliance | Needs validation |
| Audit trail storage | 🟡 MEDIUM | Observability | Needs spec |
| Flowgear error handling | 🟡 MEDIUM | Reliability | Needs spec |

---

## Final Checklist

Before the first line of code is written, ensure:

- [ ] Order lock mechanism is fully specified (source, timing, storage)
- [ ] Complete GraphQL schema is available (input & output types)
- [ ] Multi-department calculation is clarified with examples
- [ ] Blue/Green sync transaction boundaries & error handling defined
- [ ] Amtrack integration protocol is documented
- [ ] Add-on types & combinations rules are defined
- [ ] Audit trail storage & retention policy decided
- [ ] Performance testing plan & baseline environment defined
- [ ] Flowgear sync reliability & alerting rules defined
- [ ] All team members have reviewed and signed off on clarifications

---

**Document Version**: 1.0  
**Prepared By**: Development Team  
**Next Action**: Schedule Design Review Meetings with Architect  
**Timeline**: Complete clarifications by end of Week 1 before development begins
