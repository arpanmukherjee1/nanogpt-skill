# Embeddings, moderation and NSFW checks

## Embeddings
Discover: `GET /api/v1/embedding-models` (no key). Fields: `id`, `name`, `description` (language/domain focus), `dimensions`, `supports_dimensions`, `max_dimensions`, `max_tokens`, `pricing.per_million_tokens`.

Pricing: total tokens embedded ÷ 1e6 × rate (tokens ≈ characters ÷ 4). Storage cost scales with dimensions - note it when the user will store vectors.

Choose: language and domain fit (multilingual, code, …) from the description; `max_tokens` ≥ chunk size; smaller or reducible dimensions when storage/speed matter; cheapest model that fits for bulk.

Call: `POST /api/v1/embeddings` `{model, input: string | [strings], dimensions?, encoding_format?}` - OpenAI-compatible; batch many inputs per request.

## Moderation
Discover: `GET /api/v1/moderation-models` (no key). Fields: `capabilities` (`text_moderation`, `image_moderation`, `custom_policy`), `context_length`, `pricing` (per million tokens).

Call: `POST /api/v1/moderations` `{input, model?}` → `results[].flagged`, `categories`, `category_scores`. Input may be text, an array, or content parts with images.

Inline preflight: a paid safety check can be attached to supported generation requests via a request header - details at `https://docs.nano-gpt.com/api-reference/miscellaneous/inline-moderation.md`.

## NSFW image classification
`POST /api/nsfw/image` - up to 10 image URLs or data URLs per request; binary result per image.

## Pipeline notes
- When inputs come from third parties or users, a moderation stage before expensive generation avoids paying for jobs that will be rejected by content policy.
- Embedding stages are usually the cheapest in a pipeline; cache vectors in the working directory so re-runs do not pay again.
