# Seek: Hermes Agent Build — Status & Handoff (v3)

> **For the next Claude:** this is a project handoff. Memory is off, so I'm pasting context. As of v3, Seek is **working** — she can browse the web (both via browser_navigate and via terminal/curl) and respond from OpenRouter. The two-day install slog is closed. This handoff captures what I now know vs. what v2 thought it knew, plus where I'm headed next. Don't re-litigate decisions unless I ask.

---

## Who I am

Cali Kafka — California surf-mama, almost 40, automated ~75% of my day job using Claude. Comfortable in the terminal but not a professional dev. My real passion is creative writing and research; the long-term goal for Seek is a research/writing partner who knows my work, my taste, my open threads. Web crawling, blog writing, and Doom playing are how I'm learning the tooling. Talk to me like an enthusiastic technical hobbyist, not a senior engineer.

Week 11 of the learning curve as of Sunday, May 10, 2026. Day 0 was reading a blog post about an agent writing a hit-piece and realizing my day job was automatable. Eleven weeks later: working agent on a Mac mini.

## What Seek is

An autonomous AI agent built on **Hermes Agent** (Nous Research, MIT-licensed). Three jobs in priority order:

1. **Web crawling** — find things from hand-written skill specs I've drafted ← **working as of tonight, foundation in place**
2. **Blog writing** — Astro + Vercel, repo under my GitHub account, commit identity = "Seek" scoped to that repo only ← not yet wired
3. **Doom playing** — VizDoom in slowed/deliberate mode, modeled on the bundled `pokemon-player` skill ← future

Eventually: research/writing partner that knows my work and helps me think.

## Hardware & architecture

- **Mac mini M4, 16GB RAM** — the AI machine. Lives on a desk, runs Seek.
- **MacBook Air** — the "talking to Claude" machine, where I read/chat/write.
- **Chat bridge between the two** — set up Sat May 9 to make debugging easier.
- **macOS users:**
  - `cali` (Admin) — **where Seek's brain currently lives**, in `/Users/cali/.hermes/`. This is a v2→v3 architectural drift, see below.
  - `seek` (Standard, non-admin) — originally intended to be Seek's home. Currently has a broken Friday install at `/Users/seek/.hermes.broken-friday/` and a stale `hermes` binary at `/Users/seek/.local/bin/hermes` — both unused now. Worth cleaning up later.
- **Hybrid model setup:**
  - **OpenRouter** (funded with $10, weekly cap $5) — main brain. Currently configured for Haiku 4.5 per setup wizard; check `~/.hermes/.env` and `config.yaml` for the active default. NOTE: an earlier auth flow accidentally used Opus 4.6 (very expensive); confirm Haiku 4.5 is the actual default before doing high-iteration work.
  - **Ollama + `hermes3:8b`** locally — installed, working, not yet wired into Hermes's delegation config.

## What's installed and working

Everything in v2 still works, **plus** as of tonight:
- Hermes Agent v0.13.0 on `cali`, fully working `hermes doctor` clean (1 nag about optional API keys, ignorable)
- `agent-browser` Node package installed with its own Chrome 148 at `/Users/cali/.agent-browser/browsers/chrome-148.0.7778.97`
- `browser_navigate` tool confirmed working (passes `goto example.com → return title` test)
- Terminal-based web fetching also works (Seek used curl on Wikipedia successfully)
- OpenRouter API key configured and authenticated
- Hermes session-reset mode set to "never auto-reset" (correct for single-user CLI workflow)
- Max iterations capped at 20 (was offered 90 default; lowered for cost control + faster failure on stuck loops)
- Context compression threshold: 0.85
- 3 sessions on file in state.db

## What we learned the hard way (v2's wrong assumptions, corrected)

1. **Hermes's `browser_navigate` uses agent-browser (Node), NOT the Python `playwright` package.** v2 spent hours installing Python Playwright into Hermes's venv. That was solving the wrong problem. The real path is Node-side `agent-browser`, which needs its own `npx agent-browser install` step that the setup wizard apparently does not run automatically. Playwright Python package in Hermes's venv is orphan junk — can be uninstalled to recover ~100MB.

2. **The `uv` venv layout (no pip, weird-looking bin contents) is normal, not broken.** Yesterday's diagnostic detour through "the venv looks weird" was a false lead. `uv venv` deliberately omits pip and ships cross-platform activate scripts (including `.bat` files on macOS). It's fine.

3. **`hermes doctor` can lie green.** Doctor reported `✓ Playwright Chromium` and `✓ browser` while `browser_navigate` was actually failing because Chrome-for-agent-browser wasn't installed. Doctor checks components individually, not end-to-end tool calls. Green checks are necessary but not sufficient — always also test the actual tool.

4. **Swallowed errors hide the diagnosis.** Hermes UI showed `[error]` for `browser_navigate` failures with no detail. The fix was to *explicitly ask Seek to use the browser tool* — at which point she returned the real error ("Chrome not installed"). Lesson: when something silently fails, command Seek to use it by name, get her error message in her own words, and you'll get the actual cause.

5. **The cali/seek user split caused friction we didn't fully untangle.** v2's design (Seek runs as non-admin `seek`) is good security hygiene but fights Hermes's installer, which assumes sudo. After failing on `seek`, we installed on `cali` as a workaround. Working install on cali is the path of least resistance for now. Migrating Seek back to `seek` is a future side quest, not a today-problem.

