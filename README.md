# Multi-Agent Financial Research Assistant

An intelligent, agentic RAG system that answers complex financial questions by retrieving and analyzing SEC filings through four specialized agents. Features a dedicated verification layer for answer groundedness, RAGAS-based evaluation for reliability, and AWS deployment.

> **Resume line (fill in your numbers after running eval):**
> *Built an agentic RAG system (LangGraph) performing retrieval over [N] SEC filings with a dedicated
> verifier agent for answer groundedness; developed a RAGAS evaluation harness that measured faithfulness
> and cut hallucination rate from [X]% → [Y]%; deployed on AWS (EC2, ECR, S3, Terraform, GitHub Actions CI/CD).*

---

## Architecture

```
User Query
    │
    ▼
┌─────────────┐     decomposes     ┌──────────────────────────┐
│   Planner   │ ─────────────────► │  sub-queries [ 2–4 ]     │
└─────────────┘                    └──────────┬───────────────┘
                                              │
                                              ▼
                                   ┌─────────────────────┐
                                   │  Retriever Agent    │  ◄─ FAISS / Chroma vector store
                                   │  (MMR RAG, k=6)     │     sentence-transformers embeddings
                                   └──────────┬──────────┘
                                              │ retrieved docs
                                              ▼
                                   ┌─────────────────────┐
                                   │  Analyst Agent      │  ◄─ yfinance live market data
                                   │  (synthesis + calc) │
                                   └──────────┬──────────┘
                                              │ draft answer
                                              ▼
                                   ┌─────────────────────┐
                                   │  Verifier / Critic  │  ← THE RELIABILITY STAR
                                   │  (groundedness check│    checks every claim vs sources
                                   │   + hallucination   │    PASS → finalise
                                   │   detection)        │    FAIL → re-route to analyst (×1)
                                   └──────────┬──────────┘
                                              │
                                              ▼
                                        Final Answer
                               (with verification badge + source citations)
```

## Stack

| Layer | Technology |
|---|---|
| Agent orchestration | **LangGraph** (StateGraph with conditional edges) |
| RAG | LangChain + **FAISS** (or Chroma) + `sentence-transformers` |
| LLM | **Ollama / llama3.1** (local, zero cost) · OpenAI · Anthropic |
| Financial data | SEC EDGAR filings (free) · **yfinance** (live prices) |
| Backend | **FastAPI** + Uvicorn |
| Frontend | **Streamlit** + Plotly |
| Evaluation | **RAGAS** (faithfulness, relevancy, precision, recall) + LLM-as-judge |
| Deployment | **Docker** · **AWS** (EC2, ECR, S3) · **Terraform** IaC |
| CI/CD | **GitHub Actions** (test → build → push ECR → deploy via SSM) |

---

## Quick Start (local)

