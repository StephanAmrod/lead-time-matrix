# Dual-Run Synchronisation & Integrity

Mandatory for initiatives where Amtrack/Dynamics/Syspro, etc., and Moyo coexist.

## Synchronisation Directionality

**Sync Directionality**: Unidirectional sync from Moyo to Amtrack databases.

- **Production Lead Times**: Nightly sync (02:00) to minimise runtime load.
- **Warehouse Lead Times**: Real-time, transaction-safe updates.

---

## Golden Record Policy

**Golden Record**: Moyo is the single source of truth for lead time configurations.

---

## Amtrack Changes

Amtrack will no longer be custodian of data.

Amtrack needs to be updated to call the Moyo Gateway for the lead time calculations in stead of doing calculations internally.

Data changes in Moyo will not sync back to Amtrack for Lead Time calculations. This causes drift over time, making continued reliance on Amtrack calculations inaccurate.

