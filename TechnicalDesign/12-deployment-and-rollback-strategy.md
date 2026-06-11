# Deployment & Rollback Strategy

Strategy for moving to production and recovering from failure.

## Data Migration

1. **Extract-Transform-Load (ETL) Run**: Moyo runs an initial migration to parse and migrate Amtrack legacy data into the Moyo schema.
2. **Parity Run**: Run parallel shadow calculations for 5 business days, comparing Moyo API results with legacy Amtrack calculations before cutting over traffic.

---

## Rollback Plan

If, during sanity testing, system error rates exceed 0.5% or performance latencies climb above 300ms, trigger the rollback protocol:

1. **Toggle Route**: Flip the feature to point back to Amtrack's legacy database tables.
2. **Lock State**: Maintain a read-only lock on the Moyo database configuration tables during debugging to prevent configuration drift.

