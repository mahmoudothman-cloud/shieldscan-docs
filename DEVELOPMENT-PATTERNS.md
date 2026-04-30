# ShieldScan — Development Patterns

**Purpose:** Capture implementation patterns that aren't obvious from reading the code, but that future engineers will keep rediscovering if they aren't documented. Each entry is a short "when you hit X, do Y, here's why" — the kind of knowledge that otherwise lives only in expensive debugging sessions.

**Scope:** Application-side patterns (Python / FastAPI / SQLAlchemy / Pydantic / Redis). Operational patterns (deploys, runbooks, incidents) live in [OPERATIONS-RUNBOOK.md](OPERATIONS-RUNBOOK.md). Architectural decisions (one-way doors) live in ADRs in [SPECIFICATION.md](SPECIFICATION.md).

**Rule:** Add an entry the moment a debugging session uncovers a non-obvious pattern. The cost of writing it is one debugging session; the cost of not writing it is N future debugging sessions × engineers.

---

## SQLAlchemy Identity-Map Staleness on Post-Write Reads

### Context

`AsyncSessionLocal` (in [`src/app/db/__init__.py`](../shieldscan-api/src/app/db/__init__.py)) is configured with `expire_on_commit=False`. This is **load-bearing** — the session-local PostgreSQL GUC `app.current_org_id` (used by RLS) must survive `commit()`, which only happens if cached ORM state survives commits too. The default `expire_on_commit=True` would invalidate the GUC on every commit and break tenant isolation.

### The trap

With `expire_on_commit=False`, the session **identity map** also survives commits. Subsequent SELECTs against the same session for the same primary key return the **cached snapshot**, not the freshly-written DB state.

This shows up as:
- An UPSERT (or UPDATE) writes a row.
- A follow-up SELECT for the same row returns stale data.
- The endpoint's response shows the *previous* state, not the just-written state.
- Tests that exercise back-to-back writes on the same row fail in subtle ways.

The bug is silent in production for most flows because each request opens a fresh session — so there's no prior cached row to be stale. It surfaces hardest in the test fixture (which deliberately reuses one session across the whole test) and in any production endpoint that writes-then-reads the same row in one handler.

### The fix

Use the `select_fresh` helper:

```python
from app.db import select_fresh

cred = (
    await select_fresh(
        db,
        select(ProjectCredential).where(
            ProjectCredential.project_id == project_id
        ),
    )
).scalar_one()
```

`select_fresh` is a thin wrapper that passes `execution_options={"populate_existing": True}` to `db.execute(...)`. That option instructs SQLAlchemy to **re-hydrate the ORM object from the SELECT result**, bypassing the identity map for this query while leaving unrelated cached rows alone.

### When to reach for it

- After an UPSERT (`INSERT ... ON CONFLICT DO UPDATE`) when you need the post-write row.
- After an UPDATE when you need fresh column values for response shape.
- Any handler that writes a row and then reads it back to compute the response body.
- Any test that mutates a row via raw SQL (or a different mechanism) and then asserts on the result through the same session.

### What does NOT work

- **`session.expire_all()`** — raises `MissingGreenlet` in async context. The method triggers IO outside a greenlet boundary, which `AsyncSession` rejects.
- **`RETURNING (Model)` on the upsert** — the ORM still hands back the cached object for a row whose primary key already lives in the identity map. The SQL emits `RETURNING`, but the SQLAlchemy result-processor short-circuits to the cache.
- **Switching to `expire_on_commit=True`** — breaks the GUC-survives-commit contract that RLS depends on. Don't.
- **A separate `db_fresh` session per call** — possible, but multiplies connections and complicates RLS context. Heavyweight.

### Provenance

First discovered: **Task 3.X PATCH /credentials** — a second PATCH on the same project returned the first PATCH's `auth_type` because the cached `ProjectCredential` ORM object overrode the SELECT result. Fix landed in [`shieldscan-api` commit `c7a12e9`](#) inline; extracted into `select_fresh` helper + this doc in the immediate follow-up.

Pinned by: [`tests/test_select_fresh.py`](../shieldscan-api/tests/test_select_fresh.py) — uses raw-SQL UPDATE to make DB state diverge from cached state, then asserts plain SELECT returns the stale row and `select_fresh` returns the fresh one.


