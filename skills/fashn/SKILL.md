---
name: fashn
description: Make fashion images and videos with the FASHN Agent, or build the FASHN Agent API into a project. Send a plain-language prompt plus up to 6 images (models, garments, products, locations) and get back on-model photos, virtual try-ons, model swaps, packshots, catalog images, lookbooks, edits, background removal, upscales, or fashion videos. Generates through the FASHN MCP server; integrates through the FASHN REST API.
allowed-tools: Bash, Read, Write, Edit
---

# FASHN

FASHN is an AI-first company specializing in human-centric generative image models tailored for fashion applications. Its main API is the **FASHN Agent API**: send a plain-language `prompt` plus up to 6 images, and the FASHN Agent plans and runs the steps itself (try-on, product-to-model, model creation and swap, edits, background changes, packshots, upscale, background removal, video) and returns everything it made.

- **Agent Images** (`/v1/agent/images`): on-model photos, try-ons, packshots, catalog images, lookbooks, edits.
- **Agent Videos** (`/v1/agent/videos`): on-model videos, product clips, transitions between looks.

Both are Experimental. This skill covers two jobs:

1. **Generate now** (the user wants images or videos in this conversation): use the **FASHN MCP** tools. See [Generate with the FASHN MCP](#generate-with-the-fashn-mcp).
2. **Integrate** (the user wants FASHN in their app, script, or pipeline): write code against the **Agent API**. See [Integrate the Agent API](#integrate-the-agent-api).

**Don't use the FASHN SDKs** (`fashn` on npm and PyPI). They are deprecated for now: don't install or recommend them, and write new code against the REST API. If a project already uses them, leave that code alone unless the user asks to migrate it.

> Full parameters, limits, the error catalog, credit tables, and webhooks live in **[reference.md](reference.md)**. Load it when you need exact details.

## Writing prompts for the FASHN Agent

These apply to the MCP tools and the API alike:

- The Agent never asks a clarifying question; when a prompt is unclear, it guesses. State framing, background, pose, angle, and how many outputs.
- Refer to input images by position ("the dress in Image 2", "the model in the first image") or by what they show. Order the `images` array to match.
- It adds no steps you didn't ask for. Ask for an upscale, a transparent background, or variations explicitly.
- Settings stated in the prompt (aspect ratio, size, duration, sound) override the matching parameters.
- Images and videos are separate: the images tool or endpoint makes only images, the videos one only videos.
- Use `generation_mode: "fast"` for drafts and `"quality"` with a higher `resolution` for final assets.
- One request takes up to 6 input images and makes up to 8 outputs. A bigger job needs several requests.

Examples:

- **On-model photos:** "Show the dress in the first image on a model in a bright studio. Make three images from different angles: front, three-quarter, and side, full body."
- **Packshot:** "Make a packshot of the bag in the first image, alone on a plain white background with a soft studio shadow."
- **Catalog:** "Put each garment from images 2 to 4 on the model from image 1, one image per garment. Full body, front view, plain light gray studio background, the same pose and lighting in every image."
- **Video:** "The model turns slowly to show the dress from every side" (one on-model photo as the start frame).

More in reference.md ([Agent prompt tips](reference.md#agent-prompt-tips)).

## Generate with the FASHN MCP

The FASHN MCP server runs the FASHN Agent for you; no API key handling or code needed. Its tools:

| Tool | What it does |
|---|---|
| `generate_images` | Runs Agent Images: `prompt`, `images`, `aspect_ratio`, `resolution`, `generation_mode`. Returns the run with results once it ends, or its `id` if still running after about 45 s |
| `generate_videos` | Runs Agent Videos: `prompt`, `images`, `duration` (4 to 15 s), `resolution`, `aspect_ratio`, `audio`. Usually returns the run `id` |
| `get_run` | Waits up to about 45 s for a run, then returns it. Call it again while the run is still processing. Free |
| `get_credits` | The organization's credit balance |

**Flow:** call `generate_images` / `generate_videos`, then call `get_run` with the `id` until the run ends. Videos take several minutes.

- **Results:** always give the user the **full-size links**. Image results also include small previews; use them only to check the output.
- **Local images:** the tools take public URLs or base64 data URIs (`data:image/jpeg;base64,...`, max 10 MB, JPEG/PNG/WebP). For a local file, encode it with `base64` and use the file's real MIME type. Downscale large photos first (around 1500 px on the long side is plenty) to keep the tool call small.
- **Saving results:** when the user wants files on disk, download the full-size links from the run's `results` (FASHN's CDN), e.g. `curl -fsSL -o look-1.png "<url>"` (`.mp4` for videos), into a folder they choose.
- **Credits:** every generation spends the organization's FASHN API credits. Before a large job (several high-resolution images, long or `1080p` videos), estimate the cost from [Credits](#credits) and tell the user; check `get_credits` if the balance might be short.

**If the MCP tools aren't available**, tell the user to connect the FASHN MCP server (`https://api.fashn.ai/mcp`, streamable HTTP). In Claude Code:

```bash
claude mcp add --transport http fashn https://api.fashn.ai/mcp
```

Other agents (Codex, Cursor, and so on) take the same URL in their MCP settings. Then restart the session so the tools load.

## Integrate the Agent API

Base URL `https://api.fashn.ai/v1`. Send `Authorization: Bearer $FASHN_API_KEY` and `Content-Type: application/json`.

### Authentication

- Get an API key from the Developer API dashboard: https://app.fashn.ai/api
- All FASHN calls are server-side: **never hard-code the key or expose it client-side**.
- Read the key from the *project's* own environment (a gitignored `.env`) or its secret manager. Never echo it back, write it to a shell profile, or commit it.

### Routes and body

| | Images | Videos |
|---|---|---|
| Async start (returns `{ id, error }`) | `POST /v1/agent/images/run` | `POST /v1/agent/videos/run` |
| Poll | `GET /v1/agent/status/{id}` | `GET /v1/agent/status/{id}` |
| Synchronous (holds the request open) | `POST /v1/agent/images` | `POST /v1/agent/videos` |

The body is top-level JSON (unknown fields are rejected with `400`):

- Both: `prompt` (required, 1 to 10,000 chars), `images` (up to 6 public URLs or base64 data URIs, max 10 MB each, JPEG/PNG/WebP).
- Images: `aspect_ratio`, `resolution` (`1k` default, `2k`, `3k`, `4k`), `generation_mode` (`fast`, `balanced` default, `quality`), `return_base64`.
- Videos: `duration` (4 to 15 s, default `5`), `resolution` (`480p` default, `720p`, `1080p`), `aspect_ratio` (only for videos without a start image), `audio` (default: sound only if the prompt asks).

**Prefer async + poll.** One image takes about 40 s and a video a few minutes; a large run can take 30 (images) or 40 (videos) minutes, which outlasts most HTTP client and serverless timeouts. Use the synchronous routes only for short scripts. In serverless apps, start the run in one request and poll from a job queue, or have the browser poll your own server endpoint (which calls FASHN server-side), or use a [webhook](#webhooks).

### Run and poll

TypeScript (Node 18+ `fetch`):

```ts
const API = 'https://api.fashn.ai/v1';
const headers = { Authorization: `Bearer ${process.env.FASHN_API_KEY}`, 'Content-Type': 'application/json' };
const sleep = (ms: number) => new Promise((r) => setTimeout(r, ms));

type AgentPrediction = { id: string; tool: string; status: 'completed' | 'failed'; results: string[]; credits_used: number };
type AgentRun = {
  id: string;
  status: 'processing' | 'completed' | 'failed';
  error: { name: string; message: string } | null;
  predictions: AgentPrediction[];
  credits_used: number;
};

export async function runFashnAgent(kind: 'images' | 'videos', body: object, timeoutMs = 45 * 60_000): Promise<AgentRun> {
  const start = await fetch(`${API}/agent/${kind}/run`, { method: 'POST', headers, body: JSON.stringify(body) });
  const started = await start.json();
  if (!start.ok) throw new Error(`FASHN ${start.status} ${started.error}: ${started.message}`);

  const deadline = Date.now() + timeoutMs;
  while (Date.now() < deadline) {
    await sleep(3000);
    const res = await fetch(`${API}/agent/status/${started.id}`, { headers });
    if (res.status === 429) { await sleep(5000); continue; } // status route: 50 req / 10 s
    const run = await res.json();
    if (!res.ok) throw new Error(`FASHN ${res.status}: ${JSON.stringify(run)}`);
    if (run.status !== 'processing') return run;
  }
  throw new Error(`FASHN agent run ${started.id} still processing after ${timeoutMs} ms`);
}

const run = await runFashnAgent('images', {
  prompt: 'Put the dress from the second image on the model in the first image',
  images: ['https://example.com/person.jpg', 'https://example.com/dress.jpg'],
});
if (run.status !== 'completed') throw new Error(`${run.error?.name}: ${run.error?.message}`);
const outputs = run.predictions.filter((p) => p.status === 'completed').flatMap((p) => p.results);
console.log(outputs, 'credits:', run.credits_used);
```

Python (`requests`):

```python
import os, time, requests

API = "https://api.fashn.ai/v1"
HEADERS = {"Authorization": f"Bearer {os.environ['FASHN_API_KEY']}", "Content-Type": "application/json"}

def run_fashn_agent(kind: str, body: dict, timeout_s: int = 45 * 60) -> dict:
    start = requests.post(f"{API}/agent/{kind}/run", headers=HEADERS, json=body, timeout=60)
    started = start.json()
    if not start.ok:
        raise RuntimeError(f"FASHN {start.status_code} {started.get('error')}: {started.get('message')}")

    deadline = time.monotonic() + timeout_s
    while time.monotonic() < deadline:
        time.sleep(3)
        res = requests.get(f"{API}/agent/status/{started['id']}", headers=HEADERS, timeout=60)
        if res.status_code == 429:  # status route: 50 req / 10 s
            time.sleep(5)
            continue
        res.raise_for_status()
        run = res.json()
        if run["status"] != "processing":
            return run
    raise TimeoutError(f"FASHN agent run {started['id']} still processing")

run = run_fashn_agent("videos", {
    "prompt": "The model turns slowly to show the dress from every side",
    "images": ["https://example.com/model-in-dress.jpg"],
    "duration": 5,
})
if run["status"] != "completed":
    raise RuntimeError(f"{run['error']['name']}: {run['error']['message']}")
outputs = [url for p in run["predictions"] if p["status"] == "completed" for url in p["results"]]
```

### Handling the result

- `predictions` lists **every operation in run order**, including intermediate steps, failed attempts, and calls refused for credits or concurrency. Use the entries with `status: "completed"`; each has `tool` (`generate-image`, `image-upscale`, `background-remove`, or `generate-video`), `results` (PNG or MP4 URLs), and `credits_used`. A multi-step request can return more images than the user asked for.
- Run `status: "completed"` means **at least one** output was made, even if a later step failed. `failed` means none.
- Failures: the status route returns HTTP 200 with `status: "failed"` and `error: { name, message }`: `NoImageGenerated` / `NoVideoGenerated` (wrong endpoint for the output, or an unclear prompt), `ImageLoadError`, `AgentTimeout`, or a runtime error like `ContentModerationError`. Pre-run errors (`400`, `401`, `429 RateLimitExceeded` / `OutOfCredits` / `ConcurrencyLimitExceeded`) have no `id`. See [Agent errors](reference.md#agent-errors).
- Download outputs you need to keep; CDN links follow FASHN's [data retention](https://docs.fashn.ai/api-overview/data-retention-privacy). `return_base64` outputs are still stored on the CDN for three days.

### Webhooks

Skip polling by adding `?webhook_url=https://your-server.com/webhook` to the `/run` route. FASHN POSTs the terminal status body (same shape as the status response) to that URL, retrying up to 5 times over about 5 minutes. There's no signature header, so treat the webhook only as a notification: fetch the run from `GET /v1/agent/status/{id}` (authenticated) before updating your app's state, make the handler idempotent, and respond 2xx fast.

### Credits

- **Images:** per generated image, by `generation_mode` × `resolution`: `fast` 1 at any size; `balanced` 1 / 2 / 3 / 3 and `quality` 2 / 3 / 4 / 4 at `1k` / `2k` / `3k` / `4k`. An upscale or background removal costs 1. At most 32 per request.
- **Videos:** 1 / 2 / 4 credits per second at `480p` / `720p` / `1080p`, sound included. At most 480 per request (8 videos, 15 s, `1080p`).
- **Failed operations are not charged.** The exact total is the top-level `credits_used` (also the `X-Fashn-Credits-Used` header). Check the balance with `GET /v1/credits`.

**Limits:** 100 run starts per 60 s and 50 status polls per 10 s; each running operation holds one concurrency slot. Handle `429` with backoff when batching, and run batches in parallel only up to your concurrency limit.

## Single-task endpoints

FASHN also has single-model endpoints (`POST /v1/run` with `{ model_name, inputs }`, e.g. `tryon-max`). They're no longer the focus: default to the Agent API, which runs the same operations from a prompt. Use one only when the user asks for it by name, the project already uses it, or a real-time product feature needs one fixed model (e.g. a "try it on" button). Details in [reference.md](reference.md#single-task-endpoints).

## Reference

See **[reference.md](reference.md)** for every parameter (types, allowed values, defaults), limits, the complete error catalog, credit tables, webhook payloads, and rate limits. Human-readable docs: https://docs.fashn.ai.

## User's request

$ARGUMENTS
