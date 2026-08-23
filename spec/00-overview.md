# Alatyr Language Specification

## Chapter 0 — Overview

> **Status — complete first draft, under review.** Every chapter and appendix is now
> drafted: this Overview and the Memory-model, Type-system, Comptime-and-Generics,
> Declarations-and-Bindings, Functions-Parameters-and-ABI, Control-Flow,
> Modules-Visibility-and-Packages, Assembly-Correspondence, Codegen-and-Lowering,
> Standard-Library, Concurrency-Overflow-and-Floating-Point, Tooling, and Grammar
> chapters, plus the **Manifest-syntax**, **ABI-per-target**, **prelude/stdlib**, and
> **per-arch** appendices (the per-arch appendix covers all six v1 architectures and the
> closed supported-target set). The whole document remains a **draft under review** —
> not yet accepted — and the conformance requirements in §6 describe the *completed*
> specification's intent.
>
> **Version — `1.0.0`.** The specification is living and versioned. The MAJOR digit is
> the version of the *language* this document specifies (Alatyr v1); MINOR marks a
> normative revision of the text, PATCH an editorial one. The number is not a maturity
> claim — whether the document is accepted is what the Status above states, and it is not
> yet. See `CHANGELOG.md`.

This is the **Overview** of the Alatyr language specification. The chapter is
partly *informative* (it frames the language, its niche, and its non-goals) and
partly *normative*: the **design invariants** (§4) and the **conformance**
requirements (§6) are binding on the whole language and on every other chapter.

The remaining chapters define Alatyr's lexis, syntax, and semantics normatively.
Where any chapter conflicts with an invariant in §4, the invariant governs and
the conflicting text is a defect.

---

### 1. Thesis

**Alatyr is a systems programming language whose ceiling of expressiveness equals
that of the assembler** — everything expressible in the target backend's lowest form
(GNU assembler for a register ISA, WAT for WASM; CG-4) is expressible in Alatyr — **while providing powerful
abstractions that are transparent in code generation, zero-cost, and governed by
explicit programmer direction or language conventions.** The programmer always
knows what the machine does.

Code is generated as the **backend's lowest form** — GAS text (→ `as` + `ld`) for a register ISA,
or WAT (→ `wasm`) for WASM (CG-4). The backend is not LLVM; codegen is the compiler's own predictable
emitter for each target. Optimization
is **semantically-preserving** — it cannot change observable behavior (there is no UB to exploit,
I11), only cost — and behavior-changing transforms (e.g. float fast-math) are opt-in (§4, I1; CG-5).
Predictability of behavior and cost takes priority over peak automatic optimization. The language is
statically typed and ahead-of-time compiled, with no managed runtime.

Influences: Zig, Rust, Ada, D, C.

### 2. Audience and niche

**Primary niche — low-level / systems software:** operating-system kernels,
firmware, bootloaders, bare-metal and embedded targets, device drivers,
hypervisors, memory allocators, codecs, cryptography, DSP and vector kernels, and
performance hot paths — anywhere a programmer would otherwise drop to C or hand
assembly but wants stronger abstractions without losing control.

**Secondary niche — application code** that values predictability and the absence
of hidden cost.

**Audience:** programmers who want assembly-level control together with modern
ergonomics, and who always want to know the cost of what they write.

### 3. Compilation and execution model

**One language.** Alatyr has a single lexer, parser, and abstract syntax tree.
There is **no ladder of "levels" or "tiers."** Raw assembly constructs
(instructions, labels, jumps), structured constructs (functions, types,
structured control flow), and compile-time (`comptime`) metaprogramming all
belong to the one language and **may be freely mixed within a translation unit.**

**Limits.** Guarantees are opted into, per translation unit, through orthogonal
**limits** — restrictions on what a unit may do. Examples: `no_abstractions`
(assembly-correspondence only: every emitted runtime instruction is one the
programmer wrote, 1-to-1 with GAS), `no_alloc` (no dynamic allocation),
`freestanding` (no operating-system dependence). Limits are declared in the
package manifest (a package-wide ceiling) and/or per file (a stricter contract,
never looser than the ceiling), and they combine freely. There is no separate
"tier" dimension; an assembly-only contract is simply the `no_abstractions`
limit.

