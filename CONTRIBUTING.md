# Contributing to the Alatyr specification

This repository is a **specification book**, not a codebase. There is nothing to build,
nothing to run, and no test suite. What is maintained here is *coherence*: a set of
documents that must never contradict each other, held to the bar that **two independent
implementations built strictly to this text produce compatible results without guessing**
(the **`FND-3` bar**, in `design/decisions/foundations.md`).

That changes what a good contribution looks like. The most valuable thing you can send is
not a patch — it is a **precisely located contradiction or gap**.

`AGENTS.md` is the full orientation document and the canon for how work is done here;
this file is its human-facing summary. Where the two differ, `AGENTS.md` governs.

## The sources of truth, in priority order

On any conflict the **higher** source governs, and the discrepancy below it is a
**defect** to be fixed — not a second opinion to be balanced.

1. **`design/invariants.md`** — the invariants `I1–I11`. The supreme norm. A rule that
   violates an invariant is non-conforming by definition.
2. **`spec/`** — the normative specification. Within it, a feature chapter is normative
   for the *semantics* of its area; `130-grammar.md` is normative for *syntax*; where
   form and meaning diverge, the feature chapter governs the meaning. Enumerated v1 data
   lives in the appendices `140`–`170`.
3. **`design/decisions/`** — *why* the specification is the way it is, by theme, under
   stable theme-prefixed IDs (`MEM-`, `TYP-`, `FN-`, …). Not normative on its own, but
   the specification and the decisions must be reconcilable in both directions.
4. **`design/glossary.md`** — canonical terms, one definition each.
5. **`design/aspects-map.md`** — the aspect map and status.

`README.md` mirrors these; it never replaces them.

## What makes a good issue

Please open an issue before a pull request for anything beyond a typo. Three kinds are
especially welcome, and they map to the issue templates:

- **Spec defect** — two places in the repository disagree, or a chapter contradicts an
  invariant. Cite both locations by file and section (`spec/50-functions.md §5.4`), quote
  the two lines, and say which one you believe is right and why.
- **Underspecification gap** — the text names something but never gives its rule, so two
  implementers would have to guess. This is an `FND-3` blocker even when nothing is
  self-contradictory. High-yield shapes: an operation whose result rule is missing for an
  edge case (rounding, out-of-range, NaN, ordering, evaluation order); a category a rule
  depends on that is never defined; a manifest field with a default but no stated effect.
- **Design proposal** — a change to what the language *is*. Expect it to be argued, not
  merged. Two questions decide it before any text is written:
  - **Minimalism (`PRIN-1`)** — does this give something beyond what existing mechanisms
    already express, or does it merely save a name? If the latter, it is rejected.
  - **Additivity (`I10`/`PRIN-2`)** — v1 is fully specified. "Additive" governs *future*
    growth without breaking v1 programs; it is never a license to leave v1 vague.

Questions about how the language works are also fine as issues — if the answer is not
plainly in the text, that is itself a gap worth recording.

## Working on a change

- **Decide first, record second.** Anything contentious is discussed to convergence
  *before* it is written down. Do not open a PR that settles a design fork by fiat.
- **One decision = one commit.** A change to a norm updates *every* affected source in
  the same commit: invariants, decisions, glossary, aspects-map, and the chapters. The
  cross-cutting pass is what holds coherence; splitting it across commits is what breaks
  it.
- **Refine in place (pre-v1).** While the spec is a draft under review, a change to a
  not-yet-shipped decision is recorded by **editing its theme entry** to the current
  truth, with a concise `Rejected:` / `Superseded:` note. Entries carry the current
  rationale, not full history — git holds the history.
- **IDs are theme-prefixed and append-only.** Add a new entry for a genuinely new
  decision area; never reuse or shift an existing ID, and never silently repurpose the
  core meaning of a widely-cited one — supersede it with a new entry instead. Keep the
  entry's `provenance:` line — what it refines, and where it lands in `spec/` — current.
- **Fix inconsistencies, do not route around them.** If you find one while doing
  something else, either fix it in its own commit or file it.

## Cutting a revision

**Commits are not revisions.** A version number identifies a *published revision of the
document* — something an implementer can cite, pin to, and conform against — so it moves
when a revision is cut, not when a commit lands. Most commits carry no tag; their entries
accumulate under **Unreleased** in `CHANGELOG.md`.

