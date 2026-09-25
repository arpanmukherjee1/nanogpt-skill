---
name: nanogpt
description: Plan, cost-estimate and run work on the user's own NanoGPT (nano-gpt.com) account - text and chat across hundreds of models, image generation and editing, video generation, speech, voice cloning, transcription, music and sound, 3D model (GLB) generation, embeddings, moderation, web search and scraping, batch jobs - and chain them into multi-stage pipelines. Use this skill whenever the user mentions NanoGPT or nano-gpt, wants to generate, convert, clone or transcribe anything through it, asks what a NanoGPT workflow would cost or which model to use, or describes a multi-step generation pipeline while this skill is attached. The skill always analyses the workflow, picks models and estimates the full pipeline cost with the user before making any paid call.
---

# NanoGPT

Every generation call spends the user's real balance. Treat each paid call as spending their money, and never make one before the pre-flight in section 2 is complete and approved.

## 1. The credential: `nanogpt.key`

The key lives in `nanogpt.key`, in the same directory as this SKILL.md.

- **Never read it.** No `cat`, `head`, `tail`, `less`, `view`, `grep`, `sed`, `strings`, `xxd`, `base64`, no `python -c` that prints it, no recursive search/listing-with-contents of the skill directory unless the key file is excluded. Never copy, move, rename, archive or upload it, and never place it under the outputs directory.
- **Only scripts use it.** A script you write opens the file, holds the value in memory, and puts it into the HTTP auth header. Nothing else. The script must never print, log, return, interpolate into a URL/filename/error, or pass it on the command line or through an environment variable you set in a visible command.
- If the script finds the file missing, empty or still containing the placeholder, it exits with a message that does not contain the key; tell the user to fill in `nanogpt.key`. Never ask the user to paste the key into the chat, and never work around a missing key.
- The loader pattern and all scripting rules are in `references/scripting.md`. Read it before writing the first script in a conversation.

## 2. Mandatory pre-flight - before ANY paid call

Sequence: **Analyse → Decompose → Discover → Estimate → Present & Ask → Confirm.**
Allowed during pre-flight: free, unauthenticated catalog lookups and unauthenticated price quotes (see `references/scripting.md`). Not allowed: anything that can charge the balance.

### 2.1 Analyse the request
Extract and write down:
- The final deliverable(s): modality, file format, count, and size parameters (length, duration, resolution, aspect ratio, poly budget, language, voice…).
- Inputs already supplied vs. inputs still missing.
- Constraints: budget ceiling, quality bar, speed/deadline, where the result will be used, content limits.
- What "done" means, and every point where the request is ambiguous.

### 2.2 Decompose into a workflow
- Express the goal as ordered **stages**. A stage is one capability that turns one or more input artifacts into an output artifact.
- For each stage record: capability domain (→ reference file), input modality and format, output modality and format, synchronous or asynchronous, and which stages it depends on.
- Merge stages when one model can do both steps adequately; split a stage when a cheaper specialist exists, when a quality checkpoint is needed, or when an intermediate artifact must be reused.
- Check every edge: the output of one stage must be an accepted input of the next (modality, file type, size, duration, aspect ratio, prompt length). Resolve mismatches now - add a transform stage or pick a different model - not mid-run.

### 2.3 Discover candidate models per stage
- Read the stage's reference file, then query that domain's live catalog (free, no key). Filter by capability flags and input/output modalities.
- Shortlist 2–3 candidates that span price/quality tiers. For each, note why: capability fit, input constraints satisfied, quality signals in name/description/label/tags, typical speed, price unit.
- Never rely on remembered model IDs, prices or parameters. Catalogs change constantly; only the live catalog counts.

### 2.4 Estimate cost and time
- Per stage: catalog price unit × the planned parameters (count, resolution, seconds, characters, tokens, variant). How to read each domain's pricing is in its reference file.
- Prefer an exact unauthenticated quote wherever the endpoint supports it; otherwise compute from the catalog and label it an estimate.
- Add expected iterations (drafts, variants, re-rolls) explicitly.
- Produce a pipeline total as a range: **low** = every stage succeeds first try at chosen settings; **high** = planned iterations plus one retry of the most failure-prone stage.
- Estimate wall-clock time from typical durations; call out long asynchronous stages.

### 2.5 Present the plan and ask
- Present compactly: one row per stage - recommended model, main alternative, key parameters, cost, time - followed by the total range and the assumptions behind it.
- Ask only questions whose answers change model choice, cost or feasibility: quality vs. cost tier, missing inputs, output specifications, iteration budget, where the user wants review checkpoints.
- Use the interactive question tool when one is available; at most three questions per round, each option mapped to a concrete plan variant with its cost.
- Revise the plan and estimate after the answers; repeat only if something material changed.

