# asknessie

Multi-tenant SaaS personal-assistant platform. A user can reach the assistant via web UI, mobile, email, SMS, voice (in-app and phone), and Slack. The brain is one durable agent; the channels are pluggable adapters.

## Stack

**Backend:** Python 3.13, FastAPI, Pydantic v2, SQLAlchemy 2.0 (async) + asyncpg, Alembic, Claude Agent SDK (Python), Inngest, structlog.
**Frontend:** Next.js 15 (web), Expo (mobile). Both TypeScript.
**Infra:** GCP — Cloud Run, AlloyDB, Memorystore Redis, Cloud KMS, Secret Manager, Pub/Sub, FCM, Vertex AI (for Anthropic).
**Vendors:** Twilio (SMS/voice), LiveKit (in-app voice), Postmark (email in) + Resend (email out), WorkOS (auth), Stripe (billing), Langfuse (LLM tracing), Sentry (errors).

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
        agents/                  # agent loop, prompts, tools
        channels/                # channel adapters (email, sms, web, voice, ...)
        services/                # business logic
        db/                      # SQLAlchemy models, sessions, repositories
        integrations/            # vendor clients (anthropic, twilio, postmark, ...)
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
    skills/                      # specialty skills, see below
  CLAUDE.md
```

## Hard rules (non-negotiable on every change)

These never go away. Skills go deeper; this list never gets shorter.

1. **Tenant context is mandatory.** Every service function takes a `TenantContext` (tenant_id, user_id, plan, limits). Every DB session is opened tenant-scoped (`SET app.tenant_id`) so Postgres RLS enforces isolation. Every log line is bound with `tenant_id`. Every LLM call is tagged with `tenant_id`. Every background event carries `tenant_id` in the payload.
2. **No raw `WHERE tenant_id = ?` in app code.** RLS does that. If you're tempted to write it, the session isn't scoped correctly — fix that instead.
3. **No plaintext tenant secrets anywhere.** OAuth tokens, API keys, etc. are envelope-encrypted via Cloud KMS, stored as ciphertext, fetched per-call, used, dropped. Never logged. Never in LLM context. Never cached longer than the request.
4. **No PII in logs or traces by default.** Redact at the logger. Email/phone/full names go through a redactor before they hit any sink.
5. **Async-first.** Never call sync I/O (`requests`, sync DB drivers, `time.sleep`) inside an `async def`. It blocks the event loop and freezes every other in-flight request on the worker.
6. **Pydantic at every boundary.** Request bodies, response bodies, config, LLM tool args, channel messages, queue payloads. No raw dicts crossing module lines.
7. **Errors typed, not strings.** Domain errors are typed exception classes per module. Map to HTTP at the API boundary, not in services.
8. **No new files when an edit will do.** No speculative abstractions. Three repeated lines beat a premature base class.

## Specialty skills

For deeper guidance on a topic, invoke the matching skill. CLAUDE.md keeps the rules above always in context; skills load on demand with depth, do/don't lists, pitfalls, and verification steps.

| Topic | Skill |
|---|---|
| Configuration, env, settings, feature flags | `python-configuration` |
| Tooling, dev workflow, uv/ruff/pyright/just | `python-developer-experience` |
| Logging, observability, structlog, redaction | `python-logging` |
| Errors, domain exceptions, HTTP mapping | `python-exception-handling` |
| Type system, Protocol, Generic, narrowing | `python-typing` |
| Tests, pytest, fixtures, real-DB testing | `python-testing` |
| Async, event loop, TaskGroup, cancellation | `python-concurrency` |
| Class vs function, Protocols, DI, SOLID-in-Python | `python-code-organization` |
| Domain modeling, value objects, services | `python-domain-design` |
| FastAPI routers, deps, streaming, lifespan | `fastapi-design` |
| Postgres RLS, tenant scoping, session vars | `postgres-multi-tenant-rls` |
| KMS, Secret Manager, tenant token vault | `secrets-and-credentials` |

When working on a task, check whether a skill applies before guessing. When multiple apply, invoke each as needed — they compose.

## Working agreements

- Don't add abstractions speculatively. Don't add backwards-compat shims, feature flags, or "in case we need it later" code. Change the thing.
- Don't write docstrings that restate the function name. Write them only when invariants, side effects, or units need to be stated.
- If you're about to write more than ~150 lines without showing a diff, stop and check in.
- If unsure between two designs, present both with the tradeoff. Don't pick silently.
- Verification before declaring done: `uv run ruff check .`, `uv run pyright`, `uv run pytest` all green. "Looks right" is not done.
