# Story: Lead Time Matrix - Observability & Admin Audit Logs forwarded to Signoz

**TD Non-Functional Requirements : Observability**

**BRD References:** N/A (NFR-driven)
**Tech Design References:** TechnicalDesign/11 (Observability)
**Dev Hours:** 8

---

## Overview

Per TechnicalDesign/11: *"Every administrative adjustment to overrides, print codes, or special dates must generate a structured log entry detailing `UserId`, `Timestamp`, `EntityId`, and `ChangeDelta` (Value Before vs. Value After). Send logs to the centralised monitoring stack (Signoz) for analysis."*

Implement the cross-cutting **admin audit log** infrastructure for the LeadTimeMatrix module:

1. A structured-log entry is emitted for **every** admin write that influences calculation inputs.
2. Every entry contains the four mandatory fields plus contextual metadata.
3. All module logs (audit + operational) are forwarded to **Signoz** with consistent tags.
4. A baseline set of dashboards/alerts is provisioned for SLA breaches.

---

## Scope of Audited Operations

The audit log MUST be emitted for every write to the following entities (covers all "administrative adjustments to overrides, print codes, or special dates" plus the broader matrix configuration):

| Domain Area | Entities |
|-------------|----------|
| Special Dates | `SpecialDate`, `SpecialDateCategory` |
| Branding hierarchy | `BrandingDepartment`, `PrintCode`, `InvoiceCode`, `SetupCode`, `BrandingGroup`, `BrandingByItem` |
| Quantity matrix | `QuantityBreak` |
| Order types | `OrderType` (and warehouse-side groups under each) |
| Adjustments / overrides | Any `AdjustmentDays` or `OverrideDays` field on the entities above |

The same audit applies to operations performed via the Update (Blue) tables - the log records the Blue write, not the eventual Green sync.

---

## Audit Entry Schema

Every audited write produces a structured log entry with the following fields:

| Field | Type | Required | Notes |
|-------|------|----------|-------|
| `UserId` | string | yes | Authenticated user; for system jobs use `system:flowgear-sync`, `system:openholidays-sync`, etc. |
| `Timestamp` | DateTimeOffset | yes | UTC, ISO-8601 with offset |
| `EntityType` | string | yes | E.g. `SpecialDate`, `BrandingGroup`, `QuantityBreak` |
| `EntityId` | string | yes | PK of the affected row (or composite key serialised) |
| `Operation` | enum | yes | `Create` \| `Update` \| `Delete` |
| `ChangeDelta` | object | yes for `Update` | `{ "<fieldName>": { "before": <value>, "after": <value> } }` - only changed fields |
| `Snapshot` | object | yes for `Create` and `Delete` | Full entity snapshot |
| `Module` | string | yes | Always `"LeadTimeMatrix"` |
| `CorrelationId` | string | yes | Request id from `IHttpContextAccessor` or job id for system writes |
| `Source` | string | yes | `"Web"`, `"GraphQL"`, `"Flowgear"`, `"OpenHolidays"`, etc. |

### Example

```json
{
  "Module": "LeadTimeMatrix",
  "UserId": "user:12345",
  "Timestamp": "2026-01-15T14:23:01.512Z",
  "EntityType": "QuantityBreak",
  "EntityId": "789",
  "Operation": "Update",
  "ChangeDelta": {
	"LeadTimeDays": { "before": 5, "after": 7 },
	"UpperLimit":   { "before": 100, "after": 150 }
  },
  "CorrelationId": "5f3c1a8b-...",
  "Source": "Web"
}
```

---

## Implementation

### Audit interceptor

Add an EF Core `SaveChangesInterceptor` or domain-event handler in `LeadTimeMatrix.Infrastructure/Audit/`:

```csharp
public sealed class AuditingSaveChangesInterceptor : SaveChangesInterceptor
{
	public override ValueTask<InterceptionResult<int>> SavingChangesAsync(...) {
		// For each tracked entity in the audited list:
		//   - capture Before snapshot from EntityEntry.OriginalValues
		//   - capture After snapshot from EntityEntry.CurrentValues
		//   - compute ChangeDelta (only differing fields)
		//   - enqueue an audit record on the current ICurrentAuditScope
	}

	public override async ValueTask<int> SavedChangesAsync(...) {
		// After commit succeeds, emit the queued audit logs via ILogger
		// (so a rolled-back transaction never produces a misleading audit trail)
	}
}
```

Register against the `ApplicationDbContext`:

