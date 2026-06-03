---
name: python-durable-execution
description: Use when building or extending the in-house durable executor, designing durable functions, handling retries/idempotency/fan-out/scheduling, debugging stuck jobs, or considering "should I just call Inngest." Use when long-running work needs to survive Cloud Run restarts and crashes.
---

# python-durable-execution

## When to use this skill

Triggered for any durable-execution work: defining a new durable function, writing a step, designing idempotency keys, handling retries, fan-out + join, scheduling, debugging a failed/stuck job. Also during review — any background work that "should survive a restart" is durable territory.

## Core principles

1. **Jobs table is the source of truth.** Workers poll, never push. Crashes mid-step leave state in Postgres; the next poll resumes.
2. **Steps are idempotent.** Every step has an idempotency key derived from the job + step name + step inputs. Re-running with the same key returns the cached result instead of re-executing.
3. **Retries with exponential backoff and jitter.** Transient failure → retry. Permanent failure (definite 4xx, validation error) → fail the job. Distinguish by exception class.
4. **Per-tenant concurrency limits.** One runaway tenant cannot starve other tenants. Enforce at dequeue time.
5. **Cancellation is cooperative.** A `cancellation_requested` flag on the job row; long steps check it; cancelled jobs land in a separate status.
6. **Swap-readiness is sacred.** The `DurableExecutor` interface matches Inngest's shape (`step`, `sleep`, `wait_for_event`, parallel/serial). Implementation is ours; if we swap to Inngest, adapter is one file.
7. **Dead-letter for the unrecoverable.** After N attempts, the job moves to `failed` with full state for forensics. Never silently drop.
8. **Observability built in.** Every dequeue, attempt, step result, retry, and completion logs structured. Cloud Trace span per step.

## Always

### Schema for the jobs table

```sql
CREATE TABLE jobs (
    id                    UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id             UUID NOT NULL REFERENCES tenants(id),
    fn_name               TEXT NOT NULL,
    args                  JSONB NOT NULL,
    state                 JSONB NOT NULL DEFAULT '{}'::jsonb,   -- step results memoized here
    status                TEXT NOT NULL DEFAULT 'pending',       -- pending | running | sleeping | completed | failed | cancelled
    attempts              INT NOT NULL DEFAULT 0,
    max_attempts          INT NOT NULL DEFAULT 5,
    last_error            JSONB,
    scheduled_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    wake_at               TIMESTAMPTZ,                           -- for 'sleeping' status
    started_at            TIMESTAMPTZ,
    completed_at          TIMESTAMPTZ,
    cancellation_requested BOOLEAN NOT NULL DEFAULT false,
    idempotency_key       TEXT UNIQUE                            -- dedup at enqueue
);
CREATE INDEX jobs_dequeue ON jobs (status, scheduled_at) WHERE status IN ('pending', 'sleeping');
CREATE INDEX jobs_by_tenant ON jobs (tenant_id, status);
ALTER TABLE jobs ENABLE ROW LEVEL SECURITY;
ALTER TABLE jobs FORCE ROW LEVEL SECURITY;
CREATE POLICY tenant_isolation ON jobs
    USING (tenant_id = current_setting('app.tenant_id', true)::uuid)
    WITH CHECK (tenant_id = current_setting('app.tenant_id', true)::uuid);
```

### The `DurableExecutor` interface (Inngest-shaped)

```python
from typing import Protocol, TypeVar, Awaitable, Callable

T = TypeVar("T")

class DurableContext(Protocol):
    job_id: UUID
    tenant_id: TenantId
    attempt: int

    async def step[T](self, name: str, fn: Callable[[], Awaitable[T]]) -> T:
        """Run a step once. Cached on retry by name + job."""

    async def sleep(self, name: str, duration: timedelta) -> None:
        """Sleep until `now + duration`. Survives restarts; resumes after sleep."""

    async def wait_for_event(self, name: str, predicate: Callable[[dict], bool], timeout: timedelta) -> dict:
        """Wait for an event matching predicate, or raise on timeout."""

    async def parallel[T](self, name: str, steps: list[Callable[[], Awaitable[T]]]) -> list[T]:
        """Run steps in parallel; results memoized as a group."""

class DurableExecutor(Protocol):
    def function(self, name: str, *, max_attempts: int = 5) -> Callable: ...
    async def enqueue(self, fn_name: str, args: dict, *, tenant_id: TenantId, idempotency_key: str | None = None) -> UUID: ...
    async def cancel(self, job_id: UUID) -> None: ...
```

