# RSpec + Testing Stack Setup

## What it is

[RSpec](https://rspec.info/) is a behaviour-driven testing framework for Ruby.
This doc replaces the default **Minitest** setup with RSpec and wires up the
companion gems most Rails apps use: FactoryBot (test data), Faker (fake values),
Shoulda Matchers (one-liner model/controller matchers), SimpleCov (coverage),
and Capybara + Cuprite (headless-Chrome system/feature specs — great for
testing Hotwire/Turbo interactions).

## Why we use it

- Expressive `describe`/`it` syntax and a huge ecosystem.
- FactoryBot + Faker make test data readable and DRY.
- Cuprite drives real Chrome over CDP (no Selenium/JVM), so Turbo Stream and
  Stimulus behaviour can be tested end-to-end.

## Prerequisites

- Ruby 3.3.9, Rails 8.1, PostgreSQL running (all present).
- A Chrome/Chromium binary on your machine for system specs (Cuprite uses it).

## Install

```bash
# Test framework + data + matchers (development so generators are available)
bundle add rspec-rails       --group "development,test"
bundle add factory_bot_rails --group "development,test"
bundle add faker             --group "development,test"
bundle add shoulda-matchers  --group "test"

# Coverage + system specs
bundle add simplecov --group "test" --require false
bundle add capybara  --group "test"
bundle add cuprite   --group "test"   # gem name: cuprite (Capybara driver)
```

Then generate the RSpec scaffolding:

```bash
bin/rails generate rspec:install
```

This creates `.rspec`, `spec/spec_helper.rb`, and `spec/rails_helper.rb`.

## Configuration

### 1. Remove Minitest

Rails scaffolded a `test/` directory. RSpec uses `spec/` instead:

```bash
rm -rf test
```

### 2. Tell generators to use RSpec + FactoryBot

`config/application.rb`, inside `class Application < Rails::Application`:

```ruby
config.generators do |g|
  g.test_framework :rspec,
    fixtures: true,
    view_specs: false,
    helper_specs: false,
    routing_specs: false
  g.fixture_replacement :factory_bot, dir: "spec/factories"
end
```

### 3. SimpleCov (must load first)

Top of `spec/spec_helper.rb`, **before any application code**:

```ruby
require "simplecov"
SimpleCov.start "rails" do
  add_filter "/spec/"
  enable_coverage :branch
end
```

### 4. Shoulda Matchers

Bottom of `spec/rails_helper.rb`:

```ruby
require "shoulda/matchers"

Shoulda::Matchers.configure do |config|
  config.integrate do |with|
    with.test_framework :rspec
    with.library :rails
  end
end
```

### 5. FactoryBot syntax methods

Create `spec/support/factory_bot.rb`:

```ruby
RSpec.configure do |config|
  config.include FactoryBot::Syntax::Methods
end
```

### 6. Capybara + Cuprite driver

Create `spec/support/capybara.rb`:

```ruby
require "capybara/rspec"
require "capybara/cuprite"

Capybara.register_driver(:cuprite) do |app|
  Capybara::Cuprite::Driver.new(
    app,
    window_size: [1200, 800],
    browser_options: {},
    process_timeout: 10,
    inspector: true
  )
end

Capybara.default_driver = :rack_test        # fast, no browser
Capybara.javascript_driver = :cuprite       # used for `js: true` specs
```

### 7. Load the support files

Uncomment this line in `spec/rails_helper.rb`:

```ruby
Dir[Rails.root.join("spec/support/**/*.rb")].sort.each { |f| require f }
```

### 8. Ignore coverage output

Append to `.gitignore`:

```gitignore
/coverage
```

### 9. Example spec (proves the suite runs)

`spec/requests/home_spec.rb` (adjust the path to a real route once you add one):

```ruby
require "rails_helper"

RSpec.describe "Smoke", type: :request do
  it "boots the app" do
    expect(Rails.application).to be_present
  end
end
```

Example factory `spec/factories/users.rb` (once you have a `User` model):

```ruby
FactoryBot.define do
  factory :user do
    email { Faker::Internet.unique.email }
  end
end
```

## How to run

```bash
bundle exec rspec                    # whole suite
bundle exec rspec spec/models        # a folder
bundle exec rspec spec/requests/home_spec.rb:5   # one example
bundle exec rspec --tag js           # only JS/system specs (Cuprite)
```

## How to verify

- `bundle exec rspec` exits `0` and prints `1 example, 0 failures`.
- After a run, `coverage/index.html` exists — open it in a browser.
- A `spec/system` spec tagged `js: true` opens Chrome headlessly and passes.

## Common gotchas

- **SimpleCov must be required before your app** or coverage reads ~0%. Keep it
  at the very top of `spec_helper.rb`.
- **`config.use_transactional_fixtures = true`** (default in `rails_helper.rb`)
  does NOT roll back system specs that use a real browser + separate DB
  connection. For system specs add a truncation strategy or the
  `database_cleaner-active_record` gem.
- Cuprite needs a Chrome binary; on CI install Chrome or use the
  `browser-actions/setup-chrome` action (see `github-ci.md`).
- Remember `require "rails_helper"` (not `spec_helper`) at the top of specs that
  need Rails loaded.

## Hands-on checklist

- [ ] `bundle add` the 7 gems (rspec-rails, factory_bot_rails, faker, shoulda-matchers, simplecov, capybara, cuprite)
- [ ] `bin/rails generate rspec:install`
- [ ] `rm -rf test`
- [ ] Add the `config.generators` block to `config/application.rb`
- [ ] Add SimpleCov to the top of `spec/spec_helper.rb`
- [ ] Add Shoulda Matchers config to `spec/rails_helper.rb`
- [ ] Create `spec/support/factory_bot.rb` and `spec/support/capybara.rb`
- [ ] Uncomment the `spec/support/**/*.rb` require in `rails_helper.rb`
- [ ] Add `/coverage` to `.gitignore`
- [ ] Add an example spec
- [ ] Verify: `bundle exec rspec` is green and `coverage/index.html` is generated
