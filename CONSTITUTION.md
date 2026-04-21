# ShieldScan — Constitution

**Version:** 1.0
**Date:** 2026-04-20
**Author:** Mahmoud Hassan, Odyssey Technology
**Status:** Foundational document — changes require unanimous founder approval

> This is the highest-authority document in the ShieldScan project. When any other document, decision, or action conflicts with this constitution, **the constitution wins**. This document exists to ensure ShieldScan is built correctly, ethically, and sustainably — regardless of who is doing the work.

---

## Table of Contents

**Part I — Project Governance**
1. [Purpose & Mission](#1-purpose--mission)
2. [Core Values](#2-core-values)
3. [Ownership & Authority](#3-ownership--authority)
4. [Decision-Making Process](#4-decision-making-process)
5. [Roles & Responsibilities](#5-roles--responsibilities)
6. [Ethical Boundaries](#6-ethical-boundaries)
7. [Financial Principles](#7-financial-principles)
8. [Customer Commitments](#8-customer-commitments)
9. [External Contributors & Partners](#9-external-contributors--partners)
10. [Amendment Process](#10-amendment-process)

**Part II — Engineering Constitution**
11. [Non-Negotiable Engineering Rules](#11-non-negotiable-engineering-rules)
12. [Architectural Principles](#12-architectural-principles)
13. [Code Quality Standards](#13-code-quality-standards)
14. [Security Rules](#14-security-rules)
15. [Testing Requirements](#15-testing-requirements)
16. [Deployment Rules](#16-deployment-rules)
17. [Data Handling Rules](#17-data-handling-rules)
18. [AI Usage Rules](#18-ai-usage-rules)
19. [Dependency Rules](#19-dependency-rules)
20. [Forbidden Practices](#20-forbidden-practices)

---

# Part I — Project Governance

## 1. Purpose & Mission

### 1.1 Mission Statement

**ShieldScan exists to make application security accessible to small and mid-market businesses in the MENA region and globally — providing enterprise-grade vulnerability scanning, AI-powered remediation, and compliance evidence at prices small businesses can actually afford.**

### 1.2 Why We Exist

Security testing today is broken for 95% of businesses:
- Enterprise tools (Veracode, Checkmarx) cost $50K-$200K/year — inaccessible to SMEs
- Open-source tools (ZAP, Nuclei, MobSF) require expert configuration — unusable by non-specialists
- Most tools ignore mobile security entirely
- Nobody speaks Arabic, prices in local currency, or understands MENA regulatory requirements
- Every tool was built by security engineers for security engineers — founders, lawyers, and insurance brokers are ignored

ShieldScan fixes all of this in one platform.

### 1.3 What Success Looks Like

We will have succeeded when:
- **1,000+ SMEs** in MENA rely on ShieldScan as part of their compliance and security posture
- **At least one Gulf fintech or bank** cites ShieldScan in their regulatory filings
- **Arabic-language security education** through ShieldScan has measurably improved Egyptian developer security literacy
- **Insurance premium discounts** driven by ShieldScan reports save customers more money than they pay in subscriptions
- **ShieldScan runs reliably** for years after founder involvement ends

### 1.4 What We Will Never Become

We will never become:
- A surveillance tool
- A weapon for nation-state attackers
- An ad-supported consumer product
- A platform that sells customer data
- A company that profits from insecurity rather than fixing it

---

## 2. Core Values

These are the values that guide every decision. When in doubt, return to these.

### 2.1 Honesty Over Marketing

We tell customers what our product actually does, not what sounds impressive.
- If a scan missed something, we say so
- If our tool isn't the best fit for a customer, we refer them elsewhere
- If a feature isn't ready, we don't pretend it is
- Our marketing never contradicts our documentation

### 2.2 Security Over Speed

We will never ship code that compromises customer security to meet a deadline.
- A one-week delay to fix a security issue is always better than shipping vulnerable code
- Every feature gets a security review — no exceptions
- When in doubt, we choose the more secure option

### 2.3 Customers Over Growth

We optimize for customer outcomes, not growth metrics.
- We don't add friction to make users "engage" more
- We don't hide features behind artificial paywalls
- We don't use dark patterns in billing, cancellation, or retention
- A customer who cancels because they no longer need us is a success, not a failure

### 2.4 Ethics Over Revenue

We will turn down revenue that conflicts with our mission.
- We don't sell to surveillance regimes targeting dissidents
- We don't sell to known malicious actors
- We don't help customers evade lawful security audits
- We maintain this principle even if a competitor accepts the business

### 2.5 Transparency Over Opacity

We operate in the open wherever possible.
- Our security posture is publicly documented (security.txt, status page)
- Our incident reports are public within 14 days
- Our pricing is public — no "contact sales" for standard tiers
- Our terms of service are readable in plain language

### 2.6 Longevity Over Hype

We build for decades, not exit events.
- We don't chase trends that conflict with the mission
- We don't raise money we don't need
- We optimize for sustainable unit economics from day one
- We choose boring technology when boring technology works

### 2.7 Respect for the Region

We are MENA-native, not MENA-adjacent.
- Arabic UI is first-class, not an afterthought
- Regional regulations are treated with the same seriousness as GDPR
- Local payment methods (bank transfer, cash, cheque) are supported
- Cultural context shapes our product decisions

---

## 3. Ownership & Authority

### 3.1 Ownership Structure

- **Founder & CEO:** Mahmoud Hassan (Odyssey Technology)
- **Legal entity:** Odyssey Technology (Egypt) — owner of ShieldScan product and IP
- **Future equity:** Any employee/co-founder equity grants require written agreements specifying vesting, cliff, and exit terms

### 3.2 Authority Hierarchy

When decisions need to be made:

```
Constitution (this document)
        ↓
Founder (Mahmoud)
        ↓
Product Roadmap Documents (SPECIFICATION.md + derivatives)
        ↓
Individual engineering decisions (by implementer)
```

### 3.3 Reserved Founder Decisions

Only the founder can make final decisions on:
- Core pricing and business model
- Which customers we accept or decline
- Equity grants
- Legal commitments (contracts, partnerships, acquisitions)
- Major architectural pivots
- Brand identity changes
- Amendments to this constitution

### 3.4 Delegated Decisions

Engineers and operators can make final decisions on:
- Implementation approach within architectural constraints
- Bug fix prioritization within a sprint
- Library choices within the technology stack
- Refactoring within existing modules
- Test coverage approach
- Documentation updates

---

## 4. Decision-Making Process

### 4.1 Decision Types

**Type 1 — Reversible, small scope:** Engineer decides. Example: which HTTP library to use for a specific endpoint.

**Type 2 — Reversible, larger scope:** Engineer proposes, founder acknowledges. Example: changing the React routing library.

**Type 3 — Hard to reverse:** Founder approves explicitly. Example: adding a new scan category, changing database schema strategy.

**Type 4 — Constitutional:** Founder + explicit constitutional amendment. Example: accepting government surveillance customers.

### 4.2 Disagreement Resolution

When team members disagree:
1. Write down both positions with reasoning (10 minutes each)
2. Identify the specific testable assumption in dispute
3. If testable in < 1 day: run the test
4. If not testable: founder decides
5. Document the decision in an ADR (Architectural Decision Record)

### 4.3 Speed vs Consensus

**In development:** Lean toward speed. Fix broken decisions later.
**In production:** Lean toward consensus. Broken production decisions hurt customers.
**In security:** Always pause for review. Security mistakes are rarely cheap.

### 4.4 Documentation Requirement

**Every significant decision must be documented.** If it's not in Git, it didn't happen.
- Architectural decisions → ADRs in SPECIFICATION.md §13
- Customer commitments → written contracts
- Internal policy decisions → this constitution or its amendments
- Financial decisions → ledger + memo

---

## 5. Roles & Responsibilities

### 5.1 Current Roles

**Founder (Mahmoud):**
- Product vision and roadmap
- Customer relationships (early stage)
- Major architectural decisions
- Financial stewardship
- Brand and marketing

**Claude Code (AI development agent):**
- Implementation of approved specifications
- Test writing and execution
- Documentation maintenance
- Code review (automated)
- **Not authorized** to make Type 3 or Type 4 decisions independently

**Future roles (to be filled):**
- SRE / DevOps engineer
- Security researcher
- Customer support
- Sales (Gulf region)
- Legal counsel (as-needed)

### 5.2 Responsibility Principles

- **One owner per responsibility.** No ambiguous "team owns this" situations.
- **Explicit handoff.** When ownership changes, it's documented.
- **No surprise ownership.** You cannot be made responsible for something without being told explicitly.

### 5.3 Relationship with AI Agents

Claude Code and similar AI agents are tools, not team members. Specifically:
- AI agents do not have final decision authority
- AI agents must escalate to humans when facing ambiguity
- AI-generated code is the responsibility of the human who merged it
- AI agents cannot commit to customers on behalf of ShieldScan
- AI agents follow this constitution — deviations are bugs

---

## 6. Ethical Boundaries

These are the lines we will not cross. They exist because short-term profit is not worth long-term damage.

### 6.1 Customer Acceptance Rules

**We will accept customers who:**
- Operate legitimate businesses with web/mobile applications
- Need security testing for their own property
- Can demonstrate ownership of domains/apps they ask us to scan

**We will reject customers who:**
- Attempt to scan infrastructure they don't own
- Show patterns consistent with attack preparation
- Are on relevant sanctions lists
- Request customizations clearly aimed at weaponizing our tools
- Are known surveillance state actors targeting civilians

### 6.2 Scan Authorization

- **Mandatory domain verification** before any scan — no exceptions
- Users must attest to ownership via signed ToS
- Verified ownership does not extend to subdomains we auto-discover that are owned by third parties
- On-prem agent deployments require explicit written authorization

### 6.3 Vulnerability Disclosure

When we discover a zero-day or unreported vulnerability during our work:
- We report to the affected vendor within 48 hours
- We follow responsible disclosure (90-day timeline typical)
- We do not weaponize findings
- We do not sell findings to buyers outside the affected vendor
- We publish write-ups only after vendor confirmation

### 6.4 Data Use

- We never sell customer data
- We never train AI models on customer scan data without explicit opt-in
- We never share customer findings between organizations (unless explicitly consented for aggregate threat intelligence)
- We comply with the strictest applicable regulation (typically GDPR)
- Customer data is customer property — we're just holding it

### 6.5 Government Relations

- We comply with lawful court orders with jurisdiction over us
- We publish transparency reports disclosing aggregate government requests
- We challenge overbroad requests through legal channels
- We never provide "back doors" or direct access to customer data
- We notify customers of government requests unless legally prohibited

---

## 7. Financial Principles

### 7.1 Unit Economics First

We maintain positive unit economics from day one:
- Each subscription must cover its variable costs (AI, storage, infrastructure) with margin
- We do not subsidize unsustainable customer acquisition
- Growth without margin is death, not victory

### 7.2 Cash Flow Discipline

- We maintain 12+ months of runway at all times once revenue exceeds $10K MRR
- Major expenses require 30-day advance consideration
- We pay vendors on time — late payment damages relationships and reputation
- We collect from customers on time — generosity on receivables starves cash flow

### 7.3 Investment Philosophy

**We will only take outside investment when:**
- The money unlocks growth we can't achieve organically
- The investor understands and respects our mission
- The terms preserve founder control of mission-critical decisions
- The valuation is defensible on fundamentals, not hype

**We will decline investment that:**
- Requires abandoning the MENA focus
- Forces us to accept customers we'd otherwise reject
- Creates pressure for liquidity events that conflict with long-term health
- Dilutes founder equity below effective control thresholds

### 7.4 Revenue Diversity

We do not let any single customer exceed 20% of MRR. If approached by a customer that would do so, we either:
- Decline, OR
- Negotiate terms that account for concentration risk

### 7.5 Pricing Integrity

- Published prices are real prices — no hidden discounts for "if you push back"
- Enterprise pricing is negotiable; SME pricing is not
- Price increases require 60-day notice to existing customers
- Grandfathered pricing is honored until the customer terminates or materially changes their plan

---

## 8. Customer Commitments

These are the promises we make to every customer.

### 8.1 Service Commitments

- **Uptime:** 99.9% for paid tiers, measured monthly
- **Scan accuracy:** We will remediate confirmed false positives within 30 days of report
- **Support response:** Business+ customers get response within 1 business day
- **Data availability:** Customer data accessible for download for 90 days after cancellation

### 8.2 Transparency Commitments

- **Status page** (status.shieldscan.io) is always current
- **Incident reports** for SEV-1 incidents published within 14 days
- **Terms of Service** changes require 30-day notice
- **Privacy Policy** changes affecting data handling require 60-day notice
- **Security posture** documented publicly and updated quarterly

### 8.3 Exit Commitments

When customers leave:
- **Full data export** in open formats (JSON, SARIF, PDF) available for 90 days
- **No lock-in penalties** or cancellation fees
- **Credit cards not charged** beyond the billing period of cancellation
- **Data deletion** completed within 30 days of cancellation upon request
- **No "retention specialists"** or dark-pattern retention flows

### 8.4 Complaint Process

Every customer complaint receives:
- Acknowledgment within 1 business day
- Investigation by a qualified team member
- Written response with findings and actions
- Escalation path to the founder for unresolved issues

---

## 9. External Contributors & Partners

### 9.1 Open Source Commitments

ShieldScan depends on open source. We give back by:
- Contributing bug fixes to the tools we integrate
- Submitting improvements to Nuclei templates, MobSF rules
- Maintaining our own open-source CLI and GitHub Action
- Acknowledging open-source projects in our product and marketing
- Sponsoring critical infrastructure projects once revenue permits

### 9.2 Partnership Criteria

Partners (resellers, integrators, channel partners) must:
- Agree to this constitution's ethical rules
- Sign agreements with clear termination clauses
- Maintain customer acceptance standards matching ours
- Not misrepresent ShieldScan's capabilities

### 9.3 Contractor Rules

External contractors working on ShieldScan:
- Sign NDA + IP assignment before access
- Read and acknowledge this constitution
- Cannot commit directly to main branch
- Work is reviewed by internal team before merge

### 9.4 Academic & Research Use

We offer free or heavily discounted access to:
- Accredited educational institutions
- Independent security researchers
- Non-profit security organizations

Conditions: findings are shared back, no commercial resale, our name is used appropriately.

---

## 10. Amendment Process

### 10.1 How to Change This Document

This constitution changes only through a documented amendment process:

1. **Proposal** — Any team member can propose changes in writing
2. **Discussion period** — Minimum 7 days for review
3. **Founder approval** — Founder signs off explicitly
4. **Version bump** — New version number, old version archived
5. **Team acknowledgment** — All team members re-read and acknowledge

### 10.2 What Cannot Be Amended

Certain sections are permanently locked:

- **Section 1.4** (What we will never become)
- **Section 6** (Ethical Boundaries)
- **Section 2.4** (Ethics Over Revenue)

These can only change if the founder explicitly decides to change the company's fundamental identity — and that must be communicated publicly.

### 10.3 Emergency Amendments

If immediate change is required (legal compliance, safety issue), the founder can make a temporary amendment with:
- Documented reason
- 30-day review deadline
- Full amendment process or rollback within 30 days

---

---

# Part II — Engineering Constitution

## 11. Non-Negotiable Engineering Rules

These rules cannot be broken. Not for a deadline. Not for a customer. Not for any reason. Violations are grounds for reverting code and re-doing the work correctly.

### 11.1 The Ten Commandments

1. **Thou shalt run `scripts/verify-versions.sh` before any installation.**
2. **Thou shalt write the failing test before writing the implementation.**
3. **Thou shalt never commit secrets, API keys, or customer data to Git.**
4. **Thou shalt never disable Row-Level Security on tenant tables.**
5. **Thou shalt never scan a target without verified domain ownership.**
6. **Thou shalt never hide AI costs from logging and monitoring.**
7. **Thou shalt never log customer credentials, session tokens, or PII.**
8. **Thou shalt never break the deployment pipeline to "fix it later".**
9. **Thou shalt never use `:latest` Docker tags in production.**
10. **Thou shalt never deploy code that hasn't passed automated tests.**

### 11.2 When Rules Conflict

If two rules conflict, the more conservative (safer) rule wins.

If a rule appears to conflict with customer requirements, escalate to the founder. Do not silently bend the rule.

If a rule appears to prevent a legitimate need, propose a constitutional amendment — don't ignore the rule.

---

## 12. Architectural Principles

### 12.1 Simplicity Over Cleverness

Code is read 10x more than it is written. We optimize for readers.
- Choose boring technology when boring technology works
- Use well-known patterns over novel ones
- Write the simplest thing that could possibly work
- If a junior engineer can't understand it in 5 minutes, rewrite it

### 12.2 Explicit Over Implicit

- No magic
- No framework "auto-discovery" where explicit imports work
- No hidden dependencies between modules
- Every API contract is documented, not inferred

### 12.3 Boundaries Matter

Our bounded contexts (see SPECIFICATION.md §4) have clear interfaces:
- Python API and Go workers communicate ONLY via Redis
- React frontend and API communicate ONLY via HTTP
- AI pipeline is a black box callable from orchestrator — no direct database access from inside
- Cross-boundary changes require updating the contract document

### 12.4 Fail Gracefully

- Partial success is better than total failure
- If 5 of 8 tools succeed, report findings from the 5
- If AI pipeline fails, show vulnerabilities without fixes
- If Qdrant is down, degrade to fingerprint-based dedup
- Never fail silently — always log and alert

### 12.5 Scale Second, Correctness First

We prioritize correctness over performance in every tradeoff.
- Slow and correct beats fast and wrong
- Optimize only after profiling shows a specific bottleneck
- Scale problems are easier to fix than correctness bugs
- Premature optimization is still the root of much evil

### 12.6 Data Flow Direction

Data flows in one direction through the system:
```
User Input → Validation → Business Logic → Storage → Retrieval → Presentation
```
Do not shortcut this flow. Do not let presentation layer write to storage. Do not let validation happen at the storage layer.

---

## 13. Code Quality Standards

### 13.1 General

- **Line length:** 100 characters soft limit, 120 hard limit
- **File length:** 500 lines soft, 1000 hard. Larger files get split.
- **Function length:** 50 lines soft. Longer functions get extracted.
- **Cyclomatic complexity:** 10 soft limit per function
- **Comments:** Explain *why*, not *what*. The code explains what.

### 13.2 Python (shieldscan-api)

- **Linter:** `ruff` (configured in pyproject.toml)
- **Formatter:** `ruff format` (replaces black)
- **Type checker:** `mypy` with strict mode
- **All public functions:** must have type hints
- **All endpoints:** must have Pydantic request/response schemas
- **All database queries:** must use SQLAlchemy 2.0 async syntax
- **No raw SQL** unless behind a repository abstraction

### 13.3 Go (shieldscan-engine)

- **Linter:** `golangci-lint` with full ruleset
- **Formatter:** `gofmt` (standard)
- **All public functions:** must have doc comments starting with function name
- **All errors:** must be handled or explicitly ignored with comment
- **All contexts:** must be propagated through the call chain
- **Panics:** forbidden outside `main()` initialization
- **No `init()` functions** that do business logic

### 13.4 TypeScript (shieldscan-web)

- **Linter:** `eslint` with strict TypeScript rules
- **Formatter:** `prettier`
- **`any` type:** forbidden. Use `unknown` and narrow.
- **All components:** must be typed props interfaces
- **All API calls:** must go through TanStack Query, never raw fetch
- **No `localStorage`/`sessionStorage`** — use TanStack Query cache or server-side state

### 13.5 Documentation

- **Every module** has a top-of-file comment explaining purpose
- **Every public function** has documentation
- **Every API endpoint** is documented in OpenAPI schema
- **Every database table** has a comment on create
- **Every architectural decision** has an ADR in SPECIFICATION.md §13

### 13.6 Naming

- **Consistent vocabulary:** "scan" is a scan, "run" is a run. Don't mix terms.
- **Full words over abbreviations:** `vulnerability` not `vuln`, except in URLs/filenames
- **Domain language in code:** match the glossary in SPECIFICATION.md §14
- **No personal preferences:** Python uses snake_case, Go uses camelCase, TypeScript uses camelCase

---

## 14. Security Rules

### 14.1 Input Validation

- **Every user input** is validated before use
- **Every external API response** is validated before use
- **Every database read** is treated as untrusted input
- **Validation happens at the boundary** (API endpoint, service entry point), not throughout the code

### 14.2 Authentication & Authorization

- **Authentication:** JWT via httpOnly cookies (dashboard) or API keys (programmatic)
- **Authorization:** Every endpoint checks the user has access to the requested resource
- **No client-side authorization decisions:** the server is the source of truth
- **Session invalidation:** every password change invalidates all active sessions

### 14.3 Secrets Management

- **Secrets are environment variables or Vault entries** — never hardcoded
- **Secrets are never logged**
- **Secrets are never committed to Git** (enforced by pre-commit hook + gitleaks in CI)
- **Secrets rotate on a schedule** per OPERATIONS-RUNBOOK.md §11.1

### 14.4 Encryption

- **At rest:** Customer credentials in `project_credentials` use Fernet
- **In transit:** TLS 1.3 minimum, TLS 1.0/1.1 forbidden
- **At motion:** All queue messages containing sensitive data are encrypted or not sensitive
- **Key management:** Rotation plan documented, tested annually

### 14.5 Dependency Security

- **No dependency is trusted blindly** — every new dependency is reviewed
- **Security advisories checked weekly** via Dependabot
- **CVEs with CVSS ≥ 7.0:** patched within 48 hours
- **Deprecated dependencies:** replaced on a planned schedule

### 14.6 Audit Trail

- **Every significant action** writes to `audit_logs`
- **Audit logs are append-only** — no UPDATE or DELETE
- **Audit logs include:** who, what, when, where (IP), context
- **Audit logs are never purged** within the 7-year retention window

### 14.7 ShieldScan Scans ShieldScan

We must always scan our own infrastructure with our own product:
- Continuous self-scan in CI
- All critical findings blocked from production
- Internal penetration test annually
- Security.txt published at https://shieldscan.io/.well-known/security.txt

---

## 15. Testing Requirements

### 15.1 TDD Is Mandatory

The process is:
1. Write a failing test
2. Run the test — confirm it fails for the expected reason
3. Write the minimum code to make it pass
4. Run the test — confirm it passes
5. Refactor if needed, tests still pass
6. Commit

**Skipping step 1 or step 2 is not acceptable.** These steps prove the test actually tests something.

### 15.2 Test Coverage Minimums

- **Critical paths** (auth, scans, billing): 90%+ coverage
- **Business logic:** 80%+ coverage
- **UI components:** 60%+ coverage
- **Infrastructure scripts:** smoke tests minimum

### 15.3 Test Categories

**Unit tests:** Fast, isolated, no external dependencies. Run on every commit.

**Integration tests:** Test component interactions. Real database, real Redis, mocked external APIs. Run on every PR.

**End-to-end tests:** Full user journeys. Run before every deploy.

**Load tests:** Performance validation. Run before releases.

**Security tests:** Run against our own infrastructure. Continuous.

### 15.4 Test Independence

- Tests must not depend on each other's execution order
- Tests must not share state except through fixtures
- Tests must clean up after themselves
- Flaky tests are fixed within 48 hours or removed

### 15.5 Test Quality

- **Test names describe behavior:** `test_scan_rejects_unverified_domain`, not `test_scan_1`
- **One assertion per test** when possible
- **Tests include both happy path and error cases**
- **Tests are documented** if the behavior tested isn't obvious

---

## 16. Deployment Rules

### 16.1 The Deploy Protocol

Every deploy follows this sequence:
1. All tests pass in CI
2. Deploy to staging
3. Smoke tests pass on staging
4. Manual approval for production
5. Blue-green deploy to production
6. Health checks confirm new deployment
7. Traffic shifts
8. Old deployment kept 1 hour (rollback safety)

### 16.2 Rollback Readiness

- **Every deploy must be rollback-able within 5 minutes**
- **Database migrations must be additive** — never drop in the same deploy
- **Feature flags for risky changes** — toggle without redeploy
- **Deployment logs retained 90 days** for post-mortem

### 16.3 No Friday Deploys

No production deploys on Friday afternoons, weekends, or day before holidays — unless critical security patch.

### 16.4 Maintenance Windows

Scheduled maintenance follows OPERATIONS-RUNBOOK.md §13.1:
- Weekly windows are announcement-only (no user impact)
- Monthly windows < 5 min user impact, announced 7 days ahead
- Quarterly windows up to 1 hour, announced 7 days ahead

### 16.5 Worker Deployment

Workers drain gracefully before updates:
- Mark as draining in Redis
- Wait for active scans to complete (up to 30 min)
- Stop service
- Deploy new binary
- Health check before accepting traffic

---

## 17. Data Handling Rules

### 17.1 Data Classification

| Class | Examples | Handling |
|---|---|---|
| **Public** | Marketing content, pricing | No special handling |
| **Internal** | Team Slack, dev docs | Team-only access |
| **Customer Confidential** | Scan results, vulnerabilities | Org-scoped RLS, encrypted at rest |
| **Customer Sensitive** | API keys, auth cookies | Fernet encryption, never logged |
| **PII** | Emails, names, phone numbers | GDPR/DPL handling, export/delete on request |

### 17.2 Data Retention

Per SPECIFICATION.md §5.4. Key rules:
- **Raw findings:** 90 days
- **Vulnerabilities, scan history:** indefinite (until customer deletes)
- **Mobile uploads:** 30 days post-scan
- **Audit logs:** 7 years
- **Deleted projects:** 30-day soft delete, then hard delete

### 17.3 Data Sovereignty

- **MENA customers' data** stored in Frankfurt region (closest available)
- **EU customers' data** stored in EU region
- **Cross-border data transfer** requires customer consent
- **Local data residency** offered to enterprise customers on request

### 17.4 Data Deletion

- Customer-requested deletion: completed within 30 days
- GDPR "right to be forgotten": honored globally, not just for EU users
- Backups: deletion propagates to backups within 60 days
- Dark-pattern retention: forbidden

---

## 18. AI Usage Rules

### 18.1 AI Cost Control

- **Every AI call is logged with cost tracking**
- **Per-scan AI budget** enforced by tier (see SPECIFICATION.md §9)
- **Monthly AI spend alerting** — alert at $5K, block at $10K without approval
- **Cached responses preferred** — 40%+ cache hit rate target

### 18.2 AI Transparency

- **Customers are told** which AI models are used on their data
- **Training data isolation:** customer data is never used to train our AI models without explicit opt-in
- **AI-generated content is labeled** — fixes, summaries, etc.
- **AI hallucinations must be caught** — AI fix output validated before showing to customer

### 18.3 AI Prompt Engineering

- **Prompts are version-controlled** alongside code
- **Prompt changes require testing** — quality, cost, regression
- **Sensitive data in prompts** — minimized, never the full customer data
- **System prompts include safety guards** — refuse harmful requests

### 18.4 AI Failure Modes

- **API unavailable:** degrade to non-AI features
- **Rate limited:** queue with retry, don't drop requests
- **Invalid output:** flag for human review, don't auto-show to customer
- **Hallucinated code fix:** validated syntactically before displayed

### 18.5 Model Selection

- **Right model for the job** — don't use Opus for chat, don't use Haiku for reasoning
- **Cost/quality tradeoff** documented per use case
- **Model versions pinned** — `claude-opus-4-7`, not `claude-opus-latest`
- **Multi-provider strategy** — fallback paths when primary provider fails

---

## 19. Dependency Rules

### 19.1 Adding a Dependency

Before adding any dependency:
1. Can we reasonably implement this ourselves in < 1 day?
2. Is the library actively maintained (commit in last 6 months)?
3. Does it have ≥ 500 GitHub stars or clear reputable maintainer?
4. Is the license compatible (MIT/Apache/BSD preferred)?
5. Does it introduce security vulnerabilities (check Snyk/npm audit)?
6. Is the API stable (v1.0+)?

If no to 2+ questions, don't add it.

### 19.2 Version Pinning

Per VERSIONS.md — every dependency is pinned. No `^` or `~` in production lockfiles.

### 19.3 Dependency Updates

- **Patch updates:** auto-merged via Dependabot after CI passes
- **Minor updates:** reviewed weekly, merged if no breaking changes
- **Major updates:** planned project, not a drive-by merge

### 19.4 Removing a Dependency

When replacing or removing:
- Verify no other code depends on it
- Remove from all lockfiles
- Remove from Docker images
- Remove from documentation
- Verify binary size/install time improved

---

## 20. Forbidden Practices

The following are forbidden. Not discouraged — forbidden.

### 20.1 Forbidden in Code

- **`eval()` or equivalent dynamic code execution**
- **Shell injection patterns:** `os.system(user_input)`, `exec.Command("sh", "-c", user_input)`
- **Hard-coded credentials or secrets**
- **Disabling security features "temporarily"** without a tracking ticket
- **Catching exceptions to hide them** — `except: pass` without reason
- **Commented-out code** — delete it, Git remembers
- **TODO comments without owner and deadline**
- **Copy-pasted code** — extract a function instead

### 20.2 Forbidden in Git

- **Force-push to main or production branches**
- **Committing secrets** (enforced by pre-commit hooks + CI)
- **Committing large binary files** — use Git LFS or R2
- **Merging without review** (except single-person hotfixes with post-hoc review)
- **Amending pushed commits**
- **Rebasing shared branches** after others have pulled

### 20.3 Forbidden in Deployment

- **Deploying untested code**
- **Deploying on Fridays without cause**
- **Deploying without rollback plan**
- **Manual production database edits** (except documented incident response)
- **`:latest` Docker tags in production**
- **Bypassing CI to deploy faster**

### 20.4 Forbidden in Operations

- **Accessing production customer data** without a documented support ticket
- **Sharing credentials** between team members
- **Storing credentials in unencrypted notes, emails, or chat**
- **Running destructive commands** without confirmation
- **Disabling monitoring** for any reason

### 20.5 Forbidden in Customer Interactions

- **Over-promising features** — say what we have, not what we wish
- **Lying about incidents** — transparency even when painful
- **Hiding problems** that affect the customer
- **Using dark patterns** in signup, billing, or cancellation
- **Spamming** — every email has a clear unsubscribe

---

## Enforcement

### How This Constitution Is Enforced

1. **Documentation review:** This constitution is reviewed quarterly with the full team
2. **Violation response:**
   - Accidental violation → fix + document in team retrospective
   - Pattern of violations → discussion with founder
   - Deliberate violation after warning → grounds for termination
3. **External violations:** Partners/contractors who violate this constitution lose access
4. **Public accountability:** Major violations affecting customers disclosed publicly

### Final Authority

The founder (Mahmoud Hassan) is the final interpreter of this constitution. In their absence, the most senior team member. If still unresolved, external legal counsel.

---

## Acknowledgment

By contributing to ShieldScan (writing code, making decisions, interacting with customers), you acknowledge you have read and agreed to this constitution.

**Claude Code specifically:** Every session, confirm you've read CONSTITUTION.md and CLAUDE.md. If a request conflicts with this constitution, decline and escalate.

---

*End of Constitution. This document's authority is absolute within the ShieldScan project.*

*Last amended: 2026-04-20 (initial version)*
*Next review: 2026-07-20*
