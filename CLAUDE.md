# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this repo is

A Claude Code **plugin** that packages six specialized subagents (the "Scarlet Crew") plus a `/assemble-crew` bootstrap command. There is no application code, build, or test suite — everything is markdown agent/command definitions and JSON manifests.

## Commands

```bash
claude plugin validate .                                   # sanity-check manifest & components after edits
claude plugin marketplace add .                            # install locally for manual testing
claude plugin install scarlet-crew@scarlet-crew-marketplace
```

Run `validate` after changing any frontmatter or manifest.

## Architecture

The repo is **both the plugin and its own single-plugin marketplace**: `.claude-plugin/plugin.json` is the plugin manifest, and `.claude-plugin/marketplace.json` is a marketplace whose only entry points at `./`. This allows local installs without publishing. Keep the plugin **id** (`name: scarlet-crew`) stable; the human-facing name is changed only via `displayName` in `plugin.json`.

### The crew design

The six agents in `agents/` are designed around **productive tension** — momentum (Chips) vs. skepticism (Barnacle), novelty (Knot) vs. simplicity — with Cork as the convergence engine. The operating loop is: parallel input → Cork synthesis → converge, with dissent recorded. Each agent's frontmatter encodes its role deliberately:

The payoff — that this structured tension yields better decisions than a single agent — is an **unproven hypothesis**, and the crew costs real tokens and coordination overhead. It's worth that cost on genuinely fuzzy or high-stakes problems; for single edits or quick lookups, use one agent directly. The running evidence is this repo's own git history.

- **No tool allowlists — role is enforced by the prompt.** No agent carries a `tools:` key, so every crew member inherits the full toolset, MCP servers included (allowlists silently blocked MCP tools, which is why they were dropped in 0.2.0). The constraint that keeps Barnacle skeptical and Cork coordination-first now lives entirely in each agent's prompt — `agents/cork.md`'s *Judgment* section is the reference for when Cork acts directly vs. delegates. If you reintroduce a `tools:` key, know that you're re-breaking MCP access for that agent.
- **Models are cost/depth choices.** Cork and Barnacle run `opus` (synthesis/deep critique), Brass `sonnet`, Marco and Knot `haiku` (cheap by design), Chips `inherit`.
- **`memory: project`** on Barnacle, Marco, and Knot lets them accumulate per-project insight in the host project's `.claude/agent-memory/<name>/`.

### Two execution modes (referenced throughout the docs)

1. **Agent Teams** (experimental, `CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS=1`): Cork as lead, peers talk to each other.
2. **Fallback hub-and-spoke**: `claude --agent cork` — Cork spawns the others as one-shot subagents; no cross-talk, no nesting.

Agent definitions (especially `cork.md`) explicitly address both modes; keep that dual framing when editing them.

### MCP placement constraint

Crew-wide MCP servers live in `.mcp.json` at the plugin root, **not** in agent frontmatter — a subagent's `mcpServers`/`skills` frontmatter is ignored when it runs as an Agent Teams teammate. Don't move MCP config into the agent files.

**Exception — servers needing secrets:** `${ENV_VAR}` expansion is broken in plugin-root `.mcp.json` (anthropics/claude-code#9427 — placeholders are sent literally), so any env-var-dependent server (e.g. the optional GitHub MCP server) must be added at user scope instead. See the README's "MCP servers (optional)" section for the full rationale and the `claude mcp add` command — don't duplicate it here. Don't put env-var-dependent config back into `.mcp.json` until that bug is fixed.

### `/assemble-crew`

`commands/assemble-crew.md` bootstraps the crew in a *host* project: it infers scope from that repo, asks at most 2–3 clarifying questions, and writes a Scarlet Crew section into the host's `CLAUDE.md` (merge, never clobber).
