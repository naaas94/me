# QI - Quality Intelligence

A local-first CLI system for personal development tracking and reporting, designed for 52-week compounding data accumulation.

## What QI Does

QI turns daily self-observations and ad-hoc notes into structured, computable signals and narrative reports. It captures daily metrics through a low-friction check-in, ingests notes from an external note-capture system (SnR QuickCapture), classifies them into structured events, engineers features over rolling time windows, and generates weekly/monthly reports with optional LLM-assisted narrative synthesis via Ollama.

**Core philosophy**: *Signal accumulation per unit friction* -- maximize insight yield while keeping daily effort under two minutes.

**What compounding looks like after 12-52 weeks**: 365 micro-observations, hundreds of classified events, 52 weekly retros with intervention tracking, 12 monthly dossiers. Enough data to answer "what conditions produce high output?", "what precedes avoidance spikes?", and "which interventions actually moved the needle?"

## Quick Start

```bash
# Install (editable, with dev tools)
pip install -e ".[dev]"

# Initialize QI (creates ~/.qi directory, database, config, principles file)
qi init

# First daily check-in
qi dci

# Quick check-in (energy, mood, sleep only)
qi dci --quick
```

## How the System Works

QI has four layers: **Capture**, **Processing**, **Reporting**, and **LLM Synthesis**. Each layer is independent -- the system produces useful output even without LLM, and even without note imports.

### Data Flow

```
                External System                           QI System
               ┌─────────────┐
               │  SnR QC DB  │──── qi import-snr-db ────┐
               │  (SQLite)   │                           │
               │             │──── qi import-snr ────────┤ (JSONL)
               └─────────────┘                           │
                                                         ▼
                                                  notes_imported
                                                         │
                                                    qi process
                                                   (heuristics)
                                                         │
               ┌─────────────┐                           ▼
               │   qi dci    │──────────────────►     events
               └─────────────┘                           │
                                                  Feature Engine
               ┌─────────────┐                  (means, deltas,
               │   qi week   │──────────────►    streaks, counts)
               └─────────────┘                           │
                                                         ▼
                                                  qi report weekly
                                                  qi report monthly
                                                         │
                                           ┌─────────────┴──────────────┐
                                           │                            │
                                    Deterministic              LLM Narrative
                                     Sections                   (Ollama)
                                           │                            │
                                           └─────────────┬──────────────┘
                                                         ▼
                                                     Artifact
                                              (markdown + metadata)
```

**Sync shortcut**: `qi report weekly --sync` chains import + process + report in one command.

### 1. Capture Layer

**Daily Check-In (DCI)** -- the primary data stream. A stepped interactive prompt that takes 30-120 seconds:

| Section | Fields | Required |
|---------|--------|----------|
| Core Metrics | energy, mood, sleep (0-10 float) | Yes |
| Focus & Reflection | primary_focus, one_win, one_friction | No |
| Training | training_done, cardio, gym, p2gh (bools) | No |
| Tokens & Flags | w (int), p/m/e (bools), compulsion_flag + trigger_tag | No |
| Carryover | residual items from previous days | No |

Quick mode (`qi dci --quick`) captures only the three core metrics.

