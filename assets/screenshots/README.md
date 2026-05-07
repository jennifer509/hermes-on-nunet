# Screenshot reference

Drop files into this directory using these exact filenames. The ITERATIONS doc references them by name.

| File | What it shows |
|------|---------------|
| `run-1-vague.png` | Run 1 output: pretty layout, vague answers ("Fast TTFB", "Minimal trust pages"), no real numbers |
| `run-2-fabricated.png` | Run 2 output: trust pages reported as 404s, OG card "Present" — values that look real but were fabricated because curl wasn't installed |
| `diagnostic.png` | The diagnostic output showing `curl: command not found` for every command |
| `run-3-mixed.png` | Run 3 output: real Vercel CDN, real hex palette, but several fields still "fetch failed" |
| `run-4-clean.png` | Run 4 output: real page weight (101.5 KB), response time (151 ms), SSL issuer Let's Encrypt R13/R12, mobile reflow analysis |

Once files are in place, the parent ITERATIONS-website-analysis.md will render them inline.
