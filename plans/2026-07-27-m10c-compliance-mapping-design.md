# M10.C Compliance Mapping — Stage 1 Lock-Record (light Mode-2 compressed per DQ4)

**Status:** 🔒 **CLOSED (2026-07-31)** — full M10.C lifecycle landed, all γ: Stage 1 this doc `bee52c6` + C1 `edea03a` (3 global models + seed migration `a7c3e9f04b21` + frameworks.py inline seed + `seed_compliance_data()` helper + not-in-TENANT_TABLES ∅ test) + C2 `40b8243` (three-state posture mapper + 3 org-scoped GETs; suite 828 ZERO regressions). Stage 4 P5.A 3-commit chain (docs this commit; api DRIFT-LOG + engine cross-ref forthcoming). **NO C0** — no ADR (data-model + seed over existing patterns, like M10.A; ADR-034 was reserved for M10.B only). But unlike M10.A this **HAD a migration** (schema + a first-of-kind data/seed migration), so *C0-less ≠ migration-less* here. Third M10 sub-milestone; 66 cumulative drift preserved (0 catalogued); compliance mapping reachable end-to-end.

**Date:** 2026-07-27.

**Authority:** M10 decomposition `41ad3e6` (M10.C = Task 10.5; compressed shape per DQ4); M10.C entry probe (3 KEY findings: the 3-vs-2-table scope-delta resolved in SPEC-canon's favor; GLOBAL reference data per SPEC line 345 — no TenantMixin/RLS, `models/identity.py` precedent; the seed data-migration is first-of-kind, no `op.bulk_insert` precedent in the repo); M10.C Mode-2 brainstorming CLOSED (2 gates / ~9 locks); alembic head `f1a2b3c4d5e6` (M10.B reports migration = the C1 parent). 66 cumulative drift discipline; 12-instance averted-prediction lineage; 2-instance ADR-mislabel lesson; 3-instance date-stamp lesson.

---

## 1. Gate-Lock Table (2 gates / ~9 locks)

### Gate 1 — Data Model + Seed
- **G1-L1 THREE TABLES (scope-delta resolved):** `compliance_frameworks` → `compliance_controls` → `cwe_control_mappings` per SPEC §DB canon (lines 331-333). The plan-literal / decomposition "2-table" naming omits `compliance_controls`; resolved in canon's favor (echoing M10.A 7-vs-4 + M10.B 5-vs-4). Normalized so controls are first-class rows the frameworks endpoint enumerates independently of mappings.
- **G1-L2 GLOBAL REFERENCE DATA:** per SPEC line 345 these 3 tables are explicitly enumerated as **NOT tenant-scoped** (alongside `plans` + `marketplace_templates`). `Base + TimestampMixin`, **NO TenantMixin, NO RLS, NOT in TENANT_TABLES** — the deliberate **inverse of the M10.B Catch-1 dual-mechanism**; C1 asserts the mirror-image *not-in-TENANT_TABLES* check. Precedent: `models/identity.py` (Organization/User/Membership).
- **G1-L3 SEED — inline + curated:** SOC2 CC-series + ISO 27001:2013 Annex A controls + a curated CWE→control mapping subset (~15-40; representative, not exhaustive) authored inline in `services/compliance/frameworks.py`; a data migration bulk-inserts it (`op.bulk_insert`, first-of-kind).
- **G1-L4 ISO 27001:2013:** matches the acceptance test's `A.14.2.5` Annex A control; the :2022 renumber is forward-pinned. PCI omitted at MVP (Enterprise-tier).

### Gate 2 — Posture + Endpoints + Tier
- **G2-L5 THREE-STATE POSTURE:** per control → `pass` (mapped, no matching finding) / `fail` (≥1 mapped finding) / `not_assessed` (no CWE mapping exists for that control). Unmappable controls are **never overclaimed as pass** — honesty for a compliance artifact.
- **G2-L6 SCAN-PRIMITIVE + ORG-AGGREGATE:** per-scan × framework is the primitive; org-wide posture is the worst-state-per-control aggregate across the org's scans.
- **G2-L7 THREE GETs:** `GET …/compliance/frameworks` (list) · `GET …/scans/{scan_id}/compliance/{fw}` (per-scan; framework-slug 404 + `_load_terminal_scan` 409 reuse from M10.B) · `GET …/compliance/posture` (org-wide). All org-scoped; **no audit rows** (V-JJH — GETs).
- **G2-L8 UNGATED + TIER→BILLING:** all ship ungated; tier-enforcement forward-pinned to billing (**3rd instance** after M10.A /fix + M10.B report endpoints).

---

## 2. Architecture

Three **global reference tables** (`frameworks → controls → cwe_control_mappings`) hold industry-standard compliance data shared across all orgs. Findings map to controls via the **CWE string** on `Vulnerability.cwe_id` (`String(20)`, indexed, `"CWE-89"`). Posture is three-state (pass/fail/not_assessed) so the artifact never overclaims coverage it doesn't have.

**GLOBAL-not-tenant (the deliberate inverse of M10.B Catch-1):** M10.B's reports table had to be *added* to `TENANT_TABLES` (or its cross-tenant test would false-pass). These compliance tables must be *kept OUT* of `TENANT_TABLES` (they are global per SPEC line 345). C1 asserts the mirror-image check: `{"compliance_frameworks","compliance_controls","cwe_control_mappings"} ∩ TENANT_TABLES == ∅`.

---

## 3. Table Shapes (all `Base + TimestampMixin`, NO TenantMixin, NO RLS)

```
compliance_frameworks: id (pk) · slug (unique: soc2|iso27001) · name · version (ISO=2013)
compliance_controls:   id (pk) · framework_id FK→compliance_frameworks · control_id (CC6.1|A.14.2.5) · title · description
cwe_control_mappings:  id (pk) · cwe_id (str, "CWE-89") · control_id FK→compliance_controls
```

FK-create order in the migration: frameworks → controls → mappings (parent `f1a2b3c4d5e6`).

---

## 4. Seed

Inline in `services/compliance/frameworks.py`: SOC2 (CC-series) + ISO 27001:2013 Annex A controls + a curated CWE subset covering scanner-emitted CWEs (89 SQLi, 79 XSS, 78 cmd-injection, 22 path-traversal, 287 auth, 311/312 crypto, …). Loaded via an `op.bulk_insert` **data migration** (first-of-kind — no seed-migration precedent in the repo). PCI **OMITTED** at MVP. Representative, not exhaustive (exhaustive → post-launch expansion).

---

## 5. Posture (`mapper.py`)

Per (scan × framework): for each control in the framework → gather the CWEs mapped to it → check whether any of the scan's vulnerabilities carry a mapped CWE → `fail` if yes, `pass` if the control has mappings but no matching finding, `not_assessed` if the control has no CWE mapping at all. Org-wide posture = worst-state-per-control across the org's scans.

---

## 6. Endpoints (3 org-scoped GETs; ungated; no audit per V-JJH)

- `GET /orgs/{org_id}/compliance/frameworks` — list seeded frameworks (+ control counts).
- `GET /orgs/{org_id}/scans/{scan_id}/compliance/{fw}` — per-scan posture for a framework (slug → 404 if unknown; `_load_terminal_scan` → 404/409 reuse from M10.B).
- `GET /orgs/{org_id}/compliance/posture` — org-wide posture aggregate.

---

## 7. Stage 3 (no-C0; compressed)

- **C1** — 3 global models (no TenantMixin/RLS) + schema migration + `frameworks.py` inline seed + data migration (`op.bulk_insert`) + model/seed tests + the **not-in-TENANT_TABLES** assertion.
- **C2** — `mapper.py` posture logic + `routes/compliance.py` (3 endpoints) + registration + route tests.
- Then **P5.A ×3** (docs annotations + api DRIFT-LOG + engine cross-ref).

---

## 8. Forward-Pins

- Tier-gating → billing milestone (3rd instance: /fix + report-endpoints + compliance).
- PCI framework seed → Enterprise tier.
- Exhaustive control / CWE coverage → post-launch expansion.
- ISO 27001:2022 renumber → future migration.
- $49 compliance-PDF add-on (external audit format) → M10.B report territory.

---

## 9. V-KK Pre-C1 DEFERRED-EMPIRICAL Grounding List

- **V-KKA:** `models/identity.py` global-model shape (`Base + TimestampMixin`, no TenantMixin) to mirror for the 3 compliance models.
- **V-KKB:** `op.bulk_insert` idiom (first-of-kind — ground on the alembic op API, no repo precedent) + how the data migration references the inline `frameworks.py` seed (import at migration-author-time vs an inline literal in the migration — decide; a migration importing app code is fragile across schema evolution, lean toward an inline literal or a frozen copy).
- **V-KKC:** FK-ordering in `create_table` (frameworks → controls → mappings; `controls.framework_id` FK; `mappings.control_id` FK).
- **V-KKD:** `Vulnerability.cwe_id` join for posture (`String(20)`, indexed, `"CWE-89"` format; null cwe_id → unmapped).
- **V-KKE:** the not-in-TENANT_TABLES verification (assert the 3 tables absent from `app.db.policies.TENANT_TABLES` — the mirror-image of M10.B Catch-1).
- **V-KKF:** seed idempotency / re-run (does the data migration's `downgrade` delete the seed; does `upgrade` guard against double-insert on re-run).
