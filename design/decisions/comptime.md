# Decisions — Comptime, generics, protocols, introspection

**Why the language is the way it is** for comptime: the evaluator, generics, constraints,
introspection, and the comptime facts the language exposes. Each entry records the
*rationale* and the *rejected alternatives*; the normative *what* lives in the spec
(linked per entry), the *how* in the compiler. IDs are theme-prefixed (`CT-N`) and
**append-only** — assigned in articulation order, never renumbered. The `provenance:` line
records what an entry refines and where it lands in `spec/`.

---

## The evaluator and the universe

### CT-1. Comptime is the main language run earlier — power, not totality
*spec: Comptime §2*

**Why.** Comptime is not a separate weak sublanguage; it is **the same language executed
earlier** (uniformity). It is therefore Turing-complete — arbitrary recursion and loops.
The invariants the language actually rests on — erasure (I7), zero-cost (I2),
predictability — **do not depend on the evaluator's power**, so there is no reason to
cripple it. Protection against non-termination is a **step budget**, not a totality
proof: budget-exhaustion is a clean error, never a silent partial result, and the result
is **independent of the budget's magnitude** (a deterministic value, or "exhausted").
Steps are counted in **abstract steps, not wall-clock**, so evaluation is reproducible
across machines, and the per-operation weights are **implementation-defined** (the budget
is *not* a conformance point — FND-3 needs only that any evaluation that *completes* within
budget yields the identical result everywhere). The budget tiers like any other
non-semantic policy: compiler default → manifest → a local raise at a specific
computation, with a CLI override for one-off debugging.

**Rejected.** *Total comptime* — a separate weak sublanguage plus a large
termination-checker that rejects valid programs, for no invariant gain. *Pinning a
normative abstract-step VM* — a heavy artifact that over-constrains implementations
without serving FND-3, since the magnitude-independence already makes completed evaluations
portable.

### CT-2. The comptime universe — what may exist, what may flow in, and the erased boundary
*spec: Comptime §1, §3*

**Why.** Comptime must be **pure and reproducible** so that type-producing functions can
be memoized (TYP-4) and two builds agree. That fixes its boundaries on every side. Its
**inputs are controlled**: the target machine model (I6), manifest config, the program's
own source, plus a declarative `embed` of a file's content (hashed into the build) —
arbitrary I/O (filesystem outside `embed`, network, clock, environment, randomness) is
**forbidden** because it would make the build non-reproducible. Its **memory** is a
comptime arena in the compiler's address space that is **erased**: comptime allocation is
not runtime allocation and does not fall under I3, and a value that crosses into runtime
materializes into **static** (`.rodata`/`.data`), never the runtime heap — so I3 is
preserved. **Purity** (no observable side effects, only local mutation, no comptime global
mutable state) gives both memoization and determinism. And **phase separation** bounds
what comptime can know: types, sizes, layouts, constants, symbol identity — but **never**
runtime values or runtime addresses.

---

## Generics and protocols

### CT-3. Generics are comptime parameters — generic types and monomorphized functions
*spec: Types §generics, Comptime §4*

**Why.** Because types and functions are ordinary values executed earlier (TYP-1), generics
need **no separate machinery**. A **generic type** is just a comptime function returning
`type` (`Vec(T: type) type { return struct {…} }`), **memoized by its arguments** so that
`Vec(u8)` twice is the *same* type (TYP-4). A **generic function** is a function with comptime
parameters, **monomorphized** per set of comptime arguments — zero-cost, no dynamic
dispatch (I2). Comptime type parameters are **inferred** from runtime-argument types where
determinable (`max(x, y)` infers `T`), so the generality is free at the call site.

### CT-4. Constraints are comptime predicates — no trait/interface machinery
*spec: Types §generics, Comptime §6*

**Why.** A constraint is **an ordinary comptime function `fn(type) bool`** (TYP-1) — there is
no separate system for traits, interfaces, or inheritance. A predicate is a full comptime
computation over a type, so it can express *anything*: "T has `<`", "T has a field
`x: u64`", "size < 16", "all fields are integers". Composition is just boolean comptime
operations (`and`/`or`/`not`); a named composite is a predicate that calls predicates.
**Explicit constraints are encouraged** — they document the requirement and **localize the
error at the call site with the constraint's name**, curing the C++/Zig pain of an error
deep inside an instantiated body — but a **structural fallback works without them** (using
an unsupported operation simply fails at instantiation). **Both styles are admitted**:
structural (a form check) and nominal (a check of an explicit brand-marker, TYP-4), chosen
per constraint.

