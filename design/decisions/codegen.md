# Decisions — Codegen, lowering, optimization, backends, conventions

**Why the language is the way it is** for code generation. Each entry records the
*rationale* and the *rejected alternatives*; the normative *what* lives in the spec
(linked per entry), the *how* in the compiler. IDs are theme-prefixed (`CG-N`) and
**append-only** — assigned in articulation order, never renumbered. The `provenance:` line
records what an entry refines and where it lands in `spec/`.

---

## The emitter and the lowering pipeline

### CG-1. Our own predictable emitter, not LLVM
*spec: Codegen §1–§5 (spec/90), Assembly (spec/80)*

**Why.** Codegen transparency (I1) requires the compiler to own its emitter: byte-exact
output under `no_abstractions`, full control of the construct→assembly mapping, and
independence from an external backend. So code is generated as **GAS assembler text**,
handed to `as` + `ld` — the compiler does not encode or link itself. The pipeline is
**staged** (structured surface → an asm-correspondence surface of `bitsN`/instruction
intrinsics → GAS text → `as` → `ld`); high constructs **never bypass** the lower ones,
which is what makes the single-frontend story (FND-9) hold all the way down. The **lowering
rules are the I1 contract**: each construct has a **documented, executable rule** so a
programmer predicts the output from source + rule without running the compiler, and
spec ≡ code never diverges. Codegen is **pure and deterministic** — one `(source,
target)` yields one GAS text, no hidden state, reproducible. The GAS emission is
specified **normatively as a construct→directive mapping** (`.quad`/`.byte`/`.section`/
`.global`/labels) so independent implementations converge on the same GAS form; a
*centralized* emitter is a non-normative implementation guide (the map is normative, not
the emitter's architecture).

**Rejected.** **LLVM** (or any opaque external optimizing backend) — it forfeits
byte-exact, predictable output and the per-construct rule contract; optimization must run
on the compiler's own internal IR before emission, never inside an opaque backend.

### CG-2. Instructions are per-arch intrinsics; data is ordinary values
*spec: Assembly (spec/80), per-arch appendix (spec/170)*

**Why.** The asm level reuses the general machinery rather than inventing a second
grammar. **Instructions are per-arch intrinsics** — a kind of builtin (OP-1/TYP-5) taking
the form of a function call (`movq(…)`), unified with the numeric intrinsics (a `u64 +`
lowers via `add` to `addq`), available directly at the asm level and 1:1 with a single
GAS instruction (I1 — there is nothing to check on a raw machine instruction).
**Operands** follow a fixed schema (register, immediate, memory, label, reloc/TLS, plus
per-arch SIMD), with which kinds an instruction accepts being part of its table and
validity checked at compile time. **Registers are arch-data identifiers, not keywords** —
ordinary case-sensitive names from the machine model, in scope for the chosen target (or
qualified `<arch>::name` outside an arch gate), so the instruction/register surface grows
as data, not as language. Memory and SIMD are written as **ordinary expressions**: the
addressing operand **`at(base=, index=, scale=, disp=)`** lives on the **arch surface** too
(in scope for the chosen target, `<arch>::at(…)` qualified outside) — co-located with the
registers it is built from, since a memory operand is part of the operand schema, not a
data-place op (it is **not** in the dissolved `mem::` cluster, MEM-7); UFCS SIMD decorators
like `v0.lanes(f32,4)` likewise — "a method on an operand" is just UFCS, not a new mechanism.
**Operand order is destination-first** (`movq(rax, 60)` ≡ `rax.movq(60)`), reordered to
AT&T on emission while preserving which instruction is chosen. **Data at the asm level
is ordinary values too**: raw bytes are `[u8; N]` values (inline literals or `embed`),
placed via ordinary data-bindings into sections derived from mut×init — no separate
`bytes(…)` type and no data-level escape is needed, since any bytes are expressible as
`[u8; N]`. The instruction *schema* is fixed in the spec; the instruction *list*,
addressing modes, and decorators are **per-arch data tables that grow additively** (FND-6).

**Rejected.** **Special operand brackets `[…]` and suffixes** (`.4s`, `{k1}{z}`) — concise
but a separate grammar; expressions + UFCS decorators express the same forms within the
one grammar. **A separate `bytes(…)` data primitive** — redundant with `[u8; N]`.
**A kind-dependent reading of the array `;N` form**, and **`T(v)` as the array-default
token** — the latter overloads construction; the `[T = v; N]` form keeps type-vs-value
disambiguation syntactic. **Placing the addressing operand in a cross-cutting `mem::`
module** (superseded, `mem::at`) — it is an instruction operand built from arch registers,
so it belongs on the arch surface, not among data-place ops (MEM-7).

### CG-3. The raw GAS escape — completeness when the tables are incomplete
*spec: Assembly (spec/80), per-arch appendix (spec/170)*

**Why.** The curated, typed instruction set (with operand-kind checks and checked guards)
can never enumerate every mnemonic an architecture offers, yet I4 demands the language
reach the target's lowest documented form. So beside the typed set sits a **raw GAS
escape** — emission of an arbitrary mnemonic/string, validated only by `as`, with none of
the compiler's checks or guards — guaranteeing completeness (I4) when the tables fall
short. Typed instructions are a curated checked set; the raw escape is the fallback
(generalizing `bytes(…)`/`snippet`). It is a **raw operation** and therefore lives behind
the `unchecked` mode (CG-7). The per-arch appendix closes the schema with a fixed format
— five tables per arch (registers, curated instructions, addressing modes, reloc/TLS
decorators, SIMD decorators) — plus a **closed set of supported targets** (canonical
`arch×os×env×container` triples; a target outside the set is a Config diagnostic), all
growing additively. A keyword-collision rule keeps instruction names ordinary
identifiers: where a bare mnemonic collides with a keyword (`and`/`or`/`not`), the
intrinsic takes a size suffix (`andq`) while the bare mnemonic is still emitted (1:1 at
the GAS level, not of the spellings).

---

## Backends, optimization, conformance

### CG-4. Pluggable backends — the lowering form is per-backend
*spec: Codegen §1 (spec/90), Assembly (spec/80)*

**Why.** The upper language is backend-neutral, so hard-wiring "codegen = GAS" into it
would needlessly block structured/VM targets. Codegen therefore targets a **backend**,
not GAS alone: the pipeline (configuration → semantics → internal IR →
backend-specific assembly-correspondence → emit) is **backend-parametric**. A
register-ISA backend emits GAS (`as`+`ld`) for the six v1 arches; a structured/VM backend
(e.g. WASM → WAT) emits its own form — and each is still the compiler's **own** predictable
emitter, not an opaque optimizing backend, so "not LLVM" (CG-1) stands. Constructs split
in two: a **portable core** (types→data blocks, functions/ABI, structured control flow,
data/sections, comptime/generics) lowers to **every** backend; **target-capability**
constructs (registers/`@reg`, raw transfer and code-point labels, `asm`, byte-exact
pinning) exist **only** on backends that support them — on a structured backend their use
is a compile error, gated like an unavailable instruction (`when target…`). This
generalizes the machine model (I6) to "register machine **or** stack/structured machine"
and reframes the guarantees relative to the chosen backend: **I4 becomes completeness vs
the target's lowest documented form** (GAS for an ISA, WAT for WASM) and **I1's byte-exact
becomes backend-exact** under `no_abstractions`. Only register-ISA backends + GAS are v1;
other backends are additive (FND-6), but the abstraction and the portable/target split are
fixed **now** so a new backend adds without reworking the language (I10) — a portable-core
program ports unchanged, a target-capability one is already target-gated.

**Rejected.** **Hard-wiring "codegen = GAS"** into the language (the earlier reading of
CG-1/I4) — it blocked structured/VM targets even though the upper language is
backend-neutral; the real guarantee is transparent, predictable lowering to the target's
own lowest form, of which GAS is one instance.

### CG-5. Semantically-preserving optimization (there is an optimizer)
*spec: Codegen §3 (spec/90)*

**Why.** The earlier "no optimizer at all" reading of CG-1 left zero-cost (I2) an unbacked
promise and made `volatile`/`atomic`/`fence` look redundant. The guarantee the language
actually needs is **predictability + no-UB**, not the absence of optimization — so the
compiler **MAY** optimize, on its own internal IR before emission (still its own GAS
emitter, not a C-style backend). Because there is **no UB to exploit** (I11), every
optimization is **semantically-preserving**: it never changes observable behavior
(results, traps, the order of observable `volatile`/`atomic`/IO effects), only cost.
**Observable-preserving** transforms (inlining, DCE, CSE, register allocation, peephole,
removal of a *provably*-unnecessary checked guard) are allowed by default and disablable;
**observable-changing** ones (float reassociation/contraction, fast-math) are opt-in and
source-visible only, never applied by default (CG-9). Ordinary (non-`atomic`/`volatile`)
memory is optimized under **single-thread sequential semantics** — reordered/eliminated
freely while preserving the thread's observable behavior; inter-thread ordering comes
**solely** from `atomic`/`volatile`/`fence` (never reordered); a data race on ordinary
memory is hardware-defined, never UB, and the compiler makes **no aliasing assumptions**.
**Cost is a ceiling** (I1): a documented lowering rule is the upper bound; optimization
may only lower observable cost. **Zero-cost (I2) is backed by a guaranteed minimum** —
monomorphization, trivial-wrapper inlining, and elimination of provably-unnecessary guards
(precisely defined in Codegen §3.5) — only the minimum is part of conformance; anything
beyond is implementation latitude. Knobs follow "one language + orthogonal limits":
`no_abstractions` is the byte-exact floor; an optional `no_opt` limit gives predictable
un-optimized output; optimization aggressiveness is a non-semantic build-profile axis.
**`assert` and the checked-guard family are never stripped** by a profile/level — a guard
is dropped **only** by *proving* it can never fire (correct optimization, not turning
checks off).

**Rejected.** **"No optimizer at all"** (the earlier reading of CG-1) — it left I2 unbacked
and the concurrency primitives apparently redundant; a semantically-preserving optimizer
delivers the predictability + no-UB the language actually needs.

---

## The verification mode (`checked`/`unchecked`)

### CG-6. The verification axis is `checked`/`unchecked` — a fixed, honest default
*spec: Codegen §3 (spec/90), Assembly (spec/80)*

**Why.** Chapters already wrote "in safe code" without the axis being firmly fixed, so
its default, nature, and relation to the limits are pinned. The axis is named
**`checked`/`unchecked`, not `safe`/`unsafe`**, out of honesty: it names the *mechanism*
(presence/absence of built-in verification) and promises nothing about safety — our
default is weaker than Rust's (races are hardware-defined, there is no borrow checker), so
"safe" would import an unearned connotation. **`checked` is a fixed language default, not
overridable by the manifest**: unlike the limits (a manifest ceiling), the checked basis
is fixed, and `unchecked` only *narrows* it, only *lexically visibly* — you cannot end up
in unchecked without the word `unchecked` present, so the absence of the word does not lie
(I3). Absence of `unchecked` promises "this code introduces no unchecked operation" (each
is provably correct or deterministically traps, I11) — **not transitively**: a call may
encapsulate `unchecked` interior, but because of I11 a botched interior yields an
incorrect-but-defined value/trap, never UB-contamination of the caller. **`unchecked` is
a bare keyword, not a `@`-attribute**: it changes the verification *contract/role* of a
region (a fixed set), the structural sibling of `comptime{}`, and `@` decorates entities
rather than wrapping expressions/blocks. **`no_unchecked` is a limit** (manifest/file)
banning the `unchecked` keyword anywhere in the **unit's own source** — transitive over
the unit's nested scopes but **not** over the cross-unit call graph (a `no_unchecked` unit
may still call separately-audited trusted units that use `unchecked` internally) — an
audit-grade "this unit's source is raw-escape-free". The common shape across mode axes:
**the default is unnamed · the marker marks the exception · the limit forbids**.

