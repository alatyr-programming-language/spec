# Principles & priorities

The cross-cutting **priorities** that govern many decisions — *why* the specific choices in
the theme files keep coming out the way they do. Above these sit the **supreme norms**, the
invariants `I1`–`I11` (`../invariants.md`); below them, the per-theme decisions. A principle
is not a separate rule an implementation conforms to (that is an invariant or a spec clause)
— it is the priority that *resolves* design forks. Each carries a **`rests on:`** line (the
invariants and decision entries that embody it).

---

### PRIN-1. Minimalism — no redundant entity
*rests on: OP-1 → operators.md OP-1; the design method*

Do not introduce what existing mechanisms already express. The test: *does this give
something beyond what exists, or merely save names?* If the latter, reject it. This is why
operators are overloadable functions rather than keywords, why there is one reference plus
library handles (not a reference zoo), why `get` is an ordinary function, and why the index
read resolves to the existing `at` rather than a parallel `index` shim. Minimalism is a
priority, not an absolute — it yields when a distinct entity buys a real, otherwise-absent
capability (the `scoped` result qualifier, `@owning`).

### PRIN-2. Completeness now; only additive growth deferred
*rests on: I10, FND-6 → foundations.md FND-6*

The v1 surface (foundations, semantics, syntax, the base abstraction primitives) is
**complete from the start** — a programmer is bounded by the language's set and cannot
extend it. "Additive" concerns only *future* growth (new instructions, library functions,
target features) that cannot break existing programs; it is never a license to leave v1
underspecified (the FND-3 bar). Test for deferral: *can it be added later without breaking
existing programs?*

### PRIN-3. No magic = package-closed determinism
*rests on: I1, I3, codegen.md CG-11, memory.md MEM-5; spec Overview §4.1*

"No magic" has an operational meaning: the **package + its manifest + the language
conventions** form a closed logical space that **fully and unambiguously determines** the
generated machine code. Nothing outside that space (no ambient global, no build-host state,
no compiler whim) may change the result. Determinism is judged at the **package boundary**,
not the literal call site — which is what lets a use site omit an already-declared,
already-visible fact (e.g. an elided allocator) without that being "hidden".

### PRIN-4. Transparency is a priority, traded against ergonomics
*rests on: I3, memory.md MEM-5; PRIN-3*

Showing a fact *where* it acts is preferred — but it is a priority, not an absolute. Where
the fact is an **already-declared, already-visible binding**, the language may let a use
site omit it, decided by **ergonomics and real usage frequency**. The trade never extends to
hiding a fact the package cannot recover (PRIN-3): local omission is licensed only by
package-boundary determinism, never against it.

### PRIN-5. Conventions are layered, never un-overridable
*rests on: codegen.md CG-11; precedent codegen.md CG-6/CG-7*

A codegen convention resolves in up to three layers, each deterministic in the package:
**(0)** a package convention (the manifest), **(1)** a lexical scope, **(2)** the explicit
site. `unchecked { … }` is a layer-1 convention; the ambient allocator (`alloc::with`) is
another. The explicit site (layer 2) always exists, so a convention never blocks dropping to
the raw assembly ceiling (I4) — the bottom of the ladder stays reachable.

### PRIN-6. Configuration is comptime over a fixed lever set
*rests on: comptime.md CT-10; I6/I7*

Configuration is not open-ended text or ad-hoc flags but **comptime effectors over a fixed
set of levers**, evaluated and erased before runtime. The manifest and the type system are
one system (I6): the machine model and the convention defaults are comptime inputs, so the
generated code is a pure function of them (PRIN-3).

### PRIN-7. One language, orthogonal limits — no tier ladder
*rests on: I5, foundations.md FND-10*

There is one grammar and one AST; raw assembly, structured constructs, and comptime coexist
by default. Guarantees come from **orthogonal opt-in limits** (`no_abstractions` / `no_alloc`
/ `freestanding` / …), combined freely as a translation unit's local contract — never from a
superset ladder of "levels".
