# Database: Postgres

## Local setup

- Runs in the root `compose.yaml` (`postgres:<major>` image; check the current major and pin it). Bound to `127.0.0.1:5432`, data in a named volume.
- Connection via `DATABASE_URL` only. Dev and test databases: `<name>` and `<name>_test`.
- Schema changes go only through the stack's migration tool (Alembic / drizzle-kit). Never hand-edit the dev DB and never edit a migration that has been applied anywhere. Write a new one.

## Naming

| Thing | Convention | Example |
|---|---|---|
| Tables | `snake_case`, plural | `users`, `order_items` |
| Columns | `snake_case`, singular | `email`, `created_at` |
| Primary key | `id` | |
| Foreign key | `<singular>_id` | `user_id` |
| Booleans | `is_` / `has_` prefix | `is_active` |
| Timestamps | `_at` suffix | `deleted_at` |
| Indexes | `ix_<table>_<cols>` | `ix_orders_user_id` |
| Unique | `uq_<table>_<cols>` | `uq_users_email` |
| Foreign key constraints | `fk_<table>_<col>_<ref>` | `fk_orders_user_id_users` |
| Check | `ck_<table>_<rule>` | `ck_orders_total_positive` |

Configure the ORM's naming convention once (SQLAlchemy `MetaData(naming_convention=...)`; explicit names in Drizzle) so constraint names are predictable in migrations.

## Column defaults

- **IDs:** `uuid primary key default gen_random_uuid()`. UUIDs are safe to expose in URLs and don't leak row counts. Use `uuidv7()` if the Postgres version has it (time-ordered, friendlier to indexes). Plain `bigint generated always as identity` is fine for internal join tables that are never exposed.
- **Timestamps:** every table gets `created_at timestamptz not null default now()` and `updated_at timestamptz not null default now()`, with `updated_at` maintained by the ORM or a trigger. Always `timestamptz`, never `timestamp`. Store UTC.
- **Text:** `text` + a check constraint for a max length if it matters (not `varchar(n)`).
- **Money:** `numeric(12,2)` or integer minor units (`amount_cents bigint`). Never `float`/`real`.
- **Enums:** `text` + check constraint (easy to change in a migration) rather than native `enum` types.
- **Soft delete:** only when the domain needs it (`deleted_at timestamptz null`). Don't add it by default.
- `not null` unless null has a real meaning.
- **Foreign keys:** always declared, always indexed (Postgres doesn't index FK columns automatically). Choose `on delete` deliberately (`restrict` by default).

## Practices

- Seed data via a script (`scripts/seed`), not migrations. Use obviously fake data (`example.com`, `555-01xx`).
- The app connects as a non-superuser role in anything beyond local dev.
- Long or risky migrations (big table rewrites, new non-null columns on large tables) get a note in the PR or decision record.
- Log slow queries at `warning` from the app side (threshold via `DB_SLOW_QUERY_MS`, default 500).
