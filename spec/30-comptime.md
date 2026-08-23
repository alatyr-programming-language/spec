# Alatyr Language Specification

## Chapter — Comptime and Generics

> **Status — draft (under review).** This chapter is a draft and is not yet
> accepted; the specification as a whole is under development. See the status note
> at the top of the Overview chapter.

This chapter defines **compile-time evaluation** (*comptime*), the
**comptime/runtime boundary**, **generics** (generic types and functions),
**constraints** as predicates, **type introspection**, **capability queries**, and
**protocols**. It builds on the Type-System chapter's foundational principle —
*a type is a value, and a generic is a compile-time function returning a type*
(Type System §1) — and makes the execution model of that principle normative.

Requirement keywords (**MUST**, **MUST NOT**, **SHOULD**, **MAY**) follow RFC 2119
(Overview §6). This chapter gives the **concrete syntactic forms** it introduces as
grammar fragments (§10) and the **static semantics** of conditionally-present code
(§9); the *consolidated* EBNF and lexical rules (token spellings, identifiers,
whitespace, comments) are assembled in the Grammar chapter, and the forms here
conform to the unified declaration model of the Declarations chapter.

The governing invariants are **I7** (compile-time computation is fully erased
before run time and never leaks into runtime cost; the boundary is explicit and
checkable), **I2** (zero-cost), **I3** (nothing hidden), and **I1** (transparent,
predictable lowering).

---

### 1. The comptime/runtime boundary

#### 1.1 The default is runtime; `comptime` marks the exception

Evaluation is **runtime by default**. The bare keyword **`comptime`** marks the
compile-time exception. This follows the language-wide axis convention (Overview;
the phase axis is *runtime* (unnamed default) / `comptime` (marker) /
`no_comptime` (limit)). Phase auto-inference is **rejected** — it would make
erasure unpredictable, against I1/I7.

#### 1.2 What `comptime` marks

`comptime` applies to:

- a **binding** — a compile-time-known constant (`comptime PI : f64 = 3.14`);
- a **parameter** — one that MUST be supplied compile-time-known
  (`comptime N : usize`);
- a **block / scope** — evaluated wholly at compile time (§8.1).

#### 1.3 `type` values are comptime by nature

A value of kind `type` is comptime **by nature** (Type System §1); the marker is
implied. A parameter `T : type` therefore does **not** require `comptime` — it is
already comptime. An explicit `comptime` is needed only for **non-`type`** comptime
parameters (e.g. `comptime N : usize`, an array length).

#### 1.4 `comptime` implies immutable and known

`comptime` entails **immutable** and **compile-time-known**. This is a **phase**
property, distinct from mutability (Memory §3): a `comptime` value is fixed
because it is computed before run time, independent of any `mut` permission.

#### 1.5 Erasure

All comptime computation — type-level computation, comptime blocks, comptime
control flow, predicates, introspection — is **fully erased before run time** (I7).
It produces types, sizes, layouts, constants, and emitted-code shape; it leaves
**no** runtime cost and **no** runtime artifact of its own machinery. The boundary
is **explicit and checkable** by the programmer (§1.1).

#### 1.6 Crossing the boundary; phase separation

A comptime value that is **used at run time** is **materialized into static
storage** (`.rodata`/`.data`), **never** the runtime heap (Memory §2;
preserving I3). This is the bridge from the comptime arena (§2.5) to the runtime
memory model.

**Phase separation:** comptime knows **types, sizes, layouts, constants, and symbol
identity**; it does **not** know **runtime values or runtime addresses**. A runtime
value cannot be read at comptime; a comptime value can be baked into runtime.

---

### 2. The comptime evaluator

#### 2.1 The same language, executed earlier

Comptime is the **same language**, executed before run time (uniformity; Type
System §1). It permits **arbitrary recursion and loops** — it is Turing-complete,
not a weaker total sub-language.

#### 2.2 The step budget

Non-termination is bounded by a **step budget**, not by a totality checker
(rejected: a totality checker would reject valid programs, and our guarantees —
erasure, zero-cost, predictability — do not require totality).

- The budget is a **resource ceiling, not a meaning parameter**: the result does
  **not** depend on the budget's magnitude — a comptime evaluation either yields a
  deterministic result or fails with **"budget exhausted"**. There are **no** silent
  partial results.
- The budget is counted in **abstract steps, not wall-clock time**, so it is
  **reproducible** across machines.
