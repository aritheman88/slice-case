# slice-case

Static site for **slice-case.net** — an interactive relationship graph of the
people and entities in the "Slice" pension-fraud affair (פרשת סלייס), built
from 62 public litigation filings.

This repo is **generated output**, not source. The actual pipeline (PDF
extraction, database, merge, DOCX profiles) lives in the separate
[`slice`](https://github.com/aritheman88/slice) repo; `index.html` here is
produced by that repo's `src/make_graph.py` + `src/make_site.py`.

## Contents

- `index.html` — the graph (cytoscape.js), with a client-side password
  overlay (same soft-filter pattern as efoliknot.net — a deterrent, not real
  security; the password is visible in the page source).
- `CNAME` — GitHub Pages custom-domain config for `www.slice-case.net`.

## This repo is intentionally public

Per its own README, the `slice` pipeline repo keeps `data/` and `output/`
(source PDFs, per-file CSVs, the full database) out of git — deliberately,
since that material names real people in connection with unproven fraud
allegations. **This repo is the one deliberate exception**: GitHub Pages'
free tier only serves a custom domain from a public repo, so the choice here
was made knowingly — see the project owner's decision to publish as-is
(2026-09) rather than pay for a private-repo host. The password gate on the
live site is cosmetic; anyone can read `index.html`'s source directly on
GitHub regardless of it.

## Updating

From the `slice` repo:

```bash
python src/make_graph.py   # rebuild output/graph.html from slice.db
python src/make_site.py    # wrap it with the password gate -> ../slice-case-site/
```

Then commit and push from `slice-case-site/`.
