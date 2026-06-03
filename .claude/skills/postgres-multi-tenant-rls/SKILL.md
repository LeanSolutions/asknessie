---
name: postgres-multi-tenant-rls
description: Use when designing tables for multi-tenant data, configuring Row-Level Security, opening DB sessions, writing migrations that add new tables, debugging "why am I seeing zero rows", or auditing tenant isolation. Use whenever Postgres + tenant_id meet in the same change.
---

# postgres-multi-tenant-rls

## When to use this skill

Triggered for any tenant-data change: new table, new column on a tenant table, new query, session lifecycle, migrations, backfill scripts, audit reviews, debugging "this works for tenant A but not B." This is the #1 highest-blast-radius surface in a multi-tenant SaaS — get it right by construction.

## Core principles

1. **RLS is the only tenant filter.** Application code does not write `WHERE tenant_id = ?`. Postgres does it via policy. If you're writing the WHERE in Python, the session isn't scoped — fix that.
2. **`tenant_id` is `UUID NOT NULL` on every tenant table.** No nullable. No defaults that could create unscoped rows. Foreign key to `tenants(id)` for joins; the RLS policy enforces the isolation.
3. **Session variable is the binding.** `SET LOCAL app.tenant_id = '<uuid>'` at the start of every request's transaction. The policy reads this variable. No tenant context = no rows.
4. **Default-deny by design.** Tables have RLS enabled with no permissive policies until you add them. The fail-closed mode is "see nothing."
5. **`FORCE ROW LEVEL SECURITY`.** Without this, the table owner role bypasses RLS — which means if your app connects as the owner, isolation is silently disabled. Always force.
6. **Indexes lead with `tenant_id`.** Queries are tenant-scoped; the index should be too.
7. **RLS is tested.** A cross-tenant test (write as A, switch to B, assert no rows) runs in CI for every tenant table.
8. **Bypass is rare, audited, narrow.** Admin tooling that needs cross-tenant view uses a separate connection with a different role that's been explicitly granted bypass — and every such call writes an audit row.

## Always

### Schema for a tenant-scoped table

```sql
CREATE TABLE threads (
    id          UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id   UUID NOT NULL REFERENCES tenants(id) ON DELETE RESTRICT,
    title       TEXT NOT NULL,
    status      TEXT NOT NULL DEFAULT 'active',
    created_at  TIMESTAMPTZ NOT NULL DEFAULT now(),
    archived_at TIMESTAMPTZ
);

-- Index leads with tenant_id; second column matches your query patterns
CREATE INDEX threads_by_tenant_created ON threads (tenant_id, created_at DESC);

-- Enable + force RLS
ALTER TABLE threads ENABLE ROW LEVEL SECURITY;
ALTER TABLE threads FORCE ROW LEVEL SECURITY;

-- Policy: rows match the session's tenant_id
CREATE POLICY tenant_isolation ON threads
    USING (tenant_id = current_setting('app.tenant_id', true)::uuid)
    WITH CHECK (tenant_id = current_setting('app.tenant_id', true)::uuid);
```

The `, true` in `current_setting` makes it return NULL if unset instead of raising — combined with the UUID cast, an unset session var means no row matches, which is the fail-closed behavior we want.

`WITH CHECK` enforces on inserts/updates: you can't write a row with a different `tenant_id`. Without it, RLS only filters reads.

### Set the session variable in `get_db`

```python
async def get_db(
    request: Request,
    ctx: TenantDep,
) -> AsyncIterator[AsyncSession]:
    engine: AsyncEngine = request.app.state.engine
    async with AsyncSession(engine) as s:
        await s.execute(
            text("SET LOCAL app.tenant_id = :tid"),
            {"tid": str(ctx.tenant_id)},
        )
        yield s
```

`SET LOCAL` scopes the var to the current transaction. When the session closes, the var goes away. Connection pooling returns the connection to the pool clean. No leak between requests.

### Query without `tenant_id` in the WHERE

```python
# Yes — RLS handles it
async def list_threads(s: AsyncSession, limit: int) -> list[Thread]:
    rows = await s.execute(
        select(ThreadRow).order_by(ThreadRow.created_at.desc()).limit(limit)
    )
    return [row_to_thread(r) for r in rows.scalars()]
```

```python
# No — redundant, and a signal the session isn't scoped if you "had to add" this
async def list_threads(s: AsyncSession, tenant_id: UUID, limit: int) -> list[Thread]:
    rows = await s.execute(
        select(ThreadRow)
        .where(ThreadRow.tenant_id == tenant_id)  # ← RLS already does this
        .order_by(ThreadRow.created_at.desc())
        .limit(limit)
    )
```

