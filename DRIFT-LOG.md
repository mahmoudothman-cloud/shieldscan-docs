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

### 2026-04-23 — Task 3.2: idempotent re-verify emits a fresh audit row

**Decision pin (no code change).** A successful `POST /v1/orgs/{org_id}/projects/{id}/verify` on a project where `domain_verified` was already `True` emits a fresh `PROJECT_DOMAIN_VERIFIED` audit row regardless. The DB-side state transition is a no-op (column was True, stays True), but the audit log gets a new entry on every attempt.

**Rationale:** audit logs track *events*, not *state*. A re-verify action IS an event — the user took an action, the system performed a check, the check succeeded. Suppressing the audit because the end state is identical would lose information:

- *When* did the customer re-verify? (Sequence of timestamps tells us.)
- *How often* are they re-verifying? (A sustained re-verify cadence often correlates with troubleshooting DNS issues elsewhere — operationally useful intel.)
- *Did they verify, lose access, then re-verify successfully?* (The audit trail makes this visible; deduplication would mask it.)

Consistent with the `PROJECT_DOMAIN_VERIFICATION_FAILED` discipline, which fires on every failure attempt regardless of prior state — symmetric treatment of attempts on both success and failure paths.

**For future engineers:** if you're tempted to add `if not was_already_verified: audit(...)` to the success branch — don't. Audit-row dedup adds complexity for questionable benefit (the only "noise" suppressed is intentional event records). The pattern as shipped is the desired behavior.

### 2026-04-30 — Task 4.1: Redis Streams over Pub/Sub for progress events (ADR-014)

**ADR-014 ships.** Plan literal §4.1 + §4.4 sketched a Pub/Sub-only design for `shieldscan:progress:{scan_id}`. SPEC §6.4 requires `Last-Event-ID` 60s replay — Pub/Sub cannot satisfy this on a multi-process uvicorn worker. ADR-014 captures the decision: Streams (`XADD` + `XREAD BLOCK` for live, `XRANGE` for replay) as a single primitive; multi-process reconnects work correctly because all workers read the same Stream.

**Hybrid (Streams + Pub/Sub) considered and rejected** for MVP scope: two writes per event, two consumer codepaths, two failure modes — justifiable at 100k+ concurrent connection scale (Discord, Slack pattern), not at MENA-SMB MVP scale (1–3 concurrent scans × <10 viewers each). `XREAD BLOCK` is Redis's Streams-equivalent of `SUBSCRIBE` and handles the fan-out trivially. Wrapper isolation makes a future hybrid swap mechanical if sustained-load profiling shows fan-out hot.

**Forcing function:** `test_subscriber_replay_returns_history` requires `XRANGE` semantics. Test docstring explicitly references ADR-014 so a future "simplification" reverting to Pub/Sub fails immediately with a self-documenting message.

**Commit:** `shieldscan-api` `349fc5e` (api), `shieldscan-docs` `60d71f8` (ADR + SPEC patch).

### 2026-04-30 — Task 4.1: SPEC §7.2 wording patched (Channel → Stream)

**Cosmetic + semantic.** SPEC §7.2 line 769 originally read "Channel: `shieldscan:progress:{scan_id}`" — Pub/Sub vocabulary inconsistent with the ADR-014 Streams decision. Patched to "Stream:" + a paragraph noting the producer/consumer primitives (`XADD ... MAXLEN ~ 1000` produce, `XRANGE`/`XREAD BLOCK` consume) and a final paragraph noting that cancel + completions remain Pub/Sub.

Same docs commit as ADR-014 — patching the spec without the ADR would be confusing in isolation.

### 2026-04-30 — Task 4.1: Mixed-primitive use (Streams for progress, Pub/Sub for cancel + completions)

**Pin.** Streams used ONLY for `shieldscan:progress:{scan_id}` because §6.4 requires replay. Pub/Sub stays for:
- `shieldscan:cancel:{scan_id}` — one-shot live-only signal. A worker not subscribed when cancel emits cannot usefully consume a stale cancel.
- `shieldscan:completions` — one-shot completion broadcast. The orchestrator's completions consumer either receives live or rebuilds state from the DB row at startup. Replay adds no value.

This is **intentional, not transitional**. A future "let's unify on Streams everywhere" refactor should NOT happen without a fresh ADR — the asymmetry is the right answer for these specific channels.

### 2026-04-30 — Task 4.1: Cancel-vs-completion race — completion wins

