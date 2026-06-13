# Synadia Insights

- Website - https://www.synadia.com/insights
- Trial - https://www.synadia.com/insights/trial
- Docs - https://docs.synadia.com/insights
- Feedback - insights-report@synadia.com

## Agent Skills

This repo doubles as a [Claude Code plugin marketplace](https://code.claude.com/docs/en/plugin-marketplaces)
distributing [Agent Skills](https://code.claude.com/docs/en/skills) for working with Insights.

### Install in Claude Code

```
/plugin marketplace add synadia-io/insights
/plugin install insights-skills@synadia-insights
```

The `insights-skills` plugin currently bundles:

- **query-insights** — query an Insights database (DuckDB over NATS) with the `insights query` / `insights db`
  subcommands: schema discovery, epoch scoping, aggregation contexts, and audit-check macros.

### Install in Gemini CLI

The repo root is also a [Gemini CLI extension](https://geminicli.com/docs/extensions/). Installing it loads
the skill content as session context via `GEMINI.md`:

```
gemini extensions install synadia-io/insights
```

Manage it with `gemini extensions list`, `gemini extensions update insights-skills`, and
`gemini extensions remove insights-skills`.

### Use elsewhere

The skill itself is a single [`SKILL.md`](plugins/insights-skills/skills/query-insights/SKILL.md) following the
open [Agent Skills](https://agentskills.io/specification) format — it's the source of truth, and every install
path above points back to it. To use it with a tool that has no one-command installer, copy that skill folder
(`plugins/insights-skills/skills/query-insights/`) into the tool's skills directory:

| Tool | Where to copy the skill folder |
| --- | --- |
| **Claude Code** (personal) | `~/.claude/skills/query-insights/` |
| **Claude Agent SDK** | a `~/.claude/skills/` or project `.claude/skills/` dir discovered via `setting_sources` |
| **OpenAI Codex CLI** | `~/.agents/skills/query-insights/` (user) or `.agents/skills/` (project) |
| **Cursor** | `.cursor/skills/query-insights/` (per project) |
| **Claude Desktop / claude.ai** | zip the folder and upload under Settings → Capabilities → Skills |

> Note: the install commands for Claude Code and Gemini CLI are verified against current docs. Other tools read
> the open `SKILL.md` format but each has its own (evolving) discovery path — confirm the exact directory against
> your tool's docs before relying on it.

