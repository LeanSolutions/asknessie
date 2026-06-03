# asknessie

Multi-tenant SaaS personal-assistant platform. A user can reach the assistant via web UI, mobile, email, SMS, voice (later), and Slack. The brain is one durable agent; channels are pluggable adapters.

The stack is deliberately lean. We DIY small utilities behind swap-ready interfaces instead of bringing in a SaaS for every concern. We prefer GCP-native services over third-party vendors. We accept vendor dependencies only where the alternative is "become an engineer in someone else's problem."

## Stack

### Backend
Python 3.13, FastAPI, Pydantic v2, SQLAlchemy 2.0 async + asyncpg, Alembic, structlog, httpx.

### Frontend (later)
Next.js 15 (web), Expo (mobile). TypeScript.

### GCP infra
Cloud Run (Gen 2, `min-instances=1` on hot paths), AlloyDB for PostgreSQL (with ScaNN vector search), Memorystore Redis, Cloud KMS, Secret Manager, Pub/Sub, BigQuery, Cloud Storage, Cloud Logging, Cloud Trace, Cloud Error Reporting, Vertex AI, FCM.

### LLM routing — all via Vertex AI

| Task | Model |
|---|---|
| Voice acks, intent triage, simple Q&A | Claude Haiku 4.5 |
| Main agent loop, tool use (default) | Claude Sonnet 4.6 |
| Explicit hard-reasoning step (rare) | Claude Opus 4.7 |
| Side tasks (classify, summarize, redact) | Gemini 2.5 Flash |
| Multimodal (image/video understanding) | Gemini 2.5 Pro |
| Embeddings | Vertex `text-embedding-005` |

Routing lives in our in-house `LLMClient` wrapper. One Vertex AI surface, one bill, one IAM model. Prompt caching is mandatory on every call where it can apply — biggest single speed and cost lever.

### Outside-GCP vendors (kept deliberately small)

| Vendor | Purpose | Why not DIY |
|---|---|---|
| Anthropic (consumed via Vertex AI) | LLM | Obvious |
| Twilio | SMS, voice phone numbers | No alternative |
| Postmark | Inbound email parsing | MIME/threading/attachments are a swamp |
| Resend (or Postmark outbound) | Outbound email | Deliverability/IP reputation |
| LiveKit Cloud | WebRTC media transport | WebRTC ops is a profession |
| WorkOS | B2B auth, SSO, SCIM | SSO/SCIM is genuinely hard |
| Stripe | Payments | No alternative |
| Expo | Mobile platform (incl. Expo Push) | Already in stack for mobile |

That's it. ~7 outside vendors.

## What we DIY (and where the swap-ready interface lives)

| Concern | We use | Interface | Vendor we're not using |
|---|---|---|---|
| LLM gateway (cache, retry, cost log) | `integrations/llm_client.py` | `LLMClient` | Helicone / Portkey |
| Durable execution | `lib/durable.py` (jobs table + worker) | `DurableExecutor` | Inngest |
| Tool definitions | typed Python functions + Pydantic args | `Tool` Protocol | MCP framework |
| Notifications (push/SMS/email) | `services/notify.py` | `NotifyDispatcher` | Knock |
| Feature flags | Postgres table + Redis cache + evaluator | `FlagEvaluator` | PostHog flags |
| Product analytics ingestion | `/events` → Pub/Sub → BigQuery | event schema | PostHog SaaS |
| LLM observability | structured logs + `llm_calls` table + BigQuery sink | log schema | Langfuse SaaS |
| Error tracking | structured ERROR logs → Cloud Error Reporting | (none — pass-through) | Sentry |
| Tenant credential vault | Cloud KMS envelope encryption + ciphertext in AlloyDB | `CredentialVault` | (no obvious SaaS) |

Each interface is the swap-out point. The `in-house-utilities` skill governs how we keep these small.

## Deferred decisions (when to revisit)

| Decision | Current pick | Trigger to swap |
|---|---|---|
| LLM tracing UI | DIY logs + BigQuery | Prompt iteration / A-B agent comparison becomes a daily workflow → self-host Langfuse on Cloud Run |
| Voice STT/TTS provider | Google Cloud STT + TTS, behind `STTClient` / `TTSClient` Protocol | Real user complaints about voice latency → swap to Deepgram + Cartesia |
| Frontend error tracking | Cloud Error Reporting (backend only for now) | Next.js ships and source maps matter → add Sentry just for frontend |
| Durable execution | DIY ~300 LOC behind `DurableExecutor` | Reliability incidents we can't engineer out, or fan-out complexity grows → swap to Inngest or Temporal |
| Product analytics UI | BigQuery + Looker Studio | Funnel/retention/cohort analysis becomes a daily workflow → add PostHog |
| Tool protocol | Direct Anthropic tool-use (no MCP) | External tool marketplace need → add MCP adapter behind `Tool` Protocol |
| Session replay | None | Specific UX-debugging need → Sentry replay or PostHog |

## Common commands

