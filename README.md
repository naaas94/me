# Alejandro Garay — Applied AI Engineer 

**I design, ship, and evaluate end‑to‑end AI systems** with a focus on NLP: retrieval‑augmented generation (RAG), agentic workflows, evaluation harnesses, and production patterns at individual scale. This repo is the **entry point** to my portfolio: code, diagrams, and evidence.

**Last updated:** 2025‑12-01

---

## At a glance

* **Primary vector:** AI/ML **Applied Engineer** (NLP | LLM-based Systems)
* **Differentiator:** symbolic‑linguistic rigor + production patterns (pipelines, orchestration, testing, monitoring) rather than demo‑only prototypes
* **Background:** Linguistics/Philosophy/Translation → AI/NLP (retrieval‑first, auditable systems)


---

## Hiring signals (pattern → evidence)

| Real‑world pattern                             | What I built                      | Evidence                                                             | First file to open                                                 |
| ---------------------------------------------- | --------------------------------- | -------------------------------------------------------------------- | ------------------------------------------------------------------ |
| Ingestion → Train → Evaluate → Serve → Monitor | **Model Training Pipeline (MTP)** | CI badge, MLflow runs, deterministic seeds, latency & cost snapshots | `mtp/README.md` → `examples/train_run_example.py`                        |
| RAG with retrieval quality gates               | **Retail Copilot**       | recall@k, precision@k, answer faithfulness, context‑utilization      | `retail-copilot/docs/SOW_Dossier.pdf`                    |
| Post‑hoc guardrails/verification               | **Agentic Reviewer**              | hallucination checks, citation verification, rubric‑based scoring    | `agentic-reviewer/README.md` → `demo.py (ipynb coming soon)`        |
| Productionized text classification             | **Privacy Case Classifier (PCC)** | F1/accuracy with data slices; confusion matrix; error analysis       | `pcc/README.md` → `notebooks/model_analysis.ipynb`                       |
| Observability for LLM APIs                     | **Simple Model API (SMA)**        | Prometheus metrics (RPS, P50/P95), structured logs, k6 load test     | `simple-model-api/README.md` → `Makefile`                     |
| Embedding analysis & migration                 | **Embedding Mapper**              | pairwise shifts, trust‑region plots, retrieval deltas                | `embedding-mapper/README.md` → `examples/compare_embeddings.ipynb` |
| NL→SQL with safety & tenancy                   | **Retail Copilot**                | Intent taxonomy, SQL templates, validation rules, golden‑set evals   | `retail-copilot/docs/dossier.pdf` → `catalog/intents.yaml`        |

---

## Architecture (portfolio map)

```mermaid
flowchart LR
  subgraph Data Layer
    SRC[Raw corpora / CSV / APIs]
    FEAT[Feature/Embedding Store]
  end

  subgraph Build & Train
    EDP[EDP - Enterprise Data Pipeline]
    MTP[MTP - Model Training Pipeline]
    REG[Model Registry]
  end

  subgraph Serving & Apps
    SMA[Simple Model API]
    RAG[RAG Service]
    PCC[PCC Classifier]
    AR[Agentic Reviewer]
  end

  subgraph Observability & Eval
    EVAL[Eval Harness]
    LOGS[Logs/Metrics: Prometheus/Grafana]
  end

  SRC --> EDP --> MTP --> REG --> SMA
  FEAT <--> RAG
  PCC --> SMA
  SMA --> LOGS
  RAG --> LOGS
  SMA --> AR
  RAG --> AR
  MTP --> EVAL
  RAG --> EVAL
```
---

## Evaluation (RAG/LLM + systems)

**Retrieval metrics**

* `recall@k`: fraction of queries where at least one gold doc is in top‑k.
* `precision@k`: fraction of top‑k that are relevant.
* `MRR@k`: mean reciprocal rank of first relevant doc within k.
* `nDCG@k`: graded relevance with position discount.

**Answer‑level metrics**

* **Faithfulness**: proportion of claims grounded in retrieved context (LLM‑ or rule‑based).
* **Context utilization**: % of answer tokens attributable to provided context.
* **Answer correctness**: labeling via gold answers or rubric (exact/partial match).

**Operational**

* **Latency** `P50/P95`, **throughput** `RPS`, **cost/request**, **timeouts/error rate**, and **SLO/error budget** (e.g., `P95 < 800ms`, monthly error budget 0.5%).


---

## Observability

* **Stack:** `prometheus`, `grafana`, `opencensus` (exporters), structured JSON logs.
* Dashboards: `/observability/grafana/provisioning/` (panels for latency, RPS, errors, cost).
* Alerts: `/observability/alerts/` (ex: high P95, elevated 5xx, drift detector fired).
* Screenshots for reviewers: `/docs/img/observability_*.png`.

Run locally:

```bash
make up-metrics   # Grafana on http://localhost:3000 (admin/admin locally)
```

---

## Featured projects

### 1) Retail Copilot PoC → RAG MVP 

