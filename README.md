# Arpan's Claude Skills

Claude skills by Arpan Mukherjee, published as a [Claude Code plugin marketplace](https://code.claude.com/docs/en/plugin-marketplaces). Each plugin is a separate set of skills, so you install only the ones you need.

| Plugin | Skills | What it does |
|---|---|---|
| `nanogpt` | `nanogpt` | Plans, prices and runs work on your own [NanoGPT](https://nano-gpt.com) account, with a cost pre-flight before any paid call |

## Install in Claude Code

Run these inside Claude Code:

```
/plugin marketplace add arpanmukherjee1/skills
/plugin install nanogpt@arpan-skills
```

If the first command fails with `Permission denied (publickey)`, add the marketplace over HTTPS instead: `/plugin marketplace add https://github.com/arpanmukherjee1/skills.git`.

Plugin skills carry the plugin's name as a prefix, so this one runs as `/nanogpt:nanogpt`. Claude also picks it up by itself when you mention NanoGPT. To get updates later, run `claude plugin update nanogpt@arpan-skills` in your shell.

## NanoGPT

Lets Claude plan, price and run work on your NanoGPT account: chat across hundreds of models, image generation and editing, video, speech, voice cloning, transcription, music and sound effects, 3D models (GLB), embeddings, moderation, web search and scraping, and batch jobs. It can also chain them into multi-stage pipelines.

- **Cost pre-flight before any paid call.** Claude splits the request into stages, shortlists 2–3 models per stage from NanoGPT's live catalogs, and prices each stage. It uses NanoGPT's free unauthenticated price quote where the endpoint offers one and calculates from the catalog otherwise. You get a per-stage plan with a low–high total, and nothing that spends money runs until you approve it.
- **Live catalogs, not remembered prices.** Model IDs, parameters and prices are read at run time, because NanoGPT's catalog changes constantly.
- **Resumable pipelines with a cost ledger.** Every run keeps a JSON manifest of job IDs, files and actual costs. An interrupted pipeline resumes without paying twice, and at the end you get actual spend vs. estimate for each stage.
- **Spending safeguards.** It checks your balance first. It asks again if a stage runs more than ~25% over its estimate or total spend is about to pass what you approved. It never resubmits an async job whose status is unknown. It retries per-second rate limits but stops and asks when your key's daily cap is hit.
- **No dependencies.** Scripts use only the Python standard library.

### Set up your API key

1. Create an API key at [nano-gpt.com/api](https://nano-gpt.com/api). Use a dedicated key rather than your main one so you can revoke it at any time. If your account supports spending limits, give it a daily cap so a pipeline can't drain your balance.
2. Save the key, and nothing else, in `~/.config/nanogpt/nanogpt.key`:

   ```bash
   # macOS / Linux
   mkdir -p ~/.config/nanogpt && nano ~/.config/nanogpt/nanogpt.key
   chmod 600 ~/.config/nanogpt/nanogpt.key
   ```

   ```powershell
   # Windows (PowerShell)
   mkdir -Force $HOME\.config\nanogpt; notepad $HOME\.config\nanogpt\nanogpt.key
   ```

The key file lives outside the plugin, so updates don't touch it. The skill tells Claude never to read, print or log the key. Only the Python scripts it writes open the file, and only to set the auth header. If the key is missing, Claude tells you where to put it. The skill tells it never to ask you to paste the key into the chat.

### Usage

Mention NanoGPT in your request:

> "Using NanoGPT, turn this product photo into a 10-second video with background music. What would it cost?"

> "Transcribe this 40-minute interview on NanoGPT with speaker labels, then summarise it."

> "Make a textured 3D model of a ceramic teapot with NanoGPT. Draft it cheaply first."

Claude replies with a plan before spending anything. Each stage gets a row with the recommended model, an alternative, key settings, cost and time. Below that are the total range and the questions whose answers change the price. It starts only after you approve.

In Claude Code, scripts and intermediate files go in a `nanogpt-work/` folder in the current directory, and results are saved to the current directory unless you name another folder.

### Use it on Claude.ai

The same skill works as a custom skill on Claude.ai. There the key has to be inside the skill folder, because the sandbox has no home directory you can write to.

1. Clone this repo (or use **Code → Download ZIP** and extract it).
2. Copy `skills/nanogpt/nanogpt.key.example` to `skills/nanogpt/nanogpt.key` and replace the placeholder with your key. Git ignores this file.
3. From the repo root, zip the `nanogpt` folder so the archive contains `nanogpt/SKILL.md`:

   ```bash
   # macOS / Linux
   (cd skills && zip -r ../nanogpt.zip nanogpt)

   # Windows (PowerShell or cmd; tar ships with Windows 10 and later)
   tar -a -c -f nanogpt.zip -C skills nanogpt
   ```

4. Make sure **Code execution and file creation** is on (Settings → Capabilities). Then open [Customize → Skills](https://claude.ai/customize/skills), click **+** → **Create skill** → **Upload a skill**, and choose `nanogpt.zip`.

The zip contains your key, so keep it private and never commit or share it. Git ignores `*.zip` in this repo.

### Disclaimer

This is an independent project, not affiliated with NanoGPT or Anthropic. Once you approve a plan, the skill spends real money from your NanoGPT balance. Estimates come from NanoGPT's catalogs and quotes, and actual charges can differ. The 3D route isn't in NanoGPT's public docs (see `skills/nanogpt/references/3d.md`) and may change without notice.

## Repository layout

```
.claude-plugin/
  marketplace.json                  # The "arpan-skills" marketplace: one entry per plugin, each listing its skills
skills/
  nanogpt/
    SKILL.md                        # Key rules, cost pre-flight, pipelines, failure policy
    nanogpt.key.example             # Key template for Claude.ai uploads
    references/                     # Per-domain API guides: text, image, video, audio, 3D, web, account, scripting
```

## Adding a skill

1. Put it in `skills/<name>/SKILL.md`.
2. In `.claude-plugin/marketplace.json`, add its path to an existing plugin's `skills` list, or add a new plugin entry that lists it. A plugin loads only the skills it lists, so one repo can offer several separate plugins.
3. Run `claude plugin validate .` and push.

Never rename a published plugin. Installs are recorded under the plugin's name, so a renamed plugin disappears for existing users.

## License

MIT, see [LICENSE](LICENSE).
