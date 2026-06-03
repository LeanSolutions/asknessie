---
name: fastapi-design
description: Use when designing or modifying FastAPI surface — routes, dependencies, request/response models, lifespan, middleware, streaming, versioning, OpenAPI metadata. Use whenever the API edge is the subject: how an endpoint is shaped, how a dependency is wired, how a long response is delivered, how startup/shutdown works.
---

# fastapi-design

## When to use this skill

Triggered for any FastAPI-edge change: new route, new dependency, response shape, streaming, middleware, OpenAPI tags, status codes, versioning. Also during review — endpoints are the public surface; every decision here is visible to clients.

## Core principles

1. **Routes are thin.** Parse input → call service → return response. No business logic. If the handler is longer than 10 lines, the logic belongs in a service.
2. **Dependencies for cross-cutting.** Auth, DB session, settings, tenant context, integrations. `Depends` is the DI container.
3. **Separate models per direction.** `ThreadCreate`, `ThreadRead`, `ThreadUpdate`. Reusing one model for input and output is how fields leak the wrong direction.
4. **`response_model=` on every route.** Don't let FastAPI infer from return type alone. Explicit prevents accidental field leakage.
5. **Lifespan for startup/shutdown.** Async context manager, not `@app.on_event`. Build clients, connect pools, run health checks; tear down in reverse.
6. **Stream long responses.** Agent replies, long lists, exports — SSE (`text/event-stream`). Never make a client wait 30s for a buffered JSON.
7. **Background work goes to durable executor.** Not `BackgroundTasks`. `BackgroundTasks` runs in-process, dies on restart, has no retry. Real async work needs `DurableExecutor`.
8. **Version under `/v1` from day one.** `/v2` lives alongside until `/v1` sunsets. The cost of starting versioned is zero; the cost of retrofitting is real.
9. **OpenAPI hygiene.** Every route has a tag, summary, response codes documented. The spec is the contract with frontend; treat it as a first-class artifact.

## Always

### Lifespan in `main.py`

```python
from contextlib import asynccontextmanager
from fastapi import FastAPI

@asynccontextmanager
async def lifespan(app: FastAPI) -> AsyncIterator[None]:
    settings = get_settings()
    configure_logging(settings.env, settings.log_level)
    engine = create_async_engine(...)
    redis = await aioredis.from_url(...)
    anthropic = build_anthropic_client(settings)
    app.state.engine = engine
    app.state.redis = redis
    app.state.anthropic = anthropic
    logger.info("app_started", env=settings.env)
    try:
        yield
    finally:
        await redis.close()
        await engine.dispose()
        logger.info("app_stopped")

app = FastAPI(
    title="asknessie",
    lifespan=lifespan,
    docs_url="/docs",
    redoc_url=None,
)
```

Every long-lived resource (DB pool, Redis pool, vendor clients) is built once at startup and accessed via dependencies.

### Dependencies via `Annotated` aliases

```python
SettingsDep    = Annotated[Settings, Depends(get_settings)]
DbDep          = Annotated[AsyncSession, Depends(get_db)]
RedisDep       = Annotated[Redis, Depends(get_redis)]
TenantDep      = Annotated[TenantContext, Depends(get_tenant_context)]
CurrentUserDep = Annotated[User, Depends(get_current_user)]
LLMDep         = Annotated[LLMClient, Depends(get_llm_client)]

async def get_db(request: Request) -> AsyncIterator[AsyncSession]:
    engine = request.app.state.engine
    async with AsyncSession(engine) as s:
        yield s
```

Each dep alias is the canonical way to get that thing. Routes consume them; no globals, no module-level singletons.

### Tenant-scoped DB session in the dep

```python
async def get_db(
    request: Request,
    ctx: TenantDep,
) -> AsyncIterator[AsyncSession]:
    engine = request.app.state.engine
    async with AsyncSession(engine) as s:
        await s.execute(text("SET LOCAL app.tenant_id = :tid"), {"tid": str(ctx.tenant_id)})
        yield s
```

Every DB session opened during a request is RLS-scoped. The handler can't accidentally query cross-tenant data — Postgres enforces. See `postgres-multi-tenant-rls`.

### Routers organized by resource

```python
# src/api/v1/threads.py
router = APIRouter(prefix="/threads", tags=["threads"])

@router.post(
    "",
    response_model=ThreadRead,
    status_code=201,
    summary="Create a thread",
    responses={409: {"description": "Conflict — duplicate idempotency key"}},
)
async def create_thread(
    body: ThreadCreate,
    svc: Annotated[ThreadService, Depends(get_thread_service)],
    ctx: TenantDep,
) -> ThreadRead:
    thread = await svc.create(body, ctx)
    return ThreadRead.model_validate(thread)
```

