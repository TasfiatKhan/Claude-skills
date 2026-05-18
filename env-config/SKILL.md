---
name: env-config
description: Set up or audit environment configuration — .env files, secrets separation, and cross-service config handling. Use when the user says "set up env config", "handle secrets", "add a new env var", or "audit environment setup".
---

You are setting up or auditing environment configuration for a project.

## Core principles

- **Never commit secrets.** API keys, DB passwords, tokens — always in `.env`, never hardcoded
- **One `.env.example` per service** — committed to git, contains all keys with blank or dummy values and a comment explaining each
- **One `.env` per service** — gitignored, contains real values, never committed
- **Each service reads only its own vars** — backend doesn't reach into frontend config and vice versa

## Structure to enforce

```
project/
  backend/
    .env              # real values — gitignored
    .env.example      # committed — blank values, comments explaining each
  frontend/
    .env              # real values — gitignored
    .env.example      # committed — blank values, comments explaining each
```

## .env.example format

Each variable should have a comment explaining what it is and where to get it:

```
# Django secret key — generate with: python -c "import secrets; print(secrets.token_hex(50))"
SECRET_KEY=

# PostgreSQL connection
DB_NAME=witly
DB_USER=witly
DB_PASSWORD=
DB_HOST=db
DB_PORT=5432

# Anthropic API key — get from console.anthropic.com
ANTHROPIC_API_KEY=

# OpenAI API key — used for Whisper transcription only
OPENAI_API_KEY=
```

## When adding a new env var

1. Add it to `.env` with the real value
2. Add it to `.env.example` with a blank value and a comment
3. Add it to the service's config reader (Django `settings.py`, `decouple`, etc.)
4. Document it in README or CLAUDE.md if it's non-obvious

## Common mistakes to catch

- Hardcoded API keys or passwords anywhere in source code
- `.env` files accidentally committed (check `.gitignore`)
- Vars defined in one service but needed in another (usually a design smell)
- Missing vars in `.env.example` that are required to run the project
- `DEBUG=True` or development DB credentials left in production config

## When auditing

1. Read `.gitignore` — confirm `.env` is listed
2. `git log --all -- "**/.env"` — check if `.env` was ever committed
3. `grep -r "sk-\|AKIA\|password=" --include="*.py" --include="*.ts"` — scan for hardcoded secrets
4. Compare `.env.example` against actual `.env` — flag any missing keys
