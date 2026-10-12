---
name: update-agents-md
description: Run at the end of a session to save what was learned into the project's agent instruction files (AGENTS.md and/or CLAUDE.md). Proposes concise additions as diffs and applies only the ones the user approves.
disable-model-invocation: true
---

# Update Agent Instructions

Review this session for learnings about working in this codebase. Update the project's agent instruction file (AGENTS.md, CLAUDE.md, or both) with context that would help future agent sessions be more effective, whichever harness they run in.

## Step 1: Reflect

What context was missing that would have helped the agent work more effectively?

- Shell commands that were used or discovered
- Code style patterns followed
- Testing approaches that worked
- Environment/configuration quirks
- Warnings or gotchas encountered

## Step 2: Find the Instruction Files

```bash
find . \( -name node_modules -o -name .git -o -name vendor -o -name dist -o -name build \) -prune \
  -o \( -name AGENTS.md -o -name CLAUDE.md \) -print 2>/dev/null | head -20
```

Only shared, checked-in project files are in scope. Do not write to personal or global files (`CLAUDE.local.md`, `AGENTS.override.md`, `~/.claude/CLAUDE.md`, `~/.codex/AGENTS.md`).

For each directory with a file, check how AGENTS.md and CLAUDE.md relate (`ls -l` for symlinks; look for an `@AGENTS.md` line in CLAUDE.md), then choose the target:

| Setup | Where additions go |
| --- | --- |
| Neither file | Ask the user (see below) |
| One file only | That file. Also add one optional note to the proposal saying which harnesses won't see it (see Step 4) |
| Symlink (`CLAUDE.md -> AGENTS.md` or reverse) | The symlink target, never the link name |
| CLAUDE.md imports `@AGENTS.md`, or both are independent | Route by content (below) |

**Routing by content:**

- **Content for any harness → AGENTS.md.** Commands, architecture, gotchas, code style, environment, testing, workflow.
- **Claude-only content → CLAUDE.md.** Claude Code hooks, slash commands, `@file` imports, MCP server or subagent notes. In an import setup, put these below the `@AGENTS.md` line.

Put nested learnings in the file closest to the code they describe (e.g. `packages/api/AGENTS.md`), if one exists.

**If neither file exists**, ask the user to choose:

1. **Create CLAUDE.md only.** Works for Claude Code users only.
2. **Create AGENTS.md plus a CLAUDE.md containing just `@AGENTS.md`.** Every harness reads the same content.
3. **Do nothing.** Report the learnings in chat and leave the repo alone.

## Step 3: Draft Additions

**Keep it concise:** one line per concept. These files are loaded into the agent's prompt, so brevity matters.

Format: `<command or pattern>` - `<brief description>`

Avoid:

- Verbose explanations
- Obvious information
- One-off fixes unlikely to recur
- Duplicating content that already lives in the other file

## Step 4: Show Proposed Changes

For each addition:

````
### Update: ./AGENTS.md

**Why:** [one-line reason]

```diff
+ [the addition - keep it brief]
```
````

If only one of the two files exists, end the proposal with one optional line. For example: "Note: only CLAUDE.md exists, so Codex/Cursor/Copilot won't see these learnings. Optionally, move the content to AGENTS.md and replace CLAUDE.md with `@AGENTS.md`." Do not act on it unless asked.

## Step 5: Apply with Approval

Ask whether the user wants to apply the changes. Only edit files they approve.
