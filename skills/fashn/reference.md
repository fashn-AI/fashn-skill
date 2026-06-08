# FASHN API & SDK Reference

Authoritative reference for the FASHN prediction API and the official SDKs. Verified against the TypeScript SDK `fashn@0.15.0`. See [SKILL.md](SKILL.md) for the integration overview and decision guide.

- **Base URL:** `https://api.fashn.ai/v1`
- **Auth:** `Authorization: Bearer $FASHN_API_KEY` (key from https://app.fashn.ai/api)
- **Packages:** TypeScript `fashn` (npm) · Python `fashn` (PyPI), both read `FASHN_API_KEY` from the environment.

## Prediction lifecycle

Every job is `POST /v1/run` with a body of `{ model_name, inputs }`, then poll `GET /v1/status/{id}` until terminal. The SDKs' `subscribe()` wraps run + poll.

| Step | REST | TypeScript SDK | Python SDK |
|---|---|---|---|
| Submit | `POST /v1/run` | `client.predictions.run({ model_name, inputs })` | `client.predictions.run(model_name=, inputs=)` |
| Poll | `GET /v1/status/{id}` | `client.predictions.status(id)` | `client.predictions.status(id)` |
| Submit + auto-poll | (build your own loop) | `client.predictions.subscribe({ model_name, inputs })` | `client.predictions.subscribe(model_name=, inputs=)` |
| Balance | `GET /v1/credits` | n/a (call REST) | n/a (call REST) |

`subscribe()` options (TS SDK): `pollInterval` (ms, default `1000`), `timeout` (ms, default `300000` = 5 min), `maxRetries` (default `3`), and callbacks `onEnqueued(requestId)` / `onQueueUpdate(status)`. The Python SDK uses snake_case: `poll_interval`, `timeout`, `max_retries`, `on_enqueued`, `on_queue_update`. In-progress statuses (`starting`/`in_queue`/`processing`) are only delivered via the queue-update callback; the resolved value is always terminal.

### Response shapes

`POST /v1/run` (and SDK `run`) returns immediately:

```json
{ "id": "9dafef71-6e90-4bc9-ac05-d0d97c612722", "error": null }
```

`GET /v1/status/{id}` (and SDK `status` / `subscribe`):

```json
{
  "id": "9dafef71-...",
  "status": "completed",
  "output": ["https://cdn.fashn.ai/.../output_0.png"],
  "error": null
}
```

- **`status`**: `starting` | `in_queue` | `processing` | `completed` | `failed`. SDK `subscribe()` may additionally resolve to `canceled` or `time_out`.
- **`output`**: array of CDN URLs (images, or MP4 for `image-to-video`); base64 data-URIs when `return_base64: true`.
- **`error`**: `null`, or a runtime error `{ name, message }` (see [Errors](#errors)).
- **Credits**: REST exposes the `x-fashn-credits-used` response header on status; SDK exposes `creditsUsed` (TS) / `credits_used` (Python) on the `subscribe`/`run` result.

## Common input parameters

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

## Endpoints

### Virtual Try-On (flagship): `tryon-max`

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

### Virtual Try-On (legacy): `tryon-v1.6`

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

### Product to Model: `product-to-model`

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

### Model Create: `model-create`

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

### Model Swap: `model-swap`

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

### Face to Model: `face-to-model`

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

### Edit: `edit`

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

### Reframe: `reframe`

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

### Background Remove: `background-remove`

Removes the background, producing a transparent PNG cutout. Up to ~4 MP. Fast (~1-3s). Output is always PNG.

| Input | Type | Req | Notes |
|---|---|---|---|
| `image` | image | ✅ | Source |
| `return_base64` | bool | | Returns base64 PNG instead of CDN URL |

### Image to Video: `image-to-video`

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

## Credits & pricing

- Cost scales with `generation_mode`, `resolution`, and `num_images`. The documented `tryon-max` table is above; for current per-endpoint pricing see https://fashn.ai/pricing#api.
- **Failed predictions are not charged.**
- Actual spend per call: read the `x-fashn-credits-used` header (REST) or `creditsUsed`/`credits_used` (SDK).
- Balance: `GET /v1/credits` → `{ "credits": { "total": 234, "subscription": 100, "on_demand": 134 } }`. `subscription` credits reset each billing cycle and are spent before `on_demand`.

## Errors

**API errors**: HTTP 4xx/5xx returned *before* the job runs. REST body: `{ "error": "<Code>", "message": "<desc>" }`. SDKs throw `APIError` subclasses (TS: `Fashn.APIError` with `err.status`; Python: `fashn.APIError` with `e.status_code`).

| HTTP | Codes |
|---|---|
| 400 | `BadRequest` |
| 401 | `UnauthorizedAccess` |
| 404 | `NotFound` |
| 429 | `RateLimitExceeded`, `ConcurrencyLimitExceeded`, `OutOfCredits` |
| 500 | `InternalServerError` |

**Runtime errors**: the job ran but failed: HTTP 200 with `status: "failed"` and `error: { name, message }`. Always branch on `status` even when nothing throws.

| `error.name` | Cause / fix |
|---|---|
| `ImageLoadError` | Couldn't fetch/decode an input image. Ensure URLs are public with correct Content-Type; base64 includes the `data:image/...;base64,` prefix. |
| `ContentModerationError` | Input flagged. For try-on, adjust `moderation_level`. Explicit/inappropriate imagery with real people is prohibited. |
| `PoseError` | Body pose not detectable (try-on only). Improve image quality per model-photo guidelines. |
| `InputValidationError` | Invalid/missing parameter combination (e.g. reframe needs `aspect_ratio`). |
| `ThirdPartyError` / `3rdPartyProviderError` | Upstream provider refused/failed (often caption/content restrictions). Modify inputs and retry. |
| `PipelineError` | Internal processing failure. Retry (free on failure); contact support with the prediction ID if it persists. |
| `InternalServerError` | Unexpected server error. Retry. |
| `PollingTimeout` | `subscribe()` exceeded `timeout`. Increase `timeout` or poll manually. |

## Webhooks

Skip polling by passing a `webhook_url` **query parameter** on `/run`:

```
POST https://api.fashn.ai/v1/run?webhook_url=https://your-server.com/webhook
{ "model_name": "tryon-max", "inputs": { ... } }
```

FASHN POSTs the terminal envelope to that URL:

```json
{ "id": "123...-u1", "status": "completed", "output": ["https://cdn.fashn.ai/.../output_0.png"], "error": null }
```

…or on failure `{ "id": "...", "status": "failed", "error": { "name": "ImageLoadError", "message": "..." } }`.

- Delivered on terminal states only (`completed` / `failed`).
- Failed deliveries (non-2xx) are retried up to **5 times** over **~5 minutes**.
- **No signature verification** is provided, so make handlers **idempotent** and respond `2xx` quickly.

## Rate limits

Requests are rate-limited per endpoint. When batching, handle `429` (`RateLimitExceeded` / `ConcurrencyLimitExceeded`) with backoff.

## Notes

- All FASHN calls are server-side; never expose `FASHN_API_KEY` in client code.
- Use `return_base64: true` when you don't want outputs stored on FASHN's CDN; base64 outputs are not stored server-side.
- `seed` makes results reproducible for identical inputs; vary it to force new variations.
- The SDK auto-retries connection errors/timeouts (3 retries by default) for API calls.