### CT-5. `when` — the single comptime existence-gate
*spec: Comptime §6, §7*

**Why.** `when <comptime-predicate>` guards a declaration: it exists/is valid *only when*
the predicate holds. One word **unifies** two needs that are the same thing underneath —
generic constraints (a predicate over parameters → an error when used for an unsuitable
`T`) and platform/feature gating (a predicate over the machine model → simply absent on
an unsuitable target). In both, "does not exist when false; access to the non-existent is
an error". Crucially, `when` is the **sole existence-gate**: an `@`-attribute is a *total
endomorphism* over its construct (CT-10) — it may transform or trap the construct but can
never remove it from existence — so availability/absence is expressed **only** by `when`,
never by an attribute. Several predicates compose as a boolean comptime expression, or as
a named composite.

**Rejected.** `where` (covers only constraints — does not unify with gating) and `with`
(does not fit gating) — both rejected in favor of the one word `when`, ending the
`where`/`when` confusion.

---

## Introspection and the comptime execution surface

### CT-6. Type introspection — one `typeinfo(T)` reify and a capability query
*spec: Comptime §5, Types §introspection*

**Why.** Generic library operations (`eq`, `hash`, serialization, `Display`) are built by
**auto-inference over a type's structure** — iterating `typeinfo(T).fields` — and
monomorphized to zero cost. So the language exposes **one** builtin reify, `typeinfo(T)`,
yielding a comptime `TypeInfo` value (a tagged enum: `Scalar`, `Struct`, `Enum`, `Array`,
`Union`, `Pointer`, `Function`, `Brand`), inspected by **ordinary comptime code** — one
reify instead of a scatter of query-builtins, with `size`/`align` derivable from it. That
derivation is the surface: **`T.size()` / `T.align()`** (and `typeof(v).size()` for a value)
are prelude **word-functions** — compiler-folded like `typeinfo`/`bitcast`, since a type's
layout is a fact of the machine model (I6), not library code — and a field's `offset` is read
from `typeinfo(T).fields`. The method form `T.size()` is the face: **no flat prefix name is
promoted and there is no `mem::size`/`align`/`offset` query surface** (MEM-7), keeping the
"one reify, no scatter of query-builtins" rule.
A `Scalar` carries its **numeric interpretation** (`kind`: `Bits`/`Uint`/`Int`/`Float`/
`Bool`/`Char`) *alongside* its `bits`, so a structural derive can render a leaf correctly —
distinguishing signed from unsigned, and `bool`/`char` from a same-width integer. A
`Brand`'s `marker` is a **`str`** (the brand's name, matched against a string literal),
and a brand naming a **scalar interpretation** (the prelude `uN`/`iN`/`fN`/`bool`/`char`/…)
reifies as `Scalar{kind}`, while a **user** nominal brand reifies as `Brand{underlying,
marker}`. Introspection is **comptime-only and fully erased — no runtime RTTI** (I2/I3/I7):
runtime type-tags are explicit programmer data, never a hidden type table. A second
builtin completes the picture — a **capability query** (`resolves`/`compiles`), needed
*because* operators are free functions (TYP-5): "`T` has `<`" means "is `lt(T,T)`
resolvable", which `typeinfo` cannot show. Its cost is context-dependence (it sees only
what is visible at the check point); the nominal/brand style is available when strictness
is wanted.

### CT-7. The comptime/runtime boundary is an explicit `comptime` marker
*spec: Comptime §1, §8*

**Why.** The default phase is **runtime**; `comptime` is an explicit, bare keyword (a
fundamental phase modifier) marking a binding (a comptime-known constant), a parameter
(must be comptime-known), or a block (run at compile time). The boundary is explicit so
that **erasure stays predictable** (I1/I7): one can read where the phase line falls.
`type`-values are **comptime by nature** (CT-2), so a `T: type` parameter needs no marker;
the explicit `comptime` is required only for *non-type* comptime parameters (`comptime N:
usize`, e.g. an array size). `comptime` thus implies *immutable and known at compile time*
— a phase distinct from mutability (MEM-7).

