---
name: coherence-audit
description: >-
  Run an end-to-end coherence / gap-finding pass over a coherence-disciplined
  specification repository (the Alatyr spec book). Use when asked to "check
  coherence", "find inconsistencies / gaps", audit cross-references, or verify
  the spec against its sources of truth (invariants / decisions / glossary).
  Tool-neutral procedure usable by Claude and Codex agents. Report-only:
  proposes fixes and refinement additions for HUMAN approval — never auto-edits the
  spec, never commits, never modifies itself.
---

# Coherence check

This is **spec-maintenance meta-tooling**, NOT the Alatyr toolchain. It never
becomes part of an implementation, build, or test of the language. It reads the
specification and reports; a human applies anything it proposes.

## What it does, in one line

Find places where two normative statements cannot both be true, a cross-reference
is wrong, a decision/invariant is contradicted, terminology drifts, or a promised
piece of v1 content is missing — then report them, propose fixes for approval, and
propose what to remember for next time.

## Operating contract (hard guardrails)

1. **Report-only.** Output is a report plus *proposed* patches. The human (or an
   explicit follow-up turn) applies them. Apply fixes **one decision = one commit**,
   updating every affected source of truth synchronously (the repo's rule).
2. **No self-modification.** This skill never rewrites itself. "Learning" happens
   only by proposing additions to `refinements.md` — a thin layer of **distilled
   instruction** (generalized heuristics, never an incident log), added only when it
   improves the skill at its purpose, and only with human approval.
3. **No auto-commit, no auto-push.**
4. **Always declare coverage and limits.** Every report states what was checked
   against what, and what was *not* checked. A clean result means "no defect found
   in the covered surface", never "fully verified". This is the antidote to false
   confidence — a green run must not discourage the next human read.
5. **Verify before reporting.** Every candidate finding is confirmed by opening the
   source line(s) — adversarially try to refute it. Report only what survives.
   False positives cost trust; prefer to drop an uncertain finding (and note it).

## Sources of truth (priority order — higher governs lower)

From the repo's own rules (`AGENTS.md`):

1. `design/invariants.md` — invariants `I1–I11`. Supreme. A rule that violates an
   invariant is non-conforming by definition.
2. `spec/` — the normative chapters (`00`–`130`) and appendices (`140`–`170`).
   Within it: a feature chapter is normative for its *semantics*; the Grammar
   chapter (`130`) is normative for *form*; enumerated v1 content lives in the
   appendices (schema in the narrative chapter, data in the appendix).
3. `design/decisions/` — the decisions, **by theme** (`memory.md`, `types.md`, …) with
   stable theme IDs (`MEM-`/`TYP-`/`CT-`/…), indexed by `README.md`. Not normative alone,
   but every spec change must reconcile with it.
4. `design/glossary.md` — canonical terms (single definitions). The spec governs
   over the glossary on conflict, but flag the lag.
5. `design/aspects-map.md` — status and the cross-cutting decision summary.

`README.md` mirrors these; it must agree on the counts.

## Procedure

### 0. Load context
Read all six sources above and the **refinements file** (`refinements.md` in this
skill's directory) — distilled instruction learned from prior runs: where to spend the
pass, verification discipline, and known non-defects to suppress. It is instruction, not
a findings log.

### 1. Mechanical pre-pass (cheap; catches the regression class)
Grep-level checks across `spec/` + `design/` + `README.md` + `AGENTS.md`:
- **Reference integrity:** every decision citation is a theme ID (`MEM-`/`TYP-`/…)
  defined as a `### XXX-N` header in a `design/decisions/` theme file; every `Ixx` in
  `invariants.md`. A bare `Dxx` anywhere is itself a defect — the legacy layer is gone.
- **Status lines:** every chapter/appendix carries `Status — draft (under review)`
  (Overview may say `complete first draft, under review`).
- **Removed/renamed constructs appear ONLY as negations**, never as live syntax —
  e.g. `use`, `snippet`, `const`, `Atomic(T)`, `@noreturn`, `@heap/@region/`
  `@generational`, `sizeof`, a `label` introducer keyword, `T?` Option-sugar.
  (A line that says "there is no `X`" / "the earlier `X`" is fine; one that *uses*
  `X` as current syntax is a defect.)
- **Canonical spellings:** `ptr(T)`/`deref`/`T.size()`/`T.align()`, arch-surface `at`, `bitsN`, `@alloc`,
  `@reg/@stack/@static/@scoped`, `@label`, `checked/unchecked/no_unchecked` (never
  `safe/unsafe` as the axis), `Never`.
- **Count consistency** across `README.md`, `AGENTS.md`, `aspects-map.md` (I-count,
  per-theme decision-ID counts vs the `design/decisions/README.md` theme table, arch
  count, chapter+appendix count).

NOTE in the report: this class was clean at the last full audit. It mostly guards
against *regressions*; it rarely finds *current* gaps.

### 2. Semantic fan-out (the high-value class)
This is where real defects live. Split the ~18 files into clusters of related
chapters and audit each cluster against the sources, in parallel where the harness
allows (Claude: parallel `Agent` calls or a `Workflow`; Codex: equivalent
sub-agents or sequential focused passes). A working clustering:

- **A — memory/types/comptime:** `10`, `20`, `30` (vs I2/I3/I6/I7/I8/I11; `MEM-*`/`TYP-*`/`CT-*`,
  `STD-1` and neighbours).
- **B — declarations/functions/control:** `40`, `50`, `60` (vs `FN-*`/`CF-*`/`SYN-*`/`CG-6`/
  `CG-7`; the `?` operator, `@label` two kinds, the `out`/`-> T` result model).
- **C — modules/assembly/codegen:** `70`, `80`, `90` (vs I1/I4/I5/I9; `MOD-*`,
  `CG-*`; `::` vs `.`, mangling, no_abstractions byte-exact, rule-not-byte
  conformance).
- **D — stdlib/concurrency/tooling/grammar:** `100`, `110`, `120`, `130` (vs `STD-*`,
  `CC-*`, `TOOL-*`, `OP-*`/`SYN-*`; overflow trap-set, atomics-on-ptr, float
  determinism, profiles non-semantic, reserved keywords).
- **E — appendices ↔ their chapters:** `140↔120`, `150↔50`, `160↔100`, `170↔80`
  (schema-in-chapter / data-in-appendix; field names, enumerated v1 content,
  triples, counts).
- **F — global cross-cutting:** terminology drift across all files, named
  chapter-reference resolution (`§N` exists in the named target), and the count
  consistency from step 1.

For each cluster, look specifically for:
1. **Contradictions** — two normative statements that cannot both hold.
2. **Wrong cross-references** — a `§X.Y` or named-chapter reference that points to a
   nonexistent or wrong section. (Check the target actually has that section AND
   actually contains the referenced content — a section can exist yet not define
   what the reference claims; that is still a defect.)
3. **Decision/invariant mismatches** — a claim attributed to a decision entry that
   the entry does not make, or a rule that contradicts an invariant.
4. **Completeness gaps** — an enumeration (storage specifiers, ABI names, fields,
   built-ins) that omits an item present in the grammar/another source; promised v1
   content that no appendix actually enumerates.
5. **Terminology drift** — one concept named differently across chapters or vs the
   glossary.

### 3. Verify
Open each surviving candidate in-source. Try to refute it. Keep only confirmed
defects. Note dropped candidates briefly (may feed the refinements file's non-defects list).

### 4. Report
Group by severity: **BLOCKER** (internal contradiction / unnameable-or-missing v1
content / invariant violation) · **MINOR** (attribution / cross-ref) · **COSMETIC**
(ordering, imprecise-but-resolving refs). For each: `file:line` (both sides for a
two-file conflict), the conflicting texts quoted briefly, which source governs, and
a one-line fix. End with the **coverage/limits declaration** (clusters covered,
sources checked, what was not examined).

### 5. Propose fixes (for approval)
After the report, present the concrete edits as proposed patches/diffs — surgical,
matching house style, touching every affected source of truth for each fix. Do NOT
apply them; wait for the human. When applying later, one decision = one commit.

### 6. Propose refinement additions (for approval)
Derive only **generalized instruction that would improve a future run** — a sharper
place to look, a verification rule, or a non-defect to stop re-raising. Phrase each as a
heuristic, never as an incident log. Add nothing that does not earn its place. Present
them as proposed additions to `refinements.md`. Do NOT write them without approval.

## Notes
- The value is the run-time reasoning; this file only guarantees the run is
  structured, complete, and remembers. Keep the file boring and stable — the one
  maintenance point is the cluster map in step 2, which degrades gracefully (group
  by *relatedness*, not by exact file numbers, if chapters are renumbered).
- Scale effort to the request: a quick check can run the mechanical pre-pass plus
  one or two clusters; "thorough"/"audit" runs all clusters with adversarial verify.
