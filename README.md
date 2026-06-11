# 🦀 The Scarlet Crew

A reusable **Claude Code plugin** that packages a crew of six specialized subagents you can deploy to any project, plus a `/assemble-crew` bootstrap command that tailors the crew to a new project's scope.

The crew is designed around **productive tension** — momentum vs. skepticism, novelty vs. simplicity — with an orchestrator that drives the team to converge:

| Agent | Role | Model | Access | The tension |
|-------|------|-------|--------|-------------|
| **Cork** | Quartermaster / orchestrator | `opus` | read + spawn crew | Drives convergence; coordination-first |
| **Chips** | Developer / builder | `inherit` | full tools | **Momentum** — ships working code |
| **Brass** | QA | `sonnet` | read + run (Bash) | Verifies what actually works |
| **Barnacle** | Skeptic / devil's advocate | `opus` | read-only | **Skepticism** — "do we even need this?" |
| **Marco** | Newbie parrot | `haiku` | read-only | Surfaces unstated assumptions |
| **Knot** | Navigator / research | `haiku` | read + web/fetch | **Novelty** — brings options to the table |

The loop: **parallel input → Cork synthesis → converge.** Cork decomposes a problem, fans work out to the crew in parallel rounds, distills the disagreements, and re-queues a tighter iteration until the crew lands a decision — with the dissent (especially Barnacle's) on the record.

> **Display name:** the plugin **id** is `scarlet-crew` (keep it stable). The human-facing name lives in one place — `displayName` at the top of [`.claude-plugin/plugin.json`](.claude-plugin/plugin.json). Change it there (candidates: *The Scarlet Interruptor / Hallucinator / Orchestrator*) without touching anything else.

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

CLI equivalents:

```bash
claude plugin marketplace add /path/to/scarlet-crew
claude plugin install scarlet-crew@scarlet-crew-marketplace
claude plugin validate /path/to/scarlet-crew      # sanity-check the manifest & components
```

Once installed, run `/assemble-crew` in any project to tailor the crew (writes shared context into that project's `CLAUDE.md`). Pass an optional brief: `/assemble-crew building a CLI for parsing flight logs`.

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

> *"Act as Cork, the Scarlet Crew lead. Assemble the crew as a team — spawn Chips, Brass, Barnacle, Marco, and Knot as teammates from their agent definitions — and orchestrate work on: «your task here». Run parallel rounds and converge, with Barnacle's dissent recorded."*

When you reference a crew member by name, the teammate honors that definition's `tools` allowlist and `model` and appends its prompt as instructions.

### 2. Fallback — hub-and-spoke, no experimental flag

```bash
claude --agent cork
```

Cork runs as the main thread and spawns the others as **one-shot subagents** (Cork's `tools` includes `Agent(chips, brass, barnacle, marco, knot)`). This is hub-and-spoke: **no agent-to-agent chatter and no nesting** — every perspective routes through Cork, so Cork synthesizes harder.

### Tradeoffs

- **Token cost:** Agent Teams uses **significantly more tokens** — each teammate has its **own context window**, and usage scales with the number of active teammates. The fallback is cheaper and more predictable.
- **Experimental:** Agent Teams is experimental with **known rough edges** — `/resume` and `/rewind` don't restore in-process teammates, task status can lag, shutdown can be slow, and a lead manages only one team with no nesting. The fallback avoids all of this at the cost of peer-to-peer collaboration.

---

## MCP servers

The crew expects **GitHub's official remote MCP server**. Add it once at **user scope** so it's available to the crew in every project:

```bash
claude mcp add --scope user --transport http github https://api.githubcopilot.com/mcp/ \
  --header 'Authorization: Bearer ${GITHUB_PERSONAL_ACCESS_TOKEN}'
```

Export `GITHUB_PERSONAL_ACCESS_TOKEN` in your shell profile — a classic PAT (`ghp_…`, minimum `repo` scope) works best; a `gh auth token` OAuth token also works. Two constraints force this shape:

- **Plain `/mcp` OAuth sign-in doesn't work** — GitHub's remote MCP server lacks dynamic client registration ([github/github-mcp-server#1404](https://github.com/github/github-mcp-server/issues/1404)), so it must be a PAT header.
- **The plugin can't ship the server itself** — `${ENV_VAR}` expansion is broken in plugin-root `.mcp.json` ([anthropics/claude-code#9427](https://github.com/anthropics/claude-code/issues/9427)); the placeholder is sent to GitHub literally and auth fails with HTTP 400. User- and project-scope configs expand it correctly.

The plugin's [`.mcp.json`](.mcp.json) is kept (empty) as the extension point for crew-wide servers that **don't** need secrets — once #9427 is fixed, the GitHub server can move back in.

**Adding more servers** (Home Assistant, etc.) — add another entry under `mcpServers`. For example, a generic HTTP/SSE server:

```jsonc
{
  "mcpServers": {
    "home-assistant": {
      "type": "sse",
      "url": "http://homeassistant.local:8123/mcp_server/sse",
      "headers": { "Authorization": "Bearer ${HASS_TOKEN}" }
    }
  }
}
```

> ### ⚠️ Important: why MCP lives in `.mcp.json`, not the agent files
>
> A subagent's `skills` and `mcpServers` **frontmatter is not applied when that subagent runs as an Agent Teams teammate**. Teammates load skills and MCP servers from your **project and user settings**, the same as a regular session. That's exactly why crew-wide MCP belongs in `.mcp.json` (which the plugin contributes to project config) rather than being declared per-agent — so it's available to the crew **whether they run as subagents or as teammates**.

---

## Layout

```
scarlet-crew/
├── .claude-plugin/
│   ├── plugin.json          # manifest — displayName is the one knob to rename the crew
│   └── marketplace.json     # single-plugin marketplace (source: ".")
├── agents/
│   ├── cork.md              # orchestrator / Agent Teams lead (opus)
│   ├── chips.md             # builder (inherit, full tools)
│   ├── brass.md             # QA (sonnet, read + Bash)
│   ├── barnacle.md          # skeptic (opus, read-only, memory: project)
│   ├── marco.md             # newbie (haiku, read-only, memory: project)
│   └── knot.md              # navigator / research (haiku, read + web, memory: project)
├── commands/
│   └── assemble-crew.md     # /assemble-crew bootstrap command
├── .mcp.json                # crew-wide MCP servers (GitHub example)
└── README.md
```

Barnacle, Marco, and Knot carry `memory: project`, so they accumulate project-specific insight across sessions in `.claude/agent-memory/<name>/` (version-controllable).

---

## License

MIT
