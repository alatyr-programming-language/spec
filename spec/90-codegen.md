# Alatyr Language Specification

## Chapter — Codegen and Lowering

> **Status — draft (under review).** This chapter is a draft and is not yet
> accepted; the specification as a whole is under development. See the status note
> at the top of the Overview chapter.

This chapter defines the **lowering pipeline** (source → backend-specific
assembly-correspondence → backend emission; for a register ISA, GAS → `as` → `ld`), the
**lowering contract** (I1), the **determinism and conformance** model, the
**cross-cutting lowering** owned here, the **GAS emission** model for register-ISA
backends, and **containers / toolchain** handoff. It is the formalization of **I1**: a
programmer predicts the emitted backend form from the source and the documented rules,
without running the compiler.

Per-construct lowering rules live in each feature chapter's *Lowering* section; this
chapter owns the **pipeline**, the **determinism/cost contract**, the **emission
model**, and the **cross-cutting** lowering (ABI frames, aggregate copies, guard
insertion, placement). It builds on the Assembly-Correspondence chapter (the lowering
target), the Functions chapter (ABI), the Type-System chapter (layout), the Comptime
chapter (erasure, monomorphization), and the Tooling chapter (toolchain handoff, the
manifest).

This chapter introduces **no surface syntax** (it lowers existing constructs), so it
has no syntax-fragment section. Requirement keywords follow RFC 2119 (Overview §6).

---

### 1. The pipeline

#### 1.1 Stages

Lowering is **staged**, in this order:

1. **Configuration** — the manifest fixes the **target** and thus the **machine model**
   (Overview §3; Tooling chapter);
2. **parse** — to the AST;
3. **semantic analysis** — name resolution, typing, the static checks (mutability,
   linearity, definite assignment, exhaustiveness, limits), comptime evaluation,
   **monomorphization** of generics, and **erasure**. These are **interleaved**, not
   sequential: a comptime expression is type-checked **before** it is evaluated, and
   name/type/constraint checking applies only to **selected** branches, **satisfied**
   `when` declarations, and each **monomorphized** instantiation (the two-phase rule,
   Comptime §9). An ill-formed program is rejected here;
4. **lowering** — structured constructs → the backend-specific
   **assembly-correspondence** surface, applying guard insertion, placement, and ABI
   framing (§4);
5. **backend emission** — the assembly-correspondence form → the backend's lowest form
   (for a register ISA, **GAS text**, §5);
6. **toolchain handoff** — for a register ISA, shell out to **`as`** then **`ld`** (§6).

Each of stages 3–5 is a **pure** function of its input (determinism, §3).

**Optimization** is a **semantically-preserving** pass on the internal IR (§1.2), interleaved with
lowering: it is **pure** and changes only **cost**, never observable behavior (no UB to exploit, I11;
CG-5). It is bounded by the **`no_opt`** limit (un-optimized output) and the byte-exact `no_abstractions`
floor (Tooling §3); a **guaranteed minimum** (monomorphization, trivial-wrapper inlining, removal of
provably-unnecessary guards) backs zero-cost (I2).

#### 1.2 No language-level IR, no semantic bypass

The **language-level** lowering target **is** the assembly-correspondence
surface (Assembly chapter): there is **no** second, language-defined intermediate
*language*, and a high construct's **observable** lowering goes **through** the
assembly-correspondence constructs (instructions, labels, data) — it never
**semantically bypasses** them to define behavior outside that model (one language;
Overview §3). This constrains the *language model and the observable result*, **not**
the implementation: a conforming compiler **MAY** use an internal IR (SSA, etc.) as a
private implementation detail, provided its observable lowering matches the rules
(§2.3). The compiler does **not** itself encode machine code or link — that is `as`
and `ld` (decisions CG-1).

The **backend is pluggable** (CG-4): the assembly-correspondence and the emitted form are the
backend's — GAS (`as`/`ld`) for a register ISA, WAT (`wasm`) for WASM (additive, FND-6) — while the internal
IR and the **portable-core** lowering (types, functions/ABI, structured control flow, data, comptime) are
shared. The register-ISA pipeline (§1.1) is the v1 instance; raw constructs (registers, raw transfer, `asm`)
are a **target-capability**, absent on a structured/VM backend.

---

### 2. The lowering contract (I1)

#### 2.1 Documented rules

