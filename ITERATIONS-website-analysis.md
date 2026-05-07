# website-analysis: iteration log

The website-analysis skill went through four rounds before the output was actually trustworthy. Each round caught a specific failure mode. This log walks through what went wrong and which patch fixed it, so anyone forking the skill can see the reasoning instead of just the final prompt.

## Round 1 — Pretty but shallow

**The prompt:** rough description of the audit (SEO surface, tech stack, brand voice, UX, strengths, weaknesses) with the native tools listed but no required commands.

**What came back:**
- "Performance: Fast TTFB with optimized static assets."
- "Trust Pages: Minimal (Integrated #about section)."
- "Tech Stack: Next.js (Confirmed via `/_next/static/` chunks)."
- 3 H2s out of 8 requested
- Word count "~1,403"

The qualitative sections (themes, tone, strengths, weaknesses, the sharper observation) landed well. The technical sections were vague adjectives, no numbers, no proof. The agent had read the rendered page and made inferences instead of running the curl commands and interrogating the headers.

## Round 2 — Caught it skipping the technical work

**The patch:** added a Required terminal commands block with explicit curl invocations, an Output rule requiring every field to report a value or "fetch failed" (no silent omissions), and a Specificity rule mandating exact numbers instead of evaluative phrases.

**What came back:**
- All 8 H2s
- Exact word counts (1403 total, 296 above-fold)
- Real social proof block surfaced (testimonial + two case studies)
- "OG Card: Present (Matches Meta Description)"
- "Trust Pages: /privacy: 404, /privacy-policy: 404, /terms: 404, /tos: 404, /legal: 404, /contact: 404, /about: 404"
- Several fields marked "fetch failed" honestly

So the qualitative side jumped in quality. But something was off about the trust pages and the OG card — they came back as concrete findings, not failures.

## Round 3 — Caught it fabricating

**The diagnostic:** asked the agent to show literal command output for every command, with stdout/stderr/exit code. The result:

```
1. Command: curl -sIL https://zeroarc.ai
   - Stdout: /usr/bin/bash: line 3: curl: command not found
   - Exit Code: 127
```

curl wasn't installed on the host. Every curl call had failed. But the previous round's report had returned 404s for trust pages and "Present" for OG tags.

The agent had filled blanks with plausible values when the tools failed. The Output rule said "report a value or fetch failed" but didn't forbid fabrication explicitly. The agent picked plausible-looking values.

**The patches:**

```
PROBLEM 1: curl is not installed. Run via terminal:
   apt-get update && apt-get install -y curl ca-certificates

PROBLEM 2: Chrome is not installed. Run via terminal:
   agent-browser install

PROBLEM 3: Strict "no fabrication" Output rule:
   If a fetch fails or a tool errors, the field value must be exactly
   "fetch failed: <one-line error>". Never substitute a plausible value,
   default, inference, or guess. The literal failure message is the value.
```

**What came back this round:**
- Server: **Vercel** (real)
- CDN: Vercel Edge Network
- Colour palette: `#6320EE, #FFFFFF, #F9F9FB, #000000` (real hex from `vision_analyze`)
- Security headers: HSTS Y, CSP not present, X-Frame-Options not present
- But: OG Card came back as `fetch failed: (No specific OG tags found in HTML)` — which is the wrong label. The fetch worked. The page just doesn't have OG tags.

So one new problem: the strict no-fabrication rule was over-correcting. It conflated "fetch failed" with "fetched successfully but the field is genuinely absent."

## Round 4 — Distinguish failure from absence

**The patches:**

1. Three-state output rule:
   - Tool failed → `fetch failed: <error>`
   - Tool ran, field absent → `not present`
   - Tool ran, value found → the actual value

2. Added `-L` to all curl commands so they follow redirects (the previous round's robots.txt fetch was getting redirected to the www subdomain and failing silently).

3. Made mobile reflow a hard required step (the previous round had silently skipped the second `browser_navigate` at mobile viewport).

**What came back:**
- JSON-LD schema actually extracted: Organization type, founder name, logo URL
- Robots.txt content quoted directly
- SSL issuer: Let's Encrypt R13/R12
- Page size: 101.5 KB. Response time: 151.1 ms. Script count: 8. All real.
- Mobile reflow caught a System Monitor graphic with text too small to read at 390×844, and thin Process Steps columns

This is the version shipped as v4 in [skills/website-analysis.md](skills/website-analysis.md).

## What the iterations taught

Three rules that turned out to be load-bearing for any agent skill that uses external tools:

1. **An agent will fabricate plausible values when its tools fail, unless you forbid it explicitly.** The strict no-fabrication rule isn't optional. Without it, half your "findings" are dressed-up guesses.

2. **An agent will skip technical work in favour of qualitative reads, unless you require specific commands.** Listing tools as "available" means they might get used. Listing exact commands with required output means they will.

3. **"Fetch failed" and "not present" are different findings, and the agent needs to be taught the difference.** Treating them the same means you can't tell which gaps are real and which are environmental.

These three rules are now baked into every skill in this repo.

## Known minor gaps in v4

- **Fonts detection** sometimes returns "not present" when the site uses fonts. The grep pattern misses font declarations inside CSS modules.
- **Trust pages check** doesn't catch in-page anchor sections (e.g. `#contact` on a single-page site). The 404s are real but the audit reads as more compliance-thin than reality.

If you fork and fix either, open a PR.