This shape mirrors Inngest's surface. When we swap, the adapter is mechanical.

### Defining a durable function

```python
executor = PostgresDurableExecutor(...)

@executor.function("research_topic", max_attempts=3)
async def research_topic(ctx: DurableContext, args: dict) -> dict:
    topic: str = args["topic"]

    # Step 1: gather sources (cached after success)
    sources = await ctx.step("gather_sources", lambda: gather_sources(topic, ctx.tenant_id))

    # Step 2: summarize each in parallel
    summaries = await ctx.parallel(
        "summarize_sources",
        [lambda s=s: summarize(s) for s in sources],
    )

    # Step 3: sleep an hour (resilient across restarts)
    await ctx.sleep("cooldown", timedelta(hours=1))

    # Step 4: compose final
    report = await ctx.step("compose_report", lambda: compose_report(topic, summaries))

    return {"report_id": str(report.id)}
```

Each `ctx.step(name, fn)` is keyed by `(job_id, name)`. On retry, completed step results are read from `state[name]` instead of re-executed.

### Worker loop

```python
class PostgresDurableWorker:
    def __init__(self, engine: AsyncEngine, executor: PostgresDurableExecutor, concurrency: int = 5) -> None:
        self._engine = engine
        self._executor = executor
        self._semaphore = asyncio.Semaphore(concurrency)
        self._running = True

    async def run(self) -> None:
        while self._running:
            async with self._engine.begin() as conn:
                # Dequeue up to `concurrency` jobs, respecting per-tenant limits
                rows = await conn.execute(text("""
                    UPDATE jobs SET status = 'running', started_at = now(), attempts = attempts + 1
                    WHERE id IN (
                        SELECT id FROM jobs
                        WHERE status = 'pending' AND scheduled_at <= now()
                          AND NOT EXISTS (
                              SELECT 1 FROM jobs j2
                              WHERE j2.tenant_id = jobs.tenant_id AND j2.status = 'running'
                              GROUP BY j2.tenant_id HAVING COUNT(*) >= :per_tenant_limit
                          )
                        ORDER BY scheduled_at
                        FOR UPDATE SKIP LOCKED
                        LIMIT :batch
                    )
                    RETURNING id, tenant_id, fn_name, args, state, attempts, max_attempts
                """), {"per_tenant_limit": 3, "batch": 10})
                batch = list(rows)

            if not batch:
                await asyncio.sleep(1.0)  # poll backoff; replace with LISTEN/NOTIFY for lower latency
                continue

            async with asyncio.TaskGroup() as tg:
                for row in batch:
                    tg.create_task(self._run_job(row))

    async def _run_job(self, row) -> None:
        async with self._semaphore:
            try:
                # Set tenant scope for everything this job touches
                async with AsyncSession(self._engine) as s:
                    await s.execute(text("SET LOCAL app.tenant_id = :tid"), {"tid": str(row.tenant_id)})
                    ctx = PostgresDurableContext(job_row=row, session=s)
                    fn = self._executor.get(row.fn_name)
                    try:
                        result = await fn(ctx, row.args)
                        await self._mark_completed(s, row.id, result)
                    except RetryableError as err:
                        await self._mark_for_retry(s, row, err)
                    except CancelledByCaller:
                        await self._mark_cancelled(s, row.id)
                    except Exception as err:
                        await self._mark_failed_or_retry(s, row, err)
            except Exception as err:
                logger.exception("worker_loop_error", job_id=str(row.id), exc_info=err)
```

Key choices:
- `FOR UPDATE SKIP LOCKED` — multiple workers can dequeue without contention.
- Per-tenant cap via subquery — one tenant can't hog the workers.
- Tenant context set per job so RLS protects everything the job touches.
- `TaskGroup` ensures bounded concurrency per worker.

### Step caching

