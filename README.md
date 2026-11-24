# CloudServe Support Automation Capstone

A CLI-first support-ticket automation capstone powered by Google Gemini and Chroma.

## 1. Setup

### Automated (macOS / Linux)

Run from the repository root:

```bash
source setup/setup.sh
```

The script installs `uv` and Python 3.13.13 if missing, creates `.env` from `.env.example` (prompting for `GEMINI_API_KEY`), syncs `.venv` with `uv sync --frozen`, and verifies the checked-in Chroma vector store and SQLite database.

### Manual

**macOS / Linux:**

```bash
uv sync --frozen
cp .env.example .env
source .venv/bin/activate
```

**Windows (PowerShell):**

```powershell
uv sync --frozen
Copy-Item .env.example .env
.venv\Scripts\Activate.ps1
```

Add your API key to `.env` (ignored by Git — never commit it). `GEMINI_API_KEY` or `GOOGLE_API_KEY` is required to run.

Optional settings: `DRAFT_GENERATION_PROVIDER` (defaults to `GEMINI`; use `OFFLINE` for extractive drafts), `GEMINI_EMBEDDING_MODEL`, `GEMINI_LLM_MODEL`, and `GEMINI_QUALITY_JUDGE_MODEL`.

## 2. Run

Run against the validation set:

```bash
uv run cloudserve run --input data/tickets/validation_tickets.json --skip-draft --skip-draft-review
```

With `.venv` activated, use `cloudserve run --input data/tickets/validation_tickets.json --skip-draft --skip-draft-review` directly.

- `--skip-draft` escalates draft-eligible tickets to human review without calling the draft provider.
- `--skip-draft-review` skips post-generation draft review checks when drafting is enabled.
- Omit both flags for full Gemini draft generation and review.

Outputs (`results.jsonl`, `decision_log.jsonl`, `metrics.json`, `metrics.md`, `exceptions.jsonl`, `failure_report.json`, `failure_report.md`) are written to `outputs/<RUN_ID>/`, and durable records are persisted in `data/runtime/cloudserve.sqlite3`. Running with Gemini keys makes external API requests and may incur provider charges.

Optional post-run quality judge (when drafts were generated):

```bash
uv run cloudserve evaluate-quality --run-dir outputs/<RUN_ID> --input data/tickets/validation_tickets.json --ground-truth data/evaluation/ground_truth_responses.json
```

### Reinitialize after KB article changes

> **Important:** Whenever you edit an existing article, add a new article, or delete an article in `data/kb/`, you must re-ingest the Chroma vector store. Otherwise retrieval uses a stale index and will not reflect your changes.

```bash
uv run python scripts/init_vector_store.py
uv run --env-file .env python scripts/ingest_kb.py
```

`ingest_kb.py` upserts changed passages, removes stale records, and rewrites `data/index/manifest.json`. It requires an API key in `.env` and makes Gemini embedding requests. To also reset the SQLite runtime database and output schemas, run `source setup/setup.sh --reinit-db`.