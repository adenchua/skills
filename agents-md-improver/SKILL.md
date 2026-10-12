---
name: agents-md-improver
description: Audit and improve agent instruction files (AGENTS.md and CLAUDE.md) in a repository. Scans for every AGENTS.md / CLAUDE.md, works out how they relate (symlink, `@AGENTS.md` import, or independent), scores them against a quality rubric, outputs a report, then makes targeted updates with approval. Use when the user asks to check, audit, review, update, improve, or fix AGENTS.md or CLAUDE.md, or mentions "agent instructions", "CLAUDE.md maintenance", "AGENTS.md maintenance", or "project memory optimization".
---

# Agent Instructions Improver

Audit, evaluate, and improve the agent instruction files in a codebase so that coding agents — Claude Code (`CLAUDE.md`) or any harness that reads `AGENTS.md` (Codex, Cursor, Copilot, Gemini CLI, etc.) — have the right project context.

**This skill can write to AGENTS.md / CLAUDE.md files.** It always presents a quality report first and only edits after user approval.

## Workflow

### Phase 1: Discovery

Find every instruction file in the repository:

```bash
find . \( -name node_modules -o -name .git -o -name vendor -o -name dist -o -name build \) -prune \
  -o \( -name AGENTS.md -o -name CLAUDE.md \) -print 2>/dev/null | head -50
```

Only shared, checked-in project files are in scope. Ignore personal and global files (`CLAUDE.local.md`, `AGENTS.override.md`, `~/.claude/CLAUDE.md`, `~/.codex/AGENTS.md`).

**Then classify each directory that has a file.** The relationship is per directory, since nested packages can have their own pair.

```bash
# Symlinks: one name points at the other
find . -name node_modules -prune -o \( -name AGENTS.md -o -name CLAUDE.md \) -type l -exec ls -l {} \;
# Imports: a line such as `@AGENTS.md` in CLAUDE.md pulls the other file in
find . -name node_modules -prune -o -name CLAUDE.md -exec grep -Hn '^@' {} +
```

