# Story: Lead Time Matrix - GraphQL Endpoint Setup (Internal & External instances)

**BRD References:** N/A (architecture-driven)
**Tech Design References:** TechnicalDesign/07 (API Contracts - Calculation Query)
**Dev Hours:** 12

---

## Overview

Stand up **two distinct GraphQL endpoint instances** exposed by the LeadTimeMatrix module:

1. **Internal GraphQL** — authenticated, ops/admin use. Supports `useStaged` and `includeBreakdown` flags. Returns full breakdown details, including blue (staged) data.
2. **External GraphQL** — public/partner-facing. Safe subset only — no staged data, no breakdown.

Both instances expose the same `productionLeadTime` query but differ in:
- Auth requirements
- Available query parameters
- Response shape

The shared resolver invokes the calculation pipeline (delivered by stories 3.3 and 3.4).

---

## Routing

Both endpoints are routed under the module's API versioning prefix (see the **API versioning at /api/v1/** story):

| Instance | Path | Auth | Schema |
|----------|------|------|--------|
| Internal | `/api/v1/leadtimematrix/graphql/internal` | Required (Moyo session/JWT) | Full schema with `useStaged`, `includeBreakdown`, `breakdown` field |
| External | `/api/v1/leadtimematrix/graphql/external` | Gateway-issued partner key | Subset schema — no `useStaged`, no `breakdown` field |

---

## Schema Split

Use HotChocolate's **schema name** feature to register two schemas in the same host:

```csharp
services
	.AddGraphQLServer("internal")
	.AddQueryType<InternalQuery>()
	.AddType<ProductionLeadTimeInternalType>()
	.AddType<BreakdownType>()
	.AddAuthorization();

services
	.AddGraphQLServer("external")
	.AddQueryType<ExternalQuery>()
	.AddType<ProductionLeadTimeExternalType>();
```

Map them in `Program.cs`:

```csharp
app.MapGraphQL("/api/v1/leadtimematrix/graphql/internal", schemaName: "internal")
   .RequireAuthorization();

app.MapGraphQL("/api/v1/leadtimematrix/graphql/external", schemaName: "external");
```

External auth (partner key) is enforced by the existing API Gateway — the module trusts the gateway and does **not** re-validate identity for external calls, but the route must be configured to require the gateway's standard partner-key middleware.

---

## Internal Query

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
	slaDate
	leadTime
	mode
	breakdown @include(if: $includeBreakdown) {
	  baseDays
	  addOns { code days }
	  adjustmentDays
	  overrideApplied
	  calendarShifts { date reason }
	  finalComputationPath
	}
  }
}
```

### Internal Inputs
- `orderId: ID` (optional) - Look up calculation context from an existing order
- `input: ProductionLeadTimeInput` (optional) - Inline calculation context (orderType, brandingDepartments, approvalPaidTimestamps, branchDeliveryMode, etc.)
- `useStaged: Boolean` (default `false`) - When `true`, use staged (blue) lead times
- `includeBreakdown: Boolean` (default `false`) - When `true`, include the `breakdown` field

### Internal Response
- `slaDate: DateTime` (ISO-8601)
- `leadTime: Int` (days)
- `mode: String` (`live` | `staged` — server-inferred from `useStaged`)
- `breakdown: BreakdownType` (only populated when `includeBreakdown = true`)

---

## External Query

```graphql
query productionLeadTime($orderId: ID, $input: ProductionLeadTimeInput) {
  productionLeadTime(orderId: $orderId, input: $input) {
	slaDate
	leadTime
	mode
  }
}
```

### External Inputs
- `orderId: ID` (optional)
- `input: ProductionLeadTimeInput` (optional)
- **No** `useStaged` flag — external callers can never read staged data
- **No** `includeBreakdown` flag — external responses never include `breakdown`

### External Response
- `slaDate`
- `leadTime`
- `mode` (always `"live"` — hardcoded, since external never has access to staged)

---

## Resolver

Both schemas delegate to the **same** application service (`IProductionLeadTimeCalculator`) registered in `LeadTimeMatrix.Application`. The resolver layer is responsible only for:

- Mapping GraphQL input → application command
- Detecting which schema is being queried (and therefore whether `useStaged` is allowed)
- Mapping the application result → schema-specific response type
- Applying the `Internal`-only `includeBreakdown` projection

The resolver MUST NOT contain calculation logic — that lives in the application/domain layer.

---

## Date/Time Inputs

When the calculation context includes the action date (`approvalPaidTimestamp`), the resolver returns an absolute `slaDate`. When only `leadTime` (in days) can be computed (no action date provided), `slaDate` is `null` and only `leadTime` is returned.

---

## Acceptance Criteria

- [ ] Two GraphQL endpoints are exposed: `/api/v1/leadtimematrix/graphql/internal` and `/api/v1/leadtimematrix/graphql/external`
- [ ] Internal endpoint requires authentication; external endpoint does not but expects gateway-issued partner key
- [ ] Internal query accepts `useStaged` and `includeBreakdown`; external query does not
- [ ] External response shape contains only `slaDate`, `leadTime`, `mode`; never `breakdown`
- [ ] External `mode` is always `"live"` regardless of any client-supplied flag
- [ ] Both endpoints invoke the same `IProductionLeadTimeCalculator` application service
- [ ] When `useStaged = true` (Internal only), the calculator returns staged (blue) results and `mode = "staged"`
- [ ] When `includeBreakdown = true` (Internal only), the response includes `baseDays`, `addOns`, `adjustmentDays`, `overrideApplied`, `calendarShifts`, `finalComputationPath`
- [ ] HotChocolate schema registration uses **named schemas** (`"internal"` and `"external"`) so they coexist in the same host
- [ ] Schema is exposed via standard introspection (Banana Cake Pop / Apollo Sandbox) on the Internal endpoint only — disable introspection on the External endpoint in production
- [ ] Functional tests cover: live calculation (both endpoints), staged calculation (Internal only), breakdown response shape (Internal only), external endpoint rejects `useStaged` parameter (schema validation error)

---

## Out of Scope

- The actual calculation pipeline implementation — that is delivered by **3.3-calculate-jobcard-lead-time.md** and **3.4-calculate-order-lead-time.md**
- Mutations / write endpoints — calculations are read-only queries; admin writes use REST endpoints from the BRD 2.x stories
- Subscriptions — not part of this module's scope
- Caching — handled by the **calculation-input-caching.md** story
