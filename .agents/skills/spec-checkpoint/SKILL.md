---
name: spec-checkpoint
description: >-
  Generate a compact "mental-model checkpoint" from the CURRENT Alatyr
  specification into a gitignored `*.tmp` folder — a dense, ≤30-minute read that
  lets a programmer confirm whether their understanding of the language matches the
  spec. Not a tutorial: it is organized as falsifiable claims with explicit
  is/isn't contrasts and a surprise-zone, so the reader spends time on
  disagreements, not on the obvious. Use when asked for a "mental-model check",
  "spec checkpoint", "sanity-check set", "concept digest", or "does my view match
  the spec". Output-only: writes the folder and reports; never edits the spec,
  never commits, never modifies itself. Fragments are teaching claims derived from
  a draft-under-review spec, not verified programs (the reference toolchain is
  still in development, so nothing here is implementation-validated).
---

# Spec mental-model checkpoint

This is **spec-companion meta-tooling**, NOT the Alatyr toolchain and NOT part of
the normative spec. It reads the specification and emits a *bounded, high-signal*
digest whose single purpose is **calibration**: a programmer reads it in under half
an hour and, claim by claim, can say "yes, that matches how I think" or "no — my
view diverges here." It does not teach the language; it tests the reader's model
against the spec.

## What it does, in one line

Read the current `spec/` (+ `design/`) and write a checkpoint folder — a short
README that sets the reading mode, one dense mental-model document (invariants →
per-aspect claims → surprise-zone → divergence checklist), and a one-page syntax
cheat-sheet — every claim falsifiable, contrast-driven, and traceable to a chapter.

## Design intent (read this before generating)

The artifact succeeds only if a reader can **detect disagreement fast**. So:

1. **Claims, not lessons.** Each aspect is one short, falsifiable statement of how
   the language decides it — phrased so the reader can immediately agree or balk.
   Not "here is how memory works" but "memory is value/place/binding/address; a
   value need not have an address; mutability is AND-ed along the whole path."
2. **`X`, not `Y` contrasts are the primary tool.** The fastest calibration is the
   explicit negation: what the reader might assume that is *wrong*. Every claim that
   has a common wrong mental model carries a `— not Y` / `— ожидают иначе` line.
3. **A surprise-zone.** Collect the 8–12 most counter-intuitive decisions in one
   place, each flagged and chapter-tagged. This is where divergence is likeliest, so
   it is where the reader should slow down — not on the obvious claims.
4. **Falsifiable + traceable.** Each claim points to its chapter (so a surprised
   reader knows where to go), but the prose does not cite decision/invariant IDs — this
   is a digest, not the spec.
5. **Depth via consequences, not volume.** Make claims bite on the non-obvious
   *implications* (linearity replaces RAII-drop; `unchecked` is a grant not a mode;
   a profile is non-semantic; conformance is rule+cost, not byte-identity), not by
   adding sections. One sharp example per idea.
6. **Time budget is a hard constraint.** Target a ≤30-minute read end to end
   (~2500–3500 words + micro-fragments total across all files). If it grows past
   that, cut claims or tighten prose — never pad. Cover *all* aspects, one crisp
   claim each.

## Operating contract (hard guardrails)

1. **Output-only, into `<target>.tmp/`.** Default folder: `checkpoint.ru.tmp/` at the
   repo root (override via argument, e.g. `out=check.tmp`). The `.tmp` suffix is
   **required** — `.gitignore` already ignores `*.tmp`, so the pack never enters git
   history. **Never** write under `spec/` or `design/`.
2. **No spec edits, no commits, no push.** If a claim you are drafting exposes a real
   spec defect or ambiguity, **do not fix it here** — note it in the report and
   suggest `/coherence-audit`.
3. **No self-modification.**
4. **Confirm before overwriting human work.** If the target folder exists and is
   non-empty, it may hold manual edits — say so and confirm before regenerating.
5. **Honest framing, every time.** The README must state: claims are derived from a
   **draft-under-review spec** that no toolchain has yet been run against, so fragments
   are illustrative, not verified programs. "Consistent with the spec as read", never
   "compiles".

