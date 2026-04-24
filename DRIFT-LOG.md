# ShieldScan — Version Drift Log

**Purpose:** Track versions that have drifted past VERSIONS.md pins, so deliberate upgrade sessions can be scheduled.

**Rule:** Per CONSTITUTION.md §11.1 Commandment #1, pinned versions win. Entries here are *candidates* for a future upgrade session, not silent upgrades.

---

## Open drift (as of 2026-04-20)

Captured from `scripts/verify-versions.sh` run at project start. Re-run before every upgrade session to refresh.

### Python libraries (shieldscan-api)

| Library | Pinned | Latest observed | Classification | Target milestone to address |
|---|---|---|---|---|
| fastapi | 0.135.0 | 0.136.0 | patch — caret range handles at `poetry install` | No action; verify at M11 |
| sqlalchemy | 2.0.36 | 2.0.49 | patch — caret range handles | No action; verify at M11 |
| pydantic | 2.9.0 | 2.13.3 | minor drift, 4 minor versions | **Review before M9** (AI pipeline uses pydantic schemas heavily) |
| **anthropic** | 0.40.0 | 0.96.0 | **MAJOR SDK rewrite**, 56 minor versions | **Dedicated upgrade session required before M9** — breaking API changes expected |

### Security scanning tools (shieldscan-engine)

Per VERSIONS.md §4.1, security tools should "always upgrade to latest stable." These are not blockers for M1–M4 (API + scaffolding) but must be addressed before M5 (Go Worker) and M6 (Native Tool Runners).

| Tool | Pinned | Latest observed | Target milestone |
|---|---|---|---|
| Nuclei | v3.7.1 | v3.8.0 | M6 |
| Subfinder | v2.6.7 | v2.13.0 | M6 (used in recon, M8) |
| httpx | v1.6.10 | v1.9.0 | M6 / M8 |
| Gitleaks | v8.21.2 | v8.30.1 | M6 |
| Trivy | v0.58.0 | v0.70.0 | M7 (persistent Docker service) |
| ZAP | 2.16.0 | v2.17.0 | M7 |
| sslyze | 6.1.0 | 6.3.1 | M6 |
| semgrep | 1.95.0 | 1.160.0 | M6 — large jump, review changelog |
| checkov | 3.2.340 | 3.2.524 | M6 — patch range |
| wapiti3 | 3.2.4 | 3.2.10 | M6 — patch range |

### verify-versions.sh script issues (resolved)

| Issue | Status |
|---|---|
| Python parser returned oldest 3.13 release instead of newest; no pre-release filter | **Fixed** in commit `9929bfe` (shieldscan-api) |
| PostgreSQL parser returned minor integer from oldest major (v6.3) instead of supported 18.x | **Fixed** in commit `9929bfe` (shieldscan-api) |
| Trivy GitHub API returned "unknown" on first run | Self-resolved on re-run (intermittent rate limit) |

---

## Upgrade session protocol

When addressing an entry:

1. Re-run `scripts/verify-versions.sh` to confirm drift still exists.
2. Read the changelog for the pinned→latest range.
3. Update `VERSIONS.md` with the new pin + date.
4. Update `pyproject.toml` / `go.mod` / `provision-worker.sh` per VERSIONS.md §5.
5. Run full test suite.
6. Remove the entry from this log (or move to "Closed" section with commit SHA).

---

## Technical decisions (non-drift)

### 2026-04-20 — `continuous_clean_days` uses ORM default only

`Organization.continuous_clean_days` has a SQLAlchemy `default=0` (applied by the ORM on INSERT) but **no** PostgreSQL `server_default`. All inserts go through SQLAlchemy today. If raw SQL inserts become a pattern (data migrations, admin scripts, direct DB writes from another service), revisit and add `server_default=text("0")` to avoid a `NOT NULL violation`.

### 2026-04-20 — JSONB over JSON for all PostgreSQL JSON columns

