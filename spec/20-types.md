# Alatyr Language Specification

## Chapter — Type System

> **Status — draft (under review).** This chapter is a draft and is not yet
> accepted; the specification as a whole is under development. See the status note
> at the top of the Overview chapter.

This chapter defines what types **are**, the machine-level **kernel** they reduce
to, the **machinery** by which all other types are built, how types are
**identified** and **converted**, and how **values** of them are constructed. It
builds on the Memory chapter (values, places, layout) and is the foundation
for the Comptime-and-Generics, Functions, and Codegen chapters.

Requirement keywords (**MUST**, **MUST NOT**, **SHOULD**, **MAY**) follow RFC 2119
(Overview §6). This chapter gives the **concrete syntactic forms** it introduces as
grammar fragments (§11); the *consolidated* EBNF and lexical rules are assembled in
the Grammar chapter, and the forms here conform to the unified declaration model of
the Declarations chapter. The *execution model* of compile-time type computation —
the comptime/runtime boundary, generics, constraints, introspection — is normative
in the Comptime-and-Generics chapter; this chapter establishes the one principle it
rests on (§1) and otherwise treats type construction declaratively. Per-target
layout numbers (native widths, pointer width, endianness) come from the machine
model fixed by configuration (Overview §3); this chapter is normative for the *rules*
that consume them.

---

### 1. Types are values

A **type** is a value. Specifically, a type is a **layout recipe over the machine
model**: it determines how a value is laid out in data blocks of the target's
native widths (I6), and what operations apply to it. The kind of a type value is
`type`.

Because a type is an ordinary comptime value:

- a **generic** is a compile-time function whose result is a type
  (e.g. a function from a `type` parameter to a `struct`);
- a **constraint** ("trait") is a compile-time **predicate** over types — there is
  no separate trait machinery;
- type-level computation runs at compile time and is fully erased before run time
  (I7).

This chapter uses that principle to present type construction (§5) declaratively.
The *execution* of compile-time computation — when it runs, how the boundary is
marked, how it is erased, generics and constraints in full — is the
Comptime-and-Generics chapter. **Every type ultimately decomposes into data blocks
of the target's native widths** (I6); the bottom of that decomposition is the
kernel (§2).

---

### 2. The kernel — raw bit-blocks `bitsN`

#### 2.1 Definition

The **kernel** is the set of types the compiler lowers **directly** to machine data
blocks and machine operations, with no library definition beneath them. The single
kernel primitive is the **raw bit-block** `bitsN`.

`bitsN` exists **only at the target's native widths** — `bits8`, `bits16`,
`bits32`, `bits64` on a 64-bit target; the available set is fixed by the machine
model (I6). A width that is not native (e.g. `bits3`, `bits128` on a 64-bit
target) is **not** a kernel type; it is derived (§7).

#### 2.2 Operations on `bitsN`

A raw bit-block carries **only interpretation-independent** operations — those that
do not presuppose the bits mean a number:

- bitwise `&` `|` `^` `~`;
- shift `shl` / `shr` and rotation `rotl` / `rotr` — **named operation-functions** in call/UFCS form,
  **not** glyph operators (there is no `<<`/`>>`; OP-2, OP-6). On `bitsN` the shift is **logical**
  (unsigned — vacated bits ← 0); the count is a `usize` and an over-width shift (`n ≥ N`) is a checked-guard
  trap (Concurrency §6, I11 — a checked operation, dropped to hardware behavior inside `unchecked`), while
  rotation is total (count taken `mod N`, no trap);
- bit-equality;
- load, store, move (Memory chapter);
- `bitcast` (§4.4) — reinterpret the block as another type of **equal bit-width**.

`bitsN` has **no arithmetic** (no `+`, `<`, …): arithmetic presupposes a numeric
interpretation, which is supplied by `u`/`i`/`f` (§3).

#### 2.3 Run-time storage versus comptime numbers

The `bitsN` ontology describes **run-time storage**. In compile-time evaluation,
numbers are mathematical numbers, not fixed-width blocks (Comptime chapter); a
comptime number is **materialized** into a fixed-width block when it crosses the
boundary into run time. `bitsN` is therefore the substrate of run-time data, not a
constraint on compile-time arithmetic.

#### 2.4 Mapping to the surface

`bitsN` is the vocabulary of the assembly-correspondence surface (Assembly
chapter). Numeric convenience (`u`/`i`/`f`) sits in the prelude and is available on
**all** surfaces, including under `no_abstractions` — *derivedness is about how a
type is defined, not about who may use it* (§3.5).

---

### 3. Numeric interpretations — `u`, `i`, `f`

#### 3.1 Interpretations of a block

`uN`, `iN`, and `fN` are **numeric interpretations** of a raw block, defined in the
prelude over the kernel via the construction machinery (§5). Each carries its own
operations, whose operators lower to the target's instruction intrinsics
(e.g. `u64 +` → the target add). They are **zero-cost**: a `uN` value occupies
exactly its block; the interpretation adds no representation.

**How an operator lowers to its intrinsic (the prelude mechanism).** A native
operator is an ordinary **library function** (§4.5) whose body uses the
**instruction intrinsic as a statement** (Assembly §80): it materializes a result
local, applies the **destination-first** instruction to it (which *mutates* it —
the instruction does not return a value), and returns the local — e.g.
`@inline + := fn(in a : u64, in b : u64) -> u64 { mut out : u64 = a; add(out, b); return
out }`, the instruction selected per target by a `when target.arch` body. The
operator is marked **`@inline`** (Codegen §3.5 / CG-10), so the compiler
**substitutes** its body — that single instruction — at every call site, regardless of
operand shape (I2 — zero-cost; the operator function then carries no `call`).
Inlinability is **declared by the marker**, not inferred from the body's shape. The kernel keeps
**no** built-in numeric knowledge — `u64`'s `+` is library code over the `add`
intrinsic, exactly as `u128`'s `+` is library code over a multiword routine.

#### 3.2 Signedness lives in operations

Signedness is a property of the **operations**, not of storage (the precedent is
LLVM's `iN`). `u32` and `i32` share the block `bits32`; they differ in which
add/compare/shift intrinsics their operators select. Concretely for the right shift
`shr` (OP-6): on a `uN`/`bitsN` operand it selects the **logical** shift (vacated high
bits ← 0); on an `iN` operand it selects the **arithmetic** shift (vacated high bits ←
the sign bit) — so `shr` needs no separate signed name, the operand's interpretation
picks the intrinsic exactly as it does for `+` and `<`.

#### 3.3 Kernel floats versus soft float

`f32` and `f64` are interpretations of `bits32`/`bits64` whose operators lower to
**hardware floating-point** instructions where the target provides them. Where the
target lacks hardware support, the corresponding float is a **derived soft-float**
type (§7) with the same name and visible cost. Floating-point determinism is
specified in the Concurrency-Overflow-Float chapter (§7).

#### 3.4 Relation to `bitcast`

`bitcast` between an interpretation and its block — or between two equal-width
interpretations — is the **identity on the block**: the same bits, a different
interpretation, zero instructions by construction (§4.4).

#### 3.5 Availability

The numeric interpretations are available on every surface and under every limit.
A type's being *derived* (defined in the prelude rather than in the compiler)
constrains neither where it may be used nor its cost.

---

### 4. Type identity and conversions

#### 4.1 Identity

Type identity is **nominal, by declaration**. Each **named** type declaration is a
**distinct** type, even if its layout is identical to another's (`Meters` and
`Seconds`, both `u64`-shaped, are distinct and do not interconvert implicitly).

Identity is **structural only for anonymous layouts** — tuples and unnamed blocks:
two tuples with the same ordered element types are the same type.

A type produced by a **type-function** (`F(T) -> type`, Comptime §1.3) is **nominal**,
identified by the function together with its argument values: `F(A)` and `F(B)` are
**distinct** types even when their layouts coincide. This is what lets a parameterized
type whose layout does **not** mention its parameter still carry a distinct identity per
instantiation — e.g. `Handle(Node)` and `Handle(Edge)` (both a single `usize` index,
Memory §5.4) are different types. The structural rule above applies only to a *directly
written* anonymous tuple/block, never to a type-function's result.

#### 4.2 The conversion lattice

Every conversion between types belongs to exactly one of the following classes. The
class fixes whether the conversion may be implicit and what it costs:

| Class | Meaning | Bits | Cost | Notation |
|-------|---------|------|------|----------|
| **widen** | lossless size increase within one numeric domain (`u8`→`u64`) | extended | sign/zero-extend instruction | `T(v)`; **implicit** outside `no_abstractions` |
| **narrow** | size decrease, possibly lossy (`u64`→`u8`) | truncated | checked by default (traps if out of range); inside an `unchecked` scope narrowing truncates (CG-7) | `T(v)`; always explicit |
| **numeric** | domain change (`int`↔`float`, signed↔unsigned value) | changed | conversion instruction (`float`→`int`: toward-zero, range/NaN checked — Concurrency §7.7) | `T(v)`; always explicit |
| **brand** | add/remove a brand over **the type it directly brands**, same numeric domain (`Meters`↔`u64`) | unchanged | zero | `T(v)`; always explicit |
| **reinterpret** | same-size bit reinterpretation | unchanged | zero | `bitcast`; always explicit |

#### 4.3 Implicit versus explicit is governed by limits

The boundary between implicit and explicit conversion is **tied to the unit's
limits** (I8, Overview §3):

- under `no_abstractions`: **nothing** converts implicitly;
- otherwise: only **widen** (lossless, meaning-preserving) is implicit; **narrow**,
  **numeric**, **brand**, and **reinterpret** are **always explicit**.

#### 4.4 The two explicit forms — `T(value)` and `bitcast`

There are two explicit conversion forms, and they are **distinct operations**, not
two spellings of one:

- **`T(value)`** — a **value** conversion. It *preserves the value* (widen, narrow,
  numeric) or *re-brands* it (brand), and may emit an instruction. It is the
  uniform constructor form for any type — kernel interpretation, prelude, or
  user-defined — and the form a user **conversion-constructor** plugs into (§4.6).
- **`bitcast`** — a **bit** reinterpretation. It *preserves the bits*, requires
  **equal bit-width**, and emits ~zero instructions. `bitcast(T, x)` ≡
  `x.bitcast(T)` (UFCS; §4.5). `bitcast` does **not** check any validity invariant
  of `T`; producing an out-of-range bit pattern this way is explicit
  reinterpretation (I8), and any later use has hardware-defined meaning, never
  undefined behavior the compiler exploits (I11).

`bitcast` is the only conversion in the **reinterpret** class; all others go through
`T(value)`.

For a validity-decorated type `R = @require(pred) U`, a single-argument `R(value)`
whose source is assignable to the immediate `U` is the distinct
**contract-construction** operation of §8.1, not a sixth conversion-lattice class: the
explicit gate runs after that assignment compatibility is established. If the source is
not assignable to `U`, only a **direct** `@convert` to `R` may match under §4.6; the
implementation never searches for a conversion to `U` and then inserts the gate. The
gate is never an implicit conversion.

#### 4.5 Operators and conversions are overloadable builtins

Conversions, comparisons, and arithmetic are **builtin functions in the prelude**,
not keywords (Overview §3; operator detail in the Comptime/operators material).
Through UFCS, `f(a, b)` ≡ `a.f(b)`. Operators are sugar over these functions
(§5.3), so they are overloadable per type and do not enlarge the grammar. Reserved
words are kept only for what a function cannot express (control flow, declarations).

**Indexing `a[i]` is one of these overloadable operators** (OP-3). The postfix
forms desugar to ordinary prelude functions resolved by UFCS over the base's type:

- a scalar index `a[i]` ≡ `at(a, i)` — the **bounds-checked element access**
  (out of range traps, I11; the check is library code, mode-dependent like any
  checked operator above);
- a write `a[i] = v` ≡ `index_set(a, i, v)` (in an assignment **target** position);
- a range index `a[lo..hi]` ≡ `index_range(a, lo, hi)` — the **sub-slice** view.

Arrays and slices supply these as the **compiler-known** built-ins (the bounds
check / pointer arithmetic emitted directly; §3.5 / §6.4 / Memory §4.3) — there is
no source-level prelude function to override for them. A **library aggregate**
(`Vec(T)`, Stdlib §6) supplies its own `at` / `index_set` / `index_range`, so
`v[i]` on a `Vec` is exactly `v.at(i)` (receiver auto-ref applies, so `at`
borrows `v`). The read arm reuses the container's **established named read `at`** —
no separate `index` shim with the identical body (OP-1). This keeps a single indexing surface across built-in and library
sequences (the `+`/`==` treatment, applied to `[]`), without enlarging the grammar
(the `[]` postfix already exists; Grammar §3.4).