```python
class PostgresDurableContext:
    def __init__(self, job_row, session: AsyncSession) -> None:
        self._row = job_row
        self._s = session

    async def step[T](self, name: str, fn: Callable[[], Awaitable[T]]) -> T:
        state = self._row.state or {}
        if name in state:
            return state[name]  # already done; skip
        result = await fn()
        # Persist before returning so a crash now leaves state consistent
        state[name] = result
        await self._s.execute(
            text("UPDATE jobs SET state = :state WHERE id = :id"),
            {"state": json.dumps(state), "id": str(self._row.id)},
        )
        self._row.state = state
        return result
```

Persist after each step, before returning. A crash mid-step loses only that one step's progress.

### Retry with backoff + jitter

```python
async def _mark_for_retry(self, s, row, err: Exception) -> None:
    if row.attempts >= row.max_attempts:
        await self._mark_failed(s, row.id, err)
        return
    delay = min(2 ** row.attempts, 600) + random.uniform(0, 10)  # cap at 10min + jitter
    next_run = datetime.utcnow() + timedelta(seconds=delay)
    await s.execute(text("""
        UPDATE jobs SET status = 'pending', scheduled_at = :next, last_error = :err
        WHERE id = :id
    """), {
        "next": next_run,
        "id": str(row.id),
        "err": json.dumps({"type": type(err).__name__, "message": str(err)}),
    })
```

`RetryableError` (transient network, 429, 5xx) triggers retry. Other exceptions terminate after the current attempt unless explicitly marked retryable.

### Enqueue with idempotency

```python
async def enqueue(
    self,
    fn_name: str,
    args: dict,
    *,
    tenant_id: TenantId,
    idempotency_key: str | None = None,
) -> UUID:
    async with self._engine.begin() as conn:
        await conn.execute(text("SET LOCAL app.tenant_id = :tid"), {"tid": str(tenant_id)})
        row = await conn.execute(text("""
            INSERT INTO jobs (tenant_id, fn_name, args, idempotency_key)
            VALUES (:tid, :fn, :args, :ik)
            ON CONFLICT (idempotency_key) DO UPDATE SET idempotency_key = jobs.idempotency_key
            RETURNING id
        """), {
            "tid": str(tenant_id),
            "fn": fn_name,
            "args": json.dumps(args),
            "ik": idempotency_key,
        })
        return row.scalar_one()
```

`ON CONFLICT (idempotency_key) DO UPDATE ... RETURNING id` returns the existing job ID instead of creating a duplicate.

### Scheduling and cron

```python
# A scheduled job — wakes up at `scheduled_at`
await executor.enqueue("daily_summary", args={"date": "2026-06-03"}, tenant_id=tid)

# Cron: Cloud Scheduler hits a /cron endpoint hourly; that endpoint enqueues per-tenant jobs.
@router.post("/cron/daily-summary", include_in_schema=False)
async def cron_daily(executor: DurableExecutorDep, tenants: TenantRepoDep) -> None:
    for tid in await tenants.all_active_ids():
        await executor.enqueue("daily_summary", args={"date": today().isoformat()}, tenant_id=tid, idempotency_key=f"daily-{today()}-{tid}")
```

Cloud Scheduler runs the cron; the durable executor runs the work. Separation of "when" and "how."

## Never

- Catching all exceptions in the worker loop and continuing silently. *Why:* the worker becomes a black hole. Log, alert, mark the job appropriately.
- Long `time.sleep` to wait for an external event. *Why:* blocks the worker. Use `ctx.sleep()` which yields back to the worker pool.
- Mutating `args` after enqueue. *Why:* original `args` is the input that retries reproduce. State changes go in `state`, not `args`.
- Skipping idempotency on steps. *Why:* on retry, you'll double-call the LLM, double-send the email, double-charge the card. Every side effect must be either idempotent by nature or wrapped in a step that's memoized.
- Running unbounded jobs in parallel for one tenant. *Why:* one tenant can blow out your worker pool, your downstream quota, and your AWS bill. Per-tenant cap is non-negotiable.
- Storing huge blobs in `state` JSONB. *Why:* the row grows unboundedly; reads and writes get slow. Store references (e.g., GCS path) in state, not the blob itself.
- Sleeping with `asyncio.sleep` inside a step expecting it to survive restart. *Why:* `asyncio.sleep` is in-memory. Use `ctx.sleep`.
- Acquiring a tenant credential outside of `vault.fetch` inside a step. *Why:* if the step retries, the credential lifetime extends. Vault fetches go inside steps for proper scoping.
- Logging `args` directly when args may contain PII or secrets. *Why:* same as general logging. Redact at the logger.
- Pinning a job to a worker. *Why:* worker dies, job dies. Jobs belong to the table; any worker can pick them up.
- Manually editing `state` to "recover" a job. *Why:* fragile, untracked. Either fix the bug and re-enqueue, or write a recovery script that's reviewable.
- Treating cancellation as a hard kill. *Why:* `cancellation_requested` is a flag; steps check it. Hard-kill leaves DB and integrations in inconsistent states.