**Code generation.** The compiler lowers each construct to the **backend's lowest form** by documented
rules (the **lowering rules** are normative content of this specification) — GAS text (then `as` + `ld`)
for a register ISA, or WAT (then `wasm`) for WASM (CG-4). A construct never bypasses lower constructs to
emit machine code directly.

**Machine model.** A package's manifest is **configuration**: it declares the default (and
any allowed) target, and together with the build invocation (which MAY select a target
via `--target`) this fixes the **machine model** — native integer and floating-point
widths, the storage model (registers, or an operand stack + locals on a structured/VM backend),
endianness, and available ISA/backend features (CG-4). The rest of the language —
notably the type system — is defined *over* this machine model. The set of
supported architectures, containers, and their machine models is enumerated in
the assembly-correspondence and codegen chapters.

### 4. Design invariants (normative)

The whole language obeys the following invariants. They are **normative**: a
construct, rule, or implementation behavior that violates an invariant is
ill-formed or non-conforming by definition.

- **I1 — Transparent code generation.** Every construct emits code by a
  documented rule. A programmer predicts the emitted assembler from the source
  and the rule, *without running the compiler*. "Predict" protects **cost and
  behavior**: no hidden loads, spills, copies, allocations, or control flow beyond
  the documented rule. Under the `no_abstractions` limit, byte-exact prediction is
  additionally guaranteed (what is written is what is emitted, including
  registers); otherwise placement is determined by a published deterministic
  strategy, and the exact register number — being unobservable — is a deterministic
  detail not subject to hand tracing.

- **I2 — Zero-cost abstractions.** An abstraction compiles either to what the
  programmer would write by hand at the lower layer, or is opted into per site.
  Zero-cost is upheld by a **guaranteed minimum of semantically-preserving
  optimization** (CG-5) — monomorphization, trivial-wrapper inlining, `@inline`
  substitution (CG-10), and
  **elimination of provably-unnecessary checked-guards** (Codegen §3.5) — so an
  abstraction that reduces to nothing emits nothing. The only inherently
  non-zero-cost constructs are the checks of the checked default (arithmetic,
  bounds, alignment, and narrowing checks) that the compiler **cannot** prove
  unnecessary; each is also omitted inside an explicit `unchecked` scope covering
  its site (the verification axis is `checked`/`unchecked`, not `safe`/`unsafe`;
  CG-6/CG-7).

- **I3 — Nothing hidden.** Hidden control flow, hidden allocations, hidden copies,
  vtables, and implicit dispatch are forbidden. *Hidden* means behavior the
  programmer **cannot recover from the closed package** — its source **plus its
  manifest plus the language conventions** (§4.1 / CG-11). Determinism is at the
  **package boundary**, not the literal call site: a fact omitted at a use site is
  not hidden when it is **determinable** from that closed space (e.g. a declared
  parameter elided and filled from a lexically-visible scope).

- **I4 — Lower-layer completeness (per backend).** Everything expressible in the target
  backend's lowest documented form — GAS for a register ISA, WAT for WASM (CG-4) — is
  expressible in Alatyr for that target. On a register-ISA backend the
  assembly-correspondence level is self-sufficient (kernels, firmware, and boot code can
  be written entirely in it); cross-target code uses the portable core, and the raw layer
  (registers, raw transfer, `asm`) is a target-capability present only where the backend
  supports it.

- **I5 — One language, orthogonal limits.** A single grammar and AST. Raw
  assembly, structured constructs, and `comptime` coexist by default; guarantees
  come from orthogonal opt-in limits, combined freely. There is no superset ladder
  of levels.

