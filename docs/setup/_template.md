# <Tool Name> Setup

> Template for every doc in `docs/setup/`. Copy this file, fill in each section,
> and keep the **Hands-on checklist** at the bottom as an ordered, do-able list.

## What it is

One or two sentences describing the tool and where it fits.

## Why we use it

Why it earns a place in this project. What problem it solves.

## Prerequisites

- Ruby 3.3.9, Rails 8.1, PostgreSQL running locally (all present in this repo).
- Anything else this specific tool needs (Node, Redis, Docker, …).

## Install

Exact commands. Prefer `bundle add` (writes a version to the Gemfile for you)
over hand-editing, e.g.:

```bash
bundle add <gem> --group "development,test"
```

## Configuration

Every file to create or edit, with the **full snippet** to paste. Call out
anything the Rails scaffold already provides so we don't duplicate it.

## How to run

The day-to-day command(s), e.g. `bundle exec rspec`, `bin/rubocop -a`.

## How to verify

A concrete check that proves it works (expected output, a passing run, a URL
that renders).

## Common gotchas

Known foot-guns and their fixes.

## Hands-on checklist

- [ ] Step 1 (a single copy-pasteable action)
- [ ] Step 2
- [ ] Verify: <the proof it works>
