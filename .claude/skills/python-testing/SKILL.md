---
name: python-testing
description: Use when writing tests, configuring pytest, designing fixtures, deciding what to mock vs run real, testing FastAPI endpoints, testing async code, parametrizing variations, building test factories, or fixing flaky tests. Use whenever a change adds, modifies, or fixes behavior — every behavior change deserves a test.
---

# python-testing

## When to use this skill

Triggered for any test work: writing new tests, refactoring existing ones, configuring pytest/pytest-asyncio, designing fixtures, picking the right level (unit/integration), debugging flake. Also during review — a PR that changes behavior without a test, or fixes a bug without a regression test, fails review.

## Core principles

1. **Real Postgres, never mocked.** A DB mock that passes is meaningless. Use testcontainers for the container, transactional rollback per test for speed. Same for Redis — real container.
2. **Mock only at the network edge.** External HTTP via `respx`. Twilio/Anthropic/etc. Never `unittest.mock` for our own code — if you're mocking your own service, your seam is wrong.
3. **Tests are the second reader.** Optimize for skimming. Three-letter variable names and clever abstractions are noise; explicit setup, explicit assertion is signal.
4. **One concept per test.** A test that asserts five unrelated things is five tests in a trench coat. Parametrize for variations of one concept.
5. **Fixtures over `setUp`/`tearDown`.** Composable, scoped, type-checked. The unittest patterns don't compose well in pytest.
6. **Test the behavior, not the implementation.** If a refactor that preserves behavior breaks the test, the test was watching the wrong thing.
7. **Every bug ships with a regression test.** The test fails before the fix and passes after — both verified, both committed.
8. **Tenant-scoped by default.** Every test that touches the DB sets `app.tenant_id` (via the `tenant_session` fixture). Cross-tenant tests are a separate, explicit category.

## Always

### Configure pytest in `pyproject.toml`

```toml
[tool.pytest.ini_options]
asyncio_mode = "auto"          # async tests don't need @pytest.mark.asyncio
testpaths = ["tests"]
addopts = "-ra --strict-markers --strict-config"
markers = [
    "integration: hits real containers (Postgres, Redis); slower",
    "slow: skip in default runs",
]
```

`asyncio_mode = "auto"` is non-negotiable — `pytest-asyncio` defaults to strict mode where every async test needs a decorator, which is noise.

### Use real Postgres via testcontainers, transactional rollback per test

```python
# tests/conftest.py
import pytest
import pytest_asyncio
from sqlalchemy.ext.asyncio import AsyncSession, async_sessionmaker, create_async_engine
from testcontainers.postgres import PostgresContainer

@pytest.fixture(scope="session")
def pg_container():
    with PostgresContainer("postgres:16-alpine") as pg:
        yield pg

@pytest_asyncio.fixture(scope="session")
async def engine(pg_container):
    engine = create_async_engine(pg_container.get_connection_url().replace("psycopg2", "asyncpg"))
    async with engine.begin() as conn:
        await conn.run_sync(Base.metadata.create_all)
    yield engine
    await engine.dispose()

@pytest_asyncio.fixture
async def db(engine) -> AsyncIterator[AsyncSession]:
    """One transaction per test, rolled back at the end. Fast."""
    connection = await engine.connect()
    trans = await connection.begin()
    session = AsyncSession(bind=connection, expire_on_commit=False)
    try:
        yield session
    finally:
        await session.close()
        await trans.rollback()
        await connection.close()
```

Container starts once per session. Each test gets an open transaction that's rolled back at the end. Net effect: real Postgres semantics, sub-second per test.

### Tenant-scoped DB fixture

```python
@pytest_asyncio.fixture
async def tenant(db: AsyncSession) -> Tenant:
    t = Tenant(id=uuid4(), name="test-tenant")
    db.add(t)
    await db.flush()
    return t

@pytest_asyncio.fixture
async def tenant_session(db: AsyncSession, tenant: Tenant) -> AsyncSession:
    """A DB session with RLS scoped to the test tenant."""
    await db.execute(text("SET LOCAL app.tenant_id = :tid"), {"tid": str(tenant.id)})
    return db
```

Use `tenant_session` in tests that touch tenant-scoped tables. RLS policies will enforce isolation — your tests verify they work, not just that you wrote `WHERE tenant_id = ?`.

### FastAPI testing with `httpx.AsyncClient`