**Rejected.** **`safe`/`unsafe`** — imports Rust's unearned safety connotation onto a
weaker default. **`noopt`/`nocheck` as the axis name** — optimization is a separate axis
(its own `no_opt` limit), and `nocheck` is narrower than the essence (it omits the
admission of raw operations). **`trusted`** (the "transfer of obligation" framing) —
`unchecked` is an established precedent (C# `checked`/`unchecked`, Rust `get_unchecked`)
that reads without explanation. **`@`-attribute spelling** — wrong category (a fixed
contract-changing region, not an open-set representation attribute).

### CG-7. `unchecked` is a scoped verification mode (not grant-only)
*spec: Codegen §3 (spec/90), Assembly (spec/80)*

**Why.** `unchecked` is a **scoped verification mode parallel to `comptime`**. Within
`unchecked expr` (one expression) or `unchecked { … }` (a block) the **checked-guard
family is dropped** and each guarded operation takes its **hardware-defined** behavior —
never UB (I11): `+`, `−`, `*` and unary `−` **wrap** (two's-complement); division by zero,
`MIN / -1`, `MIN % -1` and an over-width shift take the hardware fault/result (CG-13);
indexing skips the bounds check;
narrowing truncates and float→int is hardware-defined; alignment-requiring access skips
the alignment check. The scope **additionally permits the genuinely-raw operations**
ill-formed in checked code (pointer arithmetic, `int↔ptr`, reading a raw-union member
that is not definitely active / reading `uninit`, raw control transfer, raw `asm`) — these
have no checked counterpart and were a
grant under either model — and, in an `unchecked { … }` block, **binding a `scoped` call
result to a local** (transient-only in checked code, Memory §5.3.1; staleness is then the
author's, hardware-defined). **Granularity is the programmer's choice** (one line or a
region); inside it, correctness is the author's responsibility — the trade for terseness
over per-operation visibility — and action-at-a-distance is bounded by I11. The mode is
**non-transitive by default** (a call encapsulates its own value/trap) but **observable
on demand**: a function may read the mode via `comptime if verify.checked` (a comptime
fact, like target/build gating), making it **mode-polymorphic** — instantiated per the
call site's mode. This is how **library code** — a container, a library-defined numeric
type such as `uint(N)` — makes its *own* checks drop inside `unchecked` without the
compiler recognizing them: the library declares the mode-dependence (CT-11). The built-in
guard family needs no such declaration — the compiler emits it, so the compiler drops it
(CG-13). There is **no
re-entry and no `checked` keyword** — by symmetry with the phase axis there is no
`runtime` to re-enter from inside `comptime`, so there is no `checked { }`; the
`unchecked` scope *sets* the mode and `verify.checked` only *reads* it. To get a checked
result inside an `unchecked` scope, use the everywhere-available policy operations
(`checked_*`, `saturating_*`, `wrapping_*`, `overflowing_*`) or move the operation out.

**Rejected.** **The prior grant-only model** (checked ordinary ops inside the scope, with
distinct explicit unchecked spellings) — it preserved line-level transparency and matched
Rust's `unsafe`, but left indexing/narrowing/float→int with no writeable unchecked form
(incomplete at the FND-3 bar) and clashed with the `comptime`-block precedent. The
mode-flip accepts weaker line-level transparency **inside a lexically-visible scope** (the
word `unchecked` is always present and `no_unchecked`-auditable) in exchange for
completeness and symmetry (precedent: Zig's `@setRuntimeSafety(false)`).

---

## Overflow and float

### CG-8. Integer overflow is defined — trap by default, wrap under `unchecked`
*provenance: the policy families enumerated by the v1 built-in catalogue; `checked_div`
reports a zero divisor as `None` · spec: Codegen §3
(spec/90); Concurrency §6.3; Stdlib appendix §4.4*

**Why.** Per I11 overflow is **defined, not UB**, extending the checked guards (CG-2) to
arithmetic. **By default (checked) it traps** — everywhere except inside an `unchecked`
scope — via an inline check + a defined trap (cost = the checked-guard family, I2); the
checked-overflow set is `+`, `−`, `*`, unary `−` (negate MIN), and `MIN / -1` (division
overflow), with division-by-zero and shift-≥-width as separate guards that also trap.
**`MIN % -1` is not in the set**: its mathematical remainder `0` is representable, so a
checked `%` yields `0` and does **not** trap. The compiler guards the divisor `== -1` case
— the same compare as the `MIN / -1` guard — and produces `0` without executing the divide,
so no target faults on it (`x86_64`'s `idiv` would raise `#DE`). *Rejected:* trapping on
`MIN % -1` — it fails an operation whose correct result exists, only because one ISA's
instruction computes quotient and remainder together.
**The guard is a checked-guard-family trap emitted by the compiler** — the same category
as the division-by-zero and bounds guards (CG-2), *not* something only a routed `u`/`i`
library operator supplies. It therefore holds in **every** context, including a bare or
`freestanding` program and the native backends that lower `+`/`−`/`*` directly without
routing through a library operator: the failure mechanism is the single direct inline
trap fixed by CG-13, identical in every context (a bare or `freestanding` program
included) and on every architecture. *Superseded:* an earlier reading let a std operator
supply a richer `panic` flavor while a bare context emitted a hardware trap — two
observable mechanisms for one guard, below the FND-3 bar. **Inside an `unchecked` scope
`+`, `−`, `*` and unary `−` wrap** (two's-complement) and the guards are dropped for the
whole scope, as a mode (CG-7), not per operation; `MIN / -1`, `MIN % -1`, division by zero
and an over-width shift then take the target's hardware-defined behaviour (CG-13; per-arch
appendix §2). **Explicit policy operations** exist everywhere without a grant —
`wrapping_*` (wrap), `saturating_*` (clamp), `checked_*` (→ `Option`), `overflowing_*`
(→ `(T, bool)`) — as ordinary prelude functions (OP-1), so wrap is available even outside
an `unchecked` scope. **Each family covers exactly the checked-overflow set**: `add`, `sub`,
`mul` on `uN`/`iN`, `neg` and `div` on `iN` — 20 names, `wrapping_add` … `overflowing_div`
(Stdlib appendix §4.4). A policy operation is **mode-opaque** (it does not read
`verify.checked`, CG-7), so its result is the same in both modes. Division by zero is a
separate guard, not overflow, so `wrapping_div`, `saturating_div` and `overflowing_div`
with a zero divisor trap in both modes; **`checked_div(a, 0)` is `None`**, as is
`checked_div(MIN, -1)`, because the `checked_` family's contract is "report failure as
`None`, never trap" — a caller picks `checked_` precisely to handle every failure as a
value, and a zero divisor is a failure of the call like any other. *Rejected:*
policy members for `%` (`MIN % -1` yields `0`, nothing overflows), for shifts (an over-width
count is its own guard and OP-6 rejected a wrapping/saturating count), for unsigned `neg`
and `div` (no member of the checked-overflow set), and for `uint(N)` (a library type; its
`unchecked` wrap already exists via CT-11) — each a name with no overflow to give a policy
to, or one more obligation for a library type, against PRIN-1. `checked_div` trapping on a zero divisor —
it breaks the family's contract (a `checked_` call that can still trap), and the cost of the
extra compare is the same compare the division-by-zero guard already emits. Giving
`wrapping_div` / `saturating_div` / `overflowing_div` a zero-divisor result too — their
result types have no slot for "no quotient", so any value would be invented.
`bitsN` has no arithmetic (only bit-operations); overflow concerns
the numeric interpretations `u`/`i`.

### CG-9. Float determinism — strict at comptime, no hidden contraction at runtime
*provenance: `fma`, the saturating conversion and the constants `NaN` / `Infinity` pinned by
the v1 built-in catalogue ·
spec: Codegen §3 (spec/90); Concurrency §7.3, §7.7; Stdlib appendix §4.4*

**Why.** Reproducibility (CT-1/CT-2) and predictability (I1) constrain floats.
**Comptime-float is strict IEEE-754**, round-to-nearest-even, **host-independent** (soft
emulation when needed; no x87 extended precision, no FMA contraction) → bit-for-bit
reproducible. **Runtime-float is the target's IEEE-754 without hidden
contraction/reassociation**: those change observable results, so the
semantically-preserving optimizer (CG-5) never applies them silently; FMA only on explicit
request; **subnormals preserved** (flush-to-zero / denormals-are-zero are non-IEEE and
target-dependent, so never enabled implicitly — they would break reproducibility).
**Float→int rounds toward zero**; out-of-range or NaN **traps** by default (checked, like
narrowing), `saturating_convert(T, f)` — the `saturating_*` family's conversion, in
`bitcast`'s argument order — rounds toward zero and clamps (NaN→0, ±∞→MAX/MIN) everywhere,
mode-opaque, and inside an `unchecked` scope `T(f)` is hardware-defined (no range/NaN trap).
**`fma(a, b, c)`** on `f32`/`f64` rounds once; where the selected features have no fused
instruction it lowers to a correctly-rounded software routine (the soft-float provision,
TYP-2/Types §3.3), never to `a * b + c` rounded twice — the result is bit-identical on every
target, which is the point of determinism. *Rejected:* a located compile error on a target
without a fused instruction — it would make `fma` unusable on the baseline `x86_64` and on
`i386` although a correct result is computable, and soft-float already sets the precedent of
visible-cost software arithmetic; a double-rounded fallback — it silently changes results.
**`NaN` / `Infinity`** (the Grammar §2.4 prelude constants) have fixed bit patterns, so a
`bitcast` of either is the same on every target and at comptime: `NaN` is the canonical
quiet NaN with positive sign and zero payload (`f32` `0x7FC00000`, `f64`
`0x7FF8000000000000`), `Infinity` is +∞ (`0x7F800000`, `0x7FF0000000000000`), `-Infinity`
the ordinary unary minus. Each is typed as an untyped float literal is (TYP-13: a
floating-point context, else `f64`), so no third typing rule exists; comparisons stay IEEE
754 (`NaN != NaN`). *Rejected:* leaving the NaN payload/sign to the target — two
implementations would disagree on `bitcast(u64, NaN)`; fixed-type constants (`f64` only, or
per-type `f32::NaN` names) — a second spelling where the literal rule already decides.

---

## Inlining

### CG-10. Predictable, uniform inlining + an explicit `@inline` marker
*spec: Codegen §3.5 (spec/90)*

**Why.** Every operator is an ordinary library function whose one-instruction body the
compiler inlines to that single instruction (I2, zero-cost; the wrapper is then DCE'd) —
that automatic, body-shape-keyed inline is the guaranteed minimum and stays. The defect
was an **undocumented extra restriction** the reference compiler layered on top: it
inlined the body **only when every argument was a bare ident/literal**, so a *compound*
operand (`j*j <= n` → `lt(n, j*j)`) left a native operator as a real `call` in a hot loop
— exactly the "magic" I1/I3 forbid (unpredictable from source + rule) and an I2 violation.
The fix has two parts. **(1) Drop the operand restriction**: a one-instruction operator
body inlines **regardless of operand shape** — a compound operand is bound to a fresh
local first (the same move the structural-comparison desugar already makes), so `a op b`
always inlines, predictably (I1/I2). **(2) Add an explicit `@inline` attribute** that
*generalizes* inlining beyond the operator floor: an `@inline` function is substituted at
every direct call site (arguments bound to fresh locals), emitting no `call` and no
standalone symbol unless its address is taken or it is `@export`ed (then an out-of-line
copy is also emitted). The native operators carry `@inline`, so the contract is a
**stated, source-visible fact** at the definition rather than an inferred floor.
`@inline` is **forbidden on a recursive function** (no finite substitution) — a Semantic
diagnostic. A related codegen rule: a comparison used in an `if`/`while` condition lowers
via **`setcc`→`jcc` fusion** (branch directly on `cmp` + a conditional jump, never
materializing the boolean) — fewer instructions and fewer registers, easing register
pressure for nested comparisons (deep inlined *value* expressions can still exceed the
scratch pool until a real register allocator lands — an implementation limit on placement,
never a reason to re-introduce a `call`).

**Rejected.** **`@abi(inline)`** (folding inlining into the calling-convention attribute) —
a category error: `@abi(value)` selects *how a still-called function is called* and holds
one mutually-exclusive value, whereas `@inline` *removes the call and the symbol* and must
**coexist** with an ABI (`@inline @abi(c)` is meaningful for an address-taken/exported
copy); different chapters own them, and I3 forbids one knob carrying two meanings.
**Keeping the operand restriction** — the "magic" itself, an unpredictable I1/I2-violating
fallback. **A general optimizer inline-hint** (cost-model driven) — non-deterministic
across implementations, breaking the I2 "reduces identically everywhere" guarantee for the
operator set.

---

## Layered codegen conventions

### CG-11. Codegen conventions are a package-closed, layered space
*spec: Overview §4.1 (spec/00), Codegen (spec/90); cf. MEM-5 (the ambient-allocator
application)*

**Why.** "No magic" is sharpened to its operational meaning: the **package + its manifest
+ the language conventions** form a **closed logical space that fully and unambiguously
determines the generated machine code**. Determinism is at the **package boundary**, not
the literal call site. Local self-description (a fact visible where it acts) stays a
priority — but one **traded** against ergonomics and real usage frequency *when the
omitted fact is an already-declared, already-visible binding*. This refines **I3**:
*hidden* = **not determinable from the closed package**; eliding a *declared* parameter and
filling it from a *lexically-visible* scope is **not** hidden (the dependency is in the
signature; the source is a named binding in scope). A codegen convention resolves in up to
**three layers**, each deterministic in the package: **(0)** a package convention (the
manifest), **(1)** a lexical scope, **(2)** the explicit call site. `unchecked { … }`
already lives in layer 1 (CG-6/CG-7); this **generalizes the shape**. A convention is
**never un-overridable at the call site** (layer 2 always exists) — the bottom of the
ladder remains the raw assembly ceiling (I4). The first application of this model — the
**ambient allocator** (an elidable `in` allocator parameter filled by an `alloc::with`
scope) — is articulated as **MEM-5** in `memory.md`; this entry owns only the layered,
package-closed *convention* aspect.

**Rejected.** Any codegen convention that is **not determinable from the closed package**
(invisible static state, or codegen that depends on caller context outside the signature)
— that is precisely what "hidden" now means and what the layered model forbids; every
layer is package-deterministic and layer 2 (the explicit call site) is always available so
no convention is un-overridable. (The ambient-allocator-specific rejected alternatives —
a manifest/process-static global default, a compiler-derived implicit parameter threaded
by call-graph fixpoint, an `in out` allocator, a general "ambient value by type", and an
`alloc::current()` accessor — are recorded with MEM-5.)

## The `no_abstractions` surface

### CG-12. `no_abstractions` admits only the 1:1 assembly surface — the allow-list
*provenance: refines FND-10 (the limit itself), CG-6 (orthogonal to verification) · spec:
Assembly §1 (spec/80), Overview §3 (spec/00), Control Flow §2.3 (spec/60)*

**Why.** FND-10 defines `no_abstractions` as "asm-only: only 1:1 GAS correspondence, no
structured abstractions", but the **FND-3 bar** requires the boundary to be a *decision
procedure*, not a slogan — two independent toolchains must admit the **same** translation
units. The operative criterion is Overview §3's: under the limit **every emitted runtime
instruction is one the programmer wrote, 1-to-1 with GAS**. A construct is admitted **iff**
it emits either **exactly the instruction(s) it names** or **no runtime instruction at all**
(it is erased); a construct the compiler would *synthesize* into runtime instructions the
programmer did not write is forbidden. This resolves the boundary construct-by-construct.

**Admitted — the 1:1 surface + its framing + erased comptime:**
- **Instruction intrinsics** (Assembly §2) in call and UFCS form — each emits exactly one
  GAS instruction (§3) — including the synthetic destination-first forms (`remq`, `setcc`,
  `fltsd`, …, §4) and the raw escape **`asm(…)`** (§4; native here — the unit is already raw).
- **Operands as ordinary expressions** (§5): registers (§6); immediates (comptime values);
  the `at(…)` memory operand (§7) — the *sole* memory-access form; `@label` code-points and
  a label's address as `bitsN` data; reloc/TLS and SIMD UFCS decorations (§8).
- **Raw control transfer** — `jmp` / conditional branch / indirect, direct or via a label —
  the unit's **native** control surface here, needing **no** `unchecked` grant (Control Flow
  §2.3 / CF-6): the structured analyses (§9) do not apply.
- **Data definitions** (§9): `bitsN`/typed constructors/strings/arrays, `[u8; N]` (inline or
  `embed`), placed by ordinary data bindings into sections (`@section`/`@endian`).
- **The function frame** (§2.2): a `fn` — `@abi(naked)` **or** ordinary ABI-framed — over
  **local variable bindings** that materialize operand values (`mut out : u64 = a`), the
  instruction mutating its destination local, and that destination **returned by the
  enclosing function** (`return <local>`). The prologue/epilogue and slot placement are the
  *fixed, spec-defined* frame (Memory §2.5 / Codegen §3.4), fully determined by the source —
  not a compiler-chosen abstraction. A binding's initializer must itself be a surface
  expression (an intrinsic result / register / immediate / local / data constructor).
- **Comptime** in full (`comptime if`/`for`/`match`, `when`-gating, immediates, `embed`) —
  **permitted and erased** (FND-10, I7): it emits no runtime artifact of its own machinery,
  so it cannot break the 1-to-1 property. This is the one wholesale exemption.
- Compile-time **attributes/effectors** (`@reg`, `@section`, `@endian`, `@abi`, `@label`,
  `@align`, …) — they shape emission; they are not themselves emitted.

**Forbidden — structured abstractions, a compile error located at the site:**
- **The library value-operators as surface syntax** — arithmetic `+ - * / %`, comparison,
  bitwise, `and`/`or`, unary `-`/`not`. A native operator is an ordinary *library function*
  (Type System §3.1) whose 1:1 body is the instruction intrinsic, but in checked code the
  compiler emits the operation's **checked guard** at the operator site (CG-13), and a
  guarded checkable operation is **not 1:1** (Assembly §3). Write the intrinsic
  (`add(out, b)`) directly — an intrinsic never carries a guard.
- **Structured control flow** — `if` / `while` / `for` / `loop` / `match` / `break` /
  `continue` (expression or statement): the *other* control level (Control Flow §1), lowering
  to compiler-chosen compare+branch sequences (§10). Use labels + raw transfer.
- **The `?` tryable operator** — auto-application is **off under `no_abstractions`** (glossary);
  it is a structured branch + failure propagation.
- **Calls to non-instruction functions** — an out-of-line `call` to an ordinary function is a
  multi-instruction ABI sequence the programmer did not write; functions are a structured
  construct (Overview §3). Only instruction intrinsics and `asm(…)` are the admitted "calls".
- **Structured data access** — struct/enum **field access `s.f`**, enum construction and
  pattern `match`, tuple projection, array/slice index `a[i]`, and range-slices. `struct`/`enum`
  are structured data (Overview §3); the raw equivalent is an `at(base=, index=, scale=, disp=)`
  operand plus a load/store intrinsic. (Data *definition* via constructors and `[u8; N]` stays
  admitted, §9 — it is runtime *access through the aggregate abstraction* that is not 1:1.)
- **The pointer word-functions `ptr`/`deref` and pointer arithmetic / `int↔ptr`** — genuinely-raw
  operations (CG-7) **not** in Assembly §5's positive operand surface, where memory is reached
  only through `at(…)` + a load/store intrinsic (so the surface has one memory-access form).

**Locality & enforcement.** Like `no_unchecked` (CG-6), the limit binds the **unit's own
source** — every function the unit declares is checked — and is **not** transitive over the
cross-unit call graph (FND-12): a `no_abstractions` unit may still link separately-audited
units that use abstractions internally, joined at the ABI seam. It is **orthogonal to
verification** (CG-6): it removes the guarded operations by removing their *surface*, and does
not touch the `checked`/`unchecked` axis. **Terminology is pinned to "structured
abstractions"** — Overview §3's triad (raw / structured / comptime); FND-10 and the glossary,
which read "structural", are aligned to it.

**Rejected.** **A slogan-only definition** ("no abstractions", left to the implementation) —
fails FND-3: two toolchains would disagree on `return`, field access, `ptr`/`deref`, or a bare
call. **Naked-only** (forbidding the ABI-framed frame and `return`) — Assembly §2.2 explicitly
frames the surface inside an ABI function over local operands with a returned destination, so
the frame is admitted; a naked-only reading could not express an ordinary one-instruction
operator body. **Admitting `ptr`/`deref` as single instructions** — blurs the boundary (it
would equally admit `s.f` as a single load); memory stays the `at(…)` operand so the surface
keeps exactly one memory-access form. **Forbidding comptime** — the discarded tier-ladder error
(FND-10): comptime is erased (I7), is needed everywhere, and cannot break the 1-to-1 runtime
property.

### CG-13. One failure mechanism for the whole checked-guard family — the direct inline trap
*provenance: refines CG-2/CG-8; narrows CT-11 to library code · spec: Concurrency §6.1;
Assembly §3; Codegen §3.5; Types §4.5, §7; Memory §5.4; Stdlib §4.1; Stdlib appendix §3,
§3.6, §4.1, §4.2, §5.2.1, §6, §8; per-arch appendix §2, §3.2*

**Why.** The guard family was specified failure-by-failure: overflow traps "via an inline
check", division by zero and the over-width shift "also trap", and the per-arch appendix
pins one instruction for "every checked guard **whose failure mechanism is** the direct
inline trap". That conditional left the mechanism open exactly where hardware already
faults: on `x86_64` a zero divisor raises `#DE` by itself, so one implementation can let
the machine fault (`SIGFPE`) while another emits a guard and traps (`SIGILL` from `ud2`) —
two conforming implementations with observably different failures for the same program,
which is the FND-3 bar, not a quality-of-implementation detail. It also made the
failure of *one* family depend on which arithmetic the source happened to use.

**The rule.** For the built-in numeric interpretations (the native-width `uN`/`iN`/`fN`
operators and conversions, and the built-in array/slice indexing) the compiler emits the
guard **at the operation site**; an instruction intrinsic never carries one (it is 1:1,
Assembly §3). Every checked-guard failure — the checked-overflow set (`+`, `−`, `*`, unary
`−`, `MIN / -1`), division by zero, an over-width shift, the bounds / alignment /
narrowing guards, and a false `@require` predicate (TYP-12) — fails through **one**
mechanism: the direct inline trap, emitted as the per-architecture instruction in per-arch
appendix §2. The guard is emitted **before** the operation, so a target that would fault by
itself never gets the chance: the observable failure is the same instruction everywhere.
The trap calls no `panic`, no hook, no OS exit, and does not unwind. `panic` remains an
ordinary library operation that library code may call; it is not a second mechanism for a
built-in guard. Inside `unchecked` the guard is absent and the operation takes its
hardware-defined behaviour (CG-7) — including the machine's own fault on a zero divisor or
on `MIN / -1`, which is then hardware behaviour, not a language guard. Only `+ − * unary −`
*wrap* under `unchecked`; the divide and shift guards drop to target-specific behaviour
(Concurrency §6.2; per-arch appendix §2), which is why the checked mechanism has to be
uniform. **`MIN % -1`** is the one divide case that is guarded without being a failure: in
checked code the compiler compares the divisor with `-1` (the compare the `MIN / -1` guard
already makes) and yields `0`, the correct remainder, without executing the divide (CG-8);
under `unchecked` it is hardware-defined (`x86_64`/`i386` `idiv` faults, `#DE`; `aarch64`,
`aarch32` `sdiv` and RISC-V yield `0`).

**Library-defined numeric types fail through `trap()`.** The guard family above is the
compiler's for the built-in interpretations; a library-defined numeric type — `uint(N)`
(TYP-10), and so the wider-than-native `u128` ≡ `uint(128)` — implements its
operators in library code, where the compiler emits no guard. Such an operator whose check
mirrors a family member (division or remainder by zero, overflow) MUST make the check under
`comptime if verify.checked` (CT-11) and fail through the prelude **`trap() -> Never`** —
the same direct inline trap instruction (per-arch appendix §2), with no message, no hook, no
OS exit and no unwinding, valid in every context including `freestanding` and bare — never
through `panic`. So `x / 0` fails identically for `u64` and for `uint(256)`. `trap` passes
PRIN-1: it is not a second name for `panic` but a capability `panic` cannot give — a library
failing by **exactly** the mechanism of a built-in guard, in every context; `panic`'s
behaviour is a target/limits property (a hook, stderr + exit) and so cannot be that.
Without `trap` a library could only re-spell the per-arch instruction itself in raw `asm`
under `unchecked`, once per architecture — duplicating appendix §2 in every library and
not following it when an architecture or backend is added.

**A library container's checked access fails through `trap()` too.** The same holds for a
library container's checks that mirror a family member — bounds, alignment, narrowing:
`Vec`'s indexing (`at`, `index_set`, `index_range`) and `set`, a library-implemented
`str` / `String` byte access (its range and code-point-boundary check), the arena's `get`
bounds check, and every `comptime if verify.checked { … }` bounds guard in the prelude and
stdlib catalogue. Each is made under `comptime if verify.checked` (CT-11), so it drops
inside `unchecked` exactly as the built-in bounds guard does, and fails through `trap()`,
never `panic`. So `xs[10]` fails identically whether `xs` is an array, a slice or a `Vec`,
in every context including `freestanding` and bare — one failure mechanism for one guard,
whatever implements it, the reasoning of the rule above. `panic` / `assert` stay ordinary
library operations for program **logic**: an explicit user assertion, `unwrap` on `None` /
`Err` (a defined-failure `panic`), and a library precondition that mirrors no family
member. *Superseded:* the earlier text of this entry that called "a container's own bounds
check" an ordinary `panic` — it contradicted the Stdlib appendix's trapping `Vec` indexing
and `get`, and it would make `xs[10]` fail by two mechanisms depending on the type of `xs`.

**Routed operators.** Operators are library functions (TYP-5), so an implementation may
lower `a / b` into a prelude operator body. The guard attaches to the **operation**, not to
that body: the compiler emits it at the operation site regardless of routing, and the
routed body carries none. A library-supplied `panic("division by zero")` inside the operator
is therefore not a permitted variant — it makes the observable failure depend on whether the
prelude operator is in scope, and it reintroduces the IO/hook path CG-8's superseded
sentence used to allow. `panic`/`assert` in ordinary library logic (an explicit assertion,
`unwrap` on `None`, a precondition that mirrors no family member) is unaffected and stays an
ordinary library call; a container's bounds check is not such logic (above).
*Superseded:* two other placements the text once carried — the built-in guard as library
code in the prelude operator body under `comptime if verify.checked` (the first CT-11 and
the Stdlib appendix's "mode-polymorphism guarantee"), and the guard as a property of the
instruction intrinsic (the per-arch tables' guard column) — three locations for one guard;
the operation site is the only one.

**Cost.** One compare-and-branch ahead of the divide/shift, in the same category as the
overflow guard (I2): the guard family already sets that visible ceiling, and a divide is
the most expensive integer operation on every supported arch, so the added relative cost is
the smallest of any guard in the family. A semantics-preserving optimization may still
eliminate a *provably* unnecessary guard (Codegen §3.5) — e.g. a divisor proven non-zero.

**Rejected.** Sanctioning the hardware fault where one exists (`#DE` → `SIGFPE`) — it makes
the observable failure arch- and operation-dependent, gives the same source two different
exit codes on two targets, and leaves `aarch64`, where the divide instruction returns 0
instead of faulting, needing an inserted guard anyway. Routing the failure through `panic`
— adds IO, hooks and a `freestanding`-dependent path to a mechanism that must exist in every
context (CG-8). A per-guard choice of mechanism — restates the ambiguity this entry closes.

### CG-14. A non-ISA backend has no `Arch` identity; `target.arch` matches none of them
*provenance: refines CG-4 (the backend abstraction) and FND-6 (additivity) · spec: Manifest
appendix §3.1 (`Arch`), §3.2; Tooling §2.7; Codegen §6; Assembly §10*

**Why.** `Arch` is the closed set of v1's **register ISAs**, and the per-arch appendix's charter binds every
name in it to ISA data — a trap instruction, register classes, an instruction-intrinsic table. WASM has none
of those by construction (Assembly §1: no registers, no raw goto, no code addresses), and it reaches its
target through a different lowest form (WAT, not GAS). So a WASM build has no `Arch` variant to be, and the
manifest cannot select it as the `arch` of a `Machine` variant. That left one thing genuinely open: what `target.arch`
*evaluates to* when such a backend is driven directly. An implementation answered it by folding
`target.arch` to `x86_64` — so `when target.arch == Arch.x86_64` was TRUE while emitting WASM, and a library
could not write an architecture gate at all. Any answer is better than a lie, but the answer has to be
written down or every backend picks its own.

**The rule.** `target.arch` on a non-ISA backend equals **no** `Arch` variant: `==` against every variant
folds false, `!=` folds true. Ordinary portable code keeps working — `when target.arch == Arch.x86_64 { … }
else { … }` selects the else — and the arch-gated x86 body is correctly absent rather than emitted into a
machine that cannot run it. The price is stated rather than hidden: a `match` over `target.arch` is
exhaustive only across ISA targets, so portable code must give it a **default arm**; without one it is
ill-formed on a non-ISA backend.

**The same rule covers the container-bound facets** (and since TOOL-18 such a backend arrives as a new
`Machine` variant, so "no container" is structural rather than an exception). Three v1 rules added later are written over facts a
non-ISA backend does not have, and each is scoped rather than left to be broken when such a backend lands:
the **artifact name** (TOOL-11) is indexed by the four register-ISA containers, so a non-ISA backend's
artifact naming is fixed with that backend; **`target.container`** matches no `Container` variant, exactly as
`target.arch` matches no `Arch`; the **entry symbol stated to the linker** (TOOL-12; Codegen §6) presupposes
a link step, so on a backend without one what the entry *is* belongs to that backend's rules; and
**`startup`** (crt objects and libc) is inapplicable there. Scoping them now keeps the additivity promise
(I10/FND-6): the backend arrives as an addition, not as a rewrite of v1 norms.

**Rejected.** Adding `Arch.wasm32` — it would import the per-arch appendix's ISA obligations (a trap
instruction, an instruction table, register classes) into a target that has no registers, and Assembly §10's
architecture sections would have to describe something that is not an architecture. Folding `target.arch` to
some ISA (what the implementation did) — it makes a true statement false and silently selects code written
for another machine. Making a `target.arch` query a diagnostic on such a backend — it breaks every portable
`when target.arch == …` gate, which is the construct the facet exists for.