**Convention:** use `JSONB` (not `JSON`) for every PostgreSQL JSON column project-wide. **Reason:** indexable (GIN), binary storage, identical ORM usage, no downside on PG16+. **How to apply:** when adding any JSON-typed column, import `from sqlalchemy.dialects.postgresql import JSONB` and use `mapped_column(JSONB, ...)`. Already applied to `scans.config` and `attack_surface.tech_stack`; applies forward to all new JSON columns. The IMPLEMENTATION-PLAN snippets that say `JSON` (e.g., Task 1.4 §Scan `config`) should be read as JSONB.

### 2026-04-20 — pytest-asyncio 0.24 forces function-scoped test engines

`tests/conftest.py` recreates the SQLAlchemy async engine + schema per test. Reason: pytest-asyncio **0.24** supports `asyncio_default_fixture_loop_scope` but *not* `asyncio_default_test_loop_scope`, so a session-scoped engine's asyncpg connection gets "attached to a different loop" when touched by a function-scoped test. **How to apply:** when pytest-asyncio bumps to ≥0.25 (currently pinned `^0.24.0` in `pyproject.toml`), re-evaluate session-scoped engines + savepoint rollback — 13 tests in 0.62s is fine now, will matter at 500+ tests.

### 2026-04-21 — Sub-task 2.3.3: slug-collision retry policy

**Decision.** Organization slug collisions during register are handled by retry with an integer suffix: `acme` → `acme-2` → `acme-3` → `acme-4` → `acme-5`. Cap at **5 attempts** (`MAX_SLUG_ATTEMPTS`); on exhaustion the orchestrator raises `SlugGenerationFailed`, which the endpoint layer translates to a generic HTTP 400 (no information leak per M2 envelope item 6a).

**Mechanism:** each attempt runs inside `async with db.begin_nested()` (PostgreSQL SAVEPOINT). A `UNIQUE` violation on `organizations.slug` rolls back the savepoint without poisoning the surrounding transaction, so the next candidate gets a clean slate. Approved by user 2026-04-21.

**Failure semantics:** exhaustion is legitimately vanishingly rare in production (6 distinct orgs choosing names that slugify to the same base within one namespace). On the miss, the caller retries with a modified organization name. No need for randomized suffixes or a dedicated collision table.

**Commit:** `3260561` (shieldscan-api).

### 2026-04-21 — Sub-task 2.3.3: email verification uses Redis-backed tokens (plan §2.5 literal)

**Not a deviation.** IMPLEMENTATION-PLAN §2.5 specifies "Token stored in Redis with 24h TTL." Sub-task 2.3.3 pre-implements the token issuance half of that design (orchestrator mints + emails the token) so `register` can emit a real verification flow from day one. Task 2.5 later adds the consumption endpoint (`POST /auth/verify-email`).

**Key shape:** `email_verify:{sha256(token)}` → `user_id` with `EX 86400`. Chosen to sit alongside `ratelimit:*` for consistent ops `SCAN MATCH` patterns (RUNBOOK §11.6).

**Audit entry scope (DELIBERATE omission):** `auth.email.verification_sent` audit rows carry `details = {"method": "email"}` only — **no token, no hash, no prefix**. Rationale: audit retention is 7 years (CONSTITUTION §14.6); verification tokens live 24 hours. Storing any portion of the hash creates a permanent cross-reference to Redis keys that are already gone — pure noise for SIEM consumers, potential footgun for future forensics ("why is this hash prefix showing up in audit_logs?").

**Commit:** `3260561` (shieldscan-api).

### 2026-04-21 — Secure comparison of security-sensitive values (pattern)

**Pattern, not a one-off.** Any plan literal using `==` (or equivalent) to compare security-sensitive values — tokens, hashes, HMACs, API keys, session IDs, signing-key-derived values — must be replaced with `hmac.compare_digest` (Python stdlib). Applies retroactively to code review and forward to all future tasks.

