# Helpdesk Schema Repair

PostgreSQL schema for the Helpdesk ERD. Apply the migration to a test database, then run the pgTAP suite:

- `npm run db:reset` recreates `helpdesk_test` and applies the migration.
- `npm test` runs the schema assertions (requires PostgreSQL and pgTAP/pg_prove).

## ERD-to-Schema Decisions

### Decision 1: UUID identifiers and foreign keys
ERD said: Every entity has a UUID primary key, and relationships use UUID foreign keys.
Schema decided: Primary keys use `UUID PRIMARY KEY DEFAULT gen_random_uuid()`; foreign keys are also UUID.
Reason: UUID types preserve the ERD's identifiers and allow rows to be created without requiring callers to supply IDs.

### Decision 2: Referential deletion behavior
ERD said: Relationships define ownership and authorship, with nullable ticket assignment.
Schema decided: Organization and audit references use `RESTRICT`; deleting an assigned agent uses `SET NULL`; comments and tag associations cascade with their parent ticket/tag; organization-owned tags cascade with the organization.
Reason: This preserves audit history, avoids orphaned dependent rows, and retains tickets when an assignee is removed.

### Decision 3: Timestamp and status defaults
ERD said: Creation timestamps are timestamps, ticket status is one of `open`, `pending`, `resolved`, or `closed`, and nullable lifecycle timestamps are explicitly marked.
Schema decided: Creation timestamps are `TIMESTAMPTZ NOT NULL DEFAULT NOW()`; ticket status is `TEXT NOT NULL DEFAULT 'open'` with a CHECK constraint; `resolved_at` and `deactivated_at` remain nullable `TIMESTAMPTZ` values.
Reason: Time-zone-aware timestamps provide consistent event times, new tickets start in the ERD's initial state, and optional lifecycle events remain unset until they occur.

## Rejected Table Shape

### Shape: Array-valued `tickets.tags`
What it was: A `TEXT[]` column stored all tag names directly on each ticket.
Why rejected: The ERD models tags as organization-owned entities and ticket/tag membership as a many-to-many relationship.
What would break: Tag metadata and ownership would be duplicated, and the database could not enforce references to tags or track who added each association. The schema instead uses `tags` and `ticket_tags` tables.
