# Backend surface

Use for servers, APIs, workers, and database changes.

## Pre-check gaps

In addition to the common gaps, record these:

- An OpenAPI or Swagger spec that doesn't match the changed endpoints.
- Secrets or tokens printed in new log lines.

## Test matrix

- **Happy path:** the expected request and response.
- **Invalid input:** missing fields, wrong types, out-of-range values. Expect a 4xx in the project's error format, not a 500.
- **Auth:** no token → 401, wrong user or role → 403, where the endpoint is protected.
- **Edge cases** relevant to the change: empty lists, duplicates, pagination bounds, idempotent retries, concurrent writes.
- **Regression:** one neighbouring endpoint or flow that shares the changed code.

## Start the server

Before sending requests, run the readiness check from `.claude/verify.md`. If there isn't one, poll a health endpoint or the port, for example `curl -sf localhost:PORT/health` in a loop.

## Evidence

- **Requests:** use `curl -sS -i` so the status line, headers, and body are all captured. Save each response to a file. Check the status code, the body shape (against the OpenAPI spec if there is one), and the error format.
- **Database:** use a local or disposable database. Run read-only `SELECT` queries before and after each mutating request. Check that the expected rows exist, nothing extra was written, and a failed request left no partial writes.
- **Logs:** after the scenarios, read the server log for stack traces, unexpected warnings, and leaked secrets.
- **Background work** (queues, cron, workers): trigger it, then confirm the effect (a row, a message, a file). The enqueue call alone is not evidence.

Save the exact commands to `commands.sh` so the run can be replayed.

## Failure conditions

The verdict is `NOT VERIFIED` if any scenario returns the wrong status or body, leaves the wrong database state, or puts an unhandled error in the server log.