| Setup | How to recognise it | Treat as |
| --- | --- | --- |
| Neither file | Discovery found nothing | See [No files found](#no-files-found) |
| AGENTS.md only | — | One file; note that Claude Code won't read it |
| CLAUDE.md only | — | One file; note that other harnesses won't read it |
| Symlink | `CLAUDE.md -> AGENTS.md` (or reverse) | One file: the symlink target |
| Import | CLAUDE.md contains `@AGENTS.md` | One unit: AGENTS.md plus CLAUDE.md's Claude-only extras |
| Independent | Both exist, no symlink or import | Two files that can drift apart |

**Locations:**

| Type | Location | Purpose |
| --- | --- | --- |
| Project root | `./AGENTS.md`, `./CLAUDE.md` | Primary project context, shared with the team |
| Package-specific | `./packages/*/AGENTS.md` etc. | Module-level context in monorepos |
| Subdirectory | Any nested location | Feature/domain-specific context |

#### No files found

Do not create anything on your own. Ask the user to choose:

1. **Create CLAUDE.md only.** Works for Claude Code users only.
2. **Create AGENTS.md plus a CLAUDE.md that imports it.** AGENTS.md holds the content and CLAUDE.md contains just `@AGENTS.md`, so every harness reads the same content.
3. **Do nothing.** Just report that the repository has no instruction file.

If they choose 1 or 2, draft the content from [references/templates.md](references/templates.md) and continue from Phase 4.

### Phase 2: Quality Assessment

Evaluate each file, or each unit from the table above, against the quality criteria. See [references/quality-criteria.md](references/quality-criteria.md) for detailed rubrics.

Score the **effective content** an agent would load. For an import or symlink setup, score AGENTS.md and CLAUDE.md together as one unit. A CLAUDE.md that is only `@AGENTS.md` is a valid pointer, not an empty file.

**Quick Assessment Checklist:**

| Criterion | Weight | Check |
| --- | --- | --- |
| Commands/workflows documented | High | Are build/test/deploy commands present? |
| Architecture clarity | High | Can an agent understand the codebase structure? |
| Non-obvious patterns | Medium | Are gotchas and quirks documented? |
| Conciseness | Medium | No verbose explanations or obvious info? |
| Currency | High | Does it reflect the current codebase state? |
| Actionability | High | Are instructions executable, not vague? |

**Quality Scores:**

- **A (90-100)**: Comprehensive, current, actionable
- **B (70-89)**: Good coverage, minor gaps
- **C (50-69)**: Basic info, missing key sections
- **D (30-49)**: Sparse or outdated
- **F (0-29)**: Missing or severely outdated

### Phase 3: Quality Report Output

**ALWAYS output the quality report BEFORE making any updates.**

Format:

```
## Agent Instructions Quality Report

### Summary
- Files found: X (AGENTS.md: X, CLAUDE.md: X)
- Setup: [AGENTS.md only | CLAUDE.md only | symlink | import | independent]
- Average score: X/100
- Files needing update: X

### File-by-File Assessment

#### 1. ./AGENTS.md + ./CLAUDE.md (Project Root, CLAUDE.md imports AGENTS.md)
**Score: XX/100 (Grade: X)**

| Criterion | Score | Notes |
|-----------|-------|-------|
| Commands/workflows | X/20 | ... |
| Architecture clarity | X/20 | ... |
| Non-obvious patterns | X/15 | ... |
| Conciseness | X/15 | ... |
| Currency | X/15 | ... |
| Actionability | X/15 | ... |

**Issues:**
- [List specific problems]

**Recommended additions:**
- [List what should be added]

#### 2. ./packages/api/AGENTS.md (Package-specific)
...

### Cross-Harness Notes
- [Only if relevant: see below]
```

**Cross-Harness Notes** (include only what applies; these never affect the score):

- **Single file:** one optional line saying which harnesses won't see the file. For example: "Only CLAUDE.md found, so Codex/Cursor/Copilot won't see this context. Optionally, move the content to AGENTS.md and replace CLAUDE.md with `@AGENTS.md`." Never apply this without being asked.
- **Independent files:** list content that is duplicated or contradictory between AGENTS.md and CLAUDE.md, as a drift risk. Do not merge them unless the user asks.

### Phase 4: Targeted Updates

After outputting the quality report, ask the user for confirmation before updating.

**Where each edit goes:**

| Setup | Target |
| --- | --- |
| Single file | That file |
| Symlink | The file the symlink points to (never write through the link name) |
| Import or independent | Route by content (below) |

**Routing by content** (import and independent setups):

- **Content for any harness → AGENTS.md.** Commands, architecture, gotchas, code style, environment, testing, workflow. This is almost everything.
- **Claude-only content → CLAUDE.md.** Claude Code hooks, slash commands, `@file` imports, MCP server or subagent notes. In an import setup, put these below the `@AGENTS.md` line.

**Update Guidelines (Critical):**

1. **Propose targeted additions only.** Focus on genuinely useful info:
   - Commands or workflows discovered during analysis
   - Gotchas or non-obvious patterns found in code
   - Package relationships that weren't clear
   - Testing approaches that work
   - Configuration quirks

2. **Keep it minimal.** Avoid:
   - Restating what's obvious from the code
   - Generic best practices already covered
   - One-off fixes unlikely to recur
   - Verbose explanations when a one-liner suffices

3. **Show diffs.** For each change, show:
   - Which file to update, and why it goes in that file if both exist
   - The specific addition (as a diff or quoted block)
   - A brief explanation of why this helps future sessions

See [references/update-guidelines.md](references/update-guidelines.md) for examples.

**Diff Format:**

`````markdown
### Update: ./AGENTS.md

**Why:** Build command was missing, causing confusion about how to run the project.

````diff
+ ## Quick Start
+
+ ```bash
+ npm install
+ npm run dev  # Start development server on port 3000
+ ```
````
`````

### Phase 5: Apply Updates

After user approval, apply the changes with your file-editing tool, keeping the existing content structure. Only edit files the user approved.

## Templates

See [references/templates.md](references/templates.md) for templates by project type.

## Common Issues to Flag

1. **Stale commands**: Build commands that no longer work
2. **Missing dependencies**: Required tools not mentioned
3. **Outdated architecture**: File structure that has changed
4. **Missing environment setup**: Required env vars or config
5. **Broken test commands**: Test scripts that have changed
6. **Undocumented gotchas**: Non-obvious patterns not captured
7. **Drift between files**: AGENTS.md and CLAUDE.md disagree (independent setup only)
8. **Broken pointer**: CLAUDE.md imports or links to an AGENTS.md that doesn't exist

## What Makes Great Agent Instructions

**Key principles:**
- Concise and human-readable; the file is loaded into the agent's context, so every line costs tokens
- Actionable commands that can be copy-pasted
- Project-specific patterns, not generic advice
- Non-obvious gotchas and warnings
- One source of truth: shared content in one place, not duplicated across AGENTS.md and CLAUDE.md

**Recommended sections** (use only what's relevant):
- Commands (build, test, dev, lint)
- Architecture (directory structure)
- Key Files (entry points, config)
- Code Style (project conventions)
- Environment (required vars, setup)
- Testing (commands, patterns)
- Gotchas (quirks, common mistakes)
- Workflow (when to do what)