```sh
uv sync                                              # install deps
uv run granian --interface asgi src.main:app         # run server (dev)
uv run pytest                                        # all tests
uv run pytest -k <name> -x                           # focused test
uv run ruff check . && uv run ruff format .          # lint + format
uv run pyright                                       # type check
uv run alembic upgrade head                          # apply migrations
uv run alembic revision -m "<msg>" --autogenerate    # new migration
```

## Repo structure

```
asknessie/
  apps/
    api/                         # FastAPI app (Python)
      src/
        api/v1/                  # routers, by resource
        agents/                  # agent loop, prompts, typed Tool registry
        channels/                # channel adapters (email, sms, web, voice, ...)
        services/                # business logic (notify, billing, ...)
        db/                      # SQLAlchemy models, sessions, repositories
        integrations/            # vendor clients (llm_client, twilio, postmark, ...)
        lib/                     # in-house utilities (durable, vault, flags, ...)
        config.py                # pydantic-settings
        main.py                  # app factory + lifespan
      tests/
      pyproject.toml
      Dockerfile
    web/                         # Next.js (later)
    mobile/                      # Expo (later)
  packages/                      # shared libs (later)
  infra/                         # Terraform (later)
  .claude/
    settings.json
    skills/
  CLAUDE.md
```

## Hard rules (non-negotiable on every change)

These never go away. Skills go deeper; this list never gets shorter.

1. **Tenant context is mandatory.** Every service function takes a `TenantContext`. Every DB session is opened tenant-scoped (`SET app.tenant_id`) so Postgres RLS enforces isolation. Every log line is bound with `tenant_id`. Every LLM call is tagged with `tenant_id`. Every durable event carries `tenant_id` in the payload.
2. **No raw `WHERE tenant_id = ?` in app code.** RLS does that. If you're tempted to write it, the session isn't scoped correctly — fix that instead.
3. **No plaintext tenant secrets anywhere.** Cloud KMS envelope encryption, ciphertext in AlloyDB, fetched per-call via `CredentialVault`, used, dropped. Never logged. Never in LLM context. Never cached longer than the request.
4. **No PII in logs or traces by default.** Redact at the logger. Email/phone/full names go through a redactor before they hit any sink.
5. **Async-first.** Never call sync I/O (`requests`, sync DB drivers, `time.sleep`) inside an `async def`. It blocks the event loop and freezes every other in-flight request on the worker.
6. **Pydantic at every boundary.** Request bodies, response bodies, config, LLM tool args, channel messages, queue payloads, event payloads. No raw dicts crossing module lines.
7. **Errors typed, not strings.** Domain errors are typed exception classes per module. Map to HTTP at the API boundary, not in services.
8. **DIY utilities stay small.** Each lives behind a swap-ready interface (`LLMClient`, `DurableExecutor`, `Tool`, `NotifyDispatcher`, `FlagEvaluator`, `STTClient`, etc.). Resist adding configuration knobs, plugin hooks, or "in case we need it later" surface area. If a utility crosses ~500 LOC or 10 public symbols, reach for `in-house-utilities` skill — you're probably overbuilding.
9. **No new files when an edit will do.** No speculative abstractions. Three repeated lines beat a premature base class.

## Specialty skills

For deeper guidance on a topic, invoke the matching skill. CLAUDE.md keeps the rules above in context; skills load on demand with depth, do/don't lists, pitfalls, and verification steps.

| Topic | Skill |
|---|---|
| Statement-level Pythonic idioms vs Java-style | `pythonic-style` |
| Configuration, env, settings, feature flags | `python-configuration` |
| Tooling, dev workflow, uv/ruff/pyright/just | `python-developer-experience` |
| Logging, observability, structlog, redaction | `python-logging` |
| Errors, domain exceptions, HTTP mapping | `python-exception-handling` |
| Type system, Protocol, Generic, narrowing | `python-typing` |
| Tests, pytest, fixtures, real-DB testing | `python-testing` |
| Async, event loop, TaskGroup, cancellation | `python-concurrency` |
| Class vs function, Protocols, DI, SOLID-in-Python | `python-code-architecture` |
| Domain modeling, value objects, services | `python-domain-design` |
| FastAPI routers, deps, streaming, lifespan | `fastapi-design` |
| Postgres RLS, tenant scoping, session vars | `postgres-multi-tenant-rls` |
| KMS, Secret Manager, tenant credential vault | `secrets-and-credentials` |
| Durable executor patterns, idempotency, retries | `python-durable-execution` |
| LLM call observability, BigQuery sink, swap-readiness | `python-llm-observability` |
| Product event schema, naming, BigQuery shape | `product-event-tracking` |
| Keeping DIY utilities small | `in-house-utilities` |

When working on a task, check whether a skill applies before guessing. When multiple apply, invoke each as needed — they compose.

## Working agreements

- Don't add abstractions speculatively. Don't add backwards-compat shims, feature flags, or "in case we need it later" code. Change the thing.
- Don't write docstrings that restate the function name. Write them only when invariants, side effects, or units need to be stated.
- If you're about to write more than ~150 lines without showing a diff, stop and check in.
- If unsure between two designs, present both with the tradeoff. Don't pick silently.
- Verification before declaring done: `uv run ruff check .`, `uv run pyright`, `uv run pytest` all green. "Looks right" is not done.
