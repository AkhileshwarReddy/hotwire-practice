# Redis Setup

## What it is

[Redis](https://redis.io/) is an in-memory data store used here as the backbone
for **Sidekiq** (job queue), and optionally for **caching** and **Flipper**
(feature-flag storage). This doc gets Redis running locally and gives the app a
`REDIS_URL` to connect to.

## Why we use it

- Required by Sidekiq (`sidekiq.md`).
- Fast cache store and a natural Flipper adapter (`flipper.md`).
- The `docker-compose.md` stack already defines a `redis` service — this doc
  covers the host-machine option and the shared connection conventions.

## Prerequisites

- Either Docker (Redis comes from `compose.yaml`) or Homebrew for a host install.

## Install

### Option A — Docker (recommended, already in compose)

Redis is defined in `compose.yaml`. Start just it:

```bash
docker compose up redis
```

### Option B — Homebrew (host)

```bash
brew install redis
brew services start redis    # run in background, restart on login
# or run in the foreground:
redis-server
```

## Configuration

### 1. Add the client gem (only if you talk to Redis directly)

Sidekiq bundles the `redis-client`, so you may not need this. For direct use or
caching:

```bash
bundle add redis
```

### 2. Standardize `REDIS_URL`

Everything reads one env var. Defaults:

- Host-run Rails: `redis://localhost:6379/0`
- Compose: `redis://redis:6379/0` (already set on `web`/`jobs` in `compose.yaml`)

Add to your local env (e.g. a `.env`, or your shell profile):

```bash
export REDIS_URL="redis://localhost:6379/0"
```

### 3. (Optional) Use Redis as the Rails cache store

The app currently uses **Solid Cache** (DB-backed). To switch a specific
environment to Redis, in `config/environments/development.rb`:

```ruby
config.cache_store = :redis_cache_store, { url: ENV.fetch("REDIS_URL", "redis://localhost:6379/1") }
```

> Use a different DB index (`/1`) than Sidekiq (`/0`) to keep them separate.

## How to run

```bash
redis-cli ping                     # -> PONG
redis-cli -u "$REDIS_URL" ping     # test the exact URL the app uses
redis-cli monitor                  # watch live commands (great for debugging Sidekiq)
```

## How to verify

- `redis-cli ping` returns `PONG`.
- From a Rails console: `Redis.new(url: ENV["REDIS_URL"]).ping` returns `"PONG"`
  (requires the `redis` gem).
- In Compose: `docker compose exec redis redis-cli ping` → `PONG`.

## Common gotchas

- **Port 6379 already in use** — a host Redis and the Compose Redis both bind
  6379. Run only one, or remap the Compose port.
- **Wrong DB index collisions** — Sidekiq, cache, and Flipper sharing one Redis
  DB is fine but noisy; separate them by URL DB index (`/0`, `/1`, `/2`) if you
  want isolation.
- **Data is not durable by default** — fine for dev jobs/cache; never treat dev
  Redis as a source of truth.

## Hands-on checklist

- [ ] Start Redis (`docker compose up redis` or `brew services start redis`)
- [ ] Export `REDIS_URL`
- [ ] (Optional) `bundle add redis` for direct/cache use
- [ ] (Optional) Point a cache store at Redis in an environment file
- [ ] Verify: `redis-cli -u "$REDIS_URL" ping` → `PONG`
