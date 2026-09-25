# Web search, scraping, YouTube transcripts

These tools have no model catalog; their price is per call. Get the exact price with an unauthenticated quote (both search and scrape are officially quote-capable with deterministic prices - see `scripting.md`), or read the pricing tables on the linked doc pages.

## Web search
`POST /api/web` (alias `/api/v1/data/web/search`) `{query, provider, depth, outputType, structuredOutputSchema?, includeDomains?, excludeDomains?, fromDate?, toDate?, includeImages?}`
- Providers, depths, operations (some providers also fetch, extract or research) and output types change; read `https://docs.nano-gpt.com/api-reference/endpoint/web-search.md` when choosing.
- `outputType`: raw results, a sourced answer, or structured output against a JSON schema (provider-dependent).
- Alternative: a chat model with a web-search suffix answers with web context in one call (see `text-and-chat.md`), billed as tokens plus a search charge.

Choose: raw results when a later stage will reason over them; sourced answer for a direct cited reply; structured output when the next stage needs fields. Deeper search costs more - use it only when the standard depth is insufficient.

## Scraping
`POST /api/scrape-urls` (alias `/api/v1/data/url/scrape`) with one or more URLs → cleaned content (markdown) plus raw HTML. Use it to turn pages into clean text for a later text stage.

## YouTube transcripts
`POST /api/youtube-transcribe` `{urls: [...]}` → transcript per URL with per-video success/failure. Much cheaper than downloading audio and running speech-to-text; use it whenever the source is a YouTube link.

## Other data tools
The unified data API (`/api/v1/data/...`) exposes additional lookup tools with discovery metadata: `https://docs.nano-gpt.com/api-reference/endpoint/data-api.md`.
