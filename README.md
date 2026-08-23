# Alatyr

**A systems programming language whose ceiling of expressiveness equals the assembler's —
with abstractions that stay transparent in code generation.**

![spec version 1.0.0-draft.1](https://img.shields.io/badge/spec-1.0.0--draft.1-blue)
![status: draft under review](https://img.shields.io/badge/status-draft%20under%20review-orange)
![license CC BY 4.0 / Apache-2.0](https://img.shields.io/badge/license-CC%20BY%204.0%20%2F%20Apache--2.0-green)

This repository is the **specification** of Alatyr. It is not a compiler, and it contains
no implementation code: the reference toolchain and the standard library are developed in
[sibling repositories](#the-alatyr-repositories) and are built *against* this text. What
is maintained here is a living, versioned, normative document, held to one bar:

> **Two independent toolchains built strictly to this specification produce compatible
> results without guessing.**

That bar (`FND-3`) is why the repository looks the way it
does: every rule has a recorded reason, every term has one definition, and a
contradiction between two files is a defect to be fixed rather than a matter of opinion.

---

## Status

**Version `1.0.0-draft.1` — the complete first draft of the Alatyr v1 specification,
under review. Not accepted.**

All 14 narrative chapters and all 4 appendices are written. The invariants `I1–I11` are
recorded, the design decisions are organized by theme under stable IDs, and every open
question from the design phase is closed. Every chapter carries
`Status — draft (under review)`.

What that means in practice:

- **The language is fully specified, not sketched.** v1 is enumerated down to the ABI
  tables and the per-architecture data; "additive growth" governs the *future*, and was
  never a licence to leave v1 vague.
- **The reference implementation is in development**, in the sibling
  [`compiler`](https://github.com/alatyr-programming-language/compiler) repository, with
  the standard library in
  [`stdlib`](https://github.com/alatyr-programming-language/stdlib). It has not yet been
  run against the whole of this text, so nothing here is implementation-validated: the
  code fragments in the chapters illustrate the rules, they are not tested programs.
  Implementation experience is exactly what the review phase is waiting for — it flows
  back here as refinements.
- **The text can still change anywhere.** While the `-draft` suffix is present, the
  between-release additivity guarantee in `CHANGELOG.md` does not yet hold; pre-v1
  decisions are refined in place.
- **Review is the work in progress**, and the most useful contribution is a precisely
  located contradiction or gap. See [`CONTRIBUTING.md`](CONTRIBUTING.md).

---

## What Alatyr is

**Niche.** Operating-system kernels, firmware, bootloaders, bare-metal and embedded
targets, drivers, hypervisors, allocators, codecs, cryptography, DSP and vector kernels,
and performance hot paths — anywhere a programmer would otherwise drop to C or to hand
assembly, but wants stronger abstractions without giving up control. Secondarily,
application code that values predictability and the absence of hidden cost.

**Ceiling.** Everything expressible in the target backend's lowest form — GAS for a
register ISA, WAT for WASM — is expressible in Alatyr (`CG-4`). Raw instructions, labels
and jumps, structured constructs, and compile-time metaprogramming all belong to *one*
language and may be mixed freely in a single translation unit.

**Backends.** Codegen is per backend and goes through the backend's own lowest form: GAS
text, then `as` + `ld`, for the register ISAs — **not LLVM** — with a structured/VM
backend such as WASM (→ WAT) as an additive one. Six v1 architectures: `x86_64`, `i386`,
`aarch64`, `aarch32`, `riscv64`, `riscv32`, over a closed supported-target set.

**Influences.** Zig, Rust, Ada, D, C.

### What it deliberately is not

No LLVM or opaque optimizing backend; no managed runtime, GC, implicit refcounting, or
RTTI; no hidden control flow and no implicit destructors (cleanup is explicit `defer`);
no undefined behavior, not even admitted; no borrow checker or mutable-XOR-shared
discipline; no hidden portable virtual machine of its own. Full list with reasoning:
[`spec/00-overview.md`](spec/00-overview.md) §5.

---

## What it looks like

A declaration is one form — a name bound to whatever its right-hand side evaluates to.
There is no separate keyword per kind of thing:

```
Point   := struct { x : u64, y : u64 }
Meters  := brand(u64)
add     := fn(a : u64, b : u64) -> u64 { a + b }
Vec     := fn(T : type) -> type { struct { ptr : ptr(T), len : usize } }
comptime PI : f64 = 3.14159
mut count : u64 = 0
```

`fn(…){…}` is a value-expression exactly as `struct{…}` is; a generic is just a function
with comptime parameters, monomorphized per argument set. Three modifiers, on three
orthogonal axes: `pub` (visibility), `mut` (mutability), `comptime` (phase).

Configuration is the manifest — written in the language's own lexis and grammar and
evaluated as pure, host-independent comptime data, not a foreign format. The file
`package.al` **is** the package-root module, so manifest and code can live in one file:

```
app := Package(
  version = "0.3.0",
  targets = [
    Target(machine = Machine.Bare(arch = Arch.aarch64),
           kind = Kind.executable, entry = "_start", output = "tiny-kernel",
           features = ["sve"]),
  ],
  limits  = [Limit.freestanding, Limit.no_alloc],
)
```

*(Both fragments are taken from the specification —
[`spec/40-declarations.md`](spec/40-declarations.md) §1.3 and
[`spec/140-appendix-manifest.md`](spec/140-appendix-manifest.md) §6.)*

---

## The design in brief

- **One language, orthogonal limits — no tier ladder** (`PRIN-7`, `FND-9`/`FND-10`).
  Guarantees are opted into per translation unit through independent limits —
  `no_abstractions`, `no_alloc`, `freestanding`, `no_unchecked`, `no_comptime`, `no_opt` —
  declared in the manifest as a package-wide ceiling and tightened, never loosened, per
  file. "Assembly only" is not a lower tier of the language; it is the `no_abstractions`
  limit.
- **No undefined behavior** (`I11`). Operations are `checked` by default: correct, or a
  deterministic trap. Inside an `unchecked` scope they are **hardware-defined** — never
  "whatever suits the compiler". The verification axis is `checked`/`unchecked`, not
  safe/unsafe, and `unchecked` is a *scoped mode* (`unchecked expr`, `unchecked { … }`),
  not a property of a declaration. `no_unchecked` is the transitive prohibition.
- **Transparent, zero-cost abstraction** (`I1`, `I2`). Every construct lowers to the
  backend's lowest form by documented rules that are themselves normative content of this
  specification; a construct never bypasses lower constructs to emit machine code
  directly, and an abstraction never costs more than the code it replaces.
- **Nothing hidden** (`I3`). No hidden allocation, no hidden control flow, no hidden
  boxing. A function value's representation follows from what it captures: a code pointer
  when it captures nothing, a monomorphized static closure when captures are statically
  known, an explicit `dyn` fat value over storage you named when it is type-erased.
- **Types are comptime functions** over the machine model. The kernel is raw `bitsN`
  blocks; `u`/`i`/`f` are interpretations of them; generics are monomorphization;
  constraints are comptime predicates. Compile-time values are erased (`I7`).
- **Memory:** immutability by default; linear owning values with explicit `defer`; a menu
  of lifetime mechanisms — scoped references, regions, generational handles — selected at
  the `@alloc(value)` site (`MEM-3`/`MEM-4`). No package-wide default allocator in v1.
- **One output-place model.** A result is either a single anonymous projected `-> T` or a
  named `out`. Pointers are `ptr(T)`; pointer operations are the flat word-functions
  `ptr` / `deref`, with layout via `T.size()` / `T.align()`.
- **Operators are library functions, not keywords** (`PRIN-1`, `OP-1`). Reserved words
  are spent only on what cannot be a function. `+`, `==`, `bitcast`, `panic`, `assert`
  are prelude built-ins, overridable per type, and UFCS gives `f(a, b)` ≡ `a.f(b)` for
  free. `@` is for attributes only — never for intrinsics.
- **Tooling is specified, not left to the implementation.** The manifest, the codegen
  pipeline, reproducible builds, diagnostics, and cross-toolchain behavior are normative
  on a par with the language semantics (chapters `90`, `120`; appendices `140`, `150`,
  `170`).

The 11 invariants — `I1` codegen transparency, `I2` zero-cost, `I3` nothing hidden, `I4`
completeness of the lower level, `I5` one language + limits, `I6` machine model as the
foundation of types, `I7` comptime erasure, `I8` explicitness of loss, `I9` locality of
the unit's contract, `I10` completeness of the expressive surface, `I11` no UB — are in
[`design/invariants.md`](design/invariants.md). **Where a chapter conflicts with an
invariant, the invariant governs and the chapter is a defect.**

---

## Reading the specification

Chapters are loosely coupled and cross-reference each other by anchor. File numbers are
`filename × 10` for natural sort; **the order of presentation is the one below**, per
Overview §7 — not literally by file number.

| # | Chapter | What it settles |
|---|---------|-----------------|
| [00](spec/00-overview.md) | Overview | Thesis, niche, model, **invariants (normative)**, non-goals, **conformance** |
| [10](spec/10-memory.md) | Memory model | Values, places, bindings, addresses; storage; mutability; pointers; lifetimes; closures; copy/move and linearity |
| [20](spec/20-types.md) | Type system | The `bitsN` kernel and numeric interpretations; type construction; aggregates and layout |
| [30](spec/30-comptime.md) | Comptime and generics | The comptime/runtime boundary; monomorphization; constraints as predicates; introspection |
| [40](spec/40-declarations.md) | Declarations and bindings | The unified `name : type = value` form; scope, shadowing, order |
| [50](spec/50-functions.md) | Functions, parameters, ABI | Parameter directions and result projection; calling conventions; passing modes; defaults, named args, variadics |
| [60](spec/60-control-flow.md) | Control flow | Raw labels and jumps; `if`/`match`/loops/`break`/`continue`/`return`; exhaustiveness |
| [70](spec/70-modules.md) | Modules, visibility, packages | The module tree; `pub`; import and re-export by declaration; mangling |
| [80](spec/80-assembly.md) | Assembly correspondence | The instruction model; operands and registers; data definitions; the architecture matrix |
| [90](spec/90-codegen.md) | Codegen and lowering | The pipeline; lowering rules as the `I1` contract; determinism; GAS emission; containers |
| [100](spec/100-stdlib.md) | Standard library and built-ins | Prelude tiers; intrinsics; `panic` / `exit` / `assert`; allocators |
| [110](spec/110-concurrency.md) | Concurrency, overflow, float | Atomics, fences, `volatile`; integer-overflow semantics; floating-point determinism |
| [120](spec/120-tooling.md) | Tooling | Manifest schema; the CLI; diagnostics; cross toolchains; reproducible builds |
| [130](spec/130-grammar.md) | Grammar | The complete EBNF and lexis |
| [140](spec/140-appendix-manifest.md) | Appendix — manifest | The manifest schema's enumerated v1 content |
| [150](spec/150-appendix-abi.md) | Appendix — ABI | ABI per target |
| [160](spec/160-appendix-stdlib.md) | Appendix — prelude/stdlib | The enumerated prelude and standard library |
| [170](spec/170-appendix-arch.md) | Appendix — per-arch | Tables for all six architectures; the closed supported-target set |

A **feature chapter is normative for the semantics** of its area; the **Grammar chapter is
normative for syntax**. Where form and meaning diverge, the feature chapter governs the
meaning. Enumerated v1 *data* lives in the appendices; the *schema* for it lives in the
narrative chapter that owns it.

**Suggested paths.** To evaluate the language: Overview, then Memory, then Types. To
implement it: Overview §6 (conformance), Grammar, Codegen, then the appendices. To argue
with a decision: find its theme entry in `design/decisions/` first — the alternative you
have in mind is usually recorded there under `Rejected:`.

---

## Repository layout

```
spec/                      the normative specification — 14 chapters + 4 appendices
design/
  invariants.md            I1–I11 — the supreme norm
  decisions/               why the spec is this way, by theme, under stable IDs
    principles.md            cross-cutting priorities (PRIN-*)
    README.md                navigation across the themes
    memory.md types.md …     one file per area (MEM-*, TYP-*, FN-*, CG-*, TOOL-*, …)
  glossary.md              canonical terms — one definition each
  aspects-map.md           per-aspect status and the deferred (additive) growth list
AGENTS.md                  orientation and process canon (CLAUDE.md is a symlink to it)
.agents/skills/            repo-specific agent skills (coherence audit, spec checkpoint)
CONTRIBUTING.md            how to file a defect or a gap, and the PR checklist
CHANGELOG.md               the versioning policy and the release history
CODE_OF_CONDUCT.md         Contributor Covenant 2.1
LICENSE                    the dual-licensing notice (+ the two full licence texts)
.github/                   issue templates (defect / gap / proposal) and the PR checklist
```

### Sources of truth, in priority order

On any conflict the **higher** source governs, and the lower one is a defect to fix — not
a second opinion to balance.

1. `design/invariants.md` — the invariants `I1–I11`
2. `spec/` — the normative specification
3. `design/decisions/` — the recorded decisions and their rejected alternatives
4. `design/glossary.md` — canonical terms
5. `design/aspects-map.md` — the aspect map and status

This README mirrors those sources; it never replaces them.

**Decision IDs.** Decisions are theme-prefixed and append-only (`MEM-3`, `CG-4`,
`FND-3`, …), one file per theme, indexed by
[`design/decisions/README.md`](design/decisions/README.md). An ID is never reused,
shifted, or repurposed; a decision that changes is superseded by a new entry.

---

## The Alatyr repositories

| Repository | What it holds | Status |
|---|---|---|
| [`spec`](https://github.com/alatyr-programming-language/spec) (this one) | The normative specification and the design record behind it | `1.0.0-draft.1` — complete first draft, under review |
| [`compiler`](https://github.com/alatyr-programming-language/compiler) | The reference toolchain: front end, lowering, GAS / WAT emission, the CLI | In development |
| [`stdlib`](https://github.com/alatyr-programming-language/stdlib) | The prelude and standard library specified in chapter [100](spec/100-stdlib.md) and appendix [160](spec/160-appendix-stdlib.md) | In development |

The direction of authority runs one way and does not reverse: **the toolchain conforms to
the specification, never the other way round.** When the implementation disagrees with
this text, that is a bug in the toolchain — or, just as often, a defect or a gap here. In
the second case it is fixed *here*, as a recorded decision with its reasoning, and only
then followed downstream. An implementation detail never becomes normative by having been
shipped.

That feedback loop is the reason this specification is called *living*: it does not stop
when the compiler works.

---

## For implementers

Conformance is defined by [`spec/00-overview.md`](spec/00-overview.md) §6, and it is a
factual claim about behavior — no licence grants it, and the reference toolchain does not
confer it either: an independent implementation is conforming when it matches *this text*,
not when it matches the reference. Three things worth knowing before you start:

- The specification is a **draft under review**. Cite the version you built against
  (`1.0.0-draft.1`), and expect the text to move under you until the suffix is dropped.
  The leading `1` is the *language* version this document specifies, not a claim that the
  text is settled — see `CHANGELOG.md`.
- If you have to guess, that is a **bug in this repository**, not in your reading. File
  it as an underspecification gap with the case where two conforming implementations
  would diverge — that report is worth more here than a patch.
- Reports from a second implementation are the most valuable kind the review phase can
  get, precisely because the reference toolchain and this text are written by the same
  people and can share a blind spot.

Code fragments may be lifted into a test suite or conformance corpus under Apache-2.0
without carrying an attribution notice; see [Licensing](#license).

---

## Contributing

Read [`CONTRIBUTING.md`](CONTRIBUTING.md). The short version:

- **Decide first, record second** — contentious design forks are argued to convergence
  before any text changes.
- **One decision = one commit** — a normative change updates every affected source of
  truth in the same commit. That cross-cutting pass is what holds coherence together.
- **Minimalism** (`PRIN-1`) — does this give something beyond what existing mechanisms
  express, or merely save a name? If the latter, it is rejected.
- **No implementation code, build, or tests** in this repository.

Participation is governed by the [Code of Conduct](CODE_OF_CONDUCT.md).

---

## License

Dual, because the repository holds two kinds of material at once:

- **The specification text** — all prose, tables, grammar productions, and normative
  rules — is **[CC BY 4.0](LICENSE-CC-BY-4.0)**. Quote it, translate it, build on it,
  commercially or not, with attribution.
- **Code fragments** — Alatyr examples, manifests, assembly and WAT listings — are
  additionally **[Apache-2.0](LICENSE-APACHE-2.0)**, so an implementation can lift them
  into its own sources with no attribution notice riding along, and with an explicit
  patent grant.

Full terms, including how the two interact and what is *not* licensed (the project's
names): [`LICENSE`](LICENSE).

---

## The name

**Alatyr** (Cyrillic *Алатырь*) is the cornerstone of Slavic myth — the immovable stone
at the centre of the world, from which everything else is measured. It is a fitting name
for a language whose premise is that the machine underneath is never abstracted away. The
source extension is `.al`.
