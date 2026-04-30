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

