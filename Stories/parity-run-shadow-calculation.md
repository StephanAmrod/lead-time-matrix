# Story: Lead Time Matrix - Parity Run shadow-calculation comparison

**BRD References:** N/A (deployment-driven)
**Tech Design References:** TechnicalDesign/12 (Parity Run)
**Dev Hours:** 12

---

## Overview

Per TechnicalDesign/12: *"Run a parity comparison for 5 business days where Moyo calculations run alongside Amtrack and the results are compared. Cutover only happens after sign-off on the parity report."*

Build the **shadow-calculation harness** that:

1. Captures every production lead time calculation request that hits Amtrack during the parity window.
2. Replays the same input through the Moyo `productionLeadTime` GraphQL endpoint.
3. Compares the two responses field-by-field, classifies any differences, and persists the comparison.
4. Produces a **daily go/no-go dashboard** that operators use to sign off on cutover.

The harness runs for **5 business days** before cutover with both calculation engines live; only Amtrack's response is returned to callers during this window.

---

## Architecture

The harness is an **out-of-band shadow runner** — it must not add latency or risk to the live Amtrack response path.

```
   Caller ─────► Amtrack (live, returns response)
					│
					└─► Audit queue (fire-and-forget)
							  │
							  ▼
					   Parity Worker (Moyo side)
							  │
							  ├─► Moyo GraphQL (replays input)
							  ├─► Comparator
							  └─► Parity DB (persist row)
```

Implementation lives in `src/Apps/LeadTimeMatrix/Tools/LeadTimeMatrix.Parity/` as a worker service:

```
LeadTimeMatrix.Parity/
  ├── Program.cs
  ├── Capture/                # consumes audit queue
  ├── Replay/                 # invokes Moyo GraphQL
  ├── Comparison/             # field-by-field compare + classification
  ├── Persistence/            # writes to ParityRun schema
  └── Reports/                # daily markdown + dashboard data
```

---

## Capture Side (Amtrack)

Amtrack writes a parity-event for every lead time calculation:

```json
{
  "captureId": "01HM...",
  "capturedAt": "2026-01-15T10:23:14Z",
  "callerSource": "Web|API|Internal",
  "input": {
	"orderId": "...",
	"productionLeadTimeInput": { /* full payload */ }
  },
  "amtrackResponse": {
	"slaDate": "2026-01-22T17:00:00Z",
	"leadTime": 5,
	"mode": "live"
  },
  "amtrackBreakdown": { /* legacy breakdown if available */ }
}
```

The event is published to a durable queue (Azure Service Bus / RabbitMQ — reuse whichever Moyo already uses for cross-system audit). The capture is **fire-and-forget**: queue failures must not block the live response.

---

## Replay Worker (Moyo)

The worker consumes the queue and, for each event:

1. Calls Moyo's **internal** GraphQL endpoint with `useStaged: false, includeBreakdown: true`.
2. Receives Moyo's response.
3. Hands both responses to the Comparator.
4. Persists the result regardless of pass/fail.

The worker is throttled (default 50 RPS) so the parity load doesn't compete with normal traffic on Moyo. Throttle is configurable via `Parity:MaxRps`.

The worker uses the same circuit-breaker policy as [amtrack-integration-changes.md](amtrack-integration-changes.md) — if Moyo is down, parity events queue up (durable) and are processed when Moyo recovers.

---

## Comparison Rules

For each captured event, compute a `ParityResult`:

| Outcome | Rule |
|---------|------|
| `Match` | `slaDate` identical AND `leadTime` identical |
| `WithinTolerance` | `slaDate` differs by ≤ 0 days AND `leadTime` differs by ≤ 0 days (i.e. only when tolerance > 0; default tolerance is **0** — strict match) |
| `Mismatch_LeadTime` | `leadTime` values differ |
| `Mismatch_SlaDate` | `slaDate` values differ |
| `Mismatch_Both` | both fields differ |
| `Error_Moyo` | Moyo returned an error or timed out |
| `Error_Amtrack` | Amtrack response missing in capture event |

Tolerance is configurable but starts at **0** — only true matches count toward the green-light criterion.

### Field-by-field diff

When fields differ, the persisted row includes the per-field delta:

```json
{
  "deltas": [
	{ "field": "leadTime", "amtrack": 5, "moyo": 6 },
	{ "field": "slaDate", "amtrack": "2026-01-22T17:00:00Z", "moyo": "2026-01-23T17:00:00Z" }
  ]
}
```

When the Moyo response includes a breakdown (it does — `includeBreakdown: true`), persist the breakdown alongside the deltas so analysts can root-cause mismatches without re-running.