**Pin (no code in 4.1).** If a job finishes at the same instant a cancel arrives, both `job_completed` (to `shieldscan:completions`) and the cancel signal (to `shieldscan:cancel:{scan_id}`) may publish. **Resolution at the worker side (M5 task 5.5):** if the job is already in a terminal state (completed/failed) when the cancel is consumed, cancel becomes a no-op. The DB row reflects whichever terminal state actually landed first. Documented here so 4.5 (cancel endpoint) doesn't accidentally try to override completion server-side.

### 2026-04-30 — Task 4.1: Completions subscriber explicitly punted to 4.2

**Pin.** SPEC §7.3 defines `shieldscan:completions` as a Pub/Sub channel that publishes job-completion events back to the orchestrator. M4 needs a consumer that subscribes and updates `ScanJob.status`/`finding_count`/`duration_ms` rows. **Not a 4.1 primitive** — it's orchestrator-internal coordination, surfaces in Task 4.2's scope proposal as part of the ScanOrchestrator design (likely a background task lifecycle the orchestrator owns, not a standalone wrapper class).

### 2026-04-30 — Task 4.1: Idempotency-key handling deferred to Go worker (M5.5), not Python

**Pin.** SPEC §7.5 specifies `idempotency_key` format `{scan_id}:{tool}:{unix_ts}` with 24h Redis TTL. Python orchestrator (4.2) GENERATES the key inside the dispatched job payload but does NOT touch Redis idempotency state. The SETNX-with-TTL claim happens worker-side at job-pickup time (M5 task 5.5). `test_dispatch_does_not_dedupe_by_idempotency_key` is a NEGATIVE pin — it asserts that duplicate `dispatch()` pushes duplicate items, so a future engineer adding Python-side dedupe breaks this test loudly.

ADP-5 from M4 landscape pass: option 2 (worker-side claim) confirmed.

### 2026-04-30 — Task 4.1: Stream-key cleanup TTL — OPS milestone carry-forward

**Carry-forward.** `XADD ... MAXLEN ~ 1000` bounds each stream's *length*, but the **stream keys themselves** persist after scan completion. At MVP scale this is fine (handful of concurrent scans, tiny memory footprint). At enterprise scale, accumulating thousands of inactive stream keys becomes housekeeping overhead.

**OPS-milestone scope:** janitor that purges `shieldscan:progress:*` keys 24–48h after `Scan.completed_at`. Possibly fold into a single Redis-cleanup job that also handles `shieldscan:cancel:*` channel-key residue. Tracking cost: ~half-day in OPS milestone. Don't try to add cleanup logic in 4.1 — wrong domain.

### 2026-04-30 — Task 4.1: Scan list/detail SPEC §6.2 endpoints deferred from M4 to M10

**Pin.** SPEC §6.2 lists `GET /scans`, `GET /scans/:id`, `GET /jobs[/:jid]` under the Scans domain (7 endpoints). M4 ships 4 of those (POST create, DELETE cancel, GET progress SSE, POST compare). The other 3 are read-side endpoints.

**Decision (M4 boundary):** ship them in M10 (Vulnerability & Report APIs) — the read-side cluster — rather than tacking them onto M4 as a 4.X gap-closer. Pattern variant of the SPEC-gap-closure pattern: 2.Y / 3.X closed gaps where the work fit the milestone's domain (small composition); scan read-side is read-side and benefits from consistent pattern design across the read-side cluster (scans + findings + reports). Building one per write-side milestone risks read-side pattern inconsistency.

The other M4 endpoints are write-side scan operations + the SSE stream; M4 closes after Task 4.6 (compare).

### 2026-04-30 — Task 3.X: AuthType enum lifted from M1 inline comment

**Refactor.** M1's `ProjectCredential.auth_type` carried only an inline comment (`# cookie | bearer | basic | form | custom_header`) describing legal values. Task 3.X needs the value set programmatically (Pydantic discriminated union, route-handler dispatch), so the comment is lifted into a proper `AuthType(str, PyEnum)` in `app/services/credentials.py`.

**Five canonical values** (frozen): `basic`, `bearer`, `cookie`, `form`, `custom_header`. Pinned by `CANONICAL_AUTH_TYPES` constant in the service module + the schema's `Literal[...]` discriminators. Adding a new value requires updating both surfaces and the test that asserts the canonical set.

**Disjointness from action enums:** `AuthType` is in a different namespace from `AuthAction`/`ProjectAction` — not registered in `ALL_ACTION_ENUMS`. Action enums encode audit events; `AuthType` encodes credential payload shape. Kept separate to avoid a cross-domain registry that would invite future confusion.

**Commit:** `shieldscan-api` `c7a12e9`.

### 2026-04-30 — Task 3.X: PATCH + DELETE for credentials (vs single PATCH with `none` sentinel)

