# GitHub CI + `bin/ci` Setup

## What it is

The repo already has a GitHub Actions workflow (`.github/workflows/ci.yml`) and a
local CI runner (`bin/ci` + `config/ci.rb`, using
`ActiveSupport::ContinuousIntegration`). This doc **extends both** with the new
checks added by the other setup docs:

- a **test job** running RSpec against Postgres (+ Redis) service containers
  (the scaffold has **no test job** because it shipped without a test suite),
- **ERB lint**, **ESLint/Prettier**, and **typos** steps.

## Why we use it

- Every PR gets linted, security-scanned, **and tested** before merge.
- `bin/ci` runs the exact same steps locally, so "green locally" means "green on
  CI".

## Prerequisites

Set these up first (their steps are referenced here): `rspec.md`, `rubocop.md`,
`erb-lint.md`, `javascript-linting.md`, `spelling-and-editorconfig.md`.

## Current state (scaffold)

`.github/workflows/ci.yml` has three jobs: `scan_ruby` (brakeman +
bundler-audit), `scan_js` (importmap audit), and `lint` (rubocop). `config/ci.rb`
runs setup, rubocop, and the three security scans. **No RSpec, no ERB/JS/typos.**

## Configuration

### 1. Add a `test` job to `.github/workflows/ci.yml`

Append this job (sibling to `lint`):

```yaml
  test:
    runs-on: ubuntu-latest

    services:
      postgres:
        image: postgres:16
        env:
          POSTGRES_USER: postgres
          POSTGRES_PASSWORD: postgres
        ports:
          - 5432:5432
        options: >-
          --health-cmd "pg_isready -U postgres"
          --health-interval 5s --health-timeout 5s --health-retries 5
      redis:
        image: redis:7
        ports:
          - 6379:6379
        options: >-
          --health-cmd "redis-cli ping"
          --health-interval 5s --health-timeout 5s --health-retries 5

    env:
      RAILS_ENV: test
      DATABASE_HOST: localhost
      DATABASE_USERNAME: postgres
      DATABASE_PASSWORD: postgres
      REDIS_URL: redis://localhost:6379/0

    steps:
      - uses: actions/checkout@v6

      - name: Set up Ruby
        uses: ruby/setup-ruby@v1
        with:
          bundler-cache: true

      - name: Install Chrome (for Cuprite system specs)
        uses: browser-actions/setup-chrome@v1

      - name: Prepare database
        run: bin/rails db:prepare

      - name: Run tests
        run: bundle exec rspec
```

> This relies on the ENV-driven `config/database.yml` from `docker-compose.md`
> (host/username/password read from ENV). If you skipped that, set
> `DATABASE_URL` instead.

### 2. Add ERB + typos steps to the `lint` job

Inside the existing `lint` job, after the rubocop step:

```yaml
      - name: Lint ERB templates
        run: bundle exec erb_lint --lint-all

      - name: Spell check
        uses: crate-ci/typos@master
```

### 3. Add a `lint_js` job (Node)

```yaml
  lint_js:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v6
      - uses: actions/setup-node@v4
        with:
          node-version: "22"
          cache: "npm"
      - run: npm ci
      - run: npm run lint
      - run: npm run format:check
```

### 4. Mirror everything in `config/ci.rb` (for `bin/ci`)

The current `CI.run do … end` block runs setup, rubocop, and the security scans.
Add these steps inside the block:

```ruby
  step "Test: RSpec", "bundle exec rspec"

  step "Style: ERB", "bundle exec erb_lint --lint-all"

  step "Style: JavaScript", "npm run lint"
  step "Format: Prettier", "npm run format:check"

  step "Spelling: typos", "typos"
```

Place `step "Test: RSpec"` right after the `step "Setup"` line so failures show
early.

## How to run

```bash
bin/ci                 # run the whole pipeline locally (mirrors GitHub Actions)
```

On GitHub, the workflow triggers automatically on `pull_request` and pushes to
`main`.

## How to verify

- `bin/ci` runs every step and ends with a green summary.
- Open a PR → the Actions tab shows `scan_ruby`, `scan_js`, `lint`, `lint_js`,
  and `test` all passing.
- Break a spec on a branch → the `test` job (and `bin/ci`) go red.

## Common gotchas

- **`test` job can't reach Postgres** → the `services.postgres.ports` mapping and
  `DATABASE_HOST: localhost` must both be set; `bin/rails db:prepare` must run
  before `rspec`.
- **System specs fail on CI** → Chrome isn't installed; keep the
  `browser-actions/setup-chrome` step.
- **`typos` step version** → pin `crate-ci/typos@vX` instead of `@master` for
  reproducibility once you know the version you want.
- Keep local (`config/ci.rb`) and remote (`ci.yml`) step lists in sync so
  `bin/ci` stays a faithful preview.

## Hands-on checklist

- [ ] Add the `test` job (postgres + redis services, Chrome, `db:prepare`, `rspec`) to `ci.yml`
- [ ] Add ERB lint + typos steps to the `lint` job
- [ ] Add the `lint_js` job (npm ci, lint, format:check)
- [ ] Add matching `step` lines to `config/ci.rb`
- [ ] Run `bin/ci` locally → green
- [ ] Push a branch / open a PR → verify all jobs pass in the Actions tab
