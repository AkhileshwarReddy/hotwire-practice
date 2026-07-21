# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

`hotwire-practice` is a fresh Rails 8.1 scaffold (Ruby 3.3.9, PostgreSQL, Hotwire/importmap,
Propshaft, Solid Queue/Cache/Cable, Kamal) being grown into a hands-on learning playground.
There is essentially no app code yet — no models, no routes beyond `/up`, no specs.

## Docs-first working norm (important)

When asked to "set up" a tool or gem, the deliverable is a **standalone setup doc under
`docs/setup/`** that the user follows and executes by hand — NOT direct edits to the Gemfile,
config, or source. Doing the setup manually is the point. Only touch code directly if the user
explicitly says so.

- Follow `docs/setup/_template.md`; `docs/setup/rspec.md` is the reference example. Every doc
  ends in an ordered **Hands-on checklist**.
- Planned direction already decided: RSpec (not Minitest), Sidekiq+Redis (replacing Solid
  Queue), full-app Docker Compose for dev, extended linters, Flipper, dev multi-DB.

## Commands

- `bin/ci` runs the pipeline (mirrored in `.github/workflows/ci.yml`): **lint + security only**
  (rubocop-rails-omakase, bundler-audit, importmap audit, Brakeman). No test job yet — there is
  no test framework in place; the default Minitest `test/` dir is empty.