Tagging every commit would turn the version into a commit counter and destroy the one
thing MINOR and PATCH exist for: telling an implementer whether they must re-read.

**When to cut one.** There is no cadence, and a long stretch with a growing *Unreleased*
section is normal during review. Cut a revision when someone downstream needs a stable
thing to point at:

- a batch of review findings has been resolved and the text is coherent again;
- an implementation is about to conform against the current text and needs a citable
  revision;
- a single normative change is urgent enough that implementers should not wait for the
  batch.

**How to cut one.**

1. **Pick the digit** from what accumulated under *Unreleased*: any normative entry →
   MINOR; editorial only → PATCH. MAJOR is the language version and does not move.
2. **Update the version in the four places that must agree** — the Status block in
   `spec/00-overview.md`; `README.md` (badge, Status block, implementers section,
   repositories table); `CHANGELOG.md`; and the citation example in `LICENSE` §4.
3. **Rename *Unreleased*** to the new version, and open a fresh empty *Unreleased* above
   it.
4. **Commit** as `Release <version>`.
5. **Tag and push**: `git tag -a v1.1.0 -m 'Alatyr v1 specification, revision 1.1.0'`
   then `git push --follow-tags`. The git tag carries a leading `v`; the document version
   does not.

The tag is what makes a citation resolve. "Conforms to the Alatyr specification 1.1.0"
has to land on exact text, permanently — a branch name cannot promise that.

## Pull-request checklist

- [ ] Every affected source of truth is updated **in the same commit** (invariants /
      decisions / glossary / aspects-map / chapters).
- [ ] No invariant `I1–I11` is violated — in particular I1 (transparent codegen),
      I2 (zero-cost), I3 (nothing hidden), I11 (no UB).
- [ ] Cross-references resolve: cited section numbers exist and say what you claim, and
      cited decision IDs are the *right* ones (a real ID can still be the wrong one).
- [ ] Terminology matches `design/glossary.md`; new load-bearing terms are added to it.
- [ ] The change is written to the `FND-3` bar — an implementer needs no further guess.
- [ ] English only, matching the surrounding prose.

## Conventions that are easy to break

- **The verification axis is `checked`/`unchecked`**, never safe/unsafe. `unchecked` is a
  *scoped verification mode*: inside `unchecked expr` / `unchecked { … }` the checked-guard
  family is dropped and raw operations become writeable. The transitive prohibition is the
  `no_unchecked` limit.
- **One language plus orthogonal limits** (`no_abstractions` / `no_alloc` / `freestanding`
  / …). There is no ladder of "levels" or "tiers".
- **No undefined behavior (I11).** Checked operations are correct-or-trap; unchecked and
  raw operations are *hardware-defined* — never "whatever suits the compiler".
- Function results are `-> T` (single anonymous) or a named `out`; pointers are `ptr(T)`;
  pointer operations are the flat word-functions `ptr` / `deref`; built-ins are prelude
  identifiers — `@` is for attributes only, never intrinsics.
- **Do not add implementation code, a build, or tests to this repository.** The
  reference toolchain lives in the sibling
  [`compiler`](https://github.com/alatyr-programming-language/compiler) repository and
  the standard library in
  [`stdlib`](https://github.com/alatyr-programming-language/stdlib); both are built
  *against* this text.
- **The toolchain conforms to the specification, never the reverse.** A patch here is not
  justified by "the compiler already does it" — that an implementation shipped a behavior
  is not a rationale. If the two disagree, either the compiler has a bug, or this text has
  a defect or a gap that is fixed here first, with its reasoning recorded, and followed
  downstream afterwards.

## Language and formatting

- **English** throughout — `spec/`, `design/`, and all docs.
- Prose wraps at roughly 90 columns; match the surrounding file rather than reflowing it.
- **ASCII letters only** in identifiers, labels, and enumerations. Typographic characters
  (`—`, `§`, `→`, `≤`, `…`) are used deliberately and are fine; Cyrillic look-alikes
  (`А`, `В`, `С`) inside otherwise-Latin text are a defect, because they are invisible to
  the reader and to search alike.
- Spec file numbering is filename × 10, natural-sort. **The order of presentation is per
  Overview §7**, not literally by file number.

## Licensing of contributions

By submitting a contribution you agree it is licensed under the repository's dual terms —
CC BY 4.0 for specification text, Apache-2.0 for code fragments — as set out in
[`LICENSE`](LICENSE). No separate CLA is required.
