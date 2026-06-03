---
description: Bootstrap the Scarlet Crew for the current project — infer scope from the repo (and an optional brief), ask a few sharp questions, write the shared context into CLAUDE.md, and print how to start a session.
argument-hint: "[optional one-line brief about the project or the task at hand]"
---

You are setting up the **Scarlet Crew** for this project. The crew is a set of specialized subagents — Cork (orchestrator), Chips (builder), Brass (QA), Barnacle (skeptic), Marco (newbie), Knot (navigator) — built around productive tension and driven to convergence by Cork. Your job right now is to tailor that crew to *this* project by establishing the shared context every crew member should load.

Optional brief from the user: **$ARGUMENTS**

Work through these steps:

## 1. Infer the project's scope

Read the repository to understand what this project actually is. Look at, as available:

- `README.md`, `CLAUDE.md`, docs, and any `CONTRIBUTING`/`ARCHITECTURE` files
- Manifest/lockfiles to pin the stack and tooling: `package.json`, `pyproject.toml`, `Cargo.toml`, `go.mod`, `pom.xml`, `Gemfile`, etc.
- Directory layout, entry points, and config (CI files, linters, formatters, test setup)
- The brief in `$ARGUMENTS`, if provided

From this, form a working understanding of: **purpose**, **tech stack & tooling**, **conventions**, **constraints**, and **primary use cases**. Prefer evidence from the repo over assumption.

## 2. Ask at most 2–3 sharp clarifying questions — only if scope is genuinely ambiguous

If the repo plus the brief already make the scope clear, **skip this step**. Otherwise ask only the highest-leverage questions whose answers would change how the crew operates — e.g. the primary goal for this engagement, a hard constraint (perf, compatibility, deadline, "don't touch X"), or which area to focus on. Do not interrogate; 2–3 questions maximum. Use the question tool so the user can pick quickly.

## 3. Write or update CLAUDE.md

Create `CLAUDE.md` at the project root (or update it in place — **merge, don't clobber** existing content; preserve anything already there and add/refresh a clearly marked Scarlet Crew section). Include:

- **Project goal** — what this project is for, in a sentence or two.
- **Stack & tooling** — languages, frameworks, key libraries, how to build/test/run.
- **Conventions** — code style, naming, structure, testing norms you inferred or were told.
- **Constraints** — hard limits, things not to touch, performance/compat/security requirements, deadlines.
- **The crew's operating loop** — state it explicitly so every member shares the model:
  > **Parallel input → Cork synthesis → converge.** Cork decomposes the problem and fans work out to the crew in parallel rounds; each member contributes their slice (Chips builds, Knot scouts options, Barnacle attacks, Marco surfaces assumptions, Brass verifies); Cork distills the tensions and re-queues a tighter iteration until the crew converges on a decision with the dissent recorded.
- **The crew roster** — one line each on Cork, Chips, Brass, Barnacle, Marco, Knot and their role in the tension (momentum vs. skepticism, novelty vs. simplicity).

Keep it concise and skimmable — this is shared context the whole crew loads, not an essay. Show the user the section you wrote.

## 4. Print how to start a working session

Tell the user both ways to launch the crew, and the tradeoff:

**A) Agent Teams (richer, peer-to-peer + shared task list — experimental):**
```bash
# Enable once (settings.json env, or your shell):
export CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS=1
```
Then, in a session, make Cork the lead and have it spawn the rest as teammates from these definitions, e.g.:
> "Act as Cork, the Scarlet Crew lead. Assemble the crew as a team — spawn Chips, Brass, Barnacle, Marco, and Knot as teammates from their agent definitions — and orchestrate work on: «my task». Run parallel rounds and converge."

**B) Fallback (no experimental flag — hub-and-spoke):**
```bash
claude --agent cork
```
Cork runs as the main thread and spawns the others as one-shot subagents (no agent-to-agent chatter, no nesting; every perspective routes through Cork).

Note the tradeoff: **Agent Teams uses significantly more tokens** — each teammate has its own context window — and is **experimental with known rough edges** (session resumption, task-status lag, shutdown). The fallback is cheaper and more predictable but loses peer-to-peer collaboration.

Finish with a one-paragraph summary of what you learned about the project and confirmation that the crew is assembled.