- Configuration follows the standard tiering (the non-semantic-policy tiering):
  compiler default → manifest (package default, reproducible) → a **local raise** at
  a specific evaluation. A CLI override applies to one run only (debugging; a
  package's success MUST NOT depend on a CLI flag).

**The budget is not a portability/conformance point.** The abstract-step count is
**implementation-defined** — the spec does **not** fix per-operation weights. This does
not threaten FND-3, because the *result* is budget-magnitude-independent (first bullet):
for any comptime evaluation that **completes** within the in-effect budget, all
conforming implementations produce the **identical** result. A conforming
implementation MUST provide a default budget **large enough to evaluate the v1
prelude/stdlib and the comptime of any ordinary program** (a generous default is
recommended); the manifest may raise it reproducibly. A program that **exhausts** its
budget gets a *budget-exhausted* diagnostic; because the step count is
implementation-defined, two implementations are **not** guaranteed to exhaust at the
same point — so a program that runs **near the limit is non-portable** (raise the
budget or reduce the work). The boundary is a resource limit, never a silent change of
meaning.

**Recursion depth is a second such resource.** An evaluator that runs comptime calls on
its own (native) stack MAY additionally bound **comptime call-nesting depth**, to turn
runaway recursion into a clean diagnostic before a host stack overflow. This bound is the
same **kind** of limit as the step budget: **implementation-defined**, never a change of
meaning, and a *recursion-too-deep* diagnostic — and an implementation MUST set it deep
enough to evaluate the v1 prelude/stdlib and the comptime of any ordinary program (layout
and predicate code is shallow). A program that recurses **near** the bound is likewise
**non-portable** (two implementations need not cap at the same depth); deep-but-finite
comptime recursion that approaches it should be restructured (e.g. onto an explicit
heap-allocated worklist) rather than relying on a particular implementation's depth.

#### 2.3 The comptime universe

Comptime values comprise: scalars (integer / `bool` / floating-point / `char`),
`type`, aggregates (arrays / structs / tuples / enums / raw unions), function-values,
and byte-sequences / strings. A raw-union value retains its active-member fact in the
evaluator, subject to the same fixed function-boundary and place-tracking rules as run time
(Type System §6.3).

#### 2.4 Inputs — controlled and reproducible

A comptime evaluation MAY read only **controlled, reproducible** inputs:

- the **target machine model** (I6) fixed by configuration;
- the **manifest** configuration;
- the **program's own source**;
- a controlled **`embed`** — declarative, reproducible baking of a file's contents,
  hashed into the build.

Arbitrary I/O is **forbidden** at comptime — filesystem access outside `embed`,
network, wall-clock, environment variables, and randomness — because it would make
the build non-reproducible. A conforming implementation MUST reject such access.

#### 2.5 Comptime memory and purity

Comptime evaluation uses a **comptime arena** in the compiler's address space,
which is **erased**. Comptime allocation is **not** runtime allocation and is
**not** governed by I3. A value crossing into run time is materialized per §1.6.

Comptime functions are **pure**: no observable side effects, only **local**
mutation, and **no** comptime global mutable state. Purity makes type-generating
functions **memoizable** (§3.1) and the whole evaluation **deterministic**.

#### 2.6 A checked-guard failure at comptime is a diagnostic

The evaluator runs the **same** checked-guard family as run time (Concurrency §6,
CG-13), but it has no process to trap in: a guard that fails during comptime evaluation
is a **located compile-time diagnostic** at the operation's site. This covers the
checked-overflow set, division by zero, an over-width shift, the bounds / alignment /
narrowing guards, a false `@require` predicate (Types §8.1), and an explicit `panic` /
failed `assert` reached by comptime code. The failure is **never** deferred into the
emitted program, and no wrapped, saturated, or otherwise adjusted value is materialized
in its place: a comptime binding whose initializer overflows (`K : u64 = MAX + 1`) is
rejected where it is written, not left to fail when some run-time path reaches it.

Inside an `unchecked` scope (CT-11) the evaluator drops the guards exactly as run time
does, with one difference forced by there being no machine: an operation whose unchecked
behaviour is *hardware-defined* rather than *defined* — division by zero, an over-width
shift, and **`MIN / -1`** (Concurrency §6.2) — remains a **diagnostic** at comptime,
because the evaluator has no hardware behaviour to reproduce and I11 forbids inventing
one. The two's-complement wrap of `+`, `−`, `*` and unary `−` is fully defined, so those
evaluate normally and reproducibly under `unchecked`.

---

### 3. Generics

#### 3.1 Generic types

A **generic type** is a comptime function whose result is a `type`
(`Vec := fn(T : type) -> type { … }` — an ordinary type-returning function, not a
special introducer; SYN-6). It is **memoized by its arguments**: evaluating `Vec(u8)`
twice yields the **same** type (type identity, Type System §4.1), enabled by purity
(§2.5).

The validity decorator is one such type-function: `require(U, pred)` is memoized by
the immediate `U`, the predicate's callable identity, and every comptime capture value
that specializes it (Type System §8.1). A named predicate and a separately written
function literal may expose the same `fn(in U) -> bool` call interface without having
the same callable identity.

#### 3.2 Generic functions

A **generic function** is a function with **comptime parameters**
(conceptually `max := fn(comptime T : type, a : T, b : T) ...`). It is
**monomorphized** per comptime-argument set: each distinct set produces one
specialized instance. There is **no** runtime dispatch and **no** `dyn` — the
result is **zero-cost** (I2), identical to a hand-written specialization.

#### 3.3 Inference of comptime type-parameters

Comptime **type**-parameters are **inferred** from the types of runtime arguments
where determinable: `max(x, y)` infers `T` from the types of `x` and `y`. A
parameter that cannot be inferred MUST be supplied explicitly.

#### 3.4 Lowering

Monomorphization is the lowering rule (I1): the compiler emits one concrete
instance per distinct comptime-argument set; comptime parameters and type
parameters **do not exist at run time**. Two instantiations with identical comptime
arguments share one instance (§3.1).

---

### 4. Constraints as comptime predicates

#### 4.1 A constraint is an ordinary predicate

A **constraint** is an ordinary comptime function of type `fn(type) bool` (Type
System §1). There is **no** separate trait / interface / inheritance machinery. A
predicate is full comptime computation over a type and can therefore express
**anything**: "T has operator `<`", "T has a field `x : u64`", "T's size < 16",
"all of T's fields are integers".

#### 4.2 Composition

Predicates compose with the **boolean comptime operators** `and` / `or` / `not`; a
**named composite** predicate is simply a predicate that calls other predicates.

#### 4.3 Explicit versus structural

- **Explicit constraints are encouraged**: they document the requirement and
  **localize the error to the call site**, naming the violated constraint — versus
  the C++/Zig failure mode of an error deep inside the instantiated body.
- A **structural fallback works without an explicit constraint**: using an
  operation the type does not support fails **at instantiation** (a defined
  compile error), not silently.

#### 4.4 Two styles

Both are available, chosen **per constraint**:

- **structural** — check the type's *shape* (it has the field/operation/size);
- **nominal** — check an explicit **brand marker** (Type System §5.4), for cases
  where structural matching is too permissive.

How a predicate **looks inside** a type is type introspection (§5).

---

### 5. Type introspection

#### 5.1 `typeinfo(T)`

`typeinfo(T)` reifies a type into a **comptime value** of the prelude type
`TypeInfo` — a tagged enum with cases including `Scalar{bits, kind}`,
`Struct{fields : [Field]}`, `Enum{variants}`, `Array{elem, n}`, `Union{variants}`,
`Pointer{…}`, `Function{…}`, `Brand{underlying, marker : str}`, and `Str{}` (the `str`
byte view, §3.6) — the marker is the brand's *name*, matched against a string literal; a
prelude scalar-interpretation brand reifies as `Scalar{kind}` instead (appendix §4.1) —
where `Field = {name, type, offset, mutable}`.

`Union{variants}` uses the same `Variant` schema as `Enum{variants}`. Raw-union member
entries appear in source declaration order; `Variant.payload` is `None` for zero
components, `Some(T)` for one component, and `Some((T0, T1, ...))` for two or more
components (the member's one anonymous tuple payload, Type System §6.3/TYP-11). Type
information reports no active member: activity
is compile-time analysis/evaluator state, never RTTI or a hidden discriminant.

A `TypeInfo` value is inspected by **ordinary comptime code** (`match`, iteration);
size and alignment are derivable from it (and exposed as the word-functions
`T.size()` / `T.align()`, Memory §4.3) — they are **not** members of `TypeInfo` itself.
The member surface is exactly the cases and their fields listed above: reading a member
that is not among them (`typeinfo(T).size`, `typeinfo(T).align`, a misspelling) is a
**located compile-time diagnostic**, never folded to `0` or another default. The **inverse** — constructing a type — is done with
the Type-System constructors (§5 there). One **reify** builtin replaces a scatter of
per-query builtins.

#### 5.2 Comptime-only — no runtime RTTI

`typeinfo` is **comptime-only** and **fully erased** (I2/I3/I7). There is **no**
runtime type information and **no** hidden type table: any runtime type tag is
**explicit programmer data**.

#### 5.3 Use

The prelude builds **generic operations by deriving over structure** — e.g. `eq`,
`hash`, or serialization implemented by iterating `typeinfo(T).fields` — which
monomorphize to **zero-cost** code (§3.4).

#### 5.4 Comptime field projection — `a.(f)`

A structural derive iterates `typeinfo(T).fields` and must **read the corresponding
field** of a value. The only field-access form, `a.name` (Grammar §3.4), takes a
**source-literal** name — useless when the field is known only as iterated comptime data.
`a.(f)` bridges this: it **projects the field of aggregate `a` identified by the comptime
value `f`** — a `Field` (from `typeinfo(T).fields`) or its `str` `name`.

It is **resolved at comptime** to the named field access `a.<f.name>`: the **same place**,
the same type, and the same per-field write permission (Memory §3.2), and it emits
**identical** code to a source-literal `a.name` — **zero-cost** (I2), with **no** runtime
field-by-name lookup and **no** RTTI (I3). `f` MUST resolve to an actual field of `a`'s
struct type; otherwise it is a **defined compile error**. In a **runtime** body `a.(f)`
occurs inside a `comptime for` **unrolling** (§8.3), so each unrolled copy projects **one
concrete field** and the projection is comptime-known by construction. This is the form
that makes the v1 structural derives (`eq` / `lt` / `hash`, Stdlib appendix §2.6) ordinary
**library code over `typeinfo`** (TYP-9).

```alatyr
eq := fn(T : type, in a : T, in b : T) -> bool {
  comptime for f in typeinfo(T).fields {       # unrolled once per field
    if a.(f) != b.(f) { return false }         # a.(f) ≡ a.<f.name>, monomorphized
  }
  return true
}
```

#### 5.5 Comptime variant matching — `T.(v)` (CT-9)

`a.(f)` reads a **struct** field by a comptime descriptor; the **enum** half of a structural derive needs the dual.
An enum value's variant is reached **only** by `match` (Type System §6 — never a positional `e.N`, unsound when another
variant is active, I11), and a `match` pattern names its variant by a **source literal** (`Color.Red`) — useless when
the variants are known only as iterated comptime data `typeinfo(T).variants`. Two comptime forms bridge this, parallel
to `a.(f)`:

- **The comptime variant pattern `T.(v)`** — in pattern position, resolved at comptime to the concrete variant pattern
  `T.<v.name>` (`v` a `Variant` from `typeinfo(T).variants`, or its `str` `name`). It emits **identical** code to a
  source-literal `T.<name>` (zero-cost, I2; no RTTI, I3). `T.(v)(p)` binds the matched variant's **whole payload** as
  one value `p` — a **tuple** when the variant has several components (§5.1: `Variant.payload` is that whole payload) —
  so a derive compares it by **recursing with the operator** (`p_a == p_b`). This is **distinct** from the positional
  literal form `T.Add(x, y)` (which binds each component separately); `T.(v)` MUST resolve to an actual variant of
  `T`'s enum type, else a defined compile error.
- **`comptime for` in match-arm position** unrolls to **one arm per iteration** (the arm-position analog of its
  statement-position unrolling, §8.3), so a `match` over a generic enum is **exhaustive** by construction.

```alatyr
# (inside eq, comptime-selected when typeinfo(T) is Enum)
match a {
  comptime for v in typeinfo(T).variants {     # unrolled once per variant
    T.(v)(pa) => match b {                      # T.(v) ≡ T.<v.name>, monomorphized
      T.(v)(pb) => return pa == pb              # same variant: compare payloads (recurse)
      _         => return false                 # different variant ⇒ not equal
    }
  }
}
```

Both compose existing syntax (the variant-pattern `.`-spelling, the `comptime for` unrolling) with the comptime
universe — **no new keyword** (OP-1) — so the **enum** case of the v1 structural derives (Stdlib appendix §2.6) is
ordinary library code over `typeinfo`, and the compiler keeps **no** built-in structural-comparison knowledge.

---

### 6. Capability queries and protocols

#### 6.1 Capability queries

Because operators are **free functions** (Type System §4.5), "does `T` support `<`"
is the question "does `lt(T, T)` **resolve**?" — which `typeinfo` does not answer.
This is answered by **builtins** (no new keywords): a `resolves(...)` query
(does this operation resolve for these types? → `bool`) and/or a `compiles(expr)`
query (does this expression type-check? → `bool`). These have the special semantics
of *attempting a typecheck* using ordinary call syntax; they subsume the
structural-operational predicates of §4.

**Evaluation context (precise).** A capability query performs its typecheck *attempt*
with the **ordinary name-resolution and type rules in effect at the query's lexical
position** — at module level the order-independent module scope (FN-2/FN-3); inside a
function the lexical scope visible at that point, including shadowing. It **does not
evaluate** its operand and has **no** side effects or emission (pure); it is therefore
**deterministic**, and it **counts toward the step budget** like any comptime work.
That is what "context-dependent" means: the answer is exactly what an ordinary call
written at that point would resolve to. For strictness independent of context, use the
**nominal** style (brands, §4.4).

#### 6.2 Protocols — language constructs depend on shapes, not names

A **protocol** is a comptime **shape**: a predicate plus the operations a construct
requires, checked via §6.1. Language constructs depend on **protocol shapes, not on
prelude type names** (the principle recorded with the `?` operator):

- a **condition** (in `if` / `while` / `and` / `or`) requires a **truthy** shape;
  the prelude's `bool` satisfies it;
- the **`?`** operator (Control Flow §8.2; the try operator) requires a **tryable**
  shape — "split into success / failure, extract the success, build the early-exit
  failure"; the prelude's `Option` and `Result` satisfy it;
- the **`for`** loop / iteration requires an **iterator/iterable** shape — a `next`
  that yields an **optional** shape (present / absent, **no** error payload — distinct
  from tryable); the prelude's ranges, slices, and arrays satisfy it (Control Flow §6;
  Stdlib appendix §2.4).

