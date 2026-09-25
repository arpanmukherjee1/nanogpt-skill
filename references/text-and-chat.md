# Text, chat, reasoning and batch

## Discover
`GET /api/v1/models?detailed=true` (no key). Useful fields per model:
- `id`, `name`, `description`, `category`, `aliases`, `open_weights`, `providers`
- `context_length`, `max_output_tokens`
- `architecture.input_modalities` / `output_modalities`
- `capabilities`: `vision`, `video_input`, `audio_input`, `reasoning`, `tool_calling`, `parallel_tool_calls`, `structured_output`, `pdf_upload`, `batch_api`
- `reasoning_efforts` (allowed effort levels, when the model supports them)
- `supported_batch_endpoints` (non-empty = usable in the Batch API, and which row endpoints)
- `supported_service_tiers` (speed/price tiers, when offered)
- `subscription.included` (covered by a NanoGPT subscription or not)
- `pricing`, `effective_pricing`, `cost_estimate`

## Read pricing
- `pricing.prompt` and `pricing.completion` are USD per million tokens (`pricing.unit`). Some models add cache read/write prices (per 1k tokens) or extra charges for audio/video input - read every key present.
- `effective_pricing` (when present) shows list vs. currently effective price; use the effective one.
- `cost_estimate` is NanoGPT's rough per-request figure; use it only as a sanity check.
- Estimate: `(input_tokens × prompt + output_tokens × completion) / 1e6`. Tokens ≈ characters / 4 as a first approximation. For an exact input count in Anthropic-message format, use `POST /api/v1/messages/count_tokens`.
- Reasoning models also bill hidden reasoning tokens as output - budget several times the visible answer length, scaled by the chosen effort level.
- Web-search suffixes and similar add-ons carry their own per-call charge on top of tokens.
- Chat and Responses requests can be quoted without a key (quote type: estimate) - see `scripting.md`.

## Choose
- Hard filters first: required input modalities, context length ≥ planned input + output, `max_output_tokens` ≥ planned output, tool calling / structured output when the stage needs machine-readable results.
- Use reasoning-capable models only where the stage genuinely needs multi-step reasoning; use the cheapest adequate model for bulk or mechanical transforms.
- Offline, high-volume, latency-tolerant work: prefer models with non-empty `supported_batch_endpoints` and use the Batch API (token price about 50% below synchronous).
- If the user has a subscription, models with `subscription.included` may cost nothing extra - ask.
- Offer the user 2–3 options spanning cost/quality with the per-request estimate for each.

## Call
- OpenAI-compatible: `POST /api/v1/chat/completions` `{model, messages, max_tokens, stream, temperature, tools, response_format, ...}`
- Responses API: `POST /api/v1/responses` (tools, structured `text.format`, stateful threading)
- Anthropic-compatible: `POST /api/v1/messages`
- Streaming: `stream: true` plus header `Accept: text/event-stream`; parse SSE until `data: [DONE]`.
- Model suffixes (routing preference, web search, thinking, caching, …) are appended to the model ID. The current list lives at `https://docs.nano-gpt.com/api-reference/miscellaneous/model-suffixes.md` - fetch it when a suffix is needed; suffixes can change billing (e.g. forcing pay-as-you-go).

## Batch API
- Only chat-completions and responses rows are batchable; image, video, audio, 3D and other endpoints are not.
- All rows in one batch: same endpoint, same model; non-streaming; every row needs an output cap.
- File flow: `POST /api/v1/files` (multipart, `purpose=batch`, JSONL) → `POST /api/v1/batches` `{input_file_id, endpoint, completion_window: "24h"}` → poll `GET /api/v1/batches/{id}` → `GET /api/v1/files/{output_file_id}/content`.
- Inline flow (smaller jobs): `POST /api/beta/batches` `{endpoint, model, requests: [{custom_id, body}]}` → poll `GET /api/beta/batches/{id}` and read `results`.
- Balance is checked against a conservative maximum at creation; billing happens once at the end. Full details: `https://docs.nano-gpt.com/api-reference/endpoint/batches.md`.

## Pipeline notes
- Text stages often produce prompts, scripts or captions for later stages. Force machine-readable output (structured output / JSON schema) and enforce the downstream model's prompt-length and language limits inside the text stage.
- Length of generated narration drives the cost of speech and lip-sync stages - fix a target length in the text stage and measure the result before estimating the next stage.