## Sources of truth (what each claim is checked against)

Repo priority order (`AGENTS.md`). For this skill specifically:
- **Form of every fragment → `spec/130-grammar.md`** — a fragment must be derivable
  from the grammar; reject anything that isn't.
- **Meaning → the feature chapter** for its area.
- **Invariants `I1–I11` (`design/invariants.md`)** are the worldview section verbatim
  in spirit — get them right; they are the supreme norm.
- **The "easy to break" conventions in `AGENTS.md`** are the accuracy
  checklist below — screen every claim and fragment against it.

## Output language

Default **Russian** (the reader is a Russian-speaking programmer; the pack lives in a
gitignored `*.tmp` folder, so the repo's English-only rule for `spec/`+`design/` does
not apply). Override with `lang=en`.

## The checkpoint plan (stable; this is the one maintenance point)

Fixed, small file set. Group by *purpose*, not by chapter number; degrade gracefully
if chapters are renumbered.

| File | Contents | Built from |
|------|----------|-----------|
| `README.md` | how to read it (at each claim ask "did I already think this?"), the ≤30-min budget, the honest-framing note, reading order | Overview `00` + this skill's intent |
| `00-mental-model.md` | **(A) Worldview** — `I1–I11`, one line each. **(B) Per-aspect claims** — ~30 falsifiable claims spanning all 14 chapters, each: a one-line claim · a micro-fragment where it helps · a `— not Y / ожидают иначе` line where there is a common wrong model. **(C) Surprise-zone** — the 8–12 most counter-intuitive decisions, flagged, chapter-tagged. **(D) Divergence checklist** — "mark what surprised you → which chapter to read" | all chapters; grammar `130` for form; `invariants.md` for (A) |
| `90-cheatsheet.md` | one page: how each construct is *spelled* — declarations, functions/`-> T`/`out`, types/`bitsN`, `mem::*`, control/`match`/`?`/`@label`, comptime/`when`, modules `::`/`.`, attributes `@`, manifest `package.al` | grammar `130` + feature chapters |

## Aspect coverage (each gets exactly one crisp claim in §B — do not skip one)

