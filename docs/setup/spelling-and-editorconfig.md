# Spelling (typos) + EditorConfig Setup

## What it is

Two lightweight, language-agnostic hygiene tools:

- **[typos](https://github.com/crate-ci/typos)** — a fast source-code spell
  checker (Rust binary) that catches misspellings in code, comments, and docs
  with very few false positives.
- **[EditorConfig](https://editorconfig.org/)** — a `.editorconfig` file that
  makes every editor use the same indentation, charset, and newline rules.

## Why we use it

- `typos` keeps identifiers, comments, and docs free of embarrassing spelling
  mistakes; it runs in CI with no Ruby/Node dependency.
- `.editorconfig` prevents mixed tabs/spaces and CRLF/LF churn across
  contributors and editors — no gem required, honored by VS Code, RubyMine, etc.

## Prerequisites

- For local `typos` runs: install the CLI (see below). In CI it runs via the
  `crate-ci/typos` GitHub Action (no install needed) — see `github-ci.md`.

## Install (typos CLI, local)

```bash
# macOS
brew install typos-cli

# or via cargo
cargo install typos-cli
```

`.editorconfig` needs no install — just the file plus an editor that honors it
(most do natively; VS Code needs the "EditorConfig for VS Code" extension).

## Configuration

### 1. `.typos.toml` (repo root)

```toml
[files]
extend-exclude = [
  "*.lock",
  "db/schema.rb",
  "public/",
  "vendor/",
  "node_modules/",
  "coverage/",
  "tmp/",
  "log/",
]

[default.extend-words]
# Words typos flags that are actually correct in this project.
# Example: "hotwire" and Turbo/Stimulus jargon
hotwire = "hotwire"

[default.extend-identifiers]
# Whole identifiers to never flag, e.g. a gem or API name.
```

### 2. `.editorconfig` (repo root)

```ini
root = true

[*]
charset = utf-8
end_of_line = lf
insert_final_newline = true
trim_trailing_whitespace = true
indent_style = space
indent_size = 2

[*.{rb,rake,ru}]
indent_size = 2

[*.md]
trim_trailing_whitespace = false   # two trailing spaces = hard line break

[Makefile]
indent_style = tab
```

## How to run

```bash
typos                 # check the whole repo
typos --write-changes # apply the suggested fixes
typos path/to/file    # check one path
```

`.editorconfig` applies automatically once the file exists and your editor
supports it — nothing to run.

## How to verify

- `typos` prints `no typos found` (exit `0`).
- Introduce `recieve` in a comment → `typos` flags it and suggests `receive`.
- Open a file in your editor and confirm indentation/newline settings match
  `.editorconfig` (e.g. saving trims trailing whitespace).

## Common gotchas

- **False positives** on domain words (variable-name abbreviations, product
  names) → add them under `[default.extend-words]` /
  `[default.extend-identifiers]`, don't disable the tool.
- `typos` reads code identifiers too, so `camelCase`/`snake_case` splitting can
  surface odd hits — the extend lists are the fix.
- `.editorconfig` `root = true` must be present or editors keep searching parent
  directories.

## Hands-on checklist

- [ ] Install the `typos` CLI (`brew install typos-cli`)
- [ ] Create `.typos.toml`
- [ ] Create `.editorconfig`
- [ ] Run `typos` and clear/allow-list any findings
- [ ] Verify: `typos` is clean; editor honors `.editorconfig`; wire `typos` into CI (see `github-ci.md`)
