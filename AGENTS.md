# AGENTS.md — orientation for agents

## What this repository is

This is a **specification book** for the **Alatyr** programming language — **not** its
implementation. It is a **living, versioned specification**: the normative source of
truth for the language and its tooling, maintained over the whole life of the project.
The goal is twofold:

1. **design the language and keep that design coherent** — aspect by aspect, recording
   every decision and the reasoning behind it; and
2. **produce and maintain a normative specification of the language and its tooling**
   (compiler/codegen, manifest and build system, CLI, diagnostics, cross-toolchain)
   precise enough that **implementations can be reproduced without contradiction** — two
   independent toolchain implementations built strictly to the specification yield
   **compatible** results **without guessing** (this is the **`FND-3` bar**).

The specification is **living**: it does not stop once a compiler exists. As
implementation experience accrues, design decisions are **refined** and the
specification is **versioned** here — this repository stays the home of the spec across
the language's life. It contains **no implementation code, build, or tests**, and none
should appear. The repository's artifacts are **design text and a normative
specification**.

The implementation is developed alongside, in the sibling repository
**`alatyr-programming-language/compiler`** — the reference toolchain, in development,
carrying the prelude and standard library (chapter `100` / appendix `160`) with it rather
than in a repository of their own. It is built *against* this spec, and its feedback flows
back here as refinements.

**The direction of authority does not reverse.** The toolchain conforms to the
specification, never the other way round. If the compiler does X and the spec says Y, the
resolution is one of two things — a bug in the compiler, or a defect/gap here that is
fixed **here first**, as a recorded decision with its reasoning, and only then followed
downstream. Never amend a norm merely because an implementation already behaves that way:
"the compiler does it" is not a rationale, and an implementation detail does not become
normative by having shipped.

Tooling is a **first-class subject of the specification**, not an addendum to the
language: its contract (configuration = the manifest, the codegen pipeline, reproducible
builds, diagnostics) is specified normatively on a par with the language semantics
(chapters `90-codegen` / `120-tooling` + appendices `140`/`150`/`170`).

Alatyr is a low-level systems language: a ceiling of expressiveness equal to the
assembler's (everything expressible in the target backend's lowest form — GAS for a
register ISA, WAT for WASM; CG-4 — is expressible in the language), but with abstractions
that are transparent in code generation, zero-cost.
Codegen is per backend (CG-4): GAS text → `as` + `ld` for the register ISAs (not LLVM); a structured/VM
backend such as WASM (→ WAT) is additive. Details: `spec/00-overview.md`.

## Sources of truth (in priority order)

On any conflict the **higher** source governs; a discrepancy below it is a **defect** to
be fixed, not a "second opinion".

1. **`design/invariants.md` — the invariants `I1–I11`.** The supreme norm. A construct,
   rule, or behavior that violates an invariant is ill-formed / non-conforming by
   definition. Where a specification chapter conflicts with an invariant, the
   **invariant governs** (Overview §4).
2. **`spec/` — the normative specification** (English). Chapters `00`–`130` +
   appendices `140`–`170`. This is what an implementation must conform to. Within it:
   - a **feature chapter is normative for the semantics** of its area;
   - the **Grammar chapter (`130`) is normative for syntax (form)**; where form and
     meaning diverge, the feature chapter governs the meaning (Grammar §5);
   - enumerated **v1 content lives in the appendices** (`140` manifest, `150` ABI,
     `160` prelude/stdlib, `170` per-arch tables + supported targets). The schema is in
     the relevant narrative chapter; the data is in the appendix.
3. **`design/decisions/` — the decisions (thematic).** *Why* the specification is the way
   it is: what was decided and which alternatives were rejected, organized **by theme**
   (one file per area; stable theme-prefixed IDs `MEM-`/`TYP-`/`FN-`/…), with
   **`principles.md`** for the cross-cutting priorities and **`README.md`** for
   navigation. Not normative on its own, but any change to the specification must be
   reconciled with the decisions (and vice versa).
4. **`design/glossary.md` — canonical terms.** Single definitions (value/place/binding/
   address, checked/unchecked, protocol-form, label, `?`/try, etc.).
5. **`design/aspects-map.md` — the aspect map and status.** What is designed/written/in
   progress, a per-aspect summary, and the deferred (additive) growth list; the decision
   rationale itself lives in `design/decisions/`.

`README.md` is a human-readable status overview (it mirrors, it does not replace, the
sources above).

### Repository-level documents (non-normative, but kept in sync)

None of these define the language; all of them go stale silently, so treat them as part
of the cross-cutting pass whenever a norm or the repository's status changes.

- **`CONTRIBUTING.md`** — the human-facing restatement of this file's process. If the
  process here changes, change it there in the same commit; `AGENTS.md` governs.
- **`CHANGELOG.md`** — the versioning policy and release history. A **normative** change
  gets an entry under *Unreleased*; an editorial one does not.
