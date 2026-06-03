---
name: python-llm-observability
description: Use when instrumenting LLM calls, designing the llm_calls schema, wiring BigQuery sinks, adding OpenTelemetry span trees around agent loops, redacting prompts/completions, or planning the swap-in for Langfuse later. Use whenever a Claude or Gemini call is being made, wrapped, or analyzed.
---

# python-llm-observability

## When to use this skill

Triggered for any LLM-call instrumentation: building the `LLMClient` wrapper, defining the `llm_calls` schema, wiring the BigQuery sink, adding OpenTelemetry spans around an agent step, redacting prompt fields, designing retention policy. Also during review — every LLM call site is a tracing decision.

## Core principles

1. **Every LLM call writes exactly one row.** `llm_calls` (Postgres) for hot lookup; mirrored to BigQuery via Pub/Sub for analytics. Same shape both places.
2. **Schema is Langfuse-shaped.** Field names and types mirror Langfuse so the future swap is one importer, not a re-instrumentation.
3. **OpenTelemetry spans wrap calls.** Cloud Trace gets the agent → LLM → tool tree. Span IDs are persisted on the row so a trace ties to its row.
4. **Token counts come from the API response, not estimates.** Anthropic returns exact `input_tokens` / `output_tokens` / `cache_creation` / `cache_read` — use those.
5. **Cost is computed and stored.** Per-model rates table. Cost is in the row, not derived at query time — model prices change.
6. **Prompts and completions are redacted before persisting.** PII redactor sees them before they hit the row.
7. **Per-tenant retention policy.** Default 90 days for full content; aggregate counters retained longer. Tenant compliance asks (HIPAA, regional residency) override.
8. **Streaming: one row at the end.** Per-token spans, one final row with totals.

## Always

### `llm_calls` schema

```sql
CREATE TABLE llm_calls (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id           UUID NOT NULL REFERENCES tenants(id),
    user_id             UUID,                              -- nullable for system calls
    thread_id           UUID,
    request_id          TEXT,                              -- ties to API request
    trace_id            TEXT NOT NULL,                     -- OpenTelemetry trace id
    span_id             TEXT NOT NULL,                     -- this LLM call's span
    parent_span_id      TEXT,                              -- agent step that produced this call

    -- The call
    provider            TEXT NOT NULL,                     -- 'anthropic' | 'google'
    model               TEXT NOT NULL,                     -- 'claude-sonnet-4-6' | 'gemini-2.5-flash'
    purpose             TEXT NOT NULL,                     -- 'agent_step' | 'voice_ack' | 'classify' | ...

    -- Tokens (from API response)
    input_tokens        INT NOT NULL,
    output_tokens       INT NOT NULL,
    cache_creation_tokens INT NOT NULL DEFAULT 0,
    cache_read_tokens   INT NOT NULL DEFAULT 0,

    -- Computed at call time
    cost_micros         BIGINT NOT NULL,                   -- millionths of USD; integer math

    -- Timing
    started_at          TIMESTAMPTZ NOT NULL,
    completed_at        TIMESTAMPTZ NOT NULL,
    latency_ms          INT NOT NULL,
    ttfb_ms             INT,                               -- time to first byte (streaming)

    -- Content (redacted)
    prompt              JSONB NOT NULL,                    -- system + messages + tool defs
    completion          JSONB NOT NULL,                    -- model output, tool calls
    tools_called        JSONB NOT NULL DEFAULT '[]'::jsonb,

    -- Outcome
    finish_reason       TEXT,                              -- 'end_turn' | 'tool_use' | 'max_tokens' | 'error'
    error               JSONB,                             -- null on success

    -- Metadata
    extra               JSONB NOT NULL DEFAULT '{}'::jsonb -- room for future fields without migration
);

CREATE INDEX llm_calls_by_tenant_started ON llm_calls (tenant_id, started_at DESC);
CREATE INDEX llm_calls_by_thread ON llm_calls (thread_id) WHERE thread_id IS NOT NULL;
CREATE INDEX llm_calls_by_trace ON llm_calls (trace_id);

ALTER TABLE llm_calls ENABLE ROW LEVEL SECURITY;
ALTER TABLE llm_calls FORCE ROW LEVEL SECURITY;
CREATE POLICY tenant_isolation ON llm_calls
    USING (tenant_id = current_setting('app.tenant_id', true)::uuid)
    WITH CHECK (tenant_id = current_setting('app.tenant_id', true)::uuid);
```

