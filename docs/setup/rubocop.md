# RuboCop Setup (Extending the Omakase Base)

## What it is

[RuboCop](https://rubocop.org/) is a Ruby static-analysis linter and formatter.
The repo already ships with **rubocop-rails-omakase** (Basecamp's house style)
wired through `.rubocop.yml`. This doc **extends** that base with three official
plugins: `rubocop-rspec`, `rubocop-performance`, and `rubocop-factory_bot`.

## Why we use it

- Consistent style with zero bike-shedding (`bin/rubocop -a` autocorrects).
- `rubocop-rspec` enforces good spec structure (once RSpec is set up).
- `rubocop-performance` flags slow idioms; `rubocop-factory_bot` keeps factories
  tidy.

## Prerequisites

- The omakase setup already present (`.rubocop.yml`, `bin/rubocop`).
- Do RSpec setup first (`rspec.md`) so `rubocop-rspec` has specs to lint.

## Install

```bash
bundle add rubocop-rspec        --group "development,test" --require false
bundle add rubocop-performance  --group "development,test" --require false
bundle add rubocop-factory_bot  --group "development,test" --require false
```

> These are already `require: false`-style dev tools; omakase itself is pulled in
> via `rubocop-rails-omakase` (already in the Gemfile).

## Configuration

### Current `.rubocop.yml`

```yaml
# Omakase Ruby styling for Rails
inherit_gem: { rubocop-rails-omakase: rubocop.yml }
```

### Extend it

Replace with:

```yaml
# Omakase Ruby styling for Rails
inherit_gem: { rubocop-rails-omakase: rubocop.yml }

plugins:
  - rubocop-performance
  - rubocop-rspec
  - rubocop-factory_bot

AllCops:
  NewCops: enable
  Exclude:
    - "bin/**/*"
    - "db/**/*"
    - "config/**/*"
    - "vendor/**/*"
    - "node_modules/**/*"

# House style overrides go here, e.g.:
# Layout/SpaceInsideArrayLiteralBrackets:
#   Enabled: false
```

> Newer RuboCop uses the `plugins:` key (not the deprecated `require:`) for
> plugin gems. If your RuboCop version warns, switch `plugins:` → `require:`.

### Seed a TODO file (optional but recommended on an existing codebase)

Freeze current offenses so CI stays green while you fix them gradually:

```bash
bin/rubocop --auto-gen-config
```

This writes `.rubocop_todo.yml` and adds `inherit_from: .rubocop_todo.yml` to
`.rubocop.yml`. Delete entries from the todo file as you clean them up.

## How to run

```bash
bin/rubocop            # report offenses
bin/rubocop -a         # safe autocorrect
bin/rubocop -A         # all autocorrect (incl. unsafe)
bin/rubocop app/models/user.rb   # one file
bin/rubocop -f github  # GitHub Actions annotations (used in CI)
```

## How to verify

- `bin/rubocop` runs and reports `no offenses detected` (or a known count frozen
  in `.rubocop_todo.yml`).
- Introduce a deliberate style break (e.g. bad indentation) → `bin/rubocop`
  flags it → `bin/rubocop -a` fixes it.

## Common gotchas

- **Plugin not loaded** → cop names like `RSpec/...` show as "unrecognized". Make
  sure the gem is installed and listed under `plugins:`.
- **CI cache**: the existing `lint` job in `.github/workflows/ci.yml` caches
  `tmp/rubocop` keyed on `Gemfile.lock` + `.rubocop*.yml`; changing config busts
  it automatically.
- Keep `bin/rubocop` (the binstub) as the entry point so the CI and `bin/ci`
  invocations match.

## Hands-on checklist

- [ ] `bundle add` rubocop-rspec, rubocop-performance, rubocop-factory_bot
- [ ] Add the `plugins:` and `AllCops:` blocks to `.rubocop.yml`
- [ ] (Optional) `bin/rubocop --auto-gen-config` to create `.rubocop_todo.yml`
- [ ] Run `bin/rubocop -a`
- [ ] Verify: `bin/rubocop` reports clean (or a frozen offense count)