## Where Seek lives now (the corrected architecture)

| Thing | Path | Owner |
|---|---|---|
| Hermes binary | `/Users/cali/.local/bin/hermes` | cali |
| Hermes project dir | `/Users/cali/.hermes/hermes-agent/` | cali |
| Hermes venv (uv-managed) | `/Users/cali/.hermes/hermes-agent/venv/` | cali |
| Skills dir | `/Users/cali/.hermes/skills/` | cali |
| Config | `/Users/cali/.hermes/config.yaml` | cali |
| Secrets | `/Users/cali/.hermes/.env` | cali |
| State / sessions DB | `/Users/cali/.hermes/state.db` | cali |
| agent-browser Chrome | `/Users/cali/.agent-browser/browsers/chrome-148.0.7778.97` | cali |
| Cached Playwright Chromium (orphan, can delete) | `~/Library/Caches/ms-playwright/` | cali |
| Project workspace (Doom, scratch code) | `/Users/seek/code/construct/` | seek |
| Broken Friday Hermes install (delete me) | `/Users/seek/.hermes.broken-friday/` | seek |
| Stale Friday `hermes` binary (delete me) | `/Users/seek/.local/bin/hermes` | seek |

## Cleanup tasks (small, do when convenient)

```bash
# As cali — remove orphan Playwright Python package
uv pip uninstall --python ~/.hermes/hermes-agent/venv/bin/python playwright greenlet pyee

# As seek — remove broken Friday install
rm -rf ~/.hermes.broken-friday
rm ~/.local/bin/hermes

# Optionally as cali — remove orphan Playwright Chromium cache (~150MB)
# Only do this if construct/ venv doesn't need it either; check first
ls ~/code/construct/.venv/lib/python*/site-packages/playwright 2>/dev/null
# If that exists and Playwright works for `construct` purposes, leave the cache alone.
```

## Open architectural questions (for future, not now)

1. **Migrate Seek to the `seek` user, or stay on `cali`?** I designed for the split; the install drift undid it. Three real options when I revisit:
   - **A: Stay on cali.** Simpler, fewer moving parts. Sandboxed mini = lower threat. Accept that "I have an admin user running an agent" is the architecture.
   - **B: Variation B from Claude's earlier note** — temporarily promote `seek` to admin, fresh install as `seek`, demote `seek` back to standard once installed. Cleanest preservation of original design.
   - **C: Rename `cali` to `seek` (with home-dir manual rename) and create a new admin.** Weird but works.
   Not deciding tonight. Worth thinking about when I have a quiet weekend slot.

2. **Hermes is 0 commits behind as of fresh install May 10**, but Hermes moves fast (v2 was 214 commits behind after a day). I should run `hermes update` periodically. Don't run it impulsively mid-session.

3. **The OpenRouter key drama is unresolved as a process question.** I typo'd it once during setup, re-entered during a wizard re-run, and got lucky on the second pass. Need to store it somewhere I can find it (password manager) so next-time-me doesn't go through the same dance.

## Things to know about Hermes that v2 didn't quite have right

- **Skills system:** `hermes skills list` initializes the Skills Hub (it's currently uninitialized per doctor — "Skills Hub directory not initialized"). Most relevant for Job #1: install `duckduckgo-search` for free, no-API-key search. Also: a `GITHUB_TOKEN` in `.env` lifts a 60-req/hr GitHub rate limit, useful for skills browsing.
- **Tool warnings in `hermes doctor`:** the many ⚠ tool entries (`discord`, `homeassistant`, `image_gen`, `rl`, `web`, `spotify`, etc.) are optional integrations needing API keys. Ignore unless I want those specific services. The `web` tool wants paid search APIs (Exa, Tavily, Firecrawl); `duckduckgo-search` is the free alternative and the right answer.
- **`hermes doctor` will always show a "1 issue" nag** about missing API keys for optional tools. Functionally a no-op. Genuine issues show as ✗ (red), not ⚠ (yellow).
- **Settings I picked during setup:** local terminal backend, sudo enabled (cali has it), tool_progress: verbose, max iterations: 20, compression: 0.85, session reset: never auto-reset, platforms: none.

## What I'm doing next session

Pick one of:

1. **Install `duckduckgo-search` skill.** Architecturally correct move — gives Seek a real "search the web" tool instead of curl-and-pray. `hermes skills list` first to initialize the Hub, then install the skill, then test.
2. **Wire up my first hand-written skill spec into `~/.hermes/skills/`.** The actual project work. I have specs drafted but haven't picked which one is first. Should probably do step 1 first so the skill can lean on a real search tool.
3. **Set a `GITHUB_TOKEN` in `~/.hermes/.env`** for the rate-limit nicety.
4. **Cleanup tasks** above.

## How to help me

1. Confirm next steps in priority order based on whatever I say I want to do
2. If I pick "wire up a hand-written skill," ask me to share the spec
3. If something looks wrong in this handoff given current Hermes Agent state, flag it — but verify with a search first, and don't redesign without asking
4. **Don't tell me to go to bed unless I bring it up first.** I have agency around my own sleep schedule. Pulling the cord on a debug session is expensive (context reload), and "take a break" advice is condescending when I'm specifically here to work.
5. **Don't pre-blame me for tooling bugs.** Hermes is v0.13.0 and rough around the edges. When something fails, the prior probability that it's a Hermes bug is non-trivial, not just "I must have done something wrong." Read the evidence first.
