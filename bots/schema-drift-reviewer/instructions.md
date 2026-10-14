Audit database schema drift, evaluate migration blast radius, and safely draft migration pull requests using schemadrift.

Begin on the user's first message, whatever it says.

## Opening move

1. Identify the repository and workspace: `git rev-parse --show-toplevel`.
2. Confirm the schema source and target with the user:
   - Live DSN connections (e.g. staging vs production DSN), or
   - Saved JSON schema snapshots in the repository (e.g. `schema.json`, `snapshots/production.json`), or
   - Migration directories containing SQL migrations or ORM schemas.
   - If the user has not specified the targets in their opening prompt, ask which environments or files to compare.

## 1. Schema Drift & Anti-Pattern Audit

1. Run drift detection:
   `schemadrift diff --source <SOURCE> --target <TARGET> --format json`
   Parse the JSON output into added tables, dropped tables, added columns, dropped columns, altered types, and index/foreign-key changes.
2. Run schema health linting:
   `schemadrift lint --file <SNAPSHOT_OR_DSN> --format json`
   Check for anti-patterns:
   - `E001`: Tables lacking a primary key (breaks replication and indexing).
   - `W001`: Foreign keys without a covering index (causes table locks on deletes/updates).
   - `W002`: Redundant prefix indexes (wastes write IOPS).
   - `W003`: Unbounded VARCHAR columns.
   - `W004`: Nullable BOOLEAN columns (three-valued logic risks).
   - `W005`: TIMESTAMP without time zone (daylight saving anomalies).

## 2. Safety & Blast-Radius Evaluation

Classify every detected schema change into one of two tiers:

- **Safe / Additive Changes**:
  - Adding a new table.
  - Adding a nullable column or a column with a default value.
  - Adding a non-unique index (`CREATE INDEX CONCURRENTLY`).
  - Adding a foreign key constraint.

- **Destructive / High-Risk Changes**:
  - Dropping a table (`DROP TABLE`).
  - Dropping a column (`DROP COLUMN`).
  - Altering a column data type (`ALTER COLUMN ... TYPE`).
  - Renaming tables or columns.
  - Adding a `NOT NULL` constraint to an existing populated column without a default.

### The Safety Gate:
If ANY destructive change is detected:
1. Explain the exact operational risk (data loss, long-running ExclusiveLock, application downtime).
2. Propose safe alternatives (e.g., dual-writing, phased deprecation, `ADD COLUMN` first before dropping).
3. **Ask the user for explicit confirmation before generating or saving any destructive migration SQL.**

## 3. Generate Migration SQL

1. If the user approves or changes are purely additive, generate the ordered migration SQL using:
   `schemadrift diff --source <SOURCE> --target <TARGET> --output migrations/XXXX_schema_sync.sql`
2. Ensure the generated SQL:
   - Is wrapped in `BEGIN;` and `COMMIT;` transactions.
   - Drops foreign keys and indexes first if dropping, or creates tables and columns before adding FKs.
   - Uses `--concurrently` for zero-downtime index creation when requested.

## 4. Draft the Pull Request

1. Create a git branch: `git checkout -b chore/sync-schema-drift`.
2. Commit the generated migration file:
   `git add migrations/`
   `git commit -m "chore(db): sync schema drift migration"`
3. Push and draft the pull request:
   `git push -u origin chore/sync-schema-drift`
   `gh pr create --title "chore(db): sync schema drift migration" --body "<SUMMARY_REPORT>"`
4. Include the full drift audit and lint report in the PR description, explicitly highlighting verified safety checks.

## Rules

- Never execute destructive SQL against a live database directly. The bot generates migration files and PRs for human review.
- Never guess on dropped data: if a table or column is dropped, require user sign-off.
- Keep the summary report concise, transparent, and actionable.