**SnR QuickCapture Import** -- ingests notes from the companion [SnR QuickCapture](https://github.com/agaray/snr-quickcapture) system. QC is an intelligent note-capture tool with a global hotkey, local LLM parsing (tags, sentiment, entities, intent), and hybrid SQLite/vector storage. QI consumes these notes and reuses the metadata QC already extracted, avoiding redundant LLM calls.

Two import modes:
- `qi import-snr <file.jsonl>` -- from a JSONL export file
- `qi import-snr-db [path]` -- directly from the QC SQLite database (idempotent via `ON CONFLICT(snr_id)`)

**Weekly Retro** (`qi week`) -- a 15-25 minute structured retrospective: scoreboard, wins (up to 5), frictions (up to 5), root-cause hypothesis, one-change commitment (title + mechanism + measurement), minimums for next week, and previous commitment tracking.

### 2. Processing Layer

**Heuristic Event Classifier** (`qi process`) -- converts imported notes into structured events using a hybrid approach:

- Reuses SnR QC's LLM-extracted tags and sentiment (already paid for at capture time)
- Applies QI-specific keyword heuristics for event classification
- Not every note becomes an event -- only those matching detectable patterns

Event types: `win`, `friction`, `insight`, `compulsion`, `avoidance`
Domains: `health`, `career`, `social`, `cognition`, `nature`, `finance`

Keyword rules are configurable in `config/heuristics.yaml`.

**Feature Engineering** -- deterministic computation over any date window:

| Category | Features |
|----------|----------|
| Core | energy_mean, mood_mean, sleep_mean, mood_volatility |
| Training | training_days, cardio_days, gym_days, p2gh_days, training_streak |
| Behavioral | compulsion_rate, good_period_rate, top_triggers |
| Tokens | w_total, p_rate, m_rate, e_rate |
| Events | win_count, friction_count, insight_count, compulsion_event_count |
| Streaks | dci_streak, training_streak |
| Deltas | Week-over-week change for all numeric features |
| Trends | Linear regression slope (improving / stable / declining) |

Missing data strategy: skip gaps in calculations (no interpolation), reset streaks on gaps, require minimum 3 data points for rolling means.

### 3. Reporting Layer

**Weekly Digest** (`qi report weekly`) -- deterministic sections covering core metrics, training, behavioral tracking, events, tokens, week-over-week deltas, and streaks. Optionally appends an LLM narrative.

**Monthly Dossier** (`qi report monthly`) -- same deterministic sections plus trend analysis (linear regression on energy/mood/sleep) and a summary of weekly retros (wins, frictions, commitment tracking).

Both commands support:
- `--sync` -- auto-imports from QC DB + processes notes before generating the report
- `--no-llm` -- skips LLM narrative (deterministic-only output)
- `--date YYYY-MM-DD` -- target a specific week or month

Every report is saved as an **artifact** in the database with full input/feature snapshots, rendered markdown, and LLM metadata. This makes reports reproducible and auditable.

### 4. LLM Synthesis Layer (Optional)

When enabled, the LLM layer appends a narrative section to reports. It uses Ollama (local inference) and follows a strict contract:

**Flow**:
1. Build deterministic prompts from feature data + user's `principles.md` (guiding principles and Key Results)
2. Call Ollama chat API with JSON output format
3. Validate response against Pydantic schema (`NarrativeOutput`)
4. On validation failure: one retry with a repair prompt
5. On second failure: report is still produced without the narrative (graceful degradation)

**Narrative output contract** (validated via Pydantic):
- `weekly_summary` -- what happened (<=150 words)
- `delta_narrative` -- what changed and plausible causes
- `principle_alignment` -- per-principle status (on_track / slipping / no_data) with evidence
- `kr_progress` -- Key Results assessment
- `coaching_focus` -- one theme to focus on
- `next_experiment` -- one practical change with measurable outcome
- `risks` -- top failure modes
- `confidence` -- 0-1 self-assessed confidence

**Observability**: every LLM call is traced in the `llm_runs` table with timing (total/load/prompt_eval/eval duration in ms), token counts, validation outcome, full prompts, and raw output. Enables queries like per-model latency, validation failure rate, and token usage over time.

The principles file (`~/.qi/principles.md`) contains user-defined guiding principles and Key Results. It is injected into the LLM prompt to ground the narrative in your actual goals. Edit with `qi principles edit`.

## CLI Reference

| Command | Description |
|---------|-------------|
| `qi init` | Create `~/.qi` directory, database, config, principles file |
| `qi dci` | Interactive daily check-in (stepped prompts) |
| `qi dci --quick` | Quick DCI (energy, mood, sleep only) |
| `qi dci --date YYYY-MM-DD` | Backfill a DCI for a specific date |
| `qi import-snr <path>` | Import notes from SnR QC JSONL export |
| `qi import-snr-db [path]` | Import from QuickCapture SQLite DB |
| `qi import-snr-db --week YYYY-MM-DD` | Import week containing date |
| `qi import-snr-db --since 7d` | Import last N days |
| `qi process` | Run heuristic classifier on unprocessed notes |
| `qi week` | Interactive weekly retrospective |
| `qi stats` | Show trend statistics (last 7 days default) |
| `qi stats --tokens` | Include token aggregates |
| `qi stats --days 28` | Change analysis window |
| `qi report weekly` | Generate weekly digest |
| `qi report weekly --sync` | Import + process + generate weekly report |
| `qi report weekly --no-llm` | Skip LLM narrative |
| `qi report monthly` | Generate monthly dossier |
| `qi report monthly --sync` | Import + process + generate monthly report |
| `qi report monthly --no-llm` | Skip LLM narrative |
| `qi principles edit` | Open principles and KRs in `$EDITOR` |
| `qi export --format jsonl` | Export all data for backup |
| `qi version` | Show version |

## Data Storage

All data lives in `~/.qi/` (override with the `QI_HOME` environment variable):

| File | Purpose |
|------|---------|
| `qi.db` | SQLite database (WAL mode) |
| `config.toml` | User configuration |
| `principles.md` | Guiding principles and Key Results for LLM narrative |

**Database tables**:

| Table | Content |
|-------|---------|
| `dci` | Daily check-ins (one row per date, upsert on conflict) |
| `notes_imported` | Notes from SnR QC (idempotent via `snr_id` unique constraint) |
| `events` | Structured events classified from notes |
| `weekly_retro` | Weekly retrospectives (one per week_start) |
| `artifacts` | Generated reports with full input/feature/output snapshots |
| `llm_runs` | LLM call traces (timing, tokens, prompts, validation) |

Schema is version-tracked via a `schema_version` table and SQL migration files in `migrations/`.

## Configuration

Config file: `~/.qi/config.toml`

```toml
[general]
week_start_day = "monday"
timezone = "local"

[dci]
scale_max = 10.0
quick_mode_fields = ["energy", "mood", "sleep"]

[tokens]
w = "water/walks (int)"
p = "habit p (bool)"
m = "habit m (bool)"
e = "habit e (bool)"

[snr]
# Path to SnR QuickCapture database for import-snr-db and --sync
qc_db_path = "../snr-quickcapture/storage/quickcapture.db"

[llm]
enabled = true
model = "qwen3:8b"
base_url = "http://localhost:11434"
temperature = 0.4
timeout_seconds = 600   # 0 = no timeout (for slow local models)
principles_path = "principles.md"  # relative to QI_HOME
think = false           # set true only if model supports extended thinking
```

Set `llm.enabled = false` or omit the `[llm]` section entirely to run without LLM (deterministic-only reports).

## Project Structure

```
qi/
├── __init__.py              # Package + version
├── __main__.py              # python -m qi entrypoint
├── cli.py                   # Typer CLI with all commands
├── config.py                # Config loading from ~/.qi/config.toml
├── db.py                    # SQLite connection, migrations, all CRUD
├── models.py                # Pydantic models (DCI, ImportedNote, Event, etc.)
├── capture/
│   ├── dci.py               # Interactive DCI prompt (Rich TUI)
│   ├── snr_import.py        # JSONL import from SnR QC
│   ├── snr_db_import.py     # Direct SQLite import from QC database
│   └── weekly.py            # Weekly retro interactive prompt
├── processing/
│   ├── heuristics.py        # Keyword/tag event classifier
│   ├── events.py            # Note -> event conversion
│   └── features.py          # Rolling stats, deltas, streaks, trends
├── reporting/
│   ├── weekly.py            # Weekly digest builder
│   ├── monthly.py           # Monthly dossier builder
│   └── render.py            # Markdown section renderers
├── llm/
│   ├── client.py            # Ollama HTTP client (httpx)
│   ├── prompts.py           # Versioned prompt builder
│   ├── validate.py          # Pydantic output validation + repair retry
│   ├── synthesis.py         # High-level orchestration + observability
│   └── render.py            # Narrative markdown renderer
└── utils/
    └── time.py              # Week/month bounds, date helpers

schemas/                     # JSON schemas for data contracts
config/
├── heuristics.yaml          # Event classification keyword rules
└── defaults.yaml            # Default values for features/reports
migrations/
├── 001_initial.sql          # Core tables
├── 002_artifacts_llm_metadata.sql  # prompt_version + model_id on artifacts
└── 003_llm_runs.sql         # LLM observability traces table
tests/
```

## SnR QuickCapture Integration

QI is designed to work alongside [SnR QuickCapture](https://github.com/agaray/snr-quickcapture), a separate note-capture system with:
- Global hotkey (`Alt+Space`) for instant thought capture
- Local LLM parsing via Ollama (tags, sentiment, entities, intent, action items)
- Hybrid SQLite + FAISS vector storage

QI reuses QC's parsed metadata rather than re-running LLM inference on notes. The integration is decoupled: QI reads from QC's database (or JSONL export) and stores a copy in its own `notes_imported` table. QI works standalone without QC -- the DCI is an independent capture stream.

## Desktop Shortcut (Windows)

```powershell
# Create a "QI Daily Check-In" desktop shortcut
powershell -ExecutionPolicy Bypass -File create_desktop_shortcut.ps1
```

Double-click to open a terminal and run `qi dci` interactively.

## Development

```bash
# Install with dev dependencies
pip install -e ".[dev]"

# Run tests
pytest tests/ -v

# Linting
ruff check qi/
mypy qi/
```

**Tech stack**: Python 3.11+, Typer + Rich (CLI/TUI), Pydantic v2 (validation), SQLite (storage), httpx (Ollama API), TOML (config).

## Roadmap

Planned improvements (not yet implemented):

- **Text relevance pipeline**: EOD batch using a small model to assess note/DCI relevance against principles and KRs, producing per-item digests for richer report evidence
- **Time-series context for LLM**: Pass daily arrays (energy, mood, sleep by day) to the report prompt so the model can reason about temporal patterns and correlations
- **Model tiering**: Small model for daily relevance processing, larger model for report synthesis
- **Scheduling**: External scheduler (cron / Task Scheduler) integration for automated EOD processing and report generation
- **GUI/TUI selector**: Menu-driven interface for CLI commands
- **Compiled executable**: Standalone packaged binary for DCI and other frequent commands

## License

MIT License

---

*QI - Quality Intelligence for personal development tracking*
