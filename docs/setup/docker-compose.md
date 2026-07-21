# Docker Compose (Local Dev) Setup

## What it is

A **development** Docker Compose stack that runs the whole app — Rails web
server, the background-jobs process, PostgreSQL, and Redis — in containers, so a
fresh clone boots with one command and everyone shares the same service
versions.

> The repo already has a **production** `Dockerfile` (used by Kamal). That image
> sets `RAILS_ENV=production`, drops the `development` bundle group, and
> precompiles assets — it is **not** for local iteration. This doc adds a
> separate `Dockerfile.dev` and `compose.yaml` for development.

## Why we use it

- No "works on my machine": Postgres 16 and Redis 7 are pinned.
- New contributors run `docker compose up` instead of installing Postgres/Redis
  by hand.
- Mirrors production topology (web + jobs + db + redis) closely.

## Prerequisites

- Docker Desktop (or Docker Engine + Compose v2).
- Ruby/Rails not required on the host — it runs in the container. (You can still
  run Rails on the host too; see the ENV note in `database.yml`.)

## Configuration

### 1. `Dockerfile.dev` (repo root)

```dockerfile
# syntax=docker/dockerfile:1
# Development image: full toolchain, dev+test gems, Node for JS linting.
ARG RUBY_VERSION=3.3.9
FROM docker.io/library/ruby:${RUBY_VERSION}-slim

WORKDIR /rails

# Build + runtime deps, plus Node.js for eslint/prettier and a psql client.
RUN apt-get update -qq && \
    apt-get install --no-install-recommends -y \
      build-essential git curl libpq-dev libvips libyaml-dev pkg-config \
      postgresql-client nodejs npm && \
    rm -rf /var/lib/apt/lists /var/cache/apt/archives

ENV RAILS_ENV="development" \
    BUNDLE_PATH="/usr/local/bundle"

# Install gems first for better layer caching. Source is bind-mounted at runtime.
COPY Gemfile Gemfile.lock ./
RUN bundle install

COPY . .

EXPOSE 3000
CMD ["bin/rails", "server", "-b", "0.0.0.0"]
```

### 2. `compose.yaml` (repo root)

```yaml
name: hotwire-practice

services:
  db:
    image: postgres:16
    environment:
      POSTGRES_USER: postgres
      POSTGRES_PASSWORD: postgres
    volumes:
      - postgres_data:/var/lib/postgresql/data
    ports:
      - "5432:5432"
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U postgres"]
      interval: 5s
      timeout: 5s
      retries: 5

  redis:
    image: redis:7
    volumes:
      - redis_data:/data
    ports:
      - "6379:6379"
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 5s
      timeout: 5s
      retries: 5

  web:
    build:
      context: .
      dockerfile: Dockerfile.dev
    command: bin/rails server -b 0.0.0.0
    environment:
      DATABASE_HOST: db
      DATABASE_USERNAME: postgres
      DATABASE_PASSWORD: postgres
      REDIS_URL: redis://redis:6379/0
    volumes:
      - .:/rails
      - bundle:/usr/local/bundle
      - node_modules:/rails/node_modules
    ports:
      - "3000:3000"
    depends_on:
      db:
        condition: service_healthy
      redis:
        condition: service_healthy

  jobs:
    build:
      context: .
      dockerfile: Dockerfile.dev
    command: bin/jobs            # Solid Queue today; swap to `bundle exec sidekiq` after sidekiq.md
    environment:
      DATABASE_HOST: db
      DATABASE_USERNAME: postgres
      DATABASE_PASSWORD: postgres
      REDIS_URL: redis://redis:6379/0
    volumes:
      - .:/rails
      - bundle:/usr/local/bundle
      - node_modules:/rails/node_modules
    depends_on:
      db:
        condition: service_healthy
      redis:
        condition: service_healthy

volumes:
  postgres_data:
  redis_data:
  bundle:
  node_modules:
```

> `redis` is included now so it's ready for the Sidekiq/Flipper passes; nothing
> uses it until then.

### 3. Make `database.yml` read the DB host from ENV

The scaffold's `development`/`test` blocks connect over a local Unix socket. In
Compose the DB is a separate host (`db`), so parameterize it. Edit the
`default:` anchor in `config/database.yml`:

```yaml
default: &default
  adapter: postgresql
  encoding: unicode
  max_connections: <%= ENV.fetch("RAILS_MAX_THREADS") { 5 } %>
  host: <%= ENV.fetch("DATABASE_HOST", nil) %>
  username: <%= ENV.fetch("DATABASE_USERNAME", nil) %>
  password: <%= ENV.fetch("DATABASE_PASSWORD", nil) %>
```

When the ENV vars are unset (running Rails on the host), `nil` host falls back to
the local socket exactly as before — so **both** host-run and Compose-run Rails
work.

### 4. `.dockerignore`

One already exists (it ignores `log/`, `tmp/`, etc.). Confirm it also excludes
`node_modules` and `.git` to keep the build context small.

## How to run

```bash
docker compose build                       # build web/jobs images
docker compose run --rm web bin/rails db:prepare   # create + migrate + seed
docker compose up                          # start db, redis, web, jobs
# App at http://localhost:3000
```

Useful:

```bash
docker compose run --rm web bin/rails console
docker compose run --rm web bundle exec rspec
docker compose logs -f jobs
docker compose down            # stop
docker compose down -v         # stop + wipe volumes (fresh DB)
```

## How to verify

- `docker compose ps` shows `db`, `redis`, `web`, `jobs` as `running`/`healthy`.
- `http://localhost:3000` renders the Rails welcome page (or your root route).
- `docker compose exec redis redis-cli ping` returns `PONG`.
- `docker compose exec db psql -U postgres -c '\l'` lists
  `hotwire_practice_development`.

## Common gotchas

- **First boot fails on missing DB** — run `db:prepare` once before `up`, or add
  it to an entrypoint.
- **Slow bundle on every build** — the named `bundle` volume caches gems; don't
  delete it casually. After changing the Gemfile run
  `docker compose build web` (or `docker compose run --rm web bundle install`).
- **Port already in use (5432/6379)** — a host Postgres/Redis is running. Stop
  it or change the published port (e.g. `"5433:5432"`).
- **File changes not picked up** — the bind mount `.:/rails` handles code
  reloads; if it seems stale on macOS, restart the `web` service.

## Hands-on checklist

- [ ] Create `Dockerfile.dev`
- [ ] Create `compose.yaml`
- [ ] Parameterize `host`/`username`/`password` in `config/database.yml` `default:`
- [ ] Confirm `.dockerignore` excludes `node_modules` and `.git`
- [ ] `docker compose build`
- [ ] `docker compose run --rm web bin/rails db:prepare`
- [ ] `docker compose up`
- [ ] Verify: app at `http://localhost:3000`, `redis-cli ping` → `PONG`, all 4 services healthy
