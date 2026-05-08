# Hermes on NuNet

Operator-flavoured skills for [Hermes Agent](https://github.com/NousResearch/hermes-agent), built and refined while running the agent on Telegram.

This repo is a working notebook. Each skill in here was written for a real workflow, then patched until the output was actually trustworthy. The iteration history is part of the artifact: the prompt files include the patches that got them to v-current, and dedicated ITERATIONS docs walk through what failed in earlier rounds and why.

If you're new to Hermes, run through the [Hermes Agent setup](https://github.com/NousResearch/hermes-agent) first. These skills assume you have a working install and an active Telegram gateway.

## Skills in this repo

| Skill | What it does | Native tools used |
|-------|--------------|-------------------|
| [website-analysis](skills/website-analysis.md) (v6) | Deep website audit (SEO, tech stack, brand voice, UX, performance) — four-pass dynamic fingerprinting handles consent banners, scroll-gated trackers, and SPAs | `terminal`, `web_extract`, `browser_navigate`, `browser_snapshot`, `browser_console`, `vision_analyze`, `web_search` |

More coming. Each new skill ships with its own iteration log so you can see what went wrong on the way to working.

## How to use a skill

Two ways. Pick whichever fits your workflow.

### Option 1 — Paste-and-create (the lazy way)

1. Open the skill file in this repo, e.g. [`skills/website-analysis.md`](skills/website-analysis.md).
2. Copy the full contents.
3. In your Telegram chat with Hermes, send: "Create a new skill called `website-analysis` and save it to `~/.hermes/skills/website-analysis.md` with this content:" then paste.
4. Hermes will create the file. Trigger it next time with whatever the skill's trigger phrase is (each skill documents its own).

### Option 2 — Drop-in (the fast way)

If you have shell access to the host running Hermes:

```bash
mkdir -p ~/.hermes/skills
curl -sL https://raw.githubusercontent.com/jennifer509/hermes-on-nunet/main/skills/website-analysis.md \
  -o ~/.hermes/skills/website-analysis.md
```

Hermes picks up new skill files on the next message. No restart needed.

### Option 3 — Update an existing skill via Telegram

If you already have a previous version of a skill in `~/.hermes/skills/` and just want the latest version, message your Hermes bot:

> Update my website-analysis skill to the latest version. Run this in your terminal tool: `curl -sL https://raw.githubusercontent.com/jennifer509/hermes-on-nunet/main/skills/website-analysis.md -o ~/.hermes/skills/website-analysis.md`. Then confirm the version number from the title of the new file.

The agent runs the curl via its terminal tool, the file gets overwritten, the new version is live on the next message. No shell access needed on your end.

## Why this repo exists

Most public Hermes content is developer-flavoured: install tutorials, hardware benchmarks, framework migration threads. There's a separate audience of operators (marketers, founders, ops people) running an agent through their actual workday. The skills here are for that second group. Operator voice, not developer voice. Real screenshots, real audits, real workflows.

Connected to a broader content series called "Agent Diary," published on [@jenbourke3](https://x.com/jenbourke3). Each post in the series is paired with a skill or a workflow you can copy.

## What I learned (the bit you actually want)

Across multiple iterations of building these skills, three things keep showing up:

1. **An agent will fabricate plausible values when its tools fail, unless you forbid it explicitly.** The "no fabrication" rule in every skill here is load-bearing.
2. **An agent will skip technical work in favour of qualitative reads, unless you require specific commands.** The Required terminal commands blocks aren't optional, they're the spine of the skill.
3. **"Fetch failed" and "not present" are different findings.** Until you teach the distinction, an agent treats them the same way and the audit becomes useless.

Each skill's ITERATIONS doc shows how those rules emerged from real failures, not from theory.

## Built on

These skills run on [Hermes Agent](https://github.com/NousResearch/hermes-agent) (Nous Research). For an alpha agent platform that does this kind of operator-friendly setup at the platform level, see [community.nunet.io/agents](https://community.nunet.io/agents), running on NuNet's distributed compute network. Currently in alpha, sign-ups open.

## License

MIT. Take, fork, modify, ship.

## Contact

[@jenbourke3](https://x.com/jenbourke3) on X for questions, additions, or weird edge cases your skill caught that mine didn't.