## Pitfalls

- **`FOR UPDATE SKIP LOCKED` performance with many workers.** Scales well, but very high contention can still serialize. Index `jobs(status, scheduled_at)` is critical.
- **Per-tenant cap subquery slow with many tenants.** The example uses an EXISTS subquery; at scale (>10k tenants, high job volume) consider a `tenant_running_jobs` counter table updated by triggers.
- **JSONB state size cliff.** Once a row hits ~1MB JSONB, reads slow noticeably. Store large outputs externally (GCS) and reference them in state.
- **Idempotency key collisions across versions.** If you change the args shape, the same key could produce different jobs. Include a version prefix: `f"daily-v2-{today()}-{tid}"`.
- **Cancellation cooperating with downstream side effects.** A cancellation flag set mid-step doesn't undo what the step already did. Document side-effect boundaries.
- **`scheduled_at` with timezones.** Always UTC. Mixing local time leads to "scheduled an hour late" bugs.
- **Worker not picking up sleeping jobs on time.** `status = 'sleeping'` jobs need to flip back to `pending` when `wake_at` passes. Make the dequeue query check `wake_at` too, or have a reaper.
- **Retry storm on a downstream outage.** All jobs hitting the same failing downstream retry simultaneously. Add circuit-breaker logic per integration (e.g., last-failure-time check at the integration client).
- **Cron drift on Cloud Scheduler vs UTC.** Schedule jobs in UTC explicitly. Cloud Scheduler respects timezones; misconfig is silent.
- **`ON CONFLICT (idempotency_key)` requires `idempotency_key UNIQUE`.** Verify the constraint exists; otherwise the `ON CONFLICT` clause silently does nothing useful.
- **State JSON serialization of dataclasses/Pydantic.** Define a single serialization helper that handles `UUID`, `datetime`, `Decimal`. Inconsistent serialization breaks step caching.
- **Worker shutdown signal not respected.** Cloud Run sends SIGTERM with a 10-second grace period. Trap it, set `self._running = False`, drain in-flight jobs (or mark them `pending` for re-dequeue), then exit.
- **DB connection leak in worker.** Each job opens an `AsyncSession`. If exceptions skip the `async with`, you leak. Always use the context manager.

## Verification

Before declaring durable-execution work done:

- The `jobs` table has RLS + force + policy; verified by migration test.
- The `DurableExecutor` Protocol matches Inngest's shape; if Inngest is the swap target, an adapter file (`adapters/inngest_durable.py`) exists or is documented as the swap point.
- A test enqueues a job, restarts the worker process (kill mid-step), restarts the worker, confirms the job resumes from the correct step.
- A test retries a job that raises `RetryableError` and confirms the next attempt waits at least the backoff delay.
- A test enqueues two jobs with the same `idempotency_key` and confirms only one row exists.
- A test enqueues 10 jobs for one tenant and 10 for another; confirms one tenant's slow jobs don't block the other's.
- A test runs a job with `ctx.parallel` and confirms all steps run concurrently and results are cached on retry.
- A test cancels a job mid-flight (via `cancellation_requested`); confirms the job ends in `cancelled` status without crashing.
- A test asserts that `RetryableError` triggers retry and a generic `ValueError` does not.
- Cloud Run SIGTERM handling: a test simulates SIGTERM, asserts the worker stops accepting new jobs and drains in-flight ones (or releases them).
- No `time.sleep`, `requests`, or sync drivers in the worker code (see `python-concurrency`).
- Worker logs are structured and include `job_id`, `tenant_id`, `fn_name`, `attempt`, `step_name` where relevant.
