---
name: verify-app
description: "Verify that a feature works by running the app: start it locally, exercise the backend with real requests and database checks, and/or drive the frontend in a browser with screenshots, console, and network checks. Returns VERIFIED, NOT VERIFIED, or INCONCLUSIVE with evidence. Use when the user asks \"does it work?\", \"does the backend work?\", \"verify the API\", \"test this endpoint\", \"verify the UI\", \"click through it\", \"smoke test\", \"e2e check\", or wants confirmation that a backend, API, UI, page, or full-stack change behaves correctly. Not for code review. For before/after claims (\"did this fix it?\", \"is it faster?\") use sure-boh."
argument-hint: "[backend|frontend] [feature, endpoint, page, or flow]"
---

# Verify App

Prove that a feature behaves correctly with evidence gathered in this session: real requests and responses, database rows, screenshots, console and network logs. Reasoning about the code is not evidence.

Arguments: $ARGUMENTS

## Target and surface

The **target** is the feature, endpoint, page, or flow to verify. If none was given, infer it from `git diff HEAD --stat`, recent commits, or the most recent change in the conversation.

The **surface** decides which reference to read:

| Surface | When | Read |
|---|---|---|
| Backend | first argument is `backend`, or only server, API, worker, or schema code changed | [references/backend.md](references/backend.md) |
| Frontend | first argument is `frontend`, or only UI code changed | [references/frontend.md](references/frontend.md) |
| Both | the change spans both, or the target is a user flow that calls changed endpoints | both, backend first |

State the target and surface in one line and proceed; ask only if nothing can be inferred.

## Rules

- Do not edit the code under test. Report problems in the verdict; fix them only if the user asks, then verify again.
- Run against local or dev environments only. Never send requests, submit forms, or write data against production or shared environments unless the user explicitly says so.
- Ask the user for credentials or test accounts that aren't already configured locally. Never invent them or extract them from secret stores.
- Use only evidence from this run, not results remembered from earlier in the conversation. Read the actual output; never infer success from an exit code.
- Clean up afterwards: stop every server you started, close tabs you opened, and delete test data you created.

## Workflow

Copy this checklist and track progress:

```text
- [ ] 1. Load project setup
- [ ] 2. Pre-check
- [ ] 3. Test matrix
- [ ] 4. Start the app
- [ ] 5. Run scenarios and capture evidence
- [ ] 6. Clean up
- [ ] 7. Report
```

Steps 3–6 have surface-specific details in the reference files.

### 1. Load project setup

Find out how this repo is built, started, and seeded, what URLs it serves, and which test accounts to use. Use the first source that answers it:

1. `.claude/verify.md` at the repo root. If it exists, follow its commands and respect its "Don't touch" list. Its format is in [references/project-config.md](references/project-config.md).
2. A project skill that already launches the app.
3. The repo itself: README, CLAUDE.md, `package.json` scripts, Makefile, `docker-compose.yml`, Procfile, `.env.example`.
4. Ask the user. Do this only for what you still can't work out, such as test credentials or which database is safe to use.

Note which facts you had to discover; step 7 offers to save them.

### 2. Pre-check (check only, never modify)

Use whatever the project already has; skip what it doesn't.

1. Install dependencies and build, including the type check if there is one. A build failure ends verification with `NOT VERIFIED`.
2. Run the formatter in check mode (`prettier --check`, `biome check`, `ruff format --check`, `gofmt -l`, `cargo fmt --check`, ...) and the linter. Record failures; don't stop.
3. Run the existing tests. Record failures; don't stop.
4. Record gaps as findings, only where the repo has the convention: no tests for the changed code, no CHANGELOG entry, plus the surface-specific gaps in the reference files.

### 3. Test matrix

Write the scenarios down before running anything. Every matrix covers the happy path, invalid input, and one regression check on a neighbouring feature that shares the changed code; the reference files list the rest. For both surfaces, write one matrix with backend scenarios first.

### 4. Start the app

Use the start command from step 1. Run long-lived processes with Bash `run_in_background`, redirecting output to a log file in the artifact directory, and wait for the readiness check (or poll the URL or port) with a 60-second limit before testing. For both surfaces, start the backend first and point the frontend at it.

If the app never comes up, read the log: the verdict is `NOT VERIFIED` if it crashed, `INCONCLUSIVE` if the environment is missing something (a database, env vars, credentials).

### 5. Run scenarios and capture evidence

Follow the reference file for each surface. Save every raw artifact as you go.

### 6. Clean up

Stop the servers you started (with the stop command from step 1 if there is one), close tabs you opened, delete test rows, accounts, or fixtures you created, and leave the working tree as you found it.

### 7. Report

Save artifacts under `<scratchpad>/verify-app/<target-slug>/` (the session scratchpad if the harness provides one, otherwise `/tmp`): `commands.sh` or `steps.md` with exactly what was run, plus raw responses, query results, screenshots, and logs. Redact tokens, passwords, and personal data before saving.

If step 1 had to discover or ask for anything, offer after the report to save it to `.claude/verify.md`, following [references/project-config.md](references/project-config.md). Show the lines you would add and write them only if the user agrees. Never write secret values into the file.

## Verdicts

- `VERIFIED`: the build passes and every scenario in the matrix matched the expected behaviour, with captured evidence.
- `NOT VERIFIED`: the build fails, or any scenario produced a wrong result (see the failure conditions in each reference file).
- `INCONCLUSIVE`: a scenario couldn't run (missing credentials, service, data, or browser tool), or the result is ambiguous. Name what is missing.

Pre-check findings (lint, format, missing tests or docs) are reported but don't change the verdict on their own. Don't soften a negative result.

## Output

```text
VERIFIED | NOT VERIFIED | INCONCLUSIVE
Target: <what was verified>   Surface: backend | frontend | both

Pre-check: build ✓ | types ✓ | format ✓ | lint ✗ (2 warnings) | tests ✓ (48 passed)
Gaps: <missing tests / CHANGELOG / spec drift, or "none">

Scenarios:
  ✓ <scenario> — <evidence: status and key fields, DB rows, screenshot path>
  ✗ <scenario> — expected <x>, got <y>; <evidence>

Artifacts: <artifact dir, or "inline only">
Replay: <one command or steps.md path that reproduces the key result>
```

## Example

```text
NOT VERIFIED
Target: login with rate limiting and "remember me"   Surface: both

Pre-check: build ✓ | types ✓ | format ✓ | lint ✓ | tests ✓ (112 passed)
Gaps: no test covers the 429 path; CHANGELOG not updated

Scenarios:
  ✓ [api] valid credentials — 200, body has `token` (JWT, exp 1h), sessions row inserted
  ✓ [api] wrong password — 401 {"error":"invalid_credentials"}, no sessions row
  ✗ [api] 6th attempt within 1 min — expected 429, got 401; no Retry-After header
  ✓ [ui] valid login — redirected to /dashboard, "Welcome, Test User" shown; 01-login.png
  ✓ [ui] empty form — submit disabled, both fields marked required; 02-empty.png
  ✗ [ui] 375px viewport — "remember me" label overlaps the submit button; 03-mobile.png
  ✓ [ui] regression: logout — returns to /login, session cookie cleared

Artifacts: /tmp/claude-.../verify-app/login-rate-limit/
Replay: bash /tmp/claude-.../verify-app/login-rate-limit/commands.sh
```