```csharp
options.AddInterceptors(new AuditingSaveChangesInterceptor(...));
```

### User context

`UserId` and `CorrelationId` come from a small `ICurrentUser` abstraction backed by `IHttpContextAccessor` (web) or a per-job scope (system writes). The `Source` is set per integration entry-point.

### Logger

Use `ILogger<T>` with **structured templates** so Signoz can index the fields:

```csharp
_logger.LogInformation(
	"Audit {Operation} {EntityType} {EntityId} by {UserId} from {Source} ({CorrelationId})",
	record.Operation, record.EntityType, record.EntityId,
	record.UserId, record.Source, record.CorrelationId);
```

The full payload (including `ChangeDelta` / `Snapshot`) goes via the structured-state extension method so it is captured as JSON, not flattened to the message string.

---

## Signoz Integration

The Moyo platform already forwards logs to Signoz via OpenTelemetry. This story:

1. Adds the `LeadTimeMatrix` module name as a Signoz tag on every emitted log.
2. Adds an explicit log category `"LeadTimeMatrix.Audit"` so the audit stream is cleanly filterable.
3. Provisions a **module dashboard** in Signoz with:
   - Audit events per minute (split by `Operation`)
   - Top users performing changes
   - Per-entity write counts
   - SLA panel: `productionLeadTime` GraphQL p99 latency (target < 250ms per TechnicalDesign/11)
   - Cache hit-rate (from [calculation-input-caching.md](calculation-input-caching.md))
   - Pimcore validation outcomes (from [pimcore-validation-service.md](pimcore-validation-service.md))
   - Amtrack circuit-breaker state (from [amtrack-integration-changes.md](amtrack-integration-changes.md))

### Alerts

| Alert | Condition | Severity |
|-------|-----------|----------|
| GraphQL latency SLA breach | `productionLeadTime` p99 > 250ms over 5 min | High |
| Throughput collapse | RPS drops > 50% from rolling baseline | Medium |
| Audit silence | No audit events from a known-busy entity for > 1 hour during business hours | Low |
| Pimcore failures | Failure rate > 5% over 5 min | High |
| Amtrack fallback rate | `FallbackUsed` > 1% over 5 min | High |

---

## Performance Considerations

- The interceptor must be **non-blocking** on the user request — log emission is fire-and-forget after the DB commit succeeds.
- For batch writes (e.g. bulk imports), audit entries are emitted in batches but each row still gets its own structured entry — Signoz queries depend on per-row granularity.
- Log payloads are bounded — `Snapshot` and `ChangeDelta` exclude `byte[]` columns and truncate strings > 4KB to prevent log explosion.

---

## Acceptance Criteria

- [ ] An `AuditingSaveChangesInterceptor` (or equivalent domain-event handler) is registered against the LeadTimeMatrix `ApplicationDbContext`
- [ ] Audit entries are emitted for every Create/Update/Delete on the entities listed in *Scope of Audited Operations*
- [ ] Every audit entry contains `UserId`, `Timestamp`, `EntityType`, `EntityId`, `Operation`, `Module`, `CorrelationId`, `Source`
- [ ] `Update` entries include a `ChangeDelta` of only the fields that actually changed
- [ ] `Create` and `Delete` entries include a full `Snapshot`
- [ ] System writes (Flowgear sync, OpenHolidays sync) emit audits with `UserId = "system:<source>"`
- [ ] Audit entries are emitted **after** transaction commit, so rolled-back changes don't produce misleading entries
- [ ] All module logs are forwarded to Signoz with the `LeadTimeMatrix` module tag and the `LeadTimeMatrix.Audit` category for audit logs
- [ ] A Signoz dashboard exists with the panels listed above
- [ ] All alerts in the *Alerts* table are configured and route to the on-call channel
- [ ] Performance: audit overhead on a single-row write is < 5ms; on a 1000-row batch, < 100ms total
- [ ] Unit tests verify field capture (Before/After), system-source propagation, transaction-rollback suppression, and field truncation
- [ ] Integration test verifies an end-to-end write produces a structured Signoz-shaped log entry

---

## Out of Scope

- A user-visible audit history UI — this is a backend audit-log story only; UI work (if needed) is tracked separately.
- Long-term audit storage / archival — handled by Signoz's retention policy plus the underlying log shipping infrastructure.
- PII redaction — the audited fields are configuration data, not personal data; no special PII handling required.
- Audit for read operations — only writes are audited per TechnicalDesign/11.