---

## Parity DB Schema

A dedicated schema `parity` (or a separate database — operator's preference per environment):

| Table | Purpose |
|-------|---------|
| `parity.Capture` | One row per captured Amtrack call (input + Amtrack response) |
| `parity.Replay` | One row per Moyo replay (Moyo response + breakdown) |
| `parity.Comparison` | One row per comparison: outcome, deltas, references to Capture and Replay |
| `parity.DailySummary` | Aggregated daily counts per outcome, computed by the daily report job |

The schema is intentionally separate from the LeadTimeMatrix application schema so it can be dropped after sign-off without affecting production.

---

## Daily Report

A scheduled job runs at 06:00 each business day and produces `reports/parity-<yyyy-MM-dd>.md`:

- Total captures, replays, comparisons
- Outcome breakdown (counts and percentages)
- Top 10 mismatch patterns (group by `deltas` shape)
- Top callers contributing to mismatches (group by `callerSource`)
- Per-`OrderType` mismatch rate
- A **traffic-light** verdict:
  - **Green** — `Match` rate ≥ 99.5% over the trailing 24h
  - **Amber** — `Match` rate 98.0% – 99.5%
  - **Red** — `Match` rate < 98.0% OR `Error_Moyo` rate > 0.5%

The daily report is published to Signoz (per [observability-admin-audit-logs.md](observability-admin-audit-logs.md)) AND committed to a `reports/` git folder for sign-off paper trail.

### Sign-off Criterion

Cutover proceeds only when **5 consecutive business days are Green**. Any Amber or Red day resets the 5-day counter.

---

## Operator Workflow

1. Day -7: Deploy parity harness alongside Moyo. Both engines run.
2. Day -5 to Day 0 (cutover): Daily reports reviewed at 09:00 standup.
3. On any Red/Amber day: investigate via the persisted `parity.Comparison` rows; raise fixes in Moyo or in the source data.
4. Day 0: After 5 consecutive Green days, flip cutover. Parity capture continues for 1 more week as a safety net.
5. Day +7: Decommission parity harness; retain the `parity` schema for 30 days then drop.

---

## Observability

Parity worker emits Signoz logs (per [observability-admin-audit-logs.md](observability-admin-audit-logs.md)):

- `Operation` = `ParityReplay`, `ParityCompare`
- `Outcome` = one of the comparison outcomes above
- `LatencyMoyoMs`, `LatencyAmtrackMs`
- `QueueDepth`, `BackpressureActive` from the worker host

Add a Signoz panel to the LeadTimeMatrix dashboard showing live parity match rate.

---

## Acceptance Criteria

- [ ] `LeadTimeMatrix.Parity` worker project exists under `src/Apps/LeadTimeMatrix/Tools/`
- [ ] Amtrack publishes a parity event for **every** lead time calculation it serves; queue failures are silent and never block the live response
- [ ] The Moyo worker consumes the queue, replays each input through `/api/v1/leadtimematrix/graphql/internal` with `includeBreakdown: true`, and persists the result
- [ ] Throttling (default 50 RPS, configurable) protects Moyo from parity-induced load spikes
- [ ] Comparison rules produce one of: `Match`, `Mismatch_LeadTime`, `Mismatch_SlaDate`, `Mismatch_Both`, `Error_Moyo`, `Error_Amtrack`
- [ ] Per-field deltas and the Moyo breakdown are persisted on every row (not just mismatches) so analysts can root-cause without re-running
- [ ] Schema lives in a dedicated `parity` schema (or separate DB) that can be dropped without affecting production
- [ ] Daily report `reports/parity-<yyyy-MM-dd>.md` is generated at 06:00 and includes outcome counts, top mismatch patterns, per-`OrderType` rates, and a Green/Amber/Red verdict
- [ ] Sign-off criterion: 5 consecutive Green business days before cutover; any non-Green day resets the counter
- [ ] Signoz dashboard panel shows live parity match rate
- [ ] Functional tests cover: Match, every Mismatch outcome, Error_Moyo path (timeout), throttling, idempotent re-processing of the same capture event
- [ ] A runbook (`docs/parity-runbook.md`) documents the operator workflow and the response playbook for Red/Amber days

---

## Out of Scope

- The actual cutover toggle / traffic routing — delivered by the **rollback feature toggle** story
- Modifying Amtrack's response logic — Amtrack continues to serve traffic verbatim during the parity window
- Changes to the Moyo calculation engine itself — those are delivered by stories 3.3 / 3.4
- Long-term retention of parity data — the schema is dropped 30 days post-cutover; analysts can export needed data within that window