**Decision.** Two endpoints — `PATCH /credentials` (set/replace) + `DELETE /credentials` (clear) — instead of a single PATCH with `auth_type: "none"` clearing semantics.

**Rationale:** Verb discipline matches the rest of the codebase (DELETE on `/projects`, DELETE on `/api-keys`); cleaner audit boundary (`PROJECT_CREDENTIAL_SET` vs `PROJECT_CREDENTIAL_DELETED` cannot collide); no awkward `"none"` sentinel that would require a sixth schema variant carrying nothing. Trade-off accepted: one extra route, ~30 LoC and one extra test (`test_delete_credentials_204_and_audit`).

**DELETE idempotency:** calling DELETE on a project with no credential is a no-op 204 with no audit row. Audit rows fire only for events that actually happened.

**Commit:** `shieldscan-api` `c7a12e9`.

### 2026-04-30 — Task 3.X: form-auth credential exposes five fields; `additional_fields` deferred

**Decision.** The `form` credential variant accepts exactly: `login_url`, `username_field`, `password_field`, `username`, `password`. No `additional_fields` map for arbitrary form inputs.

**Known limitation.** Real-world login forms sometimes carry hidden CSRF tokens, captcha challenges, or arbitrary submitted-state fields. The MVP form-credential ships without that flexibility — a customer whose login form needs an extra hidden field cannot configure it through this endpoint today.

**Why ship the limited shape now:** the five-field set covers the MENA SMB common case (per market research feeding milestone planning). Adding `additional_fields` later is an additive Pydantic change (no schema-level breaking change for existing clients). Until a paying customer hits this limit we defer the complexity.

**Forcing function:** `test_patch_form_happy_path` asserts the exact field set; adding a sixth field will require updating that test, surfacing the contract change to review.

**Commit:** `shieldscan-api` `c7a12e9`.

### 2026-04-30 — Task 3.X: PATCH/DELETE on archived project → 409 Conflict

**Decision.** Both PATCH and DELETE on `/credentials` return 409 `project_archived` when the parent project is archived (`archived_at IS NOT NULL`). Symmetric on the two endpoints.

**Why 409, not 410:** 410 already means *"this resource is in the deleted state"* (used by `DELETE /projects` on already-archived). 409 = *"the operation conflicts with the resource's current state"* — credentials are not themselves deleted/archived; the project's state simply forbids credential mutation. Distinct semantics, distinct status codes.

**Why a hard error, not silent-allow:** archived projects shouldn't accept new state. Allowing credential mutation on archived projects creates a silent-data-leak path (re-activate an archived project with stale credentials the customer thought were cleared). Hard 409 forces the customer to un-archive first.

**Pinned by** `test_patch_archived_project_returns_409` + `test_delete_archived_project_returns_409`.

**Commit:** `shieldscan-api` `c7a12e9`.

### 2026-04-30 — Task 3.X: single-key Fernet for MVP; multi-key `MultiFernet` rotation deferred to OPS milestone

**Carry-forward.** SPEC §10.1 promises *"key rotation support for encryption keys."* Task 3.X ships single-key Fernet only (`Fernet(settings.FERNET_KEY)`), no `MultiFernet` rotation primitive.

**Why MVP is sufficient:** zero production credentials exist pre-launch. Key rotation is a pre-launch checklist item: generate fresh key, re-encrypt seed credentials (none yet), update settings.

**OPS-milestone scope:** `MultiFernet` with primary + retiring keys, a re-encryption job that walks `project_credentials` rolling rows from old → new key, ops runbook entry for the rotation procedure. Tracking cost: ~1 day in OPS milestone.

**No SPEC update needed** — §10.1 isn't wrong, just early. The promise stands; this entry pins when it's delivered.

**Commit:** `shieldscan-api` `c7a12e9`.

### 2026-04-30 — Task 3.X: Vault-stored secrets deferred from SPEC §10.1 to OPS milestone

**Carry-forward.** SPEC §10.1 says *"All secrets in Vault (not env vars in production)."* Task 3.X reads `FERNET_KEY` from `settings` (which loads from env) — Vault integration is OPS-milestone work.

**Pre-launch posture:** env-var secrets are acceptable when (a) the env file is `chmod 600` on the host, (b) no shared infra, (c) operational discipline of not committing the file. All true for MVP.

**Vault scope when it lands:** `app/config.py` swap from env to a Vault-backed loader, secret-rotation hooks tied to deployment, audit trail for secret reads. Substantial enough to be its own milestone task, not a 3.X gap-closer.

### 2026-04-30 — Task 3.X: FERNET_KEY conftest fixture corrected

