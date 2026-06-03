---
name: python-logging
description: Use when adding log lines, configuring structlog, designing log context propagation, redacting PII in logs, choosing log levels, wiring logs to Cloud Logging or Cloud Error Reporting, or debugging "I can't find this in the logs." Applies anywhere `logger`, `print`, or `logging` would otherwise be used.
---

# python-logging

## When to use this skill

Triggered for any log-related change: adding a new log line, setting up structlog at app startup, designing what context attaches to logs, building a redactor, deciding info-vs-warning-vs-error, integrating logs with Cloud Logging / Cloud Error Reporting / future Langfuse. Also applies when reviewing code: every uncaught exception, integration call, and business event is a candidate log site.

## Core principles

1. **Logs are events, not strings.** A log line is `event_name + structured_fields`, never a sentence with values interpolated in. `logger.info("task_completed", task_id=t.id, duration_ms=...)`, not `logger.info(f"task {t.id} completed in {ms}ms")`.
2. **Context flows automatically.** Bind `tenant_id`, `user_id`, `request_id`, `thread_id` once per request (in middleware). Every subsequent log line inside that request inherits them. Never re-pass them on each call.
3. **Same code, different output.** Dev gets a colorized console renderer. Prod gets single-line JSON. Identical call sites. Configured at startup.
4. **PII redacted at the logger, not at the call site.** A redactor processor in the structlog chain strips emails, phone numbers, full names, secrets before they hit any sink. Developers should not have to remember to redact.
5. **One logger, one logging configuration.** `structlog.get_logger()` everywhere. No `logging.getLogger(__name__)` returning unconfigured loggers. No `print`.
6. **Log levels mean what they say.** Debug: developer-only diagnostics. Info: business events. Warning: recoverable anomaly. Error: handled failure. Critical: process about to die. Don't invent local conventions.
7. **Errors carry `exc_info`, not formatted tracebacks.** Pass the exception object so Cloud Logging gets the structured stack trace and Cloud Error Reporting auto-groups it.

## Always

### Initialize structlog once, in `main.py` lifespan startup

```python
import logging
import sys
import structlog
from structlog.contextvars import merge_contextvars
from structlog.processors import (
    TimeStamper, add_log_level, format_exc_info, JSONRenderer,
)

def configure_logging(env: Literal["dev", "staging", "prod"], level: str) -> None:
    shared_processors: list[structlog.types.Processor] = [
        merge_contextvars,                       # auto-pulls bound context
        add_log_level,
        TimeStamper(fmt="iso", utc=True),
        redact_pii,                              # custom processor (see below)
        format_exc_info,
    ]

    if env == "dev":
        renderer = structlog.dev.ConsoleRenderer(colors=True)
    else:
        renderer = JSONRenderer()

    structlog.configure(
        processors=[*shared_processors, renderer],
        wrapper_class=structlog.make_filtering_bound_logger(
            getattr(logging, level.upper())
        ),
        context_class=dict,
        logger_factory=structlog.PrintLoggerFactory(file=sys.stdout),
        cache_logger_on_first_use=True,
    )
```

Called once during `lifespan`. After that, anywhere in the codebase:

```python
import structlog
logger = structlog.get_logger()
```

### Bind context in middleware

```python
from structlog.contextvars import bind_contextvars, clear_contextvars
from uuid import uuid4

@app.middleware("http")
async def bind_request_context(request: Request, call_next):
    clear_contextvars()
    bind_contextvars(
        request_id=request.headers.get("x-request-id", str(uuid4())),
        method=request.method,
        path=request.url.path,
    )
    # tenant_id and user_id bound by auth dep, not here
    try:
        return await call_next(request)
    finally:
        clear_contextvars()
```

Auth dependency binds `tenant_id` and `user_id` as soon as the request is authenticated:

```python
async def get_current_user(...) -> User:
    user = ...
    bind_contextvars(tenant_id=str(user.tenant_id), user_id=str(user.id))
    return user
```

