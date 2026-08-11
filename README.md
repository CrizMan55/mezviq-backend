# Mezviq Support — Implementation Blueprint

Multi-tenant omnichannel AI customer support & booking engine.
Stack: **FastAPI · LangChain · OpenAI GPT-4o-mini (+ Claude 3 Haiku fallback) · Supabase Postgres + pgvector · Meta WhatsApp Cloud API · vanilla JS widget**

---

## 1. Directory Structure

```
mezviq-support/
├── app/
│   ├── main.py                  # FastAPI app entrypoint, router mounting, CORS
│   ├── config.py                # Pydantic Settings — loads .env
│   ├── security.py              # Webhook signature validation, token encryption, rate limiting
│   ├── deps.py                  # Shared dependencies: DB session, tenant resolver
│   ├── db/
│   │   ├── session.py           # SQLAlchemy engine/session factory
│   │   └── models.py            # Tenant, KnowledgeBase, ChatLog ORM models
│   ├── rag/
│   │   ├── ingest.py            # PDF/Excel → chunks → embeddings → pgvector upsert
│   │   ├── pipeline.py          # Retrieval + grounded prompt + LLM call
│   │   └── prompts.py           # System prompt templates (EN/ML/Manglish grounding rules)
│   ├── routers/
│   │   ├── whatsapp.py          # GET verify + POST webhook handler
│   │   ├── widget.py            # REST endpoint the web widget calls
│   │   ├── admin.py             # Tenant CRUD, KB upload (Super Admin dashboard API)
│   │   └── handover.py          # Human agent handover / live takeover endpoint
│   └── schemas.py               # Pydantic request/response models
├── alembic/                     # DB migrations (versioned schema changes)
│   └── env.py
├── widget/
│   ├── chat-widget.js           # Embeddable script (the <script data-client-id=...> tag)
│   └── demo.html                # Local test harness for the widget
├── tests/
│   ├── test_whatsapp_webhook.py
│   └── test_rag_pipeline.py
├── requirements.txt
├── alembic.ini
├── .env.example
└── README.md
```

---

## 2. Roadmap → Coding Tasks

### Milestone 1 — Core Engine & RAG Setup
1. `config.py` + `.env` — load OpenAI/Anthropic keys, DB URL, vector store creds.
2. `db/models.py` + Alembic migration — create `tenants`, `knowledge_base`, `chat_logs`.
3. `rag/ingest.py` — PDF/Excel parsing (pypdf, openpyxl) → chunk (LangChain `RecursiveCharacterTextSplitter`) → embed (`text-embedding-3-small`) → upsert into pgvector namespaced by `tenant_id`.
4. `rag/pipeline.py` — similarity search scoped to tenant → grounded prompt → GPT-4o-mini call → hallucination guard (refuse if retrieval score below threshold).
5. `rag/prompts.py` — system prompt enforcing "answer only from context" + language mirroring (EN/ML/Manglish).
6. Dependency: **1 must fully pass before 2 (webhook depends on RAG pipeline existing)**.

### Milestone 2 — WhatsApp API Integration
1. `routers/whatsapp.py` — `GET /webhook/whatsapp` (hub.challenge echo) and `POST /webhook/whatsapp` (payload parse).
2. `security.py::verify_meta_signature` — validate `X-Hub-Signature-256` on every POST.
3. Tenant resolution by `phone_number_id` → look up `whatsapp_access_token` (decrypt) → call Meta Send API.
4. Wire the webhook handler to `rag/pipeline.py` for the actual reply text.
5. Dependency: needs Milestone 1's pipeline + `db/models.Tenant`.

### Milestone 3 — Web Widget
1. `widget/chat-widget.js` — self-contained IIFE, injects floating button + iframe/panel, posts to `POST /api/widget/message`.
2. `routers/widget.py` — session-based (not WhatsApp-token-based) endpoint, same RAG pipeline, CORS scoped per `data-client-id`.
3. Dependency: Milestone 1 pipeline only (independent of Milestone 2).

### Milestone 4 — Admin Dashboard & Multi-Tenancy
1. `routers/admin.py` — `POST /admin/tenants`, `POST /admin/tenants/{id}/knowledge-base` (triggers `rag/ingest.py`), `PATCH /admin/tenants/{id}/whatsapp-credentials`.
2. Auth: Super Admin JWT/session guard (separate from tenant-level API keys).
3. Dependency: Milestones 1–3 (dashboard configures what they use).

### Milestone 5 — Testing & Production Deployment
1. Unit tests for webhook parsing + signature validation + RAG grounding.
2. Load/stress test the `/webhook/whatsapp` and `/api/widget/message` endpoints.
3. Deploy to Render/Railway/AWS with secrets in a vault (not `.env` in prod).
4. Apply the Security Checklist (§ below) before go-live.

---

## 3. Environment Variables (`.env`)

```
# App
ENVIRONMENT=production
SECRET_KEY=                      # for JWT signing (admin dashboard auth)
FERNET_ENCRYPTION_KEY=           # for encrypting whatsapp_access_token at rest

# Database
DATABASE_URL=postgresql+asyncpg://user:pass@host:5432/mezviq
SUPABASE_URL=
SUPABASE_SERVICE_ROLE_KEY=

# Vector Store (pgvector via Supabase, or swap for Pinecone)
VECTOR_BACKEND=pgvector          # pgvector | pinecone
PINECONE_API_KEY=
PINECONE_ENVIRONMENT=

# LLMs
OPENAI_API_KEY=
ANTHROPIC_API_KEY=               # Claude 3 Haiku fallback
LLM_PRIMARY=gpt-4o-mini
LLM_FALLBACK=claude-3-haiku-20240307
EMBEDDING_MODEL=text-embedding-3-small

# WhatsApp Cloud API
META_APP_SECRET=                 # used to verify X-Hub-Signature-256
META_VERIFY_TOKEN=               # used in GET challenge
META_GRAPH_API_VERSION=v20.0

# Optional
OPENAI_WHISPER_MODEL=whisper-1   # voice message transcription
RATE_LIMIT_PER_MINUTE=30
```
