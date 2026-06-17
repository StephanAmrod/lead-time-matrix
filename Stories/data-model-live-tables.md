# Story: Lead Time Matrix - Create Live (Green) Domain Entities & Database Tables

**BRD References:** Data Model (TechnicalDesign/07)
**Dev Hours:** 16

---

## Overview

Create the live (Green) domain entities and corresponding database tables for the Lead Time Matrix module. The Green tables are the authoritative source of truth for **all real-time lead time calculations** (warehouse, production, special dates, branding hierarchy) performed by both internal and external GraphQL callers.

Follow the established Clean Architecture pattern observed in the **StockChecking** module:

- **Domain layer** - Entity classes in `LeadTimeMatrix.Domain/Entities/`, inheriting `BaseAuditableEntity`
- **Infrastructure layer** - EF Core `IEntityTypeConfiguration<T>` classes in `LeadTimeMatrix.Infrastructure/Data/Configurations/`
- **DbContext** - Apply all configurations through `LeadTimeMatrixDbContext` (auto-discovered via `ApplyConfigurationsFromAssembly`)
- **Migrations** - Generate the initial EF Core migration in `LeadTimeMatrix.Infrastructure/Data/Migrations/`

See [TechnicalDesign/07-proposed-architecture-artefacts.md](../TechnicalDesign/07-proposed-architecture-artefacts.md) for the authoritative SQL schema definition.

> **Out of scope:** The Update (Blue) staging tables are delivered by the companion story [data-model-update-tables.md](data-model-update-tables.md). This story delivers the Green schema only.

---

## Entities to Create

All entities must inherit from `BaseAuditableEntity` and reside in the `Moyo.Apps.LeadTimeMatrix.Domain.Entities` namespace.

### Branding / Production Entities

| Entity | Purpose |
|--------|---------|
| `BrandingDepartment` | Top-level grouping for production branding rules |
| `PrintCode` | Print code grouped under a branding department |
| `SetupCode` | Setup code variant under a print code (colour + quantity range) |
| `InvoiceCode` | Invoice code variant under a print code |
| `BrandingGroup` | Group of products under a branding department |
| `QuantityBreak` | Quantity range and lead-time entry under a branding group |
| `BrandingByItem` | Per-item adjustment/override under a branding group |
| `PersonalisationRule` | Personalisation quantity range and lead time per branding department |

### Special Dates Entities

| Entity | Purpose |
|--------|---------|
| `SpecialDateCategory` | Category for special dates (e.g. public holiday, shutdown) |
| `SpecialDate` | A non-working date with name and category |
| `SpecialDateImpact` | Links a special date to the branding departments it affects |

### Warehouse Entities

| Entity | Purpose |
|--------|---------|
| `OrderType` | Predefined warehouse order type (Samples, Red Slips, etc.) |
| `WarehouseGroup` | Group of products under an order type |
| `WarehouseQuantityBreak` | Quantity range and lead-time entry under a warehouse group |

---

## Property Definitions

Each entity's properties must match the SQL column definitions in [TechnicalDesign/07-proposed-architecture-artefacts.md](../TechnicalDesign/07-proposed-architecture-artefacts.md). Use C# `required` for non-nullable scalar properties consistent with `StockQuote` in the StockChecking module.

### Reference Pattern - `StockQuote` (from StockChecking module)

```csharp
namespace Moyo.Apps.StockChecking.Domain.Entities;

/// <summary>
/// Represents a Stock Quote.
/// </summary>
public class StockQuote : BaseAuditableEntity
{
	public required string Number { get; set; }
	public required string Customer { get; set; }
	public required string CustomerName { get; set; }
	public ICollection<StockQuoteContact> StockQuoteContacts { get; set; } = [];
	public ICollection<StockQuoteItem> StockQuoteItems { get; set; } = [];
}
```

### Example - `BrandingDepartment.cs`

```csharp
namespace Moyo.Apps.LeadTimeMatrix.Domain.Entities;

/// <summary>
/// Top-level grouping for production branding rules.
/// </summary>
public class BrandingDepartment : BaseAuditableEntity
{
	public required string Name { get; set; }
	public required int AdjustmentDays { get; set; }
	public required string AttributeName { get; set; }
	public required int OverrideDays { get; set; }

	public ICollection<PrintCode> PrintCodes { get; set; } = [];
	public ICollection<BrandingGroup> BrandingGroups { get; set; } = [];
	public ICollection<PersonalisationRule> PersonalisationRules { get; set; } = [];
	public ICollection<SpecialDateImpact> SpecialDateImpacts { get; set; } = [];
	public ICollection<WarehouseQuantityBreak> WarehouseQuantityBreaks { get; set; } = [];
}
```

