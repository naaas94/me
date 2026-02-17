**Alejandro Garay**  
AI Solutions Architect  
Buenos Aires, Argentina · alejandroa.garay.ag@gmail.com · +54 11 5660 5331  
[GitHub](https://github.com/naaas94) · [LinkedIn](https://www.linkedin.com/in/alejandro-garay-frontini/)

---

### Summary

AI/NLP Solutions Architect with a linguistics background who turns ambiguous business and policy language into auditable AI services. I design end-to-end system architectures — from problem framing through retrieval/classification pattern selection to evaluation surfaces and operational contracts — centered on LLM-based systems, information retrieval, NL→SQL, agentic workflows, and semantic search. I deliver architecture specs hardened for production handoff, backed by working PoCs that validate the critical path, with evaluation harnesses and safety/latency/cost guardrails.

---

### Flagship Architecture Work

**Retail Copilot NL→SQL Architecture (PoC→MVP→Multi-tenant, GCP/Vertex)**  
*Independent AI Solutions Architect (Challenge project, architecture/specification)*

* Authored a 35+ page architecture SOW for a GCP-based Retail Copilot, converting natural language into validated SQL + VizSpec JSON over BigQuery, outlining a clear PoC→MVP→multi-tenant evolution path (blueprint/spec, not deployed).
* Defined an NL→intent→slots→template→validator chain with allowlists, tenant filters, dry-run bytes, and budget caps, ensuring safe-by-design query execution (incl. example planner JSON, SQL templates, and validation rules).
* Designed tenancy and blast-radius model from day 1: per-tenant datasets, logs, budgets, and service accounts; platform-shared orchestrator with tenant-scoped execution via service-account impersonation and RLS/CLS.
* Specified MLOps & evaluation surfaces: golden-set replay, structural checks, promotion gates, canary rollout, SLOs (latency, cost, faithfulness), rollback policies, and concrete evaluation metrics (SQL correctness, execution accuracy, robustness, faithfulness, contradiction rate).
* Built a public local PoC: modular monolith (core/interfaces/adapters/ui) with DuckDB simulating BigQuery + Gemini adapter, golden-set evaluation framework, routing/safety/multi-tenancy validation.

**SnR QuickCapture + QI — Local-First LLM Tooling Ecosystem**  
*System architecture & implementation*

* Architected a two-system ecosystem: SnR QuickCapture (desktop capture → neural parsing → semantic storage) feeding into Quality Intelligence (event classification → feature engineering → LLM-assisted reporting) — fully local, no cloud dependencies for core operation.
* Designed QuickCapture's processing architecture: deterministic normalizer that bypasses the LLM when confidence ≥ 0.7, reducing latency and compute for ~30-40% of inputs; Ollama neural parser for the rest; hybrid SQLite (WAL) + FAISS (384-dim) storage with atomic writes.
* Specified a 6-layer reliability model (raw audit trail → offline JSONL fallback → background sync → retry queue → dead-letter queue → emergency backup) ensuring zero note loss across network, database, and LLM failure modes.
* Designed QI's LLM synthesis contract: Pydantic-validated output schema with retry-repair flow and graceful degradation; every LLM call traced in an observability table (timing, tokens, validation outcome) for auditable narrative generation.

**Agentic Reviewer — Semantic Auditing Architecture for Classification Systems**  
*System architecture & implementation*

* Designed a unified multi-task LLM agent (evaluate → propose → reason in a single call) for semantic auditing of text classification predictions, with FastAPI REST interface exposing review, metrics, security, and drift endpoints.
* Specified production resilience patterns: circuit breaker for LLM failures, LRU caching with TTL, prompt-injection detection and input sanitization, rate limiting, and SQLite audit trail for compliance.
* Architecture designed for staged RAG enhancement: FAISS/ChromaDB vector store integration, policy document retrieval for grounded reasoning, citation support — infrastructure ready, implementation planned.

---

### Technical Skills

* **Architectural patterns:** Hybrid RAG (BM25 + dense), re-ranking (cross-encoders), NL→SQL with guardrails, agentic workflows, context assembly, verification/attribution, offline/online eval (nDCG, faithfulness), data & index versioning, freshness policies, drift/safety filters, multi-tenancy isolation.
* **Languages & Infra:** Python, SQL, Docker, GitHub Actions, Ray, Flyte, Apache Beam, Spark, BigQuery/GCS, Kubernetes.
* **NLP/LLM:** Transformers/SentenceTransformers, embeddings & FAISS/HNSW, Ollama (local inference), FastAPI services, classification, semantic search, prompt engineering, tool/agent orchestration.
* **Quality & Ops:** CI/CD pipelines, schema validation (Pydantic), config management (YAML/TOML/Hydra), experiment tracking, Prometheus metrics, structured logging, drift detection, observability hooks.

---

### Experience

**Spotify — Data Scientist, Customer Experience & Privacy** | Sep 2024 – Jul 2025

* Architected and delivered a privacy-compliant NLP classification pipeline for GDPR/CCPA workflows; translated regulatory constraints into taxonomy, labeling strategy, and evaluation requirements. Full ownership from discovery to delivery.
* Delivered a semantic retrieval capability (transformer embeddings + FAISS) for internal knowledge search; defined indexing and freshness policies.
* Partnered with Legal & Ops to turn policy into system guarantees (labeling, redaction paths, safety thresholds).
* Prototyped a symbolic-LLM "agentic reviewer" for systematic agent performance evaluation and auditability at scale.

**Spotify (via ModSquad) — Workforce Management Analyst (Data/Analytics)** | Jan 2022 – Sep 2024

* Owned analytical reporting and forecasting for global customer support operations (SLA tracking, capacity planning, performance monitoring). Built dashboards and operational tooling; collaborated cross-functionally with CX, Data, and Product teams.

**Invisible Technologies / MGM Linguistic Solutions — Asst. Project Manager (NLP Annotation & Linguistic QA)** | 2020 – 2022

* Managed NLP annotation workflows and linguistic QA for data labeling projects; enforced quality guidelines and review loops. Automated language processing tasks for data preparation and validation pipelines.

---

### Engineering Portfolio (GitHub)

* **SnR QuickCapture**: Local-first note capture with neural parsing, semantic search, and hybrid SQLite + FAISS storage. Stack: FastAPI, Ollama, SentenceTransformers, FAISS, PyQt6, Prometheus.
* **QI**: Personal development CLI — event classification, feature engineering, LLM narrative synthesis with Pydantic-validated contracts. Stack: Typer, Rich, Pydantic, SQLite, Ollama.
* **Retail Copilot**: NL→SQL architecture SOW (GCP/Vertex AI) + local PoC with modular monolith, golden-set eval. Stack: FastAPI, Gemini, DuckDB, Streamlit.
* **Agentic Reviewer**: Semantic auditing for text classification — unified LLM agent, REST API, security layer, audit logging. Stack: FastAPI, Ollama, SQLite, Pydantic.
* **PCC**: Privacy case classifier — live BigQuery pipeline with GCS model ingestion and Looker monitoring dashboard. Stack: BigQuery, MiniLM, scikit-learn, Docker, GCS.
* **MTP**: Model training pipeline — intelligent dataset monitoring, conditional retraining, GCS model registry, hyperparameter optimization. Stack: scikit-learn, GCS, Kubernetes.
* **SMA**: Model serving API with Prometheus metrics, Docker, LLM integration, CI/CD. Stack: FastAPI, Docker, GitHub Actions.

---

### Education

B.Sc. in Technical-Scientific Translation · B.Ed. in English Language Teaching — Lenguas Vivas "Sofía E. Broquen de Spangenberg".

---

### Certifications

IBM Data Science Professional Certificate · Google Data Analytics Professional Certificate · DataCamp (SQL, Python, NLP).
