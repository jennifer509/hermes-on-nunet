# Website Analysis Skill (v5)

Deep website audits for [Hermes Agent](https://github.com/NousResearch/hermes-agent) using only native tools. No third-party API keys. No Ahrefs / SEMrush / Wappalyzer / BuiltWith / SimilarWeb.

## Trigger

Any message containing a URL plus one of: `audit`, `analyze this site`, `deep dive on`, `read this site`, or `/audit <url>`.

## Native tools used

- `web_extract` — page HTML and main content
- `terminal` — runs the Required terminal commands block below
- `browser_navigate` + `browser_snapshot` + `vision_analyze` — visual read at desktop AND mobile viewports
- `browser_console` — runs the Required dynamic-fingerprinting block (catches GTM-injected pixels static curl misses)
- `web_search` — cross-referencing brand mentions and recent press

## Required terminal commands

Substitute `<url>` with the actual URL before running. Run all commands. Include exact output in the report.

```bash
curl -sIL <url>                                                          # response headers + redirects
curl -sL <url>/robots.txt | head -30                                     # robots rules
curl -sL <url>/sitemap.xml | grep -c '<loc>'                             # sitemap URL count
curl -sL -o /dev/null -w 'size:%{size_download} time:%{time_total} code:%{http_code}\n' <url>
curl -vIL <url> 2>&1 | grep -iE 'subject:|issuer:'                       # SSL cert
for p in /privacy /privacy-policy /terms /tos /legal /contact /about; do
  curl -sIL -o /dev/null -w "%{http_code} ${p}\n" <url>${p}
done
curl -sL <url> | grep -oE '<script[^>]*src="[^"]*"' | wc -l              # script count
curl -sL <url> | grep -oE 'application/ld\+json[^<]*' | head -5          # schema types
```

## Required dynamic fingerprinting

After `browser_navigate`, run this JavaScript via `browser_console` to catch tracking pixels and analytics that Tag Managers or async loaders inject after the initial HTML response. Static curl misses these — they only exist in the runtime DOM.

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
  injected_scripts: Array.from(document.querySelectorAll('script'))
    .filter(s => /pixel|analytics|tag/i.test(s.src))
    .map(s => s.src),
};
JSON.stringify(signals, null, 2);
```

Report each signal as Y/N. List `injected_scripts` URLs verbatim. Three-state output rule applies — if `browser_console` itself fails, every signal is `fetch failed: <error>`, not Y/N defaults.

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

**Tracking + analytics:** report findings from BOTH layers — static (HTML scripts visible to curl) AND dynamic (the `browser_console` fingerprint above). Static-only catches GTM containers but misses the pixels GTM loads. Dynamic catches the runtime truth. Most modern sites need the dynamic check or the audit under-reports the stack. List the static finds first, then the runtime signals (Y/N per known signature, plus the `injected_scripts` URL list).

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

See [ITERATIONS-website-analysis.md](../ITERATIONS-website-analysis.md) for the full five-run story, including what each version got wrong and which patch fixed it.

Quick summary:

- **v1** — initial draft. Pretty layout, vague answers, no real numbers.
- **v2** — added Required terminal commands block, output rule (no silent omissions), specificity rule. Caught the agent skipping curl entirely.
- **v3** — added strict "no fabrication" rule. Caught the agent fabricating trust-page 404s when curl wasn't installed.
- **v4** — distinguished `fetch failed` vs `not present`, added `-L` flag to all curl commands so they follow redirects, made mobile reflow a hard required step.
- **v5** — added a Required dynamic-fingerprinting block via `browser_console`. Caught the agent under-reporting analytics + tracking pixels because curl never executes the JavaScript that injects them.

## Known minor gaps in v5

- **Fonts detection** sometimes returns "not present" when the site does use fonts. The current grep pattern misses font declarations inside CSS modules. Open to a v6 patch.
- **Trust pages check** doesn't check for in-page anchor sections (e.g. `#contact` on a single-page landing site). Reports 404s correctly, but a single-page site with embedded contact info will look more compliance-thin than it actually is.
- **Dynamic fingerprinting signature list** is finite (8 known trackers + an `injected_scripts` regex). New tools or rebrands won't be caught until the signature list is updated. Treat it as a known floor, not a ceiling.

If you patch any of these and it works, send me the diff or open a PR.