- **I6 — Machine model as the foundation of types.** Every type ultimately
  decomposes into data blocks of the target's native widths. The machine model
  (widths, registers, endianness, features) is supplied by configuration from the
  **selected target** (the manifest's default or the `--target` choice) as input to
  type computation. The manifest and the type system are one system.

- **I7 — Compile-time erasure.** Type-level and `comptime` computation is fully
  erased before run time and never leaks into run-time cost. The comptime/runtime
  boundary is explicit and checkable by the programmer.

- **I8 — Explicitness of loss and reinterpretation.** Operations that lose
  information (narrowing) or change the meaning of a value (bit reinterpretation,
  numeric-domain change, branding) require explicit notation. What is permitted
  implicitly is determined by the unit's transparency limit.

- **I9 — Locality of a unit's contract.** A translation unit's limits are its
  local contract. Within a unit, raw assembly, structured constructs, and
  `comptime` mix freely within those limits. Between units with different limits,
  interaction is through an explicit interface (signature, ABI, symbol), not
  through implicit blending of contracts.

- **I10 — Completeness of the expressive surface.** Foundations, semantics,
  syntax, and the base abstraction primitives are complete from the start — a
  programmer cannot extend the language and is bounded by its set. Only *additive*
  material (new instructions, library functions, target features) may be deferred.
  Test: "can it be added later without breaking existing programs?"

- **I11 — No undefined behavior.** There is no undefined behavior in the C sense
  (where the specification imposes no requirement and the compiler *exploits* that
  to transform unrelated code). This is precisely what *licenses* an optimizer
  (CG-5): with no UB to exploit, every optimization is **semantically-preserving** —
  it changes only cost, never observable behavior, and never miscompiles unrelated
  code, because there is no undefinedness to assume away.
  - *Checked operations:* no UB — an operation is either provably correct or
    deterministically traps on a bad input.
  - *Unchecked / assembly operations:* dangerous operations have **hardware-defined**
    behavior ("what the machine does"), never "what the compiler wants"; the
    compiler never miscompiles unrelated code by assuming a dangerous operation did
    not occur. At the bottom lies the faithfully exposed contract of the target ISA.

#### 4.1 The closed package, and layered conventions (normative; CG-11)

These invariants rest on one operational reading of *no magic*: the **package**, its
**manifest**, and the **language conventions** form a **closed logical space that fully
and unambiguously determines the generated machine code**. Anything an implementation
emits is recoverable from that space; nothing outside it (no ambient global, no
build-host state, no compiler whim) may change the result. This is what I3 means by
*hidden* and what I1 means by *predict without running the compiler* — both are judged
**at the package boundary**.

Two consequences shape the surface syntax:

- **Local self-description is a priority, not an absolute.** A fact is best shown *where*
  it acts; but where that fact is an **already-declared, already-visible binding**, the
  language MAY let a use site **omit** it — the package still determines it. The
  trade-off is decided by **ergonomics and real usage frequency**, never by hiding a
  fact the package cannot recover.
- **Conventions are layered, and never un-overridable.** A codegen convention resolves in
  up to three layers, each deterministic in the package: **(0)** a package convention
  (the manifest), **(1)** a lexical scope, **(2)** the explicit site. `unchecked { … }`
  is a layer-1 convention (CG-6/CG-7); the **ambient allocator** (`alloc::with`, MEM-5) is
  another. The call site (layer 2) is always available, so a convention never blocks
  dropping to the raw assembly ceiling (I4).

### 5. Non-goals (informative)

Alatyr deliberately does **not**:

1. use LLVM or an opaque optimizing backend, or apply behavior-changing optimization silently —
   optimization is semantically-preserving (CG-5), predictability outranks peak optimization, and
   behavior-changing transforms (e.g. fast-math) are opt-in and visible;
2. provide a managed runtime, garbage collector, implicit reference counting, or
   run-time type information;
3. have hidden control flow or implicit destructors (RAII) — cleanup is explicit
   (`defer`);
4. exploit, or even admit, undefined behavior (see I11);
5. trade transparency for convenience — no ergonomic "magic" that hides cost;
6. abstract the machine away — the machine model is always visible; cross-platform support
   is via explicit targets (a backend's machine model is visible, including a VM target such as
   WASM — CG-4), never a hidden portable virtual machine of Alatyr's own;
7. adopt a Rust-style borrow checker or a mutable-XOR-shared discipline — the
   memory model is simpler (scoped references, regions, handles);
8. pursue type-system theory for its own sake — compile-time-functions-as-types are
   pragmatic and bounded (no dependent-type proofs, no higher-kinded types for their
   own sake);
9. break compatibility within a frozen version — stability plus additive growth;
10. be dynamically typed or interpreted — it is statically typed and ahead-of-time
    compiled to the backend's lowest form (GAS → a native binary for a register ISA;
    WAT for WASM; CG-4).

