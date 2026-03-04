# Data Integrity Rules

- Vote records are append-only — never update or delete a cast vote
- All vote mutations must be wrapped in database transactions
- Include audit timestamps (created_at, updated_at) on every table
- Vote tallying must be reproducible: same input always yields same output
- Use foreign key constraints to enforce referential integrity
- Validate ballot structure before persisting (correct race, valid candidate IDs)
- Implement idempotency keys for vote submission to prevent double-counting
- Store raw ballots alongside computed totals for independent verification