**Rejected.** *Auto-inference of phase* — it makes erasure unpredictable, against I1/I7. (A
*constant expression* — literals combined by the built-in scalar operators — is comptime by its
form, CT-13: a rule about the text, not inference.)

### CT-8. The comptime execution surface — shape the emitted code, don't create runtime control
*spec: Comptime §8*

**Why.** Comptime control is **explicit partial evaluation**: it *shapes the emitted code*
rather than producing runtime control, keeping the phase boundary visible (CT-7). The
surface is the runtime control forms run earlier: `comptime { … }` (computed entirely at
compile time, the value baked in), `comptime if` / `comptime match` (conditional
compilation — only the selected branch/arm is emitted; `comptime match` is the multi-way
form, the comptime analog of `match`, whose arm pattern-bindings are comptime values —
the natural flat dispatch over `typeinfo(T)`, replacing a nested `comptime if (match
typeinfo(T) { … })` pyramid), and `comptime for x in collection` (**unrolling** — the body
emitted once per element as straight-line code, no runtime loop). In a *runtime* function
these shape emission (a comptime condition/scrutinee/loop variable over a body that may
contain runtime operations); inside an already-comptime context `for`/`if`/`match` are
comptime without a marker. The evaluator is the **same** one (CT-1's budget, CT-2's universe) — a `comptime
for` over a finite collection terminates, a comptime-`while` is bounded by the budget.
`comptime if` (a choice in a body) and `when` (a declaration guard, CT-5) are two surfaces
of one comptime condition — kept distinct, not conflated.

### CT-9. Comptime variant matching — the enum analog of comptime field projection
*spec: Comptime §5*

**Why.** Structural derives (`Eq`/`Ord`/`Hash`) are library code over `typeinfo`, made
expressible for **structs** by comptime field projection `a.(f)` (TYP-9). The **enum** half
stayed inexpressible and so a **FND-3 gap**: a value's variant is reachable *only* by `match`
(never a positional `e.N`, unsound when another variant is active, I11), and a `match`
pattern names its variant by **source literal** — useless when the variants are known only
as iterated comptime data `typeinfo(T).variants`. So a generic `eq`/`lt`/`hash` could
compare structs but not enums, and the reference compiler had to keep built-in
`eq_walk`/`cmp_walk` enum knowledge — the very duplication the library derives exist to
remove (and which operators-as-library, TYP-5, forbids). The fix composes **existing
syntax with the comptime universe, no new keyword** (OP-1): a **comptime variant pattern
`T.(v)`** (the pattern-position analog of `a.(f)`) resolved at comptime to the concrete
variant pattern and emitting identical code (zero-cost I2, no RTTI I3), binding the
matched variant's **whole payload as one value** so a derive can recurse with the operator;
and **`comptime for` in match-arm position**, unrolling to one arm per variant — exhaustive
by construction. A companion **faithfulness fix**: a `Variant.payload` must describe the
*actual* payload (the **tuple** of all component types for a multi-component variant, not
just the first; the **anonymous struct** of the fields for named components, TYP-18), so
`T.(v)(p)` binds a faithful value.

**Rejected.** *A payload projection `a.(v)` outside a match* — reintroduces the unsound
positional access the match-only rule forbids (I11), since `a` need not be variant `v`.
*Positional comptime-arity bindings `T.(v)(x0, x1, …)`* — the count varies per variant and
the generated names are unutterable in the body, whereas one whole-payload value recurses
through the existing operator. *Raw-byte enum compare* — padding and pointer/aggregate
payloads make it non-structural.

---

## Comptime facts and effectors

### CT-10. Configuration via comptime effectors over a fixed, closed lever set
*spec: Comptime §7, Types §layout*

