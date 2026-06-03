---
name: python-exception-handling
description: Use when defining new exception classes, deciding what to catch or re-raise, designing error responses, building the API exception handler, propagating errors across module/layer boundaries, or reviewing any `try/except` block. Use whenever an HTTP error response, a domain failure, or a third-party integration failure crosses your code.
---

# python-exception-handling

## When to use this skill

Triggered for any error-related design or change: defining domain exceptions, writing `try/except`, building error responses at the API boundary, mapping integration failures to internal semantics, propagating errors across layers. Also applies during review — every `except` clause is a design decision worth checking.

## Core principles

1. **Errors are typed classes, never strings or dicts.** A failure mode is a name, not a message. The name is queryable; the message is not.
2. **Catch narrowly.** `except SpecificError:` not `except Exception:`. Bare except hides the bug you didn't write the code to handle.
3. **Re-raise with context.** `raise NewError(...) from original` preserves the chain. Lose the chain and you've lost the stack trace.
4. **Translate at boundaries, not throughout.** Domain errors stay domain errors inside services. They map to HTTPException **in exactly one place** — the API exception handler. Services never construct HTTP responses.
5. **Errors carry data, not formatted messages.** `TaskNotFound(task_id=...)` not `TaskNotFound("Task abc-123 not found")`. The handler composes the message; the exception carries the facts.
6. **Cross-tenant error messages must not leak data.** An error caused by tenant B must never reveal tenant B's data in tenant A's response.
7. **Log at the boundary where you stop re-raising.** If you catch, handle, and continue: log. If you catch and re-raise: don't log (the boundary will). Two logs for one error is noise; zero is a silent failure.
8. **`async`-aware.** `asyncio.CancelledError` must propagate. `ExceptionGroup` from `TaskGroup` must be handled with `except*` or unwrapped.

## Always

### Define a domain exception hierarchy per package

```python
# src/agents/errors.py

class AgentError(Exception):
    """Base for all agent-package errors."""

class ToolNotFound(AgentError):
    def __init__(self, tool_name: str) -> None:
        super().__init__(f"unknown tool: {tool_name}")
        self.tool_name = tool_name

class ToolExecutionError(AgentError):
    def __init__(self, tool_name: str, cause: Exception) -> None:
        super().__init__(f"tool {tool_name} failed")
        self.tool_name = tool_name
        self.cause = cause

class AgentBudgetExceeded(AgentError):
    def __init__(self, *, used_tokens: int, limit: int) -> None:
        super().__init__("agent budget exceeded")
        self.used_tokens = used_tokens
        self.limit = limit
```

Three things every exception class does:

1. Inherits from a single package base (`AgentError`).
2. Constructor takes typed kwargs and stores them as attributes — handlers and tests read those attributes.
3. The string passed to `super().__init__(...)` is a *stable identifier*, not a user-facing message. The handler composes the user-facing message.

### Re-raise with `from`

```python
try:
    result = await self._anthropic.messages.create(...)
except anthropic.RateLimitError as err:
    raise AnthropicRateLimited(retry_after=err.response.headers.get("retry-after")) from err
```

`from err` keeps the original traceback visible in logs and in Python's `__cause__`. Never `raise NewError(...) from None` unless you specifically want to suppress the cause — and write a comment explaining why.

### One exception handler per app, in `main.py`

```python
from fastapi import FastAPI, Request
from fastapi.responses import JSONResponse
import structlog

logger = structlog.get_logger()

DOMAIN_ERROR_MAP: dict[type[Exception], int] = {
    ToolNotFound: 404,
    ToolExecutionError: 502,
    AgentBudgetExceeded: 429,
    TenantNotFound: 404,
    AuthenticationRequired: 401,
    AuthorizationDenied: 403,
    ValidationFailed: 422,
}

def _status_for(exc: Exception) -> int:
    for typ, code in DOMAIN_ERROR_MAP.items():
        if isinstance(exc, typ):
            return code
    return 500

@app.exception_handler(Exception)
async def handle_exception(request: Request, exc: Exception) -> JSONResponse:
    status = _status_for(exc)
    safe_code = type(exc).__name__  # stable identifier
    if status >= 500:
        logger.error("unhandled_exception", code=safe_code, exc_info=exc)
        body = {"error": {"code": "internal_error"}}  # never leak internals
    else:
        logger.info("client_error", code=safe_code, status=status)
        body = {"error": {"code": safe_code, "detail": _safe_detail(exc)}}
    return JSONResponse(status_code=status, content=body)
```

