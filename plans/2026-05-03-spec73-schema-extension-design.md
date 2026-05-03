# SPEC §7.3 RawFinding Schema Extension — Design Doc

**Date:** 2026-05-03
**Author:** Mahmoud Hassan (Odyssey Technology) with Claude
**Status:** Approved design; pending implementation
**Related:** M6-close-followup task; ADR-024 candidacy
**Cross-references:** ADR-013 (Python sole writer), ADR-017 (findings inline in
job_completed events), ADR-022 (recon-as-pre-scan-helpers), ADR-023 (NativeRunner
OutputFile mode), SPEC §7.3 (Job Completion wire format), SPEC §8 (AI Analysis
Pipeline)

---

## 0. Executive Summary

Extend the `RawFinding` schema (SPEC §7.3) with **4 new fields** —
`References`, `Tags`, `CVSSVector`, `AdditionalCWEs` — to address
the schema-extension trigger that fired at M6.4 and accumulated through
M6 close. Concurrent updates to Engine Go struct, Python SQLAlchemy
model, Python Pydantic schema, Alembic migration, and 6-tool parser
retrofits.

Single architectural artifact: **ADR-024**.

Cross-repo coordination via **3 commits** (Approach A): Docs + Python
+ Engine.

Estimated total work: **6.5–7 hours**.

The 4 fields rescue **~12 of 38 folded/dropped fields (~32%)** across
M6 tools. The remaining ~26 folds are tool-specific metadata; their
rescue requires Evidence map (over-design at this scope) or per-tool
subfields (over-engineering). The 4-field scope captures the highest-
value architectural-pattern wins, including one **load-bearing M9
forward-pin** for §8.2 correlation algorithm.

---

## 1. Brainstorming Output (Locked Decisions)

The brainstorming session that produced this design locked six
decisions:

1. **Forward momentum** orientation (vs operational readiness or
   architectural cleanup).
2. **SPEC §7.3 followup first** (before M7 kickoff).
3. **Moderate scope:** 4 fields (References, Tags, CVSSVector,
   AdditionalCWEs).
4. **Bandwidth confirmed:** ~6–7h available for the followup task.
5. **M9 planning imminent:** forward-pin text grounded in SPEC §8
   reading.
6. **Coupling depth: Option 2 (light forward-pin).** No DB indexes.
   No concrete M9 hooks. ADR-024 documents M9 intent per field;
   M9 implementation later inherits informed assumptions.

The genuinely load-bearing insight from the brainstorming SPEC §8
read: **AdditionalCWEs requires §8.2 correlation algorithm extension.**
Without that forward-pin, M9 implementation might miss the multi-CWE
correlation upgrade (especially relevant for Dep-Check's multi-CWE
findings).

---

## 2. Problem Statement

### 2.1 Trigger Fire Status

The reductions counter — tracking M6 tools whose parsers fold or drop
fields from upstream tool output — fired its trigger at **M6.4**
(SSLyze) when 4 of 5 implemented tools showed reductions. Per the
landscape decision at 6.4, schema extension was deferred to M6 close
(Path A).

**Counter at M6 close (post-6.8):**

| Tool | Folded/Dropped Fields | Reduction Count |
|---|---|---|
| Nuclei | CVE folded, CVSSVector dropped, References dropped | 3 |
| Semgrep | (none) | 0 |
| Gitleaks | commit-metadata folded; entropy/columns/endline/tags dropped | 5 |
| SSLyze | cipher details folded; cert chain dropped; path_validation folded; ocsp dropped; plugin metadata dropped | 5–6 |
| Dep-Check | references, vulnerableSoftware, hashes, evidenceCollected, multi-CWE dropped | 5 |
| Checkov | bc_check_id, guideline, evaluations, caller_file_*, entity_tags, code_block 2-D folded | 6 |
| Nikto | osvdbid/osvdblink, namelink, iplink dropped | 3 |
| Wapiti | curl_command, referer-when-empty, wstg refs, classifications metadata folded, http_request truncation | 5 |
| CORStest | Resource/Origin folded, ACAO/ACAC folded | 2 |
| **Total** | | **~38 folded/dropped fields across 8 of 9 tools (89%)** |

### 2.2 The Architectural Decision

The reductions are not problems in themselves — folding is a
legitimate parser strategy when the canonical schema lacks
appropriate slots. The decision is **whether the canonical schema
should grow new slots**, which categorical patterns warrant
first-class fields, and how those fields couple to the M9 AI
pipeline that consumes RawFindings.

### 2.3 Scope Discipline

Three scope options were considered:

- **Conservative (2 fields: References + Tags):** Rescues ~7 of 38
  folds (~18%). Defers CVSSVector + AdditionalCWEs to M7+ data
  point. Cheaper now (~3.5h) but accumulates incremental schema
  decisions over multiple tasks.
- **Moderate (4 fields):** Rescues ~12 of 38 folds (~32%). Captures
  pattern wins; includes load-bearing AdditionalCWEs forward-pin.
  ~6.5–7h work.
- **Comprehensive (6 fields including Evidence map):** Rescues
  ~30 of 38 folds (~80%). Adds flexible-map field with
  anti-pattern risk (dumping ground for inconsistent data).
  ~10h work.

**Selected: Moderate.** The 4-field scope captures the
architectural-pattern wins; comprehensive scope's Evidence map is
genuinely over-design (per the brainstorming asymmetric-cost
analysis); conservative scope defers decisions that benefit from
the M6 9-tool data we already have.

### 2.4 Honest Rescue Accounting

Field-by-field, the 4-field scope rescues:

| Tool | References | Tags | CVSSVector | AdditionalCWEs | Tool Rescue |
|---|---|---|---|---|---|
| Nuclei | `info.reference[]` | `info.tags[]` | `info.classification.cvss-metrics` | — | 3 |
| Semgrep | `extra.metadata.references` | `extra.metadata.category` | — | — | 0–2 (metadata-dependent) |
| Gitleaks | — | `tags[]` | — | — | 1 |
| SSLyze | — | — | — | — | **0** |
| Dep-Check | `references[]` | — | CVSS3 composition | multi-CWE intersection | 3 |
| Checkov | `guideline` (wrapped) | — | — | — | 1 |
| Nikto | — | — | — | — | **0** |
| Wapiti | `wstg[]` | `module` (wrapped) | — | — | 2 |
| CORStest | — | — | — | — | **0** |
| **Total** | **6 tools** | **5 tools** | **2 tools** | **1 tool** | **~12** |

**Three tools get zero retrofit at this scope:** SSLyze, Nikto,
CORStest. Their reductions stay folded. A future schema-extension
task may address tool-specific metadata; not in scope here.

**~32% of folds rescued.** This is the honest number. Earlier
brainstorming-time estimates of 66% were inaccurate; the actual rescue
rate reflects that the 4 fields capture categorical patterns
(References, multi-CWE, CVSSVector) but don't address tool-specific
minor metadata.

### 2.5 Trigger Status Post-§7.3

Post-implementation, the reductions counter remains at **8/9 tools
(89%) with reductions** — the 4-field extension reduces *count* of
folded fields per tool but doesn't eliminate any tool's folds to
zero. The trigger remains fired.

ADR-024 explicitly acknowledges this: SPEC §7.3 followup is **one
incremental step** in addressing the schema-extension trigger; future
ADRs (Evidence map? per-tool subfields?) may extend further when
M7+ data informs which patterns warrant additional first-class
fields.

---

## 3. Solution Architecture

### 3.1 The Four Fields

#### 3.1.1 References `[]string`

Array of URLs / advisory identifiers. Free-form strings (no
structure). Examples:
- CVE links: `"https://nvd.nist.gov/vuln/detail/CVE-2024-1234"`
- Advisory URLs: `"https://example.com/security/advisory-2024-001"`
- OWASP WSTG identifiers: `"OSHP-X-Frame-Options"` (Wapiti)
- Remediation guidelines: `"https://docs.bridgecrew.io/docs/CKV_AWS_8"` (Checkov `guideline`)

**Type:** `[]string` with `omitempty`. No URL validation in schema;
strings as-emitted by tools.

**M9 forward-pin:** SPEC §8 algorithms do not directly use References.
Reserved for:
- M11 dashboard "linked findings" UI feature
- AI fix-generation prompt context (helping Claude Sonnet
  ground fixes in cited remediation guidance)
- Future cross-tool deduplication signal (findings citing
  the same CVE may be deduplication candidates beyond
  vector similarity)

#### 3.1.2 Tags `[]string`

Array of fine-grained categorical tags. Distinct from existing
`engine_category` (broad: dast/sast/sca/etc., 11 enum values).
Examples:
- Nuclei: `["xss", "owasp-top-10", "automated"]`
- Gitleaks: `["aws-access-key", "production-leak"]`
- Wapiti: `["http_headers"]` (module name wrapped)
- Semgrep: `["security"]` (metadata category)

**Type:** `[]string` with `omitempty`. Free-form; no canonical
normalization at SPEC §7.3 time. Normalization is M9/M11
implementation concern when query patterns surface.

