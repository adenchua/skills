---
name: how
description: Explain how something in the current codebase works — a subsystem, feature flow, runtime path, or overall architecture — by exploring the code and writing an onboarding-quality explanation; optionally follow with an architectural critique. Use when the user asks "how does X work?", "walk me through what happens when...", "how is X structured?", "explain the architecture of...", wants to onboard onto an area of the code, or asks what is architecturally wrong with / could be improved in an area. Not for reviewing a diff or PR.
---

# How

Answer "how does X work?" questions about the current codebase. Write at the level of a senior engineer onboarding onto a subsystem — enough to build a working mental model, not so much that it reads like annotated source code.

Two modes:

1. **Explain** (default) — explore the code and write a clear explanation.
2. **Critique** — when the user asks for architectural issues, problems, or improvements: explain first, then run independent critics and judge their findings.

## Explain Mode

### Step 1 — Orient and Assess

Parse what the user is asking about. They might ask about:

- a subsystem — "How does message virtualization work?"
- a feature flow — "How do we handle billing for on-demand usage?"
- an architectural overview — "How is the auth service structured?"
- a runtime trace — "Walk me through what happens when a user sends a message"

If the scope is ambiguous, pick the most likely interpretation and state it in one line before you start. Don't ask — explore, and let the user redirect you.

Then orient yourself with a few quick searches (Glob/Grep for the key terms, a look at the directory layout) to find where the subsystem lives and how big it is. Don't trace anything yet — just enough to decide the path:

- **Simple** — a single module, a small utility, a narrow question like "how does function X work". Go to Step 2a.
- **Complex** — spans many files or several services, a cross-cutting feature, a full architectural overview. Go to Step 2b.

When in doubt, take the simple path. If you hit a wall partway through, you can still fan out explorers for the parts you couldn't trace.

### Step 2a — Simple: Explore It Yourself

Trace the code directly: find the entry point, follow the call chain, read the definitions of the central types. Read the actual implementation — don't guess from names. Stop when you can describe the full path from trigger to effect without hand-waving. Then go to Step 3.

### Step 2b — Complex: Fan Out Explorers

Using what you learned while orienting, split the question into 2–4 exploration angles. Each angle should cover a distinct slice so the explorers aren't duplicating work. For "how does message virtualization work?" that might be:

- Explorer 1: the data model and state management
- Explorer 2: the rendering pipeline
- Explorer 3: the scroll/measurement infrastructure

Base the split on the code you saw, not on how such systems are usually built. Use 2 explorers for narrower questions and up to 4 for broad subsystems.

Launch all explorers in a **single message** so they run in parallel, using the Agent tool with:

- `subagent_type`: `Explore` (read-only: it has no edit tools)
- `model`: `sonnet`
- `description`: a 3–5 word label, e.g. "Explore render pipeline"
- `prompt`: built from [references/explorer-prompt.md](references/explorer-prompt.md), with the question, the explorer's angle, and any starting paths you found while orienting

Wait until every explorer has returned. Their findings will overlap and may contradict each other. Reconcile them, and when they disagree, read the code yourself to settle it. Fill any gaps they report as open questions, either by reading the code yourself or with one more targeted explorer. Then go to Step 3.

### Step 3 — Write the Explanation

Write the explanation yourself using the format below. Adapt it to the question; not every section is needed every time.

**Overview** — 1–2 paragraphs. What is this thing, what does it do, why does it exist. Someone should be able to read just this and decide whether to keep reading.

**Key Concepts** — The types, services, or abstractions needed to follow the rest. Brief definitions, not exhaustive.

**How It Works** — The core, and usually the longest section. Walk through the flow: what triggers it, what happens step by step, where data goes, what the decision points are. Use prose, not pseudocode. Reference files and functions as `path/to/file.ts:42` so the reader can jump to them. Only include a code block when a specific snippet is essential to the point.

When several components interact or data moves through stages, add a diagram. Default to a plain-text/ASCII diagram, since it renders in a terminal. Use mermaid only if the explanation is being written to a file or a viewer that renders it. Skip the diagram if prose already makes the flow clear.

**Where Things Live** — A short file/directory map: just what someone needs to find to start working here.

**Gotchas** — Non-obvious behavior, historical context that explains why something looks odd, known sharp edges. Omit if there's nothing worth calling out.

Style:

- Be concrete: "`OrderService.submit()` calls `PaymentGateway.charge()`", not "the service delegates to the gateway".
- When something is complex, explain *why* it's complex. Don't just describe the complexity.
- When something is simple, don't pad it.
- Use an analogy only if it genuinely helps.
- State gaps honestly: "I couldn't determine how X connects to Y" beats a plausible guess.

## Critique Mode

### Step 1 — Explain First

Run the full Explain flow above. You need to understand the architecture before you can critique it, and the explanation is what the critics start from.

### Step 2 — Spawn Critics

Launch the critics in a **single message**, all with `subagent_type: general-purpose`. Every critic is a Claude model, so their blind spots overlap. Most of the variety therefore comes from giving each critic different **primary lenses** from the rubric, and some from mixing models:

| Critic | `model` | Primary lenses |
|--------|---------|----------------|
| A | `opus` | Abstraction Fit, Boundary Discipline |
| B | `sonnet` | Data Model, Complexity vs. Value |
| C | `opus` | Evolution Readiness, Consistency |

If a lens pair clearly doesn't apply to this subsystem, drop that critic. For large, cross-service, or invariant-heavy architectures, also pass `effort: "high"` if your Agent tool supports it.

Build each prompt from [references/critic-prompt.md](references/critic-prompt.md), filling in:

1. The explanation from Step 1, so critics don't re-explore from scratch
2. The relevant file paths, as starting points
3. The full contents of [references/critique-rubric.md](references/critique-rubric.md)
4. That critic's primary lenses

`general-purpose` agents *can* edit files. The template tells critics they are read-only, so keep that line.

### Step 3 — Lead Judgment

You're a pragmatic lead, not an aggregator. Merge duplicate findings. A finding that several critics raised independently deserves more weight. Before ranking any finding you're unsure about, read the cited code yourself.

Sort the findings into:

- **Act on** — architectural problems worth fixing now
- **Consider** — real concerns, but the cost/benefit is unclear
- **Noted** — valid observations, low priority
- **Dismissed** — wrong, missing context, or a style preference (say why in one line)

Present the explanation first, then the verdict under a separate heading. List the findings by category. Give each one a short title, one or two sentences on the problem and its cost, and the key `file:line` evidence. The explanation must stand on its own, so a reader who only wants to understand the system never has to wade through the critique. If nothing survives, say the architecture is sound for what it does.
