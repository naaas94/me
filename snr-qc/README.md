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
3. **Parse** -- If the normalizer is confident enough (tags + directive present, confidence >= 0.7), the LLM is skipped entirely. Otherwise, a local LLM (Ollama, `qwen3:0.6b` by default) extracts tags, sentiment, entities, intent, action items, and a summary.
4. **Embed & Store** -- A `SentenceTransformer` (`all-MiniLM-L6-v2`, 384-dim) generates an embedding. The note and its vector are stored atomically in SQLite and FAISS.
5. **Search** -- Semantic search over all stored notes via the `/search` endpoint or (planned) in-launcher search mode.

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
ollama pull qwen3:0.6b
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
- **Ollama** connection for LLM parsing (`qwen3:0.6b`)
- SQLite + FAISS storage engine
- Background threads: hourly heartbeat, 4-hour offline sync
- Bounded thread pool for LLM work with timeout accounting

### Launcher (`scripts/launcher.py`)

Lightweight PyQt6 desktop overlay:

- Frameless, translucent, always-on-top 700x100px bar
- `Alt+Space` global hotkey via `keyboard` library
- Periodic hotkey re-registration (every 30s) for sleep/wake resilience
- Optimistic UI: input clears and window hides immediately on submit
- Offline fallback: saves to `storage/offline_notes.jsonl` when the Brain is unreachable
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

### Storage Engine (`scripts/storage_engine.py`)

Hybrid storage system:

- **SQLite** -- structured data with WAL mode, thread-local connections
- **FAISS** -- `IndexFlatL2` with `IndexIDMap`, 384 dimensions, persisted to disk
- **Embedding** -- generated via `all-MiniLM-L6-v2` (CPU or GPU, auto-detected)
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

The Brain serves at `http://127.0.0.1:8000`. No authentication (localhost only).

| Method | Endpoint | Description |
|--------|----------|-------------|
| `POST` | `/capture` | Capture and process a note |
| `GET` | `/search` | Semantic vector search |
| `GET` | `/health` | System health with memory, LLM runtime, and stats |
| `GET` | `/stats` | Database and tag statistics |
| `GET` | `/pending` | Raw notes still pending processing |
| `GET` | `/failed` | Failed notes eligible for retry |
| `GET` | `/dlq` | Dead letter queue entries |
| `POST` | `/retry/{id}` | Re-process a failed raw note |

### POST /capture

```bash
curl -X POST http://127.0.0.1:8000/capture \
  -H "Content-Type: application/json" \
  -d '{"text": "python, ml: Implemented transformer for text classification"}'
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
```

Returns notes ranked by semantic similarity (L2 distance in embedding space).

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
| 3 | Offline sync | Background thread syncs JSONL → database every 4 hours |
| 4 | Retry | Failed notes can be retried via `/retry/{id}` or batch processor |
| 5 | Dead letter queue | After max retries → DLQ with full stack trace for manual review |
| 6 | Emergency backup | Database failure → `storage/emergency_backup.jsonl` |

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
| `QUICKCAPTURE_PARSE_WORKERS` | `4` | Max background parser threads in the Brain |
| `QUICKCAPTURE_LLM_TIMEOUT_SECONDS` | `240` | Per-attempt LLM parse timeout |
| `QUICKCAPTURE_LLM_TIMEOUT_ATTEMPTS` | `1` | Max parse attempts per entry |
| `QUICKCAPTURE_CAPTURE_TIMEOUT_SECONDS` | `300` | Launcher HTTP timeout for `/capture` |
| `QUICKCAPTURE_OLLAMA_KEEP_ALIVE` | *(none)* | Ollama model residency hint |
| `QUICKCAPTURE_OLLAMA_NUM_CTX` | *(none)* | Ollama context size override |
| `QUICKCAPTURE_OLLAMA_NUM_PREDICT` | *(none)* | Ollama output token cap |

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

### Batch processor

Designed for scheduled runs (e.g., cron):

```bash
python scripts/batch_processor.py               # Process everything
python scripts/batch_processor.py --pending-only # Only pending raw notes
python scripts/batch_processor.py --retry-only   # Only retry failed notes
python scripts/batch_processor.py --sync-only    # Only sync offline JSONL
```

## Testing

```bash
# Run all tests
pytest tests/

# With coverage report
pytest tests/ --cov=scripts --cov=observability --cov-report=html

# Specific suites
pytest tests/test_normalizer.py -v          # 99 normalization tests
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
│   ├── normalizer.py              # Deterministic normalization pipeline
│   ├── parse_input.py             # NeuralParser (Ollama LLM)
│   ├── validate_note.py           # Multi-dimensional validation
│   ├── storage_engine.py          # SQLite + FAISS hybrid storage
│   ├── models.py                  # Data models (ParsedNote, etc.)
│   ├── tag_intelligence.py        # Tag suggestions and drift detection
│   ├── batch_processor.py         # Scheduled batch processing
│   ├── review_outliers.py         # Outlier review CLI
│   ├── snr_preprocess.py          # SNR export preprocessing
│   ├── quick_add.py               # Legacy CLI entry point
│   ├── migrate_database.py        # Schema migration script
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
├── storage/                       # Runtime data (gitignored except schema)
│   ├── quickcapture.db            # SQLite database
│   ├── vector_store/              # FAISS index + ID map
│   ├── offline_notes.jsonl        # Offline capture fallback
│   └── emergency_backup.jsonl     # Emergency DB-failure fallback
│
├── logs/                          # Application logs
│   ├── brain.log
│   └── launcher.log
│
├── tests/                         # Test suite
│   ├── test_normalizer.py         # 99 normalization tests
│   ├── test_basic_functionality.py
│   ├── test_observability.py
│   ├── test_failed_notes.py
│   └── test_runtime_reliability.py
│
├── qc-documentation/              # Detailed system documentation (14 docs)
├── debugging/                     # Debug and diagnostic scripts
│
├── start_quickcapture.bat         # Windows startup script
├── pyproject.toml                 # Python project config
├── requirements.txt               # Dependencies
└── CHANGELOG.md                   # Version history
```

## Roadmap

Planned features (see `quicklauncher_query_roadmap.md` for details):

- **Launcher search mode** -- Toggle between capture and search in the overlay bar
- **NL-to-query endpoint** (`POST /ask`) -- Natural language questions answered via LLM-driven query routing or safe read-only SQL
- **In-launcher result display** -- Show search results and NL answers directly in the quick bar

## Tech Stack

| Layer | Technology |
|-------|------------|
| Language | Python 3.11+ |
| Backend | FastAPI, Uvicorn |
| Frontend | PyQt6, `keyboard` |
| LLM | Ollama (`qwen3:0.6b`) |
| Embeddings | SentenceTransformers (`all-MiniLM-L6-v2`) |
| Database | SQLite (WAL mode) |
| Vector store | FAISS (`IndexFlatL2`, 384-dim) |
| Metrics | Prometheus client |
| Logging | structlog |
| ML stack | PyTorch, scikit-learn |

---

**Author**: Alejandro Garay
**Version**: 1.0.0
**License**: MIT