* **Problem**: convert natural language queries into validated SQL + VizSpec JSON over BigQuery with safe multi‑tenant execution.
* **Pattern**: NL→intent→slots→template→validator chain; spec‑first architecture with PoC→MVP→multi‑tenant evolution path.
* **Implementation**: 35+ page architecture dossier; planner JSON, SQL templates, validation rules, tenant isolation (RLS/CLS, per‑tenant datasets/budgets), golden‑set eval, promotion gates.
* **Evidence**: Intent taxonomy, glossary, SQL policies, router/planner prompts, test scaffolds, runbook, monitoring dashboards, SLOs (latency, cost, faithfulness), canary rollout specs.
* **Start here**: `retail-copilot/docs/dossier.pdf` → `catalog/intents.yaml` → `prompts/planner-retail-v2.md`
* **Status**: Architecture/spec delivered — 2025
* **Repo**: PoC [repo](https://github.com/naaas94/retail-copilot-gcp) to back SOW

#### Architecture diagrams

**System context (at a glance)**

```mermaid
graph TD
  User -->|HTTPS| APIGW
  APIGW --> API
  API --> ORCH

  subgraph Vertex_AI [Vertex AI]
    LLM[Endpoint: Gemini Flash]
    PIPE[Vertex Pipelines]
    MATCH[Matching Engine]
  end

  subgraph Data [BigQuery & Catalog]
    BQ[(BigQuery)]
    AUDIT[(BQ Audit Logs)]
    CATALOG[Catalog: intents / templates / policies]
  end

  subgraph Guardrails
    VALID[SQL Validator]
    POLICY[Policy Engine]
  end

  subgraph Output
    ASM[Answer Assembler]
    VIZ[VizSpec JSON]
    LOOKER[(Looker optional)]
  end

  subgraph Obs [Observability]
    LOGS[Cloud Logging]
    MON[Cloud Monitoring]
    ERR[Error Reporting]
    TRACE[Cloud Trace]
  end

  ORCH -->|plan| LLM
  ORCH -->|load| CATALOG
  ORCH -->|validate| VALID
  VALID -->|safe SQL| BQ
  BQ --> ASM
  ASM --> VIZ --> LOOKER

  API --> LOGS
  ORCH --> LOGS
  LLM --> LOGS
  BQ --> AUDIT --> LOGS
  LOGS --> MON
  LOGS --> ERR
  LOGS --> TRACE

  ORCH -. synonyms .-> MATCH

  classDef node fill:#0b253a,stroke:#00ffff,stroke-width:1,color:#e6f8ff;
  class APIGW,API,ORCH,LLM,PIPE,MATCH,BQ,AUDIT,CATALOG,VALID,POLICY,ASM,VIZ,LOOKER,LOGS,MON,ERR,TRACE node;
```

**Data lineage (from NL to Viz)**

```mermaid
graph LR
  Q[NL query] --> I[Intent map + slots]
  I --> T[Template render]
  T --> G[Guardrails validate]
  G --> E[(BigQuery run)]
  E --> X[Explain block]
  E --> Z[VizSpec JSON]
  Z --> D[Dashboard or Looker]
  X --> U[User traceability]
```

### 2) PCC — Privacy Case Classifier

* **Problem**: classify privacy requests/cases into workflow buckets (GDPR/CCPA compliance).
* **Pattern**: supervised text classification + error analysis + data slices; policy‑to‑pipeline translation.
* **Implementation**: `scikit-learn`/Transformers (MiniLM), Flyte orchestration, BigQuery, Docker, MLflow runs.
* **Evidence**: macro‑F1 on fixtures; slice metrics (by entity/type); confusion matrix; regulatory constraint mapping.
* **Start here**: `pcc/notebooks/pcc_demo.ipynb`

### 3) MTP — Model Training Pipeline

* **Problem**: reproducible training with experiment tracking & registries.
* **Pattern**: `MLflow` + deterministic training + pinned deps + structured configs.
* **Implementation**: Makefile targets (`make train/eval/register`), CI, Docker multi‑stage.
* **Evidence**: MLflow artifacts, metrics tables, model in `registry/`.
* **Start here**: `mtp/examples/minimal_run.py`

### 4) SMA — Simple Model API

* **Problem**: serve models with SLAs and visibility.
* **Pattern**: FastAPI + Prometheus + k6 load test + JSON logs.
* **Implementation**: `docker-compose up`, `/metrics` endpoint, request IDs, middleware timing.
* **Evidence**: P50/P95 latency charts; RPS under load; error rates.
* **Start here**: `simple-model-api/README.md` → `make loadtest`

---

## More projects

* **RAG Service** — retrieval + rerank + prompt templates + eval harness (`recall@k`, faithfulness). Built for Spotify internal knowledge search with transformer embeddings + FAISS, freshness policies.
* **Agentic Reviewer** — post‑hoc auditing loop (citation checks, rubric scores, red‑team prompts). Symbolic‑LLM hybrid for systematic agent performance evaluation.
* **Embedding Mapper** — compare embedding models; visualize drift and retrieval deltas.
* **EDP (Enterprise Data Pipeline)** — ingestion/validation to dataset release with schema contracts. Stack: Apache Beam, Spark, Ray, Kafka.

---

## Operating standards (applies to all repos)

* [x] Pinned dependencies & lockfiles
* [x] Deterministic seeds & reproducible runs
* [x] Makefile targets for common actions
* [x] CI: lint (`ruff`), typecheck (`mypy`/`pyright`), tests (`pytest`)
* [x] Structured logs + metrics hooks
* [x] Minimal data fixtures checked into repo
* [x] Doc: Quickstart + E2E path ≤10 minutes

---

## About me

I come from **linguistics/philosophy/translation** (B.Sc. Technical‑Scientific Translation, B.Ed. English Language Teaching). I use that symbolic lens to design reliable NLP systems: clear problem framing, careful retrieval/representation choices, and evaluation you can trust.

Most recently at **Spotify** (Sep 2024 – Jul 2025) as Data Scientist on Customer Experience & Privacy, where I architected privacy‑compliant NLP pipelines, delivered semantic retrieval capabilities, and partnered with Legal/Ops to turn policy into system guarantees.

Currently based in **Buenos Aires, Argentina**.

* **GitHub:** [github.com/naaas94](https://github.com/naaas94)
* **LinkedIn:** [linkedin.com/in/alejandroa-garay](https://www.linkedin.com/in/alejandroa-garay/)
* **Email:** alejandroa.garay.ag@gmail.com