Field naming mirrors Langfuse (their `Generation` model): `model`, `input`, `output`, `usage` fields. Swapping = mapping our rows to their import API.

### The LLM client wrapper writes the row

```python
class LLMClient:
    def __init__(self, anthropic: AnthropicClient, gemini: GeminiClient, db: AsyncSession, publisher: PubsubPublisher) -> None:
        self._anthropic = anthropic
        self._gemini = gemini
        self._db = db
        self._pub = publisher

    async def complete(
        self,
        *,
        ctx: TenantContext,
        purpose: LLMPurpose,
        messages: list[Message],
        tools: list[ToolDef] | None = None,
        thread_id: ThreadId | None = None,
    ) -> Completion:
        model = route_model(purpose)                                # see LLM routing in CLAUDE.md
        provider = provider_for(model)
        tracer = trace.get_tracer(__name__)

        started = utcnow()
        with tracer.start_as_current_span("llm.call", attributes={
            "llm.provider": provider,
            "llm.model": model,
            "llm.purpose": purpose,
            "tenant.id": str(ctx.tenant_id),
        }) as span:
            try:
                resp = await self._call(provider, model, messages, tools)
            except Exception as err:
                span.record_exception(err)
                await self._record_row(ctx, purpose, model, provider, messages, tools, started, error=err, span=span, thread_id=thread_id)
                raise

        await self._record_row(ctx, purpose, model, provider, messages, tools, started, resp=resp, span=span, thread_id=thread_id)
        return resp

    async def _record_row(self, ctx, purpose, model, provider, messages, tools, started, *, resp=None, error=None, span, thread_id):
        completed = utcnow()
        usage = getattr(resp, "usage", None)
        row = LLMCallRow(
            tenant_id=ctx.tenant_id,
            user_id=ctx.user_id,
            thread_id=thread_id,
            request_id=current_request_id(),
            trace_id=format(span.get_span_context().trace_id, "032x"),
            span_id=format(span.get_span_context().span_id, "016x"),
            parent_span_id=current_parent_span_id(),
            provider=provider,
            model=model,
            purpose=purpose,
            input_tokens=getattr(usage, "input_tokens", 0),
            output_tokens=getattr(usage, "output_tokens", 0),
            cache_creation_tokens=getattr(usage, "cache_creation_input_tokens", 0),
            cache_read_tokens=getattr(usage, "cache_read_input_tokens", 0),
            cost_micros=compute_cost_micros(model, usage),
            started_at=started,
            completed_at=completed,
            latency_ms=int((completed - started).total_seconds() * 1000),
            prompt=redact(_serialize_prompt(messages, tools)),
            completion=redact(_serialize_completion(resp)) if resp else {},
            tools_called=_tools_called(resp),
            finish_reason=getattr(resp, "stop_reason", None),
            error=_serialize_error(error) if error else None,
        )
        self._db.add(row)
        await self._db.flush()
        await self._pub.publish("llm_calls", row.to_event_dict())
```

The row is written *whether the call succeeded or failed* — failures are first-class data, not absent.

### Cost computation

```python
@dataclass(frozen=True)
class ModelRates:
    input_micros_per_mtok: int        # micros (1/1_000_000 USD) per 1M tokens
    output_micros_per_mtok: int
    cache_write_micros_per_mtok: int
    cache_read_micros_per_mtok: int

MODEL_RATES: dict[str, ModelRates] = {
    "claude-haiku-4-5":  ModelRates(1_000_000, 5_000_000, 1_250_000, 100_000),
    "claude-sonnet-4-6": ModelRates(3_000_000, 15_000_000, 3_750_000, 300_000),
    "claude-opus-4-7":   ModelRates(15_000_000, 75_000_000, 18_750_000, 1_500_000),
    "gemini-2-5-flash":  ModelRates(300_000, 1_200_000, 75_000, 75_000),
    "gemini-2-5-pro":    ModelRates(1_250_000, 10_000_000, 312_500, 312_500),
}

def compute_cost_micros(model: str, usage) -> int:
    r = MODEL_RATES[model]
    in_tok = (usage.input_tokens - getattr(usage, "cache_read_input_tokens", 0)) if usage else 0
    out_tok = usage.output_tokens if usage else 0
    cache_create = getattr(usage, "cache_creation_input_tokens", 0) if usage else 0
    cache_read = getattr(usage, "cache_read_input_tokens", 0) if usage else 0
    return (
        in_tok * r.input_micros_per_mtok // 1_000_000
        + out_tok * r.output_micros_per_mtok // 1_000_000
        + cache_create * r.cache_write_micros_per_mtok // 1_000_000
        + cache_read * r.cache_read_micros_per_mtok // 1_000_000
    )
```