**Why.** Machine-level configuration, contracts, and effects are expressed by **comptime
functions composing a fixed core set of *levers*** — primitives codegen honors, each with
a documented lowering rule (I1). **Composition is open** (library- and prelude-defined,
indistinguishable, OP-1), but the **lever set is closed**: the programmer cannot add a new
honored codegen behavior (I10) — the ceiling is levers ∪ the `asm(…)` escape (I4), and
nothing emits behavior without a documented rule. The two surfaces — prefix `@name(args)`
and UFCS `construct.name(args)` — are the **same comptime function**, and a prelude
attribute is indistinguishable from a library one (no privilege). **Roles are by shape**,
one mechanism: a `fn(K) bool` is a predicate (for `when`/`comptime if`); an
**endomorphism `fn(K) K`** over the decorated construct-kind is a lever/attribute. An
effector is **total over its construct** — it transforms or rejects-with-a-checked-trap,
but it **cannot remove the construct from existence** (that is solely `when`'s province,
CT-5). `@` is **static and never wraps an expression** — it decorates a *construct*
(type/field/signature/binding/code) at compile time, so an effector sees structure, never
a runtime value, preserving the phase boundary (CT-7); runtime operation effects
(`atomic::load`, `volatile::store`) are ordinary builtin calls, not `@`. This recasts the
earlier layout attributes and the attribute catalog as comptime functions over levers,
unprivileges `Option` (the enum niche-fold is one general compiler layout rule), and closes
the "how libraries configure types in comptime" thread.

### CT-11. Verification mode is an observable comptime fact — `comptime if verify.checked`
*provenance: scope narrowed to library code by CG-13; a library check mirroring the family
fails through `trap()` (CG-13) · spec: Comptime §7, §8, §9*

**Why.** The built-in guard family is the compiler's: it emits each guard at the operation
site and drops it inside `unchecked` (CG-13). **Library code** has checks of its own — a
library-defined numeric type's division by zero (`uint(N)`, TYP-10), a container's bounds
check — that should equally **fail by default and drop inside `unchecked`**, while
`unchecked` is **lexical and non-transitive** (CG-7) — it does not reach into a callee's
body. These collide: a library `/` called under `unchecked` would keep its check and trap.
The compiler **cannot** just drop it, because that would require it to *recognize* an
arbitrary library `if … { trap() }` as "the division check" — numeric knowledge the
compiler holds only for its own closed guard family (TYP-2/TYP-5). So the **library must
declare** which code is verification-mode-dependent, read by a mechanism the language
already has. The resolution: the **verification mode is a comptime-known fact**,
queryable through the unified comptime guard exactly like target gating and build flags,
exposed as the comptime boolean **`verify.checked`**. The `unchecked` scope *sets* the
mode; `verify.checked` *reads* it (distinct — still no `checked` keyword, no re-entry). A
function that **observes** the mode (`comptime if verify.checked { <guard> }`) is
**mode-polymorphic** — instantiated per the call site's mode (≤ 2 variants, the unused one
DCE'd, zero-cost I2, the choice visible I1/I3): a library operator is written **once** and
describes both modes. A library check that mirrors a member of the built-in family — a
library-defined numeric type's overflow or division/remainder by zero, a container's bounds
check (`Vec` indexing, the arena's `get`) — fails through `trap()`, the
built-in guard's own instruction (CG-13), so `unchecked xs[i]` drops it and a checked
`xs[10]` fails alike for an array and a `Vec`; ordinary library logic (an explicit
assertion, `unwrap` on `None`) may `panic`. A function that does **not** observe it is
**mode-opaque** — fixed at definition, a caller's `unchecked` does not alter it (CG-7
non-transitivity preserved as the default). The mode reaches **only** explicitly-observing functions, and only one level — no
hidden deep propagation (I3). This works uniformly for inlinable and non-inlinable
operators (a multi-word `u128 +`, a `Matrix +` still selects its mode-correct
instantiation).

**Rejected.** *(B) Blanket transitivity of `unchecked` into callees* — strips a callee's
own checks from the outside (action-at-a-distance, breaking the locality CG-7 rests on).
*(C) A dedicated `@verified` marker* — a new entity for what `when` plus the existing
comptime↔verification parallel already express, against minimalism (OP-1). *Passing the
mode as an explicit parameter* — pollutes every operator signature and is not how
target/build facts are read. *(A, superseded) "mandatory inlining makes the guard lexical
so `unchecked` drops it"* — unsound: `unchecked` drops only the built-in
checkable-operation family, not an arbitrary library `if … { panic("…") }`, so an inlined
guard would survive; recognizing it would reintroduce built-in numeric knowledge. CT-11
makes the drop **explicit in the library** instead. *Superseded:* CT-11 as first written
made the **built-in** operator guards library code in the prelude operator bodies (`add(out,
b); if out < a { panic("overflow") }`) — a second placement of the guard family, which
CG-13 fixes at the operation site; CT-11 now covers library code only.