**Rationale:** `==` on strings/bytes short-circuits on the first mismatching byte. Timing differences across requests let an attacker recover the secret byte-by-byte. `hmac.compare_digest` runs in time dependent only on the shorter operand's length, not its contents.

**Applies to:** `verify_api_key` (Task 2.2, applied), any future token validators (Task 2.3+ refresh-token rotation, email verification tokens, password reset tokens), HMAC webhook signatures (Stripe in M10, any inbound signed callback).

**Does NOT apply to:** comparing non-secret public values (email addresses, usernames, URLs, organization slugs). `==` is correct there — constant-time comparison of public strings costs performance without adding security.

### 2026-04-21 — Task 2.2: verify_api_key uses hmac.compare_digest (not ==)

**Minor deviation.** IMPLEMENTATION-PLAN.md §2.2 literal:

```python
def verify_api_key(plain: str, stored_hash: str) -> bool:
    return hashlib.sha256(plain.encode()).hexdigest() == stored_hash
```

**Actual implementation:** `hmac.compare_digest(candidate_hash, stored_hash)`.

**Reason:** `==` on the candidate hash vs stored hash is vulnerable to timing analysis on the API-key auth path. `compare_digest` is the correct stdlib primitive for comparing secrets. Not ADR-worthy — this is implementation-of-existing-requirement (secure comparison), not new architecture. Falls under the "secure comparison of security-sensitive values" pattern logged above.

**Applied:** `shieldscan-api/src/app/services/api_keys.py`.
**Approved:** user, 2026-04-21.
**Commit:** `ace008e` (shieldscan-api).

### 2026-04-21 — Task 2.1: direct bcrypt (replacing passlib)

**Plan deviation.** IMPLEMENTATION-PLAN.md §2.1 imports `passlib.context.CryptContext`. Actual implementation uses the `bcrypt` library directly.

**Reason:** passlib 1.7.4 (its last release, Oct 2020) probes `bcrypt.__about__.__version__` — an attribute removed in bcrypt 4.1. With the installed bcrypt 5.0.0 the probe fails silently and passlib raises a misleading `ValueError: password cannot be longer than 72 bytes` on any input. passlib is effectively unmaintained (5+ years no release).

**Options considered at Task 2.1:**
- **A.** Pin `bcrypt < 4.1` — rejected, pins to an EOL bcrypt line and creates future CVE exposure.
- **B.** Monkey-patch `bcrypt.__about__` — rejected, encodes a workaround in product code.
- **C.** Drop passlib, use `bcrypt` directly — **chosen.** 8 LoC, PyPA-maintained, future-proof. The spec-level requirement ("bcrypt cost=12 hashing") is satisfied identically.

**Applied:**
- `shieldscan-api/pyproject.toml`: `passlib` removed, `bcrypt = "^5.0"` added.
- `VERSIONS.md §2.3`: same swap (docs commit `d75ff51`).
- `SPECIFICATION.md §13`: ADR-010 added (docs commit `4e6099e`).
- `shieldscan-api/src/app/services/auth.py`: calls `bcrypt.hashpw` / `bcrypt.checkpw` directly.

**Consequence:** the bcrypt 72-byte password-length limit is now an API-layer concern — Task 2.2 register endpoint must reject passwords >72 bytes with HTTP 400. `hash_password`/`verify_password` do not enforce.

**Approved:** user, 2026-04-21.
**ADR:** 010 (SPECIFICATION.md §13).
**Commit:** `f6b9bbf` (shieldscan-api).

---

## Closed

### 2026-04-20 — M1.5 absorbs secret_verified column

M1.5 absorbs ADDENDUM-TOOLS-5 §8 `secret_verified` column (was scheduled for M6.5.0). Rationale: nullable column, zero cost, eliminates forward reference. Approved by user 2026-04-20. Partial index `idx_raw_findings_verified_secret ON raw_findings(scan_id) WHERE secret_verified = TRUE` ships in the same M1.5 migration. M6.5.0 task is reduced to a verification step.
