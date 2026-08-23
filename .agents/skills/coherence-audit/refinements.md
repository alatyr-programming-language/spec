# Refinements — learned instruction for `coherence-audit`

This file is **not** a log of past findings (git history and the run reports hold those).
It is a thin layer of **additional, distilled instruction** appended to `SKILL.md`. A
line is added **only when it demonstrably improves the skill at its purpose** — finding
real coherence defects and underspecification gaps faster, with fewer false positives.

Rules for this file:
- **Generalize every entry to a heuristic.** Never record an incident ("on date X, line Y
  cited the wrong entry"); record the *rule learned from it* ("a cited ID can be real but
  wrong — read the cited decision, don't trust the number").
- **Earn its place.** If an entry no longer sharpens a run, prune it. Keep this short.
- **Human-approved.** The skill proposes additions; a human accepts them here.

## Where to spend the pass (priority)

- **Lead with gap-hunting under the `FND-3` bar, not reference-checking.** The mechanical /
  cross-ref class has been clean across runs on this spec — treat it as a regression
  guard (a quick grep pre-pass), and spend the real budget on underspecification.
- **High-yield gap shapes** — probe these in every run (none are grep-catchable):
  - something "declared / explicit / never inferred" with **no mechanism** to declare it;
  - an operation or conversion **named but with its result rule absent** — rounding,
    out-of-range, NaN, ordering legality, evaluation order, discriminant value,
    subnormals;
  - a surface **delegated to an appendix that self-refers without enumerating** its
    members;
  - a load-bearing guarantee resting on an **undefined grammar terminal**;
  - a schema **field with no value-space or no resolution rule**.
  - **a rule's own categories**: when a new rule is a **table** or a classification, check that
    every *column and category* it introduces is **defined somewhere**, not just that the rows
    are complete. A table that makes an outcome depend on an undefined category ("reachable
    from the entry", "the platform primitive") is an `FND-3` blocker even though every cell is
    filled — and the missing definition is usually invisible to grep, because the term reads
    like ordinary English.
  - **schema field with a default but no effect**: a field that has a default, a validation
    rule, a CLI flag, or a conformance duty, but **no rule anywhere saying what it does**, is
    an orphan (the field catalog's own bar). Check the *effect* side whenever a field gains
    weight; giving an inert field a default makes it look implemented and is the easiest way to
    smuggle one in. Related: if the field names a directory or file that holds **inputs**,
    a "location only, never build input" clause about it is self-contradictory.
  - **"the toolchain does X instead of the platform"**: any rule that replaces a platform
    mechanism (start-up files, a syscall, a linker default) must be checked **against every
    supported triple**, not against the one the author had in mind. The mechanism may simply
    not exist on one of them (no stable syscall ABI, no static system library, no link step),
    and the default configuration is exactly where that hurts.
  - **closed-set member with no rule**: when a chapter enumerates a **closed/fixed set**
    (the machine levers, the attribute categories, the supported triples, the ABI names),
    verify **each named member has its own definition/surface/lowering rule** — not merely
    that the set is listed. A set that promises "each has a documented rule" while one
    member is defined nowhere is an `FND-3`-bar blocker (the lever set named `repr` but no
    chapter specified it).
- **High-yield coherence shapes** — also not grep-catchable, must read:
  - **decision-attribution**: a cited theme ID that exists but is the *wrong* one;
  - **resolves-but-absent**: a `§N` that exists yet does **not** contain the rule the
    reference claims.
  - **example-only surface**: a construct shown only by **example** (esp. a binding/LHS
    shape or other syntax) with **no grammar production** behind it — a recurring gap
    class. When a feature chapter demonstrates a form, confirm the Grammar chapter
    actually derives it.
  - **orphaned grammar terminal** (the dual of example-only surface): a terminal/
    production that **is defined** in the Grammar chapter but appears on **no RHS** —
    unreachable from any start symbol, so the construct still isn't derivable. After
    confirming a terminal exists, confirm something **uses** it (`label-attr` was defined
    yet wired into no production). Grep the terminal name and check for a second hit.
  - **same-numbered chapter/appendix citation**: a bare "`Chapter §N`" citation pointing
    at **enumerated data** is suspect when that chapter has a **same-numbered appendix**
    (e.g. Stdlib chapter §5.2 vs Stdlib appendix §5.2). Confirm the cited content lives in
    the **chapter's** §N, not the appendix's — the repo reserves bare chapter names for
    the narrative chapter and cites the appendix as "‹X› appendix §N".

## After a change set (post-change runs)

When the run audits a **recent change set** (not a cold read), **weight the propagation
surface over the edit site**. Defects cluster not where edits landed (just-touched spots
are usually fine) but in what *refers to* the changed thing:
- **derived / summary sources** — `aspects-map` (esp. Deferred/status),
  `README`/`AGENTS`, `glossary` — they lag, because edits land in the normative chapters;
- **descriptive prose around a changed token** — a mechanical rename or count-bump updates
  the token but not the sentence that *describes* it in the old terms.
- **a closed-set / family enumeration is usually duplicated across several decisions** — when
  a member is removed from (or added to) a named family (the introducer family, the machine-
  lever set, the keyword list, the ABI names), grep *every* enumeration of that family across
  the whole decision set and spec. The defining decision gets updated; the sibling decisions that
  re-list the set for their own purpose are the ones that drift (the introducer family was
  fixed in its home decisions but a label/control decision still re-listed the dropped member).

- **an appendix's own conformance list is the last thing anyone updates** — the chapter's list
  gets revised with the norm, its appendix's does not. After changing a rule that an appendix
  states, open that appendix's conformance section explicitly; a stale item there is worse than
  stale prose, because it is the implementer's checklist.
- **inserting into a numbered list**: renumber with plain integers and shift the rest. Letter
  suffixes (`4a.`, `4b.`) are not list markers in CommonMark, so the inserted block *and every
  item after it* collapse into the preceding item's paragraph — a rendering break that reads
  fine in the diff. Check any new item in a numbered list for this.

So after a change, audit "everything that mentions or summarizes what changed", not the
change itself.

## Verification discipline

- Confirm every candidate in-source before reporting. For an enumeration reference, check
  the target **contains the items**, not merely that the section exists.
- Prefer dropping an uncertain finding (and noting it) over emitting a false positive.

## Known non-defects — do NOT re-raise

- **Negation mentions** of removed constructs ("no `Atomic(T)`", "`@alloc` unifies the
  earlier `@heap`/`@region`", "`use` removed", "no `-> T`", "no separate `@noreturn`") are
  correct. Flag a removed construct **only** when used as *live* syntax.
- **List-ordering variance** (e.g. `riscv32`/`riscv64` order across files) — set-identical,
  cosmetic at most.
- A term a brief assumes but that legitimately lives elsewhere (e.g. `Never` is fixed by
  STD-1, not the glossary) is not a defect.
