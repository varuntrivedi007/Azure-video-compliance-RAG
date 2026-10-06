# Brand Guardian AI — Video Compliance QA Pipeline

An AI pipeline that audits YouTube videos for brand and regulatory compliance. It extracts speech and on-screen text from a video with **Azure Video Indexer**, retrieves relevant rules from a knowledge base in **Azure AI Search** (RAG), and uses **Azure OpenAI** to flag violations and produce a PASS/FAIL report.

## How it works

```
YouTube URL
   │
   ▼
[Indexer node]  yt-dlp download → upload to Azure Video Indexer → wait for processing
   │            → extract transcript + OCR text
   ▼
[Auditor node]  similarity search in Azure AI Search (top 3 rule chunks)
   │            → Azure OpenAI chat model compares video content against rules
   ▼
Compliance report: status (PASS/FAIL), list of violations, summary
```

The workflow is a [LangGraph](https://github.com/langchain-ai/langgraph) `StateGraph`: `START → indexer → auditor → END`.

The knowledge base is built from the PDFs in `backend/data/`:
- `1001a-influencer-guide-508_1.pdf` — FTC disclosure guidance for influencers
- `youtube-ad-specs.pdf` — YouTube ad specifications

## Project structure

```
.
├── main.py                          # CLI entry point: runs one audit and prints the report
├── pyproject.toml / uv.lock         # Dependencies (managed with uv)
├── backend/
│   ├── data/                        # Rule PDFs that feed the knowledge base
│   ├── scripts/
│   │   ├── index_documents.py       # Chunks PDFs, embeds them, uploads to Azure AI Search
│   │   └── explanation.txt          # Notes on the indexing run
│   ├── src/
│   │   ├── api/
│   │   │   ├── server.py            # FastAPI app: POST /audit, GET /health
│   │   │   └── telemetry.py         # Azure Monitor (OpenTelemetry) setup
│   │   ├── graph/
│   │   │   ├── state.py             # VideoAuditState and ComplianceIssue schemas
│   │   │   ├── nodes.py             # Indexer and Auditor nodes
│   │   │   └── workflow.py          # LangGraph wiring
│   │   └── services/
│   │       └── video_indexer.py     # Azure Video Indexer client (download, upload, poll, extract)
│   └── Dockerfile                   # Placeholder (empty)
└── azure_functions/                 # Placeholder for an Azure Functions deployment (empty)
```

## Requirements

- Python 3.12
- [uv](https://docs.astral.sh/uv/)
- Azure resources:
  - Azure OpenAI with a chat deployment and a `text-embedding-3-small` deployment
  - Azure AI Search
  - Azure Video Indexer (ARM-based account)
  - Optional: Application Insights for telemetry
- Azure login that `DefaultAzureCredential` can use (e.g. `az login`) — used to get Video Indexer tokens

## Setup

1. Install dependencies:
   ```bash
   uv sync
   ```

2. Create a `.env` file in the project root (it is gitignored):
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

   # Optional: Azure Monitor telemetry
   APPLICATIONINSIGHTS_CONNECTION_STRING=

   # Optional: LangSmith tracing
   LANGCHAIN_TRACING_V2=
   LANGCHAIN_ENDPOINT=
   LANGCHAIN_API_KEY=
   LANGCHAIN_PROJECT=
   ```

3. Build the knowledge base (run once, or whenever the PDFs change):
   ```bash
   uv run python backend/scripts/index_documents.py
   ```

## Usage

### CLI

Runs an audit on the sample video URL set in `main.py`:

```bash
uv run python main.py
```

### API

```bash
uv run uvicorn backend.src.api.server:app --reload
```

- Swagger docs: http://localhost:8000/docs
- Health check: `GET /health`
- Audit a video:
  ```bash
  curl -X POST http://localhost:8000/audit \
    -H "Content-Type: application/json" \
    -d '{"video_url": "https://youtu.be/dT7S75eYhcQ"}'
  ```

Example response:

```json
{
  "session_id": "ce6c43bb-c71a-4f16-a377-8b493502fee2",
  "video_id": "vid_ce6c43bb",
  "status": "FAIL",
  "final_report": "Video contains 2 critical violations...",
  "compliance_results": [
    {
      "category": "Misleading Claims",
      "severity": "CRITICAL",
      "description": "Absolute guarantee detected"
    }
  ]
}
```

## Notes

- Only YouTube URLs are supported right now.
- Video Indexer processing is polled every 30 seconds, so an audit can take several minutes.
- The `/audit` endpoint runs the workflow synchronously and blocks until the audit finishes.
