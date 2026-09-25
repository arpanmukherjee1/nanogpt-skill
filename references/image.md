# Image generation and editing

## Discover
- `GET /api/v1/images/models` (no key) - normalized catalog; each model also carries an `endpoints` link.
- `GET <that endpoints link>` (e.g. `/api/v1/images/models/{modelId}/endpoints`) - per-provider route details: `supported_parameters`, `allowed_passthrough_parameters`, image-input constraints, and itemised `pricing` entries (`billable`, `unit`, `cost_usd`, per resolution).
- Legacy equivalent: `GET /api/v1/image-models?detailed=true`.

Fields to filter and compare:
- `architecture.modality` / `input_modalities` - text-only generation vs. text+image (reference/edit) vs. image-only transforms
- `capabilities`: `image_generation`, `image_to_image`, `inpainting`, `nsfw`
- `supported_parameters`: `resolutions`, `aspect_ratio`, `max_images` (input references), `max_output_images`, quality/rendering-speed options, seed support
- `description`, `tags`, `label` - strengths (text rendering, photorealism, style, consistency), "new" markers

## Read pricing
- `pricing.per_image` is keyed by resolution (and sometimes `by_rendering_speed` or add-ons such as LoRA). Cost = price for the chosen key × output count × planned iterations.
- The endpoints link gives the authoritative per-route price list.
- Exact quotes (deterministic, no key): the OpenAI-compatible generation and JSON edit endpoints - see `scripting.md`.

## Choose
- Needs an input image (edit, reference, style, identity/character consistency) → `image_to_image` true; several references → check the maximum input image count.
- Needs legible text in the image, a particular style, or photorealism → read descriptions/tags; offer a cheap and a premium option.
- Output resolution and aspect ratio must satisfy the downstream stage (video input limits, 3D input expectations, print size) - choose the smallest resolution that does.
- Iterate at the cheapest acceptable resolution/tier, then produce the final at the needed tier.
- Only consider `nsfw` models when the user's request actually requires it and it is appropriate.

## Call
Normalized (preferred, JSON only):
`POST /api/v1/images` `{model, prompt, n, resolution, aspect_ratio, quality, output_format, seed, input_references: [url | data URL | {type:"image_url", image_url:{url}}]}` plus any model-specific keys listed in `supported_parameters`. Do not mix `input_references` with legacy image fields in one request. No streaming. Full field list: `https://docs.nano-gpt.com/api-reference/endpoint/image-api-generate.md`.

OpenAI-compatible (quote-capable):
- `POST /api/v1/images/generations` `{model, prompt, n, size, response_format: "b64_json"|"url", imageDataUrl | imageDataUrls, maskDataUrl, strength, seed, ...}`
- `POST /api/v1/images/edits` (JSON with data URLs, or multipart)

Responses: `data[i]` holds either `b64_json` or `url` (signed, roughly 1-hour expiry); usually also `cost` and `remainingBalance`. Save every image locally immediately.

Some model families run asynchronously and return a task ID instead of images; their status route is listed in the docs index (`https://docs.nano-gpt.com/llms.txt`).

## Pipeline notes
- For later animation or 3D reconstruction: full subject in frame, clean or plain background, even lighting, the aspect ratio the next model expects.
- For multi-view or character-consistency work: generate views from the same reference with the same model and seed; review consistency before any expensive downstream stage.
- Keep the seed and exact prompt in the manifest so a chosen image can be reproduced at a higher tier.