`_safe_detail` returns only the explicitly-declared safe fields (the typed attributes on the exception class). It never `str(exc)`s — that could leak.

### Use `except*` for TaskGroup errors

```python
async def fan_out_tools(tools: list[Tool], ctx: TenantContext) -> list[Any]:
    try:
        async with asyncio.TaskGroup() as tg:
            tasks = [tg.create_task(t.execute(ctx)) for t in tools]
        return [t.result() for t in tasks]
    except* ToolExecutionError as eg:
        # All ToolExecutionErrors from the group
        raise PartialToolFailure(failures=list(eg.exceptions)) from eg
    except* Exception as eg:
        # Anything else — re-raise the group as-is
        raise
```

### Always let `CancelledError` propagate

```python
async def long_running_step(...) -> Result:
    try:
        return await do_work()
    except asyncio.CancelledError:
        raise  # never swallow
    except IntegrationError as err:
        logger.warning("integration_failed", exc_info=err)
        return fallback()
```

If you `except Exception`, narrow it to *specifically* exclude `CancelledError`, or re-raise it first:

```python
except asyncio.CancelledError:
    raise
except Exception as err:
    ...
```

### Log at the boundary where you stop re-raising

```python
# Service-level: catches and converts, re-raises domain error → don't log here
async def execute_tool(self, name: str, args: dict, ctx: TenantContext) -> Any:
    tool = self._registry.get(name) or (_ for _ in ()).throw(ToolNotFound(name))
    try:
        return await tool.run(args, ctx)
    except Exception as err:
        raise ToolExecutionError(name, err) from err

# Boundary-level (the FastAPI handler): logs once, never re-raises
# (see exception handler above)
```

### Errors that cross the tenant boundary must be sanitized

When an error from one tenant's data could surface in another tenant's response (e.g., shared cache, cross-tenant lookup gone wrong), the handler must replace any tenant-derived strings with the safe identifier:

```python
class CrossTenantAccessDenied(AuthorizationDenied):
    def __init__(self, *, attempted_tenant: UUID, actor_tenant: UUID) -> None:
        super().__init__("cross-tenant access denied")
        # Internal only — never serialized into the response
        self.attempted_tenant = attempted_tenant
        self.actor_tenant = actor_tenant
```

`_safe_detail` for this class returns `{}` — no tenant IDs in the response.

## Never

- `raise Exception("oops")`. *Why:* untyped, unactionable, can't be caught specifically. Define a class.
- `except Exception:` outside the outermost handler. *Why:* hides bugs you didn't write the code to handle. Catch the specific class.
- `except:` (bare except). *Why:* catches `KeyboardInterrupt`, `SystemExit`, `CancelledError`. Always at least `except Exception:` if you must.
- `raise NewError(...)` without `from original` when an original exists. *Why:* loses the cause chain. Stack trace gets harder to read.
- `except SomeError: pass`. *Why:* silent failure. At minimum, log it with a reason. Usually means you've misunderstood the error path.
- `raise HTTPException(...)` inside services, repositories, or domain code. *Why:* couples business logic to the transport layer. Raise a domain error; let the handler translate.
- Including the raw exception message in the response body. *Why:* leaks internals — DB structure, vendor error text, sometimes secrets. Use a stable `code` + explicitly-declared safe fields.
- Mapping domain → HTTP in multiple places (services + middleware + handlers). *Why:* drift guaranteed. One mapping table, one place.
- Reformatting exception messages: `raise AnthropicError(str(err) + " happened")`. *Why:* breaks log parsing, drops the cause. Use `from err` and structured fields.
- `logger.error(...)` followed by `raise`. *Why:* the outer boundary will log it. Two log entries for one error is noise.
- Catching `Exception` and continuing without logging. *Why:* the original sin of "silently broken." Always log; almost always re-raise.
- Treating `asyncio.CancelledError` as a regular exception. *Why:* cancellation must propagate, or you'll hang shutdown and leak tasks.

