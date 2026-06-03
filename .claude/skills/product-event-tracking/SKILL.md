---
name: product-event-tracking
description: Use when defining or naming product events, designing the events schema, building the ingestion endpoint, configuring the BigQuery sink, or deciding which signals to track. Use whenever the question "should I track this?" or "what do I call this event?" comes up — backend or frontend.
---

# product-event-tracking

## When to use this skill

Triggered for any product analytics work: adding an event, naming an event, designing the properties for an event, designing the ingestion endpoint, the Pub/Sub → BigQuery pipeline, the dashboards. Also during review — every new event is a long-lived contract; getting the name and shape right matters more than getting them fast.

## Core principles

1. **Events are nouns_verbed in past tense.** `thread_created`, `message_sent`, `agent_step_completed`. Snake-case. Past tense because the event already happened.
2. **Five mandatory fields on every event.** `event`, `user_id`, `tenant_id`, `timestamp`, `properties`. Plus `event_id` for idempotency. If any is missing at ingest, reject.
3. **`properties` is JSONB but disciplined.** Each event class has a typed Pydantic schema for its properties. Schema discipline beats schema flexibility for queries.
4. **One row per event in BigQuery.** Same schema for every event; `properties` carries the per-event detail. Tools like Looker Studio prefer one wide table.
5. **PII redacted at ingest.** Same redactor used by logging and LLM observability. No raw email/phone/secret values in events.
6. **Server-side ingest only.** Frontend SDK posts to our `/events` endpoint with the user's session. Server validates, enriches (geo, version), publishes. No direct frontend → BigQuery / PostHog.
7. **Track the few that matter.** Resist the urge to track every click. A handful of business-meaningful events beats fifty noisy ones. Add events when you'd actually query them.
8. **Stable forever.** An event name, once shipped, is a contract. Renaming requires dual-emit + cutover + cleanup. Plan names like API endpoints.

## Always

### The event envelope

```python
from pydantic import BaseModel, Field, ConfigDict
from uuid import UUID, uuid4
from datetime import datetime
from typing import Any

class EventEnvelope(BaseModel):
    model_config = ConfigDict(frozen=True)
    event_id: UUID = Field(default_factory=uuid4)        # idempotency
    event: str                                            # 'thread_created'
    tenant_id: UUID
    user_id: UUID | None                                  # null for system events
    timestamp: datetime                                   # client-provided; server may overwrite if skewed
    properties: dict[str, Any]                            # validated against event-specific schema
    context: EventContext

class EventContext(BaseModel):
    model_config = ConfigDict(frozen=True)
    source: Literal["web", "mobile", "api", "backend", "agent"]
    app_version: str | None = None
    user_agent: str | None = None
    ip_country: str | None = None                         # enriched server-side
    request_id: str | None = None
```

### Typed properties per event class

```python
# events/threads.py
class ThreadCreatedProperties(BaseModel):
    thread_id: UUID
    channel: Literal["web", "mobile", "email", "sms", "voice", "slack"]
    initial_message_length: int

class MessageSentProperties(BaseModel):
    thread_id: UUID
    message_id: UUID
    role: Literal["user", "assistant", "system"]
    char_count: int
    has_attachment: bool

# Registry: name → schema
EVENT_SCHEMAS: dict[str, type[BaseModel]] = {
    "thread_created": ThreadCreatedProperties,
    "message_sent": MessageSentProperties,
    "agent_step_completed": AgentStepCompletedProperties,
    # ...
}
```

The registry is the source of truth. New event = new schema entry + the code that emits it. Pyright catches misnamed properties; the ingest endpoint validates against the registered schema before publishing.

### Ingest endpoint

```python
@router.post("/events", status_code=202)
async def ingest_events(
    payload: BatchEventsIn,
    ctx: TenantDep,
    publisher: PubsubDep,
    request: Request,
) -> None:
    for envelope in payload.events:
        if envelope.tenant_id != ctx.tenant_id:
            raise CrossTenantEventRejected(actor=ctx.tenant_id, claimed=envelope.tenant_id)

        schema = EVENT_SCHEMAS.get(envelope.event)
        if schema is None:
            raise UnknownEvent(name=envelope.event)
        try:
            schema.model_validate(envelope.properties)
        except ValidationError as err:
            raise EventValidationFailed(event=envelope.event, errors=err.errors()) from err

        enriched = envelope.model_copy(update={
            "context": envelope.context.model_copy(update={
                "ip_country": geo_for(request.client.host),
                "request_id": current_request_id(),
            }),
            "properties": redact(envelope.properties),
        })
        await publisher.publish("product_events", enriched.model_dump(mode="json"))
```