```python
# tests/conftest.py
@pytest_asyncio.fixture
async def client(db: AsyncSession) -> AsyncIterator[AsyncClient]:
    async def override_db():
        yield db
    app.dependency_overrides[get_db] = override_db
    transport = ASGITransport(app=app)
    async with AsyncClient(transport=transport, base_url="http://test") as c:
        yield c
    app.dependency_overrides.clear()
```

No live server, no port collisions, no shutdown ordering issues. Same `app` object, in-process.

### Mock external HTTP with `respx`

```python
import respx
from httpx import Response

async def test_anthropic_retry_on_429(client: AsyncClient):
    with respx.mock(base_url="https://api.anthropic.com") as mock:
        mock.post("/v1/messages").mock(side_effect=[
            Response(429, headers={"retry-after": "1"}),
            Response(200, json={"content": [{"type": "text", "text": "ok"}]}),
        ])
        resp = await client.post("/v1/threads/abc/messages", json={"text": "hi"})
        assert resp.status_code == 200
        assert mock.calls.call_count == 2
```

`respx` intercepts at the httpx transport layer. No real network. Asserts on call count and arguments are first-class.

### Factories for test data

```python
# tests/factories.py
from polyfactory.factories.pydantic_factory import ModelFactory

class UserFactory(ModelFactory[User]):
    __model__ = User
    __set_as_default_factory_for_type__ = True

class CreateThreadInFactory(ModelFactory[CreateThreadIn]):
    __model__ = CreateThreadIn
    title = "test thread"
```

Polyfactory generates realistic data based on Pydantic field types. Override per test where the value matters; let the factory fill the rest.

For non-Pydantic objects, plain factory functions:

```python
def make_message(*, thread_id: UUID, role: str = "user", text: str = "hello") -> Message:
    return Message(id=uuid4(), thread_id=thread_id, role=role, parts=[TextPart(text=text)])
```

### Parametrize for variations of one concept

```python
@pytest.mark.parametrize(
    ("plan", "msg_count", "expected"),
    [
        ("free",  10,  True),
        ("free",  100, False),
        ("pro",   1000, True),
        ("pro",   10001, False),
    ],
    ids=["free-under", "free-over", "pro-under", "pro-over"],
)
def test_message_quota(plan: str, msg_count: int, expected: bool):
    assert can_send_message(plan, msg_count) is expected
```

The `ids=` parameter gives readable test names. Without it, parametrize generates `test_message_quota[free-10-True]` — fine, but custom IDs are clearer.

### Async tests are just `async def`

```python
async def test_create_thread(client: AsyncClient, tenant: Tenant):
    resp = await client.post("/v1/threads", json={"title": "first thread"})
    assert resp.status_code == 201
    assert resp.json()["title"] == "first thread"
```

With `asyncio_mode = "auto"`, no decorator needed.

### Regression tests for bugs

When fixing a bug:

1. Write the test that demonstrates the bug. Confirm it fails (`pytest -k regression_xyz` shows red).
2. Apply the fix. Confirm the same test passes.
3. Commit the test in the same PR as the fix.

Name regression tests after the symptom, not the cause:

```python
async def test_thread_list_excludes_other_tenants_after_user_switch(...):
    # regression: thread list cached per user but not invalidated on tenant switch
    ...
```

## Never

- `unittest.mock` for our own code. *Why:* mocking your own seam means your design is wrong, or your test is checking that you called your own code, which is not a behavior. The exception: third-party HTTP via `respx` (which is purpose-built, not raw `mock`).
- Mocking the database. *Why:* DB mocks pass when real code breaks. Use the testcontainers transactional fixture.
- Mocking `AsyncSession`, `Engine`, or `text()`. *Why:* same as above. Use a real session.
- `monkeypatch` for our own modules. *Why:* if you need to patch your own code to test it, your dependencies aren't injected. Fix the design.
- `setup_method` / `teardown_method` / `setUp` / `tearDown`. *Why:* fixtures compose; class lifecycle methods don't. We're not on unittest.
- Shared mutable state across tests. *Why:* test order matters then, flake follows. Fixtures are per-test by default; lift to higher scope deliberately.
- `assert foo` with no message in non-obvious assertions. *Why:* `AssertionError` with no context is debugging hell. `assert foo, f"expected x but got {actual}"`.
- `time.sleep` to "wait for the async to finish." *Why:* either the test is wrong about ordering, or it's racing. Use proper awaiting.
- Real network calls in unit tests. *Why:* flake, cost, slowness, leaks. `respx` for HTTP; real containers for DB/Redis.
- `pytest.skip` on a flaky test. *Why:* the flake is information. Either fix it or quarantine to `@pytest.mark.slow` with an issue number.
- Tests that print to stdout. *Why:* `pytest -s` becomes the only way to see output, breaks `-q` mode. Use `caplog` to assert on logs.
- Catching exceptions in tests to assert "no exception." *Why:* if no exception is the expected behavior, just let it run. If an exception is expected, use `pytest.raises`.
- Tests that depend on each other. *Why:* test isolation is the whole point. Each test must work alone, in any order, in parallel.
- 100%-coverage chasing. *Why:* coverage is a floor, not a target. Critical paths first; trivial getters last. Coverage is a tool to find gaps, not a number to maximize.

