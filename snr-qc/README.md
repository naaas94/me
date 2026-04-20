# QuickCapture

A local-first note capture system with neural parsing, semantic search, and intelligent tagging. Part of the Semantic Note Router (SNR) ecosystem.

Press `Alt+Space` anywhere on Windows, type a thought, and QuickCapture parses it with a local LLM, generates semantic embeddings, and stores everything in a hybrid SQLite + FAISS vector database -- all running entirely on your machine.

## Table of Contents

- [How It Works](#how-it-works)
- [Quick Start](#quick-start)
- [Architecture](#architecture)
- [Processing Pipeline](#processing-pipeline)
- [API Reference](#api-reference)
- [Reliability Model](#reliability-model)
- [Configuration](#configuration)
- [Observability](#observability)
- [CLI Tools](#cli-tools)
- [Testing](#testing)
- [Project Structure](#project-structure)
- [Roadmap](#roadmap)

## How It Works

1. **Capture** -- Press `Alt+Space` to summon a minimal overlay bar. Type a thought and hit Enter.
2. **Normalize** -- The input is canonicalized (Unicode, whitespace, quotes), and deterministic features are extracted: directives (`todo:`, `idea:`, ...), tag headers (`python, ml: ...`), and metrics (`m 2`, `days 0-14`).
3. **Parse** -- If the normalizer is confident enough (tags + directive present, confidence >= 0.7), the LLM is skipped entirely. Otherwise, a local LLM (Ollama, `qwen3.5:9b` by default) extracts tags, sentiment, entities, intent, action items, and a summary.
4. **Embed & Store** -- A `SentenceTransformer` (`all-MiniLM-L6-v2`, 384-dim) generates an embedding. The note and its vector are stored atomically in SQLite and FAISS.
5. **Search** -- Semantic search via `GET /search`, or use the launcher’s **Search** mode (Capture / Search toggle) to query in the overlay without changing the capture hotkey path.

## Quick Start

### Prerequisites

- Python 3.11+
- [Ollama](https://ollama.ai) installed and running
- Windows (for the PyQt6 launcher with global hotkey)

### Installation

```bash
git clone <repository-url>
cd snr-quickcapture
pip install -r requirements.txt

# Pull the default LLM model
ollama pull qwen3.5:9b
```

### Running

**One command (Windows):**

```bash
start_quickcapture.bat
```

This starts both processes with a 10-second stagger so the Brain is ready before the Launcher connects.

**Manual startup (any terminal):**

```bash
# Terminal 1 -- Brain (FastAPI server, keeps ML models in memory)
python scripts/server.py

# Terminal 2 -- Launcher (PyQt6 overlay with global hotkey)
python scripts/launcher.py
```

### First capture

1. Wait for the Brain to finish loading models (watch `logs/brain.log` or the terminal).
2. Press `Alt+Space`. A dark, frameless bar appears at the top-center of your screen.
3. Type something like `python, ml: Implemented transformer for text classification` and press Enter.
4. The bar disappears. The note is parsed, embedded, and stored. Check `http://127.0.0.1:8000/health` to confirm.

## Architecture

QuickCapture runs as two cooperating processes on `localhost`:

```
 Alt+Space
    │
    ▼
┌────────────────────────────────┐
│  Launcher  (PyQt6)             │
│  - Global hotkey listener      │
│  - Optimistic UI               │
│  - Offline fallback → JSONL    │
└────────────┬───────────────────┘
             │ POST /capture
             ▼
┌────────────────────────────────────────────────────┐
│  Brain  (FastAPI @ 127.0.0.1:8000)                 │
│                                                    │
│  ┌──────────────┐  ┌───────────────────────────┐   │
│  │ Raw Capture   │  │ Normalizer                │   │
│  │ (audit trail) │→ │ canonicalize → directives │   │
│  └──────────────┘  │ → tag headers → metrics   │   │
│                     └──────────┬────────────────┘   │
│                                │                    │
│                     ┌──────────▼────────────────┐   │
│                     │ LLM Decision              │   │
│                     │ skip if confidence >= 0.7  │   │
│                     └──┬───────────────┬────────┘   │
│                 skip   │               │  parse     │
│                     ┌──▼──┐    ┌───────▼────────┐   │
│                     │Rule │    │ NeuralParser    │   │
│                     │only │    │ (Ollama LLM)    │   │
│                     └──┬──┘    └───────┬────────┘   │
│                        └───────┬───────┘            │
│                     ┌──────────▼────────────────┐   │
│                     │ Merge tags + Store         │   │
│                     │ SQLite + FAISS + metrics   │   │
│                     └───────────────────────────┘   │
└────────────────────────────────────────────────────┘
```

### Brain (`scripts/server.py`)

FastAPI server that holds all ML models in memory for fast inference:

- **SentenceTransformer** (`all-MiniLM-L6-v2`) for embedding generation
- **Ollama** connection for LLM parsing (`qwen3.5:9b`)
- SQLite + FAISS storage engine
- Shared normalize → parse → persist path via **`scripts/processing_pipeline.py`** (same pipeline as batch processing)
- Background threads: hourly heartbeat, 4-hour offline sync
- Bounded thread pool for LLM work with timeout accounting
- Default data paths (`storage/`, `logs/`, backups) resolve from the **repository root** via `scripts/paths.py` (not the process working directory)

### Launcher (`scripts/launcher.py`)

Lightweight PyQt6 desktop overlay:

- Frameless, translucent, always-on-top bar (expands in **Search** mode for results)
- **Capture / Search** toggle: Search mode calls `GET /search` with the same Brain auth headers as capture and lists hits (distance, metric, summary, tags, date)
- `Alt+Space` global hotkey via `keyboard` library
- Periodic hotkey re-registration (every 30s) for sleep/wake resilience
- Optimistic UI: input clears and window hides immediately on submit (capture mode)
- Offline fallback: saves to `storage/offline_notes.jsonl` when the Brain is unreachable (atomic append via temp + rename)
- Health checks every 60 seconds with color-coded status feedback

## Processing Pipeline

### Normalization Layer (`scripts/normalizer.py`)

A deterministic, idempotent preprocessing pipeline that runs before any LLM call:

| Pass | Function | Purpose |
|------|----------|---------|
| A | `canonicalize()` | Unicode NFKC, smart quotes → straight quotes, whitespace normalization |
| B | `extract_directive()` | Detects prefixes: `todo:`, `insight:`, `idea:`, `question:`, etc. |
| C | `extract_tag_header()` | Parses `tag1, tag2: body text` patterns (with false-positive disambiguation) |
| D | `extract_metrics()` | Extracts counters and ranges: `m 2`, `days 0-14`, `used 5 tokens` |
| E | `normalize_tag()` | Lowercases, spaces → hyphens, dots allowed for hierarchy, max 30 chars |

The pipeline produces a `NormalizedEntry` with a confidence score. If confidence >= 0.7 and tags are present, the LLM is bypassed entirely (~30-40% of well-structured inputs).

### Neural Parser (`scripts/parse_input.py`)

When the normalizer's confidence is too low, the `NeuralParser` calls Ollama with few-shot prompting to extract:

- **summary** -- one-line description
- **tags** -- semantic topic tags
- **sentiment** -- positive / neutral / negative
- **entities** -- key concepts or proper nouns
- **intent** -- meeting, task, idea, learning, etc.
- **action_items** -- extracted to-dos
- **people** -- mentioned names

Deterministic tags from the normalizer always take priority over LLM-suggested tags via `merge_tags()`.

### Shared processing path (`scripts/processing_pipeline.py`)

Server (`/capture`, offline sync, single-entry helpers) and **`scripts/batch_processor.py`** both call this module so normalization, LLM timeouts, and persistence stay consistent across API and scheduled batch runs.

### Storage Engine (`scripts/storage_engine.py`)

Hybrid storage system:

- **SQLite** -- structured data with WAL mode, thread-local connections
- **FAISS** -- `IndexFlatL2` with `IndexIDMap`, 384 dimensions, persisted to disk (vector mutations and `id_map` I/O are serialized for thread safety)
- **Embedding** -- generated via `all-MiniLM-L6-v2` (CPU or GPU, auto-detected); vectors are persisted as **EmbeddingBlob v1** in SQLite (not pickle). Use `scripts/migrate_database.py` to convert legacy databases.
- **Tag statistics** -- usage counts, avg confidence, avg semantic density
- **Dead letter queue** -- notes that exhaust all retries

### Database Schema

| Table | Purpose |
|-------|---------|
| `raw_notes` | Audit trail -- every input is saved here before processing |
| `notes` | Processed notes with tags, embeddings, metadata, validation results |
| `dead_letter_queue` | Notes that failed after max retries, with full error context |
| `tag_statistics` | Aggregate tag metrics (usage count, avg confidence, avg density) |
| `processing_metrics` | Per-note processing telemetry (LLM time, confidence, errors) |

## API Reference

The Brain serves at `http://127.0.0.1:8000`.

### Authentication (optional)

By default the API is open on localhost. Set **`BRAIN_API_TOKEN`** to a non-empty value to require the **`X-Brain-Token`** header on requests. The launcher reads the same env var and sends the header automatically; use the header with `curl` or other clients when auth is enabled.

| Method | Endpoint | Description |
|--------|----------|-------------|
| `POST` | `/capture` | Capture and process a note |
| `GET` | `/search` | Semantic vector search (hits include `distance` and `metric`) |
| `POST` | `/ask` | Natural-language **read**: structured query plan from Ollama, compiled to parameterized `SELECT` on a read-only SQLite connection (see below) |
| `GET` | `/health` | System health with memory, LLM runtime, and stats |
| `GET` | `/stats` | Database and tag statistics |
| `GET` | `/pending` | Raw notes still pending processing |
| `GET` | `/failed` | Failed notes eligible for retry |
| `GET` | `/dlq` | Dead letter queue entries |
| `POST` | `/retry/{id}` | Re-process the **existing** failed `raw_notes` row (no duplicate raw row) |

### POST /capture

```bash
curl -X POST http://127.0.0.1:8000/capture \
  -H "Content-Type: application/json" \
  -d '{"text": "python, ml: Implemented transformer for text classification"}'
# When BRAIN_API_TOKEN is set, add: -H "X-Brain-Token: …"
```

Response (single entry):

```json
{
  "status": "captured",
  "note_id": "a1b2c3d4-...",
  "trace_id": "tr-...",
  "summary": "Implemented transformer for text classification",
  "tags": ["python", "ml"],
  "processing_time_ms": 142.5,
  "llm_used": false,
  "confidence": 0.85
}
```

Multi-entry inputs (separated by `---`, `***`, or `===`) return `entry_count`, `successful`, `failed`, and an `entries` array.

### GET /search

```bash
curl "http://127.0.0.1:8000/search?query=machine+learning&limit=5"
# With Brain auth enabled:
curl "http://127.0.0.1:8000/search?query=machine+learning&limit=5" \
  -H "X-Brain-Token: $BRAIN_API_TOKEN"
```

Returns notes ranked by **Euclidean L2** distance in embedding space. Each hit includes **`distance`** (lower is nearer) and **`metric`: `"l2"`** (FAISS reports squared L2 internally; the API exposes true L2). Older clients that assumed a dummy **`score`** field should use **`distance`** / **`metric`** instead.

### POST /ask

Send a JSON body with a **`question`** string. Ollama returns a validated structured plan (allowlisted tables/columns, row cap); the server compiles it to parameterized `SELECT` only and runs it read-only. Arbitrary SQL from the model is not executed, and `notes.embedding_vector` is not exposed through this API.

```bash
curl -X POST http://127.0.0.1:8000/ask \
  -H "Content-Type: application/json" \
  -d '{"question": "How many notes tagged python?"}'
```

### GET /health

Returns component status including:

- `parser` -- LLM model name and readiness
- `storage` -- database and vector store health
- `memory.stats` -- process RSS/VMS, estimated Ollama memory
- `llm_runtime` -- in-flight workers, timeout counters, spike warnings
- `stats` -- total notes, pending, failed counts

## Reliability Model

QuickCapture is designed around the principle: **never lose a note**.

### Defense layers

| Layer | Mechanism | Trigger |
|-------|-----------|---------|
| 1 | Raw note capture | Every input → `raw_notes` table *before* processing |
| 2 | Offline fallback | Brain unreachable → `storage/offline_notes.jsonl` |
| 3 | Offline sync | Background thread syncs JSONL → database every 4 hours (offline file rewrites are atomic: temp + rename) |
| 4 | Retry | Failed notes can be retried via `/retry/{id}` or batch processor |
| 5 | Dead letter queue | After max retries → DLQ with full stack trace for manual review |
| 6 | Emergency backup | Database failure → `storage/emergency_backup*.jsonl` (runtime spill files) |

### Note lifecycle

```
pending → processing → completed
                    ↓
                 failed → retrying → completed
                               ↓
                          dead_letter
```

### Memory management

- Automatic memory pressure detection via `psutil`
- Processing deferred when system memory > 90%
- Raw-note-only mode (no LLM) when memory > 95%
- Periodic garbage collection via heartbeat thread

### Hotkey resilience

- `Alt+Space` (not a Windows-key binding)
- Handle-based bind/unbind with periodic forced re-registration every 30 seconds
- Exponential backoff on registration failures
- Full lifecycle logging to `logs/launcher.log`

## Configuration

### Environment variables

| Variable | Default | Description |
|----------|---------|-------------|
| `BRAIN_API_TOKEN` | *(unset)* | If set and non-empty, requires `X-Brain-Token` on Brain HTTP requests (launcher sends it automatically) |
| `QUICKCAPTURE_PARSE_WORKERS` | `4` | Max background parser threads in the Brain |
| `QUICKCAPTURE_LLM_TIMEOUT_SECONDS` | `240` | Per-attempt LLM parse timeout |
| `QUICKCAPTURE_LLM_TIMEOUT_ATTEMPTS` | `1` | Max parse attempts per entry |
| `QUICKCAPTURE_CAPTURE_TIMEOUT_SECONDS` | `300` | Launcher HTTP timeout for `/capture` |
| `QUICKCAPTURE_OLLAMA_KEEP_ALIVE` | *(none)* | Ollama model residency hint |
| `QUICKCAPTURE_OLLAMA_NUM_CTX` | *(none)* | Ollama context size override |
| `QUICKCAPTURE_OLLAMA_NUM_PREDICT` | *(none)* | Ollama output token cap |
| `RAW_NOTES_RETENTION_DAYS` | `90` | Retention window for completed `raw_notes` when running `scripts/raw_notes_retention.py` (see script `--help`) |

### YAML config files (`config/`)

| File | Governs |
|------|---------|
| `storage_config.yaml` | SQLite paths, WAL, cache size, FAISS settings, backup retention |
| `observability.yaml` | Metrics collection intervals, health thresholds, drift detection |
| `tag_intelligence.yaml` | Tag suggestion confidence, drift windows, hierarchy definitions |
| `semantic_validation.yaml` | Semantic density, content classification, coherence scoring |
| `grammar_rules.yaml` | Parsing rules, validation thresholds, processing rules |

## Observability

The `observability/` package provides four subsystems:

| Module | Purpose |
|--------|---------|
| `metrics_collector.py` | Prometheus-compatible metrics (ingestion latency, coherence, tag quality) |
| `health_monitor.py` | System health scoring with critical/warning issue detection |
| `performance_tracker.py` | Operation-level timing and bottleneck detection |
| `drift_detector.py` | Semantic and performance drift alerts |

Health data is surfaced through the `/health` endpoint and logged to `logs/brain.log`.

## CLI Tools

| Command | Script | Description |
|---------|--------|-------------|
| `quick-add` | `scripts/quick_add.py` | Legacy CLI for adding notes without the launcher |
| `review-outliers` | `scripts/review_outliers.py` | Interactive review and correction of low-quality notes |
| batch processor | `scripts/batch_processor.py` | Process pending notes, retry failures, sync offline backup |
| migrate database | `scripts/migrate_database.py` | Schema upgrades; optional **`--migrate-tags`** / **`--tag-dry-run`** to normalize stored tags (backs up by default) |
| raw notes retention | `scripts/raw_notes_retention.py` | Policy purge of **completed** old `raw_notes` (default dry-run; **`--execute`** requires backup — see script help) |

### Batch processor

Designed for scheduled runs (e.g., cron). Raw-note retention is a **separate** job (not invoked automatically from the batch processor):

```bash
python scripts/batch_processor.py               # Process everything
python scripts/batch_processor.py --pending-only # Only pending raw notes
python scripts/batch_processor.py --retry-only   # Only retry failed notes
python scripts/batch_processor.py --sync-only    # Only sync offline JSONL
```

## Testing

CI (`.github/workflows/ci.yml`) installs `pip install -r requirements.txt` then **`pip install -e ".[ci]"`** or **`".[dev]"`** and runs:

```bash
pytest -m "not heavy" --tb=short -q
```

Slow or opt-in suites may be marked **`heavy`**; some concurrency tests use **`threading`**. Locally:

```bash
pip install -r requirements.txt
pip install -e ".[dev]"   # or ".[ci]" for a lighter extra set

# Default full run (excludes heavy)
pytest -m "not heavy" tests/

# Everything including heavy
pytest tests/

# With coverage report
pytest -m "not heavy" tests/ --cov=scripts --cov=observability --cov-report=html

# Examples
pytest tests/test_normalizer.py -v          # Normalization tests
pytest tests/test_basic_functionality.py -v  # Core pipeline tests
pytest tests/test_runtime_reliability.py -v  # Hotkey, health, runtime tests
pytest tests/test_observability.py -v        # Observability subsystem tests
pytest tests/test_failed_notes.py -v         # Error handling and DLQ tests
```

## Project Structure

```
snr-quickcapture/
├── scripts/                       # Core application
│   ├── server.py                  # FastAPI Brain (backend)
│   ├── launcher.py                # PyQt6 overlay (frontend)
│   ├── brain_auth.py              # Optional X-Brain-Token / BRAIN_API_TOKEN
│   ├── paths.py                   # Repository root resolution for storage/logs
│   ├── processing_pipeline.py     # Shared normalize → parse → persist
│   ├── normalizer.py              # Deterministic normalization pipeline
│   ├── parse_input.py             # NeuralParser (Ollama LLM)
│   ├── nl_ask.py                  # NL read path (structured plan → read-only SQL)
│   ├── nl_ask_prompts.py          # Prompts for /ask
│   ├── validate_note.py           # Multi-dimensional validation
│   ├── storage_engine.py          # SQLite + FAISS hybrid storage
│   ├── models.py                  # Data models (ParsedNote, etc.)
│   ├── tag_intelligence.py        # Tag suggestions and drift detection
│   ├── batch_processor.py         # Scheduled batch processing
│   ├── raw_notes_retention.py     # Optional completed raw_notes retention job
│   ├── review_outliers.py         # Outlier review CLI
│   ├── snr_preprocess.py          # SNR export preprocessing
│   ├── quick_add.py               # Legacy CLI entry point
│   ├── migrate_database.py        # Schema migration + optional tag migration
│   └── kill_port.py               # Port cleanup utility
│
├── observability/                 # Monitoring subsystem
│   ├── metrics_collector.py       # Prometheus metrics
│   ├── health_monitor.py          # Health scoring
│   ├── performance_tracker.py     # Operation timing
│   └── drift_detector.py          # Drift detection
│
├── config/                        # YAML configuration
│   ├── storage_config.yaml
│   ├── observability.yaml
│   ├── tag_intelligence.yaml
│   ├── semantic_validation.yaml
│   └── grammar_rules.yaml
│
├── storage/                       # Runtime data (mostly gitignored)
│   ├── quickcapture.db            # SQLite database
│   ├── vector_store/              # FAISS index + ID map
│   ├── backups/                   # Pre-migration / retention backups (gitignored)
│   ├── offline_notes.jsonl        # Offline capture fallback
│   └── emergency_backup*.jsonl    # Emergency DB-failure fallback
│
├── logs/                          # Application logs
│   ├── brain.log
│   └── launcher.log
│
├── tests/                         # Test suite (markers: heavy, threading)
│   ├── test_normalizer.py
│   ├── test_basic_functionality.py
│   ├── test_capture_http.py
│   ├── test_nl_ask_readonly_plan.py
│   ├── test_processing_pipeline_rule_only.py
│   └── …                          # Additional modules (search, auth, retention, FAISS lock, …)
│
├── .github/workflows/
│   └── ci.yml                     # Push/PR: Python 3.11–3.12 × ci/dev extras
│
├── debugging/                     # Debug and diagnostic scripts
│
├── start_quickcapture.bat         # Windows startup script
├── pyproject.toml                 # Python project config (extras: ci, dev)
├── requirements.txt               # Dependencies
├── CHANGELOG.md                   # Version history
└── api_flow.md                    # API / pipeline flow reference
```

Backlog plans and decision logs for larger efforts may live under **`.dev/plans/`** in a developer checkout (most of `.dev/` is gitignored by default).

## Roadmap

**Shipped in recent work:**

- **Launcher search mode** -- Capture / Search toggle with in-overlay `GET /search` results
- **`POST /ask`** -- Natural-language reads via validated structured plans and read-only SQL (not arbitrary model SQL)

**Still open (examples):**

- **In-launcher NL answers** -- Show `/ask` results in the quick bar (capture/search HTTP paths unchanged until then)
- **Async capture** -- Decouple capture latency from LLM work (see project backlog / decision logs)

## Tech Stack

| Layer | Technology |
|-------|------------|
| Language | Python 3.11+ |
| Backend | FastAPI, Uvicorn |
| Frontend | PyQt6, `keyboard` |
| LLM | Ollama (`qwen3.5:9b`) |
| Embeddings | SentenceTransformers (`all-MiniLM-L6-v2`) |
| Database | SQLite (WAL mode) |
| Vector store | FAISS (`IndexFlatL2`, 384-dim) |
| Metrics | Prometheus client |
| Logging | structlog |
| ML stack | PyTorch, scikit-learn |
| CI | GitHub Actions (Python 3.11 / 3.12, `ci` / `dev` extras) |

---

**Author**: Alejandro Garay
**Version**: 1.0.0
**License**: MIT