**M9 forward-pin:** SPEC §8 algorithms do not directly use Tags.
Reserved for:
- M11 dashboard fast filtering (e.g., "show me all findings tagged
  `xss` across the scan")
- Future M9 AI categorization input (LLM-based finding
  classification could use Tags as feature)
- Cross-tool grouping (e.g., "all DAST findings tagged
  `owasp-top-10`")

**Important constraint:** Tags must NOT duplicate
`engine_category`. If a parser emits Tags that match
engine_category values, the parser should drop them (Tags is for
finer-grained sub-categorization, not category re-statement).

#### 3.1.3 CVSSVector `string`

Canonical CVSS 3.1 vector string. Format:
`"CVSS:3.1/AV:N/AC:L/PR:N/UI:N/S:U/C:H/I:H/A:H"`.

**Type:** `string` with `omitempty`. Stored as-emitted by tools
(no parsing into 8-dimension subfields at SPEC §7.3 time).

**M9 forward-pin:** SPEC §8.3 currently uses derived `base_cvss`
numeric for scoring (multiplicative formula with corroboration +
exploitability multipliers). CVSSVector is **reserved for future
exploitability_multiplier derivation**.

Specifically: the §8.3 `exploitability_multiplier` value `1.5`
("publicly accessible + unauthenticated") could be derived from
CVSSVector AV (Attack Vector) field — `AV:N` (network, with
`PR:N`/`UI:N` for unauthenticated/no-user-interaction) maps to
the highest-exploitability bucket without separate detection
logic. M9 implementation may evolve §8.3 to parse CVSSVector
when available; current spec leaves this as future enhancement.

**Trigger to revisit M9 integration:** §8.3 algorithm
implementation. If §8.3 lands without CVSSVector parsing,
Tags-based exploitability detection is the fallback.

#### 3.1.4 AdditionalCWEs `[]string`

Array of CWE identifier strings beyond the primary `CWEID`
field. Format: `["CWE-89", "CWE-20"]`.

**Type:** `[]string` with `omitempty`. Empty (or absent) when
finding has only one CWE; populated when tool reports multi-CWE.

**M9 forward-pin (LOAD-BEARING):** SPEC §8.2 correlation
algorithm currently uses `cwe_id` singular for `cwe_exact` (0.40)
and `cwe_parent` (0.25) matches:

```python
if dast.cwe_id == sast.cwe_id:
    score += WEIGHTS["cwe_exact"]
elif is_cwe_parent_child(dast.cwe_id, sast.cwe_id):
    score += WEIGHTS["cwe_parent"]
```

**M9 implementation MUST extend these checks to consider
intersection with `additional_cwes`:**

```python
# Extended logic (M9 implementation):
all_dast_cwes = {dast.cwe_id} | set(dast.additional_cwes or [])
all_sast_cwes = {sast.cwe_id} | set(sast.additional_cwes or [])

if all_dast_cwes & all_sast_cwes:
    score += WEIGHTS["cwe_exact"]
elif any_parent_child(all_dast_cwes, all_sast_cwes):
    score += WEIGHTS["cwe_parent"]
```

This is the **load-bearing forward-pin** of the entire SPEC §7.3
extension. Without it, M9 correlation misses multi-CWE matches
(especially Dep-Check, which routinely emits 2–4 CWEs per CVE).

**ADR-024 must document this explicitly with code-shaped
forward-pin** so M9 implementation finds it.

### 3.2 What's Deliberately Excluded

**Evidence map (`map[string]string`):** Anti-pattern risk.
Flexible-map fields tend to accumulate inconsistent usage across
tools; one tool stores `"hash": "abc"`, another stores
`"sha256": "abc"`, M9 has to handle both. Better to add specific
fields when patterns warrant, not preempt with a dumping ground.

**Structured Reference type (`[]Reference{URL, Type}`):** Premature.
SPEC §8 doesn't differentiate reference types (CVE vs advisory vs
docs). Adding type metadata before M9 needs it is over-design.
Future: if M11 dashboard needs to render different reference types
differently, extend then.

**CVSSVector parsed into subfields:** Premature. SPEC §8.3 doesn't
currently parse the vector. Storing as canonical string preserves
all information and defers parsing to M9 implementation when (or
if) needed.

**Per-tool metadata fields (e.g., NucleiTemplateID, GitleaksRule):**
Tool-specific concerns; would proliferate the schema. The
remaining ~26 folded fields are tool-specific metadata; their
rescue (if warranted) is a future schema extension.

**DB indexes on the new fields:** Premature optimization. SPEC §8
algorithms don't query `raw_findings` by References/Tags/
CVSSVector/AdditionalCWEs; they iterate findings already loaded
for a scan. Indexes are M9/M11 concerns when query patterns
surface.

---

## 4. Per-Tool Retrofit Checklist

### 4.1 Tools Receiving Retrofits (6)

#### 4.1.1 Nuclei (M6.1)

**Source data:** Nuclei JSONL output already parsed.

**Retrofit:**
- `References`: Populate from `info.reference[]` (currently
  dropped per 6.1 reduction). If absent, leave nil.
- `Tags`: Populate from `info.tags[]` (currently folded into
  Description per 6.1). Drop tags that duplicate `engine_category`
  values (e.g., drop `dast` if appears in tags).
- `CVSSVector`: Populate from `info.classification.cvss-metrics`
  (currently dropped per 6.1). Format is already canonical CVSS
  string per Nuclei convention.

**Tests:** Update `nuclei_test.go` parse tests; add fixtures
covering findings with full CVSS vectors and reference arrays.

**M6 reductions update:** Nuclei reductions count drops 3 → 0.

#### 4.1.2 Semgrep (M6.2)

**Source data:** Semgrep JSON `extra.metadata` (when present).

**Retrofit:**
- `References`: Populate from `extra.metadata.references`
  (when present). Often absent for ad-hoc rules; nil is fine.
- `Tags`: Populate from `extra.metadata.category` (single
  value, wrap as `[]string{category}`). Drop if matches
  engine_category.

**Tests:** Update `semgrep_test.go` fixtures with metadata-rich
rules.

**M6 reductions update:** Semgrep stays at 0 (no reductions
existed pre-§7.3).

#### 4.1.3 Gitleaks (M6.5)

**Source data:** Gitleaks JSON array.

**Retrofit:**
- `Tags`: Populate from `tags[]` (currently dropped per 6.5).

**Constants-only (Pattern 4) preserved:** Gitleaks still uses
constant `SeverityCritical` + `CWEHardcodedCredentials`. Tags
is per-finding.

**Tests:** Update `gitleaks_test.go` fixtures to cover tags
extraction.

**M6 reductions update:** Gitleaks reductions count drops 5 → 4
(commit-metadata fold + entropy/columns/endline still folded;
only `tags` rescued).

#### 4.1.4 Dep-Check (M6.7)

**Source data:** Dep-Check JSON file output (ADR-023 OutputFile
mode); per-CVE dependency info.

**Retrofit:**
- `References`: Populate from `references[]` (currently dropped
  per 6.7).
- `CVSSVector`: Compose from `cvssV3` subfields:
  ```go
  vector := fmt.Sprintf(
      "CVSS:3.1/AV:%s/AC:%s/PR:%s/UI:%s/S:%s/C:%s/I:%s/A:%s",
      cvssV3.AttackVector, cvssV3.AttackComplexity,
      cvssV3.PrivilegesRequired, cvssV3.UserInteraction,
      cvssV3.Scope, cvssV3.ConfidentialityImpact,
      cvssV3.IntegrityImpact, cvssV3.AvailabilityImpact,
  )
  ```
  **Caveat:** Dep-Check emits attack-vector values as full words
  (`"NETWORK"`); CVSS canonical uses single-letter codes (`"N"`).
  Mapping table required.
- `AdditionalCWEs`: Populate from CWE intersection (currently
  dropped per 6.7's "multi-CWE dropped"). Primary CWE → `CWEID`;
  remaining CWEs → `AdditionalCWEs`.

**Tests:** Update `depcheck_test.go` fixtures to cover multi-CWE
findings + CVSS vector composition. Add table-driven test for
attack-vector value mapping.

**M6 reductions update:** Dep-Check reductions count drops 5 → 2
(vulnerableSoftware, hashes, evidenceCollected still dropped).

#### 4.1.5 Checkov (M6.7)

**Source data:** Checkov JSON.

**Retrofit:**
- `References`: Populate from `guideline` URL (single value,
  wrap as `[]string{guideline}` if non-empty; nil otherwise).

**Constants-only (Pattern 4) preserved:** Checkov still uses
constant `SeverityMedium` + `CWEIaCMisconfiguration`. References
is per-finding.

**Tests:** Update `checkov_test.go` fixtures with `guideline`
populated and absent.

**M6 reductions update:** Checkov reductions count drops 6 → 5
(bc_check_id, evaluations, caller_file_*, entity_tags, code_block
2-D fold still in place; `guideline` rescued).

#### 4.1.6 Wapiti (M6.6)

**Source data:** Wapiti JSON via OutputFile mode (ADR-023).

**Retrofit:**
- `References`: Populate from `wstg[]` array (currently dropped
  per 6.6).
- `Tags`: Populate from `module` field (single value, wrap as
  `[]string{module}`). Drop if matches engine_category.

**Tests:** Update `wapiti_test.go` fixtures.

**M6 reductions update:** Wapiti reductions count drops 5 → 3
(curl_command, referer-when-empty, classifications metadata fold,
http_request truncation still in place; `wstg` + `module` rescued).

### 4.2 Tools NOT Receiving Retrofits (3)

#### 4.2.1 SSLyze (M6.4)

**Why no retrofit:** SSLyze findings emerge from plugin-rules
parser pattern; per-finding output doesn't carry References,
Tags, CVSSVector, or multi-CWE. Plugin rules synthesize findings
from structured TLS diagnostics — there's no upstream metadata
to rescue.

**Reductions stay at 5–6.**

#### 4.2.2 Nikto (M6.6)

**Why no retrofit:** Nikto XML emits `<item>` records with
description/uri/method only. No references, tags, CVSS, or CWE
data. The folded fields (osvdbid/osvdblink — defunct since 2016;
namelink, iplink) are not rescuable into the new schema fields.

**Reductions stay at 3.**

#### 4.2.3 CORStest (M6.6)

**Why no retrofit:** CORStest text-with-ANSI parser extracts
URL/origin/header values only. No metadata to rescue.

**Reductions stay at 2.**

### 4.3 Total Retrofit Effort Estimate

| Tool | Effort | New Fixtures |
|---|---|---|
| Nuclei | ~20 min | 1 (vector + refs + tags) |
| Semgrep | ~10 min | 1 (metadata-rich rule) |
| Gitleaks | ~10 min | 1 (tags-populated) |
| Dep-Check | ~30 min (vector composition) | 2 (multi-CWE + vector) |
| Checkov | ~10 min | 1 (guideline-populated) |
| Wapiti | ~15 min | 1 (wstg + module) |
| **Total** | **~95 min** | **7 new fixtures** |

Plus per-tool test updates (~10 min each) = **~155 min total
parser retrofit work**, slightly under brainstorming estimate of
~1.5h.

---

## 5. Cross-Repo Coordination Plan (Approach A)

### 5.1 Three Commits

**Commit 1 (shieldscan-docs):** ADR-024 + SPEC §7.3 schema doc
update + DRIFT-LOG entries

**Commit 2 (shieldscan-api):** Python SQLAlchemy model + Pydantic
schema + Alembic migration + ingest tests + fixture updates +
CompletionsConsumer test extensions

**Commit 3 (shieldscan-engine):** Engine RawFinding struct
extension + 6-tool parser retrofits + fixture updates + tests +
DRIFT-LOG entries + reductions counter update

### 5.2 Order of Commit Landing

**Strict ordering matters** because of `DisallowUnknownFields`
invariant:

1. **First:** Docs commit. ADR-024 + SPEC §7.3 update lands as
   the shared schema doc per ADR-017 reference. No code impact
   yet; documentation primes the implementation.

2. **Second:** Python commit. SQLAlchemy/Pydantic accept the new
   fields (with `Optional[...]` so existing engine emissions
   without these fields still validate). Alembic migration runs.

3. **Third:** Engine commit. Now Engine can emit findings with
   the new fields; Python consumer accepts them. Per-tool retrofit
   populates the fields where data exists.

**Why this order:** If Engine ships first, Python's
`DisallowUnknownFields` rejects events with unknown fields (well —
Python uses Pydantic strict validation, similar effect). Schema
must be additive on Python side first.

**The reverse failure mode:** If Python ships first with new
fields required, Engine emissions without them fail validation.
Mitigation: all 4 fields are `Optional` on Python side, so
empty/absent is valid.

### 5.3 Per-Commit Detailed Breakdown

#### Commit 1: shieldscan-docs

**Files:**
- `SPECIFICATION.md` — §7.3 update; §13 ADR-024 added; §14 glossary
  potentially updated for new fields
- `DRIFT-LOG.md` — followup task acknowledgment + reductions
  counter post-§7.3 status entry

**Effort:** ~1 hour (mostly ADR-024 drafting)

**Commit message shape:**
```
docs(spec): add ADR-024 RawFinding schema extension + SPEC §7.3 update

4 new optional fields: References, Tags, CVSSVector, AdditionalCWEs.
Addresses M6 schema-extension trigger (8/9 tools with reductions).
Light M9 forward-pin coupling: AdditionalCWEs is load-bearing for §8.2
correlation algorithm; CVSSVector reserved for §8.3 future-derivation;
References + Tags reserved for M11 + future AI categorization.

ADR-024 details rationale; per-tool retrofit deferred to engine commit.
```

#### Commit 2: shieldscan-api

**Files:**
- `src/app/models/raw_findings.py` — SQLAlchemy model extension
- `src/app/schemas/raw_findings.py` — Pydantic schema extension
- `alembic/versions/XXXX_spec_73_extension.py` — migration (additive;
  reversible)
- `tests/services/test_completions_consumer.py` — ingest tests covering
  new fields
- `tests/fixtures/job_completed_*.json` — fixtures with new fields populated

**Effort:** ~2.5 hours (cross-repo: ~2h + tests: ~30min)

**Migration shape:**
```python
def upgrade():
    op.add_column('raw_findings', sa.Column('references', sa.ARRAY(sa.String()), nullable=True))
    op.add_column('raw_findings', sa.Column('tags', sa.ARRAY(sa.String()), nullable=True))
    op.add_column('raw_findings', sa.Column('cvss_vector', sa.String(255), nullable=True))
    op.add_column('raw_findings', sa.Column('additional_cwes', sa.ARRAY(sa.String()), nullable=True))

def downgrade():
    op.drop_column('raw_findings', 'additional_cwes')
    op.drop_column('raw_findings', 'cvss_vector')
    op.drop_column('raw_findings', 'tags')
    op.drop_column('raw_findings', 'references')
```

**Pydantic schema extension:**
```python
class RawFindingCreate(BaseModel):
    # ... existing fields ...
    references: list[str] | None = None
    tags: list[str] | None = None
    cvss_vector: str | None = None
    additional_cwes: list[str] | None = None
```

**Commit message shape:**
```
feat(api): SPEC §7.3 schema extension — 4 new RawFinding fields

References, Tags, CVSSVector, AdditionalCWEs added to SQLAlchemy model
+ Pydantic ingest schema + Alembic migration. All Optional;
backward-compatible with existing engine emissions.

CompletionsConsumer ingest tests verify new fields persist correctly;
existing tests remain green.

Per ADR-024.
```

#### Commit 3: shieldscan-engine

**Files:**
- `internal/events/events.go` — RawFinding struct extension
- `internal/events/events_test.go` — wire-format roundtrip tests
- `internal/events/testdata/job_completed_python_v1.json` — fixture update
- `internal/tools/nuclei/parse.go` + `nuclei_test.go` + testdata — retrofit
- `internal/tools/semgrep/parse.go` + `semgrep_test.go` + testdata — retrofit
- `internal/tools/gitleaks/parse.go` + `gitleaks_test.go` + testdata — retrofit
- `internal/tools/depcheck/parse.go` + `depcheck_test.go` + testdata + cvss_mapping.go — retrofit
- `internal/tools/checkov/parse.go` + `checkov_test.go` + testdata — retrofit
- `internal/tools/wapiti/parse.go` + `wapiti_test.go` + testdata — retrofit
- `DRIFT-LOG.md` — followup task entries

**Effort:** ~3 hours (struct extension: ~5min + 6-tool retrofits: ~155min
+ DRIFT entries: ~30min)

**Struct extension shape:**
```go
// internal/events/events.go
type RawFinding struct {
    // ... existing fields ...
    References     []string `json:"references,omitempty"`
    Tags           []string `json:"tags,omitempty"`
    CVSSVector     string   `json:"cvss_vector,omitempty"`
    AdditionalCWEs []string `json:"additional_cwes,omitempty"`
}
```

**Commit message shape:**
```
feat(engine): SPEC §7.3 schema extension — 4 new RawFinding fields + 6-tool retrofit

References, Tags, CVSSVector, AdditionalCWEs added to RawFinding struct.
6 tools retrofitted to populate new fields where source data exists:
- Nuclei: References, Tags, CVSSVector
- Semgrep: References, Tags (metadata-dependent)
- Gitleaks: Tags
- Dep-Check: References, CVSSVector (composed from cvssV3), AdditionalCWEs
- Checkov: References (from guideline)
- Wapiti: References (wstg), Tags (module)

Tools without applicable source data: SSLyze, Nikto, CORStest (no retrofit).

M6 reductions counter post-§7.3: ~26 folds remaining (down from ~38).
SPEC §7.3 trigger remains fired (8/9 tools still have reductions);
future incremental schema extensions may address tool-specific metadata.

Per ADR-024.
```

### 5.4 Test Strategy

**Wire-format roundtrip:**
- Engine emits `job_completed` event with new fields populated
- Python CompletionsConsumer ingests event
- Pydantic accepts all 4 new fields as Optional
- SQLAlchemy persists to PostgreSQL
- Query returns event with new fields preserved

**Per-tool retrofit:**
- New fixtures cover findings with new fields populated
- Existing fixtures (without new fields) still parse correctly
  (backward-compat verification)
- Test assertions verify field-population logic
  (e.g., Nuclei `info.reference[]` → `RawFinding.References`)

**Backward-compat:**
- Existing fixtures from M6.1–M6.8 must still load and parse without
  the new fields (omitempty handles this on Engine side; Optional handles
  on Python side)
- Existing tests must remain green without modification

**Cross-repo verification:**
- Engine fixtures with new fields populated → Python consumer accepts
- Python ingest with new fields → DB persists → query returns correctly

### 5.5 Rollback Plan

If implementation surfaces unexpected issues:

**Cross-repo state at any point:**
- Docs commit landed only: no code impact; revert ADR-024
- Docs + Python commits landed: Python accepts new fields (Optional); Engine
  doesn't emit them; system functional with empty new fields
- All 3 commits landed: full state; rollback requires reverting all 3
  in reverse order (Engine → Python → Docs)

**Critical:** Don't land Engine without Python first. The Python schema
must be ready for the new fields before Engine emits them.

**Migration rollback:** Alembic `downgrade` removes the 4 columns.
Backward-compat preserved.

---

## 6. ADR-024 Draft

```markdown
### ADR-024: RawFinding schema extension — References, Tags, CVSSVector, AdditionalCWEs
**Status:** Accepted (2026-05-XX, M6-close-followup task)

**Context.**
The reductions counter — tracking M6 tools whose parsers fold or drop
fields from upstream tool output — fired its trigger at M6.4 (SSLyze)
and accumulated through M6 close at 8/9 tools (89%) with ~38 total
folded/dropped fields.

The reductions are not problems in themselves — folding is legitimate
when canonical schema lacks slots. The decision is whether to extend
the canonical RawFinding schema with new categorical fields.

Per SPEC §7.3 (the canonical RawFinding wire-format doc, identified
as such in ADR-017's "Schema versioning" follow-up): RawFinding is the
shared schema between Engine (Go) and Python orchestrator. Extension
requires synchronized changes across Engine struct, Python SQLAlchemy
model, Python Pydantic schema, Alembic migration, and wire-format
fixtures.

The brainstorming process locked moderate scope (4 fields) over
conservative (2) and comprehensive (6) options. The 4 fields capture
the highest-value categorical patterns; comprehensive scope's Evidence
map was rejected as anti-pattern (flexible-map dumping ground).

**Decision.**
Extend `events.RawFinding` (and the SQLAlchemy + Pydantic mirrors)
with 4 new optional fields:

```go
// internal/events/events.go
type RawFinding struct {
    // ... existing fields ...
    References     []string `json:"references,omitempty"`
    Tags           []string `json:"tags,omitempty"`
    CVSSVector     string   `json:"cvss_vector,omitempty"`
    AdditionalCWEs []string `json:"additional_cwes,omitempty"`
}
```

Field semantics:
- **References:** Array of URLs / advisory identifiers (CVE links,
  remediation guidelines, OWASP references). Free-form strings.
- **Tags:** Array of fine-grained tags. Distinct from existing
  `engine_category` (broad: dast/sast/etc.). Tags is finer-grained
  per-tool sub-categorization.
- **CVSSVector:** Canonical CVSS 3.1 vector string
  (`"CVSS:3.1/AV:N/..."`). Stored as-emitted; no parsing into 8D
  subfields at this scope.
- **AdditionalCWEs:** Array of CWE strings beyond primary `CWEID`.
  Empty when finding is single-CWE.

All fields Optional with `omitempty`; backward-compatible with
existing engine emissions.

**Rationale.**

Three scope alternatives considered and rejected:

| Alternative | Why rejected |
|---|---|
| **Conservative (2 fields):** References + Tags only | Defers CVSSVector + AdditionalCWEs to future task. AdditionalCWEs is load-bearing for §8.2 correlation; deferring delays M9 forward-pin. |
| **Comprehensive (6 fields including Evidence map):** | Evidence map is flexible-map anti-pattern: tends to accumulate inconsistent usage across tools. Better to add specific fields when patterns warrant than preempt with dumping ground. |
| **Status quo (no extension):** | 38 folded/dropped fields across M6 is the trigger fire point. Continuing without extension means M7+ tools accumulate folds against current schema; cleanup cost grows non-linearly. |

The 4-field selection is grounded in SPEC §8 read:
- **References:** Used by M11 dashboard + AI fix-generation prompts
  + future cross-tool dedup signal
- **Tags:** Used by M11 filtering + future AI categorization input
- **CVSSVector:** Reserved for §8.3 exploitability_multiplier
  derivation (currently uses derived base_cvss + separate detection)
- **AdditionalCWEs:** **Load-bearing for §8.2 correlation algorithm**
  extension to handle multi-CWE findings (especially Dep-Check)

**Cross-reference:** ADR-022 (recon-as-pre-scan-helpers) and ADR-023
(NativeRunner OutputFile mode) both invoke asymmetric-cost meta-
principle to justify architectural commitments. ADR-024 applies the
same meta-principle: extension cost (~6.5h cross-repo work) is
asymmetrically smaller than alternative cost (38 folds compounding
across M7+ tools, schema-extension trigger remaining fired without
incremental progress, M9 missing multi-CWE correlation upgrade).

**M9 forward-pin (load-bearing).**

SPEC §8.2 currently uses `cwe_id` singular for correlation:

```python
if dast.cwe_id == sast.cwe_id:
    score += WEIGHTS["cwe_exact"]
elif is_cwe_parent_child(dast.cwe_id, sast.cwe_id):
    score += WEIGHTS["cwe_parent"]
```

**M9 implementation MUST extend these checks to consider intersection
with `additional_cwes`:**

```python
# Extended logic (M9 implementation):
all_dast_cwes = {dast.cwe_id} | set(dast.additional_cwes or [])
all_sast_cwes = {sast.cwe_id} | set(sast.additional_cwes or [])

if all_dast_cwes & all_sast_cwes:
    score += WEIGHTS["cwe_exact"]
elif any_parent_child(all_dast_cwes, all_sast_cwes):
    score += WEIGHTS["cwe_parent"]
```

Without this extension, M9 correlation misses multi-CWE matches
(especially Dep-Check). This is the load-bearing M9 forward-pin of
the entire SPEC §7.3 extension.

**M9 forward-pins (non-load-bearing).**

- **References:** SPEC §8 algorithms do not use; reserved for UI
  cross-linking + remediation display + future dedup signal.
- **Tags:** SPEC §8 algorithms do not use; reserved for M11 filtering
  + future AI categorization. Must NOT duplicate engine_category.
- **CVSSVector:** SPEC §8.3 currently uses base_cvss numeric.
  Reserved for future exploitability_multiplier derivation
  (replacing separate publicly-accessible detection logic).
  M9 implementation may evolve §8.3 to parse CVSSVector when
  available.

**Consequences.**

Positive:
- 12 of 38 folded fields (~32%) rescued via per-tool parser retrofits
  in 6 of 9 tools (Nuclei, Semgrep, Gitleaks, Dep-Check, Checkov,
  Wapiti).
- M9 correlation algorithm has documented forward-pin for multi-CWE
  upgrade.
- M11 dashboard and future AI features have schema slots ready.
- Cross-repo schema-coordination pattern exercised (synchronized
  Engine + Python + DB updates); future schema extensions inherit
  this workflow shape.

Negative:
- Reductions counter does NOT clear post-§7.3. 8/9 tools still have
  reductions (count per tool reduced but not eliminated). Trigger
  remains fired; future incremental schema extensions may address
  tool-specific metadata.
- 3 tools (SSLyze, Nikto, CORStest) get zero retrofit at this scope;
  their reductions stay folded.
- Schema-extension cost (~6.5h cross-repo) is real; future schema
  extensions will incur similar costs unless batched with M7+ tool
  data.
- Two new tracked patterns at 1st instance:
  - Multi-repo schema-coordination commits (Engine + Python +
    docs ordering)
  - Optional-field additive migrations (Alembic upgrade with
    backward-compat)

**Alternatives considered (and rejected).** See Rationale table
above.

**Anti-patterns this prevents.**
- Evidence map accumulating inconsistent per-tool usage.
- Schema-extension via per-tool subfields (NucleiTemplateID,
  GitleaksRule, etc.) proliferating the schema.
- Premature DB indexes on fields that §8 algorithms don't query.
- Premature parsing of CVSSVector into 8D subfields when §8 doesn't
  need them yet.

**Triggers to revisit.**

1. **M9 §8.2 implementation.** When M9 lands the correlation
   algorithm, verify the multi-CWE extension is implemented per
   forward-pin above.
2. **M9 §8.3 implementation.** Decide whether to derive
   exploitability_multiplier from CVSSVector AV field (replacing
   separate detection logic) or maintain separate logic. Either is
   acceptable; CVSSVector schema is forward-compat.
3. **Trigger remains fired (8/9 tools post-§7.3).** Future incremental
   schema extensions may address remaining tool-specific metadata.
   Likely candidates: NucleiTemplateID + GitleaksRuleID (per-tool
   identifiers); Evidence map IF a 3rd+ instance of "tool-specific
   binary artifact storage" emerges across M7 tools.
4. **M7 tool reductions accumulate.** If M7 surfaces new categorical
   patterns (e.g., Trivy SBOM data, Nmap port-scan structure), a
   second SPEC §7.3 extension task may be warranted.
5. **DB query patterns surface.** If M11 dashboard or M9 pipeline
   shows query patterns that benefit from indexes on Tags/References/
   etc., add indexes via additive migration.

**Forcing functions.**
- Per-tool retrofit checklist documented in design doc; each retrofit
  has explicit field-mapping assertion.
- Wire-format roundtrip test ensures Engine + Python agreement on new
  fields.
- ADR-024 forward-pin text references SPEC §8.2 explicitly; M9
  implementation engineer searching for "additional_cwes" finds the
  algorithm-extension requirement.
- Reductions counter post-§7.3 status documented in DRIFT-LOG;
  future engineers see ongoing trigger status.

**Open follow-ups.**
- M9 §8.2 algorithm extension (per load-bearing forward-pin).
- Reductions counter tracking continues; future SPEC §7.3 extensions
  may land at M7 close or later.
- Optional-field migration pattern documentation if 3rd+ instance
  emerges (currently 1st instance at SPEC §7.3).
- Multi-repo commit-ordering pattern documentation if 3rd+ instance
  emerges (currently 1st instance at SPEC §7.3; 6.3 was a precedent
  but lighter-weight).

**Cross-references.**
- ADR-013: Python sole writer constraint (load-bearing for cross-repo
  coordination).
- ADR-017: Findings inline in job_completed events (identifies SPEC
  §7.3 as canonical RawFinding schema doc; "Schema versioning of
  RawFinding" follow-up resolved by ADR-024).
- ADR-022: Recon-as-pre-scan-helpers (asymmetric-cost meta-principle
  cross-reference).
- ADR-023: NativeRunner OutputFile mode (asymmetric-cost meta-principle
  cross-reference).
- SPEC §7.3: Canonical schema doc (updated in this commit to add 4
  fields + their semantics).
- SPEC §8.2: Cross-layer correlation algorithm (load-bearing forward-pin).
- SPEC §8.3: Severity scoring formula (CVSSVector forward-pin).
```

---

## 7. Test Strategy Detail

### 7.1 New Test Categories

**Per-tool retrofit tests (~6 × 1–3 tests per tool = ~12–15 tests):**
- `TestParseOutput_PopulatesReferences` — fixture with refs → finding
  with References array
- `TestParseOutput_PopulatesTags` — fixture with tags → finding with
  Tags array
- `TestParseOutput_DropsEngineCategoryFromTags` — regression guard
- `TestParseOutput_PopulatesCVSSVector` — Nuclei + Dep-Check
- `TestParseOutput_PopulatesAdditionalCWEs` — Dep-Check
- `TestParseOutput_BackwardCompat` — fixtures without new fields still
  parse (each tool)

**Wire-format roundtrip (~3 tests):**
- `TestRawFindingMarshal_NewFieldsOmitemptyWhenAbsent` — engine side
- `TestRawFindingUnmarshal_NewFieldsOptionalOnIngest` — engine side
- `test_completions_consumer_persists_new_fields` — Python side

**CVSS vector composition (~5 tests):**
- `TestComposeCVSSVector_NetworkVector` — Dep-Check input → canonical
  string
- `TestComposeCVSSVector_LocalVector` — different attack-vector value
- `TestComposeCVSSVector_AllFieldsPresent`
- `TestMapAttackVectorWord_NetworkToN` — table-driven attack-vector
  mapping
- `TestComposeCVSSVector_HandlesEmptyFields` — graceful degradation

**Estimated total new tests:** ~25 across Engine + Python.

### 7.2 Updated Existing Tests

Existing M6 parser tests must remain green. Adjustments needed where
fixtures change:
- Tools with retrofits: existing fixtures may be augmented with new
  fields; existing assertions remain valid.
- 3 tools without retrofits: no changes needed.

### 7.3 Backward-Compat Test Pin

Critical: `TestParseOutput_HandlesFixturesWithoutNewFields` for each
retrofitted tool. Validates that existing fixtures (pre-§7.3) still
parse without the new fields (Engine side: omitempty; Python side:
Optional).

---

## 8. DRIFT-LOG Entries Plan

**Engine commit (Commit 3) DRIFT entries (~10):**
1. SPEC §7.3 schema extension landed (followup task)
2. RawFinding struct extended with 4 fields (per ADR-024)
3. Per-tool retrofit checklist + outcomes (Nuclei + 5 others)
4. SSLyze/Nikto/CORStest no-retrofit acknowledgment
5. CVSS vector composition for Dep-Check (attack-vector word→letter
   mapping)
6. Multi-CWE intersection extraction for Dep-Check
7. Reductions counter post-§7.3: ~26 folds remaining (down from ~38);
   trigger remains fired
8. Two new tracked patterns at 1st instance:
   - Multi-repo schema-coordination commits
   - Optional-field additive migrations
9. M9 §8.2 algorithm-extension forward-pin (load-bearing)
10. Asymmetric-cost meta-principle 3rd invocation (after ADR-022/023)

**Docs commit (Commit 1) DRIFT entries (~3):**
1. ADR-024 lands (M6's 3rd ADR; M6-followup architectural artifact)
2. SPEC §7.3 schema doc updated with 4 new fields + semantics
3. M6-close-followup task acknowledgment (closes the SPEC §7.3
   trigger-fire deferral from 6.4)

---

## 9. Implementation Plan Handoff

This design doc is the architectural artifact. Implementation plan
shape:

**Per `writing-plans` skill, the implementation plan would cover:**

**Phase 0: Pre-implementation verification**
- Verify SPECIFICATION.md current §7.3 wire-format
- Verify Python repo `raw_findings.py` SQLAlchemy model location
- Verify Alembic migrations directory structure

**Phase 1: Docs commit**
- Draft ADR-024 final (per template above)
- Update SPEC §7.3 with new fields + semantics
- Add DRIFT-LOG entries
- Commit + push

**Phase 2: Python commit**
- Extend SQLAlchemy model
- Extend Pydantic schema
- Generate Alembic migration
- Update CompletionsConsumer tests
- Update fixtures
- Verify all tests pass
- Commit + push

**Phase 3: Engine commit**
- Extend RawFinding struct
- Update fixture(s) at internal/events/testdata/
- Per-tool retrofit (Nuclei, Semgrep, Gitleaks, Dep-Check, Checkov,
  Wapiti — in this order)
- Add DRIFT-LOG entries
- Run full test suite (race + lint + format)
- Commit + push

**Phase 4: Cross-repo verification**
- Smoke test: Engine emits findings with new fields; Python ingests;
  DB persists.
- Verify reductions counter status: 8/9 tools (89%) still have
  reductions but per-tool counts reduced.

**Estimated total: 6.5–7 hours per brainstorming bandwidth confirmation.**

---

## 10. Open Questions / Triggers

### 10.1 Pre-Implementation Decisions

1. **Dep-Check CVSS attack-vector mapping table:** Is there a canonical
   CVSS spec mapping (NETWORK→N, ADJACENT_NETWORK→A, LOCAL→L,
   PHYSICAL→P) that's authoritative? Implementation should reference it.
2. **Semgrep metadata variability:** Custom rules may have different
   metadata shapes. Ingest should be tolerant (missing metadata → nil
   Tags/References, not error).
3. **Wapiti `module` field as Tag:** Module names are technical
   (e.g., `http_headers`, `xss`). Are these the right Tag values,
   or should there be a normalization layer? Lean: ship as-is at
   §7.3 time; M11 normalizes if needed.

### 10.2 Future Triggers

1. **M9 implementation:** Verify multi-CWE correlation extension
   lands per ADR-024 forward-pin.
2. **M11 dashboard:** Verify Tags/References query patterns
   surface; add indexes if warranted.
3. **M7 tool reductions:** Track whether M7 surfaces patterns
   warranting another SPEC §7.3 extension.
4. **Reductions counter tracking continues:** Post-§7.3, counter
   at 8/9 tools (89%); incremental schema extensions may follow.

---

## 11. Brainstorming Process Acknowledgment

This design emerged from a structured brainstorming session that
locked six decisions through five clarifying questions. The
grounded SPEC §8 read was particularly valuable — it surfaced the
load-bearing AdditionalCWEs forward-pin that distinguishes this
schema extension from a generic "add useful fields" exercise.

The forcing-function pattern from M5/M6 (architectural-commitments
preamble in scope proposals; trigger-based deferral; empirical
re-evaluation) extends naturally to followup tasks. ADR-024's
asymmetric-cost cross-reference to ADR-022/023 establishes the
meta-principle as project corpus norm.

This design doc itself follows the M6 pattern of documenting
architectural intent before code lands. Future engineers reading
the project corpus see the brainstorming → design → ADR →
implementation chain explicitly.

---

**Status:** Ready for implementation per writing-plans skill workflow.
**Approval required:** None outstanding; brainstorming locked all
decisions.
**Next action:** Transition to writing-plans skill for the
phase-by-phase implementation plan, OR proceed directly to Phase 1
(Docs commit) implementation with this design doc as the reference.
