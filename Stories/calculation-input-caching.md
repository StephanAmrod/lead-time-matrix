# Story: Lead Time Matrix - Calculation Input Caching with Daily Invalidation

**TD Non-Functional Requirements : Performance**

**BRD References:** N/A (NFR-driven)
**Tech Design References:** TechnicalDesign/11 (Caching Strategy)
**Dev Hours:** 8

---

## Overview

Per TechnicalDesign/11: *"Cache calculation inputs. Clear the cache immediately on active matrix adjustments — this happens once a day at night."*

Implement a caching layer in front of the calculation services so:

1. **Repeat queries** within the same business day for the same input shape return cached results without re-running the calculation pipeline.
2. The cache is **fully invalidated** every night when the production-matrix Flowgear sync runs (the moment Blue → Green data flips).
3. **Warehouse changes** apply immediately, so warehouse-impacted cache entries must be invalidated synchronously when warehouse configuration changes (or warehouse calculations bypass the cache entirely - see decision below).

The cache lives in the Application layer in front of `IProductionLeadTimeCalculator` and `IOrderLeadTimeCalculator` (delivered by stories 3.3 and 3.4 respectively).

---

## Implementation Approach

Use the existing Moyo `IDistributedCache` (Redis-backed) - same pattern used by other Moyo modules. Wrap the calculator services with a **decorator** registered via DI:

```csharp
services.AddScoped<ProductionLeadTimeCalculator>();
services.AddScoped<IProductionLeadTimeCalculator>(sp =>
	new CachingProductionLeadTimeCalculator(
		sp.GetRequiredService<ProductionLeadTimeCalculator>(),
		sp.GetRequiredService<IDistributedCache>(),
		sp.GetRequiredService<ILogger<CachingProductionLeadTimeCalculator>>()));
```

Apply the same pattern for `IOrderLeadTimeCalculator`.

---

## Cache Key Design

Cache keys are computed from a **deterministic hash of the calculation input**, prefixed with the calculator type and the cache "epoch":

```
ltm:calc:<calculator>:<epoch>:<sha256(input)>
```

Where:
- `<calculator>` is `prod` or `order`
- `<epoch>` is the current cache epoch (see Invalidation below) - typically a date string `yyyyMMdd`
- `<sha256(input)>` is the hex SHA-256 of a canonical JSON serialisation of the calculation input (sorted keys, ISO-8601 timestamps, `null`-stripped)

> Embedding the epoch *in the key itself* means invalidation is a key-prefix flush rather than a per-key delete. It also gives us "free" historical retention if needed for debugging.

### Inputs participating in the hash

For `prod`:
- `orderId` (when supplied) **OR** the full `ProductionLeadTimeInput` payload
- `useStaged` flag (live and staged caches are independent)
- `includeBreakdown` flag (full and slim cached payloads are independent)

For `order`:
- `orderId` (when supplied) **OR** the full `OrderLeadTimeInput` payload
- The set of jobcards (for branded orders) - hashed by jobcard id + lock-snapshot version

---

## Cache Value

The cached value is the **fully formed response object** the calculator would return - NOT a raw matrix lookup. This means a cache hit short-circuits the entire calculation pipeline.

Time-to-live: end of current business day (effectively until the next nightly invalidation). Use `DistributedCacheEntryOptions.AbsoluteExpiration = NextMidnightUtc + 30min` to let entries linger past the sync window for safety.

---

## Invalidation

There are two invalidation triggers:

### 1. Nightly production sync (TechnicalDesign/07, /08)

The Flowgear sync runs at 02:00. As part of the **production-lead-time sync nightly story**, the sync handler must:

1. Apply Blue → Green updates (Flowgear's responsibility).
2. **Bump the cache epoch** to the new business day (e.g. `20260116`).
3. Optionally: issue a `FLUSHDB`-equivalent on the `ltm:calc:` prefix to free Redis memory promptly.

After the epoch bumps, all subsequent cache lookups miss and re-populate against the freshly synced Green data.

### 2. Warehouse changes (TechnicalDesign/07)

Warehouse changes apply **immediately** - they don't go through Blue staging. Two acceptable strategies:

| Option | Description | Complexity |
|--------|-------------|-----------|
| **A. Bypass** | Calls that involve a warehouse component skip the cache entirely | Low |
| **B. Targeted invalidate** | On warehouse write, evict all cache entries that include warehouse data | High - requires reverse-index |

**Recommended: Option A** - the calculation pipeline checks whether the input requires warehouse data (any unbranded path or any add-on involving Console/CMT). If so, the calculator decorator skips the cache. Document this decision in the code with a reference to TechnicalDesign/07.

---

## Cache Statistics

The decorator must emit per-call observability data per [observability-admin-audit-logs.md](observability-admin-audit-logs.md):

- `Operation` = `CalcCacheLookup`
- `Calculator` = `prod` | `order`
- `Outcome` = `Hit` | `Miss` | `Bypass`
- `LatencyMs`

Forward to Signoz; surface hit-rate on the module dashboard.

---

## Acceptance Criteria

- [ ] `CachingProductionLeadTimeCalculator` and `CachingOrderLeadTimeCalculator` decorators exist and are registered as the public `IProductionLeadTimeCalculator` / `IOrderLeadTimeCalculator` implementations
- [ ] The underlying non-cached calculators are still resolvable for tests
- [ ] Cache keys follow the `ltm:calc:<calculator>:<epoch>:<sha256(input)>` shape
- [ ] Canonical JSON serialisation is deterministic across runs (sorted keys, ISO-8601 timestamps, `null` stripped)
- [ ] `useStaged` and `includeBreakdown` flags are part of the cache key (live/staged and full/slim are independent)
- [ ] Repeat calls with identical input return the cached value without invoking the inner calculator
- [ ] The nightly Flowgear sync bumps the cache epoch; subsequent calls miss the previous-day cache
- [ ] Warehouse-impacting calculations **bypass** the cache (documented in code with TD reference)
- [ ] Per-call structured logs (`Hit` / `Miss` / `Bypass`) flow to Signoz
- [ ] Cache hit-rate is visible on the LeadTimeMatrix dashboard
- [ ] Functional tests cover: cold miss, hot hit, epoch bump → miss, warehouse bypass, `useStaged` differentiates from live cache, `includeBreakdown` differentiates from slim cache

---

## Out of Scope

- The calculator implementations themselves - delivered by [3.3-calculate-jobcard-lead-time.md](3.3-calculate-jobcard-lead-time.md) and [3.4-calculate-order-lead-time.md](3.4-calculate-order-lead-time.md)
- The Flowgear sync workflow - delivered by [flowgear-sync-support-api.md](flowgear-sync-support-api.md) and the **nightly sync** story (BRD 3.7)
- Multi-region cache replication - the existing Moyo `IDistributedCache` Redis configuration is reused as-is
- Cache warming on startup - cold misses are acceptable at the start of each business day
