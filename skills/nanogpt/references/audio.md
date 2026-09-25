# Audio: speech, voice cloning, transcription, music, sound effects

## Discover
`GET /api/v1/audio-models?detailed=true` (no key). One catalog covers every audio capability. Filter on:
- `capabilities`: `text_to_speech`, `speech_to_text`, `voice_clone`, `text_to_music`, `text_to_audio` (sound effects), `audio_to_audio`, `diarization`, `word_timestamps`, `stem_separation`, `music_extension`, `music_cover`, `audio_inpainting`, `audio_reference`, and others as they appear
- `architecture.modality` - e.g. text→audio, audio→text, text+audio→audio
- `supported_parameters` - `voices`, `max_chars`, `min_duration` / `max_duration`, `response_formats`, `timestamp_granularities`, `supported_formats`, `max_file_size_mb`, `max_request_body_mb`, `supports_remote_url`
- `description` - languages, streaming, style controls, clone compatibility

## Read pricing
Shapes differ per model; read every key:
- `per_thousand_chars` → characters of text ÷ 1000 × rate (speech)
- `per_minute` → audio minutes × rate (transcription)
- `per_second` (+ `minimum`) → seconds × rate, not below the minimum (music, sound effects)
- `per_generation` → flat per call (predictable; often clone or music models)
- `multiplier_parameter` (+ min/max) → price scales with the named request parameter
- `per_billing_interval` / `per_prompt_char_block` → price per interval or per block of prompt characters
Measure real text length or audio duration locally before estimating dependent stages.

## Choose
- Speech: voice and language availability (`voices`, description), clone compatibility, `max_chars` per request (split long scripts), streaming need, price per 1k characters × script length.
- Transcription: diarization, word timestamps, output formats, file size limits and remote-URL support, price per minute × duration.
- Music/sound effects: duration range, flat vs. per-second pricing for predictability, whether reference audio or extension is supported.
- Voice cloning: models with `voice_clone`; check what the clone produces and how it is reused (see below) and its retention rules.

## Call
Speech, music, sound effects (sync, returns raw audio bytes):
`POST /api/v1/audio/speech` `{model, input, voice, response_format, speed, instructions, stream}` - for music/SFX models `input` is the prompt and `voice` is ignored; unsupported fields are ignored per model. Check `content-type` and save the bytes.

Speech (native, may be async):
`POST /api/tts` `{model, text, voice, speed, ...model-specific fields}` → `200` with audio bytes or JSON `audioUrl`, or `202` with `{runId, model}` → poll `GET /api/tts/status?runId=&model=` until `status: "completed"` with `audioUrl`.

Transcription:
- OpenAI-compatible multipart: `POST /api/v1/audio/transcriptions`
- Native: `POST /api/transcribe` - JSON `{audioUrl, model, language, diarize, ...}` or multipart with an `audio` file for small files. `200` → result; `202` → poll `POST /api/transcribe/status` with the identifying fields returned by the submit response (`runId`, `cost`, `paymentSource`, file info, …).

Voice cloning (asynchronous, provider-specific):
- `POST /api/voice-clone/<provider>` then `POST /api/voice-clone/<provider>/status` with the returned `runId`.
- Current providers, request fields and output types: fetch `https://docs.nano-gpt.com/api-reference/endpoint/voice-cloning.md` at run time. A clone yields either a reusable voice ID (passed as `voice` to a compatible speech model) or an embedding-file URL (passed in a provider-specific speech field).
- Clone artifacts have retention limits (unused voice IDs can be deleted; hosted embedding files can expire). Persist returned IDs/files immediately and tell the user the retention rule stated in the docs.
- If a clone-capable catalog model has no documented route, say so; do not invent one.

YouTube transcripts are a separate, cheaper tool - see `web-tools.md`.

## Pipeline notes
- The reference clip for cloning should be clean speech in the target language; ask for it or its length if missing.
- Generated narration feeds per-second video stages - produce it at the final length first, measure it, then estimate the video stage.
- Match the audio format and sample limits the next stage accepts; transcode locally if needed.
