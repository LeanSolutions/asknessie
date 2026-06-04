# Bill-Payment Inbound Triage — MVP Scope & Plan

## Context

The first concrete product slice of asknessie. We pick one capability from the broader PA surface — **inbound bill-payment-comm triage** — and ship it end-to-end. It exercises every load-bearing piece of the architecture we designed (multi-tenant + RLS, durable execution, LLM client wrapper + observability, channel adapter pattern, KMS credential vault, structured events, notify dispatch) without taking on the breadth of a full assistant.

The slice: watch a user's Gmail inbox, classify each new message for bill-payment relevance + spam-likeness, log the spam decisions for a read-only UI, and send the user one coalesced email per 2-minute window summarizing legitimate bill comms.

Why this one:
- Forces real Gmail OAuth + tenant credential vault wiring (not a toy integration).
- Forces async durable execution (the watch → triage → notify chain doesn't fit in a request).
- Forces LLM observability (every triage decision is an `llm_calls` row; spam ratios become a dashboard).
- Forces RLS at scale-shape (every table in the feature is tenant-scoped from day one).
- Demonstrates "proactive monitoring" — one of the clear PA gaps in the industry.

User-selected scope (locked):
- **Channels:** Email only.
- **Inbound mechanism:** Gmail OAuth + `users.watch` + Pub/Sub push.
- **Spam feedback:** Read-only spam log; no user-side correction in MVP.
- **Notify cadence:** Per-event, coalesced over a 2-minute window per user.

Out of scope (deliberate, leaves clean swap-in interfaces):
- SMS / Slack / WhatsApp / voice / web chat ingress.
- Bill-payment automation; integrations with billers.
- "Unmark spam" / "report missed spam" feedback.
- LLM-written notification body (templated only).
- Full MIME storage in GCS (snippet + headers only).
- Cross-tenant admin views.
- Multi-user-per-tenant UX (schema carries `user_id` so it's a frontend lift later).

---

## Phasing

The repo has **zero application code today**. Phase 0 is prerequisite bootstrap; phases 1–3 are the feature. Each phase is independently shippable.

### Phase 0 — Bootstrap (prerequisite, separate PR)
Roughly: `apps/api/` skeleton with `pyproject.toml`, `.python-version`, `justfile`, `pre-commit-config.yaml`, `Dockerfile`, `src/config.py` (pydantic-settings), `src/main.py` (lifespan, exception handler, middleware, routers wired), `src/db/session.py` (engine, RLS-scoped session), `src/db/base.py` (declarative base + `tenant_id` mixin), `src/lib/durable.py` (Postgres-backed `DurableExecutor` per `python-durable-execution` skill), `src/lib/vault.py` (Cloud KMS envelope `CredentialVault`), `src/integrations/llm_client.py` (Vertex AI + Anthropic + cost log + Pub/Sub publish per `python-llm-observability`), `src/services/notify.py` (NotifyDispatcher with Postmark/Resend wrapper), `src/lib/events.py` (`EventTracker` + `EVENT_SCHEMAS` registry per `product-event-tracking`), WorkOS auth dependency, structlog config with PII redactor, Alembic initialized, `tests/conftest.py` with testcontainers Postgres + tenant_session fixture, a `/health` route, and a smoke `/v1/whoami` route as the first end-to-end proof. Estimate: ~3,500 LOC across foundation files. Verification: `just verify` green from clean clone; a request with a WorkOS token returns user + tenant.

**This plan assumes Phase 0 has shipped.** If it has not, do that first.

### Phase 1 — Gmail OAuth + ingest pipeline (this feature)
Wire the Gmail integration end-to-end. After this phase: a user can connect their Gmail, every new INBOX message lands in `inbound_messages`, but nothing classifies it yet.

### Phase 2 — Triage + spam log
Add the LLM classification durable function, the `bill_triage_decisions` and `bill_spam_log` tables, and the read-only spam-log API + Next.js page. After this phase: spam decisions are observable end-to-end.

### Phase 3 — Coalesced notifications
Add `pending_bill_notifications` (the coalescer) and the dispatch durable function. After this phase: users get a single email per 2-minute window listing legitimate bill comms.

### Phase 4 (deferred unless MVP slips past 7 days) — watch renewal + sweep wiring
Cloud Scheduler hits a tiny internal endpoint hourly to re-`watch` mailboxes whose `watch_expires_at < now() + 1d`, and every 30s to claim any `pending_bill_notifications` past `window_due_at` that the dispatch job didn't pick up.

---

## Module layout (new files added by phases 1–3)

Under `apps/api/src/`:

```
channels/email_inbound/
  __init__.py
  errors.py                  ~30 LOC   typed domain errors
  models.py                 ~120 LOC   PubSubPushEnvelope, GmailNotification,
                                       GmailMessageRef, GmailMessageContent,
                                       BillTriageDecision
  gmail_client.py           ~180 LOC   httpx wrapper around Gmail API
  pubsub_verifier.py         ~80 LOC   Google JWT verify + audience check
  triage.py                 ~120 LOC   pure orchestration; calls LLMClient
  prompts.py                 ~50 LOC   single classification prompt + schema
  repository.py             ~150 LOC   async SQLAlchemy repositories
services/
  email_inbound_service.py  ~120 LOC   ingest_pubsub_push, enroll_gmail
  bill_notification_service.py ~100 LOC queue_for_coalescing, send_window
  durable_functions/
    gmail_triage.py         ~150 LOC   process_history + triage_message
    bill_notifications.py    ~80 LOC   dispatch_pending_notification
api/v1/
  email_inbound.py          ~120 LOC   webhook + spam-log routes
  gmail_oauth.py             ~80 LOC   OAuth callback + rewatch + delete
db/models/
  email_inbound.py          ~180 LOC   SQLAlchemy models for new tables
alembic/versions/
  <rev>_email_inbound_triage.py ~250 LOC  all DDL + RLS + indexes
```

Frontend (Next.js, under `apps/web/`):

```
app/(authed)/spam/
  page.tsx                   ~80 LOC   server component; calls API; table UI
  components/SpamTable.tsx   ~60 LOC   client component; cursor pagination
```

**Total new code: ~2,200 LOC backend + ~140 LOC frontend.** Stays well under the "framework" bar per `in-house-utilities`.

No new utility under `lib/` is created. The coalescer is intentionally domain code, not a generic utility.

---

## Schema (single Alembic revision)

All tables: `tenant_id UUID NOT NULL`, `ENABLE ROW LEVEL SECURITY`, `FORCE ROW LEVEL SECURITY`, policy `USING (tenant_id = current_setting('app.tenant_id', true)::uuid) WITH CHECK (...)`, lead index on `(tenant_id, ...)`. Per `postgres-multi-tenant-rls`.

### `gmail_mailboxes`
```
id, tenant_id, user_id, email_address (CITEXT), google_account_sub,
credential_ref (vault handle), watch_history_id, watch_expires_at,
label_ids (TEXT[]), status ('active'|'paused'|'revoked'), timestamps
UNIQUE (tenant_id, email_address)
INDEX (tenant_id, status)
INDEX (email_address) WHERE status = 'active'   -- for the privileged push lookup
```

### `gmail_history_cursors`
`mailbox_id PK FK`, `tenant_id`, `last_history_id BIGINT`, `updated_at`.

### `inbound_messages`
```
id, tenant_id, user_id, mailbox_id FK,
gmail_message_id, gmail_thread_id,
from_address (CITEXT), from_name, subject, snippet,
received_at, raw_storage_uri (nullable — deferred),
triage_state ('pending'|'triaged'|'error'),
triaged_at, created_at
UNIQUE (tenant_id, gmail_message_id)   -- THE idempotency key
INDEX (tenant_id, received_at DESC)
INDEX (tenant_id, triage_state) WHERE triage_state = 'pending'
```

### `bill_triage_decisions`
```
id, tenant_id, inbound_message_id FK,
is_bill_related BOOL, is_spam BOOL,
bill_kind ('invoice'|'receipt'|'reminder'|'statement'|'other'|'none'),
confidence NUMERIC(4,3), reasoning TEXT,
llm_call_id FK -> llm_calls(id),   -- per python-llm-observability
created_at
INDEX (tenant_id, inbound_message_id)
```

### `bill_spam_log`
Written by the triage step when `is_spam=true`. Real table (not view) for RLS + pagination simplicity.
```
id, tenant_id, user_id, inbound_message_id FK,
from_address (CITEXT), subject, received_at, spam_reason, created_at
INDEX (tenant_id, user_id, received_at DESC)
```

### `pending_bill_notifications` (the coalescer)
```
id, tenant_id, user_id,
window_opened_at, window_due_at (= opened + 2 min),
items JSONB,    -- [{inbound_message_id, from, subject, received_at, bill_kind}, ...]
state ('open'|'sending'|'sent'|'failed'),
durable_job_id, sent_at, timestamps
UNIQUE (tenant_id, user_id) WHERE state = 'open'   -- race-safe coalescing
INDEX (tenant_id, state, window_due_at) WHERE state = 'open'
```

### RLS bypass — narrowly scoped, audited
The Pub/Sub push handler must look up `gmail_mailboxes` by `email_address` *before* it knows the tenant. We create one dedicated DB role `asknessie_ingestor` with `BYPASSRLS`, granted only on `gmail_mailboxes (id, tenant_id, user_id, status)`. Used by exactly one repository method `MailboxLookupRepository.find_active_by_email_address(email)`, which writes a row to a new `bypass_rls_access_log` table for every call: `(at, role, table_name, columns, lookup_key_hash, request_id)`. This is the only RLS exception in the feature.

---

## Endpoints (all under `/v1`)

| Method | Path | Auth | Purpose |
|---|---|---|---|
| POST | `/email/inbound/gmail/push` | Pub/Sub JWT (bearer) | Webhook for Gmail push notifications |
| POST | `/integrations/gmail/oauth/callback` | WorkOS session | Exchange OAuth code, store tokens, start watch |
| POST | `/integrations/gmail/rewatch` | WorkOS session | Idempotent re-watch (manual + future cron) |
| DELETE | `/integrations/gmail/{mailbox_id}` | WorkOS session | Revoke + stop watch + clear credentials |
| GET | `/email/bill-spam` | WorkOS session | Paginated read-only spam log |

Routes are thin handlers (per `fastapi-design`): parse → service → response model. All errors propagate as typed domain exceptions; the single exception handler in `main.py` maps them (per `python-exception-handling`). `response_model=` mandatory on every route.

---

## Durable functions

Per `python-durable-execution`. Use existing `DurableExecutor` surface only — no new methods.

### `email.gmail.process_history` (per-tenant concurrency = 1, retries = 5, exp+jitter)
Per-tenant concurrency 1 keeps the history cursor monotonic.
Steps: `load_mailbox_and_cursor` → `fetch_credentials` (vault, audited) → `list_history` (Gmail API) → `enqueue_triage_per_message` (one `triage_message` job per new message, idempotency key `triage:{mailbox_id}:{gmail_message_id}`) → `advance_cursor`.

### `email.gmail.triage_message` (per-tenant concurrency = 8, retries = 4)
Steps: `dedupe_check` (`INSERT ... ON CONFLICT DO NOTHING` on `(tenant_id, gmail_message_id)` — gap #1 closed) → `fetch_message` (Gmail API, snippet + headers) → `classify` (single `LLMClient.complete` call with `purpose="email.bill_triage"`) → `persist_decision` (write `bill_triage_decisions`; if spam, write `bill_spam_log`; mark `inbound_messages.triage_state='triaged'`) → `route_decision` (if bill-related + not-spam: call `queue_for_coalescing`; if newly-opened window, enqueue dispatch with `delay_until=window_due_at`) → `emit_events`.

### `email.bill.dispatch_pending_notification` (per-tenant concurrency = 4, retries = 3)
Steps: `claim_window` (atomic `state='open' -> 'sending'` UPDATE; exit if not claimed) → `render_email` (templated; one row per item) → `send_email` (existing `NotifyDispatcher.send_email`) → `mark_sent` → `emit_event`.

### Sweep / watch renewal (Phase 4, Cloud Scheduler)
Two internal POST endpoints (`/internal/sweep-notifications`, `/internal/gmail/renew-watches`) hit by Cloud Scheduler. Each is idempotent. Keeps `DurableExecutor` surface untouched (per `in-house-utilities` — `cron=` arg not introduced).

---

## Coalescing design (2-minute window, race-safe)

1. **First bill comm for user U.** `queue_for_coalescing` runs:
   ```sql
   INSERT INTO pending_bill_notifications (id, tenant_id, user_id,
       window_opened_at, window_due_at, items, state)
   VALUES (gen_random_uuid(), :t, :u, now(), now() + interval '2 minutes',
           jsonb_build_array(:item), 'open')
   ON CONFLICT (tenant_id, user_id) WHERE state = 'open'
   DO UPDATE SET items = pending_bill_notifications.items || EXCLUDED.items,
                 updated_at = now()
   RETURNING id, window_due_at, (xmax = 0) AS was_inserted;
   ```
2. `was_inserted = true` → enqueue `dispatch_pending_notification` with `delay_until = window_due_at` and `idempotency_key = f"dispatch:{pending_notification_id}"`.
3. **Subsequent bill comm at T+45s.** Same SQL; `xmax=0` is false (UPDATE path); `items` array grows; no new dispatch enqueued.
4. **Dispatch fires at T+2min.** `claim_window` atomically flips `open → sending`. Sends one email listing all items. Marks `sent`.
5. **Bill comm at T+2min+5s.** Partial unique is gone (row is `sent`). New INSERT opens a fresh window.

Race safety comes from the partial unique index + atomic UPDATE. No Redis, no in-memory state, no locks.

---

## LLM classification

- **One call, not two.** Bill-shape + sender-mismatch is the spam signal — splitting would double cost and create inconsistent decisions.
- **Model:** Gemini 2.5 Flash via Vertex AI (CLAUDE.md routing for side tasks).
- **Surface:** existing `LLMClient.complete(...)` (or `complete_structured` if already present). JSON-mode prompt + Pydantic validation of `BillTriageDecision`. **No new method on `LLMClient`** (per `in-house-utilities`).
- **Purpose tag:** `"email.bill_triage"` — discriminator for cost dashboards and BigQuery analysis.
- **Input fields:** `from, subject, received_at, snippet, plaintext_body (truncated 4KB), list_unsubscribe_header_present`.
- **Output schema (Pydantic):**
  ```
  is_bill_related: bool
  bill_kind: Literal['invoice','receipt','reminder','statement','other','none']
  is_spam: bool
  confidence: float
  reasoning: str   # one sentence, used as spam_reason
  ```
- **Prompt caching** (per CLAUDE.md): system prompt + schema are stable → cached. Handled by `LLMClient`.
- **One `llm_calls` row + one Pub/Sub event per triage**, automatic via the wrapper. `bill_triage_decisions.llm_call_id` foreign-keys to it.

---

## Events emitted (added to `EVENT_SCHEMAS`)

| Event | Trigger | Notable payload fields |
|---|---|---|
| `gmail_mailbox.enrolled` | OAuth callback success | `mailbox_id, email_domain (not address), enrolled_at` |
| `gmail_push.received` | Every accepted push | `mailbox_id, history_id, message_count_estimate` |
| `bill_triage.evaluated` | `persist_decision` step done | `inbound_message_id, is_bill_related, is_spam, bill_kind, confidence, from_domain, llm_call_id` |
| `bill_spam.logged` | When `is_spam=true` | `inbound_message_id, from_domain, spam_reason` |
| `bill_notification.coalesced` | Second+ item joins open window | `pending_notification_id, item_count` |
| `bill_notification.sent` | Dispatch step completes | `pending_notification_id, item_count, window_duration_seconds` |

All conform to `EventEnvelope` (per `product-event-tracking`). Server-side ingestion only. Email domains, not addresses (PII).

---

## Cuts vs full version (interfaces preserved)

- Full MIME in GCS — `inbound_messages.raw_storage_uri` column reserved.
- User-side spam correction — `bill_triage_decisions` is append-only; future `bill_triage_overrides` table without touching it.
- Multi-user-per-tenant UX — schema carries `user_id` everywhere; pure frontend lift.
- Auto watch-renewal — endpoint exists; Cloud Scheduler wiring is Phase 4.
- LLM-written notification body — `render_email` is the swap point; templated for MVP.
- Other channels — `NotifyDispatcher.send_email` is the only surface used; future channels don't touch this domain.
- Cross-tenant admin view — RLS denies; future admin role gets `BYPASSRLS` on a service account, not a schema change.

---

## Surface-expansion audit (vs `in-house-utilities`)

| Utility | New surface in this feature | Verdict |
|---|---|---|
| `LLMClient` | none (uses existing `complete` / `complete_structured`) | OK |
| `DurableExecutor` | uses existing `function`, `step`, `delay_until`; Cloud Scheduler covers the `cron=` need | OK |
| `NotifyDispatcher` | uses existing `send_email` | OK |
| `CredentialVault` | uses existing `store` / `get` / `delete` + existing audit | OK |
| `EventTracker` | six new event names in `EVENT_SCHEMAS` (data, not surface) | OK |
| `BypassRlsRepository` (new) | one method, one table, one tiny audit-row helper | OK — narrowly scoped, audited |

No DIY utility grows. The coalescer is intentionally domain code, not a new `lib/`.

---

## Verification

### CI (automated)
- **Unit:** `pubsub_verifier` against fixture JWKS. `triage.classify` with canned `BillTriageDecision` JSONs (mock `LLMClient`). `bill_notification_service.queue_for_coalescing` against testcontainers Postgres — three scenarios: cold open, second-item join, third-item-after-send opens fresh window. Validate the partial unique index actually fires under concurrency.
- **Repository:** `inbound_messages` dedupe via `(tenant_id, gmail_message_id)` — same insert twice produces one row. RLS test: session with `app.tenant_id = A` cannot see rows for tenant B.
- **BYPASSRLS:** the lookup method writes the audit row on every call; a test asserts an attempt to read other columns from the privileged role raises a permission error.
- **Durable steps:** memoization test — run `triage_message` twice with the same idempotency key, assert one set of side effects.
- **End-to-end (in-process):** fixture push payload → router → in-test executor → assertions on rows + on `NotifyDispatcher` invocation.
- **Coalescing window:** push two synthetic messages 30s apart (test-clock), assert exactly one notification email sent containing both items.

### Manual (dev environment)
- Enroll a real Gmail test account via the OAuth callback.
- Send yourself a real invoice from a known biller domain → notification email arrives ~2 min later.
- Send yourself a phishy "you owe $500" from a freemail domain → lands in `bill_spam_log`, no notification.
- Revoke Google access in the user's Google account settings → next push fails auth; mailbox flips to `revoked`.

### Production smoke
- Pub/Sub dead-letter topic on the subscription; alert on DLQ depth > 0.
- Log alert: `bill_triage.evaluated` event count drops to 0 for 15 min during business hours.
- LLM cost dashboard filtered by `purpose='email.bill_triage'`.
- Latency SLO: p50 push-receive → user-notification ≤ 2 min 30 s, p95 ≤ 5 min.

---

## Critical files to modify / create

Phase-1 / Phase-2 / Phase-3 backend:
- `apps/api/alembic/versions/<rev>_email_inbound_triage.py`
- `apps/api/src/channels/email_inbound/{models,errors,gmail_client,pubsub_verifier,triage,prompts,repository}.py`
- `apps/api/src/services/{email_inbound_service,bill_notification_service}.py`
- `apps/api/src/services/durable_functions/{gmail_triage,bill_notifications}.py`
- `apps/api/src/api/v1/{email_inbound,gmail_oauth}.py`
- `apps/api/src/db/models/email_inbound.py`
- `apps/api/src/db/bypass_rls.py` (the new `BypassRlsRepository` + audit helper)
- `apps/api/src/lib/events.py` — add six `EVENT_SCHEMAS` entries
- `apps/api/src/main.py` — register new routers
- `apps/api/src/config.py` — add Gmail / Pub/Sub / Postmark / Resend settings

Frontend:
- `apps/web/app/(authed)/spam/page.tsx`
- `apps/web/app/(authed)/spam/components/SpamTable.tsx`

Infrastructure (Terraform, deferred to a separate IaC PR):
- Pub/Sub topic `gmail-inbound` + push subscription pointing at `POST /v1/email/inbound/gmail/push`.
- Pub/Sub dead-letter topic + alert.
- Cloud KMS key (`tenant-secrets`, if not from Phase 0).
- Cloud Scheduler jobs (Phase 4).
- DB roles `asknessie_app` (default app role, RLS-subject) and `asknessie_ingestor` (`BYPASSRLS` on the mailbox-lookup view only).

---

## Open risks to acknowledge

1. **Watch expires every 7 days.** If MVP ships and is in production for 8 days without Phase 4, every mailbox silently goes dark. Phase 4 must ship within the first week.
2. **Pub/Sub push retries.** Idempotency is guarded by `(tenant_id, gmail_message_id)`; we still need to monitor DLQ for poison-pill messages.
3. **Gmail API quotas.** Per-user limits are generous, but a heavy first-sync (many history entries on enrollment) can hit them. Watch on `INBOX` label only and `historyTypes=messageAdded` to keep volume low.
4. **LLM cost spikes on noisy mailboxes.** Per-tenant rate limit on `email.bill_triage` purpose in `LLMClient` is a Phase 4 add if costs become a concern.
5. **OAuth refresh failure.** If a refresh fails (user revoked offline), mailbox flips to `revoked` and stops triaging. UI should surface this — frontend MVP doesn't yet; add to backlog.