**Bookkeeping.** The previous fixture in `tests/conftest.py` used the literal string `"test-fernet-key-base64-urlsafe-44-bytes="`, which is NOT a valid Fernet key (not 32 bytes of urlsafe-base64). The malformed key never crashed because no test instantiated Fernet — Task 3.X is the first code path to do so.

**Fix:** replaced with `"c2hpZWxkc2Nhbi10ZXN0LWZlcm5ldC1rZXktMzJieSE="` — valid, deterministic, base64 encoding of `b"shieldscan-test-fernet-key-32by!"`. Determinism matters because Test 14 asserts byte-level plaintext absence in the encrypted blob.

**Forcing function:** `test_fernet_key_in_settings_is_valid_at_import` instantiates Fernet at import time. Future broken keys fail at test collection, not mid-run.

**Commit:** `shieldscan-api` `c7a12e9`.

### 2026-04-30 — Task 3.X: `project_credentials.project_id` UNIQUE constraint added (M1 schema correction)

**Schema correction.** M1 schema declared `project_credentials` with a non-unique `project_id` index. Task 3.X's PATCH semantics require exactly one credential per project — without UNIQUE, the application could (under a SELECT-then-INSERT race) end up with two credential rows for one project.

**Resolution.** Migration `d4f6b1e9a527` adds `UNIQUE (project_id)` constraint. Application-layer enforcement uses PostgreSQL UPSERT (`INSERT ... ON CONFLICT (project_id) DO UPDATE`) for atomicity, with the new constraint as the conflict target.

**Downgrade safety:** dropping the constraint is harmless. Upgrading on a dev DB with duplicate `project_id` rows fails loudly (unique-violation) — operator dedupes manually. No data migration needed pre-launch.

**Commit:** `shieldscan-api` `c7a12e9`.

### 2026-04-26 — Task 3.3: `.zip` dropped from mobile-upload allowed extensions

**Plan deviation.** IMPLEMENTATION-PLAN.md §3.3 lists the allow-list as `{".apk", ".ipa", ".zip"}`. Shipped as `{".apk", ".ipa"}`.

**Three reasons:**

1. **Adversarial input surface.** A `.zip` upload would force content-sniffing (or trust the filename) to decide platform. Either path is fragile — filenames lie, and content-sniffing on attacker-controlled input is a known footgun pattern.
2. **No clear MVP use case.** APKs and IPAs are themselves ZIP archives; a separate `.zip` extension only matters if customers send arbitrary bundles, which isn't the mobile-scan flow.
3. **Virus scanning deferred.** Accepting unscanned `.zip` while malware analysis is still scan-time-only (entry below) compounds risk for no MVP benefit.

**Reversibility.** Adding `.zip` back is a one-line change to `ALLOWED_EXTENSIONS` plus a platform-detection rule. Do it if a real workflow demands it; the test pinning `.zip → 400 invalid_extension` (`test_upload_zip_extension_rejected`) is the forcing function for that follow-up.

**Commit:** `shieldscan-api` `d9b0fbf`.

### 2026-04-26 — Task 3.3: error envelope is `{"detail": <code>}`, not `{"error": {"message": ...}}`

**Plan deviation, codebase-consistent.** IMPLEMENTATION-PLAN.md §3.3 test snippets assert `r.json()["error"]["message"]`. Shipped as `{"detail": <code>}` per the codebase-wide `ErrorResponse` standard established in Task 2.3.4.

The plan literal predates 2.3.4's standardization. Reading the plan literally would diverge every Task 3.3 error path from every other endpoint in the codebase — strictly worse than ignoring the literal.

**Pattern for future tasks:** plan literals whose error-shape predates 2.3.4 use `{"detail": ...}` without ceremony. No DRIFT-LOG entry needed per occurrence; this entry establishes the pattern.

**Commit:** `shieldscan-api` `d9b0fbf`.

### 2026-04-26 — Task 3.3: virus / malware scanning deferred to scan-time

**Decision.** The mobile-upload endpoint accepts `.apk` / `.ipa` files as content-addressable blobs. ZIP magic-bytes is the only content check. There is **no virus scanning at upload time**.

Static + dynamic analysis (including malware indicators surfaced by MobSF) happens later in the mobile-scan worker (M5+). The upload endpoint is intentionally **not a security gate** — its job is to land the artifact safely in R2 and trigger downstream scanning.

**Why this is OK at MVP:** R2 storage is private (no public read URLs). Files don't execute on our infrastructure between upload and scan. Customers can only retrieve their own org's uploads. The risk surface is "customer uploads malware their own scanner will flag" — exactly what the product is designed to do.

**Revisit when:** we add public download URLs, when uploads become a customer-facing distribution mechanism, or when free-tier abuse (e.g. uploading malware as a free file-host bypass) shows up.