### 6. Conformance (normative)

The **completed** specification is designed to be **normative and
implementation-complete**: it is written so that two independent implementations,
given the same source and the same target, produce compatible results according
to the rules herein, without guessing. (This goal governs how every chapter is
written; the present document is a draft — see the status note at the top of this
chapter.)

- Requirement keywords **MUST**, **MUST NOT**, **SHOULD**, **MAY** are used per
  RFC 2119.
- A **conforming implementation** accepts exactly the programs this specification
  declares well-formed, rejects ill-formed programs with a diagnostic, and emits,
  for a well-formed program and a selected target, code consistent with the
  normative lowering rules.
- The **v1 content** — instruction tables per architecture, ABIs per target, and
  the prelude / standard-library definitions — is enumerated normatively in the
  respective chapters and appendices. "Additive growth" means a future version MAY
  add such content without invalidating a conforming v1 program; it does **not**
  mean v1 leaves the content unspecified. v1 is fully specified.
- Where behavior is **implementation-defined**, it is explicitly marked as such.
  There is **no undefined behavior** (I11).

### 7. Structure of this specification

Each chapter is normative for its area. Chapters are loosely coupled and reference
one another by anchor.

- **0. Overview** (this chapter) — thesis, niche, model, invariants, non-goals,
  conformance.
- **Memory model** — values, places, bindings, addresses; storage classes;
  mutability; pointers; lifetime mechanisms; closures; copy/move and linearity.
- **Type system** — the kernel (`bitsN`) and numeric interpretations; the
  type-construction machinery; aggregates and layout; layout control; value
  construction.
- **Comptime and generics** — the comptime/runtime boundary; generics; constraints
  as predicates; comptime execution; type introspection.
- **Declarations and bindings** — the unified `name : type = value` form; scope,
  shadowing, and order.
- **Functions, parameters, and ABI** — parameter directions and result projection;
  calling-convention ABIs; passing modes; defaults, named arguments, variadics.
- **Control flow** — raw labels and jumps; structured `if`/`match`/loops/`break`/
  `continue`/`return`; exhaustiveness and safety.
- **Modules, visibility, and packages** — the module tree; `pub`; import and
  re-export by declaration; dependencies; symbol mirroring and mangling.
- **Assembly correspondence** — the instruction model; operands and registers;
  data definitions; the architecture matrix.
- **Codegen and lowering** — the lowering pipeline; lowering rules as the I1
  contract; determinism; GAS emission; containers.
- **Standard library and built-ins** — the prelude tiers; intrinsics; `panic` /
  `exit` / `assert`; allocators.
- **Concurrency, overflow, and floating point** — atomics, fences, `volatile`;
  integer-overflow semantics; floating-point determinism.
- **Tooling** — the manifest schema (configuration); the CLI; diagnostics; cross
  toolchains and reproducible builds.
- **Grammar** — the complete EBNF and lexis.

### 8. Notation and terminology (informative)

- **Grammar** is given in EBNF, defined in the Grammar chapter.
- **Requirement keywords** follow RFC 2119 (§6).
- **Terminology.** Key terms — *value*, *place*, *binding*, *address*, *limit*,
  *comptime value*, *lowering*, and others — are used with the precise meanings
  given where they are introduced. Source identifiers are **case-sensitive** and
  **underscore-significant** (compared as written); the exact rules are in the Lexis
  chapter.

---

*Rationale and the decision history behind this specification are kept separately
in the project's design journal and are not part of the normative text.*
