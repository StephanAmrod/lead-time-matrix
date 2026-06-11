# Current State & Technical Debt "Tax"

## Current State Analysis

Historically, lead times were configured in legacy Amtrack SQL tables.

- **Affected Components**: Amtrack Ordering Engine, Amtrack Jobcard Dispatcher, Moyo Stock Checker.
- **Known Constraints**: Legacy Amtrack processes run on MS SQL Server with stored procedures. Direct SQL reads are coupled.
- **Integration Flow**: Legacies push changes via database replication, leading to data inconsistencies and timing race conditions.

---

## Technical Debt Register

Any decision that deviates from the Architecture Principles (e.g., direct DB integration) must be recorded here.

| Debt Classification | Risk Score (1-10) | Impact on Moyo/Amtrack | Repayment Milestone/Date |
|---|---|---|---|
| [e.g., No Observability] | | | |

---

**Note**: No TDD with known debt will be approved without an entry in the centralised Debt Register.

