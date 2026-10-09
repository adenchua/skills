# Project verify config: `.claude/verify.md`

`verify-app` reads this file. It holds the per-repo facts the skill cannot know: how to start the app, how to tell it's ready, and what not to touch. Keep entries short and as exact commands. Omit any line that doesn't apply, and leave a section out entirely if the project has no backend or no frontend.

Never write secret values into this file. Name the env var or file that holds them instead.

```markdown
# Verify config

## Checks
Build: <command>
Format check: <command>
Lint: <command>
Tests: <command>

## Backend
Start: <command, e.g. docker compose up -d db && npm run dev:api>
Ready: <command, e.g. curl -sf localhost:4000/health>
Base URL: <e.g. http://localhost:4000>
DB (read-only): <connection command, e.g. psql "$DATABASE_URL_READONLY">
Test users: <where credentials live, e.g. .env.example TEST_USER_*>
Seed / reset: <commands>
Stop: <command, if not just killing the process, e.g. docker compose down>

## Frontend
Start: <command, e.g. npm run dev:web>
URL: <e.g. http://localhost:5173>
Login as: <env var names for the test account>
Depends on: <backend must be running? mock server?>

## Don't touch
- <paths, endpoints, or flows that hit live services, e.g. /billing calls live Stripe>
```

When updating an existing file, change only the lines you learned something new about. Keep the other sections and anything the user wrote by hand.
