# Video generation

## Discover
`GET /api/v1/video-models?detailed=true` (no key). Fields:
- `architecture.modality` / `input_modalities` - text, image, video and/or audio in; video out
- `capabilities`: `text_to_video`, `image_to_video`, `video_to_video` (extend, edit, upscale, restyle), `audio_input` (lip-sync / avatar / speech-driven), `audio_generation` (native soundtrack), `fps_control`, `nsfw`
- `supported_parameters.parameters` - an object whose keys are the request fields the model accepts; each has `type`, `label`, `description`, `default`, `options`. Send only these keys, with values from `options`.
- `description`, `tags`, `label` - mode details, strengths, required inputs

## Read pricing
Video pricing shapes vary per model. Read every key in `pricing`:
- per second by resolution (+ default duration/resolution) → seconds × rate for the chosen resolution
- per second with a list of supported durations → seconds × rate
- per duration step or per video with a fixed duration → flat price for that step
- base price by resolution × duration multipliers/overrides
- with-audio vs. without-audio prices, or an audio multiplier
- separate rates for edit/reference modes or quality vs. speed modes, and minimum/maximum billable durations
- `minimum` + `note`, or `raw` → the catalog cannot give the full price; get a quote

Quotes: send the exact submit body to the submit endpoint without a key and with the quote header (see `scripting.md`). It is not on the official quote list but has returned exact prices in testing; fall back to catalog arithmetic otherwise. The submit response also reports the pre-charged `cost`.

## Choose
- Input fit first: which modalities the stage actually has (prompt only, start image, source video, driving audio).
- Duration and resolution options must cover the deliverable; prefer the shortest duration and lowest resolution that satisfy it, especially for drafts.
- Needs native sound → `audio_generation`; needs speech driven by a given audio track → `audio_input`.
- Extend/upscale/edit of an existing clip → `video_to_video` plus description of the mode; check the maximum source length.
- Compare price per final second, typical speed and quality signals; offer tiers.

## Call
Submit: `POST /api/generate-video`
`{model, prompt, <parameter keys from supported_parameters>, imageDataUrl | imageUrl, videoUrl | videoDataUrl, audioDataUrl | audioUrl, referenceImages, referenceVideos}` - send only the input fields the model uses.
- Inline data URLs are limited to about 4 MB; larger media must be passed as a public HTTPS URL.
- Response `202`: `{runId, status: "pending", cost (pre-charge), remainingBalance}`.

Poll: `GET /api/video/status?requestId=<runId>`
- `data.status`: `IN_QUEUE` → `IN_PROGRESS` → `COMPLETED` | `FAILED` | `CANCELED`
- `COMPLETED` → `data.output.video.url` and final `data.cost`
- `FAILED` → `data.error` / `userFriendlyError`; failed jobs are normally refunded automatically

Other routes:
- `GET /api/generate-video/recover?limit=` - list recent runs (recovers lost job IDs; never re-submit instead)
- Some model families have special extend or content-retrieval routes; they are listed in the docs index (`https://docs.nano-gpt.com/llms.txt`) and the video guide `https://docs.nano-gpt.com/api-reference/video-generation.md`.

Limits: submission is rate-limited (on the order of 50 requests/minute); polling every few seconds with backoff is expected.

## Pipeline notes
- A start image must match the model's accepted aspect ratio and size; prepare it in the image stage.
- For audio-driven video, the audio's duration sets both the video length and the per-second cost - measure it locally before estimating.
- Video jobs can take minutes; record the run ID in the manifest before polling and set a generous timeout.
