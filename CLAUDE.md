# CLAUDE.md

**For Claude Code: This file is your operating manual.** Read it at the start of every session. It is shorter than CONSTITUTION.md on purpose — it tells you what to do, not why. For the "why" and the full governance rules, see CONSTITUTION.md.

---

## Session Startup Checklist

Every time you start work on this project:

```
[ ] 1. Read this file (CLAUDE.md) — you're doing it now
[ ] 2. Read VERSIONS.md §3 — run scripts/verify-versions.sh before any installs
[ ] 3. Check IMPLEMENTATION-PLAN.md for the current milestone
[ ] 4. Check git log to see where the previous session left off
[ ] 5. Confirm which repo you're in: shieldscan-api or shieldscan-engine
[ ] 6. Read the next task's "Files" + "Step 1" before writing anything
```

Never skip these steps. They prevent 90% of mistakes.

---

## The Document Hierarchy

When information conflicts, higher wins:

```
1. CONSTITUTION.md       ← Absolute authority. Never violate.
2. VERSIONS.md          ← Authoritative on software versions
3. CLAUDE.md (this file) ← Operational rules — what to do now
4. SPECIFICATION.md      ← Product truth — architecture, API, DB
5. TOOL-ARCHITECTURE.md  ← Scan engine design
6. IMPLEMENTATION-PLAN.md ← Task-by-task build plan
7. OPERATIONS-RUNBOOK.md ← Deploy + ops
```

**Example:** IMPLEMENTATION-PLAN.md says `fastapi = "^0.110.0"`. VERSIONS.md says `^0.135.0`. → Use `^0.135.0`.

---

## Core Operating Rules

### Rule 1: TDD Is Mandatory

Every feature follows this exact sequence:

```
1. Write failing test
2. Run test → verify it fails
3. Write minimal implementation
4. Run test → verify it passes
5. Refactor if needed
6. Commit
```

**If you skip step 1 or step 2, stop and start over.** The failing test is how we prove the test actually tests something.

### Rule 2: Commit After Every Task

Use the exact commit messages shown in IMPLEMENTATION-PLAN.md. Small, focused commits are easier to revert.

Format:
```
<type>(<scope>): <short description>

<optional body explaining why, not what>
```

Types: `feat`, `fix`, `refactor`, `test`, `docs`, `chore`, `ci`, `build`

Examples:
- `feat(auth): add JWT refresh token rotation`
- `fix(scans): handle MobSF timeout gracefully`
- `test(mobile): add APK magic byte validation tests`

### Rule 3: Never Deviate Silently

If you think "I could simplify this" or "I don't need that test" — **stop**. These docs were written after extensive analysis.

If something genuinely seems wrong:
1. Re-read the relevant section of SPECIFICATION.md and TOOL-ARCHITECTURE.md
2. Check if there's an ADR explaining the decision
3. If still unclear, ask the user — don't silently change course

### Rule 4: Escalate Ambiguity

You have decision authority for **Type 1** (reversible, small scope) choices. Everything else escalates.

**Escalate to user when:**
- Task description is ambiguous or contradicts other docs
- You'd need to add a new dependency not in VERSIONS.md
- You'd need to change database schema beyond what the task specifies
- You encounter an actual security issue
- The task involves customer data handling
- You'd need to disable a test to continue
- Production deploy approval is needed

**Decide yourself when:**
- Implementation detail within task boundaries
- Choice between equivalent libraries already listed in VERSIONS.md
- Error message wording
- Variable naming (within CONSTITUTION.md §13.6 rules)
- Test organization

### Rule 5: Verify Versions Before Installing

Before any `pip install`, `npm install`, `go install`, `apt install`, or `docker pull`:

```bash
./scripts/verify-versions.sh
```

If the script doesn't exist yet (early in the project), **you create it first** per VERSIONS.md §3.

Never install anything with `:latest` or an unpinned version.

### Rule 6: Secrets Don't Go In Git

Before every commit, check for:
- API keys (especially Anthropic, OpenAI, Stripe)
- Database passwords
- JWT secret keys
- Fernet keys
- Any value that looks like a credential

If a secret is accidentally committed:
1. **Do not just delete it** — Git remembers
2. Rotate the secret immediately
3. Remove from git history using `git filter-repo` or BFG
4. Force-push (emergency exception to Rule "no force push")
5. Document the incident

### Rule 7: Respect the Constitution

CONSTITUTION.md §20 lists forbidden practices. Memorize them:
- No `eval()` / dynamic code execution
- No shell injection patterns
- No hard-coded credentials
- No `except: pass`
- No commented-out code
- No `:latest` Docker tags in production
- No bypassing CI
- No manual production database edits