### 1. Prerequisites
- Python 3.11+
- [Ollama](https://ollama.ai) installed and running: `ollama pull llama3.1`

### 2. Install

```bash
git clone <your-repo>
cd financial-research-assistant
python -m venv .venv
.venv\Scripts\activate        # Windows
# source .venv/bin/activate   # Mac/Linux
pip install -r requirements.txt
cp .env.example .env          # edit if needed
```

### 3. Ingest SEC filings

```bash
python scripts/ingest.py --tickers AAPL MSFT TSLA --filings 10-K 10-Q --num 4
```

This downloads filings from SEC EDGAR, chunks them, and builds the FAISS vector store under `data/vectorstore/`.

### 4. Start the API

```bash
uvicorn src.api.main:app --reload --port 8000
```

### 5. Start the frontend

```bash
streamlit run frontend/app.py
```

Open http://localhost:8501 — ask a question, watch the four agents collaborate.

---

## Docker Compose (full stack)

```bash
cp .env.example .env
docker-compose up --build
```

| Service | URL |
|---|---|
| FastAPI | http://localhost:8000/docs |
| Streamlit | http://localhost:8501 |
| Ollama | http://localhost:11434 |

---

## API Reference

| Endpoint    | Method | Description                                       |
| ----------- | ------ | ------------------------------------------------- |
| `/health`   | `GET`  | Health check                                      |
| `/query`    | `POST` | Execute the full agent pipeline                   |
| `/ingest`   | `POST` | Download and index financial filings for a ticker |
| `/evaluate` | `POST` | Run the RAGAS evaluation harness                  |

### Example Query

```bash
curl -X POST http://localhost:8000/query \
  -H "Content-Type: application/json" \
  -d '{"query": "Compare Apple and Microsoft revenue growth from 2022 to 2023 and highlight key risk factors."}'
```

---

## Evaluation

Evaluate the RAG pipeline using the built-in QA dataset.

**Run locally:**

```bash
python -m src.evaluation.ragas_eval
```

**Or via the API:**

```bash
curl -X POST http://localhost:8000/evaluate
```

### Evaluation Metrics

| Metric                      | Description                                                                       |
| --------------------------- | --------------------------------------------------------------------------------- |
| **Faithfulness**            | Measures whether answer claims are supported by the retrieved context.            |
| **Answer Relevancy**        | Measures how relevant the generated answer is to the query.                       |
| **Context Precision**       | Measures the proportion of retrieved chunks that are relevant.                    |
| **Context Recall**          | Measures how much of the ground-truth answer is covered by the retrieved context. |
| **Verifier Grounding Rate** | Percentage of answers that pass the in-pipeline verifier agent.                   |
| **Hallucination Rate**      | Percentage of answers containing unverifiable claims.                             |

Evaluation results are saved to `data/eval_results.csv`.

### Before vs. After Verification

Fill in the actual evaluation results to compare performance with and without the verifier.

| Metric               | Without Verifier | With Verifier |
| -------------------- | ---------------: | ------------: |
| Hallucination Rate   |             [X]% |          [Y]% |
| Average Faithfulness |              [A] |           [B] |

---

## AWS Deployment

### Prerequisites

* AWS CLI configured with appropriate credentials
* Terraform ≥ 1.5
* An S3 bucket for Terraform state (update `terraform/main.tf`)

### Infrastructure Provisioning

```bash
cd terraform
terraform init
terraform apply
```

Terraform provisions the following AWS resources:

| Resource              | Purpose                         |
| --------------------- | ------------------------------- |
| **VPC**               | Isolated network infrastructure |
| **EC2 (`t3.medium`)** | Application hosting             |
| **ECR**               | Container image repositories    |
| **S3**                | Artefact storage                |
| **IAM Roles**         | Access control and permissions  |

### CI/CD Configuration

Configure the following secrets under **GitHub → Settings → Secrets and variables → Actions**.

| Secret                | Description                                                          |
| --------------------- | -------------------------------------------------------------------- |
| `AWS_DEPLOY_ROLE_ARN` | IAM role ARN for GitHub OIDC authentication                          |
| `EC2_INSTANCE_ID`     | EC2 instance ID (e.g., `i-0abc123`)                                  |
| `ECR_REGISTRY`        | ECR registry URL (e.g., `<account>.dkr.ecr.us-east-1.amazonaws.com`) |

### Deployment Workflow

Pushing changes to `main` triggers the GitHub Actions pipeline:

1. **Test** — Run automated tests.
2. **Build** — Build Docker images.
3. **Push** — Publish images to Amazon ECR.
4. **Deploy** — Deploy the application to EC2 via AWS Systems Manager (SSM).


## Project Structure

```
financial-research-assistant/
├── src/
│   ├── agents/
│   │   ├── state.py          # LangGraph AgentState schema
│   │   ├── planner.py        # Decomposes query → sub-queries
│   │   ├── retriever_agent.py# RAG retrieval over vector store
│   │   ├── analyst.py        # Synthesises answer + yfinance data
│   │   ├── verifier.py       # Groundedness check + finaliser
│   │   └── graph.py          # LangGraph workflow + retry logic
│   ├── rag/
│   │   ├── embeddings.py     # sentence-transformers (local)
│   │   ├── vector_store.py   # FAISS / Chroma build + load
│   │   └── retriever.py      # MMR retrieval + formatting
│   ├── data_ingestion/
│   │   ├── sec_downloader.py # SEC EDGAR downloader
│   │   └── document_processor.py # Load + chunk documents
│   ├── evaluation/
│   │   ├── eval_dataset.py   # Ground-truth QA pairs
│   │   ├── ragas_eval.py     # RAGAS pipeline evaluation
│   │   └── llm_judge.py      # LLM-as-judge fallback
│   ├── api/
│   │   ├── main.py           # FastAPI app
│   │   └── schemas.py        # Pydantic request/response models
│   ├── config.py             # Settings (pydantic, dotenv)
│   └── llm_factory.py        # Provider-agnostic LLM factory
├── frontend/
│   └── app.py                # Streamlit UI
├── scripts/
│   └── ingest.py             # CLI ingestion script
├── tests/
│   ├── test_agents.py        # Agent unit tests (mocked LLM)
│   └── test_api.py           # API integration tests
├── terraform/
│   └── main.tf               # AWS infrastructure (EC2, ECR, S3)
├── .github/workflows/
│   └── ci-cd.yml             # Test → Build → Deploy pipeline
├── docker-compose.yml
├── Dockerfile
├── Dockerfile.frontend
└── requirements.txt
```
