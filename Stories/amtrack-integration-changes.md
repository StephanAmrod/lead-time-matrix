# Story: Lead Time Matrix - Amtrack Integration Changes (delegate to Moyo + circuit breaker)

**BRD References:** N/A (architecture-driven)
**Tech Design References:** TechnicalDesign/08 (Dual-Run Synchronisation & Integrity), TechnicalDesign/09 (Decommission Plan), TechnicalDesign/11 (NFR - Failover/Availability)
**Dev Hours:** 16

---

## Overview

Amtrack currently performs lead time calculations against its own local database. Per TechnicalDesign/08, **Moyo becomes the single source of truth** ("Golden Record") and Amtrack must **stop computing lead times locally** and instead call the Moyo Gateway.

This story covers:

1. Replacing Amtrack's internal lead time calculation calls with calls to Moyo's GraphQL endpoint
2. Implementing the **circuit-breaker fallback** required by TechnicalDesign/11
3. Removing legacy Amtrack calculation paths (per TechnicalDesign/09)
4. Auth, retry, observability around the new integration

---

## Direction of Travel

| Concern | Before | After |
|---------|--------|-------|
| Where calculations run | Amtrack (local DB) | Moyo (via GraphQL) |
| Source of lead time data | Amtrack DB | Moyo DB (Golden Record) |
| Sync direction | Bi-directional drift | **Unidirectional** Moyo → Amtrack (configuration sync only, not for calc) |
| Amtrack's role | Custodian | Consumer (with read-only fallback) |

Per TechnicalDesign/08: *"Data changes in Moyo will not sync back to Amtrack for Lead Time calculations. This causes drift over time, making continued reliance on Amtrack calculations inaccurate."*

---

## Integration Pattern

Amtrack must call the Moyo **External GraphQL endpoint** (see [graphql-endpoint-setup.md](graphql-endpoint-setup.md)):

```
POST https://moyo-gateway.amrod.local/api/v1/leadtimematrix/graphql/external
```

with the standard `productionLeadTime` query. Response shape:

```json
{
  "data": {
	"productionLeadTime": {
	  "slaDate": "2026-02-15T17:00:00Z",
	  "leadTime": 5,
	  "mode": "live"
	}
  }
}
```

### Auth

Amtrack uses its existing service principal / partner key issued by the Moyo gateway. No special permissions beyond the standard external-instance access.

### Retry policy

| Attempt | Delay |
|---------|-------|
| 1 | immediate |
| 2 | 200ms |
| 3 | 500ms |

After 3 failed attempts the **circuit breaker** opens (see below).

---

## Circuit Breaker (TechnicalDesign/11)

Required behaviour per TechnicalDesign/11: *"If Moyo API responses exceed 1,500ms or fail three times consecutively, Amtrack fails back to its local read-only database replica."*

| Trigger | State |
|---------|-------|
| 3 consecutive failures (timeout, 5xx, network error) | **Open** |
| Response time > 1,500ms | Counted as a failure for circuit purposes |
| Half-open probe succeeds | **Closed** |
| Half-open probe fails | **Open** |

### State behaviour

- **Closed (healthy)** - All Amtrack calculations call Moyo; results are returned live.
- **Open** - Amtrack falls back to its **read-only** local replica. The replica is the legacy Amtrack DB schema, kept in read-only mode. Results computed from the replica are flagged `source: "Amtrack-Fallback"` in Amtrack's own logs.
- **Half-Open** - After 60 seconds in `Open`, allow one probe call to Moyo. On success, transition to `Closed`. On failure, stay `Open` for another 60 seconds.

### Configuration

```json
{
  "MoyoGateway": {
	"BaseUrl": "https://moyo-gateway.amrod.local/",
	"RequestTimeoutMs": 1500,
	"RetryCount": 3,
	"CircuitBreakerOpenDurationSeconds": 60,
	"FallbackReplicaConnectionString": "<from secrets>"
  }
}
```

---

## Removal of Legacy Calculation Paths (TechnicalDesign/09)

Once the Moyo integration is in place and the parity run (see [parity-run-shadow-calculation.md](parity-run-shadow-calculation.md)) confirms correctness, Amtrack's **internal lead time calculation code paths must be removed**:

1. Identify all Amtrack code paths that compute lead times locally (search for `LeadTime`, `CalculateDueDate`, `BrandingDays`, etc. in the Amtrack codebase).
2. Replace each call site with a call to `IMoyoLeadTimeClient.GetProductionLeadTimeAsync(...)`.
3. The local replica path is reachable **only** through the circuit-breaker fallback - it is no longer the primary code path.
4. Delete or quarantine any tests asserting the old internal calculation behaviour - these are superseded by Moyo's tests and the parity run.

> **Note on rollback:** The deployment-rollback story ([rollback feature toggle](#)) supports flipping back to the **full** legacy path (not just the read-only replica) during cutover sanity testing. After cutover is signed off, that toggle is disabled.

---

## Observability

Per TechnicalDesign/11, every Moyo call from Amtrack must emit a structured log entry:

- `Operation` = `MoyoLeadTimeCall`
- `OrderId` / input identifiers
- `DurationMs`
- `Outcome` = `Success` | `Timeout` | `RemoteError` | `FallbackUsed`
- `CircuitState` = `Closed` | `Open` | `HalfOpen`
- `RetryCount`

Logs forward to **Signoz**. Add an alert when `FallbackUsed` rate exceeds 1% over a rolling 5-minute window.

---

## Acceptance Criteria

- [ ] A new `IMoyoLeadTimeClient` abstraction exists in Amtrack's codebase, calling `/api/v1/leadtimematrix/graphql/external`
- [ ] All previously-internal lead time calculation call sites in Amtrack route through the new client
- [ ] The client uses the standard Moyo gateway service principal for auth
- [ ] Retry policy: 3 attempts (200ms, 500ms backoff)
- [ ] Circuit breaker opens on 3 consecutive failures **OR** any response > 1,500ms
- [ ] Open-state requests fall through to a **read-only** local replica path that returns `source: "Amtrack-Fallback"` in the response context
- [ ] Half-open probes return the circuit to closed on success
- [ ] Configuration values for timeouts, retry, circuit duration, and fallback replica connection are externalised
- [ ] All calls emit Signoz logs with `Operation`, `Outcome`, `CircuitState`, `RetryCount`, `DurationMs`
- [ ] A Signoz alert fires when `FallbackUsed` rate exceeds 1% over 5 minutes
- [ ] Legacy internal-calculation code in Amtrack is **removed** (not commented out) once parity is signed off; the read-only replica path is reachable only through the circuit breaker
- [ ] Tests cover: closed-state happy path, retry-then-success, retry-exhaustion → open, response > 1500ms → open, half-open success, half-open failure, fallback returns replica result with correct source flag

---

## Out of Scope

- The Moyo GraphQL endpoint implementation - delivered by [graphql-endpoint-setup.md](graphql-endpoint-setup.md)
- The parity comparison harness - delivered by [parity-run-shadow-calculation.md](parity-run-shadow-calculation.md)
- The deployment-time rollback toggle - delivered by the **rollback feature toggle** story
- Amtrack UI changes (this is a back-end integration story only)
- Decommissioning Amtrack entirely - the read-only replica remains as the fallback during the transition period