> The `Id` (PK) is inherited from `BaseAuditableEntity`. The configuration class maps `Id` to the column name from the data model (e.g. `DepartmentId`) where required for downstream Flowgear / external SQL compatibility — see EF Core Configurations below.

---

## EF Core Configurations

For each entity, create a corresponding `IEntityTypeConfiguration<TEntity>` class in `LeadTimeMatrix.Infrastructure/Data/Configurations/`. Each configuration must:

1. Map the entity to the correct table name via `builder.ToTable("...")`
2. Map `Id` to the column name from the data model where it differs (e.g. `builder.Property(e => e.Id).HasColumnName("DepartmentId");`)
3. Apply `HasMaxLength` constraints matching the SQL definitions (e.g. `Name VARCHAR(100)` → `.HasMaxLength(100)`)
4. Configure foreign key relationships explicitly via `HasOne`/`WithMany`/`HasForeignKey`
5. Apply the standard audit field defaults from the StockChecking pattern:
   - `builder.Property(e => e.CreatedOn).HasDefaultValueSql("(sysdatetimeoffset())");`
   - `builder.Property(e => e.LastModifiedOn).HasDefaultValueSql("(sysdatetimeoffset())");`
   - `builder.Property(e => e.CreatedBy).IsRequired();`
   - `builder.Property(e => e.LastModifiedBy).IsRequired();`
6. Apply unique constraints where required (e.g. `SpecialDate.Date` is `UNIQUE`)

### Reference Pattern - `StockQuoteConfiguration` (from StockChecking module)

```csharp
namespace Moyo.Apps.StockChecking.Infrastructure.Data.Configurations;

/// <summary>
/// Configuration for the <see cref="StockQuote"/> entity.
/// </summary>
public class StockQuoteConfiguration : IEntityTypeConfiguration<StockQuote>
{
	public void Configure(EntityTypeBuilder<StockQuote> builder)
	{
		builder.ToTable("StockQuote");

		builder.Property(e => e.Number).HasMaxLength(50);
		builder.Property(e => e.Customer).HasMaxLength(50);
		builder.Property(e => e.CustomerName).HasMaxLength(50);

		builder.Property(e => e.CreatedOn).HasDefaultValueSql("(sysdatetimeoffset())");
		builder.Property(e => e.LastModifiedOn).HasDefaultValueSql("(sysdatetimeoffset())");
		builder.Property(e => e.CreatedBy).IsRequired();
		builder.Property(e => e.LastModifiedBy).IsRequired();
	}
}
```

### Example - `BrandingDepartmentConfiguration.cs`

```csharp
namespace Moyo.Apps.LeadTimeMatrix.Infrastructure.Data.Configurations;

/// <summary>
/// Configuration for the <see cref="BrandingDepartment"/> entity.
/// </summary>
public class BrandingDepartmentConfiguration : IEntityTypeConfiguration<BrandingDepartment>
{
	public void Configure(EntityTypeBuilder<BrandingDepartment> builder)
	{
		builder.ToTable("BrandingDepartment");

		builder.Property(e => e.Id).HasColumnName("DepartmentId");
		builder.Property(e => e.Name).HasMaxLength(100).IsRequired();
		builder.Property(e => e.AttributeName).HasMaxLength(100).IsRequired();

		builder.Property(e => e.CreatedOn).HasDefaultValueSql("(sysdatetimeoffset())");
		builder.Property(e => e.LastModifiedOn).HasDefaultValueSql("(sysdatetimeoffset())");
		builder.Property(e => e.CreatedBy).IsRequired();
		builder.Property(e => e.LastModifiedBy).IsRequired();
	}
}
```

---

## Foreign Key Relationships

The following relationships must be configured via Fluent API on the parent (principal) side using `HasMany(...).WithOne(...).HasForeignKey(...)`:

| Child Entity | FK Property | Parent Entity | Notes |
|--------------|-------------|---------------|-------|
| `PrintCode` | `DepartmentId` | `BrandingDepartment` | |
| `SetupCode` | `PrintCodeId` | `PrintCode` | |
| `InvoiceCode` | `PrintCodeId` | `PrintCode` | |
| `BrandingGroup` | `DepartmentId` | `BrandingDepartment` | |
| `QuantityBreak` | `GroupId` | `BrandingGroup` | |
| `BrandingByItem` | `BrandingGroupId` | `BrandingGroup` | |
| `SpecialDate` | `CategoryId` | `SpecialDateCategory` | |
| `SpecialDateImpact` | `SpecialDateId` | `SpecialDate` | |
| `SpecialDateImpact` | `DepartmentId` | `BrandingDepartment` | |
| `WarehouseGroup` | `OrderTypeId` | `OrderType` | |
| `WarehouseQuantityBreak` | `WarehouseGroupId` | `WarehouseGroup` | |
| `WarehouseQuantityBreak` | `DepartmentId` | `BrandingDepartment` | |
| `PersonalisationRule` | `DepartmentId` | `BrandingDepartment` | |

