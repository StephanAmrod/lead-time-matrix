# Story: Lead Time Matrix - Data Migration ETL from Amtrack legacy schema

**TD Deployment & Rollback Strategy : Data Migration**

**BRD References:** N/A (deployment-driven)
**Tech Design References:** TechnicalDesign/12 (Data Migration)
**Dev Hours:** 16

---

## Overview

Per TechnicalDesign/12: *"Moyo runs an initial migration to parse and migrate Amtrack legacy data into the Moyo schema."*

Build the **one-shot ETL** that brings the existing Amtrack lead time configuration into Moyo. The ETL must:

1. **Extract** the existing configuration from the Amtrack legacy schema.
2. **Transform** the legacy shape to the Moyo Live (Green) and Update (Blue) tables defined by [data-model-live-tables.md](data-model-live-tables.md) and [data-model-update-tables.md](data-model-update-tables.md).
3. **Load** the Moyo tables in the correct dependency order.
4. Produce a **pre-flight reconciliation report** so operators can review row counts and key spot-checks before flipping the switch.
5. Be **idempotent** — running the ETL a second time against an already-loaded Moyo schema must converge on the same state without duplication.

---

## Sources to Migrate

| Domain | Source (Amtrack) | Target (Moyo Green) |
|--------|------------------|---------------------|
| Branding departments | Amtrack `BrandingDept` (or equivalent) | `BrandingDepartment` |
| Print codes | Amtrack `PrintCode` | `PrintCode` |
| Setup codes | Amtrack `SetupCode` | `SetupCode` |
| Invoice codes | Amtrack `InvoiceCode` | `InvoiceCode` |
| Branding groups | Amtrack `BrandingGroup` | `BrandingGroup` |
| Branding by item | Amtrack `BrandingByItem` | `BrandingByItem` |
| Quantity breaks | Amtrack `QuantityBreak` (production-side) | `QuantityBreak` |
| Order types | Amtrack `OrderType` | `OrderType` (plus seed list) |
| Warehouse groups | Amtrack `WarehouseGroup` | `BrandingGroup` (warehouse hierarchy under each `OrderType`) |
| Warehouse quantity breaks | Amtrack `WarehouseQuantityBreak` | `QuantityBreak` |
| Special dates | Amtrack `SpecialDate` (and any "shutdown" tables) | `SpecialDate` (+ `SpecialDateCategory`) |
| Special date categories | Implicit in Amtrack — derive from distinct categories | `SpecialDateCategory` |

> **Note:** The exact source-table names need confirmation from the Amtrack DBAs during implementation. The mapping above is the canonical target list; the source side will be confirmed in the kickoff workshop.

---

## ETL Architecture

Implement as a **standalone .NET console / worker project** under `src/Apps/LeadTimeMatrix/Tools/`:

```
src/Apps/LeadTimeMatrix/Tools/LeadTimeMatrix.Migration/
  ├── Program.cs                    # CLI entry, argument parsing
  ├── Pipeline/                     # ordered phase classes
  │   ├── 01-PreFlight.cs
  │   ├── 02-Extract.cs
  │   ├── 03-Transform.cs
  │   ├── 04-Load.cs
  │   └── 05-Reconcile.cs
  ├── Mappings/                     # per-entity mappers
  └── Reports/                      # markdown / CSV report writers
```

CLI invocation:

```
dotnet run --project LeadTimeMatrix.Migration -- \
  --source "Server=...;Database=Amtrack;..." \
  --target "Server=...;Database=Moyo;..." \
  --mode {dryRun|apply} \
  --report ./reports
```

Modes:

- **`dryRun`** — Extract + Transform + Reconcile-without-write. Produces the full report so operators can sign-off before applying.
- **`apply`** — Extract + Transform + Load + Reconcile. Persists the result.

---

## Pipeline Phases

### 1. Pre-flight checks

- Both connection strings reachable
- Target Moyo schema is at the latest EF Core migration
- Source Amtrack DB is reachable in **read-only** mode (the ETL never writes to Amtrack)
- Required source tables exist; required source columns present

### 2. Extract

For each source entity, stream rows in batches (default 5,000) using ADO.NET / Dapper. **No** ORM mapping at this stage — capture rows as `dynamic` / dictionaries. This avoids type drift between the two schemas.

### 3. Transform

For each entity, run the per-entity mapper that:

- Validates required fields present and non-null
- Coerces types (e.g. legacy `varchar` enum → Moyo enum)
- Resolves FKs to Moyo IDs (using lookup dictionaries built from earlier-mapped entities)
- Records any rows that fail validation in a per-entity reject CSV (`reports/rejects/<entity>.csv`)