The prelude types are **not privileged by name**; any user type that satisfies the
shape gets the construct. This keeps the core language decoupled from specific
library types (a kernel may define its own tryable error type and use `?`), and is
resolved entirely at comptime + monomorphized — **zero-cost, no RTTI**.

---

### 7. `when` — the unified comptime guard

#### 7.1 Guarding a declaration

`when <comptime-predicate>` guards a declaration: the declared entity **exists / is
valid only when the predicate is true**. Referencing an entity whose guard is false
is a **defined compile error** ("does not exist when the predicate is false"). When
the predicate is **false**, the guarded declaration's signature and body are parsed
but **not** name-resolved, type-checked, or emitted, and **may** reference entities
that do not exist for the current target or type arguments — this is the basis of
target gating (the precise two-phase rule is §9).

#### 7.2 What it unifies

`when` unifies two things that are the same mechanism:

- **generic constraints** — a predicate over the parameters; using the entity for an
  unsuitable `T` is an error at the use site (with the named predicate, §4.3);
- **target / feature gating** — a predicate over the machine model; the entity is
  **absent** on an unsuitable target.

Both reduce to "the entity does not exist when the predicate is false."

#### 7.3 Multiple predicates

Combine predicates with a **boolean comptime expression**
(`when Addable(T) and Comparable(T)`), or name a **composite** predicate (§4.2).

