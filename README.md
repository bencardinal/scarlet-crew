# 🦀 The Scarlet Crew

A reusable **Claude Code plugin** that packages a crew of six specialized subagents you can deploy to any project, plus a `/assemble-crew` bootstrap command that tailors the crew to a new project's scope.

The crew is designed around **productive tension** — momentum vs. skepticism, novelty vs. simplicity — with an orchestrator that drives the team to converge:

| Agent | Role | Model | Access | The tension |
|-------|------|-------|--------|-------------|
| **Cork** | Quartermaster / orchestrator | `opus` | all tools | Drives convergence; coordination-first |
| **Chips** | Developer / builder | `inherit` | all tools | **Momentum** — ships working code |
| **Brass** | QA | `sonnet` | all tools | Verifies what actually works |
| **Barnacle** | Skeptic / devil's advocate | `opus` | all tools | **Skepticism** — "do we even need this?" |
| **Marco** | Newbie parrot | `haiku` | all tools | Surfaces unstated assumptions |
| **Knot** | Navigator / research | `haiku` | all tools | **Novelty** — brings options to the table |

No agent carries a `tools:` allowlist — every crew member inherits the full toolset (including any MCP servers you have configured). Role is enforced by the prompt, not by the tool list: Barnacle still argues rather than builds, and Cork still delegates rather than codes.

The loop: **parallel input → Cork synthesis → converge.** Cork decomposes a problem, fans work out to the crew in parallel rounds, distills the disagreements, and re-queues a tighter iteration until the crew lands a decision — with the dissent (especially Barnacle's) on the record.

**Is it worth it?** The premise — that structured tension produces better decisions than a single agent — is a hypothesis, not a proven result. It costs real tokens and coordination overhead, so reach for the crew on genuinely fuzzy or high-stakes problems where being wrong is expensive; for a single edit or a quick lookup, just use one agent directly. The honest evidence for whether it pays off is this repo's own git history — the running record of what the crew decided and how those calls held up.

> **Display name:** the plugin **id** is `scarlet-crew` (keep it stable). The human-facing name lives in one place — `displayName` at the top of [`.claude-plugin/plugin.json`](.claude-plugin/plugin.json). Change it there without touching anything else.

---

## Install

This repo is **both a plugin and its own single-plugin marketplace**, so you can install it locally without publishing anything.

```bash
# From anywhere, point Claude Code at this checkout as a marketplace:
/plugin marketplace add /path/to/scarlet-crew
#   (or, if your cwd is the repo:  /plugin marketplace add . )

# Install the plugin from that marketplace:
/plugin install scarlet-crew@scarlet-crew-marketplace
```

The same commands from your shell — `/plugin …` (the in-session slash command) and `claude plugin …` (the shell command) are two interfaces to the **same operation**, not different tools:

```bash
claude plugin marketplace add /path/to/scarlet-crew
claude plugin install scarlet-crew@scarlet-crew-marketplace
claude plugin validate /path/to/scarlet-crew      # sanity-check the manifest & components
```

Once installed, the crew is ready to run. Optionally, run `/assemble-crew` in a project to tailor the crew to it (writes shared context into that project's `CLAUDE.md`) — recommended, but not required; the crew works without it. Pass an optional brief: `/assemble-crew building a CLI for parsing flight logs`.

---

## Two ways to run the crew

### 1. Agent Teams — richer, peer-to-peer (experimental)

Agent Teams gives the crew a **shared task list** and lets teammates talk to each other, not just to the lead. It's gated behind an experimental flag:

```bash
# Enable once — in settings.json:
{ "env": { "CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS": "1" } }

# …or in your shell:
export CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS=1
```

Then make Cork the lead and have it spawn the others **as teammates from these agent definitions**:

> *"Act as Cork, the Scarlet Crew lead. Assemble the crew as a team — spawn the agent types `scarlet-crew:chips`, `scarlet-crew:brass`, `scarlet-crew:barnacle`, `scarlet-crew:marco`, and `scarlet-crew:knot` as teammates — and orchestrate work on: «your task here». Run parallel rounds and converge, with Barnacle's dissent recorded."*

Reference each crew member by its **namespaced agent type** (`scarlet-crew:<name>`) — the bare names won't resolve once the crew is installed as a plugin. The teammate honors that definition's `model` and appends its prompt as instructions.

### 2. Fallback — hub-and-spoke, no experimental flag

```bash
claude --agent cork
```

Cork runs as the main thread and spawns the others as **one-shot subagents** by their namespaced agent types — see [`agents/cork.md`](agents/cork.md) for the exact types. This is hub-and-spoke: **no agent-to-agent chatter and no nesting** — every perspective routes through Cork, so Cork synthesizes harder.

### Tradeoffs

- **Token cost:** Agent Teams uses **significantly more tokens** — each teammate has its **own context window**, and usage scales with the number of active teammates. The fallback is cheaper and more predictable.
- **Experimental:** Agent Teams is experimental with **known rough edges** — `/resume` and `/rewind` don't restore in-process teammates, task status can lag, shutdown can be slow, and a lead manages only one team with no nesting. The fallback avoids all of this at the cost of peer-to-peer collaboration.

---

## MCP servers (optional)

The crew runs fine with no MCP servers. **Optionally**, give it GitHub's official remote MCP server — useful for PR/issue work. Add it once at **user scope** (not via the plugin) so it's available in every project:

```bash
claude mcp add --scope user --transport http github https://api.githubcopilot.com/mcp/ \
  --header 'Authorization: Bearer ${GITHUB_PERSONAL_ACCESS_TOKEN}'
```

Export `GITHUB_PERSONAL_ACCESS_TOKEN` in your shell profile (a classic PAT with `repo` scope, or `gh auth token`). It has to be a user-scope PAT header for two reasons: GitHub's server lacks dynamic client registration so plain `/mcp` OAuth doesn't work ([github/github-mcp-server#1404](https://github.com/github/github-mcp-server/issues/1404)), and `${ENV_VAR}` expansion is broken in plugin-root `.mcp.json` so the plugin can't ship it ([anthropics/claude-code#9427](https://github.com/anthropics/claude-code/issues/9427)).

The plugin's [`.mcp.json`](.mcp.json) is an (empty) extension point for crew-wide servers that **don't** need secrets; add secret-free servers there under `mcpServers`. Don't put MCP config in agent frontmatter — a subagent's `mcpServers`/`skills` frontmatter is ignored when it runs as an Agent Teams teammate (teammates load MCP from project/user settings), so `.mcp.json` is what reaches the crew in both run modes.

---

## Layout

```
scarlet-crew/
├── .claude-plugin/
│   ├── plugin.json          # manifest — displayName is the one knob to rename the crew
│   └── marketplace.json     # single-plugin marketplace (source: ".")
├── agents/
│   ├── cork.md              # orchestrator / Agent Teams lead (opus)
│   ├── chips.md             # builder (inherit)
│   ├── brass.md             # QA (sonnet)
│   ├── barnacle.md          # skeptic (opus, memory: project)
│   ├── marco.md             # newbie (haiku, memory: project)
│   └── knot.md              # navigator / research (haiku, memory: project)
├── commands/
│   └── assemble-crew.md     # /assemble-crew bootstrap command
├── .mcp.json                # empty extension point for secret-free crew-wide MCP servers
├── CLAUDE.md                # guidance for Claude Code working in this repo
└── README.md
```

Barnacle, Marco, and Knot carry `memory: project`, so they accumulate project-specific insight across sessions in `.claude/agent-memory/<name>/` (version-controllable).

---

## License

MIT
