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

### 2026-04-21 — Task 2.5: email normalization (latent bug fix)

**Pre-empted by Task 2.5 design review.** Pydantic EmailStr lowercases the domain but leaves the local part case-sensitive (verified: `Alice@Example.COM` → `Alice@example.com` — `Alice` preserved). Without app-layer normalization, register + login were case-sensitive at the local part, enabling:

- Duplicate user creation: `Alice@x.com` and `alice@x.com` hit the `UNIQUE` index as different bytes and both registered successfully.
- Login lock-out: user registers as `Alice@x.com`, types `alice@x.com` at login, `User.email == req.email` returns 0 rows → 401.

**Three-layer fix (shieldscan-api `d4fdc96`):**

1. `normalize_email()` helper + Pydantic `@field_validator` on every `EmailStr` field at request boundary.
2. Defensive `.strip().lower()` in `register_identity` for non-HTTP callers (CLI, background jobs).
3. Alembic rev `b7f2a9c1e8d4`: `CHECK (email = lower(email))` on `users` — catches ORM bypass (raw SQL, SECURITY DEFINER).

Three regression tests cover each layer. No data migration (pre-launch, zero production users).

### 2026-04-21 — Test harness artifact: shared-session aborted-tx visibility

**Bookkeeping entry — not a deviation.** `tests/conftest.py`'s shared `db_session` pattern causes an `IntegrityError` in one client request to abort the outer SQLAlchemy transaction on the shared session. Post-failure `SELECT`s from the same session then see either zero rows OR require a compensating `rollback()` that also unwinds the prior successful request's data.

**Isolated to test observability, not behavior.** Production code paths use a fresh `AsyncSessionLocal()` per request, so aborted-tx state never crosses request boundaries.

**When a test needs to verify state AFTER an expected-failure request**, either:
- Rely on the HTTP response shape as proof (UserResponse body serializes committed ORM rows; if the body contains the expected email, the row exists), OR
- Open a fresh `AsyncSession` bound to the test engine (not the shared fixture session) for the verification query.

Encountered during Task 2.5 Commit 1a while building `test_register_rejects_mixed_case_duplicate`. Worked around by asserting the response-body shape instead of a post-register DB query.

### 2026-04-21 — Task 2.5: plan commit-message deviation (SendGrid → ConsoleEmailService)

IMPLEMENTATION-PLAN §2.5 says `feat(auth): add email verification via SendGrid`. Actual commit message: `feat(auth): add email verification consumption endpoints`.

**Why deviated:** M2 envelope item 5 explicitly scopes email to the `ConsoleEmailService` stub for M2; real SMTP (SendGrid or Resend) is deferred to M12.5 (see DRIFT-LOG 2026-04-21 M12.5-prerequisite entry from Task 2.4). Using the plan's literal message would misrepresent what shipped — no SendGrid code exists in the commit.

**Approved:** user, 2026-04-21.
**Commit:** `b3b7ec0` (shieldscan-api).

### 2026-04-21 — M1 schema tension: `audit_logs.actor_id ON DELETE SET NULL` conflicts with append-only trigger

Encountered during Task 2.5 test development. Deleting a `User` row cascades via `audit_logs.actor_id` (`ON DELETE SET NULL` per M1.5) — which issues an `UPDATE audit_logs SET actor_id = NULL`, which the append-only trigger from M1.6 rejects with `"audit_logs is append-only: UPDATE operations are forbidden"`.

**Net effect:** a User cannot be deleted as long as any `audit_logs` row references them as `actor_id`. Which is every auth event. Which is every User who has ever registered.

**Not a Task 2.5 issue — M1 design tension.** Out of scope for 2.5. Logged for future attention. Candidate resolutions when it becomes a real problem (user-deletion feature, GDPR data purge, admin offboarding):

- Option A: NULL-out `actor_id` via a bypass function (SECURITY DEFINER path that skips the trigger for `actor_id` column only). Clean.
- Option B: Drop the `ON DELETE SET NULL` and retain `actor_id` as a historical pointer even if the user is gone (no FK). Ugly data model.
- Option C: Soft-delete users (`deleted_at` column, no row removal). Standard practice for audit-heavy systems.

**Option C is likely the right answer** when we get there. Log in the GDPR-compliance milestone.

### 2026-04-21 — Task 2.4: intentional breaking change to `/auth/me` response shape

**Pre-launch breaking change, documented for the audit trail.**

Task 2.3 shipped `/v1/auth/me` returning a flat `UserResponse`:

```json
{"id": "...", "email": "...", "full_name": "...", "email_verified": true}
```

Task 2.4 reshapes it to `AuthIdentityResponse` — a discriminated-union on `kind` — to accommodate API-key-credentialed callers (per Task 2.4 endpoint policy: API keys are allowed on `/auth/me` for CI-verify use cases). The new shape:

```json
{
  "kind": "jwt" | "api_key",
  "organization": {"id": "...", "name": "...", "slug": "..."},
  "user":    { ...UserResponse... } | null,
  "api_key": { ...APIKeySummary... } | null
}
```

**Why it's the right call now:**
- Pre-launch — no external clients yet. Reshape cost: zero.
- Alternative A (flat UserResponse with synthetic user for API-key callers) creates the "optional-field soup" problem we avoided with scopes (deferred in 2.4).
- Alternative C (new `/auth/whoami` endpoint, leave `/auth/me` JWT-only) splits the client-side branch into "check credential kind first, then call the right endpoint." Worse UX; the discriminated-union shape collapses the branch to one endpoint + one `switch (body.kind)`.
- Alternative B (chosen) is a discriminated union at the schema level rather than optional-field soup — clients branch on `kind`, and the field absent at serialization is `null`, not missing.

