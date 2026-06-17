# Story: Lead Time Matrix - Flowgear Sync Support APIs (Get pending / Mark synced)

**BRD References:** N/A (Flowgear integration support)
**Tech Design References:** TechnicalDesign/07 (Flowgear Sync Updates), TechnicalDesign/08 (Synchronisation Directionality)
**Dev Hours:** 8

---

## Overview

Implement the **internal API endpoints** that Flowgear consumes during the daily production-matrix sync (TechnicalDesign/07 and /08). The flow is:

1. Flowgear runs nightly (02:00) per TechnicalDesign/08.
2. Flowgear calls `GET /pending` to retrieve all Update (Blue) records in `New` status.
3. Flowgear applies each record to the live (Green) tables in its own workflow.
4. Flowgear calls `POST /mark-synced` with the list of record IDs and a UTC timestamp once each batch has been applied.
5. The module updates each Update record's `Status` to `Synced` and stamps `UpdateDt` with the supplied UTC timestamp.

These two endpoints are **the only** way the module exposes Update-table state changes to Flowgear; all writes back to Update records flow through `mark-synced`.

> **Note:** The actual application of Blue → Green data is performed by Flowgear, not by this API. This story covers only the *support* endpoints the workflow needs.

---

## Routing & Auth

Both endpoints live under the module's API versioning prefix (see the **API versioning at /api/v1/** story) and use the same internal auth policy as other Flowgear ↔ Moyo integrations:

| Endpoint | Method | Path |
|----------|--------|------|
| Get pending Update records | `GET` | `/api/v1/leadtimematrix/sync/pending` |
| Mark records as synced | `POST` | `/api/v1/leadtimematrix/sync/mark-synced` |

Auth: **machine-to-machine token** (existing Flowgear service principal).

---

## GET /sync/pending

Returns all Update records in `New` status grouped by entity type. Production updates only — warehouse updates apply immediately and have no Blue counterpart.

### Query parameters (optional)

- `batchSize: int` (default `1000`, max `5000`) - Page size cap; the response includes `hasMore` so Flowgear can re-poll until empty
- `entity: string` (optional) - Filter to a single Update entity (e.g. `BrandingGroup`, `QuantityBreak`, `PrintCode`, `InvoiceCode`, `BrandingByItem`, `BrandingDepartment`, `OrderType`)

### Response

```json
{
  "snapshotTakenAt": "2026-01-15T02:00:00Z",
  "hasMore": false,
  "groups": [
	{
	  "entity": "QuantityBreak",
	  "records": [
		{
		  "recordId": 1234,
		  "entityId": 567,
		  "operation": "Upsert",
		  "payload": { /* full snapshot of the Update row */ }
		}
	  ]
	},
	{
	  "entity": "BrandingGroup",
	  "records": [ /* ... */ ]
	}
  ]
}
```

Notes:
- `recordId` is the PK of the Update (Blue) row — Flowgear must echo these back in `mark-synced`.
- `entityId` is the FK to the Live (Green) row that should be inserted/updated.
- `operation` is always `Upsert` (the module never deletes via Update tables; deletes apply to Green directly per the data-model story).
- `payload` is the full snapshot of the Update row so Flowgear can apply it to Green without re-querying.

---

## POST /sync/mark-synced

Marks a list of Update records as `Synced` and stamps the supplied UTC timestamp.

### Request

```json
{
  "syncedAt": "2026-01-15T02:35:14Z",
  "recordIds": [1234, 1235, 1236, 1237]
}
```

### Response

```json
{
  "marked": 4,
  "skipped": 0,
  "skippedDetails": []
}
```

### Behaviour

- The request is **idempotent**. Records already in `Synced` status are silently skipped (counted in `skipped` and listed in `skippedDetails` for audit visibility).
- Records NOT in `New` status (e.g. already `Synced`, or in any future status) are returned as `skipped` rather than failing the entire batch.
- Records that don't exist are returned as `skipped` with reason `NotFound`.
- The `syncedAt` timestamp must be UTC and must be within ±15 minutes of server time — clock skew beyond that returns `400 Bad Request`.
- The whole batch is processed in a single SQL `UPDATE … WHERE RecordId IN (…) AND Status = 'New'` for performance.

### Skipped detail shape

```json
{
  "recordId": 1234,
  "reason": "AlreadySynced" | "NotFound"
}
```

---

## Concurrency & Audit

- The Update tables are **append-only** for record creation (per the data-model story); only `Status` and `UpdateDt` are mutated by this endpoint.
- `mark-synced` writes are wrapped in a single transaction per request.
- Every call to both endpoints emits a structured log entry per the **observability-admin-audit-logs.md** story:
  - `Operation` (`SyncPending` | `SyncMarkSynced`)
  - `RecordCount`, `BatchSize`, `DurationMs`
  - For `mark-synced`: `MarkedCount`, `SkippedCount`

---

## Acceptance Criteria

- [ ] `GET /api/v1/leadtimematrix/sync/pending` returns Update records in `New` status grouped by entity type
- [ ] Response respects `batchSize` (default 1000, max 5000) and exposes `hasMore` for paging
- [ ] Optional `entity` filter narrows the response to a single Update entity
- [ ] Each record includes `recordId`, `entityId`, `operation`, and the full `payload` snapshot
- [ ] `POST /api/v1/leadtimematrix/sync/mark-synced` accepts `syncedAt` (UTC) and `recordIds[]`
- [ ] The endpoint updates each record's `Status` from `New` to `Synced` and stamps `UpdateDt = syncedAt`
- [ ] The endpoint is **idempotent**: records already `Synced` or not found are returned in `skipped` rather than failing the batch
- [ ] Clock skew on `syncedAt` beyond ±15 minutes returns `400 Bad Request`
- [ ] Both endpoints require Flowgear's machine-to-machine auth token
- [ ] Both endpoints emit structured Signoz logs per the observability story
- [ ] Functional tests cover: empty pending response, paged response with `hasMore`, entity filter, mark-synced happy path, partial idempotent skip, skew rejection
- [ ] Performance test: `pending` returns 5000 records in under 500ms; `mark-synced` of 5000 records completes in under 1000ms

---

## Out of Scope

- The Flowgear workflow itself (defined in Flowgear's own tooling per TechnicalDesign/07's flow diagrams).
- Apply-time validation of Blue → Green transformations — Flowgear is responsible for those.
- Deletion semantics on Update tables — records are retained permanently for audit per TechnicalDesign/07.
- Re-running a previously synced batch — once a record is `Synced` it stays that way; any required corrections create a *new* Update row.
