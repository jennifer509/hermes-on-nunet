# Hermes on NuNet

Operator-flavoured skills for [Hermes Agent](https://github.com/NousResearch/hermes-agent), built and refined while running the agent on Telegram.

This repo is a working notebook. Each skill in here was written for a real workflow, then patched until the output was actually trustworthy. The iteration history is part of the artifact: the prompt files include the patches that got them to v-current, and dedicated ITERATIONS docs walk through what failed in earlier rounds and why.

These skills work with any Hermes agent that's running with an active Telegram gateway. The fastest way to get one is via [agents.nunet.network](https://agents.nunet.network) — NuNet's alpha — which handles the install, gateway, and model setup in about 60 seconds, no terminal needed. Self-hosting works too; both paths are documented below.

## Skills in this repo

| Skill | What it does | Native tools used |
|-------|--------------|-------------------|
| [website-analysis](skills/website-analysis.md) (v7) | Deep website audit (SEO, tech stack, brand voice, UX, performance) — four-pass dynamic fingerprinting handles consent banners, scroll-gated trackers, and SPAs. Guardrails: identifiable UA, robots.txt pre-flight, rate limits, bail-out on 429/403 | `terminal`, `web_extract`, `browser_navigate`, `browser_snapshot`, `browser_console`, `vision_analyze`, `web_search` |

More coming. Each new skill ships with its own iteration log so you can see what went wrong on the way to working.

## How to use a skill

Skills install as text files into your Hermes agent's `~/.hermes/skills/` directory. The agent picks up new skill files on the next message — no restart needed.

### If you deployed via agents.nunet.network (the recommended path)

You manage the agent through Telegram. No shell access on the host, and you don't need any.

To install a skill:

1. Open the skill file in this repo, e.g. [`skills/website-analysis.md`](skills/website-analysis.md).
2. Copy the full contents.
3. In your Telegram chat with the agent, send: "Create a new skill called `website-analysis` and save it to `~/.hermes/skills/website-analysis.md` with this content:" then paste.
4. The agent creates the file. Trigger it next time with the skill's documented trigger phrase.

To update a skill that's already installed:

> Send to your Hermes bot: "Update my website-analysis skill to the latest version. Run this in your terminal tool: `curl -sL https://raw.githubusercontent.com/jennifer509/hermes-on-nunet/main/skills/website-analysis.md -o ~/.hermes/skills/website-analysis.md`. Then confirm the version number from the title of the new file."

The agent runs the curl via its `terminal` tool, the file gets overwritten, the new version is live on the next message. No shell access needed on your end.

### If you're self-hosting Hermes

If you installed Hermes yourself and have shell access to the host:

```bash
mkdir -p ~/.hermes/skills
curl -sL https://raw.githubusercontent.com/jennifer509/hermes-on-nunet/main/skills/website-analysis.md \
  -o ~/.hermes/skills/website-analysis.md
```

Same install pattern, just direct from the shell.

## Why this repo exists

Most public Hermes content is developer-flavoured: install tutorials, hardware benchmarks, framework migration threads. There's a separate audience of operators (marketers, founders, ops people) running an agent through their actual workday. The skills here are for that second group. Operator voice, not developer voice. Real screenshots, real audits, real workflows.

Connected to a broader content series called "Agent Diary," published on [@jenbourke3](https://x.com/jenbourke3). Each post in the series is paired with a skill or a workflow you can copy.

## What I learned (the bit you actually want)

Across multiple iterations of building these skills, three things keep showing up:

1. **An agent will fabricate plausible values when its tools fail, unless you forbid it explicitly.** The "no fabrication" rule in every skill here is load-bearing.
2. **An agent will skip technical work in favour of qualitative reads, unless you require specific commands.** The Required terminal commands blocks aren't optional, they're the spine of the skill.
3. **"Fetch failed" and "not present" are different findings.** Until you teach the distinction, an agent treats them the same way and the audit becomes useless.

Each skill's ITERATIONS doc shows how those rules emerged from real failures, not from theory.

## Things to know before running these

Each skill in this repo makes HTTP requests against third-party sites. Each audit fires roughly 12-20 requests. Each comparison or monitor run multiplies that. Without thinking about it, that traffic looks identical to a scraper — and the host running the skill can land on IP-reputation blocklists (Spamhaus, Cloudflare, Sucuri), which affects every workload on the host, not just the audit.

I caught this myself after v6 shipped and the repo started getting picked up. Before more people installed the skill, I patched it. v7+ has six guardrails layered in.

### What's baked in (so you don't have to remember)

- **Identifiable user-agent on every request.** `User-Agent: HermesAudit/v<n> (+https://github.com/jennifer509/hermes-on-nunet)`. Site operators can see what's hitting them, allow or block it cleanly, and reach me if needed. The skills never spoof a browser UA.
- **Robots.txt pre-flight.** Before the first audit request, the agent fetches `robots.txt` and checks whether the path is `Disallow`ed. If so, it stops and asks for explicit confirmation. No silent override.
- **Sleeps between request groups.** ~6-10 seconds of pacing per audit on top of network time. Imperceptible to a real user. Significant to a WAF watching for back-to-back-to-back requests.
- **Hard caps per host.** 1 audit per URL per 24h. 10 unique URLs per host per 24h. The skills track this themselves — re-running a recent audit IS the pattern that gets blocklisted.
- **Bail-out on 429, 403, 503.** No retry-storms. No swapping UAs to evade a 403. If a site bans the audit, that's the audit result.
- **Scope rule in the prompt itself.** Each skill's intro lists what to audit and what never to audit, and explicitly names the IP-reputation risk. Lives where the agent reads it, not in a README the user might skip.

### What only you can do

- **Pick the right targets.** Audit sites you own, sites you have written permission for, or public-facing marketing pages of public companies. Don't audit personal blogs, gov/healthcare/financial sites you have no engagement with, or anything you'd be uncomfortable defending if challenged. The skill can't know whether you have permission — it can only ask.
- **Don't override the robots.txt confirmation unless you actually have written permission.** The pre-flight asks because the site asked. Saying yes anyway is on you, not the skill.
- **Watch the per-host cap when you stack skills.** v7 enforces caps within each skill. If you're running `website-analysis` AND `competitor-compare` AND `content-monitor` against the same domain in the same window, the per-skill caps don't aggregate. v8 patch on the way (a shared `~/hermes-audit-ledger.json`).

### Where this came from

v7 isn't a regression of v6 — it's the same skill plus six guardrails that protect both you and everyone else running on your host. Round 7 in [`ITERATIONS-website-analysis.md`](ITERATIONS-website-analysis.md) walks through the reasoning. Short version: I built a tool that worked, but didn't think about what its requests looked like from the target's perspective. v7 fixes that, before more people pick up v6.

## Built on

These skills run on [Hermes Agent](https://github.com/NousResearch/hermes-agent) (Nous Research). For an alpha agent platform that does this kind of operator-friendly setup at the platform level, see [agents.nunet.network](https://agents.nunet.network), running on NuNet's distributed compute network. Currently in alpha, sign-ups open.

## License

MIT. Take, fork, modify, ship.

## Contact

[@jenbourke3](https://x.com/jenbourke3) on X for questions, additions, or weird edge cases your skill caught that mine didn't.