## Pitfalls

- **`raise ... from None` accidentally.** Inside an `except` block, raising a new exception without `from` *automatically* attaches the current exception as `__context__` — that's usually fine. `from None` explicitly suppresses; only use it if the cause is genuinely irrelevant or would leak.
- **`ExceptionGroup` from `TaskGroup` slips through.** A plain `except SomeError:` does *not* match an `ExceptionGroup[SomeError]`. Use `except* SomeError:` for groups.
- **`isinstance(exc, type)` lookup order in the map.** If `AnthropicRateLimited(IntegrationError)` is in the map after `IntegrationError`, the parent matches first. Either: (a) put more specific types first, (b) use `exc.__class__.__mro__` walk.
- **Pydantic `ValidationError` not caught.** FastAPI handles it automatically for request bodies, but `Model.model_validate(...)` inside services raises and isn't translated unless you catch and re-raise as `ValidationFailed`. Decide which layer owns the translation.
- **Re-raised exception with new info loses original args.** When you catch and re-raise as a new class, the new class's `__init__` must capture everything the handler will need. Don't drop fields silently.
- **`str(exc)` in logs leaking PII.** The exception's message may interpolate user-controlled strings. Log the typed attributes (`exc.tool_name`, `exc.user_id_hash`), not the message.
- **Test exceptions caught by global `except Exception` in pytest.** If a test framework or fixture wraps your code in `except Exception:`, your typed re-raises lose information. Add tests that assert the *type* of the raised exception, not just that "something raised."
- **`raise ... from err` inside an `__init__`.** Python complains because `__init__` returns None. Use a classmethod factory if you need cause-chaining at construction.
- **Reusing a domain error from a different package.** Tempting but wrong: it couples the packages. Either move the class to a shared `lib/errors.py` or define a parallel error in the current package.
- **HTTP 500 with detail leaks internals.** Always: 5xx → opaque body (`{"error": {"code": "internal_error"}}`). Log internally with full detail; respond externally with nothing.
- **`from err` chain across an `asyncio.gather(..., return_exceptions=True)`.** `gather` returns exceptions as values; the `from` chain isn't applied for you. Wrap in `TaskGroup` (which gives you `ExceptionGroup` and clean propagation) instead.

## Verification

Before declaring exception work done:

- Every exception class in `src/` inherits from a single per-package base, which inherits from `Exception` (not `BaseException` unless deliberately).
- `rg "raise Exception\(" src/` returns nothing.
- `rg "raise HTTPException" src/` returns hits only in `src/main.py` (or wherever the exception handler lives).
- `rg "except Exception" src/` shows only the single outermost handler.
- `rg "except:" src/` returns nothing (bare except).
- Every `except ... :` block either re-raises, logs, or returns a defined fallback — none are `pass`.
- The exception handler in `main.py` has 100% coverage of the keys in `DOMAIN_ERROR_MAP` (test asserts each maps to the expected status).
- For each domain error, a test asserts: (a) the typed attributes are set, (b) the HTTP response has the expected status, (c) the response body has the expected `code` and does not contain any tenant data from the exception's internal fields.
- A cross-tenant error test confirms tenant B's data does not appear in tenant A's response body.
- An `asyncio.CancelledError` test confirms cancellation propagates through service code (no `except Exception` swallows it).
- `ExceptionGroup` from a `TaskGroup` test confirms `except*` handling works as expected.