If a code reviewer sees `.where(ThreadRow.tenant_id == ...)`, that's a red flag — either RLS isn't on, or the session isn't scoped.

### Migrations enable RLS for every new tenant table

```python
# alembic migration
def upgrade() -> None:
    op.create_table(
        "messages",
        sa.Column("id", sa.UUID, primary_key=True),
        sa.Column("tenant_id", sa.UUID, sa.ForeignKey("tenants.id"), nullable=False),
        sa.Column("thread_id", sa.UUID, sa.ForeignKey("threads.id"), nullable=False),
        # ...
    )
    op.create_index("messages_by_tenant_thread", "messages", ["tenant_id", "thread_id"])
    op.execute("ALTER TABLE messages ENABLE ROW LEVEL SECURITY")
    op.execute("ALTER TABLE messages FORCE ROW LEVEL SECURITY")
    op.execute("""
        CREATE POLICY tenant_isolation ON messages
        USING (tenant_id = current_setting('app.tenant_id', true)::uuid)
        WITH CHECK (tenant_id = current_setting('app.tenant_id', true)::uuid)
    """)
```

A pre-commit lint or migration test asserts that every new table with `tenant_id` has both `ENABLE` and `FORCE` plus a policy. This is the kind of check that catches the inevitable "forgot to enable RLS on the new table" mistake.

### Backfill scripts must set tenant_id explicitly per batch

```python
async def backfill_message_counts(engine: AsyncEngine, dry_run: bool = True) -> None:
    async with engine.begin() as conn:
        tenant_ids = (await conn.execute(text("SELECT id FROM tenants"))).scalars().all()
    for tid in tenant_ids:
        async with engine.begin() as conn:
            await conn.execute(text("SET LOCAL app.tenant_id = :tid"), {"tid": str(tid)})
            # ... work that's RLS-bounded to this tenant
            await conn.commit() if not dry_run else None
```

Backfills running without a tenant context see zero rows and "silently succeed." Always loop tenant-by-tenant.

### Admin role for bypass

```sql
-- A separate role for cross-tenant admin queries
CREATE ROLE asknessie_admin NOLOGIN;
GRANT asknessie_admin TO asknessie_admin_user;

-- Bypass for explicitly granted tables
ALTER TABLE threads OWNER TO asknessie_admin;
-- Admin role can bypass; app role cannot
GRANT BYPASSRLS ON ROLE asknessie_admin;
```

The app connects as `asknessie_app` (subject to RLS); admin tools connect as `asknessie_admin` with `BYPASSRLS`. Every admin query writes to an `audit_log` table. Use as rarely as possible.

### Test cross-tenant isolation

```python
async def test_thread_list_isolated_per_tenant(db: AsyncSession):
    t_a = make_tenant()
    t_b = make_tenant()
    db.add_all([t_a, t_b])
    await db.flush()

    await db.execute(text("SET LOCAL app.tenant_id = :tid"), {"tid": str(t_a.id)})
    db.add(ThreadRow(id=uuid4(), tenant_id=t_a.id, title="A's thread"))
    await db.flush()

    # Switch tenant
    await db.execute(text("SET LOCAL app.tenant_id = :tid"), {"tid": str(t_b.id)})
    rows = (await db.execute(select(ThreadRow))).scalars().all()
    assert rows == []  # B sees none of A's data
```

This test runs against the real Postgres testcontainer (see `python-testing`). Mocking the DB would miss this — RLS isn't running against a mock.

## Never

- `WHERE tenant_id = ?` in application queries against tenant tables. *Why:* RLS does this. Redundant filtering means the session isn't scoped — fix that.
- `tenant_id` column nullable. *Why:* a NULL `tenant_id` matches no policy and creates ghost rows that no tenant can see (or all can, depending on policy).
- Connecting as a role with `BYPASSRLS` for normal app traffic. *Why:* silently disables isolation. The whole protection model is the role + the policy together.
- Omitting `FORCE ROW LEVEL SECURITY`. *Why:* table owner bypasses RLS. If your app role *is* the owner, isolation is silently off.
- `SET app.tenant_id = ...` (without `LOCAL`). *Why:* the variable persists on the connection, leaks into the next request that pulls the same connection from the pool. Always `SET LOCAL` inside a transaction.
- Disabling RLS to "fix" a failing query. *Why:* the query isn't failing because of RLS; it's failing because the session isn't tenant-scoped. Find the session, fix the scoping.
- Caching DB rows across requests on the same worker. *Why:* the cache becomes a cross-tenant leak. Cache by `(tenant_id, ...)` key with explicit invalidation, or don't cache at all.
- Foreign keys from a tenant table to a row in another tenant. *Why:* an aggregate that spans tenants isn't an aggregate. Disallow at schema or app level.
- Migrations that forget to enable RLS on a new tenant table. *Why:* the new table is unprotected. Add a migration test that asserts every new table with `tenant_id` has RLS enabled and forced with a policy.
- Background jobs running without a tenant binding. *Why:* zero rows visible; the job "succeeds" doing nothing. Either set tenant per batch, or use a documented admin role for cross-tenant jobs.
- Logging `tenant_id` lookups that reveal another tenant's existence. *Why:* "tenant X not found" in tenant Y's logs leaks info. See `python-exception-handling` cross-tenant rules.
- Storing pgvector embeddings without `tenant_id` in the index. *Why:* nearest-neighbor across all tenants is a leak. The index must be tenant-scoped.