```python
# src/main.py
from .api.v1 import threads, messages, agents, billing
app.include_router(threads.router, prefix="/v1")
app.include_router(messages.router, prefix="/v1")
app.include_router(agents.router, prefix="/v1")
app.include_router(billing.router, prefix="/v1")
```

### Direction-separated request/response models

```python
class ThreadCreate(BaseModel):
    title: Annotated[str, Field(min_length=1, max_length=200)]
    initial_message: Annotated[str | None, Field(default=None, max_length=10_000)]

class ThreadRead(BaseModel):
    model_config = ConfigDict(from_attributes=True)
    id: UUID
    title: str
    status: Literal["active", "archived"]
    created_at: datetime
    message_count: int

class ThreadUpdate(BaseModel):
    title: str | None = None
    status: Literal["active", "archived"] | None = None
```

Never one `Thread` model used for create AND read AND update. Pyright catches signature mismatches; missing direction-separation lets fields leak.

### Stream agent responses via SSE

```python
from sse_starlette.sse import EventSourceResponse

@router.post("/threads/{tid}/messages")
async def send_message(
    tid: ThreadId,
    body: MessageCreate,
    svc: Annotated[ConversationService, Depends(get_conversation_service)],
    ctx: TenantDep,
):
    async def event_stream() -> AsyncIterator[dict[str, str]]:
        async for chunk in svc.reply_stream(tid, body, ctx):
            yield {"event": chunk.kind, "data": chunk.payload}
    return EventSourceResponse(event_stream())
```

Clients see tokens as the model emits them. No 30-second wait for a buffered response.

### Single exception handler (see `python-exception-handling`)

```python
@app.exception_handler(Exception)
async def handle(request: Request, exc: Exception) -> JSONResponse:
    status = _status_for(exc)
    code = type(exc).__name__
    if status >= 500:
        logger.error("unhandled_exception", code=code, exc_info=exc)
        return JSONResponse(status_code=status, content={"error": {"code": "internal_error"}})
    logger.info("client_error", code=code, status=status)
    return JSONResponse(status_code=status, content={"error": {"code": code, "detail": _safe(exc)}})
```

No `raise HTTPException` in services. Domain errors raise; the handler maps.

### Middleware for cross-cutting (in order)

```python
app.add_middleware(CORSMiddleware, allow_origins=[settings.web_origin], ...)
app.add_middleware(RequestIDMiddleware)         # bind request_id
app.add_middleware(TenantBindingMiddleware)     # bind tenant_id after auth
app.add_middleware(StructuredLogMiddleware)     # log request + response
```

Middleware order matters — outermost wraps everything. CORS before auth so preflight works. Logging outermost so it captures error paths.

### Versioned routers, parallel versions

```python
from .api.v1 import threads as v1_threads
from .api.v2 import threads as v2_threads

app.include_router(v1_threads.router, prefix="/v1")
app.include_router(v2_threads.router, prefix="/v2")
```

`/v1` lives until clients have migrated. Sunset with a deprecation header + a public timeline.

### OpenAPI metadata that matters

```python
app = FastAPI(
    title="asknessie API",
    version="2026.06.0",
    description="Internal API. See https://docs.asknessie.ai for client docs.",
    openapi_tags=[
        {"name": "threads", "description": "Threads and conversation history"},
        {"name": "agents", "description": "Agent runs and tool calls"},
    ],
)
```

Tags group routes in `/docs`. `summary` and `description` on routes is the contract.

## Never

- Business logic in route handlers. *Why:* routes are the transport; logic is the domain. Mixing creates an HTTP-shaped business layer.
- `raise HTTPException` in services. *Why:* couples service to HTTP. Raise a domain error; the handler maps. See `python-exception-handling`.
- `BackgroundTasks` for actual work. *Why:* runs in-process, dies on Cloud Run restart, no retry, no observability. Use `DurableExecutor` for anything that matters.
- Reusing the DB model as the response model. *Why:* DB fields drift to the API. Pydantic response model is your edge contract.
- Returning the raw SQLAlchemy entity. *Why:* serialization surprises, accidental field leakage, lazy-load triggered after session closes. Convert to `ResponseModel.model_validate(entity)`.
- Omitting `response_model=`. *Why:* FastAPI infers, but inference can leak fields. Explicit is the contract.
- `@app.on_event("startup")`. *Why:* deprecated in favor of `lifespan`. Use the context manager.
- Module-level state for DB / Redis / vendor clients. *Why:* binds at import, breaks tests, leaks across requests. Build in `lifespan`, expose via `app.state` + `Depends`.
- `request.state.tenant_id` as the way to access tenant. *Why:* not type-safe, not auto-completing. Use a `TenantContext` dep.
- Hand-written URL strings on the client side instead of generated OpenAPI types. *Why:* lose the type contract. Generate clients from the OpenAPI spec.
- Different routes returning the same resource with different shapes. *Why:* clients can't program against it. One canonical `ThreadRead`; expand via query params if needed (`?include=messages`).
- Untagged routes. *Why:* `/docs` becomes a jumbled list. Always set a tag.
- Catching exceptions in the route to format a custom response. *Why:* duplicates the handler. Let the exception propagate; the handler is the only place that maps.
- Streaming in a way that buffers before sending. *Why:* defeats streaming. Verify with a slow generator and a real client; first byte should arrive in <100ms.
- WebSocket where SSE works. *Why:* SSE is simpler, auto-reconnect, plays well with proxies. WebSocket only when you need bidirectional.