composition & limits · declarations vs assignment (no `const`) · types & `bitsN` &
conversions · memory model (value/place/binding/address, mutability path, `@alloc`,
`defer`, linearity, no RAII) · functions (result `-> T` / named `out`, directions, ABI-as-value, naked) ·
control flow (`if`/`match` expr, exhaustiveness, `?`, `and/or/not`, `@label` two
kinds, raw transfer needs `unchecked`) · comptime & generics (erasure,
type-functions, predicate constraints, `when`, no RTTI) · modules (`::` vs `.`, no
`use`, pub-chain, `@export`/`@extern`) · assembly (1:1 intrinsics, `asm(...)` `{i}`,
inline-everywhere, I4) · codegen (backend's lowest form — GAS→`as`→`ld` for a register
ISA, WAT for WASM (CG-4); no-LLVM but a semantically-preserving optimizer (CG-5, not "no
optimizer"); conformance = rule+cost not bytes, byte-exact only under `no_abstractions`) · stdlib (tiers
base/alloc/std, built-ins are prelude names not `@`, `panic`/`exit`/`assert` defined
failure, `Never`) · concurrency (no runtime, atomics/`volatile` as builtins on
`ptr`, race-freedom by construction, `async` reserved) · overflow & float
(trap-set, `wrapping`/`saturating`/`checked`/`overflowing`, float determinism) ·
tooling (manifest = single `Package` value in `package.al`, no `Package.name`,
`targets`+`default_target`, profiles non-semantic, reproducible builds).

## Accuracy checklist (screen EVERY claim and fragment)

The conventions most likely to drift — a claim/fragment violating any is wrong:

- A single anonymous result is **`-> T`**; **named/multiple** results are `out r : T`
  parameters — **never both**, and there is **no** anonymous `out T` form (FN-4). `Never`
  is the divergence type (`-> Never`; no `@noreturn`).
- Pointers **`ptr(T)`** / `ptr(mut T)`; `ptr(x)` / `deref(p)`;
  `T.size()` (no `sizeof`).
- Axis **`checked`/`unchecked`/`no_unchecked`**, never `safe`/`unsafe`. `unchecked` is
  a *grant* to write raw ops, not a mode flip (ordinary ops inside stay checked).
- Declaration `name:T=value` / `name:=value` / `name:T`; **assignment** is only
  `place=value`. **No `const`** — a constant is a `comptime` binding.
- **`@` is attributes only**; built-ins are prelude identifiers (`bitcast`,
  `T.size()`, `atomic::load`, `volatile::store`, `asm`, `embed`, …).
- **`::`** namespaces/paths; **`.`** values/fields/UFCS. **No `use`** — import/alias/
  re-export is a declaration `name := path` / `pub name := path`.
- Labels are **`@label(name)`**, two kinds (code-point `jmp` target / structured
  `break`/`continue` target). `?` is postfix try via the tryable protocol (not by the
  names `Option`/`Result`). Raw control transfer in structured code needs `unchecked`.
- Core type **`bitsN`**; `u`/`i`/`f` are interpretations; reinterpret is `bitcast`;
  narrowing traps by default.
- Owning values are linear (`defer` consumes on exit; `forget` to leak deliberately);
  no implicit RAII-drop.
- Atomics/fence/`volatile` are **builtin calls on `ptr`** (comptime `Ordering`);
  **no `Atomic(T)` type**.
- Overflow: `+ - *` (and unary `-`, `MIN/-1`) **trap** by default everywhere; wrap only
  under an unchecked op; `wrapping`/`saturating`/`checked`(→`Option`)/`overflowing`(→`(T,bool)`).
- Manifest is **`package.al`**: a single `Package` value in the anonymous package-root
  module (data-subset of Alatyr). **No `Package.name`**; the allowed set is **`targets`**
  + **`default_target`**. Profiles are **non-semantic by themselves**.

## Procedure

0. **Load context.** Read `spec/00-overview.md`, `spec/130-grammar.md`,
   `design/invariants.md`, and skim each feature chapter for its one defining
   decision. Read `AGENTS.md` key conventions.
1. **Resolve target + language** from the argument (defaults: `checkpoint.ru.tmp/`,
   Russian). If the folder exists and is non-empty, **confirm overwrite** first.
2. **Draft `00-mental-model.md`.** Write (A) the invariants, then (B) one claim per
   aspect from the coverage list — each falsifiable, with a micro-fragment and an
   is/isn't contrast where a wrong model is common; then (C) pick the 8–12 genuinely
   counter-intuitive decisions for the surprise-zone (chapter-tagged); then (D) the
   divergence checklist. Screen every claim/fragment against the accuracy checklist
   and derive each fragment's syntax from `130-grammar.md`.
3. **Draft `90-cheatsheet.md`** — terse "how it's spelled" reference, grammar-faithful.
4. **Write `README.md` last** — reading mode, ≤30-min budget, honest-framing note,
   order; navigation must match what was produced.
5. **Budget + self-check pass.** Confirm the whole pack reads in ≤30 minutes (trim if
   not). Re-read each fragment adversarially against the checklist and grammar;
   rewrite any that drift. Flag any claim you could not make fully spec-faithful (and
   why) instead of shipping a guess.
6. **Report.** Files written (path + one-line contents), the chapters drawn from, an
   estimated read time, any spec ambiguities found (candidates for `/coherence-audit`,
   not fixed here), and the coverage/limits declaration: a bounded calibration slice
   of a draft-under-review spec; fragments illustrative, not compiled.

## Notes

- Generation must be faithful to the *current* spec, so the checkpoint refreshes as
  the spec evolves. Do not hardcode example code here; keep this a thin, stable
  procedure. The one maintenance point is the checkpoint-plan table + aspect-coverage
  list.
- Complements `/coherence-audit`: that keeps the spec self-consistent; this projects a
  fast-calibration surface out of it. Neither edits the normative sources.
