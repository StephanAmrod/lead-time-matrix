# Non-Functional Requirements (NFRs)

## Performance

- **API Response Time SLA**: Calculation requests from Moyo and the web portal must resolve in under 250ms (99th percentile) to prevent latency.
- **Throughput Support**: The calculation endpoints must scale up to 10 requests per second (RPS) under high-season loads.
- **Caching Strategy**: Cache calculation inputs. Clear the cache immediately on active matrix adjustments - This happens once a day at night.

---

## Observability

- **Detailed Log Structure**: Every administrative adjustment to overrides, print codes, or special dates must generate a structured log entry detailing:
  - `UserId` and `Timestamp`
  - `EntityId` and `ChangeDelta` (Value Before vs. Value After)
- **Log Forwarding**: Send logs to the centralised monitoring stack (Signoz) for analysis.

---

## Security

Administrative functions for editing/updating dates, adjustments and overrides should be permission-driven as per the BRD.

---

## Failover/Availability

- **Circuit-Breaker Pattern**: Implement a circuit breaker on legacy Amtrack routes. If Moyo API responses exceed 1,500ms or fail three times consecutively, Amtrack fails back to its local read-only database replica.