**Commit:** `shieldscan-api` `d9b0fbf`.

### 2026-04-26 — Task 3.3: `MobileUpload.uploaded_by` changed from NOT NULL to nullable

**Schema correction.** M1 declared `mobile_uploads.uploaded_by` as `NOT NULL` referencing `users.id`. Task 3.3 enables API-key authentication on the upload endpoint per decision #5. API keys belong to an org, not a user — so the `NOT NULL` constraint was unsatisfiable on the API-key path.

**Resolution.** Migration `c8e3d2a4f9b1` drops `NOT NULL`. JWT upload writes `uploaded_by = identity.user.id`; API-key upload writes `uploaded_by = NULL` and surfaces the actor identity via the audit row's `details.api_key_prefix` instead.

**This is correction, not invention.** The audit-logs row already follows this exact pattern: `audit_logs.actor_id` has been nullable for API-key actions since M2 (Task 2.3.2). `mobile_uploads.uploaded_by` deviated from that established convention; this entry brings it back into line.

**API contract:** responses report `uploaded_by_user_id: UUID | None`. UI renders "uploaded via API key" when null. Pinned by `test_upload_via_api_key_writes_null_uploaded_by`.

**Downgrade safety:** the migration's downgrade restores `NOT NULL`. If any rows have `uploaded_by IS NULL` at downgrade time (i.e. an API-key upload landed under this revision), the ALTER fails loudly with `column "uploaded_by" contains null values` — operator must delete or backfill those rows before retrying. Silent data loss is the worse failure mode.

**Commit:** `shieldscan-api` `d9b0fbf`.

### 2026-04-26 — SPECIFICATION.md §6.2 header "Projects (7)" actually enumerates 8 endpoints

**Cosmetic typo, bookkeeping.** The §6.2 header reads "Projects (7)" but the body lists 8 endpoints (CRUD 5 + verify + the future 2 credential endpoints). Surfaced during Task 3.2 review; flagged here to bundle into a future docs cleanup pass rather than touch SPECIFICATION.md inside an unrelated commit.

**Not a 3.3 issue** — included in the 3.3 batch only because the entry batch is the natural place to track docs-bookkeeping items as they're spotted.

### 2026-04-26 — Task 3.3: new dep `python-multipart` 0.0.20

**New runtime dependency.** Added `python-multipart = "^0.0.20"` to `pyproject.toml` for FastAPI's `UploadFile` support — required mechanically by the file-upload endpoint, not a discretionary library choice.

**VERSIONS.md trace:** not yet listed in VERSIONS.md (it's a transitive-style dep that FastAPI doesn't pin itself but requires for multipart/form-data parsing). Add to VERSIONS.md in the next docs cleanup pass; pin range `^0.0.20` is conservative (0.0.x semver discipline).

**Commit:** `shieldscan-api` `d9b0fbf`.

### 2026-04-23 — Task 3.2: per-domain audit-action enum split

**Refactor.** The single `AuthAction` enum was accreting non-auth values (`EMAIL_VERIFICATION_SENT`, `PASSWORD_RESET_*`, etc.) — adding M3's project events (`PROJECT_CREATED`, `PROJECT_DOMAIN_VERIFIED`, …) would sprawl it further. Split now at the natural M3 boundary (the longer the split is deferred, the more painful it becomes).

**New shape:**
- `AuthAction` (unchanged — preserves every existing audit-row value).
- `ProjectAction` (new) for project-domain events.
- `audit()` widened to accept `AuthAction | ProjectAction`.
- `ALL_ACTION_ENUMS` registry tuple — single source of truth. Future enums (`ScanAction`, `FindingAction`, etc.) extend by appending.

**Load-bearing invariant:** string values must be globally unique across all action enums. Pinned by `test_action_enum_values_are_globally_disjoint` — verified to fail when given a duplicate value (manual sanity check during commit prep). Plus `test_action_enum_registry_lists_all_known_enums` catches a future engineer adding a new enum but forgetting to register it.

**Back-fill in same commit:** Task 3.1's create/patch/delete handlers shipped without audit (small gap). Closed here so the project-event audit trail is complete from M3 day one.

**Commit:** `shieldscan-api` `b84b7a3`.

### 2026-04-23 — Task 3.2: domain verification ships both DNS TXT + meta tag (plan literal)

**Decision.** `POST /v1/orgs/{org_id}/projects/{id}/verify` supports two methods: `dns_txt` and `meta_tag`. Plan literal IMPLEMENTATION-PLAN.md §3.2 specified both; my pre-task lean was DNS-only; user reversed that lean after weighing three arguments (MENA SMB shared-hosting reality, mitigable spoofing concern, plan-author judgment).

