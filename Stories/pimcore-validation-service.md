# Story: Lead Time Matrix - Pimcore Validation Service for Invoice & Setup Codes

**TD Executive Summary : Systems Impacted**

**BRD References:** 2.2.3 (Validation against Pimcore)
**Tech Design References:** TechnicalDesign/02 (Systems Impacted - Pimcore)
**Dev Hours:** 6

---

## Overview

Implement a reusable **Pimcore validation service** consumed by the invoice/setup code create and update endpoints (BRD 2.2.3). Pimcore is the source of truth for product metadata and the billing-side authority for invoice/setup codes. Before persisting an invoice/setup code change in Moyo, the service must confirm the code exists (and is active) in Pimcore.

The service must be:
- **Reusable** - injected into both the invoice code and setup code application handlers
- **Resilient** - retry transient failures; circuit-break sustained outages so the rest of the module stays operational
- **Observable** - log every validation attempt and outcome (per TechnicalDesign/11)

---

## Service Contract

Add the following abstraction in `LeadTimeMatrix.Application/Pimcore/`:

```csharp
public interface IPimcoreValidationService
{
	Task<PimcoreValidationResult> ValidateInvoiceCodeAsync(string code, CancellationToken ct);
	Task<PimcoreValidationResult> ValidateSetupCodeAsync(string code, CancellationToken ct);
}

public sealed record PimcoreValidationResult(
	bool Exists,
	bool IsActive,
	string? Message);
```

Implementation lives in `LeadTimeMatrix.Infrastructure/Pimcore/PimcoreValidationService.cs` and uses a typed `HttpClient` registered via `IHttpClientFactory`.

---

## HTTP Client Configuration

Configure a **typed `HttpClient`** with the following defaults (consistent with other Moyo modules):

| Setting | Value | Notes |
|---------|-------|-------|
| `BaseAddress` | from `Pimcore:BaseUrl` config | Per-environment |
| Authentication | `Pimcore:ApiKey` header (from `IConfiguration` / Secret Manager) | Never log the key |
| Connect timeout | 2s | Fail fast |
| Request timeout | 5s | |
| Retry policy | 3 attempts, exponential backoff (200ms / 400ms / 800ms) | Polly `WaitAndRetryAsync` |
| Circuit breaker | break for 30s after 5 consecutive failures | Polly `CircuitBreakerAsync` |

Register via:

```csharp
services.AddHttpClient<IPimcoreValidationService, PimcoreValidationService>()
	.AddPolicyHandler(GetRetryPolicy())
	.AddPolicyHandler(GetCircuitBreakerPolicy());
```

---

## Validation Flow

For both invoice and setup code validation:

1. Caller passes the candidate `code` string.
2. Service calls Pimcore's product/code-lookup endpoint (path/contract to be confirmed with the Pimcore team — assume `GET /v1/codes/{code}` returning `{ exists: bool, active: bool }`).
3. Map the response to `PimcoreValidationResult`:
   - HTTP 200 + `exists: true` + `active: true` → `Exists=true, IsActive=true`
   - HTTP 200 + `exists: true` + `active: false` → `Exists=true, IsActive=false, Message="Code is inactive in Pimcore"`
   - HTTP 404 → `Exists=false, IsActive=false, Message="Code does not exist in Pimcore"`
   - HTTP 5xx (after retries exhausted) or circuit breaker open → throw `PimcoreUnavailableException`
4. The application handler decides how to surface the result:
   - `Exists=false` → return validation error to the caller (matches BRD 2.2.3 wording: "Validation against Pimcore")
   - `IsActive=false` → return validation error
   - `PimcoreUnavailableException` thrown → return a clear "Pimcore unavailable, please retry shortly" error to the caller; **do NOT** silently allow the code through

---

## Configuration

Add to `appsettings.json` (and per-environment overrides):

```json
{
  "Pimcore": {
	"BaseUrl": "https://pimcore.amrod.local/",
	"ApiKey": "<from secrets>",
	"TimeoutSeconds": 5,
	"RetryCount": 3,
	"CircuitBreakerThreshold": 5,
	"CircuitBreakerDurationSeconds": 30
  }
}
```

The `ApiKey` must be supplied via Secret Manager / environment variables in non-development environments — never committed.

---

## Observability

Per TechnicalDesign/11 every Pimcore call must produce a structured log entry containing:

- `Operation` (`PimcoreValidateInvoiceCode` | `PimcoreValidateSetupCode`)
- `Code` (the value being validated)
- `DurationMs`
- `Outcome` (`Exists` | `NotFound` | `Inactive` | `Failed`)
- `RetryCount`
- `CircuitState` (`Closed` | `Open` | `HalfOpen`)

Forward these to **Signoz** alongside the rest of the module logs (see [observability-admin-audit-logs.md](observability-admin-audit-logs.md)).

---

## Acceptance Criteria

- [ ] `IPimcoreValidationService` interface lives in `LeadTimeMatrix.Application/Pimcore/`
- [ ] `PimcoreValidationService` implementation lives in `LeadTimeMatrix.Infrastructure/Pimcore/` and uses a typed `HttpClient`
- [ ] Polly retry (3 attempts, exponential backoff) and circuit breaker (5 failures / 30s) policies are wired in
- [ ] The `appsettings.json` has a `Pimcore` section; `ApiKey` is sourced from Secret Manager in non-dev environments
- [ ] The invoice code create/update handlers (BRD 2.2.3) call `ValidateInvoiceCodeAsync` before persisting; non-existent or inactive codes return a validation error
- [ ] The setup code create/update handlers (BRD 2.2.3) call `ValidateSetupCodeAsync` before persisting; non-existent or inactive codes return a validation error
- [ ] When Pimcore is unreachable, callers receive a clear "Pimcore unavailable" error and the change is **not** persisted
- [ ] Structured log entries are emitted for every call and forwarded to Signoz
- [ ] Unit tests cover: happy path, 404, inactive, transient failure with retry success, transient failure exhausting retries, circuit breaker open, cancellation
- [ ] Integration tests use `WireMock.Net` (or similar) to stub Pimcore responses end-to-end

---

## Out of Scope

- The Pimcore endpoint contract details (path, payload) must be **confirmed with the Pimcore team** during implementation; this story documents the assumed shape but the resolver layer must adapt to the real contract.
- Caching Pimcore responses — these calls are infrequent (only on admin code create/update) and freshness matters more than latency, so no caching for this service.
- Bulk validation — out of scope; one code per call is sufficient for the BRD 2.2.3 endpoints.
