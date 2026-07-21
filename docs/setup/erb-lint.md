# ERB Lint Setup

## What it is

[erb_lint](https://github.com/Shopify/erb_lint) (aka `erblint`) lints and
autocorrects `.html.erb` templates — trailing whitespace, indentation, Ruby
style inside ERB tags, and Rails-specific rules like requiring
`rails-erb-lint` safe helpers.

## Why we use it

- Views are code too; keeps `app/views/**/*.html.erb` consistent.
- Catches common template mistakes (e.g. deprecated tag styles) that RuboCop
  (which only sees `.rb`) misses.

## Prerequisites

- RuboCop set up (`rubocop.md`) — erb_lint reuses your `.rubocop.yml` for the
  Ruby it finds inside ERB tags.

## Install

```bash
bundle add erb_lint --group "development,test" --require false
```

The binary is `erb_lint` (older docs call it `erblint`; recent gem versions ship
`erb_lint`).

## Configuration

Create `.erb_lint.yml` in the repo root:

```yaml
---
EnableDefaultLinters: true

linters:
  Rubocop:
    enabled: true
    rubocop_config:
      inherit_gem:
        rubocop-rails-omakase: rubocop.yml
      # Disable cops that don't make sense inside an ERB tag:
      Style/FrozenStringLiteralComment:
        Enabled: false
      Layout/InitialIndentation:
        Enabled: false
      Layout/TrailingEmptyLines:
        Enabled: false

  FinalNewline:
    enabled: true

  SpaceInHtmlTag:
    enabled: true

  TrailingWhitespace:
    enabled: true

  ExtraNewline:
    enabled: true

  RightTrim:
    enabled: true
```

## How to run

```bash
bundle exec erb_lint --lint-all            # lint every ERB file
bundle exec erb_lint --lint-all --autocorrect   # fix what it can
bundle exec erb_lint app/views/layouts/application.html.erb   # one file
```

Add a convenience binstub (optional) `bin/erb_lint`:

```bash
bundle binstubs erb_lint
```

## How to verify

- `bundle exec erb_lint --lint-all` prints
  `No errors were found in ERB files`.
- Add a trailing space to a view → it's flagged → `--autocorrect` removes it.

## Common gotchas

- **Binary name**: use `erb_lint` (underscore). If a tutorial says `erblint`,
  it's the same tool, older packaging.
- **Rubocop-inside-ERB noise**: the `Rubocop` linter runs cops on the Ruby
  fragments; disable layout/frozen-string cops there (as above) or you'll get
  false positives that make no sense for template snippets.
- Exclude generated/vendored templates via a top-level `exclude:` key if needed.

## Hands-on checklist

- [ ] `bundle add erb_lint`
- [ ] Create `.erb_lint.yml`
- [ ] Run `bundle exec erb_lint --lint-all`
- [ ] Run `bundle exec erb_lint --lint-all --autocorrect`
- [ ] Verify: clean report; wire it into `bin/ci` and CI (see `github-ci.md`)