#### 7.4 `when` versus `comptime if`

`when` (guards a *declaration's* existence) and `comptime if` (selects a *branch* in
a body, §8.2) are the **two surfaces** of comptime conditioning. Both are kept and
**not conflated**.

#### 7.5 The verification mode as a comptime fact — `verify.checked` (CT-11)

The **verification mode** (`checked` / `unchecked`, the verification axis; Type System
§4, decisions CG-6/CG-7) is a **comptime-known fact**, queryable by `when` / `comptime
if` exactly like target/feature gating — exposed as the comptime boolean
**`verify.checked`** (true in a checked context, **false** inside an `unchecked`
scope). The `unchecked { … }` / `unchecked expr` scope **sets** the mode (Type System
§4.2; Control Flow); `verify.checked` only **reads** it — reading is not the forbidden
re-entry, so there is still no `checked` scope keyword (CG-6).

A function whose body reads `verify.checked` is **mode-polymorphic**: like a function
generic over a comptime value, it is **instantiated per the verification mode of each
call site** (at most two variants — checked / unchecked; the unused one is
dead-code-eliminated; the choice is transparent and zero-cost, I1/I2/I3). A function
that does **not** read it is **mode-opaque** — its verification is fixed at definition
and a caller's `unchecked` does not alter it (`unchecked` stays non-transitive **by
default**, CG-7). The mode reaches **only** the observing functions, one level deep (a
call inside a mode-polymorphic body runs at that body's own lexical mode unless
re-scoped).

