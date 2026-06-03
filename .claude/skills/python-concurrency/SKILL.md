---
name: python-concurrency
description: Use when writing async code, designing parallel workflows, handling cancellation, bounding fan-out, setting timeouts, debugging "the app feels slow under load," or reviewing any `async def`. Use whenever the event loop, `await`, `asyncio.TaskGroup`, or background work is the subject.
---

# python-concurrency

## When to use this skill

Triggered for any concurrent code: writing `async def`, designing fan-out, choosing `gather` vs `TaskGroup`, adding timeouts, handling cancellation, bounding parallelism with semaphores, integrating sync libraries. Also during review — every `async def`, every `await`, every `create_task` is a design choice worth checking.

## Core principles

1. **`async` is for I/O. CPU is for threads or workers.** Async gives you concurrency, not parallelism. Long CPU work in `async def` starves every other in-flight request on the worker.
2. **Never block the event loop.** Sync I/O inside `async def` is a bug, full stop. `time.sleep`, `requests.get`, sync DB calls — all forbidden in async paths.
3. **Structured concurrency by default.** `asyncio.TaskGroup` for fan-out. Bare `asyncio.create_task` is an unstructured detached task that's easy to lose track of and easy to leak.
4. **Cancellation propagates. Always.** `asyncio.CancelledError` is not an error — it's a signal. Catching it without re-raising hangs shutdowns and leaks tasks.
5. **Bound everything.** `Semaphore` for concurrent calls. `asyncio.timeout()` for deadlines. Unbounded fan-out exhausts memory, connections, and downstream services.
6. **Pick the right tool.** `TaskGroup` for "run these together, fail together." `gather` for "I want exceptions back as values." `as_completed` for streaming results. They're not interchangeable.
7. **`contextvars`, not module globals.** Per-request state (tenant, trace, user) flows through `contextvars`. Module-level globals leak across requests on the same worker.

## Always

### Use `async def` for I/O; offload CPU

```python
# I/O — async
async def fetch_user(user_id: UUID) -> User:
    async with db.session() as s:
        return await s.get(User, user_id)

# CPU-heavy work — offload from the loop
async def hash_password(pw: str) -> str:
    return await asyncio.to_thread(bcrypt.hashpw, pw.encode(), bcrypt.gensalt())
```

If the work is CPU-bound and frequent (image processing, ML inference outside an external API), `asyncio.to_thread` is a stopgap. The real answer is a worker (Cloud Run Job) or a thread/process pool — not the request handler.

### Fan out with `TaskGroup`, not `create_task`

```python
async def search_all_inboxes(queries: list[str], ctx: TenantContext) -> list[Hits]:
    async with asyncio.TaskGroup() as tg:
        tasks = [tg.create_task(search_inbox(q, ctx)) for q in queries]
    return [t.result() for t in tasks]
```

`TaskGroup` guarantees: all tasks finish (or all get cancelled together if one fails), errors surface as `ExceptionGroup`, no detached tasks leak. This is structured concurrency.

Bare `create_task` without a `TaskGroup` is for the rare "fire-and-forget" case — see "Detached tasks" pitfall below.

### Bound concurrency with `Semaphore`

```python
# Don't hammer Anthropic with 1000 parallel calls
SEMAPHORE = asyncio.Semaphore(20)

async def call_llm(prompt: str) -> str:
    async with SEMAPHORE:
        async with asyncio.timeout(30):
            return await anthropic.messages.create(...)
```

Picking the limit: usually the downstream's per-tenant rate cap, or the connection pool size of whatever's behind it. Empirically tune; don't pull a number from thin air.

### Set timeouts everywhere

```python
async with asyncio.timeout(10):
    result = await tool.execute(args, ctx)
```

Use `asyncio.timeout()` (3.11+), not the older `asyncio.wait_for()`. `timeout()` is the context manager form: cleaner, composable, integrates with `TaskGroup`.

There is no I/O call that doesn't deserve a timeout. The default of "wait forever" is a 2am page waiting to happen.

### Propagate `CancelledError`

```python
async def long_step(...) -> Result:
    try:
        return await do_work()
    except asyncio.CancelledError:
        raise  # always re-raise
    except IntegrationError as err:
        logger.warning("integration_failed", exc_info=err)
        return fallback()
```

If you must `except Exception`, exclude `CancelledError`:

```python
except asyncio.CancelledError:
    raise
except Exception as err:
    handle(err)
```

### Handle `ExceptionGroup` from `TaskGroup`

```python
try:
    async with asyncio.TaskGroup() as tg:
        for tool in tools:
            tg.create_task(tool.execute(ctx))
except* ToolExecutionError as eg:
    raise PartialToolFailure(failures=list(eg.exceptions)) from eg
```

A plain `except ToolExecutionError:` will **not** match the `ExceptionGroup`. Use `except*`.

### Propagate `contextvars` across detached work

For rare cases where you genuinely need a fire-and-forget task, copy the context:

```python
import contextvars

ctx_copy = contextvars.copy_context()
task = asyncio.create_task(ctx_copy.run(emit_metric, "thread_created"))
```

Better: use `TaskGroup` which propagates context automatically.

### Use the right gather variant

| Tool | When |
|---|---|
| `asyncio.TaskGroup` | Default. Fan out work that must all succeed or all be cancelled. |
| `asyncio.gather(*aws)` | Legacy. Fails fast on first exception — but other tasks keep running until they yield. Prefer `TaskGroup`. |
| `asyncio.gather(*aws, return_exceptions=True)` | When you want exceptions back as values to analyze. Useful for "best effort, report failures." |
| `asyncio.as_completed(aws)` | Stream results as they finish. Useful for "show first answer ASAP." |
| `asyncio.wait(aws, return_when=FIRST_COMPLETED)` | Race conditions. "Whichever finishes first wins; cancel the rest." |

### Use `Queue` for producer/consumer with backpressure

```python
queue: asyncio.Queue[Item] = asyncio.Queue(maxsize=100)  # bounded

async def producer():
    async for item in source:
        await queue.put(item)   # blocks if queue is full → backpressure

async def consumer():
    while True:
        item = await queue.get()
        try:
            await process(item)
        finally:
            queue.task_done()
```

Bounded `maxsize` is the point — unbounded queues become memory leaks under load.

### Bridge sync libraries with `asyncio.to_thread`

```python
result = await asyncio.to_thread(sync_pdf_renderer.render, doc)
```

Reserve `to_thread` for unavoidable sync libraries with no async alternative. Pre-flight: did you check for an async version of the library? Almost everything we use (`httpx`, `asyncpg`, SQLAlchemy 2.0, Anthropic SDK) has an async path.

## Never

- `time.sleep(x)` inside `async def`. *Why:* blocks the entire event loop. Use `await asyncio.sleep(x)`.
- `requests`, `urllib3`, `urllib.request`. *Why:* sync HTTP blocks the loop. Use `httpx.AsyncClient`.
- Sync DB drivers (`psycopg2`, sync SQLAlchemy). *Why:* same. Use `asyncpg`, SQLAlchemy 2.0 async session.
- `asyncio.create_task(...)` without a containing `TaskGroup` *unless* you're explicitly fire-and-forget AND you've stored the reference somewhere that owns its lifecycle. *Why:* unreferenced tasks are garbage collected; loop logs a warning; the task silently dies.
- `except asyncio.CancelledError: pass`. *Why:* swallows cancellation. Shutdowns hang. Always re-raise.
- `except Exception:` without re-raising `CancelledError` first. *Why:* `CancelledError` is a `BaseException` in 3.8+, so `except Exception` doesn't catch it — but it's a footgun if anyone makes it `Exception`-derived locally. Be explicit.
- `asyncio.run(...)` inside an `async def`. *Why:* nested event loops crash. `asyncio.run` is for the top of `main`, not inside coroutines.
- `await` on a coroutine you don't own without a timeout. *Why:* one slow downstream stalls the whole request.
- Unbounded `gather(*lots_of_tasks)`. *Why:* exhausts connections, memory, downstream quotas. Bound with `Semaphore`.
- Mixing `threading.Event`/`threading.Lock` with `asyncio` primitives. *Why:* they don't interoperate. Use `asyncio.Event`, `asyncio.Lock`.
- Holding `asyncio.Lock` across `await` you don't control. *Why:* if the awaited code awaits the same lock (indirect), deadlock. Keep critical sections short and bounded.
- `asyncio.wait_for(...)`. *Why:* deprecated for new code in favor of `asyncio.timeout()`. The context-manager form composes better.
- Calling sync code from inside an async path "because it's quick." *Why:* "quick" means "fine until production load," at which point it's a global stall.
- Spawning threads from request handlers. *Why:* uncontrolled thread pools. Use a job queue (durable executor) or `asyncio.to_thread` with the default pool.

