# Kamal And Docker Baseline

## Dockerfile

Multi-stage Ruby slim image with Node for Vite build, jemalloc, non-root `rails` user, Thruster entry.

Key stages:

1. **base** — runtime packages (`libpq`, `libvips`, `postgresql-client`, jemalloc)
2. **build** — `bundle install`, `yarn install --immutable`, bootsnap precompile
3. **final** — copy artifacts, `ENTRYPOINT bin/docker-entrypoint`, `CMD ./bin/thrust ./bin/rails server`

```dockerfile
ARG RUBY_VERSION=4.0.3
FROM docker.io/library/ruby:$RUBY_VERSION-slim AS base
WORKDIR /rails

ENV RAILS_ENV="production" \
    BUNDLE_DEPLOYMENT="1" \
    BUNDLE_PATH="/usr/local/bundle" \
    BUNDLE_WITHOUT="development" \
    LD_PRELOAD="/usr/local/lib/libjemalloc.so"
```

Build stage installs Node via node-build (`ARG NODE_VERSION`, `ARG YARN_VERSION`), copies `Gemfile.lock` + `yarn.lock`, runs `bundle install` and `yarn install --immutable`.

**Production asset build:** before `rm -rf node_modules`, run Vite/Rails asset precompile so `public/vite` exists in the image:

```dockerfile
RUN SECRET_KEY_BASE_DUMMY=1 ./bin/rails assets:precompile
```

Without this step, production containers serve no JS/CSS.

`COPY vendor/* ./vendor/` supports vendored gems if present; keep `/vendor` gitignored for `vendor/bundle`.

## docker-entrypoint

```bash
#!/bin/bash -e
if [ "${@: -2:1}" == "./bin/rails" ] && [ "${@: -1:1}" == "server" ]; then
  ./bin/rails db:prepare
fi
exec "${@}"
```

## deploy.yml Highlights

```yaml
service: app_name
image: app_name

servers:
  web:
    - 192.168.0.1

registry:
  server: localhost:5555

env:
  secret:
    - RAILS_MASTER_KEY
    - DATABASE_PASSWORD
  clear:
    SOLID_QUEUE_IN_PUMA: true

volumes:
  - "app_name_storage:/rails/storage"

asset_path: /rails/public/vite   # Vite Ruby output, not /public/assets

aliases:
  console: app exec --interactive --reuse "bin/rails console"
  shell: app exec --interactive --reuse "bash"
  logs: app logs -f
  dbc: app exec --interactive --reuse "bin/rails dbconsole --include-password"

builder:
  arch: amd64
```

## .kamal/secrets

Never put raw passwords in this file. Read from env or password manager:

```bash
RAILS_MASTER_KEY=$(cat config/master.key)
DATABASE_PASSWORD=$DATABASE_PASSWORD
```

Standard env vars this starter uses:

| Variable | Where | Purpose |
|----------|-------|---------|
| `RAILS_MASTER_KEY` | Kamal secret | Decrypt `credentials.yml.enc` |
| `DATABASE_PASSWORD` | Kamal secret | Production PostgreSQL password |
| `DATABASE_URL` | CI test job | Full test DB connection string |
| `DATABASE_HOST` | Optional clear env | Override DB host (default `localhost`) |
| `DATABASE_PORT` | Optional clear env | Override DB port (default `5432`) |
| `DATABASE_USERNAME` | Optional clear env | Override DB user (default `postgres`) |
| `RAILS_MAX_THREADS` | Optional clear env | Connection pool size |
| `RAILS_ENV` | Runtime | `development`, `test`, or `production` |

## Production Checks

After deploy-related edits:

1. `docker build -t app_name .` succeeds
2. Built image contains `public/vite/assets/`
3. `config/deploy.yml` `asset_path` matches Vite output dir
4. `RAILS_MASTER_KEY` and `DATABASE_PASSWORD` are in Kamal secrets, not in git
5. `config/environments/production.rb` has `force_ssl` aligned with Kamal proxy SSL settings

## Git History Cleanup

If `vendor/bundle` was committed:

```bash
# preferred
git filter-repo --path vendor --invert-paths

# fallback
git filter-branch -f --index-filter 'git rm -rf --cached --ignore-unmatch vendor' HEAD
rm -rf .git/refs/original/
git reflog expire --expire=now --all && git gc --prune=now --aggressive
```

Then ensure `/vendor` is in `.gitignore`. Local `vendor/bundle` can remain on disk for `bundle install --path vendor/bundle`.
