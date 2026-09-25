# NanoGPT Skill

A Claude skill that plans, prices and runs work on your own [NanoGPT](https://nano-gpt.com) account. It covers chat across hundreds of models, image generation and editing, video, speech, voice cloning, transcription, music and sound effects, 3D models (GLB), embeddings, moderation, web search and scraping, and batch jobs. It can also chain them into multi-stage pipelines.

Built for [Claude.ai](https://claude.ai) with code execution turned on. It also installs as a [Claude Code](https://code.claude.com/docs/en/skills) skill.

## What Makes This Different

- **Cost pre-flight before any paid call.** Claude splits the request into stages, shortlists 2–3 models per stage from NanoGPT's live catalogs, and prices each stage. It uses NanoGPT's free unauthenticated price quote where the endpoint offers one and calculates from the catalog otherwise. You get a per-stage plan with a low–high total, and nothing that spends money runs until you approve it.
- **Live catalogs, not remembered prices.** Model IDs, parameters and prices are read at run time, because NanoGPT's catalog changes constantly.
- **Resumable pipelines with a cost ledger.** Every run keeps a JSON manifest of job IDs, files and actual costs. An interrupted pipeline resumes without paying twice, and at the end you get actual spend vs. estimate for each stage.
- **Spending safeguards.** It checks your balance first. It asks again if a stage runs more than ~25% over its estimate or total spend is about to pass what you approved. It never resubmits an async job whose status is unknown. It retries per-second rate limits but stops and asks when your key's daily cap is hit.
- **The key stays out of the conversation.** The skill tells Claude never to read, print or log the key. Only the Python scripts it writes open `nanogpt.key`, and only to set the auth header.
- **No dependencies.** Scripts use only the Python standard library.

## Installation

### Claude.ai

1. Clone the repo into a folder named `nanogpt`. Claude.ai rejects a skill whose folder name doesn't match the skill name.

   ```bash
   git clone https://github.com/arpanmukherjee1/nanogpt-skill.git nanogpt
   ```

   Or use **Code → Download ZIP**, extract it, and rename the folder to `nanogpt`.

2. Add your API key (see [Setup](#setup)).

3. Zip the folder so the archive contains `nanogpt/SKILL.md`:

   ```bash
   # macOS / Linux
   zip -r nanogpt.zip nanogpt -x 'nanogpt/.git/*'

   # Windows (PowerShell or cmd; tar ships with Windows 10 and later)
   tar -a -c -f nanogpt.zip --exclude=.git nanogpt
   ```

4. In Claude, make sure **Code execution and file creation** is on (Settings → Capabilities). Then open [Customize → Skills](https://claude.ai/customize/skills), click **+** → **Create skill** → **Upload a skill**, and choose `nanogpt.zip`.

The zip contains your key, so keep it private and never commit or share it.

### Claude Code

Install it as a personal skill so the key lives outside your projects:

```bash
git clone https://github.com/arpanmukherjee1/nanogpt-skill.git ~/.claude/skills/nanogpt
```

Add your key (see [Setup](#setup)) and start a new session. To update later, run `git pull` in that folder.

The reference files were written for Claude.ai's sandbox and use its paths (`/home/claude/` for working files, `/mnt/user-data/outputs/` for deliverables). In Claude Code, tell Claude where you want the results saved.

## Setup

1. Create an API key at [nano-gpt.com/api](https://nano-gpt.com/api). Use a dedicated key rather than your main one so you can revoke it at any time. If your account supports spending limits, give it a daily cap so a pipeline can't drain your balance.
2. In the skill folder, next to `SKILL.md`, copy the template and replace the placeholder with your key. The file holds only the key.

   ```bash
   cp nanogpt.key.example nanogpt.key
   ```

`nanogpt.key` is in `.gitignore`, so git won't pick it up. If the file is missing, empty or still holds the placeholder, the scripts stop and Claude asks you to fill it in. The skill tells Claude never to ask you to paste the key into the chat.

## Usage

Mention NanoGPT in your request so Claude picks up the skill:

> "Using NanoGPT, turn this product photo into a 10-second video with background music. What would it cost?"

> "Transcribe this 40-minute interview on NanoGPT with speaker labels, then summarise it."

> "Make a textured 3D model of a ceramic teapot with NanoGPT. Draft it cheaply first."

Claude replies with a plan before spending anything. Each stage gets a row with the recommended model, an alternative, key settings, cost and time. Below that are the total range and the questions whose answers change the price. It starts only after you approve.

## File Structure

```
nanogpt/
  SKILL.md                          # Key rules, cost pre-flight, pipelines, failure policy
  nanogpt.key.example               # Template: copy to nanogpt.key and add your key
  references/
    scripting.md                    # Script template: key loader, HTTP/poll/download, quotes, manifest
    text-and-chat.md                # Chat, reasoning, structured output, Batch API
    image.md                        # Image generation and editing
    video.md                        # Video generation, extend, upscale
    audio.md                        # Speech, voice cloning, transcription, music, sound effects
    3d.md                           # 3D model (GLB) generation
    embeddings-and-moderation.md    # Embeddings, moderation, NSFW checks
    web-tools.md                    # Web search, scraping, YouTube transcripts
    account.md                      # Balance, usage, per-request billing, scoped keys
```

## Disclaimer

This is an independent project, not affiliated with NanoGPT or Anthropic. Once you approve a plan, the skill spends real money from your NanoGPT balance. Estimates come from NanoGPT's catalogs and quotes, and actual charges can differ. The 3D route isn't in NanoGPT's public docs (see `references/3d.md`) and may change without notice.

## License

MIT, see [LICENSE](LICENSE).