Every construct lowers by a **documented rule**. The rule lets a programmer **predict
the emitted assembler** from the source alone, *without running the compiler*. The
prediction protects **cost and behavior**: no hidden loads, spills, copies,
allocations, or control flow beyond the documented rule (I1).

#### 2.2 Where the rules live

Each feature chapter's **Lowering** section is the normative rule for that chapter's
constructs (e.g. Control Flow §10, Memory §4.6). **This chapter** is normative for the
**pipeline** (§1), the **cross-cutting** lowering (§4), and the **emission** (§5). The
per-construct rules and the per-arch tables are **v1 content** (fully specified for
v1; "additive", FND-6, governs only future growth — Overview §6).

#### 2.3 Status of the rules

The rules are **normative and deterministic**. A reference lowering MAY be provided;
the intent is that the rules are precise enough to mechanize ("spec ≡ code"), but a
conforming implementation need only **agree with the rules**, not share an
implementation.

---

### 3. Determinism and conformance

#### 3.1 Intra-implementation — reproducible

Codegen is **pure**: one `(source, target, manifest, profile)` maps to **one** emitted text
(GAS for a register ISA; the backend's lowest form generally, e.g. WAT — CG-4)
— no hidden state, no wall-clock, no environment. (The selected **profile** is a build
input, Tooling §1 — it tunes the non-semantic optimization level, CG-5, so it
co-determines the emitted text.) The **final bit-identical artifact** additionally depends
on the **lockfile** and **toolchain** — the full `(source, manifest, target, profile,
lockfile, toolchain)` build contract (Tooling §6.2); codegen purity is the
compilation-stage half of that contract, consistent with the comptime determinism of
Comptime §2.

#### 3.2 Inter-implementation — rule-and-cost consistency, not byte-identity

Two **conforming** implementations need **not** emit byte-identical output (GAS, or the backend's
form such as WAT; CG-4). Each MUST agree on
**observable behavior** and respect the **I1 cost ceiling** (no hidden loads/spills/copies/allocations/
control beyond the documented rule). What may legitimately differ is the **unobservable** placement
detail (§3.3) **and the degree of semantically-preserving optimization** (CG-5) — optimization may only
**lower** observable cost, so it stays within the ceiling. What may **never** differ: the observable
behavior, the **guaranteed minimum** optimization (I2/CG-5), and — under `no_abstractions` — the exact
instructions.

#### 3.3 Placement (S2) is implementation-documented — outside `no_abstractions`

**Outside `no_abstractions`**, the **register/stack placement strategy** (S2; Memory
§2.5) is a **deterministic strategy each implementation documents** — it is **not**
fixed by this specification. The specification fixes the **cost contract**: spills are
emitted by the documented rule, so their cost is predictable (I1); and the **exact
register number is unobservable** — it changes neither behavior nor cost — and is
therefore **not** subject to hand tracing. So two implementations may assign different
registers yet both conform.

#### 3.4 Under `no_abstractions` — byte-exact

Under the `no_abstractions` limit there is no placement freedom: placement is **fully
determined by the source plus the fixed, spec-defined rule of Memory §2.5** (explicit
storage specifiers, else scratch registers in the ABI-appendix `caller_saved` order by
source order, spilling by the documented rule) — **not** the impl-documented S2. So
**what is written is what is emitted**, including registers (I1), and conforming
implementations agree on the **exact instructions, registers included**, for a
`no_abstractions` unit.

#### 3.5 The guaranteed minimum (precise)

Zero-cost (I2) rests on a **guaranteed minimum** of semantically-preserving
optimization that **every** conforming implementation MUST perform — so a zero-cost
abstraction reduces identically everywhere. Anything **beyond** the minimum is latitude
that may only **lower** cost (§3.2). The minimum is **exactly**:

1. **Monomorphization** — every generic is fully monomorphized per comptime-argument
   set (Comptime §3.4); no residual generic dispatch remains.
2. **Trivial-wrapper inlining** — a **trivial wrapper** is a function whose body is a
   **single call or expression that forwards its parameters**, with **no additional
   bindings and no control flow**; every call to one is inlined to the forwarded
   operation.
3. **`@inline` substitution** — a function marked `@inline` (Declarations §2.3; CG-10) is
   **substituted at every direct call site**, with each argument bound to a fresh local
   as needed (so a **compound operand** — `j*j` in `lt(n, j*j)` — becomes an ordinary
   place); **no `call`** is emitted, and the function emits **no standalone symbol**
   unless its address is taken or it is `@export`ed (then an out-of-line copy is emitted
   too). The native operators (Type System §3.1 / TYP-2) are `@inline`, so an operator
   inlines to its instruction(s) **regardless of operand shape** — the marker is the
   contract, not an inferred body shape. (How the substituted body is then *placed* —
   register vs spill — is latitude, §3.3; a deeply-nested inlined expression may spill,
   but never reverts to a `call`.) `@inline` on a recursive function is ill-formed
   (Semantic diagnostic).
4. **Dead-guard elimination** — a checked-guard (Assembly §3; overflow CG-8) is removed
   when its condition is **provably satisfied** by **constant propagation** (a
   comptime-constant operand makes it constant-true) or by a fact established by a
   **dominating check in the same straight-line region**. No deeper analysis (general
   value-range, interprocedural, alias) is **required** by the minimum.

Additionally, an abstraction the specification **documents as zero-cost** MUST reduce
to its **hand-written lower-level equivalent** (emit nothing beyond it). An
implementation **MAY** optimize further (any semantically-preserving transform, §3.2)
but **MUST NOT** do **less** than this minimum. Only the minimum is part of
**conformance**; everything beyond is unobservable latitude (cost only) and so need not
agree between implementations.

---

### 4. Cross-cutting lowering (owned here)

The following lowerings are not owned by a single feature chapter; their **semantics**
are referenced, their **emission** is normative here:

- **ABI framing** — a function's prologue/epilogue and argument/result placement per
  its calling convention (Functions §4, §6); a `@abi(naked)` function emits **no**
  frame (the body is exactly what is emitted).
- **Aggregate copy** — a by-value aggregate copy lowers to a **documented byte copy**
  by the layout rule (Type System §6; Memory §5.9); its cost is visible (I1). A raw
  union copy is exactly `U.size()` bytes, including padding, and emits no active-member
  metadata (TYP-11). A raw-union member constructor or whole-member assignment lowers,
  after left-to-right argument/RHS evaluation, to at most one `U.size()`-byte clear plus
  one `Ti.size()`-byte copy per declared component at its tuple offset. Tuple-introduced
  and union-tail padding remain zero; no tag store or active-member check is emitted.
  For a checked `@require` constructor gate (Types §8.1), the result representation is
  preserved and the predicate receives one distinct logical by-value copy: small aggregates copy
  into their ABI-classified argument slots; large aggregates produce one full caller-owned
  byte-copy and pass its address (Functions §4; ABI appendix §6).
- **Guard insertion** — checked-default runtime guards + inline traps for checkable
  operations (Assembly §3; integer overflow CG-8), at the documented points; **omitted**
  for an explicitly-`unchecked` operation (CG-6). A false `@require` predicate branches
  directly to the per-architecture checked-failure trap (per-arch appendix §2), never to
  `assert`/`panic`; `unchecked R(v)` omits the predicate copy, call, branch, and trap.
- **Placement** — register/stack assignment per S2 (§3.3).
- **`defer`** — cleanup actions emitted, **LIFO**, on every normal exit of their
  scope (Memory §5.8); not on abort.
- **Structured control flow** — `if`/`match`/loops/`break`/`continue`/`return`/`?`
  lower to labels + branches per Control Flow §10.

No cross-cutting lowering introduces cost beyond its documented rule (I1/I3).

---

### 5. GAS emission

#### 5.1 The construct → directive mapping (normative)

Emission maps constructs to GAS by a **normative** mapping, so conforming
implementations agree on the **shape** of the GAS:

- **data** → `.byte` / `.quad` / … per the value's layout (Type System §6), with
  **target endianness** (`@endian(big|little)`, Type System §8);
- **sections** → `.section` per the derived section (Memory §2.3; `@section`
  override);
- **exported symbols** → `.global` + the symbol label, named per the mangling rule
  (Modules §6); a **present but non-exported** declaration (Modules §6.4 — reachable from
  one of the artifact's roots, yet outside its exported surface) emits its code and data
  under a local label and **no** `.global`; a declaration that is neither present nor
  exported is not emitted at all;
- **instructions** → their mnemonic, with operands **reordered** to the assembler's
  convention (AT&T source→destination; Assembly §2.2) while preserving *which*
  instruction;
- **`asm(…)`** raw escape → the template with positional `{i}` operands substituted
  (Assembly §4), passed to `as` verbatim.

The **per-target textual spelling** of an operand (register name, immediate format,
the `at(…)` addressing form, the reloc spelling) is **per-arch data** (Assembly
§5–§8; the per-arch appendices).

#### 5.2 A centralized emitter is guidance, not a rule

Routing all directives through a single emitter (rather than scattering GAS literals)
is **recommended implementation guidance**, **not** a normative requirement; what is
normative is the mapping of §5.1.

---

### 6. Containers and toolchain handoff

- The **container** (ELF / PE / Mach-O / COM) is chosen by the **target triple**
  (Tooling chapter). Codegen emits **GAS text + sections + symbols + entry + a link
  plan**; the **binary encoding and linking** are `as` + `ld` (CG-1) — codegen does not
  encode the container itself.
- **Dynamic imports** go through the container's mechanism (a PE import table, an ELF
  PLT/GOT); the asm-level **relocation operands** (Assembly §8.2) drive this.
- **Manifest-provided** linker configuration (libraries, flags, linker scripts;
  Modules §7.5) is the **Tooling** chapter's; codegen consumes it as the **link plan**
  it hands to `ld`.
- **The entry symbol is part of that plan and is always stated** — on a **register-ISA
  backend**, which is where a link step exists. On a backend with no linker (a
  structured/VM backend such as WASM; CG-4/CG-14) the entry is whatever that backend's own
  rules make it, and `startup` (start-up objects and libc, Tooling §2.2) does not apply
  there; both are fixed with that backend (additive, FND-6). For a register ISA:
  `Target.entry` is an
  Alatyr **path** (Tooling §2.2; TOOL-12); codegen resolves it to the symbol that
  declaration emits (Modules §6.1, or its exact `@export("…")`, Modules §6.3) and passes that
  symbol to the linker **explicitly** — the linker's own default entry symbol is never
  relied on. A path that names **no** declaration is a **Codegen**-stage diagnostic
  (Tooling §5) raised **before** the linker is invoked, so the failure is the
  toolchain's and deterministic rather than the linker's. Under `startup = libc` the
  start-up objects own the process entry and no `-e` is emitted; under `startup = raw`
  no start-up object is linked at all.

---

### 7. Conformance (normative summary)

A conforming implementation MUST:

1. lower in the staged order of §1.1 — configuration, parse, **interleaved** semantic
   analysis (checks + comptime + monomorphization + erasure, two-phase per Comptime
   §9), lowering, and **backend emission** (GAS → `as` → `ld` for a register ISA;
   the backend's own lowest form generally, CG-4) — with the post-parse phases pure,
   targeting the **assembly-correspondence surface** as the **language-level** lowering
   target with **no language-level IR and no semantic bypass** (an internal IR is an allowed
   implementation detail, §1.2), and never itself encoding machine code or linking
   (§1; CG-1/FND-9);
2. lower each construct by its **documented rule** so the emitted assembler — its cost
   and behavior — is predictable from source + rule without running the compiler, with
   per-construct rules in their feature chapters and the cross-cutting / emission rules
   here (§2; I1);
3. be **deterministic** — one `(source, target, manifest, profile)` → one emitted text (GAS
   for a register ISA; the backend's form generally, CG-4); the final bit-identical artifact
   additionally folds in the lockfile and toolchain (Tooling §6.2), reproducible
   — and **conform by rule-and-cost consistency**, not byte-identical output: only the
   unobservable register/stack placement (a documented, deterministic S2) may differ
   between implementations; behavior, cost contract, and (under `no_abstractions`) the
   exact instructions may not (§3; I1);
4. emit the **cross-cutting** lowerings — ABI frames (none for `@abi(naked)`),
   documented aggregate byte-copies, checked-default guards+traps (omitted under an
   `unchecked` grant), including the preserved-result + ordinary-ABI argument copy/call +
   direct target trap of a checked `@require` gate, S2 placement, LIFO `defer`, and
   structured-control branches — with no cost beyond the documented rule (§4; I1/I3);
5. for a **register-ISA backend** (the v1 instance; CG-4), emit GAS by the **normative
   construct→directive mapping** (data/sections/symbols/instructions/`asm`) with target
   endianness and AT&T operand order, taking the per-target operand spelling from the
   per-arch data; treat a centralized emitter as guidance, not a rule (§5);
6. for a **register-ISA backend** (CG-4), select the **container** by the target triple,
   emit GAS + sections + symbols + a link plan and hand assembly/linking to `as`/`ld`,
   drive dynamic imports through the container's mechanism via the reloc operands, and
   consume the manifest's linker configuration as the link plan (§6).
