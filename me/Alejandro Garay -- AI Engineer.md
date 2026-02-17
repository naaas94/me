**Alejandro Garay**  
AI Engineer (NLP / LLM Systems)  
Buenos Aires, Argentina · alejandroa.garay.ag@gmail.com · +54 11 5660 5331  
[GitHub](https://github.com/naaas94) · [LinkedIn](https://www.linkedin.com/in/alejandro-garay-frontini/)

---

### Summary

AI/NLP Engineer with a linguistics background and a systems-first approach to LLM-based products. I build end-to-end: problem framing → retrieval/classification pipelines → serving → evaluation → observability. Focus areas are semantic search, RAG, agentic workflows, and text classification — with production patterns (CI/CD, reproducibility, structured logging, metrics) baked in from day one rather than bolted on after a demo.

---

### Featured Systems

**SnR QuickCapture — Local-First Note Capture with Neural Parsing**  
*Personal tooling / daily-driver system*

* Built a desktop capture system (global hotkey → FastAPI backend → Ollama LLM) that parses, embeds, and stores notes in a hybrid SQLite + FAISS vector database, running entirely on-device.
* Designed a deterministic normalizer that skips the LLM when confidence ≥ 0.7 (~30-40% of structured inputs), reducing latency and compute.
* Implemented a 6-layer reliability model (raw audit trail, offline JSONL fallback, background sync, retry queue, dead-letter queue, emergency backup) — zero-note-loss guarantee.
* Observability subsystem: Prometheus-compatible metrics, health scoring, performance tracking, semantic/performance drift detection. 99 normalization tests.

**Quality Intelligence (QI) — Personal Development CLI with LLM Synthesis**  
*Personal tooling / integrates with SnR QuickCapture*

* Built a CLI that ingests daily check-ins and SnR QC notes, classifies them into structured events via heuristics, engineers features over rolling time windows (means, deltas, streaks, trends), and generates weekly/monthly reports.
* LLM narrative synthesis via Ollama with Pydantic-validated output contracts; graceful degradation on LLM failure. Every LLM call traced in an observability table (timing, tokens, validation outcome).
* Reuses SnR QC's parsed metadata (tags, sentiment, entities) to avoid redundant LLM calls — decoupled integration via direct SQLite import.

**Retail Copilot — NL→SQL Architecture (GCP/Vertex AI)**  
*Architecture SOW + public local PoC*

* Authored a 35+ page architecture SOW converting natural language into validated SQL + VizSpec JSON over BigQuery, with a PoC→MVP→multi-tenant evolution path.
* Defined NL→intent→slots→template→validator chain with allowlists, tenant filters, dry-run bytes, and budget caps for safe-by-design query execution.
* Designed tenancy and blast-radius model: per-tenant datasets, logs, budgets, service accounts; platform-shared orchestrator with tenant-scoped execution (RLS/CLS, service-account impersonation).
* Built a public local PoC: modular monolith (core/interfaces/adapters/ui) with DuckDB simulating BigQuery + Gemini adapter, golden-set evaluation framework.

**Agentic Reviewer — Semantic Auditing for Text Classification**  
*Portfolio system / GDPR/CCPA domain*

* Built an LLM-assisted auditing system: unified multi-task agent (evaluate → propose → reason) in a single call, exposed via FastAPI REST interface with review, metrics, security, and drift endpoints.
* Production resilience patterns: circuit breaker, LRU caching with TTL, prompt-injection detection, input sanitization, rate limiting, SQLite audit trail.
* Configurable classification domain via YAML label definitions and Jinja2 prompt templates.

**PCC — Privacy Case Classifier (Live Production Pipeline)**  
*End-to-end ML pipeline / BigQuery + GCS*

* Built a production text classification pipeline for GDPR/CCPA compliance: daily BigQuery ingestion → MiniLM + TF-IDF embeddings (584-dim) → inference → BigQuery output with monitoring logs.
* Automatic model lifecycle: GCS model ingestion with version tracking, today's-model priority with latest-model fallback, model caching and reloading.
* Live production: two BigQuery tables running daily inference with a Looker monitoring dashboard. 95%+ confidence scores on synthetic data.

---

### Technical Skills

* **LLM/NLP:** Transformers, SentenceTransformers, embeddings, FAISS/HNSW, semantic search, RAG (dense + lexical), re-ranking, prompt/tool workflows, Ollama (local inference), classification, NL→SQL.
* **Backend/Serving:** FastAPI, REST, Pydantic, Docker, basic security guardrails (validation, rate/timeout control, prompt-injection detection).
* **Data/Orchestration:** BigQuery/GCS, SQLite (WAL), Flyte, Ray, Apache Beam, Spark; schema contracts and data validation.
* **MLOps/Quality:** GitHub Actions CI, experiment tracking, deterministic runs, offline eval harnesses, Prometheus metrics, structured logging, drift detection.
* **Languages & Tools:** Python, SQL, Docker, Git, Make, YAML/TOML config management.

---

### Experience

**Spotify — Data Scientist, Customer Experience & Privacy** | Sep 2024 – Jul 2025

* Designed and delivered a privacy-compliant NLP classification pipeline for GDPR/CCPA workflows; translated regulatory constraints into taxonomy, labeling strategy, and evaluation requirements. Full ownership from discovery to delivery.
* Built a semantic retrieval capability (transformer embeddings + FAISS) for internal knowledge search; defined indexing and freshness policies.
* Partnered with Legal & Ops to turn policy into system guarantees (labeling, redaction paths, safety thresholds).
* Prototyped a symbolic-LLM "agentic reviewer" for auditing model/agent behavior and generating traceable review artifacts.

**Spotify (via ModSquad) — Workforce Management Analyst (Data/Analytics)** | Jan 2022 – Sep 2024

* Owned analytical reporting and forecasting for global customer support operations (SLA tracking, capacity planning, performance monitoring). Built dashboards and operational tooling; collaborated cross-functionally with CX, Data, and Product teams.

**Invisible Technologies / MGM Linguistic Solutions — Asst. Project Manager (NLP Annotation & Linguistic QA)** | 2020 – 2022

* Managed NLP annotation workflows and linguistic QA for data labeling projects; enforced quality guidelines and review loops. Automated language processing tasks for data preparation and validation pipelines.

---

### Engineering Portfolio (GitHub)

* **SnR QuickCapture**: Local-first note capture with neural parsing, semantic search, and intelligent tagging. Stack: FastAPI, Ollama, SentenceTransformers, SQLite, FAISS, PyQt6, Prometheus.
* **QI**: Personal development CLI — daily check-ins, note imports, heuristic event classification, feature engineering, LLM narrative synthesis. Stack: Typer, Rich, Pydantic, SQLite, Ollama.
* **Retail Copilot**: NL→SQL architecture SOW (GCP/Vertex AI) + local PoC with modular monolith, golden-set eval. Stack: FastAPI, Gemini, DuckDB, Streamlit.
* **Agentic Reviewer**: Semantic auditing for text classification — unified LLM agent, REST API, security layer, audit logging. Stack: FastAPI, Ollama, SQLite, Pydantic.
* **PCC**: Privacy case classifier — live BigQuery pipeline with GCS model ingestion and Looker dashboard. Stack: BigQuery, MiniLM, scikit-learn, Docker, GCS.
* **MTP**: Model training pipeline — intelligent dataset monitoring, conditional retraining, GCS model registry, hyperparameter optimization. Stack: scikit-learn, GCS, Kubernetes.
* **SMA**: Model serving API with Prometheus metrics, Docker, LLM integration, CI/CD. Stack: FastAPI, Docker, GitHub Actions.

---

### Education

B.Sc. in Technical-Scientific Translation · B.Ed. in English Language Teaching — Lenguas Vivas "Sofía E. Broquen de Spangenberg".

---

### Certifications

IBM Data Science Professional Certificate · Google Data Analytics Professional Certificate · DataCamp (SQL, Python, NLP).
