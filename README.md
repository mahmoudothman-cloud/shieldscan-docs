# ShieldScan Documentation Package

**For Claude Code:** Read this file first. It tells you in what order to consume the other documents and how they relate.

---

## Document Set

This package contains **10 documents** for building ShieldScan — an AI-powered, 10-category security testing platform with mobile security, recon-first pipeline, verified secret scanning, and MENA-focused go-to-market.

| # | File | Read Order | Purpose |
|---|---|---|---|
| 0 | **README.md** (this file) | First | Orientation, how to use the set |
| 1 | **CLAUDE.md** | **Second — MANDATORY for Claude Code** | Operational rules, workflow, gotchas, what to do when |
| 2 | **CONSTITUTION.md** | Third | Team governance + engineering rules — absolute authority |
| 3 | **VERSIONS.md** | **Fourth — MANDATORY** | Pinned software versions + verification procedure |
| 4 | **SPECIFICATION.md** | Fifth | Product spec — vision, architecture, DB, API, AI, pricing, ADRs |
| 5 | **TOOL-ARCHITECTURE.md** | Sixth | Deep dive on the 24 scan tools + MobSF + recon pipeline |
| 6 | **IMPLEMENTATION-PLAN.md** | Seventh — primary working doc | 15 milestones, 80+ TDD tasks with exact code |
| 7 | **ADDENDUM-CRITICAL-5.md** | **Eighth — extends spec + plan** | 5 launch-critical features: public scan, benchmarking, scheduling, onboarding, badges |
| 8 | **ADDENDUM-TOOLS-5.md** | **Ninth — extends tool architecture** | 5 additional scanning tools: trufflehog, Gobuster, Amass, Kiterunner, jsluice |
| 9 | **OPERATIONS-RUNBOOK.md** | Reference | Deployment, monitoring, incident response, DR |

---

## Execution Flow for Claude Code

```
┌─────────────────────────────────────────────────────────┐
│ 1. Read README.md (this file) — orientation             │
└────────────────────┬────────────────────────────────────┘
                     │
                     ▼
┌─────────────────────────────────────────────────────────┐
│ 2. Read VERSIONS.md — learn pinned versions             │
│    RUN scripts/verify-versions.sh BEFORE INSTALLING      │
│    ANYTHING. If versions drifted, use §4 decision matrix│
└────────────────────┬────────────────────────────────────┘
                     │
                     ▼
┌─────────────────────────────────────────────────────────┐
│ 3. Read SPECIFICATION.md — product context              │
│    Understand what we're building and why               │
└────────────────────┬────────────────────────────────────┘
                     │
                     ▼
┌─────────────────────────────────────────────────────────┐
│ 4. Read TOOL-ARCHITECTURE.md — scan engine design       │
│    Understand the ToolRunner interface + MobSF flow     │
└────────────────────┬────────────────────────────────────┘
                     │
                     ▼
┌─────────────────────────────────────────────────────────┐
│ 5. Execute IMPLEMENTATION-PLAN.md task by task          │
│    Use superpowers:executing-plans                      │
│    Commit after every task. Run tests. TDD throughout.  │
└────────────────────┬────────────────────────────────────┘
                     │
                     ▼
┌─────────────────────────────────────────────────────────┐
│ 6. Reference OPERATIONS-RUNBOOK.md when:                │
│    - Setting up worker servers (§4)                     │
│    - Configuring CI/CD (§5)                             │
│    - Setting up monitoring (§7)                         │
│    - Incidents happen (§8)                              │
└─────────────────────────────────────────────────────────┘
```

---

## Document Authority Order

When documents conflict, **this is the resolution order** (highest authority first):

```
CONSTITUTION.md         (absolute — ethics, governance, non-negotiable rules)
        ↓
VERSIONS.md             (wins on software versions — always)
        ↓
CLAUDE.md               (operational rules for implementation)
        ↓
ADDENDUM-CRITICAL-5.md  (wins on public scan, badges, benchmarks, onboarding, scheduling)
        ↓
ADDENDUM-TOOLS-5.md     (wins on trufflehog, Gobuster, Amass, Kiterunner, jsluice integration)
        ↓
SPECIFICATION.md        (wins on architecture, data model, API contracts)
        ↓
TOOL-ARCHITECTURE.md    (wins on scan engine internals for the original 19 tools)
        ↓
IMPLEMENTATION-PLAN.md  (wins on build order and code)
        ↓
OPERATIONS-RUNBOOK.md   (wins on deployment and operations)
```

**Example:** IMPLEMENTATION-PLAN.md says "Python ^3.12". VERSIONS.md says "Python 3.13.13". → Use Python 3.13.13.

**Example:** A task tempts you to skip a test to save time. CONSTITUTION.md §15.1 says TDD is mandatory. → Follow the constitution.

**Example:** ADDENDUM adds a `public_scans` table not in SPECIFICATION.md. → The addendum wins; add the table. Treat ADDENDUM as extending SPEC, not conflicting with it.

**Example:** TOOL-ARCHITECTURE.md §10 shows 19 tools in the matrix. ADDENDUM-TOOLS-5.md §9 shows 24 tools. → Use the 24-tool matrix; the addendum extends the architecture.

---

## Critical Rules for Claude Code

### Rule 1: Versions

**ALWAYS run `scripts/verify-versions.sh` before installing anything.** VERSIONS.md pins versions current as of 2026-04-20. Some may have newer stable releases. The script tells you which.

### Rule 2: TDD

**Every task in IMPLEMENTATION-PLAN.md follows TDD:**
1. Write failing test
2. Run test — verify it fails
3. Implement minimal code
4. Run test — verify it passes
5. Commit