Integer math (micros) avoids float drift. Cost in the row is in micros; aggregations sum integers, never floats.

### OpenTelemetry span tree around the agent loop

```python
async def run_agent_step(ctx: TenantContext, thread: Thread) -> AgentReply:
    tracer = trace.get_tracer(__name__)
    with tracer.start_as_current_span("agent.step", attributes={
        "tenant.id": str(ctx.tenant_id),
        "thread.id": str(thread.id),
        "agent.step_count": thread.step_count,
    }) as step_span:
        plan = await llm.complete(ctx=ctx, purpose="agent_step", messages=...)
        # llm.complete creates a child span and writes a row tied to step_span

        for tool_call in plan.tool_calls:
            with tracer.start_as_current_span("agent.tool_call", attributes={
                "tool.name": tool_call.name,
            }) as tool_span:
                result = await tool_call.tool.execute(tool_call.args, ctx)
                tool_span.set_attribute("tool.success", True)

        final = await llm.complete(ctx=ctx, purpose="agent_step", messages=...)
        return AgentReply(content=final.content, ...)
```

Cloud Trace shows the agent step as the parent span, each LLM call and tool call as children. The `llm_calls` row's `parent_span_id` ties it to the agent step in trace storage.

### Streaming: per-token spans + one final row

```python
async def stream_complete(self, ctx, purpose, messages, ...) -> AsyncIterator[Token]:
    started = utcnow()
    first_token_at: datetime | None = None
    output_chunks: list[str] = []
    usage_final: Any = None

    with tracer.start_as_current_span("llm.stream") as span:
        async for event in self._stream(provider, model, messages):
            if event.type == "content_block_delta":
                if first_token_at is None:
                    first_token_at = utcnow()
                output_chunks.append(event.delta.text)
                yield Token(text=event.delta.text)
            elif event.type == "message_delta":
                usage_final = event.usage

        await self._record_streaming_row(ctx, purpose, model, ..., started, first_token_at, output_chunks, usage_final, span)
```

The row is written at stream end; `ttfb_ms` captures perceived latency separately from total `latency_ms`.

### PII redaction before persisting

```python
def redact(payload: dict[str, Any]) -> dict[str, Any]:
    """Walks the payload, redacts emails/phones/known sensitive keys."""
    return _redact_value(payload)
```

Same redactor used by `python-logging`. Both ingestion paths share the rule set so a regex addition applies everywhere.

### BigQuery sink

Pub/Sub topic `llm_calls` with a BigQuery subscription writing to `analytics.llm_calls`. Same schema, partitioned by `started_at`, clustered by `(tenant_id, model)`. Analytical queries hit BigQuery; per-call detail hits Postgres (hot, last 7 days).

## Never

