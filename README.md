# FASHN Agent Skill

An [agent skill](https://www.skills.sh/docs) that teaches coding agents (Claude Code, Codex, Cursor, and more) how to integrate **[FASHN](https://fashn.ai)**, whose human-centric generative image models power virtual try-on, AI models, product photography, image editing, and image-to-video.

The skill covers both the **REST API** and the official **TypeScript** and **Python** SDKs, so an agent can wire FASHN into a project, or run a one-off generation inline.

## Capabilities

The skill knows the full FASHN surface (every endpoint, its `inputs`, the error catalog, credits, and webhooks):

| Use case | `model_name` |
|---|---|
| Virtual try-on (flagship / legacy) | `tryon-max` · `tryon-v1.6` |
| Product → person wearing it | `product-to-model` |
| Generate / swap a model | `model-create` · `model-swap` |
| Headshot → upper-body avatar | `face-to-model` |
| Freeform edit · reframe | `edit` · `reframe` |
| Remove background | `background-remove` |
| Image → video | `image-to-video` |

## Install

Install with [skills.sh](https://www.skills.sh/docs):

```bash
npx skills add fashn-ai/fashn-skill
```

The CLI auto-detects which coding agents you have installed and configures the skill for each. Supported agents include **Claude Code, Codex, Cursor, GitHub Copilot, Windsurf, Gemini, Cline, AMP, Antigravity, and ClawdBot**.

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

- A **FASHN API key**, generated at the [Developer API dashboard](https://app.fashn.ai/api).
  Export it as `FASHN_API_KEY` (the REST calls and both SDKs read it from the environment):

  ```bash
  export FASHN_API_KEY="your-key-here"
  ```

- For the **SDK paths**: Node.js 18+ (TypeScript) or Python 3.8+ (Python).
  Both SDKs are published as `fashn`: `npm install fashn` or `pip install fashn`.

## Usage

Once installed, your agent loads the skill automatically (restart the agent if a session is already open). Then just ask:

- **Integrate**: *"Add FASHN virtual try-on to my Next.js app using the TypeScript SDK."*
- **Run inline**: *"Do a try-on with this model photo and this garment."*
- In Claude Code you can also invoke it directly with `/fashn`.

The agent will choose the right `model_name`, wire up the prediction lifecycle (`subscribe` or manual `run` + `status`), and handle the response envelope, credits, and errors.

## What's in this repo

| File | Purpose |
|---|---|
| [`skills/fashn/SKILL.md`](skills/fashn/SKILL.md) | The skill: entry point, integration guide (REST / TS / Python), and inline quickstart |
| [`skills/fashn/reference.md`](skills/fashn/reference.md) | Full endpoint catalog: every input, the error catalog, credits, webhooks, rate limits |

## Resources

- **Docs:** https://docs.fashn.ai
- **API keys & dashboard:** https://app.fashn.ai/api
- **Pricing:** https://fashn.ai/pricing#api
- **TypeScript SDK:** https://github.com/fashn-AI/fashn-typescript-sdk
