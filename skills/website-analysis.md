# Website Analysis Skill (v7)

Deep website audits for [Hermes Agent](https://github.com/NousResearch/hermes-agent) using only native tools. No third-party API keys. No Ahrefs / SEMrush / Wappalyzer / BuiltWith / SimilarWeb.

## Before you run this on something

Each audit fires ~12-20 HTTP requests. At scale or against the wrong sites, that pattern looks identical to a scraper — and it can get your host's IP blacklisted by reputation services (Spamhaus, Cloudflare, Sucuri), which affects every workload on the host, not just this skill.

Three things to know before running:

**Where this is fine to run:**
- Sites you own
- Sites you have explicit written permission to audit
- Public-facing marketing pages of public companies (homepages, product pages, pricing pages — the stuff their PR team wants you to look at)

**Where it isn't:**
- Authenticated pages, dashboards, paywalls — the skill bails on a 401/403 anyway, but don't try
- Sites whose `robots.txt` says no for the path you're auditing
- Personal blogs and individuals' sites without consent
- Government, healthcare, or financial-services sites you don't have an engagement with
- Anywhere you'd be uncomfortable defending the audit if challenged

**What v7 enforces for you so you don't have to remember:**
- 1 audit per URL per 24 hours (re-running is exactly the pattern that gets blocklisted)
- 10 unique URLs per host per 24 hours
- Hard stop on 429, 403, or 503 — no retry-storms, no swapping UAs to evade a ban
- Identifiable user-agent (`HermesAudit/v7 (+github.com/jennifer509/hermes-on-nunet)`) so site operators can see the tool and block it cleanly if they want
- Robots.txt check before any other request — if the path is Disallowed, agent stops and asks for explicit confirmation before continuing

The scarce resource is your IP reputation, not the audit output. If you don't know whether a site is fair game, don't audit it.

## Trigger

Any message containing a URL plus one of: `audit`, `analyze this site`, `deep dive on`, `read this site`, or `/audit <url>`.

**Pre-flight (mandatory).** Before issuing any other request, the agent does this in order:
1. Fetches `robots.txt` from the target host
2. Checks whether the target path is `Disallow`ed for `*` or for `User-agent: HermesAudit`
3. If disallowed, replies to the user with the relevant `robots.txt` line: "This site asks crawlers not to access this path. Proceed anyway? Only say yes if you have explicit permission from the site owner."
4. If the user doesn't say yes, the agent abandons the audit. No partial execution — partial audits with the safety check half-skipped are worse than no audit.

The pre-flight is what makes the rest of the guardrails meaningful. Without it, an identifiable user-agent just tells a hostile site exactly who's hammering them.

## Native tools used

- `web_extract` — page HTML and main content
- `terminal` — runs the Required terminal commands block below
- `browser_navigate` + `browser_snapshot` + `vision_analyze` — visual read at desktop AND mobile viewports
- `browser_console` — runs the Required dynamic-fingerprinting block (catches GTM-injected pixels, JS-rendered content, consent-gated trackers, scroll-gated trackers)
- `web_search` — cross-referencing brand mentions and recent press

## Required terminal commands

Substitute `<url>` with the actual URL before running. Run all commands sequentially with the 1-second sleeps between groups. Include exact output in the report.

**Identifiable user-agent on every curl call.** The agent identifies itself so site operators can identify the tool, allow or block it, and contact the maintainer if needed. Don't spoof a browser UA.

```bash
UA="HermesAudit/v7 (+https://github.com/jennifer509/hermes-on-nunet)"

# Group 1 — robots + sitemap (always run first)
curl -sIL -A "$UA" <url>/robots.txt | head -30                           # confirm pre-flight result
curl -sL -A "$UA" <url>/robots.txt | head -30                            # robots rules
curl -sL -A "$UA" <url>/sitemap.xml | grep -c '<loc>'                    # sitemap URL count
sleep 1

# Group 2 — main URL response signals
curl -sIL -A "$UA" <url>                                                  # response headers + redirects
curl -sL -A "$UA" -o /dev/null -w 'size:%{size_download} time:%{time_total} code:%{http_code}\n' <url>
curl -vIL -A "$UA" <url> 2>&1 | grep -iE 'subject:|issuer:'              # SSL cert
sleep 1

# Group 3 — body parses (single fetch, two greps)
HTML=$(curl -sL -A "$UA" <url>)
echo "$HTML" | grep -oE '<script[^>]*src="[^"]*"' | wc -l                 # script count
echo "$HTML" | grep -oE 'application/ld\+json[^<]*' | head -5             # schema types
sleep 1

# Group 4 — trust pages (one curl per path, 0.5s between, bail on 429/403)
for p in /privacy /privacy-policy /terms /tos /legal /contact /about; do
  CODE=$(curl -sIL -A "$UA" -o /dev/null -w "%{http_code}" <url>${p})
  echo "${CODE} ${p}"
  if [ "$CODE" = "429" ] || [ "$CODE" = "403" ]; then
    echo "fetch failed: rate limited or forbidden ($CODE) — bailing out"
    break
  fi
  sleep 0.5
done
```

**Bail-out rules (non-negotiable).** If any curl call returns:
- `429` — STOP all further requests against this host. Wait at least 60 seconds, try the same call ONCE more, and if that also returns 429 abandon the audit and report `fetch failed: rate limited (429)` for every remaining field.
- `403` — STOP all further requests. The site has explicitly forbidden access. Report `fetch failed: forbidden (403)` and explain to the user that this site does not permit automated access.
- `503` — Wait 30 seconds and retry once. If still 503, treat as `fetch failed: service unavailable (503)`.

Never retry-storm. Never substitute a different user-agent to evade a 403. If a site bans you, that's the audit result.

## Required dynamic fingerprinting

After `browser_navigate`, run a four-pass fingerprint to catch trackers, analytics, and content that surface at different stages of page life. Each pass runs the same `browser_console` payload; the differences are what's happened on the page before it runs.

### The fingerprint payload

Same JS in every pass. Run it via `browser_console`.

```javascript
const signals = {
  ga: typeof window.ga !== 'undefined',
  gtag: typeof window.gtag !== 'undefined',
  dataLayer: Array.isArray(window.dataLayer),
  fbq: typeof window.fbq !== 'undefined',
  ttq: typeof window.ttq !== 'undefined',
  snaptr: typeof window.snaptr !== 'undefined',
  hotjar: typeof window.hotjar !== 'undefined',
  clarity: typeof window.clarity !== 'undefined',
  spa_framework:
    typeof window.__NEXT_DATA__ !== 'undefined' ? 'next'
    : typeof window.__NUXT__ !== 'undefined' ? 'nuxt'
    : typeof window.__REMIX_CONTEXT__ !== 'undefined' ? 'remix'
    : typeof window.__sveltekit_data !== 'undefined' ? 'sveltekit'
    : typeof window.React !== 'undefined' ? 'react (no framework signal)'
    : typeof window.Vue !== 'undefined' ? 'vue (no framework signal)'
    : 'none',
  ready_state: document.readyState,
  body_text_len: document.body ? document.body.innerText.length : 0,
  injected_scripts: Array.from(document.querySelectorAll('script'))
    .filter(s => s.src && /pixel|analytics|tag|track|gtm|hotjar|clarity|fb|tiktok/i.test(s.src))
    .map(s => s.src),
};
JSON.stringify(signals, null, 2);
```

### The four passes

1. **Initial fingerprint** — run immediately after `browser_navigate`. Captures what's loaded on first paint.
2. **Post-consent fingerprint** — try to dismiss any cookie/consent banner, then re-fingerprint. Many sites gate trackers behind consent; this catches them.
3. **Post-scroll fingerprint** — scroll to bottom, wait briefly, re-fingerprint. Catches lazy-loaded trackers, scroll-triggered events, and below-fold dynamic content.
4. **Final delta** — compare the three fingerprints; report a unified Y/N per signal where ANY pass found it, plus a `triggered_by` note (`initial` / `consent` / `scroll`) for each newly-discovered signal.

### Pass 2 — consent dismissal

Run via `browser_console` after the initial fingerprint:

```javascript
const acceptPatterns = /^(accept|allow|got it|i agree|i accept|ok|continue|consent|enable|allow all|accept all|accept cookies|i understand)/i;
const candidates = Array.from(document.querySelectorAll('button, a, [role="button"], [type="submit"]'))
  .filter(el => el.offsetParent !== null)
  .filter(el => acceptPatterns.test((el.innerText || '').trim()));
const target = candidates[0];
if (target) {
  target.click();
  ({ clicked: true, label: target.innerText.trim().slice(0, 60) });
} else {
  ({ clicked: false, reason: 'no consent-banner candidate found' });
}
```

If `clicked: true`, wait ~1 second (re-issue `browser_navigate` to the same URL with a small delay, OR just run the next fingerprint — by the time `browser_console` returns, most consent-triggered scripts have fired). Re-run the fingerprint payload.

If `clicked: false`, skip to Pass 3.

### Pass 3 — scroll trigger

```javascript
window.scrollTo(0, document.body.scrollHeight);
({ scrolled_to: document.body.scrollHeight, current: window.scrollY });
```

Then re-run the fingerprint payload.

### Pass 4 — synthesise

The agent compares the three captured `signals` objects and reports:

- Each tracker as `Y (initial)` / `Y (after consent)` / `Y (after scroll)` / `N`
- The `spa_framework` value (one of: next, nuxt, remix, sveltekit, react, vue, none)
- `body_text_len` from initial vs final pass — large delta (>2x) signals JS-rendered content the static curl missed
- `injected_scripts` — full URL list, deduped across passes
- If `body_text_len` doubled or more after navigation, flag the site as "JS-rendered (SPA-like)" and note that any text-based extraction MUST come from `browser_console`/`vision_analyze`, not curl

### Three-state output rule still applies

If `browser_console` itself fails, every signal is `fetch failed: <error>`, not Y/N defaults. If the consent click fails (no candidate found), report `consent_pass: not present`, not `consent_pass: failed`.

### Browser pacing

Add a 2-second `sleep` between each browser pass (initial → consent → scroll). Don't fire the consent click immediately after the initial fingerprint; many consent banners track timing as a bot signal. The total audit budget is roughly: 3-4 navigations, 4 console runs, 6-second total wait time.

## Output rule (three states, no fabrication)

1. Tool failed to run (command not found, network error, browser couldn't launch) → `fetch failed: <one-line error>`
2. Tool ran successfully but the field was absent from the response (no OG tag in HTML, no canonical, no schema, no privacy page) → `not present`
3. Tool ran successfully and returned a value → the actual value, exact

Never fabricate. If unsure whether the tool ran, that's case 1, not case 2. The literal failure message is the value.

## Specificity rule

Replace evaluative phrases like "fast" / "minimal" / "high" with exact values. Page size in KB. Response time in ms. Header presence as Y/N. Trust pages as a list of HTTP status codes per path. Word counts as exact integers, not approximate.

## Audit sections, in this order

### 1. At a glance

Domain, one-sentence positioning extracted from hero copy, primary CTA, inferred target audience.

### 2. SEO surface

Title, meta description, OG card preview, canonical, schema.org `@types` from JSON-LD, H1 + first 8 H2s, total word count + above-fold word count, robots.txt highlights, sitemap URL count, trust page status codes per path.

### 3. Tech stack signals

Server header, X-Powered-By, generator meta, framework fingerprints (`/_next/`, `/wp-content/`, `cdn.shopify.com`, Webflow, Squarespace, etc.), fonts (grep HTML for `font-family:` declarations and `<link>` tags ending `.woff2` or `.woff`), CDN, SSL cert issuer.

**SPA / JS-rendered content:** report `spa_framework` from the dynamic fingerprint (next / nuxt / remix / sveltekit / react / vue / none) AND the `body_text_len` delta between static HTML and runtime DOM. If the runtime DOM has 2x+ the text content of the static HTML, flag the site as JS-rendered and note that all content extraction came from the runtime DOM, not curl.

**Tracking + analytics:** report findings from BOTH layers — static (HTML scripts visible to curl) AND dynamic (the four-pass `browser_console` fingerprint above). Static-only catches GTM containers but misses the pixels GTM loads. Dynamic catches the runtime truth. Most modern sites need the dynamic check or the audit under-reports the stack. List the static finds first, then the runtime signals (Y/N per known signature, with `triggered_by: initial | consent | scroll`, plus the deduped `injected_scripts` URL list).

### 4. Content + brand voice

Top 3 themes, tone (3 adjectives), reading level, jargon density, CTA style, social proof (testimonials, logo bars, "Trusted by", customer counts, badges; list what's there or report absent), biggest gap.

### 5. UX / design read

Required: two `browser_navigate` calls, one at desktop default viewport, one at mobile (390×844 / iPhone 14). Take a snapshot at each.

Report: layout type, visual hierarchy, 4-colour palette as hex values from `vision_analyze`, mobile reflow (what reflows, breaks, overlaps, becomes hidden), dated vs modern feel.

### 6. Performance signals

Page size in KB, response time in ms, script count, lazy-load Y/N, security headers (HSTS Y/N, CSP Y/N, X-Frame-Options Y/N).

### 7. Three strengths

Concrete, with the specific element / quote / number per finding.

### 8. Three weaknesses

Same format.

### 9. The sharper thing

One sharp observation. Whatever surprised you. Optional.

## Output format

One Telegram message, ~1500-2000 chars, sectioned with bold headers. Use plain `→` arrows or "to". Never LaTeX `$\rightarrow$`. If the report runs longer than 2000 chars, save the full version to `~/hermes-reports/{domain}-{YYYY-MM-DD}.md` and include the path in the reply.

## Iteration history

See [ITERATIONS-website-analysis.md](../ITERATIONS-website-analysis.md) for the full seven-run story, including what each version got wrong and which patch fixed it.

Quick summary:

- **v1** — initial draft. Pretty layout, vague answers, no real numbers.
- **v2** — added Required terminal commands block, output rule (no silent omissions), specificity rule. Caught the agent skipping curl entirely.
- **v3** — added strict "no fabrication" rule. Caught the agent fabricating trust-page 404s when curl wasn't installed.
- **v4** — distinguished `fetch failed` vs `not present`, added `-L` flag to all curl commands so they follow redirects, made mobile reflow a hard required step.
- **v5** — added a Required dynamic-fingerprinting block via `browser_console`. Caught the agent under-reporting analytics + tracking pixels because curl never executes the JavaScript that injects them.
- **v6** — extended the dynamic fingerprint to four passes (initial / post-consent / post-scroll / synthesise). Closes consent-gated and scroll-gated trackers, plus surfaces SPA-rendered content via a body_text_len delta check.
- **v7** — added responsible-use guardrails: identifiable user-agent, mandatory robots.txt pre-flight, sleep between request groups, hard caps per host, bail-out on 429/403/503. Reframes the audit as a tool that protects the host's IP reputation, not just produces a report.

## Known minor gaps in v7

- **Fonts detection** sometimes returns "not present" when the site does use fonts. The grep pattern misses font declarations inside CSS modules. Still open.
- **Trust pages check** doesn't check for in-page anchor sections (e.g. `#contact` on a single-page landing site). Reports 404s correctly, but a single-page site with embedded contact info will look more compliance-thin than it actually is.
- **Consent banner clicker** uses a heuristic regex on visible button text. Sites with shadow-DOM consent UIs (OneTrust, Cookiebot in some configurations) won't be caught. The fallback is honest reporting (`consent_pass: not present`), not failure to function.
- **Dynamic fingerprinting signature list** is finite. New trackers or rebrands won't be caught until the signature list is updated. Floor, not ceiling.
- **Interaction beyond scroll** — pixels triggered by hover, form-focus, video-play, or button-click (other than the consent banner) are still invisible. v8 territory if it next bites.
- **Cross-host audit volume** is enforced per-skill (10/24h here, etc.) but there's no shared rate-limit ledger across skills running on the same host. If a user runs `website-analysis` AND `competitor-compare` AND `content-monitor` against the same domain in the same window, the per-skill cap doesn't aggregate. v8+ patch could land a `~/hermes-audit-ledger.json` shared across skills.

If you patch any of these and it works, send me the diff or open a PR.