## Pitfalls

- **`SET LOCAL` outside a transaction is a no-op (or session-scoped depending on PG version).** Make sure the session is inside a transaction when you set the var. SQLAlchemy's `AsyncSession` defaults to autobegin; verify with a small test.
- **Connection pooler resetting the session variable.** Some poolers (pgbouncer in `transaction` mode) recycle connections between transactions; `SET` (without LOCAL) is then lost. `SET LOCAL` is safe with transaction-mode pooling.
- **pgvector ANN index without `tenant_id`.** A search returns nearest neighbors across all tenants. Use a partitioned index or a `WHERE` clause inside the policy. With ScaNN on AlloyDB, ensure the index includes `tenant_id` as a filter column.
- **`USING` policy without `WITH CHECK`.** Reads filter correctly; writes can sneak in a different `tenant_id`. Always add `WITH CHECK`.
- **`SECURITY DEFINER` functions bypassing RLS.** Functions defined with `SECURITY DEFINER` run as the owner, not the caller. If owner has `BYPASSRLS`, RLS is off inside the function. Audit your functions.
- **JSONB columns containing other tenants' IDs.** RLS protects rows, not column values. A row I'm allowed to see might contain a `referrer_user_id` from another tenant — fine for some use cases, leak for others. Be explicit at the application layer.
- **`CURRENT_USER` vs the session var.** The policy uses `current_setting('app.tenant_id')`, not `current_user`. Don't conflate them.
- **Multi-statement queries where one statement is missing the scope.** Triggers, generated columns, deferred constraints — all run in the same transaction; the session var applies. But a CTE that joins with an "unscoped" subquery is impossible to write because RLS applies per-table — the protection is automatic.
- **Type cast failures on policy.** `current_setting('app.tenant_id', true)::uuid` raises if the value isn't a valid UUID. Sanitize the binding (always cast at the Python side too, never accept a raw string from a header).
- **Migration ordering of policy vs index.** Create the policy after the column exists; create indexes before enabling RLS so they're built without the read-side filter overhead.
- **Read replicas and RLS.** RLS applies on replicas too. If you query a read replica from a job that didn't `SET LOCAL`, you see nothing. Same fix as primary: tenant context per session.
- **AlloyDB columnar engine missing `tenant_id` partitioning.** When you enable the columnar engine for a table, partition by `tenant_id` to keep per-tenant scans fast.

## Verification

Before declaring tenant-isolation work done:

- Every tenant-scoped table has `tenant_id UUID NOT NULL`.
- Every tenant-scoped table has `ENABLE ROW LEVEL SECURITY` and `FORCE ROW LEVEL SECURITY` and at least one policy with both `USING` and `WITH CHECK`.
- Every tenant-scoped table has at least one index leading with `tenant_id`.
- `rg "WHERE.*tenant_id" src/` returns no hits in app code (test data setup is fine; flag if it's anywhere in repos or services).
- `rg "SET app\.tenant_id" src/` is empty (no plain SET); `SET LOCAL` only.
- The `get_db` dep sets `SET LOCAL app.tenant_id` and a test confirms swapping tenants between requests shows the right rows.
- A cross-tenant test in CI: insert as A, switch session to B, assert empty result set.
- A migration test asserts every new table with a `tenant_id` column has RLS enabled, forced, and a policy.
- A `BYPASSRLS` audit: `SELECT rolname FROM pg_roles WHERE rolbypassrls;` returns only the documented admin role.
- A pgvector / ScaNN index includes `tenant_id` for tenant-scoped vector queries.
- Backfill scripts loop per-tenant with explicit `SET LOCAL`.
- Logs around tenant errors don't reveal other tenants' existence.
