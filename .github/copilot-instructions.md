# Project: stream-chat

A streaming LLM chat app with a small RAG feature, built in a short sprint by one developer who must be able to explain every file in an interview. Prefer simple, readable code over clever code.

## Stack (do not change without asking)
- Frontend: Vite, React 18+, TypeScript (strict), hooks only, plain CSS, no UI library. Tests: Vitest, React Testing Library, jsdom.
- Backend: Python 3.12, FastAPI, uvicorn, boto3. Runs on AWS Lambda through the Lambda Web Adapter with response streaming, behind an API Gateway REST API. Tests: pytest with FastAPI TestClient. Lint: ruff.
- LLM: Amazon Bedrock Converse streaming API (converse_stream). Embeddings: Amazon Titan Text Embeddings V2 (amazon.titan-embed-text-v2:0, 512 dimensions, normalized).
- Storage: S3 on Lambda; an in-memory dict locally and in tests.
- Infra: AWS SAM (template.yaml at the repo root). CI: GitHub Actions. Frontend hosting: Netlify.

## Repo layout
- frontend/ : Vite app. src/lib/sse.ts, src/hooks/useChatStream.ts, src/components/
- backend/app/ : main.py (routes), llm.py, rag.py, storage.py, sse.py, config.py
- backend/tests/ : pytest tests
- backend/scripts/ : one-off helper scripts
- backend/run.sh : Lambda Web Adapter start script
- template.yaml : SAM template (Lambda, API Gateway, S3)
- .github/workflows/ci.yml : CI

## Architecture rules
- Streaming uses Server-Sent Events over a POST. The frontend reads it with fetch and a ReadableStream reader, not EventSource.
- Event protocol is the single source of truth. Each event is one line starting with data: followed by JSON, then a blank line:
  - {"type":"sources","items":[{"n":1,"doc":"name","snippet":"..."}]} sent first, only if documents were retrieved
  - {"type":"token","text":"..."} sent many times
  - {"type":"done"} sent once at the end
  - {"type":"error","message":"..."} for any failure after streaming has started (the HTTP status cannot change mid-stream)
- Routes: GET /health; POST /chat with session_id and messages [{role, content}]; POST /documents with session_id, filename, text; GET /documents?session_id=; DELETE /documents/{doc_id}?session_id=.
- LLMClient and Embedder are interfaces with Fake and Bedrock implementations, chosen by env vars LLM_BACKEND (fake or bedrock) and injected with FastAPI Depends so tests use the fakes. Storage is chosen by STORAGE_BACKEND (memory or s3).
- The chat route's generator is a plain def generator, not async, because boto3 blocks.
- RAG: chunk_text (800 characters, 120 overlap, paragraph-aware) then embed then store {filename, chunks, embeddings} per session. At chat time embed the last user message, rank chunks by cosine similarity in pure Python (vectors are normalized, so cosine is a dot product), keep the top 4, put them in the prompt as numbered context, answer only from that context, and say so when the answer is not there.
- Limits: 4,000 characters per message, max_tokens 800, document text at most 200 KB. Embed chunks with a ThreadPoolExecutor of 5 workers.
- Configuration comes from env vars: LLM_BACKEND, STORAGE_BACKEND, BEDROCK_MODEL_ID, EMBED_MODEL_ID, BUCKET_NAME, ALLOWED_ORIGINS, AWS_REGION. Never hard-code model IDs, keys or bucket names.

## Frontend rules
- src/lib/sse.ts exports a pure function parseSSE(buffer) that returns { events, rest }.
- src/hooks/useChatStream.ts exposes messages, isStreaming, error, send(text) and stop(). It appends tokens with functional setState, uses an AbortController, and treats AbortError as a normal end.
- The API base URL comes from import.meta.env.VITE_API_URL.
- One small component per file in src/components.

## Code style
- Python: type hints, small functions, one-line docstrings, ruff clean. TypeScript: strict mode, no any.
- Keep files under about 150 lines where possible.
- Add or update a test whenever you add a pure function.
- Do not add a dependency without saying why.

## How to work with me
- Build step by step in the order a human would write it: say which file you are creating, create it, explain it in two or three sentences, then continue. Never generate the whole project at once.
- I send one block prompt at a time (B0 to B8). Do only that block, and list the files you will create or change before you start.
- Run terminal commands yourself in the integrated terminal whenever you can (I approve each one) and show me the result. For things only I can do (AWS login or console, Netlify, interactive prompts), tell me exactly what to click or type, then wait.
- After each block, tell me what I should see to confirm it works, then stop and wait for my confirmation. When I confirm, commit and push with the commit message I give you.
- Never put secrets in code and never commit .env files.
- If a requirement is unclear, ask one short question instead of guessing.
- If you are unsure about AWS, SAM or Bedrock details (SAM streaming properties, model IDs, IAM actions), say so and point me to the docs instead of inventing them.