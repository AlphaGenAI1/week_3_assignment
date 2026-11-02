# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project

Full-stack RAG app over ~11,008 chunks of SIGGRAPH 2025 papers (a course assignment). FastAPI backend in `backend/` (port 8082), Next.js 15 / React 19 / Tailwind frontend in `frontend/` (port 3000). All LLM and embedding calls go through the OpenRouter API; vectors live in Qdrant Cloud.

## Commands

Python environment: use `C:\5_Python\Basics\.venv\Scripts\python.exe` for all Python and pip commands (e.g. `& C:\5_Python\Basics\.venv\Scripts\python.exe -m pip ...`), not the system `python`. Ask the user before installing any package.

Backend (run from `backend/`; scripts use relative paths like `./chunks.json`):

```bash
pip install -r requirements.txt
cp .env.example .env                       # fill OPENROUTER_API_KEY, QDRANT_URL, QDRANT_API_KEY, optional COHERE_API_KEY
python upload_from_npz.py --recreate       # one-time: upload precomputed BGE embeddings (needs embeddings_BAAI_bge_large_en_v1.5.npz from Drive)
python upload_to_qdrant.py --recreate      # alternative: embed via OpenRouter and upload
python retrieval_pipeline.py "3D Gaussian Splatting"     # test retrieval alone
python rag_generate.py "What is 3D Gaussian Splatting?"  # test full RAG alone
python api_server.py                       # dev server on :8082 (PORT env overrides); docs at /docs
curl -N "http://localhost:8082/api/stream?query=3D%20Gaussian%20Splatting&top_k=5"
```

On Windows, `api_server.py`'s auto-reload can log `Reloading...` and keep serving the old code, so restart the server fully after editing backend files. In PowerShell, use `curl.exe` (plain `curl` is an alias for `Invoke-WebRequest`) and `127.0.0.1` instead of `localhost`.

Production start (Render, root dir `backend`): `uvicorn api_server:app --host 0.0.0.0 --port $PORT`.

Frontend (run from `frontend/`; needs `.env.local` with `NEXT_PUBLIC_API_URL=http://localhost:8082`, see `env.local.example`):

```bash
npm install
npm run dev
npm run build
npm run lint
```

There is no automated test suite. Each backend module has a `__main__` CLI for manual testing (above). `test_backend_integration.py` is a mock `RAGGenerator` that can be swapped into `api_server.py`'s import to exercise the API/frontend without Qdrant or OpenRouter.

## Architecture

Pipeline layers, each depending on the one below:

1. **Ingestion** (`upload_from_npz.py` / `upload_to_qdrant.py`): load `chunks.json`, upsert into Qdrant collection `siggraph2025_papers` with 1024-dim `baai/bge-large-en-v1.5` vectors, using `chunk_id` as point ID and chunk metadata (title, authors, pdf/github/video links) as payload.
2. **Retrieval** (`retrieval_pipeline.py`): `RetrievalPipeline.retrieve()` = Qdrant semantic search (query embedded via `OpenRouterEmbedder`) + in-memory `BM25Index` built from `chunks.json` at startup, combined as a weighted hybrid (default 0.7 semantic / 0.3 BM25), then optional Cohere rerank. Returns `RetrievalResult` objects.
3. **Generation** (`rag_generate.py`): `RAGGenerator` owns a `RetrievalPipeline` (`.retrieval`) and provides `refine_query()`, `_format_context()`, `_build_sources_metadata()`, `_call_llm()`, `generate()`. Also exports `SYSTEM_PROMPT` and `GenerationConfig`.
4. **API** (`api_server.py`): builds one global `RAGGenerator` in the FastAPI lifespan hook. Endpoints: `GET /health`, `GET /api/info`, `POST /api/query` (non-streaming), `GET /api/stream` (SSE), `WS /ws/query`.
5. **Frontend**: `src/app/page.tsx` uses `hooks/useRAGStream.ts` (SSE against `/api/stream`). `useRAGWebSocket.ts` is an alternate WebSocket client. UI pieces are in `components/rag/`.

### Cross-file contracts that are easy to break

- **The streaming endpoints don't call `RAGGenerator.generate()`.** `stream_rag_response` (and the WebSocket handler) call `refine_query`, `retrieval.retrieve`, `_format_context` and `_build_sources_metadata` directly, then stream from OpenRouter themselves using `openrouter_api_key`, `openrouter_base_url`, `config.llm_model`, `config.temperature` and `config.max_tokens`. Changes to these private method signatures or attribute names break the API, and changes to `generate()`/`_call_llm()` do not affect what the frontend sees.
- `GenerationConfig.llm_model` is a full OpenRouter model ID (e.g. `"openai/gpt-4o"`), so the streaming code in `api_server.py` must send it as-is, without adding another `openai/` prefix.
- The WebSocket handler calls `retrieval.retrieve(..., use_hybrid=..., use_reranker=...)`, so `RetrievalPipeline.retrieve()` must accept those keyword arguments.
- `api_server.py` constructs `GenerationConfig(llm_provider=..., retrieval_top_k=..., refine_query=..., use_reranker=...)`, so the dataclass must accept those fields.
- `RetrievalResult` field names are consumed by the API (`paper_id` is used to count distinct papers) and by `_build_sources_metadata`. Don't rename them.
- `_build_sources_metadata` returns a list of unique source dicts, which the API sends unchanged as `sources` (the mock in `test_backend_integration.py` returns a dict instead, so it is no longer a drop-in replacement). The frontend's `SourcesSection`/`ReferencesSection` read those source objects.
- The SSE event shape (`{type: progress|chunk|complete|error, ...}`) is parsed by `useRAGStream.ts`. Change both sides together.
- Embedding model and vector size must be identical in the upload scripts and `OpenRouterEmbedder` (BGE-large, 1024 dims), and `COLLECTION_NAME` is hardcoded separately in each backend script.