### 2.6 Confirm, then run
- Proceed only after explicit approval of the plan and its total.
- The first script run checks the account balance (free; see `references/account.md`). If the balance is below the high estimate, say so before any paid call.
- Stop and re-confirm when: the plan changes; a stage's actual cost exceeds its estimate by more than ~25%; cumulative spend is about to pass the approved total; or a stage fails in a way that requires re-running a paid step.

## 3. Composing pipelines

**Artifacts and handoffs**
- Every stage writes its output to a local file in the working directory; deliverables are copied to the outputs directory. Never chain stages through remote URLs alone.
- Result URLs are signed and short-lived: download them immediately after completion.
- Pass artifacts to the next stage in the form its endpoint accepts (inline data URL, public URL, or multipart upload) and within its limits; resize, trim, transcode or re-encode locally when required.
- Measure what downstream pricing depends on (audio duration, text length, image size) from the actual artifact before the next stage runs, and update the estimate if it moved.

**Ordering and checkpoints**
- Run cheap and high-uncertainty stages first. Put a user review checkpoint before every expensive stage and after every stage whose output is subjective.
- When a model family offers draft, low-resolution, short or geometry-only variants, iterate there and pay for the final tier once.
- Independent branches may run in parallel; dependent stages run strictly in order. A failed stage blocks everything downstream of it.

**Asynchronous stages**
- Submit → record the job ID in the manifest immediately → poll with backoff → enforce a timeout → download the result.
- Never re-submit a job whose status is unknown or still pending; query it by its ID first. Re-submitting pays twice.

**Manifest and cost ledger**
- Keep one JSON manifest per pipeline in the working directory. Per stage: model, parameters, job/request ID, status, artifact paths, estimated cost, actual cost (from the response's cost field), timestamps.
- The manifest makes a pipeline resumable without paying twice and is the source of the final spend report.

**Failure policy**
- 400-class errors (validation, auth, insufficient balance, content policy): stop; fix the request or ask the user. Never retry blindly.
- 429 errors share one status but differ by `error.code` - always read it:
  - `rate_limit_exceeded` (per-second throughput): bounded retries with exponential backoff, respecting `Retry-After`.
  - `daily_rpd_limit_exceeded` / `daily_usd_limit_exceeded` (the key's own daily request or spend cap): never retry. Stop the pipeline, record the stage as blocked in the manifest, and tell the user the error message (it states the count and reset time) plus `Retry-After` (seconds until 00:00 UTC). Ask whether to resume after the reset, or whether they will raise the cap. The spend cap is checked against estimated cost at request time, so it can block a stage even when the balance is sufficient.
- 500-class errors: bounded retries with exponential backoff.
- Failed asynchronous jobs are usually refunded; confirm from the status/response before paying for a re-run, and tell the user.

**Delivery**
- Present the final artifacts, list kept intermediates, and report actual spend vs. estimate per stage from the ledger.

## 4. Reference map

Read only the references the current plan needs.

| Need | Reference |
|---|---|
| Writing any script: key loader, HTTP/poll/download helpers, catalog queries, price quotes, manifest | `references/scripting.md` |
| Text / chat / reasoning / tool use / structured output / batch | `references/text-and-chat.md` |
| Image generation, editing, reference-guided images | `references/image.md` |
| Video generation, image/video/audio-to-video, extend, upscale | `references/video.md` |
| Speech, voices, voice cloning, transcription, music, sound effects | `references/audio.md` |
| 3D model (GLB) generation | `references/3d.md` |
| Embeddings, moderation, NSFW checks | `references/embeddings-and-moderation.md` |
| Web search, scraping, YouTube transcripts | `references/web-tools.md` |
| Balance, usage, per-request billing, scoped keys | `references/account.md` |

## 5. Platform conventions

- Base host for every call, catalogs included: `https://api.nano-gpt.com` (all routes below are verified there; `https://nano-gpt.com` serves the same routes).
- Auth header: `x-api-key: <key>` or `Authorization: Bearer <key>`.
- Model catalogs are public, free and unauthenticated.
- Asynchronous jobs share one shape: submit → `202` with a job ID → poll a status endpoint → terminal completed/failed state → result URL.
- Anything not covered by the references: docs index `https://docs.nano-gpt.com/llms.txt`, OpenAPI spec `https://docs.nano-gpt.com/api-reference/openapi.json`. Both are incomplete - some live features are undocumented (see `references/3d.md`) - so verify before concluding a feature is absent.