## Pitfalls

- **`pytest-asyncio` strict mode noise.** Default to `asyncio_mode = "auto"`. Strict mode requires `@pytest.mark.asyncio` on every async test — that's hundreds of decorators of nothing.
- **Fixture scope mismatch.** A session-scoped fixture that uses a function-scoped fixture is an error. The Postgres container is session-scoped; the transaction is function-scoped. Don't flip them.
- **`async with` vs `yield` in async fixtures.** Use `@pytest_asyncio.fixture` (not `@pytest.fixture`) for async fixtures. Pytest's plain fixture decorator does not understand async generators correctly in older versions.
- **`AsyncClient` without `ASGITransport`.** `AsyncClient(app=app)` is deprecated in modern httpx. Use `AsyncClient(transport=ASGITransport(app=app))`.
- **`dependency_overrides` not cleared between tests.** Lift to a fixture that clears in teardown. Otherwise overrides leak into the next test in the same module.
- **Tenant context not set in tests touching tenant tables.** RLS will return zero rows and you'll think your code is broken. Use the `tenant_session` fixture.
- **`respx` not asserting calls.** A `respx` mock that's never hit silently passes. Assert `mock.calls.call_count == N` to verify the call happened.
- **Polyfactory generating randomness on every run.** For tests where value matters, override explicitly. For value-agnostic tests, randomness is fine and catches "you assumed alphabetical."
- **`testcontainers` slow to start in CI.** Use a session-scoped container (it starts once for the entire test run), and run the test suite in a single process for the container path. Parallelism via `xdist` requires shared connection pools or one container per worker.
- **Transactional rollback not enough for triggers/sequences.** Sequences advance even on rollback. Tests asserting "id=1" after the rollback fixture break. Don't assert on sequence values.
- **`session.flush()` vs `session.commit()` in fixtures.** Use `flush()` to make data visible to the same session without ending the transaction. `commit()` would defeat the rollback fixture.
- **Test that "passes" but the assertion is `assert True` after a refactor.** Add a `caplog` or a return-value check — assertion without observable behavior is no assertion.
- **`pytest.raises` matching too broadly.** `pytest.raises(Exception)` matches everything. Match the specific class: `pytest.raises(ToolNotFound)`, and assert on the exception's typed attributes.
- **`asyncio.gather` in test setup masking errors.** If a setup fan-out fails partway, `gather` may swallow with `return_exceptions=True`. Use `TaskGroup`.
- **Fixture circular dependency.** `db` depends on `engine` depends on `pg_container`. Adding `db` as a dep of `pg_container` would be circular — pytest tells you, but the error is cryptic.

## Verification

Before declaring testing work done:

- `uv run pytest` passes from clean clone with no warnings.
- `rg "from unittest.mock" tests/ src/` returns nothing except where deliberately allowed and commented.
- `rg "MagicMock|Mock\(\)|patch\(" tests/ src/` returns nothing in `src/`; in `tests/` only inside an allowed boundary helper.
- Every PR that fixes a bug has a test named after the symptom; the test fails on `git stash` of the fix and passes when restored.
- A randomized test order (`pytest -p random_order` if installed, or running individual files) still passes — proves isolation.
- A test asserts that RLS denies cross-tenant access (write data as tenant A, switch session to tenant B, assert zero rows).
- `caplog` is used to assert critical log lines (e.g., security events, error logs) — not just that "something happened."
- Average test runtime is acceptable (<2s for the median test, <30s for the suite at this size). If degraded, profile (`pytest --durations=20`).
- Container startup happens once per session; per-test cost is rollback, not migration.
