# website-analysis: iteration log

The website-analysis skill went through six rounds before the output was actually trustworthy. Each round caught a specific failure mode. This log walks through what went wrong and which patch fixed it, so anyone forking the skill can see the reasoning instead of just the final prompt.

## Round 1 — Pretty but shallow

**The prompt:** rough description of the audit (SEO surface, tech stack, brand voice, UX, strengths, weaknesses) with the native tools listed but no required commands.

![Run 1 top: vague answers, no real numbers](assets/screenshots/run-1-vague-top.png)

![Run 1 bottom: "Fast TTFB" / "Minimal" placeholders, strengths/weaknesses](assets/screenshots/run-1-vague-bottom.png)

**What came back:**
- "Performance: Fast TTFB with optimized static assets."
- "Trust Pages: Minimal (Integrated #about section)."
- "Tech Stack: Next.js (Confirmed via `/_next/static/` chunks)."
- 3 H2s out of 8 requested
- Word count "~1,403"

The qualitative sections (themes, tone, strengths, weaknesses, the sharper observation) landed well. The technical sections were vague adjectives, no numbers, no proof. The agent had read the rendered page and made inferences instead of running the curl commands and interrogating the headers.

## Round 2 — Caught it skipping the technical work

**The patch:** added a Required terminal commands block with explicit curl invocations, an Output rule requiring every field to report a value or "fetch failed" (no silent omissions), and a Specificity rule mandating exact numbers instead of evaluative phrases.

![Run 2 top: fabricated 404s on trust pages, "OG Card: Present" without proof](assets/screenshots/run-2-fabricated-top.png)

![Run 2 bottom: same audit, UX + performance + sharper thing](assets/screenshots/run-2-fabricated-bottom.png)

**What came back:**
- All 8 H2s
- Exact word counts (1403 total, 296 above-fold)
- Real social proof block surfaced (testimonial + two case studies)
- "OG Card: Present (Matches Meta Description)"
- "Trust Pages: /privacy: 404, /privacy-policy: 404, /terms: 404, /tos: 404, /legal: 404, /contact: 404, /about: 404"
- Several fields marked "fetch failed" honestly

So the qualitative side jumped in quality. But something was off about the trust pages and the OG card. They came back as concrete findings, not failures.

## Round 3 — Caught it fabricating

**The diagnostic:** asked the agent to show literal command output for every command, with stdout/stderr/exit code. The result:

![Diagnostic top: curl: command not found across every call](assets/screenshots/diagnostic-top.png)

![Diagnostic bottom: continuation + Chrome auto-launch failures](assets/screenshots/diagnostic-bottom.png)

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

![Run 3 top: real Vercel + JSON-LD schema, several "fetch failed" labels in the wrong place](assets/screenshots/run-3-mixed-top.png)

![Run 3 bottom: colour palette + UX failed, performance fields still "fetch failed"](assets/screenshots/run-3-mixed-bottom.png)

- Server: **Vercel** (real)
- CDN: Vercel Edge Network
- Colour palette: `#6320EE, #FFFFFF, #F9F9FB, #000000` (real hex from `vision_analyze`)
- Security headers: HSTS Y, CSP not present, X-Frame-Options not present
- But: OG Card came back as `fetch failed: (No specific OG tags found in HTML)`. The fetch worked. The page just doesn't have OG tags. Wrong label.

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

![Run 4 top: real schema, sitemap content, exact trust-page status codes](assets/screenshots/run-4-clean-top.png)

![Run 4 bottom: real Vercel CDN, SSL issuer, page weight, response time, mobile reflow analysis](assets/screenshots/run-4-clean-bottom.png)

- JSON-LD schema actually extracted: Organization type, founder name, logo URL
- Robots.txt content quoted directly
- SSL issuer: Let's Encrypt R13/R12
- Page size: 101.5 KB. Response time: 151.1 ms. Script count: 8. All real.
- Mobile reflow caught a System Monitor graphic with text too small to read at 390×844, and thin Process Steps columns

This is the version shipped as v4 in [skills/website-analysis.md](skills/website-analysis.md).

## Round 5 — Caught it under-reporting trackers

**The diagnostic:** ran the v4 skill against a site I knew was carrying a Meta Pixel and Hotjar. The audit reported "not present" for both. Pulled DevTools manually and the pixels were obviously there — they were getting injected by Google Tag Manager after the initial HTML response. v4's curl-based tech-stack check was reading the GTM container script in the static HTML but never executing it, so the downstream pixels (and everything else GTM was loading) stayed invisible to the audit.

**The patch:** added a Required dynamic-fingerprinting block. After `browser_navigate`, the agent runs a JavaScript payload via `browser_console` that queries the runtime `window` object for known tracking signatures (`ga`, `gtag`, `dataLayer`, `fbq`, `ttq`, `snaptr`, `hotjar`, `clarity`) and scans the live DOM for script tags whose `src` matches `pixel|analytics|tag`. The check executes in the page context, so anything Tag Manager injected after load is now visible.

The Tech stack signals section was rewritten to require BOTH layers — static (curl) and dynamic (browser_console). The audit reports the static finds first, then the runtime signals.

**What came back on the rerun:**

- `fbq: Y` (was "not present" in v4)
- `hotjar: Y` (was "not present" in v4)
- `dataLayer: Y` (already caught by v4 — GTM container was in the static HTML)
- `injected_scripts: ["https://connect.facebook.net/en_US/fbevents.js", "https://static.hotjar.com/c/hotjar-..."]`

The same site went from "looks like they're not running tracking" to a full picture of the conversion-tracking stack.

**What it taught:**

A fourth rule, on top of the three from v4:

