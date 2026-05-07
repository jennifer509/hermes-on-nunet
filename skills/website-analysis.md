# Website Analysis Skill (v4)

Deep website audits for [Hermes Agent](https://github.com/NousResearch/hermes-agent) using only native tools. No third-party API keys. No Ahrefs / SEMrush / Wappalyzer / BuiltWith / SimilarWeb.

## Trigger

Any message containing a URL plus one of: `audit`, `analyze this site`, `deep dive on`, `read this site`, or `/audit <url>`.

## Native tools used

- `web_extract` — page HTML and main content
- `terminal` — runs the Required terminal commands block below
- `browser_navigate` + `browser_snapshot` + `vision_analyze` — visual read at desktop AND mobile viewports
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

Server header, X-Powered-By, generator meta, framework fingerprints (`/_next/`, `/wp-content/`, `cdn.shopify.com`, Webflow, Squarespace, etc.), analytics/pixel scripts visible in HTML, fonts (grep HTML for `font-family:` declarations and `<link>` tags ending `.woff2` or `.woff`), CDN, SSL cert issuer.

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

See [ITERATIONS-website-analysis.md](../ITERATIONS-website-analysis.md) for the full four-run story, including what each version got wrong and which patch fixed it.

Quick summary:

- **v1** — initial draft. Pretty layout, vague answers, no real numbers.
- **v2** — added Required terminal commands block, output rule (no silent omissions), specificity rule. Caught the agent skipping curl entirely.
- **v3** — added strict "no fabrication" rule. Caught the agent fabricating trust-page 404s when curl wasn't installed.
- **v4** — distinguished `fetch failed` vs `not present`, added `-L` flag to all curl commands so they follow redirects, made mobile reflow a hard required step.

## Known minor gaps in v4

- **Fonts detection** sometimes returns "not present" when the site does use fonts. The current grep pattern misses font declarations inside CSS modules. Open to a v5 patch.
- **Trust pages check** doesn't check for in-page anchor sections (e.g. `#contact` on a single-page landing site). Reports 404s correctly, but a single-page site with embedded contact info will look more compliance-thin than it actually is.

If you patch either of these and it works, send me the diff or open a PR.
