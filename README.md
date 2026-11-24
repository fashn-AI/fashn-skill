# FASHN Agent Skill

An [agent skill](https://www.skills.sh/docs) that teaches coding agents (Claude Code, Codex, Cursor, and more) how to use the **[FASHN](https://fashn.ai) Agent**: send a plain-language prompt plus up to 6 images, and get back on-model photos, virtual try-ons, packshots, catalog images, lookbooks, and fashion videos.

The skill does two jobs:

- **Generate now**: make images and videos in the conversation through the **FASHN MCP** server, with prompting, cost, and file-handling guidance.
- **Integrate**: write code against the **FASHN Agent API** (REST) for an app, script, or pipeline.

## Capabilities

| Endpoint | What you get |
|---|---|
| Agent Images (`/v1/agent/images`) | On-model photos, virtual try-ons, model swaps, packshots, catalog images, lookbooks, edits, background removal, upscales |
| Agent Videos (`/v1/agent/videos`) | On-model videos, product clips, transitions between looks (4 to 15 s, optional sound) |

The skill knows every parameter, the run and poll flow, webhooks, credits, limits, and errors, plus how to write prompts the Agent handles well. FASHN's single-task endpoints (e.g. `tryon-max`) are covered in the reference for the few cases that still need them.

## Install

Install with [skills.sh](https://www.skills.sh/docs):

```bash
npx skills add fashn-ai/fashn-skill
```

The CLI auto-detects which coding agents you have installed and configures the skill for each. Supported agents include **Claude Code, Codex, Cursor, and many others**.

Useful flags:

```bash
# Install only for specific agents
npx skills add fashn-ai/fashn-skill -a claude-code -a codex

# Install globally (all projects), non-interactive
npx skills add fashn-ai/fashn-skill -g -y
```

Where it lands:

| Agent | Global | Per-project |
|---|---|---|
| Claude Code | `~/.claude/skills/fashn` | `.claude/skills/fashn` |
| Codex | `~/.codex/skills/fashn` | `.agents/skills/fashn` |

Manage it later with `npx skills list`, `npx skills update fashn`, or `npx skills remove fashn`.

## Prerequisites

- **To generate in the conversation:** connect the FASHN MCP server (`https://api.fashn.ai/mcp`). In Claude Code:

  ```bash
  claude mcp add --transport http fashn https://api.fashn.ai/mcp
  ```

  Other agents take the same URL in their MCP settings.

- **To integrate:** a **FASHN API key** from the [Developer API dashboard](https://app.fashn.ai/api), stored in your project's environment as `FASHN_API_KEY` (e.g. a gitignored `.env`). Never commit it or expose it client-side.

## Usage

Once installed, your agent loads the skill automatically (restart the agent if a session is already open). Then just ask:

- **Generate**: *"Put the dress from dress.jpg on the model in model.jpg, studio light, full body."* or *"Make a four-look lookbook from these six photos."*
- **Integrate**: *"Add a 'generate on-model photos' action to my product admin."* or *"Write a script that turns every product photo in ./products into an on-model photo."*
- In Claude Code and Codex you can also invoke it directly with `/fashn`.

For generations, the agent calls the FASHN MCP tools and hands you the full-size links. For integrations, it writes code against the Agent API, wires up the run, poll, or webhook flow, and handles the response, credits, and errors.

## What's in this repo

| File | Purpose |
|---|---|
| [`skills/fashn/SKILL.md`](skills/fashn/SKILL.md) | The skill: entry point, prompting tips, MCP generation guide, and Agent API integration guide (TypeScript and Python) |
| [`skills/fashn/reference.md`](skills/fashn/reference.md) | Full API reference: Agent API parameters, limits, errors, credits, webhooks, plus the single-task endpoints |

## Resources

- **Docs:** https://docs.fashn.ai
- **API keys & dashboard:** https://app.fashn.ai/api
- **Pricing:** https://fashn.ai/pricing#api
- **FASHN Agent API:** https://docs.fashn.ai/api-reference/agent-images · https://docs.fashn.ai/api-reference/agent-videos
