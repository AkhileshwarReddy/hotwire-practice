# Multiple Databases (Development) Setup

## What it is

Rails supports connecting a single app to **multiple databases** — separate
primary databases per domain, and/or read replicas. This doc adds an explicit
**secondary database** in development so you can practice `connects_to`,
per-database migrations, and replica reads.

## Why we use it

- Practice horizontal data separation (e.g. an `analytics` or `reporting` DB) and
  primary/replica read splitting.
- Understand `migrations_paths`, `bin/rails db:*:<name>` tasks, and
  `connected_to`.

> Note: production `config/database.yml` is **already** multi-DB — Solid
> Cache/Queue/Cable each get their own database (`*_cache`, `*_queue`,
> `*_cable`). This doc adds a multi-DB setup for **application** data in
> development.

## Prerequisites

- Postgres running (host or the Compose `db` service).
- Comfort with `config/database.yml` (see `docker-compose.md` for the ENV-driven
  `default:` anchor).

## Configuration

### 1. Declare a second database in `config/database.yml`

Extend `development` (and `test`) to a 2-database config:

```yaml
development:
  primary:
    <<: *default
    database: hotwire_practice_development
  analytics:
    <<: *default
    database: hotwire_practice_analytics_development
    migrations_paths: db/analytics_migrate

test:
  primary:
    <<: *default
    database: hotwire_practice_test
  analytics:
    <<: *default
    database: hotwire_practice_analytics_test
    migrations_paths: db/analytics_migrate
```

> Once a `development`/`test` env lists named connections (`primary:`,
> `analytics:`), **all** connections must be named — `primary` is the default.

### 2. Abstract base model for the secondary DB

Create `app/models/analytics_record.rb`:

```ruby
class AnalyticsRecord < ApplicationRecord
  self.abstract_class = true

  connects_to database: { writing: :analytics, reading: :analytics }
end
```

Models that live in the analytics DB inherit from it:

```ruby
class PageView < AnalyticsRecord
end
```

### 3. Generate migrations into the right path

```bash
bin/rails generate model PageView path:string --database analytics
```

The `--database analytics` flag routes the migration to `db/analytics_migrate`.

## How to run

```bash
# Create every configured database
bin/rails db:create

# Migrate all databases
bin/rails db:migrate

# Target a single database
bin/rails db:migrate:analytics
bin/rails db:migrate:primary

# Prepare everything (create + migrate + seed) — used in CI/compose
bin/rails db:prepare
```

Reading explicitly from a replica/other connection at runtime:

```ruby
AnalyticsRecord.connected_to(role: :reading) do
  PageView.count
end
```

## How to verify

- `bin/rails db:create` creates both `hotwire_practice_development` and
  `hotwire_practice_analytics_development`.
- `PageView.create!(path: "/")` writes to the analytics DB;
  `ActiveRecord::Base.connection_db_config.database` differs from
  `AnalyticsRecord.connection_db_config.database`.
- `bin/rails db:migrate:status:analytics` lists only the analytics migrations.

## Common gotchas

- **All-or-nothing naming** — the moment you add named connections, the plain
  `database:` form for that env stops working; use `primary:`.
- **Schema files** — each database gets its own schema dump
  (`db/schema.rb` for primary, `db/analytics_schema.rb` for analytics). Commit
  both.
- **`migrations_paths` must be unique per DB**, or migrations run against the
  wrong database.
- **Replicas are read-only** — writing inside `connected_to(role: :reading)`
  raises; use `:writing` for writes.
- Update `docker-compose.md`/CI to run `db:prepare` (which handles all
  databases) rather than a single-DB migrate.

## Hands-on checklist

- [ ] Convert `development`/`test` in `config/database.yml` to named connections with an `analytics` DB
- [ ] Create `app/models/analytics_record.rb` with `connects_to`
- [ ] Generate a model with `--database analytics`
- [ ] `bin/rails db:create && bin/rails db:migrate`
- [ ] Create a record on the secondary DB from the console
- [ ] Verify: both databases exist and the analytics model reads/writes the analytics DB
