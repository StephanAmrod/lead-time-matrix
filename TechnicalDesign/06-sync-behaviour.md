# Sync Behaviour (Scheduled vs. Immediate)

Updates to the matrix only take effect from midnight on the day that the change is made.

Once an order is approved and paid, its calculated timeline is locked.

- **Lock State**: Active matrix updates **must not** alter timelines for orders that are already paid, approved, or scheduled.
- **Execution Safety**: A batch matrix adjustment will only affect future, unapproved, and unpaid orders.

