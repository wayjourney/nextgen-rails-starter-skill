# nextgen-rails-starter-skill

Cursor/Codex agent skill for scaffolding production-minded Rails 8 starters.

## Stack

- Rails 8.1+, Ruby 4.0+, PostgreSQL
- Vite Ruby + Tailwind CSS 4 + Hotwire (Turbo + Stimulus)
- Solid Queue / Cache / Cable
- RSpec, GitHub Actions CI
- Rails credentials + Kamal deployment

## Install

**Cursor** — copy or symlink into `~/.cursor/skills/nextgen-rails-starter/`

**Codex** — already at `~/.codex/skills/nextgen-rails-starter/`, or clone there:

```bash
git clone https://github.com/wayjourney/nextgen-rails-starter-skill.git ~/.codex/skills/nextgen-rails-starter
```

## Usage

Reference the skill in chat: `@nextgen-rails-starter` (Cursor) or `$nextgen-rails-starter` (Codex).

## Structure

```
├── SKILL.md              # Main skill instructions
├── agents/openai.yaml    # Codex agent metadata
└── references/           # baseline, kamal, tailwind, no-co-authored-by.mdc
```

