# Setup Docs

Hands-on setup guides for `hotwire-practice` — a Rails 8.1 learning playground.
Each doc is **standalone and executable**: exact commands, full config snippets,
verification, and a checklist. You install and configure everything **by hand**
by following these docs (they are the deliverable, not the code changes).

> Base stack (already from `rails new`): Ruby 3.3.9, Rails 8.1, PostgreSQL,
> Hotwire (Turbo + Stimulus, importmap), Propshaft, Solid Queue/Cache/Cable,
> Kamal. Also already present: RuboCop (omakase), Brakeman, bundler-audit,
> importmap audit, a GitHub CI workflow (lint + security only, **no test job**),
> `bin/ci` + `config/ci.rb`, Dependabot, and a production `Dockerfile`.

## Recommended order

Work top-down. **Foundation** first (testing, containers, linting, CI), then the
**practice gems**.

### Foundation

| # | Doc | What you set up |
|---|-----|-----------------|
| 1 | [rspec.md](rspec.md) | RSpec + FactoryBot + Faker + Shoulda + SimpleCov + Capybara/Cuprite (replaces Minitest) |
| 2 | [docker-compose.md](docker-compose.md) | Full-app dev stack: `Dockerfile.dev` + `compose.yaml` (web, jobs, db, redis) |
| 3 | [rubocop.md](rubocop.md) | Extend the omakase base with rubocop-rspec / -performance / -factory_bot |
| 4 | [security-scanning.md](security-scanning.md) | Run & interpret Brakeman, bundler-audit, importmap audit (already installed) |
| 5 | [erb-lint.md](erb-lint.md) | `erb_lint` for `.html.erb` views |
| 6 | [javascript-linting.md](javascript-linting.md) | ESLint + Prettier as dev-only tooling (importmap = no bundler) |
| 7 | [spelling-and-editorconfig.md](spelling-and-editorconfig.md) | `typos` spell-check + `.editorconfig` |
| 8 | [github-ci.md](github-ci.md) | Add the missing **test job** + ERB/JS/typos steps to CI and `bin/ci` |

### Practice gems

| # | Doc | What you set up |
|---|-----|-----------------|
| 9 | [redis.md](redis.md) | Redis for jobs / cache / flags; standard `REDIS_URL` |
| 10 | [sidekiq.md](sidekiq.md) | Switch Active Job from Solid Queue to Sidekiq + Redis |
| 11 | [flipper.md](flipper.md) | Feature flags (Active Record adapter) + Flipper UI |
| 12 | [multiple-databases.md](multiple-databases.md) | Explicit dev secondary database via `connects_to` |

Template for adding new docs: [_template.md](_template.md).

## Master checklist

Tick a doc once you've completed its own hands-on checklist and verification.

**Foundation**
- [ ] 1. RSpec — `bundle exec rspec` green, `coverage/` generated
- [ ] 2. Docker Compose — `docker compose up` runs web/jobs/db/redis; app on `:3000`
- [ ] 3. RuboCop extensions — `bin/rubocop` clean with the new plugins
- [ ] 4. Security scanning — Brakeman / bundler-audit / importmap audit all clean
- [ ] 5. ERB lint — `erb_lint --lint-all` clean
- [ ] 6. JS/CSS lint — `npm run lint` and `npm run format:check` pass
- [ ] 7. typos + EditorConfig — `typos` clean, `.editorconfig` honored
- [ ] 8. CI — `bin/ci` green locally; all GitHub Actions jobs (incl. `test`) pass

**Practice gems**
- [ ] 9. Redis — `redis-cli -u "$REDIS_URL" ping` → `PONG`
- [ ] 10. Sidekiq — a job runs; `/sidekiq` UI shows it
- [ ] 11. Flipper — a flag toggles; `/flipper` dashboard works
- [ ] 12. Multiple databases — both DBs exist; secondary model reads/writes its DB

## Conventions

- Every tool gets a doc here following [_template.md](_template.md).
- Every doc ends with a **Hands-on checklist**.
- Docs note when the Rails scaffold already provides something, so we extend
  rather than duplicate.

## Roadmap / ideas for future docs

Candidates to add as the playground grows: Devise/authentication, Pundit/CanCanCan
authorization, ViewComponent, Pagy pagination, i18n, Action Cable/Turbo Streams
broadcasting, background-job cron (sidekiq-cron), observability (Sentry, Lograge,
OpenTelemetry), and API tooling (rswag/OpenAPI).
