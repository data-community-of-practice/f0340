# f0340
# Connect Grants to Publications and Researchers

A Python script that uses the publications Excel file as a bridge to produce the final two relationship tables of the pipeline graph: one linking grants directly to publication DOIs, and one linking grants to researcher UUIDs (inferred through co-authorship).

No API calls required — pure data joining.

## Pipeline context

f0340 is the last relationship-building step. It takes outputs from across the pipeline and produces the two edge tables needed to complete the grant-centred graph.

```
publications.xlsx  ──────────────────────────────────────────┐
                                                             │
f0339  →  Grants.json                                        │
                                                             ├──  f0340
f0338  →  Researchers.json                                   │
          Researcher_Publication.json  ───────────────────────┘
```

### Complete graph after f0340

| File | Type | Description |
|------|------|-------------|
| `Grants.json` | Node | Grant metadata (f0339) |
| `Researchers.json` | Node | Researcher identity (f0338) |
| `Organisations.json` | Node | Organisation metadata (f0337) |
| `Grant_Publication.json` | Edge | Grant → Publication DOI |
| `Grant_Researcher.json` | Edge | Grant → Researcher |
| `Researcher_Publication.json` | Edge | Researcher → Publication DOI (f0338) |
| `Researcher_Organisation.json` | Edge | Researcher → Organisation (f0337) |

## How it works

**Grant_Publication links** are read directly from the publications Excel file. Every row with both a `Grant_ID` and a `Publication_DOI` becomes one link. Duplicate pairs are deduplicated.

**Grant_Researcher links** are derived by joining: a researcher is linked to a grant if they authored at least one publication that is linked to that grant.

```
Grant_ID → Publication_DOI        (from publications Excel)
Publication_DOI → Researcher_ID   (from Researcher_Publication.json)
─────────────────────────────────
Grant_ID → Researcher_ID          (inferred)
```

This means a researcher may be connected to a grant they were not originally listed on — for example, a co-author who contributed to a grant-funded paper. All connections are traceable through the intermediate DOI.

## Inputs

Three inputs are required:

| Argument | File | Description |
|----------|------|-------------|
| `publications_xlsx` | `Publications_with_high_confidence.xlsx` | The original publications Excel. Must contain `Grant_ID` and `Publication_DOI` columns. Same file used by f0334 and f0339. |
| `researchers_json` | `Researchers.json` | Researcher identity nodes from f0338. Used for name display in the console summary. |
| `researcher_pub_json` | `Researcher_Publication.json` | Researcher-to-publication relationships from f0338. Used to infer grant-researcher links. |

### Required columns in the publications file

| Column | Description |
|--------|-------------|
| `Grant_ID` | Grant identifier. Matched against grant records. |
| `Publication_DOI` | Publication DOI. Used as the join key to researchers. |

## Outputs

| File | Description |
|------|-------------|
| `Grant_Publication.json` | One record per unique grant-DOI pair. |
| `Grant_Researcher.json` | One record per unique grant-researcher pair. |

### Grant_Publication record

```json
{
  "grant_id": "12345678",
  "publication_doi": "10.1111/example.001"
}
```

### Grant_Researcher record

```json
{
  "grant_id": "12345678",
  "researcher_id": "3f2a1b4c-..."
}
```

Both files contain one record per unique pair. A grant linked to ten publications generates ten records in `Grant_Publication.json`. A grant whose publications were co-authored by fifteen researchers generates fifteen records in `Grant_Researcher.json`.

## Requirements

- Python 3.7+
- Library: `openpyxl`

```bash
pip install openpyxl
```

## Usage

All three inputs are required positional arguments:

```bash
python f0340.py publications.xlsx Researchers.json Researcher_Publication.json
```

Outputs are written to the same directory as the publications file by default. Use `--output-dir` to change this:

```bash
python f0340.py publications.xlsx Researchers.json Researcher_Publication.json --output-dir ./output/
```

### All options

| Option | Description |
|--------|-------------|
| `publications_xlsx` | Path to the publications Excel file. |
| `researchers_json` | Path to `Researchers.json` (from f0338). |
| `researcher_pub_json` | Path to `Researcher_Publication.json` (from f0338). |
| `--output-dir`, `-o` | Directory for output files (default: same as publications file). |

### From Spyder or Jupyter

```python
!python "E:\your\folder\f0340.py" "E:\your\folder\publications.xlsx" "E:\your\folder\Researchers.json" "E:\your\folder\Researcher_Publication.json"
```

## Console output

```
Publications xlsx:       /path/to/Publications_with_high_confidence.xlsx
Researchers json:        /path/to/Researchers.json
Researcher-Pub json:     /path/to/Researcher_Publication.json
Output dir:              /path/to/

Grant-Publication links:  1243
Researcher-Pub links:     8432
Researchers:              2075

=======================================================
GRANT CONNECTION SUMMARY
=======================================================
Grant_Publication.json:
  Links:              1243
  Unique grants:      387
  Unique DOIs:        412
Grant_Researcher.json:
  Links:              5891
  Unique grants:      387
  Unique researchers: 2048
=======================================================

Sample grant-researcher links:
  Grant 12345678: Jane Doe, John Smith, Wei Zhang, Alex Brown, Sarah Connor
  Grant 87654321: Maria Garcia, James Wilson
  ...

Saved:
  /path/to/Grant_Publication.json
  /path/to/Grant_Researcher.json
```

## Troubleshooting

| Problem | Solution |
|---------|----------|
| `ERROR: Need 'Grant_ID' and 'Publication_DOI' columns` | The publications file must have both columns with these exact names. The error message lists what columns were actually found. |
| `ERROR: Researchers json not found` | Check the path to `Researchers.json`. This file is produced by f0338. |
| `ERROR: Researcher_Publication json not found` | Check the path to `Researcher_Publication.json`. This file is produced by f0338. |
| Fewer Grant_Researcher links than expected | A grant-researcher link is only created when a researcher from `Researcher_Publication.json` authored a DOI that appears in the publications Excel for that grant. Researchers not in the pipeline (e.g. with no Crossref metadata) will not appear. |
| Grant_Researcher links seem too many | Expected — every co-author on every grant-funded publication is linked to the grant. A multi-author paper on a single grant generates one link per co-author. |

## Limitations

- **Co-authorship inference**: Grant_Researcher links are inferred via publication co-authorship, not from the original grant participant list. A co-author who was not a named grant investigator will still appear as linked to the grant if they published a paper funded by it. For named-investigator-only links, use `participant_list` from `Grants.json` (produced by f0339) instead.
- **DOI as join key**: the join between grants and researchers uses DOI as the bridge. Publications without a DOI in the Excel file, or researchers whose publications were not indexed by Crossref, will not appear in the output.
- **First active worksheet only**: the script reads the first worksheet in the publications Excel file.

## License

This script is provided as-is for research and data management purposes.
