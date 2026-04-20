# Alejandro Garay — Applied AI Engineer

NLP · RAG · Agentic Systems · Document Intelligence

I design, build, and evaluate end-to-end LLM systems — retrieval-augmented generation, multi-agent orchestration, schema-bound document extraction, semantic search, classification pipelines. Production patterns (Pydantic contracts, reproducibility, DLQs, structured logging, metrics, auditable runs) built in from the first commit rather than retrofitted.

Linguistics background applied to meaning-critical NLP: careful problem framing, representation choices that hold under ambiguity, and evaluation you can trust. The edge is functional, not biographical.

Based in Buenos Aires. Open to Applied AI Engineer roles.

---

## Current systems

Two headline systems. Both are **personal projects implementing production patterns** — learning implementations, not enterprise deployments. The patterns are production-grade; the scope is portfolio.

### ARIA — Regulatory Impact & Compliance Agent

*GraphRAG + stateful multi-agent orchestration*

Multi-agent architecture (supervisor delegating to specialised ingestion, entity-extraction, graph-builder, and impact-analyser agents) over a custom stateful graph engine, with a LangGraph reference implementation maintained alongside for comparison. Hybrid retrieval — ChromaDB vector anchors → Neo4j graph expansion → fusion / reranking — for grounded multi-hop compliance queries with full source tracing.

FastAPI surface with MCP tool access and A2A agent delegation. Hardening: API-key + bearer auth, A2A shared secret, body-size middleware, Pydantic v2 strict contracts with schema versioning, production-mode `/docs` disable. Layered evaluation — unit, integration, golden-set YAML (retrieval / trace / contract / edge / security), E2E, security audit, agent-trajectory analysis, JSONL eval store. CI matrix on Py 3.12 / 3.13 plus nightly live-stack runs.

→ [`aria/`](./aria/README.md)

### Hermes — Schema-Bound LLM Document Extraction

*Reproducible, recoverable LLM document extraction — local or cloud.*

Two stances separate Hermes from the crowded field of LLM-extraction wrappers: every run is replayable from a content-addressed contract, and every failure has a defined path back to completion. Converts messy Excel, text-layer PDF, and scanned PDF into validated Pydantic-schema JSON; runs fully offline via Ollama or against any LiteLLM-supported provider (OpenAI, Anthropic, Google) with a single config change.

Reproducibility by construction: every run is bound to a content-addressed `contract_id` (canonical schema text + prompt-version SHA); every LLM call and result is tagged. Any output is replayable months later against a different model. Recovery by design: validation-failure repair loop with error-context re-prompting; dead-letter queue for unrecoverable chunks; `hermes retry` to resume DLQ items (optionally against a different model). Jobs promote to completed only when all chunks resolve and the DLQ is clear.

Optimization underneath: streaming I/O (openpyxl read-only in 50-row batches; PDF pages released one at a time), bounded `ThreadPoolExecutor` for cloud providers (sequential for local), per-worker SQLite connections in WAL mode, content-hash job deduplication, `--resume` after crash or interrupt without re-normalization.

→ [`hermes/`](./hermes/README.md)

---

## Other portfolio work

| Project | What it demonstrates | Open first |
| --- | --- | --- |
| **SnR QuickCapture** | Local-first neural parsing · hybrid SQLite + FAISS · deterministic routing at confidence ≥ 0.7 · 6-layer reliability stack with zero-note-loss guarantee · drift detection · 99 normalization tests | [`snr-qc/README.md`](./snr-qc/README.md) |
| **PCC — Privacy Case Classifier** | Live BigQuery inference · MiniLM + TF-IDF (584-dim) · GCS model lifecycle with version tracking and today-priority fallback · Looker monitoring dashboard | [`pcc/README.md`](./pcc/README.md) |
| **Retail Copilot** | 35+ page GCP / Vertex AI architecture SOW · NL → intent → slots → template → validator chain · tenant isolation model (RLS / CLS, per-tenant service accounts) · public DuckDB + Gemini PoC | [`retail-copilot/README.md`](./retail-copilot/README.md) |
| **Agentic Reviewer** | Unified multi-task agent (evaluate → propose → reason) · circuit breaker · LRU + TTL caching · prompt-injection detection · SQLite audit trail | [`agentic-reviewer/README.md`](./agentic-reviewer/README.md) |
| **QI — Quality Intelligence** | Personal evolution of a Spotify CX initiative · rolling-window feature engineering · dual-model LLM tier · Pydantic-validated outputs with repair retry · `llm_runs` observability table | [`qi/README.md`](./qi/README.md) |

---

## How I work

**End-to-end ownership.** Problem framing → system design → build → deploy → evaluate → stakeholder iteration. The evaluation isn't a downstream task someone else picks up — it's part of the system, versioned alongside the code.

**Build for failure from commit one.** Every system I build starts with two questions — where will this quietly degrade, and what happens when it does? That posture is visible in the code from day one: Pydantic v2 strict contracts with schema versioning, content-addressed idempotency, dead-letter queues with resume semantics, structured logging (`structlog`), Prometheus metrics, GitHub Actions CI with matrix + nightly live-stack runs. Hermes's DLQ + validation-repair loop, SnR QuickCapture's 6-layer reliability stack, and ARIA's layered eval suite are three different cashings-out of the same discipline.

**Abstraction as a working tool, not a deliverable.** I work on a concrete slice with rigor while holding the full system, its neighbors, and its failure surface in mind — moving up, down, and sideways as the problem demands. ARIA is where this shows most clearly: a custom stateful orchestration engine implemented from scratch *and* a LangGraph reference implementation maintained alongside for comparison — the same problem attacked at two levels of abstraction so the trade-offs become legible.

**Linguistic lens on meaning-critical NLP.** A linguistics background gives me two working habits: spotting ambiguity in language before it becomes a silent failure mode in semantic spaces or LLM outputs, and translating complex problem spaces fluently between technical and non-technical stakeholders. The first shows up in how I design taxonomies, write eval cases, and catch the subtle degradation modes that generic LLM pipelines gloss over; the second in how I turn regulatory or product constraints into system designs that actually ship.

---

## Background

**Spotify · Data Scientist, Customer Experience & Privacy · Sep 2024 – Jul 2025**
Privacy-compliant NLP classification for GDPR / CCPA workflows, owning the full loop from taxonomy design to evaluation. Transformer-embedding + FAISS semantic retrieval for internal knowledge. Partnered with Legal and Ops to translate regulation into system constraints. Built a symbolic-LLM review layer for auditing production CX classifiers.

Also conceived and delivered the design for **Quality Intelligence (QI)** — a CX working-group initiative evolving Quality Assurance into Quality Intelligence for 4–5k support agents globally. Gemini-backed synthesis over BigQuery-native records, 30-day agent windows into weekly KPI deltas with rolling-context narrative reports, prompt / logic version tracking, planned HITL validation.

**Spotify (via ModSquad) · Workforce Management Analyst · Jan 2022 – Sep 2024**
Forecasting, capacity planning, analytical reporting, and operational tooling for global CX.

---

## Contact

**GitHub:** [naaas94](https://github.com/naaas94)
**LinkedIn:** [alejandro-garay-frontini](https://www.linkedin.com/in/alejandro-garay-frontini/)
**Email:** alejandroa.garay.ag@gmail.com