### Use snake_case event names

```python
logger.info("thread_created", thread_id=str(t.id), channel="web")
logger.info("llm_call_started", model="claude-sonnet-4-6", task="agent_step")
logger.warning("integration_retry", vendor="twilio", attempt=2, reason="503")
logger.error("tool_execution_failed", tool="gmail_search", exc_info=err)
```

Event names are stable identifiers — you'll query them in BigQuery later. Treat them like API endpoints, not prose.

### Log exceptions with `exc_info=err`

```python
try:
    result = await tool.execute(args, ctx)
except ToolExecutionError as err:
    logger.error("tool_execution_failed", tool=tool.name, exc_info=err)
    raise
```

`exc_info=err` (the exception object) is preferred over `exc_info=True` — explicit, harder to misuse, works correctly across `await` boundaries.

### Build a PII redactor processor

```python
import re
from typing import Any

_EMAIL = re.compile(r"[A-Za-z0-9._%+-]+@[A-Za-z0-9.-]+\.[A-Za-z]{2,}")
_PHONE = re.compile(r"\+?\d[\d\s\-().]{7,}\d")
_REDACTED = "[redacted]"
_SENSITIVE_KEYS = {"password", "token", "api_key", "secret", "authorization", "cookie"}

def redact_pii(_logger: Any, _method: str, event_dict: dict[str, Any]) -> dict[str, Any]:
    return _redact_value(event_dict)  # type: ignore[return-value]

def _redact_value(v: Any) -> Any:
    if isinstance(v, dict):
        return {k: _REDACTED if k.lower() in _SENSITIVE_KEYS else _redact_value(val) for k, val in v.items()}
    if isinstance(v, list):
        return [_redact_value(x) for x in v]
    if isinstance(v, str):
        v = _EMAIL.sub(_REDACTED, v)
        v = _PHONE.sub(_REDACTED, v)
        return v
    return v
```

Place this processor *before* the renderer so it redacts both top-level fields and nested values.

### Propagate context across `asyncio.create_task`

contextvars don't auto-propagate to detached tasks. When you need them to:

```python
import contextvars

ctx = contextvars.copy_context()
task = asyncio.create_task(ctx.run(some_async_fn, args))
```

Or use `asyncio.TaskGroup` which propagates context correctly.

### Log levels (the standard, no local conventions)

| Level | Use for | Example |
|---|---|---|
| `debug` | Developer diagnostics, never on in prod | "fetched 3 candidates from vector search" |
| `info` | Business events, normal operation | `thread_created`, `tool_called`, `llm_call_completed` |
| `warning` | Recoverable anomaly or degradation | `integration_retry`, `cache_miss_unexpected`, `slow_query` |
| `error` | Failure that was handled (logged + raised or returned) | `tool_execution_failed`, `webhook_signature_invalid` |
| `critical` | Process about to die or data is at risk | `database_unreachable`, `kms_unavailable` |

## Never

- `print()` anywhere in `src/`. *Why:* unstructured, no context, no level, no JSON output. Use `logger.debug` or `logger.info`.
- `logging.getLogger(__name__)` followed by `.info(...)`. *Why:* returns an unconfigured stdlib logger that doesn't go through our processor chain. PII won't get redacted; JSON won't be JSON.
- f-strings in log messages. *Why:* defeats structured logging. `logger.info(f"user {u.email} signed in")` is one event; `logger.info("user_signed_in", user_id=u.id)` is queryable.
- Logging Pydantic models directly. *Why:* `.model_dump()` may include `SecretStr` values, and the structure isn't predictable in the renderer. Extract the fields you want: `logger.info("event", user_id=user.id, plan=user.plan)`.
- Passing `tenant_id` on every call when it's already bound. *Why:* noise. If it's bound, `merge_contextvars` adds it. If you're re-passing it, the bind isn't working — fix the bind.
- `logger.error("error happened")` with no exception. *Why:* loses the stack trace, makes Cloud Error Reporting useless. Either include `exc_info=err` or log it as info/warning.
- `except Exception: pass` with no log. *Why:* silent failures are the worst class of bug. At minimum: `logger.warning("event_name_ignored", reason=str(err))`.
- Logging full prompts or completions outside the dedicated `llm_calls` table (see `python-llm-observability`). *Why:* prompts often contain PII/tenant data. Keep them in one tightly-controlled sink with retention policy, not scattered through general logs.
- Sentence-style event names like `"User signed up successfully"`. *Why:* not queryable. Use `user_signed_up`.
- Mixing `print` and `logger` in the same module. *Why:* output ordering becomes nondeterministic; one stream is JSON, the other isn't.

