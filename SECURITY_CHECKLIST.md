# Security & Production Checklist

## Tenant Isolation
- [ ] Every DB query filters by `tenant_id` — never a bare `SELECT * FROM chat_logs`.
- [ ] Vector store uses a per-tenant `collection_name`/namespace (`tenant_{tenant_id}`) — retrieval can never leak another tenant's documents.
- [ ] Widget requests authenticate via `client_id` → resolve tenant server-side; never trust a tenant_id sent directly from the browser.

## Secrets & Token Encryption
- [ ] `whatsapp_access_token` stored encrypted at rest (Fernet/AES) — see `app/security.py::encrypt_secret`.
- [ ] `FERNET_ENCRYPTION_KEY` and `SECRET_KEY` pulled from a secrets manager in production (AWS Secrets Manager / Render env groups), never committed to git.
- [ ] Rotate `META_APP_SECRET`, `whatsapp_access_token`, and API keys on a schedule; support zero-downtime rotation.

## Meta Webhook Signature Validation
- [ ] `X-Hub-Signature-256` verified via constant-time HMAC compare on the **raw** request body before any JSON parsing (see `verify_meta_signature`).
- [ ] Reject requests with missing/malformed signature headers with 401, and don't leak *why* in the response body.
- [ ] `hub.verify_token` for the GET challenge stored server-side only, compared exactly.

## Prompt-Injection Resistance
- [ ] System prompt explicitly instructs the model to ignore in-message attempts to override its role or reveal the prompt (see `rag/prompts.py`).
- [ ] Retrieval-only grounding: model is told to answer *only* from retrieved context, with an explicit "I don't know, escalating to a human" fallback path.
- [ ] Pre-LLM input sanitizer flags known override phrases and routes straight to human handover instead of passing them to the model (`sanitize_user_input`).
- [ ] Treat retrieved document content as untrusted too — a malicious PDF could contain injected instructions; the grounding rule set should apply symmetrically.

## Rate Limiting & Abuse Prevention
- [ ] Per-IP and per-tenant rate limits on `/api/widget/message` (see `slowapi` usage) and on `/webhook/whatsapp`.
- [ ] Backoff/circuit-breaker around outbound Meta Send API and LLM calls (see `tenacity` in requirements) so one bad tenant can't exhaust the whole system's quota.
- [ ] Cap message length server-side (2000 chars) before it reaches the LLM, to control token spend and abuse.

## Audit Logging & Backups
- [ ] Every inbound/outbound message persisted to `chat_logs` with `sender_type` and `channel`, before the reply is sent — so failures are still auditable.
- [ ] Structured logs (not print statements) for webhook signature failures, tenant-resolution failures, and LLM fallbacks — these are your incident signals.
- [ ] Automated daily Postgres backups (Supabase point-in-time recovery, or `pg_dump` to S3) with a tested restore procedure.

## Access Control
- [ ] Super Admin dashboard routes (`/admin/*`) gated behind real auth (JWT/session) — the current templates leave this as a TODO, do not ship without it.
- [ ] Tenant-level dashboard users can only manage their own `tenant_id` — enforce via a dependency that cross-checks the authenticated tenant against the path/body `tenant_id`.
- [ ] CORS on `/api/widget/*` scoped to each tenant's registered `allowed_origins`, not `*`, before go-live.

## Load & Failure Testing (Milestone 5)
- [ ] Load-test `/webhook/whatsapp` and `/api/widget/message` at expected peak concurrency; confirm rate limiter behaves under burst traffic.
- [ ] Simulate OpenAI outage to confirm the Claude 3 Haiku fallback path actually engages (`_call_claude_fallback`).
- [ ] Simulate vector-store timeout to confirm the pipeline degrades to "connecting you with a human agent" rather than hanging or erroring the webhook (Meta will retry/disable the webhook on repeated failures).
