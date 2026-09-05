# Slice File — Analysis Project

Analysis and data-processing work related to the **"Slice file"** ("תיק סלייס"), connected to a
significant pension / insurance fraud affair in Israel over recent years.

> **Source material.** The corpus is **public, non-confidential litigation material** —
> mainly court transcripts and filings — not sealed or private records. We are an external
> research party. Still: nothing here is a finding of wrongdoing, so keep language careful
> ("alleged", "according to the transcript") and keep the PDFs and outputs out of git.

## Background (working summary — to be refined by the team)

Slice (סלייס) operated as a digital insurance/pension agency in Israel. The affair concerns
allegations around the handling of clients' pension and insurance savings — insurance agents
marketed loans to clients and, in exchange, arranged the transfer of their pension savings to
be managed in connection with Slice; those savings were then misappropriated. Regulators,
including the Capital Market, Insurance and Savings Authority, and law-enforcement bodies
have been involved.

The exact scope, timeline, parties, and status of proceedings should be documented in
`docs/` as the team establishes them from primary sources. Correct anything above that is
inaccurate.

## Corpus

62 litigation PDFs (~0.5 GB), several from the same case file, in a shared Google Drive
folder (link: ask a maintainer). Owner: `arielsawicki88@gmail.com`; shared "anyone with the
link → viewer". Put them under `data/` locally (git-ignored).

## Goals

- Ingest the source material into a workable dataset.
- Build reproducible analyses (entity maps, timelines, commission and money flows).
- Produce clear, sourced summaries for team review.

## Repository layout

```
slice/
├── fetch_drive.py           # optional: pull the PDFs from Google Drive into data/
├── src/extract.py           # Task 1: entity + relationship extraction -> per-PDF CSVs
├── src/build_db.py          # merges the per-PDF CSVs into output/slice.db (SQLite)
├── src/merge_entities.py    # LLM-assisted duplicate/category cleanup before build_db's 2nd pass
├── src/make_graph.py        # slice.db -> output/graph.html (interactive relationship map)
├── src/make_profiles.py     # slice.db -> output/slice_profiles.docx (character profiles)
├── prompts/                 # log of the prompts given to Claude, per task
├── data/                    # source PDFs — NOT committed (see .gitignore)
├── docs/                    # established facts, sources, methodology
├── output/                  # generated CSVs, slice.db, graph, docx — NOT committed
│   └── _state/               # batch state files (needed to resume)
├── .env                     # local secrets (API keys) — NOT committed
└── .env.example             # template for .env
```

## Setup

This folder lives inside the `MyPythonScripts` PyCharm project and uses the same
interpreter as the other projects there: the Anaconda base environment at
`C:\Users\Ariel\anaconda3` (Python 3.13). No per-project virtualenv.

```bash
# from this folder, using the shared Anaconda interpreter:
"C:\Users\Ariel\anaconda3\python.exe" -m pip install -r requirements.txt   # once deps are added

copy .env.example .env    # then paste your Claude API key
```

Teammates on other machines: point PyCharm at your local Anaconda/Python 3.13
interpreter, or create a venv — just don't commit it.

## Environment variables

Copy `.env.example` to `.env` and fill in values. `.env` is git-ignored and must never be committed.

| Variable            | Purpose                          |
|---------------------|----------------------------------|
| `ANTHROPIC_API_KEY` | Claude API access for analysis   |

## Task 1 — Character / entity mapping

Read each PDF with the Claude API (structured outputs; thinking disabled) and produce
**one `<stem>.csv` + one `<stem>.edges.csv` per PDF** in `output/`, mapping every named
person/entity in five categories, plus the relationships the excerpt states between them:

| category | who |
|---|---|
| `victim` | savers whose pension money was transferred / lost; saver-plaintiffs |
| `insurance_agent` | agents/agencies/fund-organisers who arranged pension transfers |
| `slice_executive` | Slice founders, officers, directors, controlling shareholders (pre-collapse) |
| `foreign_entity` | non-Israeli funds/companies/banks/trusts used to move, hold or hide funds |
| `israel_entity` | Israeli firms, regulators, appointees or tangential institutions that don't fit the above |