**What changed in code:**
- `/v1/auth/me` dep swapped from `get_current_user` (JWT-only) to `get_auth_identity` (either kind).
- New response schemas: `AuthIdentityResponse`, `OrganizationSummary` in `src/app/schemas/api_keys.py`.
- Existing test `test_me_returns_user` renamed to `test_me_returns_identity_for_jwt` and updated for the new shape; new `test_me_returns_identity_for_api_key` added.

**Post-launch discipline:** once external clients exist, changes to `/auth/me` (or any public response shape) go through versioned endpoints (`/v2/auth/me`) or additive-only field additions. This Task 2.4 reshape used the pre-launch window deliberately; future reshapes won't have that option.

**Commits:** `shieldscan-api` `af4c1d9`.

### 2026-04-21 — RLS UUID-cast behavior is a security positive (not a deviation)

Discovery during Task 2.4 Commit 0 while writing the CREDENTIAL_INDEXED_TABLES regression guard: the `tenant_isolation` RLS policy uses `current_setting('app.current_org_id')::uuid` in its `USING` clause, which raises `invalid input syntax for type uuid: ""` when the GUC is unset (empty string) rather than silently returning zero rows.

This is a **stronger security mode** than silent-zero-rows:
- Silent zero rows: ambiguous — "no data exists" vs "not authorized" look the same to the caller.
- Loud UUID cast error: unambiguous — "you tried to query tenant data without tenant context" is distinguishable from "you queried data that doesn't exist."

Legitimate code paths always set the GUC (orchestrator, endpoints, tests via `use_org`); accidental bypass attempts produce loud `DBAPIError`s that fail noisily rather than silently succeed with empty results.

**Code at risk:** any caller that swallows `DBAPIError` and returns `[]` would mask a tenant-context bug. Reviewed during Task 2.4 Commit 0 — our error-handling paths all propagate `DBAPIError` upward.

**Not a deviation — a discovery.** Logging here so when someone asks in 12 months "what happens if `current_org_id` isn't set in production?" the answer is recorded.

### 2026-04-21 — Dummy-bcrypt timing test: security-test-suite candidate

**Decision at Task 2.3 close:** skip the timing-distribution test for `login_nonexistent_user` vs `login_wrong_password` in the unit-test suite. CI-flaky; the mechanism is already locked by `test_login_runs_verify_password_on_nonexistent_user` which asserts the code path is taken.

**Follow-up:** actual timing-distribution testing belongs in a separate **performance/security test suite** (tentatively `tests/security/` with `@pytest.mark.security` marker, runs in a dedicated CI job with a quiet machine). Candidate tests to land there when the suite is created:
- `test_login_timing_distribution_no_enumeration` — 100 logins of each kind (wrong-password, no-such-user), assert p95 times overlap within 50ms.
- Similar distribution tests for other constant-time-critical paths (`verify_api_key`, future HMAC webhook verification).

**Not a blocker.** Not a Task-level item. File for the ops-hardening milestone.

### 2026-04-21 — M12.5 prerequisite: email send failure retry + DLQ

**Deferred from Task 2.3 with explicit M12.5 tie.** `ConsoleEmailService` stub cannot fail; real SMTP (M12.5) can. First task in M12.5 MUST add retry + audit-correction-on-failure for the email send path in `identity.py` (`register_identity` post-commit email dispatch) and any future email-sending callsite.

**Why audit-correction matters:** Task 2.3 audits `EMAIL_VERIFICATION_SENT` inside the register transaction, before the email actually fires. A silent email-delivery failure post-commit would leave the audit log saying "sent" when nothing was sent. M12.5 must close this by either (a) moving the audit to post-send-success, or (b) adding an `EMAIL_DELIVERY_FAILED` compensating audit action and firing it on SMTP failure.

**Log this in OPERATIONS-RUNBOOK §11.6** next time that file is edited — no standalone docs commit per user direction 2026-04-21.

### 2026-04-21 — M1 latent shadowing bug: `src/app/db.py` unreachable

**Discovered during Sub-task 2.3.4 Commit 2** when `get_db` needed `AsyncSessionLocal`: `src/app/db.py` was shadowed by the `src/app/db/` package (empty `__init__.py` took precedence). The file had been dead since M1 — no imports, no test coverage, no runtime reach. All 107 M1-era tests passed because nothing depended on the orphaned symbols.

**Fix:** consolidated `engine` + `AsyncSessionLocal` into `src/app/db/__init__.py`; deleted `db.py`. No behavior change (nothing previously used the orphan). `shieldscan-api` commit `f0e9d68`.

**Lesson:** *"tests pass" does not imply "all code is reachable."* Type-checkers (mypy) *might* catch this for typed imports, but files that are never imported anywhere are invisible to every tool in the default toolchain.

**Follow-up shipped:** `tests/test_module_reachability.py` (in `shieldscan-api` commit `711dfbc`) — a parametrized test that imports every `src/app/**/*.py` by dotted name. Catches the "shadowed orphan" class of bug immediately on any future occurrence. 31 cases at commit time; auto-grows with the codebase.

### 2026-04-21 — Sub-task 2.3.4: /logout revokes refresh only (not access)

**Policy pin.** `/auth/logout` revokes the refresh token's `jti` but leaves the access token alive until its natural 15-minute expiry. Rationale: revoking both would require the client to supply the access jti in the logout body (it's in the Authorization header, but header values are not body payload), AND an access token surviving 15 minutes after logout is a bounded window that the refresh-token revocation has already closed the door on (no new access tokens can be minted). Clients needing immediate full invalidation use the (forthcoming) password-change endpoint.

Regression guard: `test_logout_does_not_revoke_access_token` in `shieldscan-api` commit `711dfbc`. If someone changes the policy, this test fails loudly and forces the decision to be explicit.

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
