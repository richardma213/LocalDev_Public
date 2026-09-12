# LocalDev

A local-first AI coding assistant — a React code editor and chat UI backed by a
FastAPI server, talking entirely to a **local** LLM through
[LM Studio](https://lmstudio.ai/) (tested with `qwen2.5-7b-instruct-1m`). No
cloud API calls, no API keys, no telemetry — your code and conversations never
leave your machine.

The workspace folder is opened and read **in the browser** via the File System
Access API. The backend is a thin, stateless wrapper around the local model; it
never sees your files or paths — the frontend assembles everything it needs and
sends it in the request body.

> Full technical reference (state architecture, request lifecycle, persistence
> map, every module's job): [`docs/architecture.md`](docs/architecture.md).

## What it does

- **Two chat surfaces.** A per-file chat pane in the editor (anchored to the
  open buffer, auto/chat/edit modes) and a general main chat — sharing one
  workspace folder handle, independent transcripts.
- **Real file access from the browser.** Open a local folder, browse/edit
  files with a Monaco editor, save straight back to disk — no upload, no
  server-side file storage.
- **Retrieval-augmented context (RAG).** A hand-rolled, dependency-free BM25
  lexical index over the open workspace, built and persisted in IndexedDB.
  Every message auto-attaches the most relevant chunks from elsewhere in the
  project — no embeddings, no extra model call to retrieve.
- **Context compression.** Resolved edit-mode turns collapse to a short
  placeholder once the conversation has moved past them, so old diffs don't
  keep eating the context budget.
- **On-demand compaction.** A `/compact`-style button that summarizes older
  history into one message, streamed live, with real usage stats (compression
  ratio, tokens saved) logged automatically and shown in Settings.
- **A calibrated token meter.** The chat's token-budget estimate is checked
  against the model's real reported usage on every request and corrected with
  a measured, cross-validated factor — not just guessed.

## Status

Actively developed. Editor chat, main chat, RAG, compression, compaction, and
calibration are built and working. A multi-file edit workflow (plan → dispatch
→ per-file diff review, across several files at once) is designed but not yet
implemented — see the Extension Points table in the architecture doc.

## Running it locally

**1. LM Studio** — load a model and start its local server (default
`http://localhost:1234`), OpenAI-compatible.

**2. Backend**
```bash
cd backend
python -m venv .venv
.venv\Scripts\activate          # or `source .venv/bin/activate` on macOS/Linux
pip install -r requirements.txt
uvicorn main:app --reload --port 8000
```

**3. Frontend**
```bash
cd frontend
npm install
npm run dev
```

Open the printed local URL, open a folder, and start chatting. The frontend
defaults to `http://localhost:8000` for the backend — override with a
`VITE_API_URL` env var if you're running it elsewhere.

## Tech stack

React 19 · Vite · TypeScript · Tailwind v4 · Monaco Editor · FastAPI · Pydantic
· LM Studio (OpenAI-compatible local inference)