**CSV columns:** `file_name, case_number, category, entity_name_he, entity_name_normalized,
role_in_case, mentioned_by, page, source_quote, confidence, notes, chunk_pages, model`.
UTF-8 with BOM (opens clean in Excel). One row per (entity, page).
`<stem>.edges.csv` columns: `file_name, case_number, from_entity, to_entity, relation,
directed, page, source_quote, confidence, notes, model`.

## Task 2 — database, graph & profiles

Once the per-PDF CSVs exist:

```bash
python src/build_db.py        # -> output/slice.db (entities / mentions / edges) + CSV exports
python src/merge_entities.py  # LLM-assisted merge of duplicate names -> entity_overrides.json
python src/build_db.py        # re-run to apply the merge
python src/make_graph.py      # -> output/graph.html (interactive map; publish as an Artifact)
python src/make_profiles.py   # -> output/slice_profiles.docx (one paragraph per entity)
```

`build_db.py` and `make_graph.py` are free (no API calls) and safe to re-run any time.
`merge_entities.py` and `make_profiles.py` call the API but cache every answer
(`output/_merge_cache.json`, `output/_profile_cache.json`) — a re-run only pays for
genuinely new entities. See each script's docstring for flags (`--dry-run`, `--refresh`,
`--min`).

### Running it

One-time: `"C:\Users\Ariel\anaconda3\python.exe" -m pip install -r requirements.txt`
and put the 62 PDFs in `data/` (or run `python fetch_drive.py`).

Then just **run `src/extract.py`** (green ▶ in PyCharm — no arguments). It asks:

1. how many files to process this run (always smallest → largest),
2. whether to skip the very large PDF(s) for now,
3. shows a **cost estimate**, then asks you to confirm before anything is sent.

Run it again later with a bigger number to do the next batch — PDFs that already
have a CSV are skipped automatically. If a batch is still running when you stop,
run the script again and it offers to **resume / collect** it.

- **Batch API**, ~50% cheaper; usually minutes, occasionally up to 24h.
- Digital-text pages are sent as text; scanned pages as page images (costlier) —
  the estimate shows the scanned-page count per file.
- `output/_state/` holds batch state — needed for resume, don't delete mid-run.
- Failed chunks → `output/<stem>.partial.csv`; per-file detail in `output/<stem>.meta.json`.

**Actual cost, full corpus (62 PDFs, 4,181 pages, ~half scanned):** ~$21 batch
(extraction) + ~$1 (merge) + ~$2 (profiles) ≈ **$24 total**. A batch submission over
~135 MB of payload auto-splits (Anthropic's batch cap is 256 MB) — the script tells you
when it held files back for a follow-up run.
CLI flags for automation (`--limit`, `--resume`, `--dry-run`, `--yes`, `--force`,
`--max-mb`) are documented in the script's docstring.

Prompts given to Claude for each task are logged under `prompts/`.

## Team conventions

- Branch per task; open a PR for review before merging to `main`.
- Never commit anything from `data/` or `output/`, or any personal/identifying data.
- Keep factual claims sourced — the CSV keeps `file_name` + `page` + `source_quote` for that.

## Status

- ✅ Scaffolding, repo, Task 1 extraction pipeline (`src/extract.py`).
- ✅ All 62 PDFs available in the shared Drive folder (~515 MB).
- ✅ Full corpus extracted (all 62 PDFs, incl. the 855-page `עתירה.pdf`).
- ✅ `slice.db` built and merged (~724 entities, ~1,350 edges, ~4,200 mentions);
  `output/graph.html` and `output/slice_profiles.docx` generated.
- ⏳ Merge-quality review (`output/merge_report.csv`); next: push `slice.db` into
  Lobbybot's Postgres RDS, and spot-check extracted names/pages against source PDFs.
