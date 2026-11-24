# FASHN API Reference

Reference for the FASHN REST API, centered on the [FASHN Agent API](#fashn-agent-api-experimental). The [single-task endpoints](#single-task-endpoints) are documented at the end for the few cases that still need them. Agent sections are checked against https://docs.fashn.ai/api-reference/agent-images and https://docs.fashn.ai/api-reference/agent-videos (Nov 2026). See [SKILL.md](SKILL.md) for the overview, the FASHN MCP tools, and integration code.

- **Base URL:** `https://api.fashn.ai/v1`
- **Auth:** `Authorization: Bearer $FASHN_API_KEY` (key from https://app.fashn.ai/api)
- **SDKs:** the `fashn` npm and PyPI packages are deprecated for now. Call the REST API directly.

## FASHN Agent API (Experimental)

FASHN's main API. Send a plain-language `prompt` plus up to 6 images, and the FASHN Agent decides which operations to run (try-on, product-to-model, model creation and swap, edits, background changes, packshots, upscale, background removal, video), runs them, and returns every output. The FASHN MCP server's `generate_images` / `generate_videos` tools run these same endpoints.

Lifecycle: [Experimental](https://docs.fashn.ai/api-overview/model-lifecycle). Agent Images released 2026-10-26, Agent Videos 2026-11-16.

### Agent routes

| Purpose | Images | Videos |
|---|---|---|
| Run and wait (sync) | `POST /v1/agent/images` | `POST /v1/agent/videos` |
| Start async run | `POST /v1/agent/images/run[?webhook_url=...]` | `POST /v1/agent/videos/run[?webhook_url=...]` |
| Poll async run | `GET /v1/agent/status/{id}` | `GET /v1/agent/status/{id}` |

All parameters are **top-level JSON fields** (no `model_name` / `inputs` wrapper). Any field not listed below is rejected with `400 BadRequest`. The async start returns `{ "id": "...", "error": null }`.

### Agent Images: `/v1/agent/images`

| Input | Type | Req | Notes |
|---|---|---|---|
| `prompt` | string | ✅ | 1 to 10,000 chars. Refer to images by position ("the first image") or content |
| `images` | (URL \| data URI)[] | | Up to 6, in the order the prompt refers to them. Default `[]` (prompt only) |
| `aspect_ratio` | `1:1`\|`2:3`\|`3:2`\|`3:4`\|`4:3`\|`4:5`\|`5:4`\|`9:16`\|`16:9`\|`21:9` | | Default: Agent chooses (an edit keeps the source frame) |
| `resolution` | `1k`\|`2k`\|`3k`\|`4k` | | ~1 / 4 / 9 / 16 MP. Default `1k`. Some images are made and billed at `2k` when `3k`/`4k` is requested |
| `generation_mode` | `fast`\|`balanced`\|`quality` | | Applies to every generated image. Default `balanced` |
| `return_base64` | bool | | Results as `data:image/png;base64,...`. Images are **still stored on the CDN for three days**. Default `false` |

What it can do: generate or edit images (try-on, product-to-model, model creation and swap, background changes, packshots, transparent images), upscale, and remove backgrounds. Each is one operation; the Agent inspecting its own results is not. It makes images only: a video request fails with `NoImageGenerated`.

Timing: about 40 s for one image at defaults; multi-step requests take longer (a four-look lookbook at `2k`/`quality` takes about 5 min).

**Credits** (per generated image):

| `generation_mode` \ `resolution` | `1k` | `2k` | `3k` | `4k` |
|---|---|---|---|---|
| `fast` | 1 | 1 | 1 | 1 |
| `balanced` | 1 | 2 | 3 | 3 |
| `quality` | 2 | 3 | 4 | 4 |

An upscale or background removal costs 1. A request costs the sum of its operations, intermediate images included, so at most 32 (8 operations × 4).

### Agent Videos: `/v1/agent/videos`

| Input | Type | Req | Notes |
|---|---|---|---|
| `prompt` | string | ✅ | 1 to 10,000 chars. Describe motion, camera, and scene; refer to images by position or content |
| `images` | (URL \| data URI)[] | | Up to 6, in prompt order. Default `[]` |
| `duration` | int 4-15 | | Seconds per video. Default `5`. Outside 4 to 15 → `400` |
| `resolution` | `480p`\|`720p`\|`1080p` | | Default `480p` |
| `aspect_ratio` | `16:9`\|`9:16`\|`4:3`\|`3:4`\|`1:1` | | Only for videos made without a start image (a start-image video keeps that image's frame). Default: Agent chooses |
| `audio` | bool | | `true` adds sound, `false` forces silence. Default: sound only if the prompt asks |

Values stated in the prompt (duration, resolution, aspect ratio, sound) override these fields.

How images are used (the Agent decides from the prompt):

- **Start frame**: the video animates the image and keeps its frame.
- **Start and end frames**: the video moves from the first image to the second (e.g. a transition between two looks).
- **References**: people, garments, or products from the images appear in a new scene. If the prompt asks for a different aspect ratio than a start image has, the Agent uses it as a reference instead.

One video per generation, up to 8 per request ("one video per image", "three clips"). It makes videos only: an image request fails with `NoVideoGenerated`. Timing: a few minutes for one video at defaults; longer and higher-resolution videos take longer.

**Credits** (per second of video, sound included):

| `resolution` | Credits / s | 5 s | 10 s | 15 s |
|---|---|---|---|---|
| `480p` | 1 | 5 | 10 | 15 |
| `720p` | 2 | 10 | 20 | 30 |
| `1080p` | 4 | 20 | 40 | 60 |

A request costs the sum of its videos, so at most 480 (8 × 60).

### Agent response

Sync response, async status (once terminal), and webhook body share one shape:

```json
{
  "id": "3c1f6a52-8e0b-4d7a-9a51-2f4b7c9d0e18",
  "status": "completed",
  "error": null,
  "predictions": [
    {
      "id": "123a87r9-4129-4bb3-be18-9c9fb5bd7fc1-u1",
      "tool": "generate-image",
      "status": "completed",
      "results": ["https://cdn.fashn.ai/123a87r9-4129-4bb3-be18-9c9fb5bd7fc1-u1/output_0.png"],
      "credits_used": 1,
      "error": null
    }
  ],
  "credits_used": 1
}
```

- **`status`**: `completed` (at least one output made) | `failed` (none). While an async run is in progress, the status route returns `processing` with no partial results.
- **`predictions`**: every operation in run order, including intermediate steps, failed generations, and calls refused for credits or concurrency. `tool` is `generate-image`, `image-upscale`, `background-remove`, or `generate-video`. `results` holds PNG or MP4 URLs (data URIs with `return_base64`), empty when failed. **Use the entries with `status: "completed"`.**
- **`credits_used`**: total for the request; also in the `X-Fashn-Credits-Used` header. Each prediction carries its own.
- The Agent's text is never returned.
- A run ID that doesn't exist or belongs to another user returns `404`.

### Agent limits

| | Images | Videos |
|---|---|---|
| Input images | 6, max 10 MB each as base64, JPEG/PNG/WebP | same |
| Prompt | 10,000 chars | 10,000 chars |
| Operations per request | 8 | 8 videos |
| Window to start new operations | 15 min (request can run ~30 min total) | 20 min (~40 min total) |
| Rate limit to start a run | 100 / 60 s | 100 / 60 s |
| Rate limit on `/v1/agent/status` | 50 / 10 s | 50 / 10 s |

Each running operation holds one slot of your concurrency limit. A request that reaches 8 operations stops and returns what it made; after the start window, it starts nothing new, waits for running operations, and returns (and bills) them.

### Agent errors

Before the run starts: the usual [API errors](#errors) with no `id`: `400 BadRequest` (too many images, base64 over 10 MB, prompt too long, bad `duration`, unknown field), `429 RateLimitExceeded`, `429 OutOfCredits`, `429 ConcurrencyLimitExceeded`. During a run, a generation refused with `OutOfCredits` / `ConcurrencyLimitExceeded` stops the run; running operations finish and are returned, and if nothing was made the run fails with that `429`.

A run that made no output fails with the run body (`status: "failed"`, `id`, `predictions`, `credits_used`). The sync route uses the HTTP code below; an async run returns the same body with HTTP `200` from the status route.

| HTTP | `error.name` | Cause / fix |
|---|---|---|
| 422 | `NoImageGenerated` / `NoVideoGenerated` | Nothing made, e.g. wrong endpoint for the output type, or an unclear prompt. Make the prompt more specific or check the inputs |
| 422 | `ImageLoadError` | An input URL couldn't be downloaded. Use public URLs or base64 |
| 422 | `BadRequest` | An input image couldn't be read. Send JPEG/PNG/WebP; shrink very large images |
| 422 | runtime error name (e.g. `ContentModerationError`) | The last generation failed; see [runtime errors](#errors) |
| 504 | `AgentTimeout` | Start window passed (15 min images / 20 min videos) with no output. Simplify or split the request |
| 500 | `InternalServerError` | Retry with backoff; contact support@fashn.ai with the run `id` if it persists |

If at least one output was made, the run is `completed` even when a later step failed.

### Agent data handling

Base64 input images are stored privately while the run uses them and deleted after it ends. Outputs follow the standard [data retention](https://docs.fashn.ai/api-overview/data-retention-privacy) for API outputs (and `return_base64` outputs still sit on the CDN for three days).

### Agent prompt tips

- The Agent never asks back; when a prompt is unclear it guesses. State framing, background, pose, angle, and how many outputs.
- It adds no steps you didn't ask for (no automatic upscale or variations). Ask for them in the prompt.
- Use `generation_mode: "fast"` for drafts, `"quality"` with a higher `resolution` for final assets.
- A larger catalog than 6 inputs / 8 outputs needs several requests; they can run in parallel up to your concurrency limit.
- Example (catalog): `"Put each garment from images 2 to 4 on the model from image 1, one image per garment. Full body, front view, plain light gray studio background, the same pose and lighting in every image."`
- Example (lookbook): `"Make a lookbook of four full body photos. Image 1 is the model, image 2 is the location, and images 3 to 6 are four outfits, one per look. Make look 1 first. Then make looks 2, 3 and 4, each one directly from look 1, in a different pose, head angle and expression. In every look, use the face from image 1, so the model matches image 1 exactly."` (2k, quality, 2:3: about 12 credits.)
- Example (video): `"The model turns slowly to show the dress from every side"` with one on-model image as the start frame.
- More: https://docs.fashn.ai/getting-started/agent-quickstart and https://help.fashn.ai/usage-guides/prompting-in-fashn.

## Credits & pricing

- Agent API: see the per-image and per-second tables in [Agent Images](#agent-images-v1agentimages) and [Agent Videos](#agent-videos-v1agentvideos). Single-task pricing scales with `generation_mode`, `resolution`, and `num_images`; current prices at https://fashn.ai/pricing#api.
- **Failed operations are not charged.**
- Actual spend per call: the top-level `credits_used` field (Agent API) or the `x-fashn-credits-used` response header (both).
- Balance: `GET /v1/credits` → `{ "credits": { "total": 234, "subscription": 100, "on_demand": 134 } }`. `subscription` credits reset each billing cycle and are spent before `on_demand`.

## Errors

**API errors**: HTTP 4xx/5xx returned *before* the job runs. Body: `{ "error": "<Code>", "message": "<desc>" }`.

| HTTP | Codes |
|---|---|
| 400 | `BadRequest` |
| 401 | `UnauthorizedAccess` |
| 404 | `NotFound` |
| 429 | `RateLimitExceeded`, `ConcurrencyLimitExceeded`, `OutOfCredits` |
| 500 | `InternalServerError` |

**Runtime errors**: the job ran but failed: HTTP 200 with `status: "failed"` and `error: { name, message }`. Always branch on `status` even when nothing throws. Agent-specific names (`NoImageGenerated`, `NoVideoGenerated`, `AgentTimeout`) are in [Agent errors](#agent-errors); an Agent run whose last generation failed reports one of the names below.

| `error.name` | Cause / fix |
|---|---|
| `ImageLoadError` | Couldn't fetch/decode an input image. Ensure URLs are public with correct Content-Type; base64 includes the `data:image/...;base64,` prefix. |
| `ContentModerationError` | Input flagged. Change the inputs or prompt (single-task `tryon-v1.6` also has `moderation_level`). Explicit/inappropriate imagery with real people is prohibited. |
| `PoseError` | Body pose not detectable (try-on). Improve image quality per model-photo guidelines. |
| `InputValidationError` | Invalid/missing parameter combination (single-task, e.g. `reframe` needs `aspect_ratio`). |
| `ThirdPartyError` / `3rdPartyProviderError` | Upstream provider refused/failed (often caption/content restrictions). Modify inputs and retry. |
| `PipelineError` | Internal processing failure. Retry (free on failure); contact support with the prediction ID if it persists. |
| `InternalServerError` | Unexpected server error. Retry. |

## Webhooks

Skip polling by passing a `webhook_url` **query parameter** on a run route:

```
POST https://api.fashn.ai/v1/agent/images/run?webhook_url=https://your-server.com/webhook
{ "prompt": "...", "images": ["..."] }
```

FASHN POSTs the terminal status body to that URL: the [Agent response](#agent-response) shape for `/v1/agent/images/run` and `/v1/agent/videos/run`, or the single-task envelope (`{ id, status, output, error }`) for `/v1/run`.

- Delivered on terminal states only (`completed` / `failed`).
- Failed deliveries (non-2xx) are retried up to **5 times** over **~5 minutes**.
- **No signature verification** is provided, so treat the callback only as a notification: confirm the run's state and results through the authenticated status route (`GET /v1/agent/status/{id}` or `GET /v1/status/{id}`) before acting on it. Make handlers **idempotent** and respond `2xx` quickly.

## Rate limits

Requests are rate-limited per endpoint. When batching, handle `429` (`RateLimitExceeded` / `ConcurrencyLimitExceeded`) with backoff. Agent API: 100 run starts / 60 s and 50 status polls / 10 s; see [Agent limits](#agent-limits).

## Notes

- All FASHN calls are server-side; never expose `FASHN_API_KEY` in client code.
- Agent API `return_base64` outputs are still stored on the CDN for three days. (Single-task `return_base64` outputs are not stored server-side.)
- Retry connection errors and `5xx` with backoff. A synchronous Agent request that times out on your side leaves you with no run `id` to check, so use the async routes for anything long.

---

## Single-task endpoints

Each call runs one model: `POST /v1/run` with `{ model_name, inputs }`. These are no longer FASHN's focus; prefer the [FASHN Agent API](#fashn-agent-api-experimental), which runs the same operations from a prompt. Use a single-task endpoint only when the user asks for one by name, the project already uses one, or a real-time product feature needs one fixed model (e.g. a "try it on" button with `tryon-max`). `seed` makes single-task results reproducible for identical inputs; vary it to force new variations.

### Prediction lifecycle

Every job is `POST /v1/run` with a body of `{ model_name, inputs }`, then poll `GET /v1/status/{id}` until terminal (or pass a [webhook](#webhooks)). Check the balance with `GET /v1/credits`.

#### Response shapes

`POST /v1/run` returns immediately:

```json
{ "id": "9dafef71-6e90-4bc9-ac05-d0d97c612722", "error": null }
```

`GET /v1/status/{id}`:

```json
{
  "id": "9dafef71-...",
  "status": "completed",
  "output": ["https://cdn.fashn.ai/.../output_0.png"],
  "error": null
}
```

- **`status`**: `starting` | `in_queue` | `processing` | `completed` | `failed`.
- **`output`**: array of CDN URLs (images, or MP4 for `image-to-video`); base64 data-URIs when `return_base64: true`.
- **`error`**: `null`, or a runtime error `{ name, message }` (see [Errors](#errors)).
- **Credits**: the `x-fashn-credits-used` response header on the status call.

### Common input parameters

Most generation endpoints share these optional fields:

- `aspect_ratio`: one of `21:9`, `1:1`, `4:3`, `3:2`, `2:3`, `5:4`, `4:5`, `3:4`, `16:9`, `9:16`.
- `generation_mode`: quality/speed tradeoff. Values are endpoint-specific: try-on is `balanced` | `quality`; most generative endpoints add `fast` (`fast` | `balanced` | `quality`).
- `resolution`: `1k` (~1 MP) | `2k` (~4 MP) | `4k` (~16 MP).
- `num_images` (or `num_samples` on `tryon-v1.6`): 1-4 outputs.
- `output_format`: `png` (lossless) | `jpeg` (smaller).
- `return_base64`: `true` returns base64 instead of a CDN URL (not stored server-side; expires 60 min).
- `seed`: integer `0` to `2^32-1` for reproducibility.

Image inputs accept an **HTTPS URL or a base64 data-URI** (must include the prefix, e.g. `data:image/jpg;base64,<...>`).

---

### Endpoints

#### Virtual Try-On (flagship): `tryon-max`

Premium try-on for AI fashion photoshoots and publishable e-commerce content (PDPs, catalogs, marketing). Processing ~20-120s depending on mode/resolution.

| Input | Type | Req | Notes |
|---|---|---|---|
| `model_image` | image | ✅ | Person to dress (identity/pose preserved) |
| `product_image` | image | ✅ | Garment/accessory to place on the model |
| `prompt` | string | | Tweaks, e.g. `"remove scarf"`, `"tuck in shirt"`, `"roll up sleeves"` |
| `aspect_ratio` | enum | | See common list |
| `generation_mode` | `balanced`\|`quality` | | Default auto-selected |
| `resolution` | `1k`\|`2k`\|`4k` | | Default `1k` |
| `num_images` | int 1-4 | | Default `1` |
| `output_format` | `png`\|`jpeg` | | Default `png` |
| `seed` | int | | Default `42` |
| `return_base64` | bool | | Default `false` |

**Credits** (× `num_images`): `balanced` → 1k:2 / 2k:3 / 4k:4 · `quality` → 1k:3 / 2k:4 / 4k:5.

#### Virtual Try-On (legacy): `tryon-v1.6`

Fast, low-cost try-on from a single person + garment photo. Output 864×1296. Modes: `performance` ~5s, `balanced` ~8s, `quality` ~12-17s. Prefer `tryon-max` for publishable quality; use `tryon-v1.6` for speed/volume and fine moderation control.

| Input | Type | Req | Notes |
|---|---|---|---|
| `model_image` | image | ✅ | Person; Models Studio users can pass `saved:<model_name>` |
| `garment_image` | image | ✅ | Clothing item to try on |
| `category` | `auto`\|`tops`\|`bottoms`\|`one-pieces` | | Default `auto` |
| `garment_photo_type` | `auto`\|`flat-lay`\|`model` | | Optimizes for source photo type |
| `mode` | `performance`\|`balanced`\|`quality` | | Speed/quality tradeoff |
| `moderation_level` | `conservative`\|`permissive`\|`none` | | Controls modesty filtering |
| `segmentation_free` | bool | | Better extraction for bulky garments; set `false` if originals aren't removed |
| `num_samples` | int 1-4 | | Default `1` |
| `output_format` | `png`\|`jpeg` | | |
| `seed` | int | | |
| `return_base64` | bool | | |

#### Product to Model: `product-to-model`

Turns a product image into a person wearing it. **Standard mode** (no `model_image`) generates a new person; **try-on mode** (`model_image` provided) adds the product to that person and matches its aspect ratio.

| Input | Type | Req | Notes |
|---|---|---|---|
| `product_image` | image | ✅ | Clothing, shoes, bags, accessories, etc. |
| `model_image` | image | | Provide to enable try-on mode |
| `prompt` | string | | Person/styling/background, e.g. `"man with tattoos, studio background"` |
| `image_prompt` | image | | Inspiration image guiding pose/environment/lighting |
| `aspect_ratio` | enum | | Standard mode only (ignored in try-on mode) |
| `generation_mode` | `fast`\|`balanced`\|`quality` | | |
| `resolution` | `1k`\|`2k`\|`4k` | | |
| `output_format` | `png`\|`jpeg` | | |
| `seed` | int | | |
| `return_base64` | bool | | |

#### Model Create: `model-create`

Generates a fashion model from a text prompt, with optional pose/composition and face references.

| Input | Type | Req | Notes |
|---|---|---|---|
| `prompt` | string | ✅ | Describes model, clothing, pose, scene |
| `image_reference` | image | | Structural/composition guide; if set and `aspect_ratio` omitted, output matches its dimensions |
| `face_reference` | image | | Generated person resembles this face |
| `face_reference_mode` | `match_base`\|`match_reference` | | Adapt to scene vs. preserve the reference face |
| `aspect_ratio` | enum | | Canvas for text-only generation |
| `generation_mode` | `fast`\|`balanced`\|`quality` | | |
| `resolution` | `1k`\|`2k`\|`4k` | | |
| `num_images` | int 1-4 | | |
| `output_format` | `png`\|`jpeg` | | |
| `seed` | int | | |
| `return_base64` | bool | | |

#### Model Swap: `model-swap`

Changes a model's identity (face, skin tone, hair) while preserving the outfit and pose exactly.

| Input | Type | Req | Notes |
|---|---|---|---|
| `model_image` | image | ✅ | Source; clothing/pose preserved |
| `prompt` | string | | Identity description (ethnicity, features, hair). Empty = random |
| `face_reference` | image | | Swap to this specific identity |
| `face_reference_mode` | `match_base`\|`match_reference` | | |
| `aspect_ratio` | enum | | |
| `generation_mode` | `fast`\|`balanced`\|`quality` | | |
| `resolution` | `1k`\|`2k`\|`4k` | | |
| `num_images` | int 1-4 | | |
| `output_format` | `png`\|`jpeg` | | |
| `seed` | int | | |
| `return_base64` | bool | | |

#### Face to Model: `face-to-model`

Converts a headshot/selfie into a try-on-ready upper-body avatar, preserving facial identity.

| Input | Type | Req | Notes |
|---|---|---|---|
| `face_image` | image | ✅ | Headshot or selfie |
| `prompt` | string | | Body-shape/styling guidance, e.g. `"athletic build"` |
| `aspect_ratio` | enum | | Default `2:3`; vertical ratios look most natural |
| `generation_mode` | `fast`\|`balanced`\|`quality` | | |
| `resolution` | `1k`\|`2k`\|`4k` | | |
| `num_images` | int 1-4 | | |
| `output_format` | `png`\|`jpeg` | | Default `jpeg` |
| `seed` | int | | |
| `return_base64` | bool | | Default `false` |

#### Edit: `edit`

Versatile post-processing: restyle, change views/poses, swap backgrounds, fix details, while preserving identity and product fidelity.

| Input | Type | Req | Notes |
|---|---|---|---|
| `image` | image | ✅ | Source image |
| `prompt` | string | ✅ | Edit instruction, e.g. `"change the dress to red"` |
| `mask` | image | | White (255) = edit region, black (0) = keep; for precise local edits |
| `image_context` | image | | Extra visual context guiding the edit |
| `aspect_ratio` | enum | | |
| `generation_mode` | `fast`\|`balanced`\|`quality` | | |
| `resolution` | `1k`\|`2k`\|`4k` | | |
| `num_images` | int 1-4 | | |
| `output_format` | `png`\|`jpeg` | | |
| `seed` | int | | |
| `return_base64` | bool | | |

#### Reframe: `reframe`

Adjusts aspect ratio by intelligently outpainting (zoom-out) or cropping (zoom-in). Output ~1 MP. Returns an error if the source already matches the target within 2%.

| Input | Type | Req | Notes |
|---|---|---|---|
| `image` | image | ✅ | Source |
| `aspect_ratio` | enum | ✅ | Target ratio |
| `generation_mode` | `fast`\|`balanced`\|`quality` | | |
| `num_images` | int 1-4 | | |
| `output_format` | `png`\|`jpeg` | | |
| `seed` | int | | |
| `return_base64` | bool | | |

#### Background Remove: `background-remove`

Removes the background, producing a transparent PNG cutout. Up to ~4 MP. Fast (~1-3s). Output is always PNG.

| Input | Type | Req | Notes |
|---|---|---|---|
| `image` | image | ✅ | Source |
| `return_base64` | bool | | Returns base64 PNG instead of CDN URL |

#### Image to Video: `image-to-video`

Animates a single image into a short fashion clip. `output` is an MP4 CDN URL.

| Input | Type | Req | Notes |
|---|---|---|---|
| `image` | image | ✅ | Start frame |
| `duration` | `5`\|`10` | | Seconds |
| `resolution` | `480p`\|`720p`\|`1080p` | | |
| `end_image` | image | | Final frame; only with `resolution: "1080p"` |
| `prompt` | string | | Short motion guidance; best left empty to auto-plan |
| `negative_prompt` | string | | Motion/framing to avoid |
| `seed` | int | | |

---
