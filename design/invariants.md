# Invariants and principles

Axioms the entire language is obligated to obey. Every design decision is checked against this list.
If a chapter or decision violates an invariant — we argue until resolution, rather than sneaking it through.

Status: `[firm]` — agreed; `[prelim.]` — working, may be refined (tied to an open question).

---

- **I1. Codegen transparency.** `[firm]`
  Every construct emits code by a documented rule. The programmer predicts the emitted
  assembly by reading the source and the rule, **without running the compiler**.
  Transparency protects two things:
  - **Behavior (semantic transparency) — always.** The observable behavior — results, traps,
    and the order of observable effects (`volatile`/`atomic`/IO) — is exactly what the source and
    the rules prescribe. **Optimization is semantically-preserving** (it cannot be otherwise: there
    is no UB to exploit, I11) and never changes observable behavior. A transform that *would* change
    something observable (e.g. float reassociation/contraction) is **opt-in and visible in the
    source**, never applied silently (CG-5/CG-9).
  - **Cost — a documented ceiling, monotone under optimization.** The documented rule is the
    **upper bound** on cost: no hidden loads/spills/copies/allocations/control flow beyond it. A
    semantically-preserving optimization (CG-5) may only **lower** observable cost, never raise it.
    Exact-cost / **backend-exact** prediction — what you wrote is what comes out (exact instructions incl.
    registers for an ISA, exact WASM ops for WASM; CG-4) — is guaranteed under `no_abstractions` (and the
    `no_opt` limit).
  *Placement (S2, MEM-7):* outside `no_abstractions`, placement is deterministic per the published
  rule, but the exact register *number* is an unobservable detail (it changes neither behavior nor
  the cost ceiling) and is therefore not subject to manual tracing.

- **I2. Zero-cost abstractions.** `[firm]`
  An abstraction compiles either to what the programmer would have written by hand at the lower level, or
  is enabled explicitly per-site. Zero-cost is upheld by a **guaranteed minimum of
  semantically-preserving optimization** (CG-5) — monomorphization, trivial-wrapper inlining, **`@inline`
  substitution** (CG-10), and
  **elimination of provably-unnecessary checked-guards** (the four precisely defined in Codegen §3.5) — so an
  abstraction that reduces to nothing emits nothing; optimization beyond that minimum is implementation latitude and can only lower cost
  (I1). The only inherently non-zero-cost constructs are the checks of the checked default
  (arithmetic/bounds/alignment/narrowing) that the compiler **cannot** prove unnecessary; each is also
  omitted inside an explicit `unchecked` **scope** covering the operation site (CG-6/CG-7).

- **I3. Nothing hidden.** `[firm]`
  Hidden control flow, hidden allocations, hidden copies, vtables, implicit dispatch are forbidden.
  Hidden = behavior the programmer **cannot recover from the closed package** — its source **plus its
  manifest plus the language conventions** (CG-11). Determinism is at the **package boundary**, not the
  literal call site: a fact omitted at a use site is **not** hidden when it is **determinable** from that
  closed space — e.g. eliding a **declared** parameter and filling it from a **lexically-visible** scope
  (the dependency is in the signature; the source is a named binding in scope). Local self-description —
  a fact visible *where* it acts — remains a priority, but one **traded** against ergonomics and real
  usage frequency *only* when the omitted fact is an already-declared, already-visible binding; it never
  licenses a fact the package cannot recover.

- **I4. Completeness of the lower level (per backend).** `[firm]`
  Everything expressible in the **target backend's lowest documented form** — GAS for a register ISA, WAT for
  WASM (CG-4) — is expressible in the language **for that target**. On a register-ISA backend the
  assembly-correspondence level is self-sufficient (kernel/firmware/boot are written entirely in it). Cross-target
  code uses the **portable core**; the raw layer (registers, raw transfer, `asm`) is a **target-capability**,
  present only where the backend supports it.

- **I5. One language + orthogonal limits.** `[firm]` (see FND-10)
  One language, a single grammar and a single AST. Raw assembly constructs, structured constructs, and
  comptime **coexist by default**. Transparency/capability guarantees — via **orthogonal
  opt-in limits** (`no_abstractions`/`no_alloc`/`freestanding`/…), freely combinable; a local
  contract of the translation unit (I9). There is no ladder of superset levels.

- **I6. The machine model as the foundation of types.** `[firm]`
  Any type ultimately decomposes into data blocks of the target's native bit widths. The machine
  model (bit widths; the storage model — registers, or an operand stack + locals on a structured/VM backend;
  endianness; feature set) is supplied by configuration (the manifest) as input for computing types (CG-4). The
  manifest and the type system are one system.

- **I7. Comptime erasure.** `[firm]` (see CT-1)
  Type-level and comptime computations are fully erased before runtime and do not leak into runtime
  cost. The comptime/runtime boundary is explicit and verifiable by the programmer.

- **I8. Explicitness of loss and change of meaning.** `[firm]`
  Operations that lose information (narrow) or change the meaning of a value (reinterpret, numeric, brand)
  require explicit indication. What is allowed implicitly is determined by the level's transparency contract.

- **I9. Locality of the translation unit's contract (limits).** `[firm]` (see FND-10)
  The **limits** of a translation unit (a file, FND-11) are its local contract. **Inside** the unit, raw asm,
  structured constructs, and comptime **freely mix** (FND-9) within its limits. **Between**
  units with different limits, interaction is via an **explicit interface** (signature/ABI/symbol), not
  via implicit mixing of contracts.

- **I10. Completeness of the expressive surface.** `[firm]` (see FND-6)
  The foundation, semantics, syntax, and base primitives of abstractions are obligated to be complete at once —
  the programmer cannot extend the language, they are bounded by its set. It is permissible to defer only what is
  *additive* (addable later without breaking existing programs). The test: "addable later without breaking?".

- **I11. No undefined behavior.** `[firm]`
  The language has no UB in the C sense (where the specification imposes no requirement, and the compiler
  *exploits* that to arbitrarily transform unrelated code). This is precisely what **licenses an optimizer**
  (CG-5): with no UB to exploit, optimization can only be **semantically-preserving** (I1) — it never
  miscompiles unrelated code, because there is no undefinedness to assume away.
  - **checked level:** there is no UB — an operation is either provably correct, or deterministically catches a bad
    input (trap).
  - **unchecked/assembly level:** dangerous operations have behavior **defined by the hardware/ISA**
    ("what the machine does"), not "what the compiler wants"; the compiler never breaks unrelated code by
    assuming that the dangerous operation did not happen. At the bottom — the ISA contract honestly passed through.
  *Corollary:* the subtopic "lifetime / dangling" — is obligatory to resolve (a dangling pointer is the main
  remaining source of UB).
