# Screenshot reference

Drop files into this directory using these exact filenames. The parent ITERATIONS doc references them by name.

Each audit was captured in two halves (top + bottom of the Telegram message) because each report runs longer than one screen.

| File | What it shows |
|------|---------------|
| `run-1-vague-top.png` | Run 1 — at a glance + SEO surface, vague answers, no real numbers |
| `run-1-vague-bottom.png` | Run 1 — UX + Performance + Strengths/Weaknesses, "Fast TTFB" / "Minimal" placeholders |
| `run-2-fabricated-top.png` | Run 2 — the fabricated 404s on trust pages, OG card "Present" without proof |
| `run-2-fabricated-bottom.png` | Run 2 — UX + performance + The Sharper Thing for the same fabricated run |
| `diagnostic-top.png` | Diagnostic — first half showing `curl: command not found` repeating |
| `diagnostic-bottom.png` | Diagnostic — second half + Chrome `agent-browser install` errors |
| `run-3-mixed-top.png` | Run 3 — at a glance + SEO with mixed real values and "fetch failed" labels |
| `run-3-mixed-bottom.png` | Run 3 — UX (no colour palette / mobile reflow) + performance fields all "fetch failed" |
| `run-4-clean-top.png` | Run 4 — at a glance + SEO with real schema + sitemap content + trust-page status codes |
| `run-4-clean-bottom.png` | Run 4 — real Vercel/SSL/perf numbers + colour palette hex + mobile reflow analysis |

Once files are in place, the parent ITERATIONS-website-analysis.md will render them inline.
