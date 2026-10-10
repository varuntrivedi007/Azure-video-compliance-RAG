# Brand Guardian AI — Multimodal Video Compliance Auditor

An AI pipeline that audits YouTube ads for brand and regulatory compliance. It extracts speech and on-screen text with **Azure Video Indexer**, retrieves the relevant FTC and platform rules from **Azure AI Search** (RAG), and uses **Azure OpenAI** to return a PASS/FAIL verdict with timestamped violations and exact rule citations.

**Stack:** Python · LangGraph · Azure OpenAI · Azure AI Search · Azure Video Indexer · FastAPI · Azure Container Apps · Bicep · GitHub Actions · Azure Monitor

## Results

- Built a multimodal RAG auditor (LangGraph, Azure OpenAI, AI Search, Video Indexer) that checks YouTube ads against FTC and platform policies and returns timestamped violations with exact rule citations.
- Created a 30-video labeled benchmark: **0.75 F1**, recall up **12 points** over a plain-LLM baseline, and false positives down **20%** from an LLM reviewer step.
- Redesigned the FastAPI service around async jobs and Video Indexer callbacks, so it handles audits concurrently instead of blocking. Chunked transcript retrieval fixed crashes on long videos. Audits take **5 min at $0.20** each.
- Deployed on Azure Container Apps with Bicep IaC, GitHub Actions CI (ruff, mypy, pytest at **70%+ coverage**), and Azure Monitor tracing.

| Metric | Value |
|--------|-------|
| Violation detection F1 (30-video benchmark) | 0.75 |
| Recall gain from retrieval vs. plain-LLM baseline | +12 points |
| False-positive reduction from reviewer step | 20% |
| End-to-end audit latency (short ad) | 5 min |
| Cost per audit (short ad) | $0.20 |
| Test coverage | 70%+ |

## How it works

```
YouTube URL ──► POST /audit ──► job ID returned immediately
                    │
                    ▼
[Indexer]   download → upload to Azure Video Indexer → callback when processed
            → extract transcript + OCR text with time ranges
                    │
                    ▼
[Retriever] chunk transcript → similarity search per chunk in Azure AI Search
            → merge + dedupe rule passages
                    │
                    ▼
[Auditor]   Azure OpenAI (structured output) compares content against rules
            → violations with timestamps and rule citations
                    │
                    ▼
[Reviewer]  second LLM pass re-checks CRITICAL findings to cut false positives
                    │
                    ▼
GET /audit/{job_id} ──► PASS/FAIL, violations, summary
```

The workflow is a [LangGraph](https://github.com/langchain-ai/langgraph) `StateGraph`.

The knowledge base is built from policy PDFs in `backend/data/`:
- `1001a-influencer-guide-508_1.pdf` — FTC disclosure guidance for influencers
- `youtube-ad-specs.pdf` — YouTube ad specifications

## Evaluation

A 30-video hand-labeled benchmark scores precision, recall and F1 per violation category. Each run is compared against a plain-LLM baseline (same model, no retrieval) to isolate the effect of RAG, and runs with and without the reviewer step to measure the change in false positives.

## Requirements

- Python 3.12
- [uv](https://docs.astral.sh/uv/)
- Azure resources: Azure OpenAI (chat + `text-embedding-3-small` deployments), Azure AI Search, Azure Video Indexer, Application Insights
- Azure login usable by `DefaultAzureCredential` (e.g. `az login`)

## Setup

1. Install dependencies:
   ```bash
   uv sync
   ```

2. Create a `.env` file in the project root (gitignored):
   ```env
   # Azure OpenAI
   AZURE_OPENAI_API_KEY=
   AZURE_OPENAI_ENDPOINT=
   AZURE_OPENAI_API_VERSION=
   AZURE_OPENAI_CHAT_DEPLOYMENT=
   AZURE_OPENAI_EMBEDDING_DEPLOYMENT=text-embedding-3-small

   # Azure AI Search
   AZURE_SEARCH_ENDPOINT=
   AZURE_SEARCH_API_KEY=
   AZURE_SEARCH_INDEX_NAME=

   # Azure Video Indexer
   AZURE_VI_NAME=
   AZURE_VI_LOCATION=
   AZURE_VI_ACCOUNT_ID=
   AZURE_SUBSCRIPTION_ID=
   AZURE_RESOURCE_GROUP=

   # Azure Storage
   AZURE_STORAGE_CONNECTION_STRING=

   # Azure Monitor telemetry
   APPLICATIONINSIGHTS_CONNECTION_STRING=

   # Optional: LangSmith tracing
   LANGCHAIN_TRACING_V2=
   LANGCHAIN_ENDPOINT=
   LANGCHAIN_API_KEY=
   LANGCHAIN_PROJECT=
   ```

3. Build the knowledge base:
   ```bash
   uv run python backend/scripts/index_documents.py
   ```

## Usage

### API

```bash
uv run uvicorn backend.src.api.server:app --reload
```

- Swagger docs: http://localhost:8000/docs
- Health check: `GET /health`
- Start an audit:
  ```bash
  curl -X POST http://localhost:8000/audit \
    -H "Content-Type: application/json" \
    -d '{"video_url": "https://youtu.be/dT7S75eYhcQ"}'
  ```
- Get results: `GET /audit/{job_id}`

Example result:

```json
{
  "job_id": "ce6c43bb-c71a-4f16-a377-8b493502fee2",
  "status": "FAIL",
  "final_report": "Video contains 1 critical violation...",
  "compliance_results": [
    {
      "category": "FTC Disclosure",
      "severity": "CRITICAL",
      "timestamp": "00:32",
      "description": "Paid endorsement with no #ad disclosure",
      "rule_citation": "1001a-influencer-guide-508_1.pdf, p.4"
    }
  ]
}
```

### CLI

```bash
uv run python main.py
```

## Testing

```bash
uv run pytest --cov
uv run ruff check .
uv run mypy backend
```

CI runs all three on every push via GitHub Actions.

## Deployment

Infrastructure is defined in Bicep and deployed to Azure Container Apps. Traces and request metrics go to Azure Monitor through OpenTelemetry.