### 4. Load

Load order (dependency-respecting):

1. `BrandingDepartment` (no FKs)
2. `OrderType` (no FKs)
3. `SpecialDateCategory` (no FKs)
4. `PrintCode` (FK → `BrandingDepartment`)
5. `BrandingGroup` (FK → `BrandingDepartment` and to `OrderType`)
6. `SetupCode` (FK → `PrintCode`)
7. `InvoiceCode` (FK → `PrintCode`)
8. `BrandingByItem` (FK → `BrandingGroup`)
9. `QuantityBreak` (FK → `BrandingGroup`)
10. `SpecialDate` (FK → `SpecialDateCategory`)

Each phase loads in batches via `EFCore.BulkExtensions` `BulkInsertOrUpdate` so re-runs converge.

> **Idempotency strategy:** every entity carries a `LegacyId` shadow column populated from the Amtrack PK. `BulkInsertOrUpdate` matches on `LegacyId`; a re-run that finds the same `LegacyId` updates rather than duplicates.

The Update (Blue) tables are **NOT** populated by the ETL — they are working state for in-flight admin changes. The ETL writes the live (Green) state only.

### 5. Reconcile

- Compare row counts: Source vs Moyo target per entity.
- Spot-check N=10 random rows per entity for field-level equivalence.
- Generate `reports/reconciliation.md` with red/amber/green status per entity.
- Generate `reports/rejected-rows.md` summarising rejects from the Transform phase.

The CLI exits non-zero if any entity is **red** (count mismatch beyond a tolerance per entity, or critical mapper rejects).

---

## Reports

`reports/` folder contains:

- `summary.md` — top-level go/no-go summary
- `reconciliation.md` — per-entity row counts, hashes, spot-checks
- `rejects/<entity>.csv` — rows that failed transformation, with reason
- `mapping/<entity>.csv` — source-to-target ID mapping for traceability

These reports are committed into a dated folder (`reports/2026-01-15-1430/`) on every run so operators can compare runs over time.

---

## Run Order Around Cutover

1. Run ETL in **dryRun** mode against production source 7 days before cutover. Operators review reports.
2. Resolve mapper rejects (legacy data fixes in Amtrack OR mapper updates).
3. Re-run **dryRun** until reports are green.
4. Day before cutover: run **apply** during the maintenance window.
5. Run the **parity comparison** (see [parity-run-shadow-calculation.md](parity-run-shadow-calculation.md)) for 5 business days while traffic still flows through Amtrack.
6. Cutover.

If the parity run fails, the [rollback feature toggle](#) flips traffic back to Amtrack and the ETL is re-run after fixes.

---

## Acceptance Criteria

- [ ] `LeadTimeMatrix.Migration` console project exists under `src/Apps/LeadTimeMatrix/Tools/`
- [ ] CLI accepts `--source`, `--target`, `--mode {dryRun|apply}`, `--report` arguments
- [ ] Pre-flight checks reject the run if connection or schema validation fails
- [ ] Extract reads all source entities in 5,000-row batches and never writes to the source
- [ ] Transform produces a per-entity reject CSV with reason codes for invalid rows
- [ ] Load applies entities in the dependency-respecting order listed above
- [ ] Each entity is loaded via `BulkInsertOrUpdate` keyed on `LegacyId`, so a re-run is idempotent
- [ ] No data is written to the Update (Blue) tables by the ETL
- [ ] Reconciliation report produces red / amber / green status per entity with row counts and N=10 spot-check evidence
- [ ] CLI exits non-zero if any entity is **red**
- [ ] Reports are written to a timestamped sub-folder of `--report`
- [ ] `dryRun` mode produces the full report **without** writing to the target
- [ ] Functional tests cover: full happy-path migration, FK resolution, mapper reject capture, idempotent re-run (no duplicates), dryRun no-write guarantee
- [ ] A runbook (`docs/etl-runbook.md`) documents the cutover sequence above

---

## Out of Scope

- The actual cutover toggle / traffic routing — delivered by the **rollback feature toggle** story
- The parity comparison harness — delivered by [parity-run-shadow-calculation.md](parity-run-shadow-calculation.md)
- Migrating *order/jobcard* operational data — the ETL covers configuration data only; operational data continues to live in the originating system until that system is decommissioned
- Automated source-data fixes — the ETL surfaces rejects but humans drive the corrections in the source system
- Schema migrations on the Moyo side — those are handled by EF Core migrations from the data-model stories