**Two security tightenings layered on top of the plan's literal code:**

1. **`follow_redirects=False`** on the meta-tag fetch. Plan snippet didn't specify either way; without this, an attacker who controls a redirect handler at the customer's `target_url` could redirect us to an attacker page that carries the right meta tag — verification would succeed for a domain the customer doesn't actually control. Pinned by `test_meta_tag_redirect_does_not_follow`.

2. **`html.parser` (stdlib) instead of regex** for meta-tag detection. Plan snippet used a regex (`re.search(r'<meta[^>]+name=...')`) that fails on common HTML variations (single-quoted attrs, attribute-order changes, self-closing tags). Structured parsing handles all four pinned-by-test variants without false negatives. No new dependency — the parser is in the stdlib.

**Failure-mode UX:** the 400 body includes `expected_record` showing the customer the EXACT TXT record / meta tag they need to add. Substantial UX win at zero security cost (the verification_token is already in `ProjectResponse` per Task 3.1 §H decision #2 — it's an org-internal value, not a credential).

**Discrete `failure_reason` codes** for ops differentiation: `dns_nxdomain`, `dns_no_answer`, `dns_timeout`, `dns_token_not_found`, `dns_error`, `meta_tag_not_found`, `meta_tag_token_mismatch`, `http_status_<n>`, `http_timeout`, `http_connect_error`, `html_parse_error`. Customer-facing — leaks DNS state about the customer's OWN domain, which is fine.

**Re-verify failure semantics:** a re-verify whose record has been removed returns 400 but does NOT downgrade `domain_verified` from True to False. DNS is occasionally flaky; one failure must not invalidate historical state. Per Task 3.2 user refinement on decision #3.

**No new ADR** — composition on plan literal with documented tightenings.

**Commit:** `shieldscan-api` `5f74cc5`. Cross-ref `dnspython ^2.7` pinned in VERSIONS.md.

### 2026-04-23 — Task 3.1: PATCH target_url auto-resets domain verification when root_domain changes

**Decision.** `PATCH /v1/orgs/{org_id}/projects/{id}` with a `target_url` field that produces a different `root_domain` automatically:
- Updates `project.root_domain` to the new derived value
- Sets `project.domain_verified = False`
- Mints a fresh `project.verification_token = uuid4().hex`

Path/port-only changes within the same `root_domain` do NOT reset verification — those are legitimate same-domain target updates.

**Rationale:** verifying a domain is a security claim ("this org controls example.com"). Allowing a verified `target_url` to silently move to `different.org` while keeping `domain_verified=True` would let any project member silently re-aim a verified target at a domain they don't actually control. Auto-resetting forces a re-verification round at Task 3.2's `/verify` endpoint.

**Pinned by tests:**
- `test_patch_target_url_same_domain_keeps_verification` (no reset on same-domain change)
- `test_patch_target_url_different_domain_resets_verification` (reset + token rotation on cross-domain)

**Commits:** `shieldscan-api` `fdcb0db`.

### 2026-04-23 — Task 3.1: GET on archived project returns 200, not 404

**Decision.** `GET /v1/orgs/{org_id}/projects/{id}` for an archived project returns **200 with `archived_at` populated** in the response body, not 404.

**Rationale:** soft-delete is a reversible state, not absent state. Archived projects retain scan history that users may legitimately reference. List endpoints filter archived out by default (`?archived=true` includes them), but single-project GET always returns the row regardless of archive status.

**This is also part of the broader pattern** ADR-012 establishes for app-layer scoping: cross-tenant lookups return 404 (existence leak avoidance), but in-tenant lookups return 200 even when the row is "soft-deleted." 410 Gone is reserved for the **DELETE→DELETE** double-action signal.

**Pinned by `test_get_archived_project_returns_200`.**

### 2026-04-23 — Task 3.1: tldextract PSL vendoring carry-forward to M12.5

`tldextract` (added in Task 3.1 for `extract_root_domain`) consults the Mozilla Public Suffix List. Default behavior fetches+caches the PSL on first use. For air-gapped on-prem deployments (M12.5+) we'll need to **ship a vendored snapshot** of the PSL alongside the application — the runtime fetch is not available in restricted-network environments.

**Action carried forward to M12.5:** as part of the on-prem deployment build, vendor a current PSL snapshot into the app distribution and configure `tldextract.TLDExtract(suffix_list_urls=(), cache_dir=<bundled>)` to use it. PSL refresh policy: re-vendor on each release; PSL changes glacially so manual refresh is fine.

Not blocking M3 / SaaS deployment (default PSL fetch works there).

### 2026-04-23 — Task 3.1: PATCH project — `extra="forbid"` blocks immutable fields at schema level

**Pattern note (not a deviation).** `ProjectUpdateRequest` uses `model_config = ConfigDict(extra="forbid")` so that any client sending `organization_id`, `root_domain`, `domain_verified`, `verification_token`, `id`, or `archived_at` in the PATCH body gets a 422 — these fields aren't on the schema at all.

**Why this matters as a pattern for M3+:** every PATCH endpoint shipped after this should use the same `extra="forbid"` posture. Without it, Pydantic v2's default (`extra="ignore"`) silently drops unknown fields, which means a client trying to write to an immutable column gets a 200 success response with no field change — confusing, and a security footgun (think: a client trying to PATCH `is_admin` on a member resource and getting "success" back).

**Convention going forward:** every UpdateRequest schema in M3+ MUST set `extra="forbid"`. Pinned in 3.1 by `test_patch_project_immutable_fields_rejected`.

### 2026-04-23 — Task 3.1: API-key URL spec misalignment carry-forward

**Carry-forward from M2.** `SPECIFICATION.md` §6.2 (line 466–467) lists API-key endpoints under `/orgs/:org_id/api-keys`. Task 2.4 shipped them at `/v1/auth/api-keys` (no `/orgs/:org_id/` prefix). The shipped paths are reasonable for an account-scoped resource where the org is implicit from the JWT, but they diverge from the spec literal.

**Resolution deferred** to a future docs-only commit (no code change needed; both options remain on the table). Two paths:
- Option A: update `SPECIFICATION.md` §6.2 to match the shipped `/v1/auth/api-keys` paths (codifies what shipped).
- Option B: add route aliases under `/v1/orgs/{org_id}/api-keys` that delegate to the existing handlers (spec-literal compliance with backward compat).

Either is fine. Logged here so it doesn't get forgotten between M3 and the M12.5 / pre-launch documentation pass.

### 2026-04-23 — Task 2.Y closes SPECIFICATION/IMPLEMENTATION-PLAN gap

**Plan/spec misalignment, resolved.** SPECIFICATION.md §6 (lines 447–448) enumerated `/auth/forgot-password` + `/auth/reset-password` as part of the auth API surface alongside register/login/refresh/logout/verify-email. IMPLEMENTATION-PLAN.md never numbered these as a task — its M2 section stops at Task 2.5 (email verification flow).

**Per the CLAUDE.md document hierarchy** (SPECIFICATION wins over IMPLEMENTATION-PLAN on product truth), this is a contract gap to close, not scope creep to defer. M2 ships these endpoints as Task 2.Y to leave the spec and the implementation in agreement.

**Composition cost was small** because Task 2.X had just shipped the user-level revocation primitive (`set_user_revoked_before`) — reset is functionally a forced password change and reuses every existing pattern (token-issuance from 2.3.3, GETDEL atomic consumption from 2.5, no-enumeration policy from resend-verification, audit reserved enums from 2.3.2, rate-limit dual scope from login).

**Commit:** `shieldscan-api` `74cc9ee`.

### 2026-04-23 — Task 2.Y: reset token TTL = 1 hour

**Decision.** Password-reset tokens expire in 1 hour, set as `TOKEN_TTL_SECONDS = 3600` in `src/app/services/password_reset.py`.

**Reasoning:**
- Reset is the highest-value short-lived credential in the system — full account takeover on consumption.
- 24-hour TTL (the email-verification choice) was right for verification (verifying your own already-accessible email is a low-value target). It is wrong for reset.
- **Email-prefetch attack surface is real.** Outlook/Office365 and several spam scanners fetch URLs on email receipt; mail-relay logs may capture link contents. A 24h TTL means a compromised relay sees a usable token for 24h. 1h cuts that window 24×.
- **Industry norms cluster at 1h.** GitHub, Stripe, Auth0 default to 1 hour. Most banks shorten further (15–30 min); we don't go shorter because legitimate users may take longer to read mail on a slow client.
- Users who exceed 1h re-request. The cost of a fresh request is one more email; the cost of a 24h-window token in a compromised relay is account takeover.

### 2026-04-23 — Task 2.Y: new password may equal current on reset (NOT on change)

**Asymmetric policy, deliberate.** The two password-write endpoints differ on the "new equals current" check:

- `POST /auth/password/change`: REJECTS new == current with 400 `invalid_current_password` (merged with wrong-current per Task 2.X scope decision #2).
- `POST /auth/reset-password`: ALLOWS new == current. No check.

**Rationale:** the two endpoints prove different things.
- **Change** is a knowledge-of-current-password flow. The user demonstrably remembers their current password (they typed it in `current_password`). Forcing them to choose a different new password is reasonable: they know what they're avoiding.
- **Reset** is a knowledge-of-mailbox flow. By definition, the user does NOT remember their current password — that's why they're resetting. Asking them to type something different from a hash they can't produce would be theatre. Internally we'd have to call `verify_password(new, user.hashed_password)`; if the user happens to type their current password (which they don't remember anyway), we'd reject — pointlessly.

**Industry alignment:** GitHub, Google, AWS Cognito do not block "new password matches current" on reset. They DO on change. Same posture.

**Audit trail unaffected:** `auth.password.reset.completed` fires regardless. If the user resets to the same string, it's a noop in security terms but a deliberate event in the audit.

### 2026-04-23 — Task 2.Y: password reset does NOT auto-verify email

**Pinned by `tests/routes/test_password_reset.py::test_18_reset_does_not_change_email_verified_status` with explicit failure-message comment.**

Reset proves mailbox ownership (same mechanism as email verification). The two could be coupled — "if you can reset, you've proven mailbox; therefore mark email as verified." We deliberately do NOT.

**Reasons:**
- Verification is an affirmative, semantic action separate from reset. A user clicking "I confirm this is my email" is different from a user clicking "let me reset because I'm locked out."
- Auto-verifying on reset silently changes account state. Silent state changes are surprising — and the test docstring explicitly says: changing this is a policy decision, not a refactor.
- **Future email-change flows would be complicated.** If the user changes their primary email, reset tokens for the OLD email could end up auto-verifying the NEW one without consent. Cleaner to keep verification an explicit user choice.
- The cost of NOT auto-verifying is one extra click ("now click the verify link"). Tiny.

**If anyone reverses this in the future:** add an ADR documenting the reasoning, OR a DRIFT-LOG entry with the new design + migration plan for any users affected. Don't quietly swap the behavior.

### 2026-04-23 — Task 2.X: reuse-detection auto-trigger of user-level revocation deliberately deferred

**Decision at Task 2.X close:** the `user_revoked_before:{user_id}` mechanism shipped, but the `/auth/refresh` reuse-detection path does NOT auto-trigger it. Reuse-detection remains **targeted-jti-only** — only the reused token is revoked; the user's other outstanding tokens stay valid.

**Considerations against auto-trigger:**
- **False positives from network blips:** a client retrying a request after a transient timeout might legitimately replay the same refresh token. Auto-revoking all the user's sessions on every blip-induced retry would be hostile UX.
- **Observability loss:** once we auto-blow-away all sessions, we lose the per-jti reuse signal that tells us *which* token was replayed. The audit row stays, but the pattern of "this user has had 3 reuse events in 10 minutes" is the kind of intel that would get muddied.
- **No undo path:** `set_user_revoked_before` is one-shot. If we trigger on a false positive, the user has to log in again on every device. We can't "un-set" the revocation timestamp meaningfully.
- **Revocation storms:** a buggy client library that mishandles refresh-rotation could create cascading revocation events across all of its users — self-amplifying, hard to diagnose.

**Future ADR required** to revisit. Considerations for that ADR: per-source rate-limiting (only auto-trigger on >N reuses within M minutes), a "soft" revocation that still allows the current valid token to keep working (would need a different mechanism), or sticking with targeted-jti for the foreseeable future.

**Approved:** user, 2026-04-23.
**Commits:** `shieldscan-api` `07d0189`, `1bb0b66`.

### 2026-04-23 — Task 2.X: no caching layer for `user_revoked_before` reads

**Decision at Task 2.X close:** the auth path's `get_user_revoked_before` Redis GET runs uncached on every authenticated request. Approved short-circuit (Option C) — no further optimization for M2.

**Performance characterization:** Redis GET on a nonexistent key is sub-millisecond and O(1). The 99.9% no-revocation path costs roughly the same as the existing jti-revocation `EXISTS` check we already accepted. The set-revocation path runs once per password-change event (rare).

**If profiling ever shows latency matters,** consider in this order:
1. **Per-request memoization** — if multiple deps in the same request resolve the auth path, cache the result on the request object. Cheapest, no invalidation surface.
2. **Pipeline jti+user GETs** — batch `is_revoked` and `get_user_revoked_before` into a single Redis round-trip. Saves the second TCP round-trip; no invalidation issues.
3. **DO NOT** add a TTL'd in-memory cache behind the DB / pre-Redis layer. Cache invalidation on password change would double the invalidation surface for zero gain — the password-change handler would have to invalidate every API pod's local cache, which is exactly the problem we just solved by going to Redis in the first place.

**Approved:** user, 2026-04-23.

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
