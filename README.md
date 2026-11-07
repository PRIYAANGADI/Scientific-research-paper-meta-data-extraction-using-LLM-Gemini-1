# paperaudit: schema-enforced LLM extraction with a grounding audit

[![tests](https://github.com/YOUR-USERNAME/schema-llm-paper-extraction/actions/workflows/ci.yml/badge.svg)](https://github.com/YOUR-USERNAME/schema-llm-paper-extraction/actions)
![license](https://img.shields.io/badge/license-MIT-blue)

`paperaudit` extracts a standardized 19-field metadata record from each scientific paper (PDF) with the Gemini API, then audits every extracted field with a separate LLM judge that reads the source PDF and must quote its evidence. The audit yields a **Data Grounding Ratio (DGR)** per paper, which gates each record into automated ingestion or human review.

It was built for a pilot on Monterey Bay National Marine Sanctuary (MBNMS) literature, to reduce the manual burden of pulling metadata out of the research that sanctuary management relies on. It works on any PDF corpus.

> **What the DGR is, and is not.** It is the share of audited fields grounded in the source *according to an LLM judge*. It is a grounding rate, not verified accuracy. No human gold standard is bundled here.

## How it works

```
PDFs ──► ingest (pypdf, full text, no chunking)
          └─► extract (Gemini, Pydantic schema, temp 0)  ──► paper_records.json + extraction_matrix.xlsx
                └─► audit (Gemini judge reads the raw PDF) ──► paper_audits.json + audit_report.xlsx
                      └─► metrics (DGR, gate at 85%, per-field rates)
```

- **19 fields**: 11 *factual* (Title, DOI/Link, Citation, Publication, Author, Principal Author Contact, Date, Type, Funding, Funding Type, Technology), 7 *conceptual* (Focus, Main Topic, Topics, Branch of Science/Category, Research Theme, Refined Research Topic, Key Methodology), plus `Source` (the file name, not audited).
- **Two NA sentinels**: `NA` means the paper does not state it; `N/A` means extraction failed for that paper.
- **Five audit states**: `VERIFIED`, `CORRECT_NA` (pass); `UNSUPPORTED_INFERENCE`, `FABRICATED`, `MISSED_EXTRACTION` (fail).
- **DGR** = grounded fields / 18 audited fields. A record passes when DGR ≥ 85% (at most 2 non-grounded fields); otherwise it is routed to human review.
- Runs are **resumable**: finished papers are cached in JSON and skipped on re-run.

## Installation

Requires Python 3.10+ and a [Gemini API key](https://aistudio.google.com/apikey).

```bash
git clone https://github.com/YOUR-USERNAME/schema-llm-paper-extraction.git
cd schema-llm-paper-extraction
python -m venv .venv && source .venv/bin/activate
pip install -e ".[dev]"

cp .env.example .env        # then edit .env and add your key
export GEMINI_API_KEY=...   # or export it directly
```

## Quick start

```bash
# 1. Put your PDFs in data/papers/
# 2. Extract, then audit
paperaudit run

# Outputs land in outputs/:
#   paper_records.json, extraction_matrix.xlsx
#   paper_audits.json,  audit_report.xlsx   (Summary + Ledger sheets)
```

## Usage

```bash
paperaudit extract --pdf-dir data/papers --out-dir outputs --workers 3
paperaudit audit   --pdf-dir data/papers --out-dir outputs --threshold 85
paperaudit report  --audit-json outputs/paper_audits.json --xlsx outputs/audit_report.xlsx   # no API calls
```

Example `report` output:

```
Papers: 14 | fields audited: 252 | grounded: 233 (92.5%)
Mean DGR: 92.5% | pass: 12 | human review: 2 (threshold 85%)
Least reliable fields: Funding Type (57%), Principal Author Contact (71%), ...
```

Use it as a library:

```python
from paperaudit.ingest import discover_pdfs, load_corpus
from paperaudit.llm import make_llm
from paperaudit.extract import run_extraction
from paperaudit.metrics import summarize

texts = load_corpus(discover_pdfs("data/papers"))
records = run_extraction(make_llm("gemini-3.5-flash"), texts, "outputs/paper_records.json")

import json
reports = json.load(open("outputs/paper_audits.json"))
print(summarize(reports)["mean_dgr"])
```

Defaults (model, workers, retries, thinking budget, threshold, field lists) live in [`src/paperaudit/config.py`](src/paperaudit/config.py).

## Project layout

```
src/paperaudit/
  config.py    models, hyperparameters, field lists, sentinels
  schema.py    PaperRecord and audit report Pydantic models
  ingest.py    PDF discovery and full-text loading
  extract.py   structured extraction, retries, threading, resume
  audit.py     adversarial judge prompt and audit runner
  metrics.py   DGR, gate, per-field and per-state statistics
  export.py    formatted Excel outputs
  cli.py       `paperaudit` command
tests/         pytest suite (no network or API key needed)
```

## Tests

```bash
pytest
```

The suite uses fake LLM objects, so it runs offline and costs nothing. It covers the schema, the DGR arithmetic and 85% threshold boundary (16/18 passes, 15/18 fails), retry and resume behaviour, PDF handling, and Excel export.

## Limitations

- The judge is the same model family as the extractor, so errors may be correlated; a second judge from a different family is recommended.
- `pypdf` text extraction can lose table and figure content; the judge reads the raw PDF, so the two do not see identical input.
- Temperature 0 does not guarantee identical output from a hosted API.
- A passing record can still contain up to two non-grounded fields; send flagged fields to review in production.
- Do not commit copyrighted PDFs; `data/papers/` is git-ignored.

## License

[MIT](LICENSE)
