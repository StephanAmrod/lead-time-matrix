# Special Dates & Working Day Logic

The engine must exclude weekends and public holidays from all due-date calculations.

- **Monthly Automation Pipeline**: Every 1st of the month, Flowgear triggers a task to fetch South African Public Holidays that have been published. These will be saved with the "Public Holiday" category.
- **Duplicate and Conflict Protection**: Overlapping dates are ignored on conflicts rather than creating duplicates.
- **Immutability Rule**: Historic dates are locked. The system strictly rejects modifications to any date earlier than the current date.

