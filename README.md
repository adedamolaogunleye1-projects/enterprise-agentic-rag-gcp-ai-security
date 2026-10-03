# Enterprise Agentic RAG (Scalable Pipeline)

A production-grade, enterprise-level scalable RAG system built with **LangGraph**, **Google Cloud Platform (GCP)**, and a **Portkey LLM Gateway**. The system distinguishes between technical "True Data" and random "Noisy Data" using semantic re-ranking, history-aware planning, and NeMo Guardrails for input/output safety.

## Key Features

- **Agentic Intelligence**: LangGraph for cyclic reasoning, multi-step planning, and conversation memory.
- **Guardrails**: NeMo Guardrails gate blocks off-topic, jailbreak, and injection inputs before any retrieval.
- **LLM Gateway**: Portkey routes all LLM calls with automatic fallback between primary and backup Groq keys.
- **Enterprise Search**: Qdrant Cloud for high-performance vector search + FlashRank for local semantic reranking.
- **Observability**: Full trace nesting with **Pydantic Logfire** and **LangSmith** across every agent node.
- **Evaluation Suite**: RAGAS-powered eval pipeline (6 metrics) with a dedicated Streamlit demo app.
- **Scalable Infrastructure**: Deployed on **Google Cloud Run** with Cloud Build CI/CD and VPC Connectors.

---

## Agent Intelligence Flow

```mermaid
graph TD
    User((User)) --> UI[Streamlit UI]
    UI --> API[FastAPI /query]
    API --> Guard{NeMo Guardrails}
    Guard -->|Blocked| UI
    Guard -->|Pass| Planner{Planner Node}
    Planner -->|Conversational| Responder[Responder Node]
    Planner -->|Technical| Retriever[Retriever Node]
    Retriever --> Reranker[FlashRank Local Reranker]
    Reranker --> Responder
    Responder --> UI
    Responder -.-> Memory[(LangGraph MemorySaver)]
```

---

## Project Structure

```text
├── app/
│   ├── agents/
│   │   ├── graph.py              # LangGraph graph + PostgresSaver checkpointer
│   │   ├── state.py              # AgentState schema
│   │   └── nodes/
│   │       ├── planner.py        # Intent classification node
│   │       ├── retriever.py      # Qdrant search + FlashRank reranker node
│   │       └── responder.py      # Answer generation node
│   ├── gateway/
│   │   └── client.py             # Portkey LLM gateway — primary + fallback routing
│   ├── guardrails/
│   │   ├── rails.py              # NeMo Guardrails integration
│   │   └── colang_rules.py       # Block/allow rule definitions
│   ├── ingestion/
│   │   ├── processor.py          # Dual-mode: CLI bulk load + Eventarc webhook (POST /ingest)
│   │   ├── chunking/
│   │   │   └── splitter.py       # Text splitting strategies
│   │   └── loaders/
│   │       ├── pdf.py            # Google Document AI PDF parser
│   │       ├── html.py           # HTML parser
│   │       ├── office.py         # DOCX / PPTX parser
│   │       └── text.py           # Plain text parser
│   ├── services/
│   │   ├── gcp/
│   │   │   ├── database_service.py      # psycopg3 connection pool (unix socket)
│   │   │   └── redis_semantic_cache.py  # Cosine-distance semantic cache
│   │   └── retrieval/
│   │       ├── embedding.py      # Vertex AI text-embedding-004 (lazy-loaded)
│   │       ├── qdrant_service.py # Vector search client
│   │       └── ranking_service.py # FlashRank reranker
│   ├── config.py                 # Centralized env var management
│   └── main.py                   # FastAPI entrypoint — two gates + /query
│
├── evals/
│   ├── app.py                    # Streamlit 4-tab eval dashboard
│   ├── pipeline.py               # Phase 1 — live /query calls + Groq summarization
│   ├── metrics.py                # Phase 2 — RAGAS scoring with GoogleEmbeddings
│   ├── guardrails_eval.py        # Guardrails TP/TN/FP/FN classification
│   ├── store.py                  # GCS persistence for eval history
│   ├── data_parser.py            # Golden dataset document parser
│   └── golden_dataset.json       # 15 RAG samples + 6 guardrail test cases
│
├── ui/
│   └── app.py                    # Streamlit chat interface
│
├── docker/
│   ├── backend.Dockerfile        # FastAPI + LangGraph + Guardrails + Redis + Postgres
│   ├── ui.Dockerfile             # Streamlit only (4 packages)
│   ├── ingestion.Dockerfile      # DocAI + Qdrant + parsers
│   └── evals.Dockerfile          # RAGAS + Vertex AI + Streamlit
│
├── terraform/
│   ├── main.tf                   # VPC, GCS buckets, Redis, Eventarc SA IAM
│   ├── cloud_run.tf              # All 4 Cloud Run services + public IAM
│   ├── database.tf               # Cloud SQL Postgres 15
│   ├── ingestion.tf              # Ingestion service + Eventarc trigger (POST /ingest)
│   ├── variables.tf              # Input variable declarations
│   ├── provider.tf               # GCP + hashicorp/time providers
│   └── output.tf                 # backend_url, ui_url, evals_url, ingestion_url
│
├── notebooks/
│   ├── 01_guardrails.ipynb       # NeMo Guardrails walkthrough
│   ├── 02_llm_gateway.ipynb      # Portkey gateway exploration
│   └── 03_evals.ipynb            # RAGAS metrics walkthrough
│
├── DATA/
│   └── true_data/                # Golden documents (Kubernetes, Databricks)
│
├── DOCS/                         # 24 architectural and operational guides
├── cloudbuild.yaml               # Parallel build of all 4 Docker images
├── cloudbuild-evals.yaml         # Targeted evals-only rebuild
├── requirements.txt              # Monolith / local dev dependencies
├── requirements-backend.txt      # Backend service dependencies
├── requirements-evals.txt        # Evals service dependencies
├── requirements-ingestion.txt    # Ingestion service dependencies
└── requirements-ui.txt           # UI service dependencies (4 packages)
```