If a task seems to require a forbidden practice, escalate to user.

---

## ShieldScan-Specific Gotchas

### Gotcha 1: Mobile Scans Are Not Web Scans

MobSF is a **persistent Docker service**, not a per-scan container. It must be running before any mobile job starts. The integration is **4 REST API calls** (upload → scan → status → report), not a single subprocess call.

See TOOL-ARCHITECTURE.md §9 for the full flow. Don't treat MobSF like Nuclei or Semgrep.

### Gotcha 2: Recon Runs Before Web Scans

Every web scan starts with Subfinder + httpx. Discovered live subdomains are added to the scan target list. This can expand a "scan example.com" request to actually scanning 10-50 hosts.

Safety rails:
- Max 100 subdomains per scan (tier-dependent)
- Only subdomains of verified root domain
- Wildcard DNS detection (abort if > 50% same response)

See TOOL-ARCHITECTURE.md §8.

### Gotcha 3: Row-Level Security Is Everywhere

Every tenant-scoped table has PostgreSQL RLS. If you add a new tenant table:

1. Add `TenantMixin` to the model
2. Add the RLS policy in the migration
3. Write a test proving cross-tenant access fails

Do not skip step 3. Silent RLS misses are critical security holes.

### Gotcha 4: Fingerprint Before AI Pipeline

Every RawFinding gets a deterministic fingerprint before entering the AI pipeline. The fingerprint is `SHA-256(tool_name|finding_type|target_url|parameter|code_file|code_line)`.

This is used for:
- Primary-pass dedup (same fingerprint = definitely duplicate)
- Scan comparison (finding exists in both scans = persisting)
- Idempotency (same fingerprint across runs = same vulnerability)

Don't change the fingerprint algorithm without migrating existing data.

### Gotcha 5: AI Costs Matter

Every Claude/OpenAI call:
- Logs cost to `ai_api_calls` table
- Respects per-scan budget (see SPECIFICATION.md §9)
- Uses cached responses when possible
- Has a circuit breaker (Redis flag) for emergencies

If you add a new AI call, you must add cost tracking. Un-tracked AI calls are forbidden.

### Gotcha 6: Two Repositories, Shared Contracts

- `shieldscan-api` (Python FastAPI + React) — main application
- `shieldscan-engine` (Go) — scan workers + CLI + on-prem agent

They communicate **only via Redis** — job queue + pub/sub. The contract is documented in SPECIFICATION.md §7. If you change a contract:

1. Update SPECIFICATION.md §7 first
2. Update both sides (API and engine)
3. Deploy API first (backward compatible)
4. Deploy engine second

### Gotcha 7: Idempotency Keys on Jobs

Every scan job has an `idempotency_key` in format `{scan_id}:{tool}:{timestamp}`. Workers check Redis for the key before processing.

If the same job is dispatched twice (retry, network issue), the second execution is silently dropped.

Never remove the idempotency check. Never reuse an idempotency key.

### Gotcha 8: Workers Drain Before Updates

You can't just restart a Go worker — it might have a 20-minute MobSF scan in progress. Use the drain protocol:

```bash
redis-cli HSET worker:$id status "draining"
# wait for active_scans to reach 0
systemctl stop shieldscan-worker
```

See OPERATIONS-RUNBOOK.md §4.4.

### Gotcha 11: Ubuntu 24.04 Blocks `pip install` — Use pipx

**On Ubuntu 24.04 LTS (our target OS)**, PEP 668 is enforced. Running `pip3 install <package>` system-wide fails with `error: externally-managed-environment`.

**Never add `pip install` commands to `provision-worker.sh` on 24.04.** They will fail. Use `pipx` instead for CLI tools:

```bash
# WRONG on 24.04:
pip3 install semgrep==1.95.0

# RIGHT on 24.04:
pipx install semgrep==1.95.0
ln -sf /root/.local/bin/semgrep /usr/local/bin/semgrep
```

Our Python-based security tools (Semgrep, SSLyze, Wapiti, Checkov) are CLI tools — they install cleanly via pipx, which gives each tool its own isolated venv automatically.

