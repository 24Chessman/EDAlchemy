# EDAlchemy

**Turn a raw CSV into a narrated EDA report in under 30 seconds.**

Upload a CSV. Get back a single, self-contained HTML report with statistics, charts, data-quality flags, and a plain-English narrative written by Claude — grounded strictly in the numbers it computed, nothing invented.

No login. No database. No data stored after the request completes.

---

## Why

Data scientists spend 60–80% of project time on exploratory data analysis — and most of it is the same mechanical loop every time: load the file, check nulls, plot distributions, eyeball correlations, hunt for outliers. EDAlchemy automates that loop so you start at the insight, not at `df.head()`.

---

## What it does

- 🔍 **Auto-detects column types** — numeric, categorical, datetime, boolean, text, identifier — with zero configuration
- 📊 **Full statistical profile** — distributions, skewness/kurtosis, IQR outliers, correlations (Pearson + Spearman fallback), Cramér's V for categoricals
- 🚩 **Data quality flags** — nulls, duplicates, constant columns, likely IDs, mixed types — each with severity (info / warning / critical)
- 🎯 **Target column detection** — recognizes common target names (`target`, `label`, `churn`, `survived`, etc.), infers classification vs. regression, and ranks feature correlations against it
- 🤖 **AI-written narrative** — Claude reads the computed stats (never the raw rows) and writes a four-part summary: overview, data quality, key findings, recommendations
- 📦 **One self-contained HTML file** — every chart and style inlined, works fully offline after download, no JS dependency for readability

---

## How it works

```
Browser ──POST /upload (CSV)──▶ FastAPI Server
                                     │
                                     ├─ File Validator
                                     ├─ EDA Pipeline (in-process, synchronous)
                                     │    ├─ Ingestion → Schema Profiling
                                     │    ├─ Quality Checks
                                     │    ├─ Univariate + Bivariate Analysis
                                     │    └─ Target Detection
                                     ├─ LLM Narrator (Claude API — stats only, no raw data)
                                     └─ Report Builder (Jinja2 + Plotly)
                                            └─▶ self-contained HTML report
```

Everything runs synchronously in a single request/response cycle for a typical 50k-row dataset — no task queue, no background workers, no persistent storage.

---

## Quick start

```bash
# Clone and configure
git clone https://github.com/<your-username>/edalchemy.git
cd edalchemy
cp .env.example .env
# Add your ANTHROPIC_API_KEY to .env

# Install dependencies
pip install -r requirements.txt

# Run
uvicorn main:app --reload
```

Open **http://localhost:8000** and drop in a CSV.

### With Docker

```bash
docker build -t edalchemy .
docker run -p 8000:8000 -e ANTHROPIC_API_KEY=your_key_here edalchemy
```

> ⚠️ Windows users: `kaleido` (used for chart export) has known issues on native Windows — use Docker or WSL.

---

## API

### `POST /upload`

Upload a CSV, get back the full report as an HTML string.

```bash
curl -X POST http://localhost:8000/upload \
  -F "file=@your_data.csv"
```

```json
{ "report_html": "<!DOCTYPE html>..." }
```

| Status | Meaning |
|---|---|
| `200` | Report generated successfully |
| `400` | Invalid request (e.g. not a CSV) |
| `413` | File exceeds the size limit |
| `422` | File could not be parsed as CSV |

### `GET /health`

```bash
curl http://localhost:8000/health
# {"status": "ok"}
```

---

## Configuration

Set via environment variables or a `.env` file:

| Variable | Default | Description |
|---|---|---|
| `ANTHROPIC_API_KEY` | *required* | Claude API key used by the narrator |
| `MAX_FILE_SIZE_MB` | `100` | Maximum accepted CSV size |
| `SAMPLE_THRESHOLD_ROWS` | `100000` | Auto-sample datasets larger than this |
| `LLM_MODEL` | `claude-sonnet-4-6` | Model used to generate the narrative |

---

## Performance targets

| Dataset size | Target time |
|---|---|
| < 10k rows | < 5 seconds |
| 10k – 100k rows | < 30 seconds |
| 100k – 500k rows | < 90 seconds (charts use a stratified sample) |
| > 500k rows | Auto-sampled to 100k rows, with a warning shown to the user |

---

## Project structure

```
edalchemy/
├── main.py                  # FastAPI app entry point
├── config.py                # Settings (.env-driven)
├── pipeline/                # EDA computation modules
│   ├── ingestor.py          # Encoding/delimiter detection, sampling
│   ├── schema_profiler.py   # Semantic type inference + base stats
│   ├── quality_checker.py   # Nulls, duplicates, constants, mixed types
│   ├── univariate.py        # Per-column statistics
│   ├── bivariate.py         # Correlations, Cramér's V
│   ├── target_detector.py   # Heuristic target column ID
│   └── runner.py            # Orchestrates the full pipeline
├── llm/                     # Claude API integration
│   ├── narrator.py
│   └── prompts.py
├── report/                  # Report generation
│   ├── builder.py
│   ├── charts.py
│   └── templates/
│       └── report.html
├── models/
│   └── eda_result.py        # Pydantic models for all pipeline outputs
├── static/
│   └── index.html           # Upload UI (vanilla JS, no build step)
└── tests/
```

---

## Tech stack

- **Backend:** FastAPI + Uvicorn
- **Data processing:** pandas, NumPy, SciPy
- **Charts:** Plotly (exported to inline SVG/HTML, no external requests)
- **Templating:** Jinja2
- **LLM:** Claude API (`claude-sonnet-4-6`)
- **Frontend:** Vanilla JS — no framework, no build step

---

## Privacy

- The CSV is held in memory for the duration of the request and discarded afterward — nothing is written to disk or a database.
- Only **computed statistics** (column profiles, correlations, quality flags) are sent to Claude for the narrative — raw rows never leave the server.
- The downloaded report has no external dependencies and works fully offline.

---

## Roadmap

**Phase 2 (planned):** a follow-up chat interface. After a report is generated, the computed `EDAResult` is kept server-side under a session ID, and users can ask natural-language questions about their data — answered from the pre-computed profile, with no need to re-read the original CSV.

---

## Contributing

Issues and PRs are welcome. Before submitting a change to the pipeline, run the test suite:

```bash
pytest tests/
```

---

## License

MIT