Server validates, enriches, publishes. Frontend never reaches BigQuery; this is the only ingest path.

### BigQuery table

```sql
-- BigQuery DDL (run via Terraform)
CREATE TABLE analytics.product_events (
    event_id     STRING NOT NULL,
    event        STRING NOT NULL,
    tenant_id    STRING NOT NULL,
    user_id      STRING,
    timestamp    TIMESTAMP NOT NULL,
    received_at  TIMESTAMP NOT NULL,
    source       STRING NOT NULL,
    app_version  STRING,
    ip_country   STRING,
    request_id   STRING,
    properties   JSON NOT NULL
)
PARTITION BY DATE(timestamp)
CLUSTER BY tenant_id, event;
```

Partition by day (most queries are time-bounded). Cluster by `(tenant_id, event)` because per-tenant per-event queries are the common access pattern.

### Pub/Sub → BigQuery subscription

```
Topic:       product_events
Subscription: product_events_to_bq
Sink:        analytics.product_events
```

Use the BigQuery subscription type so Pub/Sub writes rows directly — no Dataflow job, no custom worker.

### Event naming convention

| Style | Use? | Example |
|---|---|---|
| `noun_verbed` past tense, snake_case | **Yes** | `thread_created`, `message_sent`, `user_signed_in` |
| `noun_noun_verbed` for compound | Yes | `agent_step_completed`, `tool_call_succeeded` |
| `verb_noun` | No | `create_thread` (sounds like an action, not a fact) |
| Sentence case | No | `User Created Thread` (UI-shaped, not data-shaped) |
| Generic event with a `type` property | No | `event: "click"`, `properties.type: "button"` (un-queryable) |
| Numeric suffix | No | `button_clicked_2`, `signup_v3` (lazy versioning) |

If you need to version, say so: `signup_completed` → `signup_completed_v2` is honest. Even better: keep the name, evolve the schema with backward-compatible additions.

### Properties naming convention

- `snake_case` keys.
- Use the noun describing the thing: `thread_id`, not `threadId` or `id`.
- Use units in the name when ambiguous: `latency_ms`, `cost_micros`, `duration_seconds`.
- Booleans named as questions: `has_attachment`, `is_first_session`, `was_retried`.
- No deeply nested objects in `properties` — flat is easier to query.

### Emission helper

```python
class EventTracker:
    def __init__(self, publisher: PubsubPublisher) -> None:
        self._pub = publisher

    async def track[P: BaseModel](
        self,
        event: str,
        properties: P,
        *,
        ctx: TenantContext,
        source: str = "backend",
    ) -> None:
        schema = EVENT_SCHEMAS.get(event)
        if schema is None or not isinstance(properties, schema):
            raise UnknownEvent(name=event)
        envelope = EventEnvelope(
            event=event,
            tenant_id=ctx.tenant_id,
            user_id=ctx.user_id,
            timestamp=utcnow(),
            properties=redact(properties.model_dump(mode="json")),
            context=EventContext(source=source, request_id=current_request_id()),
        )
        await self._pub.publish("product_events", envelope.model_dump(mode="json"))
```

Backend code calls `tracker.track("thread_created", ThreadCreatedProperties(...), ctx=ctx)`. Pyright catches schema mismatch at the call site.

### Idempotency

Frontend includes `event_id`. BigQuery sink dedupes via a scheduled materialized view or a `DISTINCT ON (event_id)` view. Pub/Sub has at-least-once delivery; dedup belongs downstream.

## Never

