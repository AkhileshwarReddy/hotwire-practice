# Security Scanning Setup (Brakeman · bundler-audit · importmap audit)

## What it is

Three static security scanners that ship with the Rails 8 scaffold:

- **[Brakeman](https://brakemanscanner.org/)** — scans Rails code for
  vulnerabilities (SQLi, XSS, mass-assignment, unsafe redirects, …).
- **bundler-audit** — checks `Gemfile.lock` against the ruby-advisory-db of
  known-vulnerable gem versions.
- **importmap audit** — checks pinned JavaScript packages for known
  vulnerabilities (this app uses importmap-rails, so JS deps are pinned, not
  bundled).

> All three are **already installed and already wired into CI**. This doc
> documents how to run and interpret them — there is nothing new to install.

## Why we use it

- Catch security regressions before they merge (Brakeman + audits run on every
  PR — see `.github/workflows/ci.yml`, `scan_ruby` and `scan_js` jobs).
- Zero-config, fast, no external service.

## Prerequisites

Nothing new — these come from the scaffold's `:development, :test` group and the
`bin/` binstubs (`bin/brakeman`, `bin/bundler-audit`, `bin/importmap`).

## Configuration

### Brakeman

- Config file (optional): `config/brakeman.yml`.
- To ignore a reviewed false positive, generate an ignore file:

```bash
bin/brakeman -I    # interactive; writes config/brakeman.ignore
```

### bundler-audit

- The scaffold already has `config/bundler-audit.yml` for ignoring specific
  advisories:

```yaml
# config/bundler-audit.yml
ignore:
  # - CVE-2024-xxatn   # reason it's not applicable
```

- Update the advisory database before scanning:

```bash
bin/bundler-audit update
```

### importmap audit

No config; it reads `config/importmap.rb`.

## How to run

```bash
# Brakeman — code vulnerability scan
bin/brakeman --no-pager
bin/brakeman --quiet --no-pager --exit-on-warn --exit-on-error   # CI-strict

# Gem advisories
bin/bundler-audit update && bin/bundler-audit

# JavaScript pin advisories
bin/importmap audit
```

All three run together locally via `bin/ci` (see `github-ci.md`).

## How to verify

- `bin/brakeman --no-pager` prints a report ending in
  `No warnings found` (or a list of warnings to triage).
- `bin/bundler-audit` prints `No vulnerabilities found`.
- `bin/importmap audit` prints `No vulnerabilities found`.

## Common gotchas

- **bundler-audit false-error on a stale DB** — always `bundler-audit update`
  first; CI does this implicitly via a fresh checkout.
- **Brakeman warning you've reviewed** — don't silence globally; use
  `bin/brakeman -I` to record a per-warning ignore with a note.
- **importmap audit only covers pinned packages** — if you later add a JS build
  step (esbuild/bun), use `npm audit` / `yarn audit` instead.

## Hands-on checklist

- [ ] Run `bin/brakeman --no-pager` and read the report
- [ ] Run `bin/bundler-audit update && bin/bundler-audit`
- [ ] Run `bin/importmap audit`
- [ ] (Optional) Record reviewed false positives via `bin/brakeman -I` / `config/bundler-audit.yml`
- [ ] Verify: all three report clean, and they run as part of `bin/ci`
