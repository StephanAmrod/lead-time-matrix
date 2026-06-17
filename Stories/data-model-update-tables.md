# Story: Lead Time Matrix - Create Update (Blue) Staging Entities & Database Tables

**BRD References:** Data Model (TechnicalDesign/07)
**Dev Hours:** 8

---

## Overview

Create the update (Blue) staging entities and corresponding database tables for the Lead Time Matrix module. The Blue tables stage **production** lead time changes throughout the day; Flowgear later syncs them into the live (Green) tables overnight.

> **Warehouse changes are not staged.** Per TechnicalDesign/07, warehouse lead times update immediately on the live tables, so there is **no** Blue counterpart for the warehouse entities.

Follow the same Clean Architecture pattern used by the companion story [data-model-live-tables.md](data-model-live-tables.md):

- **Domain layer** - Entity classes in `LeadTimeMatrix.Domain/Entities/` (recommended subfolder `Updates/`), inheriting `BaseAuditableEntity`
- **Infrastructure layer** - EF Core `IEntityTypeConfiguration<T>` classes in `LeadTimeMatrix.Infrastructure/Data/Configurations/` (recommended subfolder `Updates/`)
- **DbContext** - Apply all configurations through `LeadTimeMatrixDbContext` (auto-discovered via `ApplyConfigurationsFromAssembly`)
- **Migrations** - Generate a follow-up migration in `LeadTimeMatrix.Infrastructure/Data/Migrations/`

