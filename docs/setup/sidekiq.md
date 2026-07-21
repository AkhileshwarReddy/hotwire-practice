# Sidekiq Setup (Switching from Solid Queue)

## What it is

[Sidekiq](https://sidekiq.org/) is a Redis-backed background job processor and
the most widely used Active Job adapter in production Rails apps. This doc
**switches the app's Active Job backend from Solid Queue to Sidekiq**.

## Why we use it

- Industry-standard; huge ecosystem (Sidekiq Web UI, cron via sidekiq-cron,
  batches, etc.).
- Great for practicing Redis-backed queues, retries, and monitoring.

> The scaffold uses **Solid Queue** (DB-backed, run via `bin/jobs`). After this
> doc, Active Job enqueues to Sidekiq/Redis instead. See "Retiring Solid Queue".

## Prerequisites

- Redis running and `REDIS_URL` set (`redis.md`).
- Docker Compose stack understood (`docker-compose.md`) — the `jobs` service
  command changes here.

## Install

```bash
bundle add sidekiq
```

## Configuration

### 1. Point Active Job at Sidekiq

`config/application.rb` (or per-env in `config/environments/*.rb`):

```ruby
config.active_job.queue_adapter = :sidekiq
```

> The scaffold sets `config.solid_queue.connects_to` and
> `config.active_job.queue_adapter = :solid_queue` in
> `config/environments/production.rb`. Replace those with `:sidekiq` in the
> environments where you want Sidekiq.

### 2. Sidekiq server config

Create `config/sidekiq.yml`:

```yaml
:concurrency: 5
:queues:
  - default
  - mailers

# Environment-specific overrides:
production:
  :concurrency: 10
```

### 3. Redis connection (optional initializer)

Sidekiq reads `REDIS_URL` automatically. To customize, create
`config/initializers/sidekiq.rb`:

```ruby
Sidekiq.configure_server do |config|
  config.redis = { url: ENV.fetch("REDIS_URL", "redis://localhost:6379/0") }
end

Sidekiq.configure_client do |config|
  config.redis = { url: ENV.fetch("REDIS_URL", "redis://localhost:6379/0") }
end
```

### 4. Mount the Web UI

`config/routes.rb`:

```ruby
require "sidekiq/web"

Rails.application.routes.draw do
  # TODO: protect this in production (see gotchas)
  mount Sidekiq::Web => "/sidekiq"
  # ...
end
```

### 5. Update the Compose `jobs` service

In `compose.yaml`, change the `jobs` command from Solid Queue to Sidekiq:

```yaml
  jobs:
    command: bundle exec sidekiq
```

## How to run

```bash
# Host
bundle exec sidekiq

# Compose
docker compose up jobs

# Enqueue a test job from the console
bin/rails runner 'ActiveJob::Base.queue_adapter = :sidekiq; \
  Class.new(ApplicationJob) { def perform; Rails.logger.info("hi from sidekiq"); end }.perform_later'
```

## How to verify

- `bundle exec sidekiq` boots and logs `Booted Rails ... in development`.
- Enqueue any job (`SomeJob.perform_later`) → it processes; the Sidekiq log shows
  `start` / `done`.
- Visit `http://localhost:3000/sidekiq` → the Web UI shows processed/enqueued
  counts.

## Retiring Solid Queue

Once Sidekiq is your adapter everywhere:

- Remove `config.active_job.queue_adapter = :solid_queue` and the
  `config.solid_queue.*` lines from env files.
- You can leave the `solid_queue` gem installed (harmless) or remove it plus its
  `db/queue_*` schema if you want a clean cut. Keep it if you plan to compare the
  two in the practice repo.

## Common gotchas

- **Unprotected Web UI** — `Sidekiq::Web` exposes job data. In production wrap it
  in auth, e.g. `authenticate :user, ->(u){ u.admin? } do mount ... end` or HTTP
  basic auth.
- **Jobs enqueue but never run** — the Sidekiq process isn't running, or it's
  pointed at a different `REDIS_URL`/DB index than the web process.
- **Mailers** — `deliver_later` uses the `mailers` queue; make sure it's listed
  in `sidekiq.yml`.
- **CI** — the `test` job's Redis service (see `github-ci.md`) covers Sidekiq
  connectivity; most job specs use the `:test` adapter and don't need Redis.

## Hands-on checklist

- [ ] Ensure Redis is up and `REDIS_URL` is set (`redis.md`)
- [ ] `bundle add sidekiq`
- [ ] Set `config.active_job.queue_adapter = :sidekiq`
- [ ] Create `config/sidekiq.yml` (+ optional initializer)
- [ ] Mount `Sidekiq::Web` at `/sidekiq`
- [ ] Change the Compose `jobs` command to `bundle exec sidekiq`
- [ ] Verify: `bundle exec sidekiq` runs a job and `/sidekiq` UI shows it
- [ ] (Optional) retire the Solid Queue adapter config