- Sentence-case event names like `"User Created Thread"`. *Why:* not queryable, ugly in SQL, doesn't sort.
- Free-form `properties` without a registered schema. *Why:* the JSONB becomes garbage in 6 months. No one writes the queries against it.
- Adding events you wouldn't actually query. *Why:* event sprawl makes the dataset less useful. Track the few that matter.
- PII in event names or top-level properties. *Why:* the name is logged everywhere; properties get aggregated in dashboards. `properties.email = "..."` leaks anywhere a dashboard renders.
- Direct frontend → BigQuery / PostHog. *Why:* no server-side validation, no enrichment, no rate-limiting, no audit. Server ingest is the only path.
- Tracking the same fact under two names. *Why:* dashboards diverge silently. One name, one event.
- Renaming events without a dual-emit window. *Why:* analytics history breaks. Emit both old and new for ~30 days, cut over dashboards, then stop the old.
- Hard-deleting events. *Why:* loss of historical context. Stop emitting; keep the data.
- Storing the same field under two key names. *Why:* `user_id` vs `userId` vs `uid` across events makes joins painful. Use the canonical name everywhere.
- Logging events through the general logger. *Why:* two pipelines, two retention policies, two redaction sources. Events go through the tracker; nothing else.
- Skipping `event_id`. *Why:* Pub/Sub at-least-once delivery means duplicates. Without `event_id`, your funnel counts are wrong.
- Emitting events from migrations or seed scripts. *Why:* pollutes the dataset with non-user activity. Tag system events with `source: "backend"` and `user_id: null` if needed; *never* emit during data setup.
- Treating `properties.type` as the actual event. *Why:* you lose first-class queryability. Make it a real event name.

## Pitfalls

- **Client-skewed timestamps.** A mobile client with a wrong clock sends events from "yesterday." Trust client timestamp for ordering within a session, but also store `received_at` for analytical truth.
- **Schema evolution breaking old rows.** Add fields freely (BigQuery JSONB handles this); never rename or repurpose. If you must, dual-emit and migrate dashboards.
- **High-cardinality property values.** Property values that are unique per event (UUIDs, free text) blow up clustering benefit. Keep them for joins, don't try to aggregate on them.
- **Tracker called from a hot loop without batching.** Per-event publish is fine at our scale; if rate balloons, add a buffered tracker that flushes per second.
- **`event_id` reused across retries by mistake.** Each event must have a fresh `event_id`. Don't UUID-from-content hash and assume idempotency at content level — different attempts can produce different content.
- **PII in `request_id`.** Some frameworks include user-controlled paths or query params. Keep `request_id` opaque (UUID, not URL).
- **Geo enrichment leaking IP.** Compute `ip_country`, drop the IP. Never store full IP in events.
- **BigQuery cost from `SELECT *` on `properties JSON`.** Query just the properties you need: `JSON_VALUE(properties, '$.thread_id')`. `SELECT *` charges for everything.
- **Schema drift between frontend and backend SDK.** Single source of truth: the registry. Generate TypeScript types from the Pydantic schemas, or maintain a small DSL.
- **Event explosion (one event per UI interaction).** Make events meaningful at the *business* level. "User scrolled past thread #5" is not a meaningful event; "thread_archived" is.
- **Mixing dev/staging/prod events into the same BigQuery table.** Use an `env` field or separate tables. Otherwise dashboards are noisy and rate limits are hard.
- **Forgetting tenant scope on dashboards.** Looker Studio queries that aren't tenant-filtered show cross-tenant data to ops staff. Either bind tenant filter to the dashboard or restrict via authorized views.

## Verification

Before declaring event-tracking work done:

- Every event sent has a registered schema in `EVENT_SCHEMAS`.
- A test asserts unknown event names are rejected at ingest with a 422.
- A test asserts a malformed `properties` payload is rejected (schema validation).
- A test asserts `redact()` strips emails/phones in property values before publish.
- A test asserts cross-tenant events (envelope `tenant_id != ctx.tenant_id`) are rejected.
- BigQuery integration test (or staging-only): a tracked event shows up in `analytics.product_events` within ~60s.
- A dashboard exists in Looker Studio for at least one core funnel (e.g., `thread_created` → `message_sent` → `agent_step_completed`).
- The tracker is the only caller of `publisher.publish("product_events", ...)` — verify with grep.
- Every property uses snake_case keys; no `camelCase` slips through (lint or test).
- Event naming convention is documented in this skill and grep confirms no `verb_noun` style events leaked in.
- Dedup logic in BigQuery view tested: emitting the same event twice yields one row.
- A per-tenant per-event dashboard query runs in <2s on a year of data (validates partitioning + clustering).
