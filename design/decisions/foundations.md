# Decisions — Foundations: methodology, conformance, limits, niche

**Why the language is the way it is** for the foundational layer: how the design is *made*
(methodology and process), what the language *is and is not for* (niche and non-goals),
how a program declares *which subset* it conforms to (the limits model), what counts as
*configuration vs fixed meaning*, and the bar the specification itself must clear. Each
entry records the *rationale* and the *rejected alternatives*; the normative *what* lives
in the spec (linked per entry), the *how* in the compiler. IDs are theme-prefixed
(`FND-N`) and **append-only** — assigned in articulation order, never renumbered. The
`provenance:` line records what an entry refines and where it lands in `spec/`.

---

## Methodology & process

### FND-1. Spec-first, aspect-by-aspect, decide-then-write
*spec: Overview §7; design/invariants.md*

**Why.** The most expensive mistakes in a language are the irreversible ones — the
foundation, semantics, syntax, the basic primitives (FND-6). The cheapest place to undo
them is *before* prose, let alone code, exists. So the order is: work an aspect through to
a decision, **then** write the chapter — all aspects are discussed to convergence before
chapter prose begins, to minimize rewriting. The design order (driven by dependencies)
and the exposition order (the book's structure) are deliberately distinguished. Coherence
is held by a small set of cross-cutting artifacts kept in sync — invariants, glossary,
the thematic decision entries, the aspect map — and the working language throughout is
**English**.
The chosen format is a set of loosely-coupled chapter-files rather than one continuous
book, which is easier to keep coherent and to reference.

### FND-2. Prior work is a candidate foundation, not dogma
*spec: design/invariants.md*

**Why.** Earlier design work — this project's own and other languages' — is a source of
proven ideas and, just as valuably, a record of which decisions turned out *expensive*.
It is mined for both, and it settles nothing: **every aspect is re-derived from scratch
rather than inherited**, so an engineering lesson can be kept while the language model
that produced it is rejected. A decision is never justified by "that is how it was done
before"; it is justified here, in its own entry, or it does not stand.

### FND-3. The specification bar: normative, unambiguously implementable
*spec: Overview §6; design/invariants.md*

**Why.** The point of `spec/` is an **implementation-complete normative specification**:
two independent developers must be able to build **compatible** compilers **without
guessing** — the level of a real language standard. This is the supreme quality bar for
the whole repository, consistent with lowering-rules-as-normative-content and with
transparent codegen (I1). Concretely, every chapter that *defines language constructs*
must give all of: the exact syntax (EBNF + lexis); the static semantics (name resolution,
typing, the checks — mutability / linearity / definite-assignment / limits /
checked-guards, i.e. what exactly is an error or valid); the dynamic
semantics/lowering (the exact rules for lowering each construct to GAS, the per-target
ABI, the layout); the v1 content listed **exhaustively** (per-arch instruction tables,
per-target ABI, prelude/stdlib definitions); and a conformance statement (what an
implementation MUST do and what is implementation-defined). Framework/meta-chapters (the
Overview and the like) are exempt from the construct-defining requirements — they set
context plus the normative invariants/conformance but define no constructs. A working
consequence of writing the rules out at this fidelity is that small gaps surface and are
closed as design refinements as we go (this is how the bar itself, and the verification
model, came to be recorded).

---

## Niche & non-goals

### FND-4. The niche — the bottom floor with transparent abstractions
*spec: Overview §1–§2, §5; design/invariants.md*

**Why.** Alatyr is a systems language of the **"bottom floor"**: an expressiveness
ceiling equal to the assembler's — everything expressible in the target's lowest form
(GAS for register ISAs, WAT for WASM) is expressible in the language — but with powerful
abstractions that are **transparent in codegen (I1)** and **zero-cost (I2)**, governed by
explicit programmer directives or by conventions. The guiding promise is that the
programmer always knows what the machine does; the lineage is Zig/Rust/Ada/D/C. The
**primary niche** is low-level (kernels, firmware, bootloaders, bare-metal, drivers,
hypervisors, allocators, codecs, crypto, DSP, hot paths); the **secondary** is application
code that values predictability. The audience is programmers who want assembler control
*plus* modern ergonomics *plus* known cost. The pillars are the invariants I1–I11.

### FND-5. Non-goals — the deliberate boundaries
*spec: Overview §5; design/invariants.md*

**Why.** A language is defined as much by what it refuses as by what it offers; naming the
boundaries keeps the design from drifting into a general-purpose pile of features. Alatyr
deliberately does **not**: (1) use LLVM or an opaque optimizing backend — auto-optimization
is *semantically-preserving only*, predictability outranks peak optimization, and any
behavior-changing transform is opt-in; (2) have a managed runtime / GC / implicit
refcount / RTTI; (3) have hidden control flow or implicit RAII destructors — cleanup is
explicit (`defer`); (4) exploit or permit undefined behavior at all (I11); (5) sacrifice
transparency for convenience (no magic that hides cost); (6) abstract the machine away
(I6) — the machine model is visible, this is not a portable VM; (7) drag in a
borrow-checker / mut-XOR-shared — the simpler answer is scoped references / regions /
handles; (8) chase type theory for its own sake — comptime types are pragmatic (no
dependent types / HKT for their own sake); (9) break compatibility within a frozen version
— stability plus additive growth; (10) be dynamically typed or interpreted — it is static,
AOT-compiled to native code via GAS.

---

## Completeness vs additive growth

### FND-6. Completeness now vs additive growth later
*spec: Overview §6; design/invariants.md (I10)*

**Why.** The usual "build it when a second consumer appears" test does not apply to a
language: programmers do not extend the language, they are *bounded* by its set, so the
right test is **"does adding it later break existing code or force a rewrite?"**.
Everything **irreversible** — the foundation, semantics, syntax, the basic primitives of
abstraction, the expressive surface — must be **complete and correct now**; this is the
very reason to write a specification before writing code. Only the **additive periphery** —
new instructions, stdlib functions, target features, additional *non-semantic* strategies
— may wait, because it can ship later without breaking existing programs. A direct
corollary, load-bearing across the whole spec: **"additive" never means "left
underspecified in v1"** — the v1 content (instructions, ABI, prelude) is listed
exhaustively (see FND-3); additivity is purely about *future* growth.

---

## Configuration vs fixed meaning

### FND-7. What is configurable vs what is fixed semantics
*spec: Overview §6; design/invariants.md*

**Why.** A program's *meaning* must not change between packages or build sites, but its
*cost and representation* legitimately can. So the two are separated. **Non-semantic
policies** (those that change cost/representation but not the meaning of the program) are
configured by a single tiering: **compiler default → manifest (package-level,
reproducible) → per-site override**; the CLI is only a *temporary* override, because a
package depending on a CLI flag would be non-self-sufficient. **Semantic rules** (type
identity, conversion rules, mutability, …) are **not** settings — they are fixed by the
language, otherwise one source could change meaning between packages. A borderline case
sharpened this: the **verification axis** (`checked`/`unchecked`) changes behavior, so it
is *semantic* — `checked` is a fixed default, **not** overridable by the manifest (a global
toggle is dangerous), and `unchecked` is a lexical grant rather than a configuration knob.
The set of profiles/strategies itself grows *additively* (FND-6): the tiering *mechanism*
is laid in now, concrete non-semantic strategies are added later without breaking
compatibility.

### FND-8. The manifest — configuration phase and machine-model provider
*spec: Overview §3; manifest appendix (140); design/invariants.md (I6)*

*(Refined by TOOL-18: the machine model is stated as a **`Machine` variant** — one variant per platform,
holding that platform's own fields — not as flat `arch`/`os`/`env`/`container` fields; endianness follows
the arch rather than being a settable field, and `target.os`/`target.container`/`target.endian` are
projections of the variant.)*

**Why.** The manifest is the **configuration phase** of compilation. Beyond build
configuration it fixes the **machine model** — bit widths, registers, endianness, target
features — which is the input the type system depends on (I6). It is the **same Alatyr**,
evaluated first (one grammar/AST, a data subset of the language), and forms the `Config`
diagnostic stage. Making the manifest the single, source-visible place where the machine
model is pinned keeps the type system's foundation explicit and reproducible rather than
ambient.

---

## The conformance / limits model

### FND-9. One language, single frontend
*spec: Overview §3, §6; design/invariants.md*

**Why.** There is **one language** with a single lexer/parser/AST. Raw
instructions/labels, structured constructs, and comptime **coexist by default** and may be
freely mixed; the original idea of a *ladder of superset levels* was removed (see
FND-10). A single frontend avoids the grammar-synchronization and inline-nesting seams
that a multi-grammar design creates.

**Rejected.** Three separate grammars/parsers with AST→AST lowering — it spawns
grammar-synchronization and inline-nesting seams, and every construct that nests one
level inside another has to be re-specified at each seam.

### FND-10. No tier ladder — one language plus orthogonal limits
*spec: Overview §6; design/invariants.md*

**Why.** The linear ladder `assembly ⊂ functional ⊂ comptime` was removed because it
**linearized orthogonal dimensions** and broke real cases — for instance it forbade
comptime at the assembler level, yet comptime is needed *everywhere* (constants, tables,
unrolling, target-gating) and is *erased*, so it never touches the runtime contract.
Instead, guarantees are expressed by **orthogonal opt-in limits** in a **subtractive
model**: by default everything is available, and a limit *forbids*. `no_abstractions`
(asm-only: only 1:1 GAS correspondence, no structured abstractions — for boot/critical
paths; comptime is still permitted, being erased; its precise construct-by-construct
allow-list is **CG-12**) is just **one limit, not the bottom of a
ladder**; `no_alloc`, `freestanding`, optionally `no_comptime`, etc. combine freely
(`no_abstractions + no_alloc`, `no_alloc + freestanding`) in ways the linear ladder could
not. "The assembler level" is simply the `no_abstractions` limit; a tier was always a
special case of a limit. The conformance term is **"limits"** (the word "profile" was
rejected as overloaded by build-profiles). *Incremental implementation* — the reason the
ladder was introduced — is preserved as the *compiler's* concern, not a property of the
language; dialect-subsets are expressed via limits. Earlier mentions of "tier/level"
throughout the design read as "raw assembler constructs vs structured — in one language;
restrictions via limits; comptime everywhere".

**Rejected.** The linear tier ladder (it conflated orthogonal axes and broke
comptime-at-asm); "profile" as the term for a conformance unit (collides with
build-profiles).

### FND-11. Declaring conformance — one extension, the `limits` contract
*spec: Overview §6; manifest appendix (140); design/invariants.md*

**Why.** Conformance is an **explicit contract**, not something encoded in a filename.
There is **one extension `.al`**: the earlier idea of numbered `.alN` extensions encoding
levels is cancelled, because conformance is a contract (the limits) and not a file name.
The contract is declared through the standard tiering (FND-7): the **manifest** carries a
`limits:` field (the package default/ceiling), and a **file** may tighten it with a single
extensible **`@limits(...)`** attribute (stricter, ≤ the ceiling) — e.g.
`@limits(no_alloc, freestanding)` — chosen over a scatter of per-limit attributes because
one extensible list adds a new limit as just another token and stays uniform with the
other value-carrying attributes. The language name **`Alatyr`** (from the Slavic *Alatyr*
— the cornerstone, the white-burning foundation-stone) is the part of FND-11 that survives;
it resonates with the "language-foundation / transparent bedrock" thesis (I4). Growth here
is additive (FND-6): future per-function effects (over the same capability vocabulary) and
full comptime-emission of declarations are refinements within the existing language, not
built now — fine-grained per-function effects would require effect-polymorphism
(virality/coloring), which is why they are deferred.

**Rejected.** Encoding limits in the file extension / numbered `.alN` (conformance is a
contract, not a filename); HKT and eager fine-grained per-function effects (deferred —
they need effect-polymorphism with its virality/coloring cost).

### FND-12. Locality of the translation-unit contract
*spec: Overview §6; design/invariants.md*

**Why.** A unit's guarantee (its limits) is **local to the translation unit**; the
cross-unit join is an **explicit interface/ABI** — the same way `freestanding` and
`hosted` C meet at link time. Units with different limits **coexist at the symbol seam**,
never by intermixing inside one another. This keeps each unit's contract checkable on its
own and makes the boundary between, say, a `no_alloc` unit and an allocating one a visible,
specified ABI surface rather than a silent blend.
