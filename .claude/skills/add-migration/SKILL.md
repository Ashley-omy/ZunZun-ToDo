---
name: add-migration
description: Create a new Flyway database migration for ZunZun ToDo. Use whenever the schema changes — new tables, columns, indexes, constraints — or when the user mentions Flyway, migrations, "add a table", "add a column", "change the schema", or "update the database". Also use when a new feature needs somewhere to store data.
---

# Creating a Flyway Migration

## Existing migrations

!`ls -1 backend/src/main/resources/db/migration/ 2>/dev/null || echo "(no migrations yet)"`

Derive the next version number from the output above. Never guess it.

## Steps

1. Check the target schema in `docs/design-specification.md` §6 — the migration must match it
2. Create `backend/src/main/resources/db/migration/V{n}__{snake_case_description}.sql`
3. Update the matching JPA entity and repository in the same change
4. Run `cd backend && ./mvnw test` so Testcontainers applies the migration to real PostgreSQL
5. If the schema deliberately differs from the SDS, update the SDS in the same pull request

## Absolute Rules

- **Never edit a migration that has already been applied.** Always add a new file.
- Never rename a migration file once created.

## Naming

- Tables: plural snake_case (`progress_logs`, `user_hashtags`)
- Columns: snake_case (`log_date`, `completed_at`)
- Primary key: `id BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY` (join tables use a composite key)
- Foreign keys: `{singular_table}_id`
- Indexes `idx_{table}_{columns}` · unique indexes `uq_{meaning}` · checks `chk_{meaning}`

## Conventions

**Time**
- Moments in time: `TIMESTAMPTZ` (never `TIMESTAMP`), e.g. `created_at TIMESTAMPTZ NOT NULL DEFAULT now()`
- Calendar days chosen by the user: `DATE` (e.g. `log_date`) — never a timestamp
- Tables that are modified get `updated_at`

**Durations**
- Whole minutes as `INTEGER` with a range check (`minutes BETWEEN 1 AND 1440`)

**Enumerated values**
- `VARCHAR` + `CHECK (... IN (...))`, matching the existing schema; Java enums map with `@Enumerated(EnumType.STRING)`

**Foreign keys**
- Always state `ON DELETE`. User-owned data uses `ON DELETE CASCADE` so account deletion removes everything

**Indexes and constraints**
- Every new query path gets an index in the same migration
- Enforce invariants in the database where possible — e.g. a partial unique index for "one completion per task":
  `CREATE UNIQUE INDEX uq_logs_one_completion_per_task ON progress_logs(task_id) WHERE completed;`

**Optimistic locking**
- Tables edited from more than one screen (projects, tasks) have `version INTEGER NOT NULL DEFAULT 0` mapped with `@Version`

## Example

```sql
-- V5__social.sql
CREATE TABLE follows (
  follower_id  BIGINT NOT NULL REFERENCES users(id) ON DELETE CASCADE,
  followee_id  BIGINT NOT NULL REFERENCES users(id) ON DELETE CASCADE,
  created_at   TIMESTAMPTZ NOT NULL DEFAULT now(),
  PRIMARY KEY (follower_id, followee_id),
  CONSTRAINT chk_no_self_follow CHECK (follower_id <> followee_id)
);

CREATE INDEX idx_follows_followee ON follows(followee_id);
```

## Checklist

- [ ] Version continues from the existing files; no existing file modified
- [ ] Matches SDS §6 (or the SDS was updated)
- [ ] `TIMESTAMPTZ` for moments, `DATE` for calendar days, minutes as `INTEGER`
- [ ] Every foreign key declares `ON DELETE`
- [ ] New query paths are indexed
- [ ] Entity and repository updated
- [ ] `./mvnw test` passes