## Pitfalls

- **Detached task GC'd before completion.** `asyncio.create_task(f())` without storing the result lets Python GC the task. Fix: hold the reference (`task = asyncio.create_task(...)`, then await or cancel it), or use `TaskGroup`. There's also `loop.create_task` and "background task" patterns in FastAPI — same rule.
- **`contextvars` not propagating to `create_task`.** Each detached task gets the *snapshot* of contextvars at the time of creation, but only if you use `contextvars.copy_context().run(...)`. Use `TaskGroup` which handles this for you, or copy explicitly.
- **`asyncio.gather` cancellation semantics.** When the first task raises, others continue until their next `await` and get cancelled. They have a brief window to do damage. `TaskGroup` cancels eagerly — prefer it.
- **`TaskGroup` raises `ExceptionGroup`, not the underlying exception.** Old `try/except SomeError` doesn't match. Use `except* SomeError`.
- **`asyncio.timeout()` cancels via `CancelledError`.** Inside the `with` block, your code sees `CancelledError` if time expires. Don't swallow it; the `timeout()` context converts it to `TimeoutError` on exit.
- **Sync libraries with hidden async-incompatible behavior.** Some logging handlers, signal handlers, and DB drivers do internal blocking. Symptom: latency spikes under load that don't correlate with downstream slowness. Tool: `aiomonitor` or `asyncio.get_event_loop().slow_callback_duration`.
- **`asyncio.run` calling code that imports modules with module-level async startup.** Modules that build clients at import time pin to the wrong loop. Build clients inside `lifespan`, not at import.
- **CPU-bound work starving the loop.** A 200ms JSON parse in `async def` looks innocent and stalls every other request on the worker for 200ms. Offload with `asyncio.to_thread` or move to a worker.
- **`asyncio.Queue` with unbounded `maxsize=0`.** Memory leak under producer-faster-than-consumer load. Always set a maxsize.
- **`asyncio.Lock` reentrancy.** Not reentrant. The same coroutine acquiring the same lock twice deadlocks.
- **Thread-pool exhaustion via `to_thread`.** Default pool is small (often `min(32, cpu+4)`). Many concurrent `to_thread` calls queue. Use a dedicated `ThreadPoolExecutor` for hot paths.
- **Forgetting to `await` a coroutine.** Returns a coroutine object that's never run. Pyright catches most of these; the ones it misses are usually inside dicts/lists of coroutines.
- **`asyncio.gather` over a generator.** `gather(*generator_of_coros)` materializes the generator; large fan-outs blow memory. Stream with `as_completed` or chunked `TaskGroup` instead.
- **Cancellation during cleanup.** `finally` blocks can be interrupted by cancellation. Use `asyncio.shield()` for critical cleanup that must complete.

## Verification

Before declaring concurrency work done:

- `rg "time\.sleep|requests\.|urllib\." src/` returns nothing.
- `rg "from psycopg2|psycopg\b" src/` returns nothing in app code (asyncpg/SQLAlchemy async only).
- `rg "asyncio\.create_task\(" src/` — every hit either uses a `TaskGroup` parent or is an audited fire-and-forget with stored reference and comment.
- `rg "asyncio\.wait_for\(" src/` returns nothing (use `asyncio.timeout()`).
- Ruff's `ASYNC` rules enabled in `pyproject.toml` (`ASYNC100`, `ASYNC101`, etc.) and passing.
- Every external integration call has an `asyncio.timeout()` or equivalent deadline.
- A cancellation test starts a long task, cancels it, and confirms it stops within ~100ms and that `CancelledError` propagates (no swallowing).
- A load test (or k6/locust if we set one up) hits N concurrent requests to an LLM-calling route; observe that the worker doesn't OOM and downstream quota isn't blown — proves bounded concurrency works.
- An "event loop blocking" test introduces an artificial `time.sleep(0.5)` in a code path, confirms it shows up as a latency spike (so the monitoring catches it in future regressions).
- contextvars propagate across `TaskGroup`: a test binds `tenant_id` and asserts it's visible inside a fanned-out task.