**For the shieldscan-api Python project**, use Poetry as normal (Poetry creates its own venv and isn't affected by PEP 668).

See VERSIONS.md Appendix C for the full 24.04 provisioning guide.

### Gotcha 12: RLS Requires Non-Superuser App Role

PostgreSQL superusers bypass RLS unconditionally — **`FORCE ROW LEVEL SECURITY` does NOT help**. The app connects as `shieldscan_app` (non-superuser), never as `shieldscan` (admin/migration role). Tests enforce this by calling `SET ROLE shieldscan_app` on every test connection. If you see RLS behavior differ between dev and tests, **first check the connection's current role**.

Also: custom GUCs like `app.current_org_id` are session-local. SQLAlchemy releases connections back to the pool on `commit()`, which strands the GUC. For tests, bind `AsyncSession` to a single persistent `AsyncConnection`. For production, either `SET` the GUC at the start of every request OR use a per-request connection checkout that re-applies it.

See OPERATIONS-RUNBOOK.md §11.5 for the full role model.

---

## Workflow Patterns

### Starting a New Task

```
1. Read the task from IMPLEMENTATION-PLAN.md
2. Check git log — is the previous task committed?
3. Create the test file listed under "Files"
4. Write the failing test from "Step 1"
5. Run the test — confirm failure for expected reason
6. Implement per "Step 2"
7. Run the test — confirm pass
8. Commit with the exact message from the task
9. Move to next task
```

### When Tests Fail Unexpectedly

```
1. Read the error message fully — don't skim
2. Re-read the failing test — did you understand it correctly?
3. Check if a dependency issue (run verify-versions.sh)
4. Check if a previous task's output is missing
5. If genuinely stuck after 15 minutes, escalate to user
```

### When You Break Something Unrelated

```
1. Stop what you're doing
2. git status — is the current task's file still unsaved?
3. If yes: stash your changes, investigate the break
4. If no: you've committed broken code — revert immediately
5. Document what happened before resuming
```

### When Creating a Pull Request (when applicable)

```
1. Ensure all tests pass locally
2. Squash WIP commits, keep meaningful ones
3. PR title matches commit message format
4. PR description references the task in IMPLEMENTATION-PLAN.md
5. Link any ADRs created
6. Self-review the diff before requesting human review
```

---

## Common Pitfalls

### Pitfall 1: Writing Code First, Tests After

**Problem:** You write 200 lines of code, then "add tests". The tests are biased toward the code that exists, not the behavior needed.

**Solution:** Always write the test first. The test specifies behavior. The code makes the spec pass.

### Pitfall 2: Fixing Symptoms Instead of Causes

**Problem:** A test is flaky, so you add `@pytest.mark.flaky`. A database query is slow, so you add caching. A background job fails, so you increase the retry count.

**Solution:** Find the root cause. Flaky tests mean buggy code. Slow queries mean bad indexes. Failing jobs mean broken error handling.

### Pitfall 3: "I'll Clean This Up Later"

**Problem:** Commented-out code, TODO without owner, ignoring a lint warning, `# type: ignore` comments. They accumulate forever.

**Solution:** Fix it now or create a proper ticket. "Later" doesn't exist in startups.

### Pitfall 4: Over-Engineering

**Problem:** "What if we need this to be configurable?" leads to 10 levels of abstraction for a feature used once.

**Solution:** Build for the current requirement. Refactor when the second requirement arrives (rule of three).

### Pitfall 5: Copying Code Between Files

**Problem:** Bug fixed in one place, same bug still alive in three other places.

**Solution:** Extract to shared module the moment you copy-paste. Don't wait for "three instances".

### Pitfall 6: Ignoring Type Errors

**Problem:** `mypy` or `tsc` warns about a type mismatch. You `# type: ignore` it.

**Solution:** Fix the underlying type issue. Type errors are bugs caught by the type checker. Ignoring them re-enables the bug.

### Pitfall 7: Premature Optimization

**Problem:** You spend 2 hours optimizing a query that runs 10 times a day because it "feels slow".

**Solution:** Profile first. Measure the actual hot path. Optimize only what monitoring shows is a bottleneck.

---

## What To Do When...

### ...the plan says use library X but VERSIONS.md says Y

Use Y. VERSIONS.md wins on versions.

### ...you can't find information in the docs

Search order:
1. grep all 7 docs: `grep -ri "search term" /path/to/docs/`
2. Check git log for recent related commits
3. Ask the user

### ...a task depends on something not yet built

You're probably skipping ahead. Check:
1. Did you complete the previous milestone?
2. Is there a dependency listed in the task that's not done?
3. If truly a plan bug, document it and ask user how to proceed.

### ...a test is asking you to test something impossible

You probably misunderstood the test. Re-read:
1. The full test code
2. The "Step 1" description
3. The fixtures being used

If still stuck, the task description may be ambiguous — ask user.

### ...you discover a bug in an older task

Don't silently fix it as part of current task. Instead:
1. Write a failing test demonstrating the bug
2. Commit the test (will be red in CI)
3. Fix in a separate commit
4. Now continue current task

This creates a clean history showing the bug and the fix.

### ...the user asks for something outside the plan

Clarify:
- Is this a change to the plan? → Document as an addendum, update plan
- Is this a bug fix? → Create a dedicated task
- Is this a new feature? → Create an ADR, slot into appropriate milestone
- Is this a question? → Answer without changing anything

### ...you hit a rate limit on an external API

1. Don't retry in a tight loop
2. Exponential backoff with jitter
3. Respect `Retry-After` headers
4. If persistent, switch to fallback path (e.g., queue scan for later)
5. Alert user if blocking development

### ...production deploys need to happen

Claude Code does not deploy to production. Only humans approve production deploys. If a task says "deploy to production", it means:
1. Prepare the deployment artifact
2. Document what changed
3. Prepare rollback plan
4. Hand off to human for approval

---

## Quality Bar

### Your code must meet these standards before commit:

- [ ] Tests pass (ran them, don't just assume)
- [ ] Type checker passes (`mypy`, `tsc`, `go vet`)
- [ ] Linter passes (`ruff`, `eslint`, `golangci-lint`)
- [ ] Formatter applied (`ruff format`, `prettier`, `gofmt`)
- [ ] No new warnings introduced
- [ ] No `TODO` without owner
- [ ] No commented-out code
- [ ] No secrets in the diff
- [ ] Commit message follows format
- [ ] Docs updated if contracts changed

### Red flags that pause the commit:

- Any test you disabled
- Any type error you suppressed
- Any security check you bypassed
- Any dependency you added not in VERSIONS.md
- Any file larger than 1000 lines
- Any function longer than 50 lines
- Any error caught with `except: pass`

---

## Communication With User

### Be Concise

When reporting progress:
- What you did (1-2 sentences)
- What broke, if anything (1-2 sentences)
- What you need from user (1 sentence, if anything)

Don't narrate every step. The user can read git log.

### Be Honest About Uncertainty

- "I'm not sure how to handle X — options are A or B, I lean B because..."
- "The test is flaky; I couldn't determine why. Suggest investigation."
- "This task's spec conflicts with SPECIFICATION.md §5.2 — how should I proceed?"

### Don't Over-Promise

- Don't say "I'll make it perfect" — say "I'll make it pass the tests in the plan"
- Don't say "I'll handle all edge cases" — say "I'll handle the edge cases the tests specify"
- Don't say "this is production-ready" — humans decide that after review

### Pick Your Moments to Ask

Don't ask for permission for every step. The plan has already answered most questions. Ask when:
- Spec is ambiguous
- You'd add something not in the plan
- You'd remove something from the plan
- Security judgment is needed
- Customer impact is possible

---

## Off-Ramps

### When to Stop and Wait for Human

1. Any task would violate CONSTITUTION.md
2. Production environment work
3. Customer data access
4. Financial system changes (billing, pricing)
5. Legal or compliance commitments
6. Security incident response
7. Anything involving real money

### When to Definitely Ask First

1. "I think the plan is wrong here"
2. "I'd like to add a library not in VERSIONS.md"
3. "This test is hard — can I skip it?"
4. "Should this be in scope for this task?"
5. "I found a bug unrelated to my current task"

---

## Final Reminder

You are a tool, not a team member. Your code is the responsibility of the human who merges it. Your decisions are subject to human review. Your authority is limited to Type 1 decisions.

This is not a limitation — it's what allows you to move fast on the 95% of work that's clearly specified. The 5% that requires human judgment is exactly what humans should do.

**Do the work in the plan. Follow the rules in the constitution. Escalate when unclear. Commit often. Test first.**

That's the job.

---

## Quick Reference Card

| Situation | Action |
|---|---|
| Starting session | Read CLAUDE.md, check git log, find current milestone |
| Before installing anything | Run `scripts/verify-versions.sh` |
| Writing new code | Failing test first, always |
| Finishing a task | Commit with plan's exact message |
| Encountering ambiguity | Re-read docs, grep for answers, then ask |
| Found a bug in old code | Write test, commit test, fix in separate commit |
| Need to add dependency | Check VERSIONS.md — if not there, ask user |
| Security-sensitive change | Always escalate to user |
| Production deploy | Prepare artifact, hand off to human |
| Out of session, context lost | Check git log, last 5 commits tell you where to resume |

---

*End of CLAUDE.md. Now you know how to work on ShieldScan. Go build something great.*