- Logging the full prompt or completion through `structlog`. *Why:* general logs aren't redaction-controlled with the same discipline as `llm_calls`. Keep prompts in one sink with one retention policy.
- Storing the un-redacted prompt anywhere — even in dev. *Why:* dev data leaks become prod incidents.
- Estimating token counts when the API returns them. *Why:* the official count is authoritative and free. Estimates drift, cause budget bugs, and look unprofessional in customer cost reports.
- Computing cost at query time. *Why:* model rates change. Yesterday's row should reflect yesterday's cost. Compute on write, store, query as sum.
- Floating-point cost. *Why:* `0.1 + 0.2 != 0.3` strikes a billing query. Use integer micros.
- One span for the whole agent run. *Why:* you lose the LLM-call-tree structure that makes debugging possible.
- Forgetting `tenant_id` on the row. *Why:* analytics queries that aggregate by tenant fail silently or leak across.
- Schema invented from scratch ignoring Langfuse shape. *Why:* the future swap becomes a re-instrumentation instead of an importer.
- Streaming with no final row. *Why:* analytical queries can't count or sum costs without per-call rows.
- Writing the row asynchronously and losing it on crash. *Why:* the row is the trace. Write inside the request lifecycle; failures of the LLM call still write a row.
- Including raw tool credentials in `tools_called`. *Why:* tool call payloads can contain tokens. Redact at the same layer as prompts.
- Mixing system-level metrics into `llm_calls`. *Why:* keep `llm_calls` for LLM calls; system metrics belong in Cloud Monitoring.
- Logging `model_dump()` of a Pydantic model that contains `SecretStr`. *Why:* leaks. Use `model_dump(mode='json')` (masks) or explicit field exclusion. See `secrets-and-credentials`.

## Pitfalls

- **Anthropic streaming `usage` only arrives in the final event.** Buffer or capture from `message_delta`. Forgetting this means `usage = None` and zero cost on streaming rows.
- **Cache token accounting confusion.** With Anthropic prompt caching, `input_tokens` excludes cache hits; `cache_read_input_tokens` is the hit count. Compute cost with all four counters or you'll under-bill yourself.
- **Trace context not propagating across `asyncio.create_task`.** Same as `contextvars`. Use `TaskGroup` or `contextvars.copy_context()`.
- **OpenTelemetry exporter blocking on shutdown.** Use the batch span processor with a sane queue size; on Cloud Run shutdown, force-flush with a timeout.
- **BigQuery sink lag.** Pub/Sub → BigQuery is "near real-time" but not instant. Don't show "current minute" dashboards from BigQuery — use Postgres for hot data.
- **Redaction missing structured fields.** Tool args containing emails as `params["to"]` are nested. Recursive redactor handles this; verify with a test.
- **`llm_calls` Postgres table growing unboundedly.** Add a retention job: archive rows older than N days to BigQuery-only, delete from Postgres. Keep aggregate counters per tenant indefinitely.
- **Cost rate table drift.** When a model is renamed/added, the rates lookup is your single source of truth. Test that every model in the routing map has a rate.
- **Tenant boundary on Postgres queries.** `llm_calls` is RLS-scoped; admin queries need the documented bypass role + audit. See `postgres-multi-tenant-rls`.
- **Streaming finish_reason not captured.** Anthropic emits it in `message_delta`. Without capturing, your retention analytics for "calls hitting max_tokens" are blind.
- **Tool call args vs results conflation.** `tools_called` should contain `{name, args, result_summary}`. Don't store full results (can be huge); store a reference (e.g., output ID) if needed.
- **Span IDs as bytes vs hex.** Format consistently to hex when writing to DB. Otherwise joins across systems break.

## Verification

Before declaring LLM observability work done:

- Every LLM call site goes through `LLMClient`; no direct `anthropic.messages.create(...)` outside the wrapper.
- `llm_calls` table has RLS + force + policy; verified by migration test.
- Cost test: given a synthetic usage object and model, `compute_cost_micros` returns the expected integer.
- A test asserts that a successful Anthropic call produces an `llm_calls` row with non-zero tokens and cost.
- A test asserts that a failed Anthropic call (mocked to 500) still produces a row with `error` populated.
- A streaming test asserts that the row is written at stream end with `ttfb_ms` < `latency_ms`.
- OpenTelemetry test: spawn an agent step that does two LLM calls; assert the trace tree has parent + two children.
- A redaction test: pass a prompt containing `user@example.com` and assert the persisted `prompt` JSONB contains `[redacted]`.
- Pub/Sub publisher test: confirm an `llm_calls` event is published per call (matched against the DB row by `id`).
- BigQuery sink integration test (or staging-only): a call in staging shows up in BigQuery within ~60s.
- Retention job test: rows older than N days are deleted from Postgres but counted aggregates remain.
- Field map test: every field in `llm_calls` has a corresponding Langfuse `Generation` field (documented in code comment for swap-readiness).
- Routing test: `route_model("voice_ack")` returns Haiku; `route_model("agent_step")` returns Sonnet; etc.