### CT-12. A failed checked guard at comptime is a located diagnostic, never a deferred trap
*provenance: refines CT-1/CT-2/CT-11 and CG-8/CG-13 · spec: Comptime §2.6; Concurrency §6;
Declarations §7.1*

**Why.** The evaluator executes the same language as run time (CT-1), so the checked-guard
family applies to it — but the family is specified in terms of a *trap*, and there is no
process to trap during compilation. Left unstated, an implementation could (and one did)
carry the failure forward and let it trap when a run-time path reaches the constant, so
`comptime K : u64 = M + 1` (with `M` the `u64` maximum) builds green and dies later, or on a path never taken, dies never. That
is both a worse diagnostic and an implementation fork: another implementation may
reasonably reject it at the declaration. The compiler already *evaluated* the operation and
already knows the answer is unrepresentable; the only question is whether it says so.

**The rule.** A checked-guard failure during comptime evaluation — the checked-overflow
set, division by zero, an over-width shift, bounds / alignment / narrowing, a false
`@require` predicate (TYP-12), an explicit `panic` / `trap()` or failed `assert` reached by comptime
code — is a **located compile-time diagnostic** at the operation's site. Nothing is
deferred into the emitted program and no substitute value (wrapped, saturated, zero) is
materialized. This is the same shape as the existing budget-exhausted and
recursion-too-deep diagnostics (§2.2): a comptime evaluation either yields a deterministic
result or fails visibly, never silently.

**`unchecked` at comptime.** The evaluator drops the guards inside an `unchecked` scope
just as run time does, with one forced difference: an operation whose unchecked behaviour
is *hardware-defined* — division by zero, an over-width shift, `MIN / -1`, `MIN % -1` — stays a
diagnostic,
because the evaluator has no hardware whose behaviour it could reproduce and I11 forbids
inventing one. The two's-complement wrap of `+`, `−`, `*`, unary `−` is *defined*, not
hardware-defined, so it evaluates reproducibly. None of this reaches an integer constant expression, which
is evaluated exactly and range-checked whether or not it sits in an `unchecked` scope (CT-14).

**Rejected.** Deferring the failure to run time — turns a known-at-compile-time error into
a conditional runtime death, and makes two implementations disagree on whether a program
builds. Silently wrapping at comptime — the checked default exists precisely to not do
that, and it would make a constant's value depend on where it was computed. Making the
evaluator model a specific machine's fault behaviour for `unchecked` division by zero —
imports target-dependence into an evaluation the spec requires to be reproducible across
machines (§2.2).

### CT-13. A constant expression is comptime by its form — spec#39
*provenance: refines CT-7 (runtime by default, no phase inference) and CT-12 (a failed guard at comptime is
  a diagnostic); reads Type System §2.3, §9.1, Assembly §1.1, §5, §7 and CG-12/CG-15/CG-16 · spec:
  Comptime §1.1, §1.6, §2.6, §11; Type System §2.3, §9.1; Declarations §3.4; Memory §4.5; Assembly §1.1, §5,
  §7*