4. **An agent's audit reflects its execution context, not the website.** If your skill only inspects what curl returns, you're auditing the server response — not the user experience. Anything injected after page load (analytics, pixels, A/B variants, cookie banners that swap content, lazy-rendered components) is invisible. Any "tech stack" claim needs a runtime DOM check, not just a static HTML read. This is the same root pattern as the curl-not-installed fabrication from Round 3 — the agent reports the absence as truth instead of admitting the limit.

This patch shipped as v5 in [skills/website-analysis.md](skills/website-analysis.md).

## Round 6 — Caught it stopping at the front door

**The diagnostic:** posted v5 publicly. A LinkedIn commenter flagged the next case I hadn't thought through: cookie consent banners. v5's dynamic fingerprint runs once, immediately after page load. If a site holds its trackers behind a consent banner, the banner is still up when the fingerprint fires and none of the gated trackers have loaded yet. The audit reads "tracking: minimal" — which is technically true at audit-time but useless as a real read of the site.

Same problem on the other side: trackers that only fire on scroll, on form focus, or after some user signal. The audit was a snapshot of "what happens if you arrive and immediately leave." That's not the audit anyone actually wants.

Underneath both of those: SPA-rendered content. If the page is a Next.js / Nuxt / Remix app, the static HTML is mostly an empty shell. Curl reads "above-fold word count: 12" because nothing has rendered yet. The audit was treating the empty shell as the truth and reporting the site as content-thin.

Three different surfaces of the same root failure: the audit was a single moment in page life, treated as the whole picture.

**The patch:** turned the dynamic fingerprint into a four-pass sequence.

1. Initial fingerprint — same as v5, runs immediately after `browser_navigate`.
2. Consent dismissal — agent scans the visible DOM for buttons matching a regex of common consent-banner phrasings (`accept`, `allow all`, `i agree`, `got it`, etc.), clicks the first match, then re-fingerprints. If no candidate is found, the pass reports `not present` and moves on.
3. Scroll trigger — agent scrolls to bottom, re-fingerprints. Catches lazy-loaded trackers and below-fold dynamic content.
4. Synthesise — compare the three fingerprints, report each tracker as `Y (initial) / Y (after consent) / Y (after scroll) / N`, with a `triggered_by` note for each.

Added an SPA-detection field to the fingerprint: looks for `window.__NEXT_DATA__`, `window.__NUXT__`, `window.__REMIX_CONTEXT__`, `window.__sveltekit_data`, plus loose `window.React` / `window.Vue` checks. Plus a `body_text_len` delta — if the runtime DOM has 2x+ the text the static HTML had, the site is JS-rendered and any text-based extraction MUST come from the runtime, not curl.

**What came back on the rerun:**

- The original test site now showed `fbq: Y (after consent)` and `hotjar: Y (after consent)` — both gated behind a OneTrust banner that the click handler dismissed
- A separate Next.js site that had been auditing as "content-thin" (~80-word above-fold count) jumped to 600+ words once the SPA hydrated. The H2s and CTA copy that v5 missed were now in the audit.
- A third site that runs a TikTok pixel only after scroll triggered properly: `ttq: Y (after scroll)`

**What it taught:**

A fifth rule, on top of the four from v5:

5. **A page is not a snapshot.** It's a sequence of states: pre-consent / post-consent / post-scroll / post-interaction / hydrated / partially-hydrated. The audit must walk the sequence, not freeze the first frame. Picking one state and reporting it as "the site" is the same root error as Round 3 (treating curl-failure as fact) and Round 5 (treating server-response as the runtime DOM): conflating what the agent CAN see with what's there.

This patch shipped as v6 in [skills/website-analysis.md](skills/website-analysis.md).

## What the iterations taught

Five rules that turned out to be load-bearing for any agent skill that uses external tools:

1. **An agent will fabricate plausible values when its tools fail, unless you forbid it explicitly.** The strict no-fabrication rule isn't optional. Without it, half your "findings" are dressed-up guesses.

2. **An agent will skip technical work in favour of qualitative reads, unless you require specific commands.** Listing tools as "available" means they might get used. Listing exact commands with required output means they will.

3. **"Fetch failed" and "not present" are different findings, and the agent needs to be taught the difference.** Treating them the same means you can't tell which gaps are real and which are environmental.

4. **An agent's audit reflects its execution context, not the website.** Static fetchers see server responses. Dynamic fetchers see runtime DOM. Anything injected post-load is invisible to the first kind. Pick the layer that matches the claim you're making.

5. **A page is not a snapshot, it's a sequence of states.** Pre-consent / post-consent / post-scroll / hydrated / partially-hydrated. Picking one state and reporting it as "the site" is the same root error as treating curl-failure as fact. Walk the sequence, don't freeze the first frame.

These five rules are now baked into every skill in this repo.

## Known minor gaps in v6

- **Fonts detection** sometimes returns "not present" when the site uses fonts. The grep pattern misses font declarations inside CSS modules.
- **Trust pages check** doesn't catch in-page anchor sections (e.g. `#contact` on a single-page site). The 404s are real but the audit reads as more compliance-thin than reality.
- **Consent banner clicker** uses a regex-on-visible-button-text heuristic. Sites with shadow-DOM consent UIs (some OneTrust / Cookiebot configurations) won't be caught. Falls back to honest reporting (`consent_pass: not present`) rather than failing.
- **Dynamic fingerprinting signature list** is finite. New trackers or rebrands won't be caught until the signature list is updated. Floor, not ceiling.
- **Interaction beyond scroll** — pixels triggered by hover, form-focus, video-play, or button-click (other than the consent banner) are still invisible. v7 territory if it next bites.

If you fork and fix any of these, open a PR.