- **The version** — `1.0.0`, stated in `spec/00-overview.md` (the Status block),
  `README.md`, and `CHANGELOG.md`. Those three must agree. MAJOR is the *language*
  version (pinned at `1` by `I10`/`PRIN-2`), MINOR a normative revision of the text,
  PATCH an editorial one. Maturity is **not** in the version string: it lives in the
  per-chapter `Status` lines, and acceptance flips those, not the number. Versions are
  **cut as revisions, not per commit** — normative entries accumulate under *Unreleased*,
  and a release bumps those places, renames the section, and is tagged `v<version>`
  (procedure in `CONTRIBUTING.md`). Do not cut or tag a revision unasked.
- **`LICENSE`** + `LICENSE-CC-BY-4.0` + `LICENSE-APACHE-2.0` — dual licensing: CC BY 4.0
  for specification text, Apache-2.0 for code fragments. Do not restate license terms
  anywhere else; link to `LICENSE`.
- **`CODE_OF_CONDUCT.md`**, **`.github/`** issue and PR templates — the PR checklist
  encodes this file's rules and must not drift from them.

## Current status

**The full first draft of the specification is complete and under review.** All 14
narrative chapters and all 4 appendices are written; `I1–I11` are recorded and the decisions
live **by theme** in `design/decisions/` under stable theme IDs (`TOOL-`/`MOD-`/`FN-`/…),
and every open question from the design phase is closed. Each chapter carries
`Status — draft (under review)` — the draft is accepted for review but **not yet
"accepted"**. The quality bar is **`FND-3`**: the
specification is normative and **unambiguously implementable** (two independent
implementations produce compatible results without guessing).

## How to make changes (process)

- **Decide first, record second.** Contentious / design forks are discussed to
  convergence and **only then** written down. Do not silently "work them out on the fly".
- **One decision = one commit.** When changing a norm, synchronously update every
  affected source of truth (invariants / decisions / glossary / aspects-map + the
  chapters themselves) — coherence is held precisely by these cross-cutting passes.
- **Refine in place (pre-v1).** While the spec is a draft under review, a change to a
  *not-yet-shipped* decision is recorded by **editing its theme entry** in
  `design/decisions/` to the current truth, with a concise `Rejected:` / `Superseded:`
  note. Entries carry the **current** rationale plus *why*, not full history (git holds
  it). IDs are **theme-prefixed and append-only** — the next memory entry is `MEM-9`, never
  a reused number: add a **new entry** for a genuinely new decision area; never reuse or
  shift an existing ID. Keep its
  **`provenance:`** line (what the entry refines, and where it lands in `spec/`) current.
  **Guardrail:** never silently repurpose a widely-cited ID's core meaning; supersede with
  a new entry instead.
- **Minimalism (`PRIN-1`):** do not introduce what is expressible by existing mechanisms.
  Test question: "does this give something beyond what exists, or merely save names?" If
  the latter — reject it.
- **Additivity (`PRIN-2`/`FND-6`/I10):** v1 is fully specified; "additive" is only about *future*
  growth (new instructions/types/features without breaking v1 programs), **not** a
  license to leave v1 underspecified.
- **Fix** any inconsistency found between sources, do not work around it.

## Language and formatting

- **English is the repository's language** — the specification (`spec/`), the design
  record (`design/`), and all docs are written in English.
- **ASCII letters only** in identifiers, labels, and enumerations. Typographic characters
  (`—`, `§`, `→`, `≤`, `…`) are deliberate and fine; Cyrillic look-alikes (`А`, `В`, `С`)
  inside otherwise-Latin text are a defect — invisible to reader and to `grep` alike.
- **Spec file numbering:** filename × 10, natural-sort; **the order of presentation is
  per Overview §7**, not literally by file number.

## Key conventions that are easy to break

- **Invariants are not recommendations.** I1 (transparent codegen), I2 (zero-cost), I3
  (nothing hidden), I11 (no UB: checked — correct-or-trap; unchecked/raw —
  hardware-defined, never "whatever suits the compiler").
- **The verification axis is `checked`/`unchecked`** (not safe/unsafe). `unchecked` is a
  *scoped verification mode* (`CG-6`/`CG-7`): inside `unchecked expr` / `unchecked { … }` the
  checked-guard family is dropped and raw operations become writeable; the transitive
  prohibition is the `no_unchecked` limit.
- **One language + orthogonal limits** (`no_abstractions`/`no_alloc`/`freestanding`/…);
  there is no ladder of "levels" (`PRIN-7`, `FND-9`–`FND-11`).
- Function results are via `-> T` (single anonymous) or named `out`; pointers are `ptr(T)`; pointer ops are the flat
  word-functions `ptr`/`deref` (layout `T.size()`/`T.align()`); built-ins are prelude identifiers (`@` is for attributes only,
  not intrinsics).

## What NOT to do

- Do not add an implementation / compiler code / build into this repository.
- Do not change the specification without synchronizing the decisions and the aspect map.
- Do not "improve" the language with a new entity if it is expressible by existing means
  (`PRIN-1`).
- Do not leave discrepancies between sources of truth "for later".