**Question.** Comptime §1.1 makes evaluation runtime by default and rejects phase inference;
Type System §9.1 makes an *integer literal* a compile-time number but says nothing of an
operator applied to literals. The grammar has no negative literal — `-16` is unary `-` over
`16` — yet the specification writes `at(base = rbp, disp = -16)` (Assembly §7; the arch
appendix's addressing tables) and expects an encodable displacement, Assembly §5 makes an
immediate a comptime value, and Memory §4.5 reads `p == -1` through unary `-` over `1`. Read
strictly, §1.1 made `-16` and `0 - 1` runtime values, so not immediates; the other reading was
phase inference. CG-15 and CG-16 were written to hold either way and left the question open.

**Decision (owner, 2026-11-08).** **A constant expression is comptime by its form.** A
constant expression is a scalar literal, or scalar literals combined (with parentheses) by the
built-in scalar operators — arithmetic, comparison, bitwise, `and`/`or`, and the unary `-`,
`~`, `not` — the arithmetic, comparison and bitwise ones each naming the operation of the prelude's
operator function for its operands' type (evaluated as CT-14 says), `and`/`or`/`not` the built-in short-circuit forms over `bool` (OP-2). It is
comptime like the literals it is built from. The rule reads the text, not
values or uses, so it is not phase inference: an expression naming a binding, a call or a place
does not become comptime by it, and a named operation-function such as `shl` is a call. The
expression is checked against its context type like a literal, with the literal default where
there is no context, and its evaluation is a comptime one, so a failure in it is a located
compile-time diagnostic (CT-12), never a run-time trap: `x : u8 = 200 + 100` is rejected where
it is written (how the check reads is CT-14). A named constant (`NaN`, `Infinity`, a `comptime` binding) is not
a literal, and the range forms are not scalar operators, so neither makes a constant expression.
`disp = -16` and `0 - 1` are immediates.

**Why.** The specification's own examples need a negative immediate, and with no negative
literal the only spelling is an operator over a literal. A rule fixed by the form keeps the phase
line readable off the source (I1/I7), which is what CT-7 rejects inference to protect; checking
the result at compile time keeps I11 (no silent wrap) without a run-time guard on a value the
compiler already knows.

**Not decided here.** *How* the check reads where checking each operator in the context type
and checking the mathematical result differ — `-1` at an unsigned type, `-128 : i8`,
`1 - 2 + 3 : u64`, `unchecked` — and how a floating-point constant expression is evaluated are
CT-14 (spec#62). The context type an immediate slot or an
`at(…)` field gives was spec#63, since settled by CG-18 (its slot type; spec#80 for the fields
no v1 type describes). Admission under `no_abstractions`, whose list forbids the
value-operators as surface syntax (Assembly §1.1, CG-12), is spec#59.

**Rejected.** *Only literals and unary `-` on a literal are comptime* — a second, narrower notion
of constant, with `0 - 1` and `1 + 1` runtime for no reason a reader can see. *Everything that is
not a literal is runtime, a negative immediate being spelled through a `comptime` binding* —
contradicts the specification's own `disp = -16` and makes the commonest displacement a two-line
form.

**Consequences.** Comptime §1.1 gains the rule and §11's conformance item 1 names it; Type
System §2.3, §9.1, Declarations §3.4, Memory §4.5 and Assembly §5, §7 point to it, and
Assembly §5's register example
moves to a runtime operand (`y - 1`). CG-15 and CG-16 no longer leave the phase of `0 - 1` open;
Assembly §1.1 and CG-12 mark spec#59 open. Comptime §1.6 notes that a scalar comptime operand is
an immediate, and §2.6's `unchecked` paragraph points to §1.1.

### CT-14. An integer constant expression is exact, then checked; a floating-point one is per operation — spec#62
*provenance: refines CT-13 (how its check reads) and CT-12; reconciles Type System §2.3 with Memory
  §4.5 and Concurrency §6.5 (CC-8) · spec: Comptime §1.1, §2.6, §11; Type System §2.3, §9.1; Memory
  §4.5; Concurrency §6.5, §8 item 7; Declarations §3.4*

**Question.** CT-13 checks a constant expression "against its context type like a literal", which
has two readings that disagree on ordinary code: each literal takes the context type and each
operator runs with its fixed-width semantics (so `-1 : u8` is 255 and `-128 : i8` and
`1 - 2 + 3 : u64` are rejected), or the expression is evaluated over exact integers and only the
result is checked (so the opposite on all three). Type System §2.3 called comptime numbers
mathematical while Memory §4.5 read `p == -1` per operator. `unchecked` (does it make `0 - 1 : u8`
wrap?) and floating-point constant expressions (step by step in the IEEE format, or exactly with
one rounding and then range-checked) were open with it.

**Decision (owner, 2026-11-08), verbatim.** "Option 1. Integer constant expressions are evaluated
over exact integers and the RESULT is checked against the context type like a literal (Types §2.3):
`-1 : u8` is rejected, `-128 : i8` is accepted, `1 - 2 + 3 : u64` is 2; `unchecked` does not change
a constant expression. Floating-point constant expressions are evaluated per operation in the
context's IEEE format, so a constant means the same value it would at run time (I1). Memory §4.5's
`p == -1` and every unsigned `-1` in the spec are rewritten to the type's maximum."

**Derived (where every open reading agrees; not the owner's wording).** "`unchecked` does not change
a constant expression" covers it inside an `unchecked` scope and as `unchecked`'s operand: it is
evaluated and checked the same way, so `unchecked (0 - 1)` is the constant expression `0 - 1`.
"Per operation in the IEEE format" makes overflow ±∞, not a diagnostic (Concurrency §7.4). Applying the decision needs
the domain and the operators' exact meaning, which Comptime §1.1 states where every reading agrees. A `bool` literal, a comparison or their `and`/`or`/`not`
combination is boolean; otherwise an expression containing a floating-point literal is
floating-point (its context's format, or `f64` with no context; a non-floating-point context is a
compile error, TYP-13), and one of integer literals only is
integer at every context but a floating-point one. An operator over an operand outside its domain
(`1 + (1 < 2)`, `not 1`) is ill-formed, there being no prelude operator for it; a character literal
under an arithmetic or bitwise operator is not an integer constant expression and keeps CT-13's
reading. Over exact integers `+ - *` and unary `-` are the
mathematical operations, and `/`, `%`, `& | ^` over non-negative operands the quotient, remainder
and bitwise operations; an unsigned context does not select the unsigned operations, and only the
result of the integer expression as a whole is range-checked, not an integer operand of one of its
integer operators (`x : u8 = 300 - 100` is 200); a zero divisor is a diagnostic, having no
exact result. `~` of a non-negative integer is negative or ill-formed under every reading still
open, so `~0 : u8` is rejected for the decision's own reason that `-1 : u8` is, and so is the mask in
`addr & ~7` at `usize` (written `addr & ~usize(7)`). Three cases the decision does not settle stay
open: an integer-literal expression more than a single literal (unary `-` included) at a
floating-point context, whole or as an operand (`x : f64 = 7 / 2`, `-0`, `1.0 + 7 / 2` — spec#87); the domain and range check of a
comparison's operands (`1 / 2 < 0.4`, `99999999999999999999 > 0` — spec#88); and `/`, `%` with a
negative operand and the bitwise operators over negative values (`-7 / 2`, `~7`, `-8 & 255` —
spec#91). A type's maximum is written with existing means —
the literal (`255`) or, width-independently for `usize`, `~usize(0)`: not a constant expression
(`usize(0)` is a construction), so `~` is the fixed-width `usize` operation and yields every bit set.
The prelude names no `MAX` constant (Modules §4.1.1's `i64::limits` is an illustrative module path), and
this decision adds none.

**Why.** It follows the owner's wording and Type System §2.3: a constant expression means one number
whatever type it later meets, with no silent wrap (I11) and no second meaning of `-` chosen by the
context type. It makes `iN`'s minimum expressible as a literal-built constant. The floating-point
rule keeps constant folding invisible (I1): a constant expression and the same operations at run
time give the same bits.

**Rejected.** *Per operator in the context type* — `-1 : u8` silently 255, `-128 : i8` and
`1 - 2 + 3 : u64` rejected, and the flow of context into each operand left to be written. *Exact,
with a negative result accepted at an unsigned type as its two's complement when the magnitude
fits* — `-255 : u8` would be 1, a second meaning of `-` decided by the context type. *Floating-point
evaluated exactly, rounded once, and range-checked like a literal* — a constant would then differ in
its last bit, or be rejected where run time gives ±∞, from the same expression at run time.

**Consequences.** Comptime §1.1 states the rule and the per-operator meanings, §2.6 that `unchecked`
does not reach a constant expression, §11 item 1 both; Type System §2.3 confines "mathematical
numbers" to literals and constant expressions (a typed comptime value keeps its type's operators,
§2.6) and §9.1 gives the examples; Memory §4.5's sentinel becomes `p == ~usize(0)` and `p == -1` is
ill-formed; Declarations §3.4 cites the rule; Concurrency §6.5, its §8 item 7 and CC-8 note that the
unsigned negation is the operator over a typed operand, not a constant expression's.