This is what lets a **library-defined** operator or checked conversion (the
kernel/prelude split, TYP-2/TYP-5) drop its guard under `unchecked` without the compiler
recognizing the guard. A native operator is written once and describes both modes —
the guard lives in a `comptime if verify.checked { … }` block, present in the checked
instantiation and comptime-absent in the unchecked one:

```alatyr
add := fn(in a : u64, in b : u64) -> u64 {       # the native `+` body
  mut out : u64 = a
  unchecked { addq(out, b) }                     # the raw instruction always wraps
  comptime if verify.checked { if out < a { panic("integer overflow") } } # overflow guard — only when checked
  return out
}
```

`x + y` selects the checked instantiation (the guard traps on overflow); `unchecked(x
+ y)` selects the unchecked one (the guard is comptime-absent, leaving the wrapping
add — Concurrency §7 / CG-8). The same shape gives `char(n)` its code-point guard
(Type System §4.2; appendix §3.2). The two-phase checking of the mode-gated branch is
§9 (a `comptime if verify.checked`-false branch is parsed but not emitted in the unchecked
instantiation).

---

### 8. The comptime execution surface

Comptime control **shapes the emitted code**; it does **not** create runtime control
(explicit partial evaluation; the boundary is visible, §1).

#### 8.1 `comptime { … }`