---

## Long-lived Background Tasks: `session_factory` DI

### Context

Some components run as long-lived `asyncio` tasks rather than per-request handlers — the M4 `CompletionsConsumer` is the first; M5 worker dispatch listeners and the M5+ ghost-queued-scan retry janitor will follow. They open a new `AsyncSession` per event (or per work-cycle) rather than receiving one from FastAPI's request-scoped DI.

In production this is straightforward: the component takes a `session_factory` parameter and is constructed with the global `AsyncSessionLocal`. Each event opens a fresh session from the engine's pool, sets RLS context (`SET app.current_org_id`), does its work, commits, closes.

### The trap

In tests, reaching for the global `AsyncSessionLocal` fails — but in a confusing way.

`AsyncSessionLocal` (in [`src/app/db/__init__.py`](../shieldscan-api/src/app/db/__init__.py)) is constructed at module-import time from a global `engine`. The engine binds to the first event loop that uses it. With pytest-asyncio's per-test event-loop scoping (necessary for the function-scoped `db_session` fixture to avoid "attached to a different loop" asyncpg errors — see CLAUDE.md gotcha 3 + the conftest comment), the second test that touches `AsyncSessionLocal` finds an engine bound to a stale loop. The asyncpg connection then races against the previous test's leftover state and raises:

```
asyncpg.exceptions._base.InterfaceError:
    cannot perform operation: another operation is in progress
```

The error message is misleading — there's no concurrent operation in your code; it's the connection-pool/loop mismatch surfacing as if there were one.

### The fix

Inject a `session_factory` rather than reaching for the module-global. Production binds to `AsyncSessionLocal` directly. Tests bind to a factory that yields sessions on the test fixture's existing connection.

**Production wiring** (FastAPI `lifespan` in `main.py`):

```python
from app.db import AsyncSessionLocal
consumer = CompletionsConsumer(session_factory=AsyncSessionLocal, redis=redis)
await consumer.start()
```

**Test wiring** (pytest fixture):

```python
@pytest_asyncio.fixture
async def session_factory(db_session):
    """Each "fresh" session shares db_session's connection.
    Same RLS context (SET ROLE shieldscan_app + GUC), same event
    loop, no pool contention."""
    from sqlalchemy.ext.asyncio import AsyncSession as _S
    connection = db_session.bind  # AsyncConnection bound to test fixture

    class _Factory:
        async def __aenter__(self):
            self._session = _S(bind=connection, expire_on_commit=False)
            return self._session

        async def __aexit__(self, *exc):
            await self._session.close()
            # Don't close the connection — db_session owns its lifecycle.

    return lambda: _Factory()
```

The test's "fresh" sessions all share the test connection. That's semantically equivalent to production for unit-test purposes:
- RLS context survives (the connection has `SET ROLE` + `SET app.current_org_id` once; subsequent sessions inherit).
- Committed visibility is consistent (same connection, no cross-connection isolation surprise).
- `expire_on_commit=False` matches `AsyncSessionLocal`'s configuration.

### When to reach for it

Anywhere a component has this shape:

```python
class SomeBackgroundTask:
    def __init__(self, session_factory, redis):
        self._session_factory = session_factory
        ...
    async def _handle(self, event):
        async with self._session_factory() as db:
            ...  # per-event work
```

Don't write `async with AsyncSessionLocal() as db:` inline — even though it's tempting. Tests will pay for it.

### Bonus: non-request RLS context

The same component pattern raises a second concern: there's no incoming auth, no `require_org_membership` upstream, no GUC pre-set. Each event must SET its own `app.current_org_id` from a trusted field on the consumed event:

```python
async def _handle(self, event: dict) -> None:
    async with self._session_factory() as db:
        await db.execute(
            text(f"SET app.current_org_id = '{event['organization_id']}'")
        )
        # ... RLS-scoped queries follow
```

The org_id comes from the event's payload. The Go worker is the only legitimate emitter, but the channel could in principle be spoofed by anyone with Redis write access. Defense-in-depth posture: Redis runs inside our trust boundary (private VPC). Future hardening might HMAC-sign completion events at the Go side and verify in the consumer. Track that as ops-hardening-milestone work.

### Provenance

