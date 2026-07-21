# Flipper (Feature Flags) Setup

## What it is

[Flipper](https://www.flippercloud.io/docs) is a feature-flag library: turn
features on/off at runtime, roll out to a percentage of users or specific
actors/groups, all without a deploy. This doc uses the **Active Record** storage
adapter (durable, no extra infra) plus the optional **Flipper UI** dashboard.

## Why we use it

- Ship code dark, enable it gradually, kill-switch instantly.
- Practice percentage rollouts, per-actor gates, and group gates.
- Active Record adapter keeps flag state in Postgres (no Redis dependency); a
  Redis adapter is noted as an alternative.

## Prerequisites

- Postgres running (for the Active Record adapter) — already the app default.
- For per-actor gates you'll want a model (e.g. `User`) that responds to
  `flipper_id`.

## Install

```bash
bundle add flipper
bundle add flipper-active_record
bundle add flipper-ui            # optional dashboard
```

> Redis alternative: `bundle add flipper-redis` and configure with `REDIS_URL`
> (see `redis.md`) instead of the Active Record adapter below.

## Configuration

### 1. Create the storage tables

```bash
bin/rails generate flipper:active_record
bin/rails db:migrate
```

This adds `flipper_features` and `flipper_gates` tables.

### 2. Initializer

Create `config/initializers/flipper.rb`:

```ruby
require "flipper"
require "flipper/adapters/active_record"

Flipper.configure do |config|
  config.adapter { Flipper::Adapters::ActiveRecord.new }
end
```

### 3. Mount the UI (optional)

`config/routes.rb`:

```ruby
require "flipper/ui"

Rails.application.routes.draw do
  # TODO: protect this in production (see gotchas)
  mount Flipper::UI.app(Flipper) => "/flipper"
  # ...
end
```

### 4. (Optional) make a model an actor

Any model with a stable `flipper_id` can be an actor. Active Record models get
`flipper_id` (`"User;123"`) for free. For group gates, register groups in the
initializer:

```ruby
Flipper.register(:admins) { |actor| actor.respond_to?(:admin?) && actor.admin? }
```

## How to run / use

From the console or app code:

```ruby
# Boolean (global) gate
Flipper.enable(:new_dashboard)
Flipper.enabled?(:new_dashboard)            # => true

# Per-actor gate
Flipper.enable_actor(:beta_export, current_user)
Flipper.enabled?(:beta_export, current_user)

# Percentage-of-actors rollout (25%)
Flipper.enable_percentage_of_actors(:new_dashboard, 25)

# Group gate
Flipper.enable_group(:admin_tools, :admins)
```

In views/controllers:

```erb
<% if Flipper.enabled?(:new_dashboard, current_user) %>
  <%= render "dashboards/new" %>
<% end %>
```

## How to verify

- After migrating, `Flipper.features` returns a set (empty at first).
- `Flipper.enable(:demo)` then `Flipper.enabled?(:demo)` → `true`; the row exists
  in `flipper_gates`.
- Visit `http://localhost:3000/flipper` → the dashboard lists `demo` and lets you
  toggle it.

## Common gotchas

- **Unprotected UI** — `/flipper` can change app behaviour; gate it behind admin
  auth in any shared/production environment.
- **`flipper_id` on POROs** — non-AR actors must implement `flipper_id`
  themselves, or gates silently never match.
- **Adapter mismatch across processes** — web and Sidekiq must use the **same**
  adapter/store or a flag enabled in one won't be seen in the other. Active
  Record (shared DB) avoids this; if you use `flipper-redis`, point both at the
  same `REDIS_URL`.
- **Caching** — for high-traffic apps wrap the adapter in
  `Flipper::Adapters::Memoizable` (Rails middleware `Flipper::Middleware::Memoizer`)
  to avoid a DB hit per check.

## Hands-on checklist

- [ ] `bundle add flipper flipper-active_record flipper-ui`
- [ ] `bin/rails generate flipper:active_record && bin/rails db:migrate`
- [ ] Create `config/initializers/flipper.rb`
- [ ] Mount `Flipper::UI` at `/flipper` (protect it)
- [ ] Toggle a flag from the console and confirm `enabled?`
- [ ] Verify: `/flipper` dashboard shows and toggles the flag
