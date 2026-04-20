# f0340
BDBSF Initial Papers citation
# Citation Count Fetcher

A Python script that fetches the number of citations for each publication in your dataset using DOIs. Uses OpenAlex as the primary source with Crossref as a fallback.

## How it works

### Phase 1 — OpenAlex (primary, batched)
Queries OpenAlex in batches of 50 DOIs per request using the filter pipe syntax. OpenAlex aggregates citation data from Crossref, PubMed, Microsoft Academic Graph, and other sources, giving it broader coverage than any single source. Returns `cited_by_count` for each work.

### Phase 2 — Crossref (fallback)
For DOIs not found in OpenAlex, queries the Crossref API individually and extracts `is-referenced-by-count`. Crossref counts tend to be lower since they only track citations from other Crossref-registered DOIs, but this ensures maximum coverage.

## Requirements

- Python 3.7+
- Libraries: `openpyxl`, `requests`

```bash
pip install openpyxl requests
```

## Setup

Same `config.ini` as the other scripts:

```ini
[crossref]
email = yourname@example.com
delay = 1
save_every = 50
max_retries = 3
```

No API key required — both OpenAlex and Crossref are free.

## Input file format

Any Excel file with a `Publication_DOI` column. This is typically your original input file or the output of `crossref_author_fetch.py`.

## Usage

### From a terminal

```bash
python fetch_citations.py Publications_with_high_confidence.xlsx
```

### From Spyder

```python
!python "E:\your\folder\fetch_citations.py" "E:\your\folder\Publications_with_high_confidence.xlsx"
```

## Output

The script adds two columns to a copy of your input file (`<input>_with_citations.xlsx`):

| Column | Description |
|---|---|
| `Citation_Count` | Total number of citations for this publication |
| `Citation_Source` | Where the count came from: `OpenAlex`, `Crossref`, or `Not found` |

Duplicate DOIs (same publication appearing on multiple rows) receive identical citation counts.

The script also prints a summary including total citations and the top 10 most cited papers.

## Performance

OpenAlex batching makes this script efficient — 500 DOIs can be fetched in around 10 batch requests (roughly 10–20 seconds), compared to 500 individual requests for other APIs. Crossref fallback is only used for the small number of DOIs missing from OpenAlex.

## Cache and resumability

The script creates a `<input>_citation_cache.json` file. Re-running skips previously fetched DOIs. Delete the cache to force a full refresh (useful if you want updated citation counts months later).

## Pipeline context

This script can be run independently at any point — it only needs the `Publication_DOI` column. It's designed to run on your original input file:

```
python fetch_citations.py Publications_with_high_confidence.xlsx
```

It complements the main pipeline:

```
1. python crossref_author_fetch.py       → author names
2. python extract_unique_authors.py       → unique authors
3. python fetch_affiliations.py           → affiliations
4. python extract_unique_orgs.py          → unique organisations
5. python classify_orgs.py                → classification
6. python geotag_orgs.py                  → geolocation + map
7. python fetch_citations.py              → citation counts
```

## Troubleshooting

| Problem | Solution |
|---|---|
| `Citation_Count` shows `N/A` | The DOI wasn't found in either OpenAlex or Crossref. It may be invalid, very new, or from a publisher not indexed by either service. |
| Low citation counts | This is expected for recent publications. Citation counts are cumulative and grow over time. |
| Counts differ from Google Scholar | Google Scholar typically reports higher counts because it indexes non-DOI sources like theses, books, and preprints. OpenAlex and Crossref only count DOI-registered citations. |
| Want updated counts | Delete the `_citation_cache.json` file and re-run. Citation counts change over time as new papers cite existing ones. |

## Limitations

- **Citation counts are point-in-time snapshots.** They reflect the count at the time of fetching and will increase as new papers cite the work.
- **OpenAlex vs Crossref counts may differ.** OpenAlex typically reports higher counts because it aggregates multiple sources. The `Citation_Source` column tells you which was used.
- **Very new publications** (published in the last few months) may show 0 citations even if they've been cited, due to indexing delays.

## License

This script is provided as-is for research and data management purposes.