## Pitfalls

- **contextvars not flowing across `create_task`.** Detached tasks get a fresh context. Use `asyncio.TaskGroup` (which propagates) or explicit `contextvars.copy_context().run(...)`.
- **Logger created at module import time.** If `structlog.configure(...)` hasn't run yet, you get a default-configured logger that misses your processors. The fix: don't cache loggers at module top-level if config happens at lifespan startup — call `structlog.get_logger()` inside functions, or rely on `cache_logger_on_first_use=True` and ensure config runs before any logger is used.
- **JSON renderer choking on non-serializable values.** `datetime`, `UUID`, `Path`, `Decimal` all break. Add a `default=str` fallback in the JSON renderer config or pre-process those fields. Better: convert at the call site (`str(uuid)`, `dt.isoformat()`).
- **PII redactor missing nested fields.** A naive top-level scan misses `data={"user": {"email": "..."}}`. The recursive `_redact_value` above handles it; verify with a test.
- **Stdlib `logging` calls from third-party libs not hitting structlog.** Bridge stdlib → structlog with `logging.basicConfig(handlers=[structlog.stdlib.PositionalArgumentsFormatter()])` or `structlog.stdlib.recreate_defaults()`. Otherwise httpx/sqlalchemy/uvicorn logs come out unstructured.
- **`exc_info=True` inside an exception handler that runs async.** Works, but `exc_info=err` (passing the exception object explicitly) is clearer and survives unusual stack situations.
- **Log noise from chatty third parties.** uvicorn access logs, SQLAlchemy echo, httpx debug — turn them off or raise their level via `logging.getLogger("uvicorn.access").setLevel("WARNING")` in `configure_logging`.
- **Binding context in a dependency that doesn't run on every request.** If `tenant_id` is bound in a dep that some routes skip (e.g., health checks), those routes leak nothing — but tests of other routes may show missing `tenant_id` and confuse the reader. Add a startup assertion in tests that every authenticated route binds tenant context.
- **`clear_contextvars()` in middleware not running on exceptions.** Put it in `finally`, as shown above. Otherwise context leaks across requests on the same worker.
- **Cloud Logging not auto-grouping errors.** Cloud Error Reporting looks at the `severity=ERROR` level and a stack trace in the JSON payload. If `exc_info` isn't being rendered into the JSON, grouping fails. Verify with a real test in dev pointing at Cloud Logging.

## Verification

Before declaring logging work done:

- In dev, `logger.info("test_event", k=1)` produces a colorized line with timestamp, level, event, and `k=1`.
- In prod config, the same call produces a single-line JSON document with `event`, `level`, `timestamp`, `k`, `tenant_id` (if bound), `user_id` (if bound), `request_id` (if bound).
- A test logs an event with `email="user@example.com"` in the data and asserts the rendered output contains `[redacted]`.
- A test makes an authenticated request, logs inside the handler, and asserts the captured log contains `tenant_id` and `user_id`.
- `rg "print\(" src/` returns nothing. `rg "logging\.getLogger" src/` returns nothing.
- A handled exception logged with `exc_info=err` shows up in Cloud Error Reporting as a grouped error within a minute (manual check against the dev project).
- chatty third-party loggers (uvicorn access, sqlalchemy.engine) are silenced or set to warning.