Never skip the failing test step. It's how we prove the test actually tests something.

### Rule 3: Commits

**Commit after every task.** Use the commit messages shown in the plan. Small, focused commits are easier to revert if anything goes wrong.

### Rule 4: Don't Deviate

**The plan is deliberate.** If you think "I could simplify this" or "I don't need this test" — stop. These docs were written after extensive analysis. Deviations compound into weeks of debugging.

If something genuinely seems wrong, **ask the user** before changing. Don't silently improve.

### Rule 5: Mobile Security Is Special

MobSF is a persistent Docker service, not a per-scan container. It stores state, exposes a REST API, and must be running before the worker accepts mobile jobs. See TOOL-ARCHITECTURE.md §9 for the full flow. Don't treat MobSF like the other tools.

### Rule 6: Row-Level Security Is Non-Negotiable

Every tenant-scoped table has PostgreSQL RLS enabled. See IMPLEMENTATION-PLAN.md Task 1.6. If you add a new tenant table later, you MUST add RLS to it. No exceptions.

---

## Project Summary

**What we're building:** An AI-powered web and mobile application security testing SaaS for the MENA market, with:
- 9 scan categories: DAST, SAST, Mobile, SCA, Infrastructure, Recon, SSL/TLS, API, IaC+Container
- 19 integrated tools — 11 native binaries + 4 persistent Docker services + 4 conditional tools
- AI cross-layer correlation (DAST↔SAST) — unique in the market
- AI deduplication via vector similarity — 40-60% noise reduction
- Mobile security via MobSF (APK, IPA, source analysis for Android + iOS)
- Recon-first pipeline (Subfinder + httpx) discovers subdomains automatically
- React dashboard, REST API, Go CLI, GitHub Action, on-prem agent
- Stripe billing, SOC2 + ISO 27001 compliance mapping
- Phase 2: Arabic UI, WhatsApp alerts, insurance premium integration, security reputation badge

**Tech stack (see VERSIONS.md for exact versions):**
- API: FastAPI (Python 3.13) + SQLAlchemy 2.0 + PostgreSQL 18 + Redis 8 + Qdrant
- Engine: Go 1.26 + asynq + Docker SDK + MobSF REST API
- Frontend: React 18 + TypeScript + Tailwind + TanStack Query
- AI: Claude (Opus + Sonnet + Haiku) + OpenAI embeddings
- Infra: Hetzner workers + Fly.io API + Cloudflare R2 + Cloudflare edge

**Two repositories:**
- `shieldscan-api` — Python FastAPI app + React frontend (monorepo)
- `shieldscan-engine` — Go scan workers + CLI + on-prem agent

**Timeline:** 10 weeks to launch-ready MVP. See IMPLEMENTATION-PLAN.md timeline summary.

---

## Starting Implementation

### Option A: Parallel Claude Code Sessions (Recommended)

Open two Claude Code sessions — one per repository.

**Session 1 — shieldscan-api:**
```
Project: /path/to/shieldscan-api
Docs: /path/to/docs/shieldscan/
Start: Read README.md → VERSIONS.md → verify-versions.sh
       Then follow IMPLEMENTATION-PLAN.md Milestones 1-4, 9-12
```

**Session 2 — shieldscan-engine:**
```
Project: /path/to/shieldscan-engine
Docs: /path/to/docs/shieldscan/
Start: Read README.md → VERSIONS.md → verify-versions.sh
       Then follow IMPLEMENTATION-PLAN.md Milestones 5-8, 13-14
```

Both can run in parallel since the repos are decoupled via Redis contracts (see SPECIFICATION.md §7).

### Option B: Sequential Single Session

If running one session, follow the plan milestone order exactly. Don't skip ahead.

---

## Pre-Flight Checklist

Before Claude Code starts implementation, human confirms:

- [ ] Both repositories exist on GitHub (or will be created by Claude Code)
- [ ] All 5 docs are accessible to Claude Code
- [ ] Accounts provisioned:
  - [ ] Anthropic API key
  - [ ] OpenAI API key
  - [ ] Stripe account (test mode to start)
  - [ ] Cloudflare account (for R2)
  - [ ] GitHub organization for OAuth app
  - [ ] SendGrid or Resend account
- [ ] Local dev machine has:
  - [ ] Docker + Docker Compose
  - [ ] Python 3.13 installed (via pyenv)
  - [ ] Go 1.26 installed
  - [ ] Node.js 22 LTS
  - [ ] PostgreSQL client (psql)
- [ ] Target deployment infrastructure identified (Fly.io + Hetzner, or similar)
- [ ] Domain registered: `shieldscan.io` (or chosen alternate)

---

## Questions During Implementation

If Claude Code encounters ambiguity not resolved by the docs:

1. **First** — search the docs for the answer (use grep/find across all 5 files)
2. **Second** — follow the Architectural Decision Records in SPECIFICATION.md §13
3. **Third** — choose the simpler option (YAGNI)
4. **Fourth** — ask the user, document the decision, add to SPECIFICATION.md ADRs

Never silently invent solutions. Document every decision.

---

## Success Criteria for Launch

From IMPLEMENTATION-PLAN.md Milestone 15:
- [ ] All 15 milestones complete
- [ ] All tests passing (unit + integration + E2E + load)
- [ ] Security self-scan: zero critical/high findings
- [ ] All integrations working (Stripe, GitHub, Slack, SendGrid)
- [ ] Production infrastructure provisioned per OPERATIONS-RUNBOOK.md §2
- [ ] Monitoring + alerting operational
- [ ] Backups tested via disaster recovery drill
- [ ] Terms of Service + Privacy Policy published
- [ ] First 5 pilot customers identified

---

*End of README. Now read VERSIONS.md.*
