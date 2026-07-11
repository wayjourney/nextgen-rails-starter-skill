---
name: "nextgen-rails-starter"
description: "Build or update a production-minded Rails starter using this repo's conventions for Rails 8.1+, Ruby 4.0+, PostgreSQL, Vite Ruby with Tailwind CSS 4, Hotwire (Turbo + Stimulus), Solid Queue/Cache/Cable, RSpec, GitHub Actions CI, Rails credentials, and Kamal deployment."
metadata:
  short-description: "Rails 8 + Vite + Tailwind 4 + Kamal starter"
---

# Nextgen Rails Starter

Use this skill when creating or updating a small production-ready Rails starter following this pattern.

## First Steps

1. Inspect the target repo before changing files: `Gemfile`, `package.json`, `vite.config.ts`, `config/vite.json`, `config/database.yml`, `config/deploy.yml`, `.kamal/secrets`, `Dockerfile`, `.github/workflows/ci.yml`, `.ruby-version`, `.node-version`, and existing specs.
2. Keep the starter boring and shippable: Rails 8 defaults, PostgreSQL, Vite Ruby for JS/CSS, Tailwind v4, Hotwire, Solid adapters, Docker + Kamal deploy path, RSpec for tests.
3. Match the package manager already present. If the repo has `yarn.lock`, use `yarn`; do not mix npm and yarn in Docker or CI.

## Starter Shape

Prefer this baseline unless the target project already has a stronger local convention:

- Ruby `4.0+`, Rails `8.1+`, PostgreSQL via `pg`.
- `vite_rails` `~> 3.0` with `vite-plugin-rails`; frontend source in `app/frontend`.
- Tailwind CSS 4 through `@tailwindcss/vite` (not the PostCSS plugin).
- Hotwire: `turbo-rails`, `stimulus-rails`; Stimulus controllers under `app/frontend/controllers`.
- Solid Queue, Solid Cache, Solid Cable (Rails 8 defaults).
- Kamal + Thruster for production containers.
- RSpec, Factory Bot, Capybara/Selenium for system tests.
- Pry console: `pry-rails`, `pry-byebug`, `amazing_print` (dev/test); `simplecov` for coverage.
- CI: ESLint, erb_lint, Brakeman, bundler-audit, RuboCop, RSpec.

## References

Read only what the task needs:

- `references/baseline.md`: Gemfile, Vite/Tailwind, Hotwire, database.yml, credentials, dev server, tests, CI.
- `references/kamal.md`: Dockerfile, docker-entrypoint, deploy.yml, secrets, production checks.
- `references/tailwind.md`: Tailwind v4 `@source` paths for ERB templates — the most common misconfiguration.
- `references/no-co-authored-by.mdc`: Cursor rule to copy into each app as `.cursor/rules/no-co-authored-by.mdc`. Does not stop Cursor Attribution injection — see **No Co-authored-by (Cursor Attribution)**.

## Implementation Rules

- Always add project Cursor rule `.cursor/rules/no-co-authored-by.mdc` (`alwaysApply: true`). Copy from `references/no-co-authored-by.mdc`. Commit messages = subject + optional plain description only — never `Co-authored-by` trailers.
- Dev/test DB password: `Rails.application.credentials.dig(:database, :password)` in `database.yml` default block. Store the value in encrypted credentials, not plain YAML.
- Production DB password: `ENV["DATABASE_PASSWORD"]`. Kamal injects secrets at deploy time; do not put prod passwords in credentials unless the team explicitly chooses per-env credentials files.
- `config/master.key` is never committed. Share it privately (password manager). Kamal reads it via `.kamal/secrets` as `RAILS_MASTER_KEY`.
- Add `/vendor` to `.gitignore`. Never commit `vendor/bundle`. If it was committed, purge it from history (`git filter-repo --path vendor --invert-paths` or `git filter-branch`), not just `git rm --cached`.
- Tailwind v4 does not auto-scan `app/views`. Register ERB paths with `@source` in `app/frontend/stylesheets/index.css`, relative to that CSS file (see `references/tailwind.md`).
- `bin/dev` runs Rails + Vite via `run-pty` (`run-pty.json`). Layout must include `vite_client_tag` and `vite_javascript_tag "application"`.
- After adding `amazing_print`, configure the developer machine's `~/.pryrc` (create if missing) with `require "amazing_print"` and `AmazingPrint.pry!` at the end. This is per-machine, not committed to the app repo.
- ERB files must end with a trailing newline (`erb_lint` `FinalNewline` rule).
- CI `scan_ruby` and `lint` jobs are top-level siblings of `test`, not nested under `erb_lint`.
- After adding migrations:

```bash
bundle exec rails db:migrate
bundle update brakeman
```
- Run validation after edits: `bin/rubocop`, `bin/erb_lint --lint-all`, `yarn run lint:js`, `bin/rspec`, and `yarn vite build` when frontend files changed.


## No Co-authored-by (Cursor Attribution)

Cursor Agent can auto-append `--trailer "Co-authored-by: Cursor <cursoragent@cursor.com>"` when running `git commit`. That injection is **not** written by the agent in the commit command, and **project `.cursor/rules` cannot block it**.

Disable Attribution on the machine (both layers):

1. **CLI agent** — in `~/.cursor/cli-config.json`:

```json
"attribution": {
  "attributeCommitsToAgent": false,
  "attributePRsToAgent": false
}
```

2. **IDE agent** — Cursor Settings → Agents → Attribution → turn off **Commit Attribution** and **PR Attribution** (UI only; not `settings.json`).

Still ship `.cursor/rules/no-co-authored-by.mdc` in every starter so agents do not add trailers themselves. Tell the user to confirm Attribution is off if trailers keep appearing.

## Default Decisions

Preserve these unless the user asks to change them:

| Area | Decision |
|------|----------|
| Credentials | Dev/test password in `credentials.yml.enc`; prod via `DATABASE_PASSWORD` |
| Secrets sharing | `master.key` via password manager; `RAILS_MASTER_KEY` in Kamal |
| Vendor gems | `/vendor` gitignored; history purged if accidentally committed |
| Tailwind | `@source "../../views/**/*.{erb,html}"` from `app/frontend/stylesheets/index.css` |
| Pry output | `amazing_print` in Gemfile; `AmazingPrint.pry!` in `~/.pryrc` |
| After migrations | `bundle exec rails db:migrate` then `bundle update brakeman` |
| CI | Runs on all branch pushes; security/lint jobs before test |
| Git author | Rewritten with `git rebase --root --exec 'git commit --amend --reset-author --no-edit'` when needed |
| Commit trailers | Never `Co-authored-by`; ship rule + disable Cursor Attribution (CLI + IDE) |
