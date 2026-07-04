# Baseline Stack

## Ruby And Rails

```ruby
# Gemfile highlights
ruby file: ".ruby-version"   # 4.0.3

gem "rails", "~> 8.1.3"
gem "pg", "~> 1.1"
gem "puma", ">= 5.0"
gem "kamal", require: false
gem "thruster", require: false
gem "vite_rails", "~> 3.0"
gem "turbo-rails"
gem "stimulus-rails"
gem "solid_queue"
gem "solid_cache"
gem "solid_cable"
gem "jbuilder"
gem "image_processing", "~> 1.2"
gem "bootsnap", require: false

group :development, :test do
  gem "amazing_print"
  gem "brakeman", require: false
  gem "bundler-audit", require: false
  gem "dotenv", ">= 3.0"
  gem "factory_bot_rails"
  gem "pry-byebug"
  gem "pry-rails"
  gem "rspec-rails"
  gem "simplecov", require: false
  gem "debug", platforms: %i[mri windows], require: "debug/prelude"
end

group :development do
  gem "erb_lint", require: false
  gem "rubocop", require: false
  gem "rubocop-rails", require: false
  gem "rubocop-performance", require: false
  gem "rubocop-capybara", require: false
  gem "rubocop-factory_bot", require: false
  gem "web-console"
end

group :test do
  gem "capybara", require: false
  gem "selenium-webdriver", require: false
  gem "shoulda-matchers"
end
```

## Pry And Amazing Print

Add to `Gemfile` (`:development, :test`):

```ruby
gem "amazing_print"
gem "pry-byebug"
gem "pry-rails"
```

`pry-rails` replaces `rails console` with Pry. `pry-byebug` adds `break`/`next`/`step` debugging.

Configure **per developer machine** in `~/.pryrc` (create the file if it does not exist). Append at the end:

```ruby
require "amazing_print"
AmazingPrint.pry!
```

Do not commit `~/.pryrc` to the app repo — it is a personal shell config.

## JavaScript And Vite

`package.json` baseline:

```json
{
  "private": true,
  "type": "module",
  "engines": { "node": "^20.9.0 || >= 22.0.0" },
  "dependencies": {
    "@hotwired/stimulus": "^3.2.2",
    "@hotwired/turbo-rails": "^8.0.23",
    "stimulus-vite-helpers": "^3.1.0",
    "vite": "^8.0.0",
    "vite-plugin-rails": "^0.6.0"
  },
  "devDependencies": {
    "@tailwindcss/vite": "^4.3.1",
    "tailwindcss": "^4.3.1",
    "run-pty": "^6",
    "eslint": "^10",
    "prettier": "^3.8.4"
  }
}
```

`vite.config.ts`:

```ts
import { defineConfig } from "vite"
import ViteRails from "vite-plugin-rails"
import tailwindcss from "@tailwindcss/vite"

export default defineConfig({
  plugins: [
    tailwindcss(),
    ViteRails({
      envVars: { RAILS_ENV: "development" },
      envOptions: { defineOn: "import.meta.env" },
      fullReload: { additionalPaths: [] },
    }),
  ],
})
```

`config/vite.json`:

```json
{
  "all": {
    "sourceCodeDir": "app/frontend",
    "watchAdditionalPaths": []
  },
  "development": {
    "autoBuild": true,
    "skipProxy": true,
    "publicOutputDir": "vite-dev",
    "port": 3036
  },
  "test": {
    "autoBuild": true,
    "publicOutputDir": "vite-test",
    "port": 3037
  }
}
```

Frontend layout:

```
app/frontend/
├── entrypoints/application.js   # imports CSS, controllers, turbo-rails
├── controllers/                 # Stimulus controllers
├── stylesheets/
│   ├── index.css                # @import tailwindcss + @source directives
│   └── base.css                 # custom base styles
└── images/
```

`application.js` entrypoint:

```js
import "~/stylesheets/index.css"
import "~/controllers"
import "@hotwired/turbo-rails"
```

Layout (`app/views/layouts/application.html.erb`):

```erb
<%= vite_client_tag %>
<%= vite_javascript_tag "application", "data-turbo-track": "reload" %>
```

## Dev Server

`bin/dev` delegates to `run-pty`:

```json
// run-pty.json
[
  { "command": ["bin/rails", "server"], "status": { "Listening on": null } },
  { "command": ["bin/vite", "dev"], "status": { "ready in": null } }
]
```

## Database

`config/database.yml` pattern:

```yaml
default: &default
  adapter: postgresql
  encoding: unicode
  host: <%= ENV.fetch("DATABASE_HOST", "localhost") %>
  port: <%= ENV.fetch("DATABASE_PORT", 5432) %>
  username: <%= ENV.fetch("DATABASE_USERNAME", "postgres") %>
  password: <%= Rails.application.credentials.dig(:database, :password) %>
  max_connections: <%= ENV.fetch("RAILS_MAX_THREADS") { 5 } %>

development:
  <<: *default
  database: app_development

test:
  <<: *default
  database: app_test

production:
  primary: &primary_production
    <<: *default
    database: app_production
    username: app
    password: <%= ENV["DATABASE_PASSWORD"] %>
  cache:
    <<: *primary_production
    database: app_production_cache
    migrations_paths: db/cache_migrate
  queue:
    <<: *primary_production
    database: app_production_queue
    migrations_paths: db/queue_migrate
  cable:
    <<: *primary_production
    database: app_production_cable
    migrations_paths: db/cable_migrate
```

Credentials structure (`bin/rails credentials:edit`):

```yaml
secret_key_base: ...
database:
  password: ...
```

## Credentials And Secrets

| File | Git? | Purpose |
|------|------|---------|
| `config/credentials.yml.enc` | Yes | Encrypted secrets |
| `config/master.key` | No | Decryption key |
| `.kamal/secrets` | Yes (no raw secrets) | Shell that exports secrets for Kamal |

`.kamal/secrets`:

```bash
RAILS_MASTER_KEY=$(cat config/master.key)
```

Three strategies for dev/prod separation (pick one per project):

1. **Hybrid (default):** credentials for dev/test, `DATABASE_PASSWORD` env var for production.
2. **Single credentials file with env keys:** `credentials.dig(Rails.env.to_sym, :database, :password)`.
3. **Per-env credentials:** `bin/rails credentials:edit --environment production` → `config/credentials/production.yml.enc`.

## .gitignore Essentials

```
/config/*.key
/vendor
/node_modules
/public/vite*
/.env*
```

## CI Workflow

Job order: `eslint` → `erb_lint` → `scan_ruby` (Brakeman + bundler-audit) → `lint` (RuboCop) → `test` (RSpec + Postgres service).

Trigger on all `push` and `pull_request` (no branch filter).

Test job env:

```yaml
env:
  RAILS_ENV: test
  DATABASE_URL: postgres://postgres:postgres@localhost:5432
```

`erb_lint` on Ruby 4.0 may print a `parser/ruby33` compatibility warning — that is upstream noise, not a failure.

## After Migrations

```bash
bundle exec rails db:migrate
bundle update brakeman
```

## Tests

- RSpec with `spec/rails_helper.rb`, Factory Bot, Shoulda Matchers.
- System tests via Capybara + Selenium (Chrome in CI).
- Commit `db/schema.rb` so fresh environments can `db:schema:load`.
