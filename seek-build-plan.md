# Seek: Hermes Agent Build Plan

> **For the next Claude reading this:** this document is the full context for a project I (the user) started in a previous chat. Memory is off, so I'm pasting this in to get you up to speed. Treat the architecture decisions below as already made — don't re-litigate them unless I explicitly ask. Help me execute.

---

## Who I am and what I'm building

I'm a California surf-mama who recently automated ~75% of my day job using Claude. I'm comfortable in the terminal and can follow technical instructions, but I'm not a professional developer — explain things at the level of an enthusiastic technical hobbyist, not a senior engineer.

I just got a Mac mini (M4, 16GB RAM) after a 6-week wait caused by the AI-driven global RAM shortage ("RAMpocalypse" / "RAMmageddon"). The 16GB constraint is real and shapes the architecture below.

I'm building an autonomous AI agent named **Seek**.

## What Seek does

Three jobs, in priority order:

1. **Web crawling** — find things I care about based on hand-written skill specs I've already drafted. This is the first thing we wire up.
2. **Blog writing** — Seek writes a blog telling me what she found, what she's thinking about, what she's learned. Voice is TBD and will be developed through iteration; the first few posts will be rough.
3. **Doom playing** — via VizDoom in slowed/deliberate mode, using Hermes's automatic skill creation to build a tactical playbook over time. Modeled on NVIDIA's Voyager project (GPT-4 + Minecraft, 2023). Realistic ceiling: clears classic Ultimate Doom on Hurt Me Plenty difficulty, builds 30–50 tactics skills. Won't beat speedrunners.

## Architecture decisions (already made)

### Framework: Hermes Agent (Nous Research, MIT-licensed)

Chosen specifically for:
- **Automatic skill creation** — when Hermes solves a hard problem, it writes a reusable skill document to `~/.hermes/skills/` so it doesn't have to re-figure-out next time. This is the headline feature and the whole reason I picked Hermes.
- **OpenAI-compatible provider abstraction** — works with OpenRouter, Ollama, Anthropic, etc.
- **Sub-agent delegation** — different parts of the system can use different models. Critical for the cost structure below.
- **Skills directory at `~/.hermes/skills/`** — bundled skills, hub-installed skills, and auto-generated skills all live in the same folder. My hand-written specs drop in alongside.

### Hybrid model setup (because of 16GB constraint)

| Layer | Model | Used for |
|-------|-------|----------|
| Cloud (OpenRouter) | Claude / DeepSeek / GPT — TBD per task | Main agent loop, skill authoring, blog writing — anything where model quality matters |
| Local (Ollama) | `hermes3:8b` (~4.7GB, good function calling) | Doom in-loop decisions, page parsing, grunt work — anywhere local-fast-cheap beats smart-but-paid |

OpenRouter funded with **$10 in credits**, which:
- Unlocks 1000 free-model requests/day (vs. the free-tier 50/day, which is too tight for agent loops)
- Enables pay-as-you-go on premium models for skill authoring + blog writing

### Blog stack

- **Astro + Vercel + my existing GitHub account.** No separate "Seek" GitHub account — instead, inside the blog repo only (no `--global`), git config sets:
  ```
  user.name = "Seek"
  user.email = "MYID+seek@users.noreply.github.com"
  ```
  Commits show under my avatar but with Seek as the author byline. Get the noreply email from GitHub → Settings → Emails.

---

## Setup steps

### 1. macOS first-boot
Account, FileVault, iCloud. ~10 min, no Claude needed.

### 2. Homebrew
```bash
/bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh)"
echo 'eval "$(/opt/homebrew/bin/brew shellenv)"' >> ~/.zprofile
eval "$(/opt/homebrew/bin/brew shellenv)"
```

### 3. Foundation tools
```bash
brew install git uv node ripgrep fd jq
```

### 4. Ollama + local model
```bash
brew install --cask ollama
open -a Ollama       # accept the helper prompt
ollama pull hermes3:8b
ollama list          # confirm it's there
```

### 5. Hermes Agent
```bash
curl -fsSL https://raw.githubusercontent.com/NousResearch/hermes-agent/main/scripts/install.sh | bash
hermes --version
```

### 6. OpenRouter
1. Sign up at https://openrouter.ai
2. Add $10 in credits
3. Generate an API key, save it somewhere safe
4. Run `hermes model` and pick OpenRouter as primary provider, paste the key when prompted

### 7. Project workspace (separate from `~/.hermes/`)
```bash
mkdir -p ~/code/construct && cd ~/code/construct
uv init && uv venv && source .venv/bin/activate
uv add vizdoom playwright httpx
playwright install chromium
```

### 8. Blog repo
1. Create a new repo on GitHub (e.g., `seek-blog`) under my existing account
2. Bootstrap with Astro:
   ```bash
   cd ~/code
   npm create astro@latest seek-blog
   ```
   Pick a blog template, install dependencies.
3. Connect to Vercel (free tier) — point it at the repo, enable auto-deploy on push to main.
4. Configure Seek's commit identity inside the repo only:
   ```bash
   cd ~/code/seek-blog
   git config user.name "Seek"
   git config user.email "MYID+seek@users.noreply.github.com"
   ```

### 9. Drop hand-written skill specs into `~/.hermes/skills/`
My specs go here. They sit alongside the 70+ bundled skills Hermes ships with and any skills Seek auto-generates later. Start with one web-crawl skill — tighter loop than Doom, easier to debug.

### 10. Hermes delegation config
Edit `~/.hermes/config.yaml` so sub-agents route to local Ollama (cheap grunt work) while the main agent stays on OpenRouter (smart planning + skill authoring + blog writing). Exact syntax: ask the next Claude.

---

## Open questions / TBD

- **Seek's voice for the blog** — undecided, will develop through iteration. Vibes I might want: "dry observational" / "stoked surf-friend" / "deadpan archivist." Lock it down once we have raw output to react to.
- **VizDoom configuration** — must run in slowed/deliberate mode, not real-time, for the LLM loop to close. LLM inference latency (0.5–5s) ≠ Doom's 35fps. Game pauses between agent actions.
- **Skill execution vs. authoring split** — skills should be *authored* by the cloud model (where smartness matters) and *executed* by either the cloud model OR the local 8B depending on the skill. Hermes delegation supports this.
- **First skill to actually wire up** — TBD, will be one of my hand-written web-crawl specs. I'll pick one when I'm ready to wire it in.

---

## How to help me

When I paste this into a new chat, I'll tell you which step I'm on. Help me through the remaining steps, then once Hermes is installed and configured, help me turn one of my hand-written skill specs into a working Hermes skill in `~/.hermes/skills/`.

If anything in the architecture above looks wrong to you given current Hermes Agent state, flag it — but don't unilaterally redesign. Verify with a search if you're unsure.
