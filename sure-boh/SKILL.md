---
name: sure-boh
description: "Verify a claim with fresh before/after evidence: restate it falsifiably, capture a baseline and a treatment run, compare the raw artifacts, and return VERIFIED, NOT VERIFIED, or INCONCLUSIVE. Use when the user says \"verify this\", \"prove it works\", \"did this fix it?\", \"show me the evidence\", \"is it actually faster?\", or wants a before/after repro for a bug fix, UI/CLI/API change, performance, or memory claim. Not for code review, style, or subjective claims."
argument-hint: "[claim to verify]"
---

# Sure Boh

Verification is not a recap. It proves or disproves a specific claim with repeatable evidence gathered in this session, not with reasoning about the code or with results remembered from earlier in the conversation.

Claim to verify: $ARGUMENTS

If no claim was given, use the most recent change or assertion in the conversation (for example, "the fix I just made stops the crash on empty input"). State the claim you picked in one line and proceed; the user can redirect you.

## When To Use

- The user asks "sure boh", "verify this", "prove it works", "did this fix it", or "show me the evidence".
- A bug fix needs a before/after repro.
- A UI, CLI, API, performance, or memory claim needs measurement.
- A test passes but the user-visible behavior still needs confirmation.

If the claim is subjective ("the code is cleaner"), propose the closest measurable version and verify that one. Ask the user only if no measurable version exists.

## Workflow

1. Restate the claim in falsifiable form: condition, metric, and threshold. "Faster" becomes "p50 of `npm run build` drops by at least 10% over 5 runs".
2. Pick the smallest local surface that can disprove it (see Local Surfaces).
3. Capture a baseline from the old state (see Getting The Baseline). The baseline must show the problem: if the repro already passes on the old state, it can't tell the two states apart, and the verdict is `INCONCLUSIVE` until you find a repro that does.
4. Capture treatment from the changed state with the same command, data, warmup, and environment.
5. Compare raw artifacts: numbers, screenshots, terminal transcripts, HTTP responses, profiles, heap snapshots, or test output. Read the actual output; never infer it from the exit code alone.
6. Return exactly one verdict: `VERIFIED`, `NOT VERIFIED`, or `INCONCLUSIVE`.

Do not edit the code under test while verifying. If verification exposes a bug, report it in the verdict; fix it only if the user asks, and then verify again from step 4.

## Getting The Baseline

Never disturb the user's working tree to get the old state: no `git stash`, `git checkout <file>`, `git reset`, or branch switching.

- **Committed or uncommitted change in a git repo:** check the old state out into a separate worktree and run the baseline there:
  `git worktree add --detach <scratch>/sure-boh/<claim-slug>/baseline-src <base>`,
  where `<base>` is `HEAD` for uncommitted changes, or the parent commit or `git merge-base HEAD <main-branch>` for committed ones. Install dependencies there if the project needs them. Remove it afterwards with `git worktree remove`.
- **Fix not made yet:** run the repro now, before any edit, and save that output as the baseline.
- **No git and the change is already applied:** no clean baseline exists. Use a documented prior result (a CI log, an issue repro) if there is one; otherwise the verdict is at best `INCONCLUSIVE` for any before/after claim. An absolute claim ("returns 200 for empty input") can still be verified on the treatment alone.

## Local Surfaces

Use whatever the session actually provides; these are options, not requirements.

- **Code behavior:** a focused unit or integration test, or a minimal repro script in the scratchpad. A new test must fail on the baseline and pass on the treatment.
- **CLI behavior:** run it through Bash and capture stdout, stderr, and the exit code to files.
- **TUI behavior:** drive it in `tmux` (`send-keys`, then `capture-pane -p`) and save the pane text.
- **UI behavior:** if the project has a skill for launching the app, or the `run` skill is available, use it. Otherwise use whichever browser tool is connected (Claude in Chrome, the built-in browser, or a Playwright MCP server) for screenshots, DOM or accessibility snapshots, and console logs. With no browser tool, fall back to HTTP-level checks and say the visual claim is unverified.
- **API behavior:** start the server with Bash `run_in_background`, wait until it is ready, and diff the `curl -sS -i` responses. Stop the server afterwards.
- **Performance:** the same machine, the same inputs, and at least 5 runs per side after a warmup, alternating baseline and treatment runs. Report the median and the range, not the best run. `hyperfine` is ideal if it is installed.
- **Memory:** heap snapshots or peak RSS (`/usr/bin/time -v` on Linux, `/usr/bin/time -l` on macOS) before and after the suspected operation.

Long runs: the Bash tool times out at 10 minutes. Put longer jobs in the background and wait for them rather than polling in a tight loop.

## Artifact Layout

Write artifacts under the session scratchpad directory if the harness provides one, otherwise under `/tmp`:

```text
<scratch>/verify-this/<claim-slug>/
├── claim.md        # falsifiable claim, base ref, environment
├── commands.sh     # exact commands run, so anyone can replay them
├── baseline/       # raw outputs from the old state
├── treatment/      # raw outputs from the changed state
├── diff/           # diffs and comparison tables
└── verdict.md
```

If artifacts may contain secrets or sensitive data (credentials, customer data, private HTTP bodies, heap dumps), keep only minimal inline evidence and redact before saving.

## Verdict Rules

- `VERIFIED`: the baseline shows the problem, the treatment doesn't, and the difference goes the predicted way by at least the claimed threshold with no obvious confound.
- `NOT VERIFIED`: the behavior is unchanged, moves the wrong way, or misses the threshold.
- `INCONCLUSIVE`: there is no valid baseline, the baseline doesn't reproduce the problem, the signal is noisy (the run ranges overlap), the measurement failed, or an environment difference invalidates the comparison.

## Output

Use this shape:

```text
VERIFIED | NOT VERIFIED | INCONCLUSIVE
Claim: <falsifiable claim>

Evidence:
<metric/artifact>: baseline=<...>, treatment=<...>, delta=<...>, threshold=<...>

Reasoning:
<one tight paragraph naming the evidence and any confounds>

Artifacts: <path to the artifact directory, or "inline only">
Replay: <the one command that reproduces the treatment result>
```

Do not soften a negative result. A clear `NOT VERIFIED` is useful.