A **checked** operator or conversion (one that traps by default and whose check is
dropped inside `unchecked`, §4.2 / CG-8) expresses that check in **library code**, and
makes it **mode-dependent** by reading the verification mode as a comptime fact —
`comptime if verify.checked { <guard> }` (Comptime §7.5 / CT-11). Such a function is
**mode-polymorphic**: instantiated per the call site's verification mode, so the guard
is present in a checked use and comptime-absent in an `unchecked` one. This is how a
**library-defined** native operator (`u64 +`'s overflow guard) or checked conversion
(`char(n)`'s code-point guard, §4.2) satisfies both "trap by default" and "wrap/raw
under `unchecked`" **without** the compiler recognizing the guard — the kernel keeps no
built-in numeric knowledge (§5.3 / TYP-2/TYP-5). `unchecked` itself stays non-transitive for
any function that does **not** read the mode (CG-7).

**Receiver/access coercion — auto-ref and auto-deref (MOD-3 / OP-5).** A value-access
position (a UFCS method receiver, a field `a.x`, an index `a[i]`) coerces the operand by one
step between a value and a pointer to it, by the operand's type versus the type the position
wants:

- operand `R`, position wants `R` — **as-is**;
- operand `R`, position (a method's first value parameter) wants the **scoped reference**
  `ptr([mut] R)` — **auto-ref**: pass the receiver **by its address**, `a.f(b)` ≡
  `f(ptr(a), b)` (a `ptr(mut R)` slot requires `a` to be a mutable place; a
  read-only `ptr(R)` slot does not);
- operand `ptr([mut] R)`, position wants `ptr([mut] R)` — **as-is** (the pointer
  passes through);
- operand `ptr([mut] R)`, position wants the **pointee** `R` — **auto-deref**: reach the
  pointee, `p.x` ≡ `deref(p).x`, `p[i]` ≡ `deref(p)[i]`, `p.m(b)` ≡ `m(deref(p), b)`.

The coercion is resolved **exact match → auto-ref → auto-deref**, and applies **exactly one
level**: a pointer-to-a-pointer auto-derefs once, and reaching further needs an explicit
`deref`, so dereference *depth* is never hidden (I3). For a method receiver these are the
candidate forms `{ a, ptr(a), deref(a) }`; a method matching the pointer **as-is**
(its slot is `ptr(R)`) wins over one reached by auto-deref, so an explicit
pointer-receiver method (`p.free()` over `free(in p : ptr(mut T))`) is never silently
bypassed. For a **plain field/index** the pointer carries no members of its own, so the
coercion is the **unconditional** auto-deref to the pointee (a struct field §6.1, a `str`
field §3.6, the index operator §6.4 / OP-3 resolving on the pointee); a write `p.x = v` then
requires `ptr(mut R)` exactly as `deref(p).x = v` does.

Auto-ref is the sugar by which a method **borrows** its receiver (Memory §5.3) instead of
taking it by value — for an **owning** receiver (Memory §5.9), the inspect/read path that does
**not** consume it; auto-deref is its dual, the **read through a pointer** that likewise borrows
and never moves the pointee. The supplied address is an ordinary second-class scoped reference
(Memory §5.3.1); a by-value first parameter is unaffected (MOD-3/OP-5).

#### 4.6 User-declared conversion-constructors — `@convert`

The conversion lattice (§4.2) is the **builtin** numeric/brand set. A library may
declare a conversion for any other source→target pair as a **conversion-constructor**:
a function marked **`@convert`** whose **single** parameter is the source value and
whose **return type** is the target.

```
@convert pub to_u64 := fn(in v : bool) -> u64 { return if v { 1 } else { 0 } }

x := u64(b)          # b : bool — dispatches to the in-scope @convert whose
                     # parameter accepts bool and whose return type is u64
```

**Dispatch.** A `T(v)` with a **single** argument resolves, in order, to: (1) when `T`
is a required type `R = @require(pred) U` and `v` is assignable to the immediate `U`,
the built-in contract gate (§8.1); otherwise (2) a builtin lattice conversion (§4.2)
when one applies in `T`'s numeric domain; otherwise (3) the **one** in-scope `@convert`
whose return type is `T` and whose parameter accepts `typeof(v)` (the static type of
`v`; Stdlib appendix §4.1). Thus an in-scope direct `@convert` from `S` to a required
`R` remains usable when `S` is not assignable to `U`, but it MUST construct and return
`R` in its own body under the ordinary rules; dispatch does not compose `S`→`U` with
the gate. If `v` is assignable to `U`, the gate takes precedence over a competing
`@convert U`→`R`. The multi-argument and named forms `T(a, b)` / `T(field = …)` are
**aggregate construction** (§5), not conversion. A `@convert` is keyed by its
(**source parameter type**, **return type**), both fully resolved — so the return type
may be a **qualified** type (`mod::Sub::T`), and the **attribute** is the marker, not
the function's name; two conversions to the same target therefore coexist under
distinct names. When `T` is a one-field aggregate and *both* field construction and an
in-scope `@convert` accept `v`, the `T(v)` is an **ambiguity error** — disambiguate
with the named form `T(field = v)` or a named function.

**Scope, not coherence.** A `@convert` is an ordinary `pub` binding, found by the
**name-resolution rules in effect at the call site** (Comptime §6.1): it participates
in `T(v)` exactly where it is in scope (imported). There is **no global registry and no
orphan rule** — to extend a foreign type's conversions, declare the `@convert` in your
own module and import it. Two in-scope `@convert`s matching the same (source, target)
at a site are an **ambiguity error** (no guessing, FND-3).

**Always explicit, never the lattice.** A `@convert` fires **only** through an explicit
`T(v)`; it is never an implicit conversion (only **widen** is implicit, §4.3), and it
cannot override a builtin lattice class within that class's domain. Its cost is the
cost of its body (I2); **selection** is comptime and monomorphized (zero dispatch
overhead). A `@convert` is an ordinary function — its body runs at the stage its
argument is available (a comptime-known argument folds; a runtime argument runs at
runtime). The `?` operator's cross-error widening remains its own **`OutErr`** declared
conversion (Control Flow §8.2), distinct from `@convert`.

---

### 5. Type-construction machinery

#### 5.1 The finite set of layout primitives

All types are built from a **fixed, finite, complete** set of layout primitives:

1. **scalar leaf** — `bitsN` (§2), the only irreducible block;
2. **product** — `struct` / tuple (fields side by side);
3. **sum** — tagged `enum` (a tag plus the active variant's payload);
4. **array** — `[T; N]` (N copies of `T`);
5. **raw union** — overlapping members, untagged.

This set is fixed by the language. It is **complete for machine memory**: every
representable layout is a composition of these. There are **no open, user-defined
layout primitives** — all further expressiveness comes from **composition** plus
compile-time computation (§1). A conforming implementation MUST NOT provide a
layout that is not expressible as such a composition.

#### 5.2 Surface syntax desugars to constructors

The surface forms `struct{…}`, `enum{…}`, `union{…}`, `[T; N]` desugar to these
primitive constructors, and the constructors are **callable at compile time**. A
type-generating function is an ordinary comptime function that returns one
(conceptually, `Vec` is a function from a `type` to a `struct`). The exact syntax
of these forms is in the Declarations and Grammar chapters; their *layout* is §6.

#### 5.3 Operators are sugar over overloadable functions

An operator on a constructed type is sugar over a builtin function (§4.5): `u64`'s
`+` selects the target add intrinsic; a multiword `u128`'s `+` calls its own
library routine. Literals and operators are **uniform** for kernel and derived
types alike — the same `+`, the same literal syntax, regardless of whether the type
is in the compiler or the prelude. The same uniformity covers **indexing** (§4.5 /
OP-3): `a[i]` is `at(a, i)` whether `a` is a built-in array (compiler-emitted) or
a library `Vec` (its own `at`).

#### 5.4 Branding

**Branding** is the primitive that grants a **distinct nominal identity over a
shared layout** (§4.1). A brand is zero-cost: it changes identity and the
applicable operation set, never representation. The numeric interpretations of §3
(`u`/`i`/`f`) are *implemented* by branding — each is a brand over its block
carrying a numeric operation set — so `u64` and `i64` are sibling brands over
`bits64`, and `Meters` is a brand over `u64`.

Branding alone does **not** determine the conversion class (§4.2); the numeric
domain does:

- crossing between a brand and the type it **directly brands**, within the **same
  numeric domain**, is the zero-cost **brand** conversion (`Meters`↔`u64`);
- crossing between two **different numeric interpretations** — different signedness
  (`u64`↔`i64`) or different domain (`int`↔`float`) — is a **numeric** conversion
  (`T(v)`, value-checked) or a `bitcast` (bit-preserving **reinterpret**), **not** a
  brand conversion, even though both sides happen to brand the same block.

Sibling interpretations are not each other's underlying type; only the block they
share is. Every brand conversion is explicit (§4.3).

---

### 6. Aggregate layout

Layout defaults come from the machine model; per-site overrides are §8. All layout
is **predictable from the declaration** (I1): the compiler MUST NOT reorder or pad
beyond the documented rule.

#### 6.1 Product — `struct` and tuple

- Fields are laid out **in declaration order**; the compiler MUST NOT auto-reorder.
- **Padding** is standard: each field is placed at the next offset that is a
  multiple of its alignment; the struct's alignment is the **maximum** of its
  fields' alignments; the struct's size is rounded **up** to its alignment.
- A **tuple** is an anonymous positional product, accessed `.0`, `.1`, …, with
  **structural** identity (§4.1). The `.N` form is a **positional field projection**:
  `N` is a **compile-time** index (each position may have a different type, so the
  result type is fixed statically). It is **distinct** from array indexing `a[i]`
  (§6.4) — a homogeneous sequence with a possibly-**dynamic**, bounds-checked index —
  and applies **only** to tuples: arrays use `[i]` (OP-1: `a.0` would merely duplicate
  `a[0]`), and an `enum`'s payload is reached by **`match`**/destructuring, never a
  positional `e.N` (which would be unsound when another variant is active, I11).
- A field accessed **through a pointer** auto-derefs one level: when the base has type
  `ptr([mut] S)` for a struct `S`, `p.field` ≡ `deref(p).field` (§4.5 / OP-5) — the
  pointer has no fields of its own, so the dereference is unconditional. A write `p.field = v`
  requires `ptr(mut S)`.

#### 6.2 Sum — tagged `enum`

A tagged enum is a **tag** followed by a **payload**:

- the **tag** is the smallest `uN` that distinguishes the variant count; it sits
  **first**, at offset `0`, with the payload following at the payload's alignment. The
  tag's **position is fixed** in v1 — the closed lever set (§8) contains nothing that moves
  it, and `@offset`/`@packed` are struct-specific — while its **type** MAY be pinned with
  the **`@repr(T)` lever** (§8), e.g. to a fixed C-ABI width;
- each variant's **tag value** (discriminant) is assigned in **declaration order
  starting at `0`** (first variant `0`, each subsequent `+1`); a variant MAY pin an
  explicit value with `= N` (a comptime integer), after which following unassigned
  variants continue from `N + 1`. Two variants resolving to the same value are
  ill-formed;
- the **payload** is sized for the largest variant and aligned accordingly.

An enum with **zero variants** is **ill-formed**: it is uninhabited (no tag value, so no
value can ever be constructed), and the language's single uninhabited type is the named
bottom type **`Never`** (§3.7 / Stdlib §3.7) — a second one by an empty `enum {}` would be
a redundant entity (OP-1). This also closes a latent edge: a `comptime for` over
`typeinfo(T).variants` of a zero-variant `T` would unroll to **no** `match` arms, leaving a
non-exhaustive `match` (Control Flow §5.4); since such a `T` cannot exist, the situation
does not arise.

A **niche** uses an unused bit pattern of the payload to encode a variant, so the tag
costs no extra space. A type's niche is its **`niche` lever** (CT-10) — a *constructive*,
comptime set of its invalid bit patterns — never chosen by an opaque optimizer
(predictability, I1). The core and prelude supply niches uniformly through it:

- a **non-null pointer** (`ptr(T)` / `ptr(mut T)`) — the all-zero (null) pattern
  (intrinsic to non-null, MEM-8);
- **`bool`** — any byte value other than `0` or `1` (the prelude declares it via the
  `niche` lever);
- a **tagged enum whose tag has unused values** (variant count < `2^(tag width)`) — the
  least unused tag value (structural).

A **user-defined** type provides a niche the **same way** — the `niche` lever — so it is
neither privileged nor deferred. A single-extra-variant tagged enum — such as `Option(T)`,
which adds one `None` — folds that variant into the payload's niche **iff `T` provides
one**; otherwise it is an ordinary tag + payload. This is **one general layout rule**, not
special knowledge of `Option` (an unprivileged library type). So `Option(ptr(T))` is
exactly pointer-width (Memory §4.2) and `Option(bool)` is one byte — each **readable
from `T`'s declared niche**, with no compiler run.

**The `@niche(producer)` shape (the lever's contract).** The argument to `@niche(producer)`
MUST resolve at **comptime** to a deterministic function
`fn(comptime i : usize) -> Option([bits8; T.size()])`, where `T` is the decorated type:
each `Some(bytes)` is **one invalid storage-order bit pattern** of `T` (a value `T` can
**never** hold), and `None` **terminates** the sequence. The compiler enumerates it from
`i = 0` upward and uses the patterns, in that order, when folding extra tag values into a
payload niche (§6.2). It is a **compile-time diagnostic** if a returned pattern is not
invalid for `T`, if a pattern repeats, or if the function is non-terminating within the
comptime budget (Comptime §2.2). The built-in niches above are exactly this lever applied
by the core/prelude (a non-null pointer's producer yields the all-zero pattern, then
`None`; `bool`'s yields `2..=255`); a user type's `@niche` is checked and consumed
identically — neither privileged nor deferred.

#### 6.3 Raw union

A raw union is an **untagged overlap of named members**. It carries **no tag, hidden
discriminant, active-member byte, or run-time metadata** (I2/I3). The source and the
compile-time analysis may know which member was activated; that fact is not part of
the value's storage or ABI.

**Member payloads and arity (TYP-11).** For a member declaration, the component list
determines exactly one payload type:

- `none` has zero components and payload type `()` (the zero-sized unit tuple);
- `word(u32)` has one component and payload type `u32` — there is no one-element
  tuple wrapper;
- `pair(u16, u32)` has one **anonymous tuple payload** `(u16, u32)`. Its components
  use the ordinary tuple declaration order, offsets, alignment, internal padding,
  structural identity, and `.0` / `.1` projections of §6.1.

Thus every member has one payload value even when its declaration has several
components. `typeinfo(U)` reports the same rule through `Variant.payload`: `None`
for zero components, `Some(T)` for one, and `Some((T0, T1, ...))` for several
(Comptime §5.1; Stdlib appendix §4.1). Parentheses with no component are not a
second declaration form: a zero-component union member is written `none`, not
`none()`.

**Layout.** Let `P(m)` be member `m`'s payload type. Every `P(m)` begins at offset
`0`. The union's natural alignment is `max(P(m).align())`, with a minimum of `1`;
its natural size is `round_up(max(P(m).size()), union_alignment)`. The bytes from a
payload's size through the union's size are that member's **tail padding**. A
multi-component member additionally has exactly the tuple's ordinary internal and
tail padding. An `@align(N)` on the union raises its alignment and rounds its size
up again; it does not move a member away from offset `0`. A union whose members all
have zero-sized payloads has size `0` and alignment `1` (or the explicitly raised
alignment). A union with **no members** is ill-formed, like a zero-variant enum:
`Never` is the language's sole uninhabited type.

**Construction and activation.** A type-qualified member constructor constructs a
whole union value and makes that member active:

```alatyr
Cell := union { none, word(u32), pair(u16, u32) }

a : Cell = Cell.none
b : Cell = Cell.word(7)
c : Cell = Cell.pair(1, 2)       # active payload has type (u16, u32)
```

The constructor arity is the declaration's component count: `Cell.none`,
`Cell.word(x)`, and `Cell.pair(x, y)`. Arguments are positional and evaluate
left-to-right before any destination byte is written; named arguments are ill-formed.
`Cell.none()` is ill-formed;
`Cell.pair((x, y))` is one argument and is therefore ill-formed unless `pair` was
declared with one component whose type is itself that tuple. Construction is
**type-qualified**. `u.m(args)` remains the language-wide UFCS call `m(u, args)`
(Grammar §3.4); it is not a union write.

Constructing `U.m(args)` first writes zero to all `U.size()` backing bytes, then
copies exactly `Ti.size()` bytes of each already-evaluated component `i` (including
that component value's own padding), in declaration order, into its ordinary payload offset.
The component copies do not overwrite padding introduced **between/after components** by
the member's tuple layout or by the union tail; those bytes therefore remain zero. This
clear-plus-component-copies sequence is the documented cost ceiling (I1) and fixes
the complete object representation for the supplied component representations. For a
zero-component member the clear is the entire operation. This is a union-constructor
rule, not implicit initialization of an uninitialized binding (§9.4). A conforming
optimization may remove stores only when their bytes cannot be observed, including by
a permitted unchecked member read.

For a mutable union place `u`, assigning the **whole payload place** activates the
member and performs the same clear-plus-component-copies rule. The right-hand side is
fully evaluated before the destination is cleared, as for ordinary assignment:

```alatyr
u.word = 9
u.pair = (3, 4)
u.none = ()
```

A write through a nested component projection such as `u.pair.0 = 3` does not by
itself initialize or activate the other component. It is permitted in checked code
only when `pair` is already definitely active; then it preserves that active member.
Inside `unchecked`, such a partial write is permitted but does not by itself establish
definite initialization or a definitely active member.

**Typed projection and the active-member rule.** `u.m` has exactly the payload type
`P(m)` above and follows the ordinary place/value rule: from a union place it is a
place (writable only through a mutable access path), and in value context it loads
that payload. For a multi-component member, `u.pair.0` and `u.pair.1` are therefore
ordinary tuple projections. One-level pointer auto-deref applies exactly as for a
struct field (§4.5 / OP-5).

Because there is no run-time tag, checked access is established by the following fixed
**definitely-active analysis**, not by a hidden check. A **trackable union place** is
rooted at a local binding, parameter, or static binding and is reached only through a
fixed sequence of named struct fields, tuple indices, and comptime-constant array
indices. A place reached through a pointer dereference (including auto-deref), a dynamic
index, or a call result is not trackable. Union value expressions carry the same facts
while evaluated. For each trackable union place or union value expression, the analysis
extends §9.4's definite-initialization fact with an orthogonal active-member fact: one
exact member name or unknown. Every non-trackable union place is active-unknown.

- A constructor expression `U.m(args)` is definitely initialized with exact member
  `m`. Initialization or whole-union assignment to a trackable place transfers the
  source expression/place facts; a source without tracked facts is initialized but
  active-unknown. A checked whole-union read/copy still requires definite initialization.
  A whole-payload assignment `u.m = payload` makes a trackable `u` definitely initialized
  with exact member `m`; through a non-trackable place it writes the bytes but establishes
  no fact usable by a later checked read.
- A merge is definitely initialized only when every non-diverging predecessor is; it
  keeps exact member `m` only when every such predecessor has exact member `m`. Differing
  members or an unknown predecessor make the active member unknown; an uninitialized
  predecessor also prevents definite initialization.
- A parameter, an `@extern` value, and a mutable static begin each function initialized
  as applicable but active-unknown. An immutable static begins with the facts retained by
  its comptime initializer. Taking a mutable pointer to a trackable root or any containing
  subobject, or passing either `in out`, makes every overlapping tracked union place under
  that root active-unknown until a later direct whole activation. A non-mutating `ptr(U)`
  does not invalidate the root fact, but a read through the pointer is a non-trackable
  access and therefore requires `unchecked`. Function result types carry no active-member
  refinement, so a normally returned union is initialized but active-unknown to its caller.
- A checked read of `u.m` (including a nested `u.m.N` read) is well-formed **iff** its
  union base is either a trackable place or a value expression carrying facts, is
  definitely initialized, and has exact member `m`. Using
  `u.m` as the direct target of a whole-payload assignment is activation, not a read, and
  does not require a prior `m` fact. Otherwise the access is a Semantic diagnostic. No
  implementation may accept a checked read using a more permissive proof.
- Inside `unchecked`, any member may be read regardless of the state, including
  inactive or uninitialized storage. The bytes at offset `0` are interpreted as
  `P(m)`; invalid values, alignment faults, and other raw outcomes are
  hardware-defined and never language UB (CG-7/I11). The programmer is responsible
  for choosing a member whose representation is meaningful.

**Whole values, comptime, and statics.** A copyable union is copied and passed by
value as exactly `U.size()` bytes, including internal and tail padding; active-member
facts transfer only as specified above and emit no metadata. A union carrying the
`@owning` root marker or containing an owning payload transitively is unsupported in v1
and MUST be rejected, so a raw union cannot use overlap to duplicate or reinterpret an
owning resource (Memory §5.9). Union constructors are ordinary comptime expressions
when their arguments are comptime-known. A comptime union value retains its active
member in the evaluator;
when materialized into a module-level/static initializer, the implementation emits
the constructor's complete `U.size()` bytes. An immutable static initialized
with `U.m(...)` is therefore definitely `m`; a mutable static is conservatively
unknown on function entry as specified above. There is no run-time static
initializer (Memory §2.2).

```alatyr
comptime SAMPLE : Cell = Cell.pair(10, 20)
sample : Cell = SAMPLE                 # immutable static; pair is definitely active

read_pair := fn() -> u16 { sample.pair.0 }   # checked and well-formed
reinterpret := fn(in x : Cell) -> u32 {
  unchecked x.word                     # parameter activity is unknown; raw read is explicit
}
```

**Invalid forms and unsupported representations.** Member names MUST be unique. A raw
union member has no field names, defaults, `mut`, member attributes, or discriminant;
those forms, an empty `union {}`, and zero-component parentheses `m()` are ill-formed.
A raw union has no discriminants, so `member = N` and `@repr(T)` are ill-formed.
`@packed`, `@offset`, `@endian`, and `@niche` do not have a union-level meaning in v1
and are ill-formed on a raw union; apply the appropriate lever to a component type or a
component type's fields instead.
`@align(N)` is the sole union-level layout lever. A raw union MUST NOT carry the
`@owning` root marker and MUST NOT contain an owning payload transitively. A
tagged/discriminated union, per-member explicit offsets, and an owning raw-union payload
are not alternative raw-union representations: use a `struct` containing an explicit
tag plus a raw union, an ordinary explicit-offset `struct`, or a tagged `enum`,
respectively.

#### 6.4 Arrays

`[T; N]` lays `N` elements **contiguously**; the **stride** is `T`'s size rounded up
to its alignment; element access is `base + i × stride`. The index is
**bounds-checked** by default (the check is in the I2 checked-default family); **inside
an `unchecked` scope** the bounds check is omitted (the verification mode, CG-7/CG-6 —
`unchecked a[i]` for one access, `unchecked { … }` for a region). `N` is a compile-time value. A **dynamic** length is a
**slice** — a library type (pointer + length), **not** a layout primitive.

Indexing **through a pointer** auto-derefs one level: when the base has type
`ptr([mut] R)` for an indexable `R`, `p[i]` ≡ `deref(p)[i]` (§4.5 / OP-5), after which
the index operator resolves on the pointee `R` (this built-in for an array/slice, or the type's
own `index`/`index_range`, §4.5 / OP-3). A raw pointer carries no arithmetic indexing of its own,
so `[i]` on a pointer always means dereference-then-index.

#### 6.5 Zero-sized types

A zero-sized type (an empty `struct`, `[T; 0]`) has **size 0** and **alignment 1**;
it occupies no bytes and emits none. Zero-sized types are permitted.

---

### 7. Derived types (prelude)

Derived types are defined in the prelude with the §5 machinery; they **feel
built-in** but are ordinary library types (the prelude is the first and most
demanding consumer of the machinery — the language eats its own dogfood). Their
cost is **visible**.

- **Fixed-width multiword unsigned integers** — `uint(N)` (TYP-10): a **prelude
  type-function** that, for a comptime `N` that is a **positive multiple of 64**,
  yields an unsigned integer stored as a **library multiword value of exactly `N/64`
  machine words, little-endian** (word 0 = least significant). Its arithmetic and
  comparison are ordinary **library operator-functions** (§4.5) generated from the
  word count — ripple-carry `+` / `-`, schoolbook `*` keeping the low `N` bits,
  binary long-division `/` / `%`, and a hi-word-to-lo-word lexicographic **unsigned**
  comparison — with **visible** cost: a value occupies `N/64` words; the operations
  are `O(words)` (add/sub/compare) or `O(words²)` (multiply/divide). This is the
  concrete library multiword value that makes a non-native width (`u64` on a 32-bit
  target, a `usize` decomposition) an ordinary `{word…}` value, **never** a backend
  register pair. Widths that are **not** a multiple of 64 (an arbitrary `N` with a
  **partial top word**, and **sub-native** widths such as a byte-plus-mask `u3`), and
  the **signed** multiword `int(N)`, are **additive** — deferred, not part of v1
  (TYP-10; their mask / carry / overflow rules are unspecified in v1).
- **Wider-than-native named integers** — `u128` on a 64-bit target is the multiword
  `uint(128)` under a name (**`u128` ≡ `uint(128)`**): one name, a multiword
  implementation where the width exceeds the native register, **visible** cost (TYP-10).
- **Target-dependent** `usize` / `isize`: prelude aliases that compile-time-expand
  to the kernel's pointer-width interpretation for the target.
- **`char`**: a prelude alias for a Unicode code point (`u32`-shaped). A byte string
  (`str`) is **UTF-8 bytes** and its length in bytes is not its length in code
  points; text operations are in the standard library (Lexis/Stdlib chapters).

The base prelude additionally provides the foundational non-numeric types this
chapter already uses; they are ordinary library types built by the §5 machinery,
and their full definitions and operations are in the Stdlib chapter. The
type system fixes only their **nature and layout**:

- **`bool`** — a two-valued type whose canonical representation is a single byte
  (`false` = `0`, `true` = `1`); it is the result type of comparisons (§4.5) and the
  operand of the logical control operators (§10). It is **not** an integer and does
  **not** interconvert with one through the numeric **lattice** — neither implicitly
  nor as a lattice class: it sits **outside the numeric lattice** (§4.2). Any
  bool↔integer conversion is therefore a **library conversion-constructor** (`@convert`,
  §4.6) — always explicit, the library author fixing its meaning — and the language
  privileges none. See Stdlib appendix §3.1.
- **`Option(T)`** and **`Result(T, E)`** — tagged enums (§6.2). `Option(T)` folds `None`
  into `T`'s niche where `T` provides one (the §6.2 `niche` lever; e.g.
  `Option(ptr(T))` is pointer-width, Memory §4.2); otherwise it is a tag plus
  payload — one general rule, `Option` unprivileged.
- **`[T]`** (slice) — a pointer + length pair over a contiguous run of `T` (§6.4);
  the layout is that library pair, not a primitive.
- **`str`** — a slice of **UTF-8 bytes** (`[u8]`); its byte length is not its
  code-point count (§7, `char` bullet).

Both are **views**, and the view *is* the value: a `[T]` or `str` binding, field,
parameter, return value or array element holds the **two-word pair itself** (pointer +
length — 16 bytes on a 64-bit target, aligned as a pointer), and `[T].size()` /
`str.size()` report **that pair**, never the length of the run it points at (which is a
run-time property, not a type property). A view therefore has the same size wherever it
appears; nothing about it is a special case of layout.

A **raw code address** (a code-point label's address, Control Flow §2.1) needs **no
dedicated type**: it is the target's **pointer-width raw block** — `bits64` on a
64-bit target, `bits32` on a 32-bit one (the kernel `bitsN`, §2; `bitsN` is the family
meta-notation, and **source code names the concrete width**). It is a **`jmp` target**
(raw computed goto), **not callable** and **not** a function pointer — it has no
ABI/frame entry contract (a `call` target is a function value, §1; Functions §1.2),
and it is never dereferenced as data. It is **not opaque**: being a `bitsN`, it carries
the ordinary raw-block operations (§2.2) — there is **no** nominal protection, the
honest cost of using no dedicated type. The principled asymmetry with data pointers: a
**data** pointer is the typed `ptr(T)` because a typed deref (`deref`) needs `T`;
**code** has no typed deref, so its address is just raw bits. A jump table is
`[bits64; N]` (pointer-width, target-specific).

Naming is uniform: `u64` is one name whose implementation is native on one target
and multiword on another; the **cost differs and is visible**, the spelling does
not.

---

### 8. Layout control

The layout defaults of §6 may be overridden **per site**. Every override is
**visible** and is **part of the type's contract**. These overrides are the **machine
levers** — the closed v1 set `repr` / `align` / `packed` / `offset` / `endian` /
`niche` (CT-10), each a comptime function with a documented lowering rule.
Every one has **two equivalent surfaces**: a prefix attribute `@align(N) T` and a UFCS
call `T.align(N)`. A **library attribute** is an ordinary comptime function composing
them (e.g. `@nonzero` = `require` + `niche`), **indistinguishable** from a prelude one
(no privilege, OP-1/CT-10).

- **`@repr(T)`** — pin the **underlying representation type of a tagged `enum`'s tag**
  (the discriminant), overriding the §6.2 default ("the smallest `uN` that distinguishes
  the variant count", §6.2 — *its type MAY be overridden*). `T` is an integer type
  (`uN` / `iN` / a `bitsN` brand) that MUST represent every discriminant value (else a
  compile diagnostic). **Lowering:** the tag is emitted as `T` — `T`'s width, alignment,
  and endianness — with each discriminant value encoded in `T`; the payload is unchanged.
  Typical use: a C-ABI enum whose tag must be a fixed width (`@repr(i32) enum {…}`).
  It is **zero-cost** (I2): it fixes the type of the tag that already exists, adding
  nothing. It is **not** branding (which gives a *nominal identity over a base* — a
  newtype, §5.4) and **not** `@abi` (a function-declaration calling-convention attribute):
  `@repr` does exactly one thing — pin the **type** of §6.2's tag (its position is fixed).
  Levers are **not** universal: each targets a specific **representational degree of
  freedom**, applying only where that freedom exists. `@repr` is the **discriminated
  type's** representational lever — the counterpart of the **struct-specific** `@packed`/
  `@offset` (meaningless on an enum tag, just as `@repr` is meaningless on a struct
  field). The tagged-enum tag is its v1 site; future compiler-chosen carriers are an
  additive extension of the same lever (FND-6), not a new one.
- **`@packed`** — remove padding (alignment 1); unaligned access is handled per the
  alignment rule (Memory §4.6; Assembly §3).
- **`@align(N)`** — raise alignment above the natural alignment.
- **`@offset(N)`** — give a field an explicit offset (MMIO, register maps);
  overlapping fields behave union-like (unchecked-flavored).
- **`@endian(big)`** / **`@endian(little)`** — per-field or per-type endianness
  (the `endian` lever, CT-10). The argument is an **`Endian`** value (`Endian := enum
  { big, little }` — the prelude config type, also the manifest's per-target `endian`);
  the prelude binds `big` / `little` to those values, so the canonical short spelling is
  `@endian(big)` (parallel to `@abi`'s prelude-bound `c` / `sysv`). The default is the
  target's (for wire formats).
- **`@section("name")`** is **not** a layout lever — it places a *static binding* in a
  named section (a **storage/placement** attribute on a binding, Memory §2.3/§2.4); it
  does not shape a type's bits.
`@abi` is **not** a type-layout lever in v1. A type's layout is fixed by the default (§6)
plus the closed levers above (`@repr` / `@packed` / `@align` / `@offset` / `@endian` /
`@niche`, CT-10). The default `struct` layout (§6.1 — declaration order, natural alignment,
standard padding, *no auto-reorder*) already coincides with the platform's C `struct`
layout, so there is no type-level "match a named ABI" lever to add: a non-default layout
is constructed explicitly with the levers. `@abi(value)` selects a **calling convention**
on a **function** declaration (Functions §6) — never a type's data layout.

> *Rejected for v1: `@abi(value)` on a **type**.* The `Abi` value schema (ABI appendix §1)
> describes a **calling convention** only — argument/result registers, saved sets, stack
> alignment — with **no** data-layout fields, so `@abi(c) struct {…}` had no meaning an
> implementation could lower **without guessing** (a FND-3 gap). A type-layout ABI is
> **additive** (FND-6) once a data-layout schema is specified; until then the closed levers
> suffice (the default already matches C for ordinary structs).

#### 8.1 Validity contracts — `require`

A type may carry a **validity-construction contract**:
`@require(pred) U` ≡ `U.require(pred)` (the two surfaces of §8). This is the type-function
`require(U, pred)`, producing a type `R` whose **layout, size, alignment, and ABI field
classification are exactly `U`'s**, but whose **type identity is distinct**. `R` is
identified by the immediate underlying type `U`, the identity of `pred`, and every
comptime capture value specializing `pred`; applying the same type-function to the same
arguments yields the same type. A named function reference and an inline lambda with the
same body are accepted equivalently but are **not equated by body comparison**: separately
written lambdas have distinct callable identities.

`pred` MUST be a **comptime-known callable** which, after all generic/comptime arguments
and captures are specialized, has exactly the ordinary interface
`fn(in U) -> bool`. A named function and an inline `fn` obey the same rule. An inline or
static closure MAY capture only comptime-known values; those values become
monomorphization arguments and leave **no runtime environment argument**. A runtime
capture, a `dyn fn` value, `@abi(naked)`, or `@abi(syscall)` is ill-formed here. Ordinary
conventional ABIs, including the default ABI and an explicitly selected conventional
`@abi(value)`, remain ordinary calls. Overload resolution MUST select exactly one callable
with the exact post-specialization interface; no auto-ref/auto-deref or result-directed
choice applies.

The notation `fn(in U) -> bool` above names the required **call interface**. It does
not require every accepted predicate value to have the concrete one-word function-value
type of Functions §1.5: a non-capturing function has that type, while a function literal
with permitted comptime captures is a static closure specialized under Functions §1.2 /
Memory §6 whether it is supplied inline or first bound to a name. Naming is orthogonal to
representation. The two source styles are ordinary and equivalent as predicate sources:

```alatyr
Point := struct { lo : u64, hi : u64 }
ordered := fn(in p : Point) -> bool { p.lo <= p.hi }

NamedRange  := Point.require(ordered)
InlineRange := Point.require(fn(in p : Point) -> bool { p.lo <= p.hi })
```

`NamedRange` and `InlineRange` are distinct because the named declaration and the
function-literal occurrence have distinct callable identities, not because one has a
different call interface. The prefix type surface may be used wherever a type expression
is expected, for example `x : @require(ordered) Point`; the UFCS spelling above makes a
named required type by an ordinary type-valued declaration.

**Construction and compatibility.** `R(v)` is the built-in, explicit **constructor
gate** from the immediate `U` to `R`:

1. `v` is evaluated **exactly once**, including all of its observable effects, into the
   representation that will become the result;
2. in checked runtime code, the preserved value is copied to a **distinct logical
   by-value argument representation** and `pred` is invoked **exactly once** through its
   ordinary ABI;
3. if the returned `bool` is false, execution performs the target's **direct inline
   checked-failure trap** (Assembly §3; per-arch appendix §2) before the result is
   delivered; if true, the preserved representation is delivered as `R`.

For the gate path, the source expression MUST be assignable to the immediate `U` under
§4.3 before the gate is applied. There is **no implicit `U`→`R` conversion**, no
conversion-chain search, and no inherited aggregate spelling such as
`R(field = value)`: for an aggregate, write `R(U(field = value))`. A source that first
needs another explicit conversion to `U` writes it inside the gate (`R(U(x))`); a source
with a direct `@convert` to `R` follows the separate §4.6 dispatch path and does not run
an implicitly composed gate. An existing `R` copies/moves and assigns under the ordinary
rules for `R`; those operations do not reconstruct it or call `pred`. `U` and `R` are
not assignment-compatible in either direction. The existing equal-width `bitcast`
operation is the explicit raw-bit way to cross either direction and never invokes the
gate (§4.4).

**ABI cost for the predicate argument.** The call uses the ordinary by-value lowering of
Functions §4 and the selected ABI (ABI appendix §6):

- a scalar or small aggregate is copied from the preserved result into its classified
  argument register group / stack slots;
- a large aggregate produces exactly one complete byte-for-byte ABI argument copy in a
  caller-owned temporary and passes a pointer to that copy.

The preserved result uses ordinary automatic placement (Memory §2.5 / Codegen §3.3) and
ordinary call-clobber rules. If its chosen registers are caller-saved, the documented S2
placement includes the required save/reload or chooses non-clobbered/stack storage; this
is result preservation, not a second predicate-argument aggregate copy.

Thus the documented unoptimized ceiling is one result representation, one logical
predicate-argument copy, and one ordinary call. Optimization MAY coalesce physical
storage or inline/eliminate an observably redundant call only under Codegen §3's
semantics-preserving, cost-lowering rule; it MUST NOT duplicate or reorder an observable
predicate evaluation, permit the predicate to consume/overwrite the result, or move
delivery before a false-result trap.

**Verification mode and phase.** Inside `unchecked R(v)`, step 1 still occurs exactly
once, but the **entire** predicate path is absent: no predicate-argument copy, call,
branch, or trap is emitted. The once-evaluated bits are delivered as `R`; this is the
explicit contract bypass, like producing `R` with `bitcast` (I11). In a checked comptime
context, including a module-level/global initializer, the evaluator performs the same
ordering and invokes `pred` once. False is a Semantic diagnostic at the construction
site; true materializes the resulting value normally. No runtime initializer, predicate
call, or trap is emitted for a module-level construction. `unchecked` at comptime skips
the predicate in the same way as at runtime.

**Aggregates, initialization, and mutation.** v1 requires `U` to be **non-owning**
(Memory §5.9). An owning `U` is ill-formed because preserving the result while supplying
an ordinary by-value predicate argument would copy or consume one owning value. A required
type gains no ownership merely from `require`; any independent `@owning` declaration is
governed by Memory §5.9. A required
aggregate `R` MUST be initialized as a complete value: it cannot be built field-by-field,
and declared defaults belong to the underlying `U(...)` construction before `R(U(...))`.
Build an uninitialized/partially initialized `U`, complete every field under §9.4, then
pass the whole `U` through `R(u)`. In checked code a component place of `R` is read-only:
a field/element store through `R` is ill-formed because it would bypass the complete-value
gate. A `mut R` place MAY be replaced whole by another `R` without revalidation. Inside
`unchecked`, component mutation otherwise permitted by `U`'s ordinary mutability is the
explicit contract bypass and performs the requested store without a predicate call or
revalidation; `unchecked` does not make an immutable field writable.

**Stacking.** Repeated decorators are nested **nearest to the base first**:
`@require(p) @require(q) T` means `require(require(T, q), p)`. Each predicate takes its
own immediate underlying type, and each explicit constructor gate runs only its own
predicate. Consequently a base `t : T` is constructed as `Outer(Inner(t))`; `Outer(t)`
does not flatten or search through the inner contract. The ordinary expression evaluation
rule gives innermost-to-outermost predicate order, with each operand evaluated once.

`require` is a **comptime-function attribute**, not a layout/machine lever: it changes
identity and construction semantics, never representation. A type's `niche` (§6.2) is the
separate **constructive** invalid-pattern producer used for enum packing; a contract may
exist without a niche and vice versa.

A **function precondition** — a runtime check on argument *values* — is **not** `@require`
(an attribute is a prefix, where the parameters are not in scope) and **not** a `when`
clause (that is the **comptime** guard, on types, Comptime §7); it is plain **`assert` in
the body** (Stdlib §4.3), where the parameters are in scope.

---

### 9. Value construction

#### 9.1 Literals

The literal forms are integer, floating-point, boolean, character, and string. (There
is **no separate byte-string literal**: exact byte data is an ordinary `[u8; N]` array
value — §9.3, Grammar §2.4.) An **integer literal is a compile-time number** (§2.3);
its type is
**inferred from context**, and its representability in that type is **checked at
compile time** — a literal outside the target type's range is a compile error (I11),
never a silent wrap. With **no** context, a literal takes a **documented default**:
the target's native signed integer.

A **floating-point literal is likewise a compile-time number**, its type inferred from
context, and that context type MUST be a **floating-point** type: a float-spelled literal
never initializes an integer type, so `x : u64 = 1.5` **and** `x : u64 = 1.0` are both
compile errors — an integral *value* does not make a float *literal* an integer literal
(TYP-13); write `1`. Representability is checked at compile time here too: rounding a
decimal literal to the nearest value of the binary format is **representation**, not loss
(`0.1 : f64` denotes the nearest `f64`), while a literal outside the format's finite range
is a compile error, never a silent infinity. With **no** context, a floating-point literal
takes the documented default `f64`.

An **integer** literal in a floating-point context is accepted exactly when the value is
**exactly** representable in that float type (`x : f64 = 1`); one that is not
(`x : f32 = 16777217`) is a compile error, on the same rule as an out-of-range integer
literal.

#### 9.2 No literal suffixes

There are **no** type suffixes on literals (no `5u8`). A literal's type is refined
by the **constructor form `T(v)`** — uniform for every type, kernel or
user-defined (§4.4) — or by an annotation. Suffixes are rejected because they would
privilege a fixed built-in set against the uniformity principle (§5.3).

#### 9.3 Aggregate constructors

- **struct** — named fields: `Point(x = 1, y = 2)`;
- **tuple** — positional: `(1, 2)`; the zero-field unit tuple value is `()`;
- **enum variant** — `Color.Red`, `Option.Some(v)`;
- **raw-union member** — `Cell.none`, `Cell.word(v)`, `Cell.pair(a, b)`; the
  constructor produces a whole canonical union value and activates that member
  (§6.3);
- **array** — listed `[1, 2, 3]` or replicated `[v; N]`.

A constructor fills bytes by the layout recipe (§6). Constructors run in compile
time as well; a value that crosses into run time is materialized into **static**
storage (`.rodata`/`.data`), never the run-time heap (Memory chapter,
preserving I3).

#### 9.4 Initialization

- There is **no implicit, language-imposed default or zero-initialization**: the
  language never zeroes an uninitialized binding on the programmer's behalf. The
  raw-union constructor/whole-member assignment rule of §6.3 explicitly clears its
  backing bytes as part of that operation's canonical representation and visible
  lowering; it is not a default initializer.
- **Programmer-declared defaults are explicit and permitted.** A struct field may
  declare a default (`struct { x: u64 = 0, y: u64 }`), applied when the field is
  omitted at construction (`Point(y = 2)` ⇒ `x = 0`). An array type carries a
  **type-level** element default symmetrically (`[T = v; N]`), supplied at
  construction by calling the type: `[T = v; N]()` fills every element with `v`, and
  `[T = v; N](1, 2, 3)` takes the leading elements explicitly and defaults the rest
  to `v` (§11). A default applies **at construction**, not at declaration: the
  binding `x : T` is left uninitialized.
- **Definite-assignment analysis** permits deferred initialization when a write is
  provably ordered before every read; **reading uninitialized storage is forbidden**
  (it would be UB, against I11). Immutable fields MUST be assigned at construction
  (Memory §3.2).
- **The analysis is field-sensitive.** Initialization is tracked **per field** (and,
  for a fixed array `[T; N]`, per element at a comptime-constant index): a place may be
  built up incrementally (`p.x = 1` then `p.y = 2`). Reading the **whole** aggregate
  (or passing it by value) requires **every** field definitely assigned; reading a
  **not-yet-assigned field** is the error. A **declared default** counts as
  initialization at construction — so a field omitted at construction but carrying a
  default (incl. an immutable field, e.g. `struct { y : u64 = 0 }`) is definitely
  assigned, satisfying the immutable-field rule. (Initialization is per-field; **moving**
  is whole-value — Memory §5.9.)
- A validity-decorated aggregate `R = @require(pred) U` is the exception at the decorated
  boundary: `U` may be built field-by-field, but `R` itself requires one complete `R(u)`
  gate and its components are not mutable in checked code (§8.1).
- **`uninit`** is the explicit opt-out: in checked code a read of `uninit` storage
  is caught by definite-assignment analysis; an unchecked read (inside an
  `unchecked` grant) yields hardware-defined contents.

---

### 10. Arithmetic, overflow, and floating point (cross-references)

These are normatively specified in the Concurrency-Overflow-Float chapter; the type
system fixes only that they are **defined** (I11):

- **Integer overflow** is defined behavior, never UB: by default it **traps**, inside
  an `unchecked` scope it **wraps** (CG-7), and explicit
  policy operations select wrap/saturate/checked/overflowing. The **full set** — which
  operations trap, which guards are separate (division by zero, over-width shift), and
  the explicit-operation families — is enumerated in the Concurrency-Overflow-Float
  chapter §6; it is **not** restated here.
- **Floating point** is deterministic: compile-time evaluation is strict IEEE-754
  independent of the host; run-time evaluation is the target's IEEE-754 with **no**
  hidden contraction (e.g. no implicit fused multiply-add).
- **Operators**: bitwise `&` `|` `^` `~` and comparisons are greedy overloadable
  functions yielding their result; the logical `and` / `or` / `not` are
  short-circuiting **control flow** (keywords), so their laziness is visible (I3).

---

### 11. Syntax (grammar fragments)

The forms this chapter introduces are given here as **grammar fragments**; the
**consolidated EBNF** and lexical rules (numeric/string/character literals, identifiers)
are assembled in the Grammar chapter, and these fragments conform to the unified
declaration model (Declarations chapter). Nonterminals not defined here (`ident`,
`expr`, `arg-list`, `int`, `float`, `bool-lit`, `char-lit`, `string`, `item-sep`,
`field-list`, `variant-list`) come from those chapters.

```ebnf
(* type literals — value-expressions of kind `type` (§5.1–5.2) *)
struct-type   ::= "struct" "{" field-list "}"
enum-type     ::= "enum"   "{" variant-list "}"
union-type    ::= "union"  "{" [ union-member { item-sep union-member } ] "}"
array-type    ::= "[" type-expr [ "=" expr ] ";" expr "]"  (* [T; N] | [T = v; N] — N comptime (§6.4);
                                                              the optional "= v" is the type-level
                                                              element default (§9.4) *)
tuple-type    ::= "(" [ type-expr { "," type-expr } ] ")"
type-expr     ::= { attribute } type-atom              (* prefix attributes decorate a type: @align(N) T, @require(pred) T, @nonzero u32 (§8/§8.1) *)
type-atom     ::= ident                                (* bitsN, uN/iN/fN, usize, char, str, bool, … *)
                | struct-type | enum-type | union-type | array-type | tuple-type
                | type-atom "(" arg-list ")"           (* generic instantiation: Vec(T), Option(T), Result(T,E) *)
                | fn-type                              (* function-value type — a non-capturing code-pointer, one word (Functions §1.5, FN-10) *)
                | "dyn" fn-type                        (* type-erased closure type — a {code, env} fat pair, env in explicit storage (Functions §1.6, FN-11) *)
fn-type       ::= "fn" "(" [ fn-type-param { "," fn-type-param } ] ")" [ "->" type-expr ]  (* bodyless signature in type position; no names, no block *)
fn-type-param ::= [ "in" | "out" | "in" "out" ] type-expr

(* a struct field carries optional attributes (any attribute, CT-10 — `layout-attr`
   is the closed lever subset, not the whole space; legality is semantic), optional
   mut, optional default *)
field         ::= { attribute } [ "mut" ] ident ":" type-expr [ "=" expr ]
union-member  ::= ident [ "(" type-expr { item-sep type-expr } ")" ]  (* zero components omit parentheses (§6.3) *)

(* branding — a builtin yielding a distinct nominal type over a layout (§5.4) *)
brand-expr    ::= "brand" "(" type-expr ")"            (* Meters := brand(u64) *)

(* conversions (§4.4) — value conversion vs bit reinterpretation are distinct *)
value-conv    ::= type-expr "(" expr ")"               (* T(v): lattice conversion,
                                                            @convert, or explicit
                                                            require gate (§8.1) *)
bitcast-expr  ::= "bitcast" "(" type-expr "," expr ")" (* ≡ expr ".bitcast" "(" type-expr ")" ; equal-width *)

(* layout-control attributes (§8) — @-attributes on a type or field *)
layout-attr   ::= "@repr"    "(" type-expr ")"          (* pin an enum tag's underlying type (§8) *)
                | "@packed"
                | "@align"   "(" expr ")"
                | "@offset"  "(" expr ")"
                | "@endian"  "(" expr ")"
                | "@niche"   "(" expr ")"               (* constructive invalid-pattern producer (§6.2) *)
(* `@abi` is NOT a layout-attr — it is a function-declaration calling-convention attribute (Functions §6); v1 has no type-target `@abi` (§8). *)

(* value construction (§9) *)
literal       ::= int | float | bool-lit | char-lit | string  (* no byte-string literal; exact bytes = [u8; N] value *)
struct-ctor   ::= type-expr "(" [ field-init { "," field-init } ] ")"  (* Point(x = 1, y = 2) *)
field-init    ::= ident "=" expr
tuple-ctor    ::= "(" ")"                                               (* unit value; type `()` *)
                | "(" expr "," [ expr { "," expr } ] ")"              (* (1, 2) *)
variant-ctor  ::= type-expr "." ident [ "(" arg-list ")" ]            (* Color.Red | Option.Some(v) | Cell.pair(a,b); union arity §6.3 *)
array-ctor    ::= "[" expr { "," expr } "]"                          (* [1, 2, 3] — listed *)
                | "[" expr ";" expr "]"                              (* [v; N] — replicated *)
                | array-type "(" [ expr { "," expr } ] ")"          (* [T = v; N]() all defaulted to v;
                                                                       [T = v; N](1, 2, 3) first explicit,
                                                                       rest defaulted (§9.4, CG-2) *)
uninit-expr   ::= "uninit"                                          (* explicit opt-out (§9.4) *)
```

Prefix type attributes apply nearest to the `type-atom` first; repeated `@require`
decorators therefore nest as specified in §8.1.

Normative notes:

- An **integer literal** has no type suffix (§9.2); its type is inferred from
  context, and out-of-range is a compile error (§9.1). Refinement is via `T(v)` or
  an annotation.
- `T(v)` (value conversion) and `bitcast(T, x)` (equal-width bit reinterpretation)
  are **distinct** operations (§4.4); they are not two spellings of one.
- For `R = @require(pred) U`, `R(v)` is the separate explicit constructor gate of §8.1,
  not an implicit conversion or an inherited aggregate field constructor.
- Generic instantiation uses **call syntax** on a type value (`Vec(T)`,
  `Result(T, E)`), consistent with the Comptime chapter (a type is a value).
- A field default and an array element default (`[T = v; N]`) apply **at
  construction**, not at declaration (§9.4).

---

### 12. Conformance (normative summary)

A conforming implementation MUST:

1. treat `type` as a compile-time value and erase all type-level computation before
   run time (§1; Comptime chapter; I7);
2. provide `bitsN` only at the target's native widths, with the
   interpretation-independent operation set of §2.2 and no arithmetic on it;
3. define `u`/`i`/`f` as zero-cost interpretations over blocks, with signedness in
   the operations, and make them available under every limit (§3);
4. enforce nominal identity per declaration and structural identity only for
   anonymous layouts (§4.1);
5. classify every conversion into exactly one lattice class, make only **widen**
   implicit (and nothing implicit under `no_abstractions`), and keep `T(value)`
   (value conversion) and `bitcast` (equal-width bit reinterpretation) as distinct
   operations (§4.2–4.4);
6. build all types from the five layout primitives of §5.1 and provide no layout
   outside their composition;
7. lay out aggregates exactly per §6 — declaration order, standard padding,
   rule-determined niches (§6.2; never an opaque optimizer), the raw-union
   payload/layout/construction/definitely-active rules of §6.3 with no hidden tag,
   contiguous bounds-checked arrays, and zero-sized types emitting no bytes;
8. honor the §8 layout-control attributes as part of the type contract;
9. implement `@require` exactly as the §8.1 constructor gate: distinct layout-identical
   identity; a comptime-known `fn(in U) -> bool`; non-owning `U`; comptime-only captures;
   explicit immediate-`U` construction with §4.6 gate precedence, no implicit conversion
   and no composed conversion chain (only a direct `@convert` to `R` may otherwise match);
   once-only evaluation/call and preserved result; ordinary ABI aggregate copying; direct
   target trap on false; complete omission under `unchecked`; comptime/global evaluation;
   no partial initialization or checked component mutation; and nearest-first stacking;
10. infer literal types from context, reject out-of-range literals at compile time,
   provide no literal suffixes, and impose no implicit zero-initialization while
   forbidding reads of uninitialized storage in checked code (§9; I11);
11. define integer overflow and floating-point behavior per §10 with no undefined
    behavior (I11).