See [TechnicalDesign/07-proposed-architecture-artefacts.md](../TechnicalDesign/07-proposed-architecture-artefacts.md) for the authoritative SQL schema definition and the [Flowgear Sync Updates](../TechnicalDesign/07-proposed-architecture-artefacts.md#flowgear-sync-updates) section for the sync lifecycle.

---

## Entities to Create

All entities must inherit from `BaseAuditableEntity` and reside in the `Moyo.Apps.LeadTimeMatrix.Domain.Entities` namespace.

| Update Entity | Mirrors Live Entity | Purpose |
|---------------|---------------------|---------|
| `BrandingDepartmentUpdate` | `BrandingDepartment` | Staged change to a branding department |
| `PrintCodeUpdate` | `PrintCode` | Staged change to a print code |
| `SetupCodeUpdate` | `SetupCode` | Staged change to a setup code |
| `InvoiceCodeUpdate` | `InvoiceCode` | Staged change to an invoice code |
| `BrandingGroupUpdate` | `BrandingGroup` | Staged change to a branding group |
| `QuantityBreakUpdate` | `QuantityBreak` | Staged change to a quantity break |
| `BrandingByItemUpdate` | `BrandingByItem` | Staged change to a branding-by-item record |

> **Append-only.** Records in the update tables are never deleted — they are retained for audit, history, and future modules/processes (per TechnicalDesign/07 *Audit Trail*).

---

## Common Fields

Every update entity must include the following on top of the audit fields inherited from `BaseAuditableEntity`:

| Property | C# Type | SQL Column | Purpose |
|----------|---------|------------|---------|
| `Id` (inherited) | `int` | `RecordId` (PK) | Unique id of the staged update record |
| `<LiveEntityId>` | `int` | `<LiveEntityId>` | Id of the corresponding live record (e.g. `DepartmentId`, `PrintCodeId`). Populated when staging an update to an existing live record. |
| ... payload fields ... | matches live entity | matches live entity | All editable fields copied from the corresponding live entity (see TechnicalDesign/07 SQL) |
| `Status` | `UpdateRecordStatus` | `Status VARCHAR(50)` | Lifecycle state — see enum below |
| `CreatedOn` / `CreatedBy` (inherited) | | | When the change was staged and by whom |
| `LastModifiedOn` / `LastModifiedBy` (inherited) | | | Updated by Flowgear when the row transitions to `Synced` |

### Status Enum

Create `UpdateRecordStatus` in `LeadTimeMatrix.Domain/Enums/`:

```csharp
namespace Moyo.Apps.LeadTimeMatrix.Domain.Enums;

public enum UpdateRecordStatus
{
	New = 0,
	Synced = 1
}
```

Lifecycle (per TechnicalDesign/07 *Update States & Lifecycle*):

- **`New`** — record has been added by a user and is awaiting overnight Flowgear sync
- **`Synced`** — Flowgear has applied the change to the live table; `LastModifiedOn` is set to the sync time

---

## Example - `BrandingDepartmentUpdate.cs`

```csharp
namespace Moyo.Apps.LeadTimeMatrix.Domain.Entities;

/// <summary>
/// Staged change to a <see cref="BrandingDepartment"/>.
/// Synced to the live table by Flowgear on a daily nightly schedule.
/// </summary>
public class BrandingDepartmentUpdate : BaseAuditableEntity
{
	public required int DepartmentId { get; set; }
	public required string Name { get; set; }
	public required int AdjustmentDays { get; set; }
	public required string AttributeName { get; set; }
	public required int OverrideDays { get; set; }
	public required UpdateRecordStatus Status { get; set; }
}
```

---

## EF Core Configurations

Follow the same pattern as the live configurations, with these additions specific to the Blue tables:

1. Map `Id` to SQL column `RecordId`:
   ```csharp
   builder.Property(e => e.Id).HasColumnName("RecordId");
   ```
2. Map the `<LiveEntityId>` property to a regular column — **do NOT** configure a foreign key relationship to the live entity. A staged record may legitimately reference an as-yet-unsynced parent (or pre-date the existence of a live record), and adding an FK would block valid staging scenarios.
3. Persist `Status` as a string for readability in the database (consistent with the SQL `VARCHAR(50)` definition in TechnicalDesign/07):
   ```csharp
   builder.Property(e => e.Status)
	   .HasConversion<string>()
	   .HasMaxLength(50)
	   .IsRequired();
   ```
4. Apply the standard audit defaults consistent with the live configurations:
   - `builder.Property(e => e.CreatedOn).HasDefaultValueSql("(sysdatetimeoffset())");`
   - `builder.Property(e => e.LastModifiedOn).HasDefaultValueSql("(sysdatetimeoffset())");`
   - `builder.Property(e => e.CreatedBy).IsRequired();`
   - `builder.Property(e => e.LastModifiedBy).IsRequired();`
5. Apply `HasMaxLength` constraints matching the SQL definitions in TechnicalDesign/07

### Example - `BrandingDepartmentUpdateConfiguration.cs`

```csharp
namespace Moyo.Apps.LeadTimeMatrix.Infrastructure.Data.Configurations;

/// <summary>
/// Configuration for the <see cref="BrandingDepartmentUpdate"/> entity.
/// </summary>
public class BrandingDepartmentUpdateConfiguration : IEntityTypeConfiguration<BrandingDepartmentUpdate>
{
	public void Configure(EntityTypeBuilder<BrandingDepartmentUpdate> builder)
	{
		builder.ToTable("BrandingDepartmentUpdate");

		builder.Property(e => e.Id).HasColumnName("RecordId");
		builder.Property(e => e.Name).HasMaxLength(100).IsRequired();
		builder.Property(e => e.AttributeName).HasMaxLength(100).IsRequired();

		builder.Property(e => e.Status)
			.HasConversion<string>()
			.HasMaxLength(50)
			.IsRequired();

		builder.Property(e => e.CreatedOn).HasDefaultValueSql("(sysdatetimeoffset())");
		builder.Property(e => e.LastModifiedOn).HasDefaultValueSql("(sysdatetimeoffset())");
		builder.Property(e => e.CreatedBy).IsRequired();
		builder.Property(e => e.LastModifiedBy).IsRequired();
	}
}
```

---

## Indexing Recommendations

For Flowgear sync performance, add a non-unique index on `Status` for each update table (Flowgear queries `WHERE Status = 'New'` daily):

```csharp
builder.HasIndex(e => e.Status);
```

---

## DbContext Registration

In `LeadTimeMatrix.Infrastructure/Data/LeadTimeMatrixDbContext.cs`:

1. Expose a `DbSet<T>` for each of the 7 update entities
2. `ApplyConfigurationsFromAssembly` will auto-discover the new configurations — no additional `OnModelCreating` changes are required

---

## Migration

Generate the follow-up EF Core migration (after the `InitialLiveTables` migration from the companion story has been applied):

```powershell
dotnet ef migrations add InitialUpdateTables `
  --project src/Apps/LeadTimeMatrix/LeadTimeMatrix.Infrastructure `
  --startup-project src/Apps/LeadTimeMatrix/LeadTimeMatrix.Api `
  --context LeadTimeMatrixDbContext
```

Verify the generated migration:

- Creates the 7 update tables (`BrandingDepartmentUpdate`, `PrintCodeUpdate`, `SetupCodeUpdate`, `InvoiceCodeUpdate`, `BrandingGroupUpdate`, `QuantityBreakUpdate`, `BrandingByItemUpdate`)
- Uses `RecordId` as the PK column on every table
- Includes the `Status VARCHAR(50)` column on every table
- Includes an index on `Status` for every table
- Does **NOT** create FK constraints to the live tables (by design)
- Adds the standard audit columns (`CreatedOn`, `CreatedBy`, `LastModifiedOn`, `LastModifiedBy`) on every table

Commit the migration files under `LeadTimeMatrix.Infrastructure/Data/Migrations/`.

---

## Acceptance Criteria

- [ ] All 7 update entity classes exist in `LeadTimeMatrix.Domain/Entities/` (or `Updates/` subfolder) and inherit `BaseAuditableEntity`
- [ ] `UpdateRecordStatus` enum exists in `LeadTimeMatrix.Domain/Enums/` with values `New` and `Synced`
- [ ] All 7 EF Core configuration classes exist in `LeadTimeMatrix.Infrastructure/Data/Configurations/` (or `Updates/` subfolder)
- [ ] `LeadTimeMatrixDbContext` exposes a `DbSet<T>` for every update entity
- [ ] A migration `InitialUpdateTables` creates the 7 update tables with `RecordId` as PK and `Status` stored as `VARCHAR(50)`, matching the SQL in TechnicalDesign/07
- [ ] A non-unique index on `Status` exists for every update table
- [ ] **No** foreign key constraints exist between the update tables and the live tables (per design rationale above)
- [ ] Every configuration applies the audit-field defaults consistent with the live configurations
- [ ] `dotnet build` succeeds for the full solution
- [ ] `dotnet test` succeeds for `LeadTimeMatrix.FunctionalTests` (a smoke test that resolves the `DbContext` and inserts/reads a `BrandingDepartmentUpdate` row with `Status = New` is sufficient for this story)

---

## Out of Scope

- **Live (Green) tables** — delivered by the companion story [data-model-live-tables.md](data-model-live-tables.md)
- **Flowgear daily sync flow** — separate Flowgear story; this story only delivers the schema
- **CRUD endpoints that write to the update tables** — delivered by the per-feature stories (2.2.x, 2.3.x)
- **Promotion/approval workflow beyond `New` → `Synced`** — additional statuses (e.g. `Pending`, `Approved`) noted in TechnicalDesign/07 are not part of the daily Flowgear sync described in *Update States & Lifecycle* and are out of scope for this story
- **Permissions** — delivered by [4.1-permissions.md](4.1-permissions.md)