Cascade-delete behaviour must be reviewed per relationship and explicitly set (default `Restrict` is recommended) to avoid accidental cascade through the hierarchy.

---

## Unique Constraints

| Entity | Column(s) | Source |
|--------|-----------|--------|
| `SpecialDate` | `Date` | SQL `Date DATE NOT NULL UNIQUE` |

Apply via `builder.HasIndex(e => e.Date).IsUnique();`.

---

## DbContext Registration

In `LeadTimeMatrix.Infrastructure/Data/LeadTimeMatrixDbContext.cs`:

1. Expose a `DbSet<T>` for each of the 14 entities
2. In `OnModelCreating`, call:
   ```csharp
   modelBuilder.ApplyConfigurationsFromAssembly(typeof(LeadTimeMatrixDbContext).Assembly);
   ```
3. Keep the design-time `LeadTimeMatrixDbContextFactory` consistent with the StockChecking `StockCheckingDbContextFactory` pattern

---

## Initial Migration

Generate the initial EF Core migration:

```powershell
dotnet ef migrations add InitialLiveTables `
  --project src/Apps/LeadTimeMatrix/LeadTimeMatrix.Infrastructure `
  --startup-project src/Apps/LeadTimeMatrix/LeadTimeMatrix.Api `
  --context LeadTimeMatrixDbContext
```

Verify the generated migration:

- Creates all 14 live tables (`BrandingDepartment`, `PrintCode`, `SetupCode`, `InvoiceCode`, `BrandingGroup`, `QuantityBreak`, `BrandingByItem`, `SpecialDateCategory`, `SpecialDate`, `SpecialDateImpact`, `OrderType`, `WarehouseGroup`, `WarehouseQuantityBreak`, `PersonalisationRule`)
- Creates all FK constraints exactly as defined in TechnicalDesign/07 section *Relationships*
- Creates the unique constraint on `SpecialDate.Date`
- Adds the standard audit columns (`CreatedOn`, `CreatedBy`, `LastModifiedOn`, `LastModifiedBy`) on every table

Commit the migration files under `LeadTimeMatrix.Infrastructure/Data/Migrations/`.

---

## Reference Data Seeding

In `LeadTimeMatrix.Infrastructure/Data/ApplicationDbContextInitialiser.cs` (following the StockChecking initialiser pattern):

1. Seed the predefined `OrderType` records required by **BRD 2.3.3**:
   - Samples
   - Red Slips
   - Replacements – Unbranded
   - Console – Branded
   - Console – Unbranded
   - Unbranded Orders
   - Branches
2. Seed initial `SpecialDateCategory` records:
   - Public Holiday
   - Shutdown
3. Add a `DataGenerators/` folder following the StockChecking layout. Implementing development sample data is **optional** for this story but the folder must exist for future stories to extend.

Seeding must be idempotent — re-running the initialiser must not duplicate or alter existing records.

---

## Acceptance Criteria

- [ ] All 14 live entity classes exist in `LeadTimeMatrix.Domain/Entities/` and inherit `BaseAuditableEntity`
- [ ] All 14 EF Core configuration classes exist in `LeadTimeMatrix.Infrastructure/Data/Configurations/`
- [ ] `LeadTimeMatrixDbContext` exposes a `DbSet<T>` for every entity and applies all configurations via `ApplyConfigurationsFromAssembly`
- [ ] An initial migration `InitialLiveTables` creates the 14 live tables, FK constraints, and the unique constraint on `SpecialDate.Date` exactly matching the SQL in TechnicalDesign/07
- [ ] `ApplicationDbContextInitialiser` seeds the 7 predefined `OrderType` records and the initial `SpecialDateCategory` records idempotently
- [ ] Every configuration applies the audit-field defaults consistent with `StockQuoteConfiguration`
- [ ] `dotnet build` succeeds for the full solution
- [ ] `dotnet test` succeeds for `LeadTimeMatrix.FunctionalTests` (a smoke test that resolves the `DbContext` and applies the migration is sufficient for this story)

---

## Out of Scope

- **Update (Blue) staging tables** — delivered by [data-model-update-tables.md](data-model-update-tables.md)
- **API endpoints / GraphQL queries / CRUD logic** — delivered by the per-feature stories (2.1.x, 2.2.x, 2.3.x)
- **Flowgear sync flows** — delivered by separate Flowgear stories
- **Permissions** — delivered by [4.1-permissions.md](4.1-permissions.md)
