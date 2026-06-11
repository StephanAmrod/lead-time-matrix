# Principles Compliance Checklist

Confirm alignment with the Architecture Principles.

| Principle | Compliant? (Y/N) | Comments |
|---|---|---|
| API-first integration model | Y | All interactions between systems (Amtrack ERP, Moyo Stock Check, Website) use structured API endpoints. |
| No direct database integrations | Y | No external applications query Moyo's database directly. All data access goes through API gateways. |
| Logic in Domain Services (not UI) | Y | The entire calculation logic is built into the Moyo core service, completely decoupled from any database or UI view. |
| Backward compatibility for shared services | Y | The API is versioned ( /api/v1/ ), allowing legacy endpoints to remain active if payload structures change. |

