# Local-only LibreChat Setup Plan (Claude first, private documents later)

This plan is optimized for a **single-user**, **local-only** setup on **Windows 10/11** using **Docker Desktop + WSL2**.

Goals:
- Start quickly with **Anthropic Claude** for chat.
- Keep the app **not exposed to the public internet** (only `localhost`).
- Later: enable **chat with your documents** using **local/private embeddings** (no cloud embeddings provider).

> Note on privacy: With RAG, the **embeddings** can be computed locally, but when you ask Claude a question, LibreChat may send **relevant document snippets** to Claude to answer accurately. If you need *zero document text ever leaving your machine*, you’d need a fully local LLM for answering too.

---

## Phase 0 — Decide the run mode (we’re using local-only Docker)

We will run everything locally with Docker containers:
- LibreChat API/UI
- MongoDB (chat history)
- Meilisearch (search/indexing)
- (Later) RAG API + pgvector + Ollama (local embeddings)

No cloud compute required.

---

## Phase 1 — Get Claude-only chat working (fastest success)

### 1) Prerequisites
- Docker Desktop installed
- WSL2 enabled (Docker Desktop → Settings → Resources → WSL integration)
- Anthropic API key (Claude)

### 2) Create `.env`
In the repo root:

1. Copy `.env.example` → `.env`
2. Edit `.env` and set:

Required:
- `HOST=localhost`
- `PORT=3080` (or your preferred port)
- `DOMAIN_CLIENT=http://localhost:3080`
- `DOMAIN_SERVER=http://localhost:3080`
- `ANTHROPIC_API_KEY=YOUR_KEY_HERE`

Recommended (Windows + Docker):
- `UID=1000`
- `GID=1000`

Recommended (security hygiene even for local-only):
- Replace `JWT_SECRET` and `JWT_REFRESH_SECRET` with new random values (don’t reuse the example values)

Cost/UX guardrail (optional but recommended):
- Set `ANTHROPIC_MODELS=` to only the models you intend to use.
  Example:
  - `ANTHROPIC_MODELS=claude-3-5-haiku-20241022,claude-3-7-sonnet-20250219`

### 3) Start with Docker Compose
From the repo root:

```bash
docker compose up -d
```

Then open:
- http://localhost:3080

### 4) Create your user
Register a user in the UI (since this is single-user local-only).

Optional hardening afterward:
- Set `ALLOW_REGISTRATION=false` in `.env` once your account exists.

### 5) Validate baseline functionality
- Confirm the UI loads
- Select an Anthropic model
- Send a message
- Confirm responses stream back

---

## Phase 1.5 — Optional “powerful but controlled” defaults

These are optional tweaks that help keep your setup powerful without surprise usage:

1) Keep a “cheap default” model available (Haiku) and a “quality” model (Sonnet).

2) Avoid enabling lots of automation loops immediately (agents/tool recursion) until baseline is stable.

3) Keep it local-only:
- Don’t expose database ports.
- Only publish the app on `localhost`.

---

## Phase 2 — Enable private/local document chat (local embeddings)

LibreChat already includes `rag_api` + `vectordb` in `docker-compose.yml`. To keep embeddings local, we’ll add **Ollama** and use the RAG API image that supports local embeddings.

### 1) Add an override file for the RAG stack
Create `docker-compose.override.yml` in the repo root.

We will:
- Switch `rag_api` to `ghcr.io/danny-avila/librechat-rag-api-dev:latest` (the variant that supports local embeddings)
- Add an `ollama` service

> We will generate this file together when you’re ready so it matches your machine (CPU/GPU).

### 2) Configure the RAG API to use local embeddings
In `.env`, set the embeddings provider/model for RAG.

The exact variables depend on the RAG API configuration, but the goal is:
- **Embeddings provider:** local (Ollama)
- **Embeddings model:** a solid embeddings model supported by Ollama

### 3) Start/restart containers
```bash
docker compose up -d
```

### 4) Validate document chat
- Upload a small PDF or text file
- Ask a question that requires citing the document
- Confirm citations/sources appear (if enabled)

### 5) Performance tuning (later)
- If indexing is slow on CPU, choose a smaller embeddings model.
- Adjust file limits if needed.
- Consider excluding large folders or sensitive material you never want summarized.

---

## Phase 3 — Add 3–5 model providers (later)

Once stable with Claude:
- Add provider API keys in `.env`
- Enable providers/models in `librechat.yaml` (or via `.env` model lists where applicable)
- Set defaults and keep a cheap-fast model available for everyday tasks

---

## Maintenance (local-only, single user)

Lightweight, but do this occasionally:
- Update: `git pull` then rebuild/restart containers if needed
- Back up `./data-node` (MongoDB data) and `./uploads`
- Keep your `.env` secrets private

---

## Next action (we do one step at a time)

Start Phase 1 now:
1) Create `.env` from `.env.example`
2) Set `ANTHROPIC_API_KEY`, `UID/GID`, and confirm `PORT=3080`
3) Run `docker compose up -d`

Tell me:
- Do you already have Docker Desktop installed and WSL2 enabled?