First discovered: **Task 4.2 `CompletionsConsumer`** ([`shieldscan-api` commit `cf3b30a`](#)). The asyncpg "another operation in progress" error surfaced when the second test in the file ran against the global engine. Fix in the test fixture; production code unchanged.

Pinned by: tests in [`tests/services/test_completions_consumer.py`](../shieldscan-api/tests/services/test_completions_consumer.py) all consume the `session_factory` fixture rather than the global `AsyncSessionLocal`.


---

## API-key Audit Attribution

### Context

Audit rows track *who* did *what* via `audit_logs.actor_id` (FK to `users`). The original M2 design assumed a 1-to-1 between authenticated requests and a user identity — JWT auth carries a `sub` claim that resolves to a user, populates `actor_id`, done.

**API-key auth breaks that assumption.** API keys belong to an organization, not a user (per ADR-012 + the M3 mobile upload schema correction). The credential authenticates the request as the org but doesn't identify a specific human actor.

### The pattern

When emitting an audit row from an endpoint that accepts both JWT and API-key auth (via `require_org_membership()`), populate `actor_id` only on the JWT path. On the API-key path, leave it `None` and surface the calling key's prefix in the audit `details` payload:

```python
actor_id = identity.user.id if identity.user is not None else None
details: dict = {
    "scan_type": scan.scan_type.value,
    # ... domain-specific fields
}
if identity.api_key is not None:
    details["api_key_prefix"] = identity.api_key.key_prefix

await audit(
    db,
    organization_id=org_id,
    actor_id=actor_id,
    action=ScanAction.SCAN_DISPATCHED,
    resource_type="scan",
    resource_id=scan.id,
    ip_address=get_real_ip(request),
    user_agent=request.headers.get("User-Agent"),
    details=details,
)
```

### Why `key_prefix` and not a full key reference

The first 12 characters of an API key (e.g. `ss_live_abcd`) are intentionally non-secret — surfaced in the API key list view, used by customer support to identify which key a customer is asking about. They're long enough to be useful for identification (a customer with 3 keys can tell which one fired) but short enough that they don't leak the secret tail.

A full FK to `api_keys.id` would also work but couples the audit log to the api_keys table — when a key is revoked + GC'd, the audit row's reference would either dangle or auto-NULL. Storing the prefix as a string keeps the audit row self-contained and human-readable in ops review without joins.

### Why `actor_id IS NULL` instead of a synthetic system user

An alternative design would seed an `api_key_user` row in `users` and point `actor_id` there for API-key audits. Rejected because:

- It conflates "no user actor" with "this synthetic actor" — ops review would have to know the synthetic user is special.
- The synthetic user can't be deleted (it's referenced by audit rows), creating a permanent special case in the users table.
- `audit_logs.actor_id` was `nullable=True` from M2 specifically for this case.

`NULL + details.api_key_prefix` is the cleaner shape: the audit row says exactly what's true. No human triggered this; this API key did.

### When to reach for it

Any endpoint whose auth dep is `require_org_membership()` (the default factory — JWT or API key) and that emits an audit row. Currently four implementations:

- Task 3.3 — `POST /mobile-uploads`
- Task 3.X — `PATCH /credentials` + `DELETE /credentials`
- Task 4.5 — `DELETE /scans/:id` (cancel)
- Task 4.6 — `POST /scans/compare`

The pattern is also baked into Task 4.2's `ScanOrchestrator.dispatch()` (which receives an `AuthIdentity` and emits `SCAN_DISPATCHED` directly) — the orchestrator is just the first non-route surface that encountered it.

### What does NOT belong in `details`

- The full API key plaintext or hash. Plain leaks the credential; hash provides no operational benefit and couples the audit log to the api_keys table's SHA strategy.
- The IP / user-agent. Those have dedicated top-level columns on `audit_logs` (INET + Text respectively, indexable for CIDR queries). Putting them in JSONB defeats the indexing.
- Domain-secret content. Audit details for a credential write should never include the credential value (Task 3.X pinned this with the response-doesn't-return-decrypted regression test).

### Provenance

First implementation: **Task 3.3** ([`shieldscan-api` commit `d9b0fbf`](#)) — mobile upload via API key. Re-applied in 3.X, 4.5, 4.6. Triple-pin precedent established; promoted from per-task convention to documented pattern at Task 4.6.