---

## Tech Stack

| Layer | Technology |
|-------|-----------|
| Orchestration | LangChain + LangGraph |
| LLMs | Groq (Llama 3.3 70B) via **Portkey** gateway |
| Guardrails | NeMo Guardrails |
| Vector DB | Qdrant Cloud |
| Reranking | FlashRank (local, zero-latency) |
| Embeddings | HuggingFace sentence-transformers |
| Cloud Compute | Google Cloud Run (Serverless) |
| Cloud Storage | Google Cloud Storage (GCS) |
| Document Parsing | Google Document AI (PDF) |
| Observability | Pydantic Logfire + LangSmith |
| Evaluation | RAGAS + custom Tool Correctness (Jaccard) |

---

## Getting Started

### 1. Install dependencies

```powershell
python -m venv tenvv
.\tenvv\Scripts\activate
pip install -r requirements.txt
```

### 2. Configure environment

Create a `.env` file — see `commands.md` Section 2 for all required keys (Groq, Qdrant, GCP, Portkey, Logfire, LangSmith, JUDGE_GROQ for evals).

### 3. Run data ingestion

```powershell
python -m app.ingestion.processor DATA --wipe
```

### 4. Launch the app

```powershell
# Terminal 1 — FastAPI backend
uvicorn app.main:app --reload --port 8000

# Terminal 2 — Streamlit UI
streamlit run ui/app.py
```

### 5. Run the eval suite (optional)

```powershell
# Requires the FastAPI backend running on :8000
streamlit run evals/app.py
```

---

## Documentation Index

| # | Guide | What it covers |
|---|-------|---------------|
| 1 | [System Overview](DOCS/01_SYSTEM_OVERVIEW.md) | High-level vision and end-to-end flow |
| 2 | [Ingestion Engine](DOCS/02_INGESTION_ENGINE.md) | Document parsing and indexing pipeline |
| 3 | [Node Intelligence](DOCS/03_NODE_INTELLIGENCE.md) | Planner, Retriever, Responder internals |
| 4 | [Observability](DOCS/04_TRACING_AND_OBSERVABILITY.md) | Logfire + LangSmith tracing |
| 5 | [GCP Prod Setup](DOCS/05_GCP_PROD_SETUP.md) | Step-by-step infrastructure provisioning |
| 6 | [Deployment Strategy](DOCS/06_DEPLOYMENT_STRATEGY.md) | Cloud Build and Cloud Run details |
| 7 | [Env Variables](DOCS/07_ENVIRONMENT_VARIABLES.md) | Complete configuration dictionary |
| 8 | [GCP Roles & Services](DOCS/08_GCP_ROLES_AND_SERVICES.md) | IAM and service breakdown |
| 9 | [Infra Architecture](DOCS/09_INFRA_ARCHITECTURE.md) | The 3-tier cloud blueprint |
| 10 | [Redis Caching](DOCS/10_REDIS_CACHING.md) | Response caching layer design |
| 11 | [Microservices Transition](DOCS/11_MICROSERVICES_TRANSITION.md) | Scaling beyond monolith |
| 12 | [Known Gotchas](DOCS/12_KNOWN_GOTCHAS.md) | GCP quirks and architectural decisions |
| 13 | [FlashRank Reranking](DOCS/13_FLASHRANK_RERANKING.md) | Local semantic reranker deep-dive |
| 14 | [VPC Networking](DOCS/14_VPC_NETWORKING.md) | Private networking and VPC connectors |
| 15 | [Guardrails](DOCS/15_GUARDRAILS.md) | NeMo Guardrails implementation |
| 16 | [LLM Gateway](DOCS/16_LLM_GATEWAY.md) | Portkey routing, fallback, and observability |
| 17 | [Evals](DOCS/17_EVALS.md) | RAGAS metrics theory and token budget |
| 18 | [Evals Pipeline](DOCS/18_EVALS_PIPELINE.md) | Live eval pipeline and Streamlit demo |

---

*Built for High-Scale Enterprise Document Intelligence.*