## Pitfalls

- **Dependencies cached per-request unexpectedly.** FastAPI caches `Depends(fn)` results per request. Two different routes asking for `get_db` get the same session within one request — that's the point. If you want a *new* session, use `Depends(get_db, use_cache=False)` or build it explicitly.
- **Yielding from a dep without proper cleanup.** Async generators in deps must close in `finally`. A leaked session is a leaked connection.
- **CORS preflight blocking SSE.** Preflight returns 204 by default; if the SSE endpoint requires a custom header, list it in `expose_headers`.
- **`response_model` stripping fields you want.** `response_model_exclude_none=True` removes nulls, which sometimes hides intent. Default to off; opt in per route.
- **Pydantic validation error not on the expected field.** A nested model's error appears at the top level with a path. Make sure the API's error handler unpacks the path so clients know which field failed.
- **Middleware order surprise.** Middleware added later wraps everything added earlier (FastAPI is LIFO). Visualize the stack.
- **Streaming response timing out at the proxy.** Cloud Run, nginx, Cloud LB all have idle timeouts. SSE keepalive (a `:` comment every 15-20s) prevents disconnects.
- **`response_model` of `list[Thread]` not validating Thread correctly.** Sometimes Pyright + Pydantic disagree. Wrap in a Pydantic model: `class ThreadList(BaseModel): items: list[ThreadRead]`.
- **`Depends(get_db)` running before auth.** If `get_db` requires `TenantContext` (for RLS), the auth dep must run first. FastAPI orders deps by reference order — make `get_db` depend on `get_tenant_context`.
- **`/health` blocked by tenant binding.** Health check has no tenant. Make it a separate router with no auth dep, mounted before everything else.
- **`Path` parameter type doesn't validate UUID.** `tid: UUID` works for path params; `tid: str` accepts anything. Use the typed form.
- **Mixing sync and async deps.** A sync dep used by an async route is wrapped in a threadpool. Fine for fast deps; harmful for slow ones. Make all deps async unless they're truly trivial.
- **OpenAPI showing the wrong type for a Pydantic generic.** Sometimes Pyright sees the right type but the OpenAPI generator inlines incorrectly. Test the spec with `client.get("/openapi.json")` in a unit test.
- **`StreamingResponse` content type defaulting wrong.** Set `media_type` explicitly: `media_type="text/event-stream"` for SSE.
- **Multiple workers losing session affinity for SSE.** SSE connections are sticky to the worker that opened them. With horizontal Cloud Run, that's fine within one worker; if you need fanout across workers, use Pub/Sub or Redis pub/sub to coordinate.

## Verification

Before declaring API design work done:

- Every route has `response_model=`, a tag, and a `summary`.
- `rg "raise HTTPException" src/` returns only `src/main.py` or wherever the global handler lives — not in services or repos.
- `rg "BackgroundTasks" src/` returns nothing (use `DurableExecutor`).
- `rg "@app\.on_event" src/` returns nothing (use `lifespan`).
- DB session dep includes `SET LOCAL app.tenant_id`; verified by a test that switches tenants between requests.
- Routes are versioned: every route in `src/api/v1/` mounted under `/v1`.
- A streaming endpoint test confirms first byte arrives within 100ms (no buffering).
- A test confirms `/health` works without auth and doesn't require tenant binding.
- A test confirms exception handler maps each domain error class to the expected status with the expected body shape.
- `client.get("/openapi.json")` returns a valid spec with tags and response models populated.
- CORS allows the web origin; preflight succeeds for the SSE endpoint with custom headers.
- No SQLAlchemy entities returned directly from routes; all conversions go through `ResponseModel.model_validate(...)`.