A block evaluated **wholly at compile time**, yielding a baked comptime value.

#### 8.2 `comptime if` / `comptime match`

**Branch selection at compile time** (conditional compilation): **only the chosen
branch is emitted**; the others are parsed but **not** name-resolved, type-checked, or
emitted, and **may** reference entities absent on the current target (the precise
two-phase rule is §9). The controlling condition (`comptime if`) or scrutinee
(`comptime match`) MUST be comptime-known.

**`comptime match`** is the **multi-way** form — the comptime analog of `match` exactly
as `comptime if` is of `if`. It matches a comptime value against patterns (the same
patterns as a runtime `match`, §5.4 — variant + binding, literal, range, `_`),
comptime-selects the **first** matching arm, and emits **only** that arm (the others are
parsed but not name-resolved/type-checked/emitted, §9). An arm's pattern **bindings**
(e.g. the `b`/`k` of `Scalar(b, k)`, the `under`/`m` of `Brand(under, m)`) are
**comptime values** within the arm — like a `comptime for` loop variable. It is the
natural multi-way dispatch over a comptime value such as `typeinfo(T)`, replacing a
nested `comptime if (match typeinfo(T) { … })` pyramid with flat arms; a `comptime match`
over an enum (e.g. `typeinfo(T)`'s) is exhaustive on the same rule as a runtime `match`
(§5.4), the wildcard `_` arm covering the rest.

#### 8.3 `comptime for x in collection`

**Unrolling**: the body is emitted **once per element**, with `x` bound as a
**comptime value**. The result is straight-line code with **no runtime loop**.

#### 8.4 Marking

In a **runtime** context the `comptime` marker is explicit (§1). Inside a **comptime
context** (a `comptime` block or a comptime function) `for` / `if` / `match` are
**already comptime** and need no marker.

#### 8.5 Mixing

In a runtime function, comptime control **shapes emission**: the condition or
loop variable is comptime, while the emitted body MAY contain runtime operations.
This is explicit partial evaluation — the comptime parts are erased, the emitted
parts remain.

#### 8.6 Termination

The evaluator is the same throughout (§2): the step budget (§2.2) and the
universe/purity rules (§2.3–2.5) apply. A `comptime for` over a **finite** collection
terminates; a comptime `while` is **budget-bounded**.

---

### 9. Static semantics of conditionally-present code

`comptime if` / `comptime match` (§8.2) and `when` (§7) make code **conditionally
present**: a branch/arm may be unselected, or a declaration may be guarded-false. The
checking applied to such code is defined in **two phases**. A conforming implementation MUST
apply them exactly, so that two implementations accept and reject the same programs
(Overview §6).

#### 9.1 Phase A — always, target-independent

- Every region of code — selected or not, guarded-true or guarded-false — MUST be
  **lexically and syntactically well-formed** per the grammar (§10). Parsing is
  target-independent and does **not** require referenced names to exist.
- The controlling comptime **condition / predicate** MUST be a **well-typed comptime
  expression of type `bool`**. *When* it is evaluated depends on its free variables:
  - **closed** — every operand is comptime-known in the declaration's own scope → it
    is evaluated **immediately**, at the declaration / branch point (e.g.
    `when target.arch == Arch.x86_64`, a target gate);
  - **dependent** — it refers to comptime parameters of an enclosing generic that are
    **not yet bound** (e.g. `when Addable(T)`, a constraint) → it **cannot** be
    evaluated until those parameters are bound, so it is evaluated **per
    instantiation** (§9.2). Until then the guarded entity is neither selected nor
    rejected; it is *pending*.

  A controlling expression that can **never** be comptime-known — it depends on a
  **runtime** value — is a compile error. Each evaluation counts against the step
  budget (§2.2).

#### 9.2 Phase B — conditional (resolution, typing, emission)

Name resolution, type checking, constraint checking, lowering, and emission apply
**only** to:

- the **selected** branch of a `comptime if`;
- a `when`-guarded declaration whose predicate is **true** — and, for a generic, to
  each **instantiation whose arguments satisfy** the guard.

Code **excluded** from Phase B — a non-selected branch, or a `when`-false
declaration's signature and body — is **not** name-resolved or type-checked, and
**MAY** reference identifiers, types, functions, fields, or instructions that **do
not exist** for the current target or type arguments. This exclusion is precisely
what makes **target gating** and **per-target / per-type specialization** possible
(CT-5): a `when target.arch == Arch.x86_64` body may use x86_64-only instructions and is
simply **absent** (and unanalyzed) on another target; referencing the absent entity
at a use site is a compile error (§7.1). The build target is observable through
**`target.arch`** (compared against `Arch.<variant>`) and **`target.os`** (against
`Os.<variant>` — `Os.linux`/`Os.windows`/`Os.macos`/`Os.freebsd`/`Os.none`, the last
being freestanding); both fold to a comptime value, so `comptime if`/`when` can gate on
either (e.g. a hosted-vs-freestanding `panic`). The **build profile** is likewise
observable: **`build.debug`** (a comptime `bool`) and **`build.profile`** (the active
profile's name, a `str`) fold for the selected profile, so a `comptime if build.debug { … }`
debug gate is zero-cost in a release build.

#### 9.3 Consequences

- **Exclusion is wholesale:** an excluded region's own nested `comptime if` / `when`
  / declarations are likewise neither evaluated nor checked.
- A **generic** guarded by `when` is checked **per satisfying instantiation**
  (monomorphization, §3.4): a constraint violation surfaces at the **use site** with
  the named predicate (§4.3), while a type error in the body for an otherwise-
  satisfying argument surfaces **at instantiation** (the structural fallback) —
  Alatyr does **not** fully type-check generic bodies against constraints in the
  abstract (no trait system, §4.1).
- Because the grammar is fixed and condition evaluation is deterministic (§2.2), all
  conforming implementations partition code into Phase-B-checked and excluded
  **identically** — there is no implementation freedom here.

---

### 10. Syntax (grammar fragments)

The forms this chapter introduces are given here as **grammar fragments**; the
**consolidated EBNF** and lexical rules are assembled in the Grammar chapter, and
these fragments conform to the unified declaration model (Declarations chapter).
Nonterminals not defined here (`block`, `expr`, `binding`, `ident`, `param`,
`type-expr`, `fn-expr`, `arg-list`) come from those chapters.

```ebnf
(* phase marker *)
comptime-binding ::= "comptime" binding              (* comptime PI : f64 = 3.14 *)
type-param       ::= ident ":" "type"                (* T : type   — comptime by nature *)
comptime-param   ::= "comptime" param                (* comptime N : usize *)

(* comptime blocks and control, written in a runtime context *)
comptime-block   ::= "comptime" block
comptime-if      ::= "comptime" "if" expr block [ "else" ( comptime-if | block ) ]
comptime-for     ::= "comptime" "for" ident "in" expr block
(* inside a comptime context, plain if / for are already comptime — no marker *)

(* declaration guard: gates ANY declaration (CT-5), not only a function. For a function
   value it sits in the fn-sig; for a typed binding between `: T` and `=`; for an inferred
   binding at the end. The full binding grammar is Declarations §6 / Grammar §3.2. *)
when-clause      ::= "when" expr
(* e.g.:  add := fn(T : type, a : T, b : T) when Addable(T) { … }   (constraint)         *)
(*        memcpy_fast := fn(…) when target.arch == Arch.x86_64 { … } (target gate)       *)
(*        MAX_CPUS : usize when target.arch == Arch.x86_64 = 256     (gated constant)    *)
(*        Simd := fn(T : type) -> type when has_vec(target) { … }    (gated type alias)  *)

(* generic instantiation — ordinary call syntax, NOT angle brackets *)
type-instance    ::= type-expr "(" arg-list ")"      (* Vec(u8) → a type *)
generic-call     ::= fn-expr  "(" arg-list ")"       (* max(u64, a, b) or, inferred, max(a, b) *)

(* comptime builtins — ordinary call syntax (free functions, OP-1/§5–§6) *)
typeinfo-query   ::= "typeinfo" "(" type-expr ")"           (* → a TypeInfo value *)
resolves-query   ::= "resolves" "(" fn-expr "," arg-list ")"(* → a comptime bool *)
compiles-query   ::= "compiles" "(" expr ")"                (* expr is typechecked, NOT evaluated → bool *)
```

Normative notes:

- `comptime if` / `comptime for` require a **comptime-known** controlling expression
  (§9.1); `comptime for`'s `expr` MUST be a comptime **finite** collection (§8.6).
- `compiles(expr)` and `resolves(...)` **do not evaluate** their operand; they
  perform a comptime typecheck *attempt* and yield a comptime `bool` (§6.1) — they
  are the only constructs that observe whether resolution succeeds **without** making
  failure an error.
- Generic instantiation uses **call syntax**, not angle brackets: a type-function
  applied to arguments yields a `type` (§3.1); a generic-function call infers
  comptime type-parameters where determinable (§3.3).

---

### 11. Conformance (normative summary)

A conforming implementation MUST:

1. treat runtime as the default phase and `comptime` as the explicit marker; treat
   `type` parameters as comptime without a marker; not auto-infer phase (§1);
2. fully erase all comptime computation before run time, leaving no runtime cost or
   artifact, and materialize a boundary-crossing comptime value into **static**
   storage, never the heap (§1.5–1.6; I7/I3);
3. evaluate comptime as the same language with arbitrary recursion/loops bounded by
   a **reproducible step budget** that is a ceiling (deterministic result or
   "budget exhausted", never a silent partial), counted in abstract steps (§2.2);
4. restrict comptime inputs to the machine model, manifest, own source, and
   `embed`, and reject arbitrary I/O; keep comptime functions pure with only local
   mutation (§2.4–2.5);
5. implement generic types as memoized comptime type-functions and generic
   functions by monomorphization, with no runtime dispatch and no surviving type
   parameters; key a `require(U, pred)` type by its immediate `U`, callable identity,
   and comptime capture values (§3; Type System §8.1);
6. treat constraints as ordinary `fn(type) bool` predicates with no separate trait
   machinery, localize an explicit-constraint violation to the use site, and fail a
   structural mismatch at instantiation with a diagnostic (§4);
7. provide `typeinfo(T)` as a comptime-erased reification with no runtime RTTI and
   no hidden type table, and **comptime field projection** `a.(f)` resolving to the
   named field access `a.<f.name>` (same place/type/permission, identical emitted
   code; an `f` that is not a field of `a` is a compile error), and the **comptime
   variant pattern** `T.(v)` resolving to the variant pattern `T.<v.name>` (with
   `comptime for` unrolling match arms over `typeinfo(T).variants`; a `v` that is not
   a variant of `T` is a compile error) — the struct and enum halves of a structural
   derive (§5);
8. provide capability queries (`resolves` / `compiles`) as comptime builtins, and
   resolve language constructs that require a type capability through **protocol
   shapes**, not prelude type names (§6);
9. implement `when` as a declaration guard unifying constraints and target gating,
   and `comptime if` / `comptime for` as branch-selection and unrolling that emit
   only the selected / unrolled code (§7–§8);
10. apply the **two-phase** checking of §9 — Phase A (parse + condition evaluation)
    to all code; Phase B (name/type/constraint checking, lowering, emission) only to
    selected branches and satisfied `when` declarations — so that excluded code may
    reference target-/type-absent entities and all implementations partition code
    identically (§9);
11. accept the syntactic forms of §10 (with the consolidated EBNF in the Grammar
    chapter), including comptime-known controlling expressions and the
    non-evaluating `compiles` / `resolves` operands;
12. introduce **no** hidden control flow, allocation, or runtime type information in
    any of the above (I3).
