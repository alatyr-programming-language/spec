# Alatyr Language Specification

## Chapter — Memory Model

> **Status — draft (under review).** This chapter is a draft and is not yet
> accepted; the specification as a whole is under development. See the status note
> at the top of the Overview chapter.

This chapter defines how an Alatyr program names, stores, addresses, and manages
the lifetime of data. It is foundational: the Type System, Functions, and Control
Flow chapters build on the vocabulary and rules established here.

Requirement keywords (**MUST**, **MUST NOT**, **SHOULD**, **MAY**) follow RFC 2119
(Overview §6). This chapter gives the **concrete syntactic forms** it introduces as
grammar fragments (§7); the *consolidated* EBNF and lexical rules are assembled in
the Grammar chapter, and the forms here conform to the unified declaration model of
the Declarations chapter. Per-architecture instruction selection for the lowering
contracts below is given in the Assembly-Correspondence and Codegen chapters; this
chapter is normative for *what* each operation means and *which* machine effect it
MUST produce, not for the exact mnemonic on a given target.

---

### 1. Values, places, bindings, and addresses

Alatyr distinguishes four foundational entities. They MUST NOT be conflated.

#### 1.1 Value

A **value** is a typed datum (e.g. the integer `5`, `true`, a struct
`{ x: 1, y: 2 }`). A value has a type (a layout recipe; see the Type System
chapter) but has **no identity, no location, and no scope**. A value is the
result of evaluation.

#### 1.2 Place

A **place** (storage location) is a location that holds a value. A place has:

- a **type**;
- a **storage class** — where it lives (§2): register, stack, static, or
  allocator-managed;
- a **mutability** — whether it may be written through a given access path (§3);
- a **lifetime** — for how long the location is valid (§5).

A place is **addressable** if and only if it is **memory-resident**. A
register-resident place is **not** addressable (§2.6). Taking the address of a
place yields an address value (§4).

#### 1.3 Binding

A **binding** associates a **name** with a place. A binding has a **scope** — the
lexical region in which the name is visible (defined in the Declarations chapter).
Scope is a property of the binding (the name), not of the place or the value.

#### 1.4 Address

An **address** is a **value** (of pointer width on the target) that denotes the
location of a memory-resident place. Because it is a value, an address may be
stored, passed, and computed with, subject to §4. Register-resident places have
no address.

An address obtained by taking the address of a place (§4.3) is **well-formed**: it
denotes that memory-resident place for as long as the place is valid (§5). An
address **fabricated** under `unchecked` — from an integer or by pointer arithmetic
(§4.5) — denotes whatever location its bits name; ensuring it actually addresses a
valid place is the programmer's responsibility, and access through one that does
not has hardware-defined behavior (I11). The guarantees in §4 (non-nullness, the
niche-`0` optional) hold for the checked `ptr(T)` type, not for raw fabricated
bits.

#### 1.5 Three orthogonal axes

A binding-and-place carries three **orthogonal** properties; an implementation
MUST NOT collapse them into one (cf. C's conflation of `static`):

| Axis | Question | Carried by |
|------|----------|-----------|
| **Space** — storage class | *where* the value lives | the place (§2) |
| **Time** — lifetime / extent | *how long* the place is valid | the place (§5) |
| **Name** — scope | *where in the source* the name is visible | the binding |

Their divergence — a name leaving scope while an address to its place is still
live — is the dangling problem, addressed in §5.

#### 1.6 Place-expressions and value-expressions

Every expression is classified as either a **place-expression** or a
**value-expression**.

- **Place-expressions** denote a place: a variable reference, a field access
  `s.f`, an index `a[i]`, and a dereference (§4.3).
- **Value-expressions** denote a value: literals, arithmetic, comparisons, a call
  whose result is taken by value, and an address-of (which yields an address
  *value*).

The fundamental operations relating values and places are:

- **Load** (place → value): reading a place in value context yields its value.
- **Store** (value → place): assignment evaluates the right-hand value-expression
  and writes it into the left-hand place-expression.
- **Address-of** (place → address): yields the address of a memory-resident place
  (§4).
- **Dereference** (address → place): yields the place that an address denotes
  (§4).

A store's left operand MUST be a place-expression; assigning to a value-expression
is ill-formed.

A **compound assignment** `place ⊕= rhs` (Grammar §3.2; OP-2), where `⊕` is one of the
binary glyph operators `+ - * / % & | ^` (OP-2), is **sugar** lowered by this documented
rule (I1; nothing hidden, I3): the **place is evaluated once**. Its address is taken once
into a fresh temporary `t`, then the store is `deref(t) = deref(t) ⊕ rhs` — so any
side effects in the place (an index `a[f()]`, a dereference `deref(p())`) occur
**exactly once**, and `rhs` is evaluated once, after the place. For a plain binding the
rule degenerates to `x = x ⊕ rhs`. Because `⊕` is an overloadable operator-function (Type
System §5.3; TYP-5), `⊕=` applies to any type that defines `⊕`; it introduces no new
overload point. The place MUST be mutable (§3) and definitely assigned (it is read), as
the desugared form requires. Compound assignment is a **statement**, not an expression
(Grammar §3.1), like ordinary `=`.

> Copy versus move semantics of a value as it is read, assigned, or passed is
> defined in §5.9.

---

### 2. Storage classes

#### 2.1 The four classes

| Class | Addressable? | Default lifetime | Cost | Notes |
|-------|:---:|------|------|-------|
| **register** | no | function activation (often until last use) | none | finite count; ABI-governed save rules |
| **stack** | yes | function activation (LIFO) | cheap (SP adjustment) | bounded extent; exhaustion behavior is target-/toolchain-defined |
| **static** | yes | whole program | none at run time | address fixed at link time |
| **allocator-managed** | yes | governed by a lifetime mechanism (§5) | the allocation's cost (visible) | requires an allocator; see §5 |

Dynamic allocation is the only class with a non-trivial cost and is therefore
always explicit (I3): it is requested through a lifetime mechanism (§5), never
implicitly.

#### 2.2 Default storage class by declaration site

- A binding declared **inside a function** defaults to **automatic** storage
  (register or stack), chosen by the placement strategy (§2.5).
- A binding declared at **module (top) level** defaults to **static** storage.

A module-level data binding MUST be initialized with a compile-time-known value
(or left explicitly uninitialized; see the Declarations chapter). **Run-time
computation MUST NOT occur during module-level initialization** — there are no
implicit static initializers and therefore no static-initialization-order
problem. Run-time initialization of global state is performed by explicit
initialization functions the program calls.

#### 2.3 Static sub-classification (derived)

For a static binding the emitted section is **derived**, not written by hand, by
the following rule:

- immutable and initialized → `.rodata`;
- mutable and initialized → `.data`;
- mutable and uninitialized → `.bss`.

An explicit `@section("name")` overrides this derivation.

#### 2.4 Storage specifiers

The default storage class MAY be overridden per binding by a storage specifier
(attribute form, Declarations chapter):

- `@reg` — force a register; `@reg(name)` — force a specific named register (§2.6);
- `@stack` — force the stack;
- `@static` — force static storage (whole-program lifetime);
- `@scoped` — the scoped (second-class) lifetime; non-allocating (§5.3);
- `@alloc(allocator)` — dynamic allocation through the given allocator value; the
  allocator's type selects the lifetime mechanism (§5). The binding it introduces is the
  mechanism's **reference** (for a region, `@alloc(a) x := init` binds `x : Handle(T)` —
  §5.4 / MEM-3), not the value itself; capacity exhaustion traps (the recoverable path is
  `allocator.allocate(T, …)?`, whose result is typed by the mechanism — `Handle(T)` for a
  region, Stdlib appendix §5.1).

`@reg`, `@stack`, `@static`, and `@scoped` are non-allocating. Dynamic allocation is
exclusively `@alloc(...)`.

#### 2.5 Placement strategy

Within a function, the choice between register and stack for an automatic binding
is made by a **documented, deterministic placement strategy** (call it *S2*):

- the strategy is deterministic — the same source and target produce the same
  placement;
- spills are emitted by a documented rule, so their **cost** is predictable
  (I1);
- the exact register *number* is an unobservable detail (it changes neither
  behavior nor cost) and is therefore not subject to hand tracing.

A conforming implementation MUST document its placement strategy and MUST be
deterministic. It MUST NOT perform placement that introduces hidden cost beyond
the documented rule.

Under the `no_abstractions` limit the impl-documented S2 strategy does **not**
apply; placement is **fully determined by the source plus a fixed, spec-defined
rule**, so the emitted output is byte-exact **and identical across conforming
implementations** (I1) — there is no allocator choice left:

- a binding with an explicit storage specifier (`@reg` / `@reg(r)` / `@stack` /
  `@static`) is placed exactly there;
- an un-pinned automatic binding or an anonymous expression temporary occupies the
  target's **caller-saved (scratch) registers in the order listed by `caller_saved`
  in the ABI appendix §1 (the `Abi` schema)**, assigned in **source evaluation order** and released
  at **last use** (a deterministic liveness over the source); when the scratch
  registers are exhausted it **spills to stack** by the same documented rule (the
  next free slot in evaluation order).

So two implementations emit the same registers and the same instructions for a
`no_abstractions` unit; byte-exact prediction holds for the programmer and across
implementations (I1).

#### 2.6 Register residence and addressability

A register-resident place has **no address**. Therefore:

- Under the `no_abstractions` limit, taking the address of a register operand is
  **ill-formed** (a register is not addressable in the ISA).
- Otherwise, taking the address of a binding the strategy placed in a register
  **forces it into memory** (a spill) by a documented rule; the spill is
  predictable (I1), not hidden.

Pinning a binding to a specific register is `@reg(name)`; taking its address is
then ill-formed.

---

### 3. Mutability

#### 3.1 Immutable by default

A binding is **immutable** unless declared `mut`. Mutation is therefore a visible,
opt-in property (I3/I8). Mutability is a **write permission attached to the path**
by which a place is reached (binding, field, or pointer) — **not** a property of
the value. A value has no mutability.

#### 3.2 Two axes: per-binding and per-field

- **Per-binding:** `x` (immutable) versus `mut x` (mutable).
- **Per-field:** a field of an aggregate is immutable unless declared `mut`. A
  `mut` field is *permitted* to be written, giving a type-guaranteed permanently
  immutable field where it is not `mut` (an invariant field set once at
  construction, regardless of the holder).

An immutable field MUST be initialized at construction (definite assignment); it
MUST NOT be assigned afterward.

#### 3.3 The AND rule

The effective write permission for a path is the **logical AND** of every step:
writing `c.buf.x` requires that `c` is mutable **and** field `buf` is mutable
**and** field `x` is mutable. Any immutable step makes the store ill-formed.

When a step is a dereference, the relevant permission is the **pointee
mutability** of the pointer's type (§4), not the mutability of the pointer
binding.

#### 3.4 Checking

Mutability is checked **at compile time, always** — independent of `checked`/`unchecked`.
A write through an immutable path is ill-formed.

---

### 4. Pointers

#### 4.1 Pointer types

A pointer type carries:

- a **target type** `T`;
- the **pointee mutability**: `ptr(T)` (immutable pointee) or
  `ptr(mut T)` (mutable pointee);
- an **alignment** — by convention the natural alignment of `T` from the machine
  model, overridable per site (`@align(N)`); the pointer type carries the actual
  alignment so that checked access adapts;
- **non-nullness**: a value of `ptr(T)` is **always** a valid (non-null)
  address.

#### 4.2 Absence — the optional pointer

There is **no null-pointer value**. Absence is expressed as
`Option(ptr(T))`. The `None` case is represented by the documented niche
sentinel — the all-zero address — so that `Option(ptr(T))` has the **same
width as a raw pointer** (zero cost, I2) and the absence check cannot be
forgotten (the optional MUST be unwrapped to obtain a `ptr(T)`). A null
dereference is therefore unrepresentable.

#### 4.3 Operations

- **Address-of**: `ptr(x)` (equivalently `x.ptr()`) takes a memory-resident place `x`
  (§1.6, §2.6) and yields an address value.
- **Dereference**: `deref(p)` (equivalently `p.deref()`) takes an address `p` and yields
  the place it denotes. `deref(p)` in value context loads; `deref(p) = v` stores. The write
  permission of the resulting place equals the **pointee mutability** of `p`'s
  type (§3.3).
- **Layout queries** (comptime, type-level): `T.size()` and `T.align()` yield a type's
  size and alignment in bytes (`usize`); a field's offset is read from `typeinfo(T).fields`
  (each `Field` carries its `.offset`). They are pure comptime built-ins, fully erased
  (I7) and derivable from `typeinfo(T)`; the Stdlib appendix §4.1 lists their signatures.
  For a value, `typeof(v).size()`. A field is matched against `T`'s fields by **name** —
  unambiguous (no field-vs-value parse), an absent name a comptime error.

Pointer operations are the **flat prelude word-functions** `ptr` (take address) and
`deref` (follow an address); layout is the word-functions `T.size()` / `T.align()` (plus
`typeinfo`); the addressing operand `at` lives on the arch surface (Assembly §7). There is
no `mem` module. Each word-function exists in both spellings by UFCS — prefix `ptr(x)` or
postfix `x.ptr()` (MOD-3/CT-10). A bare field access `x.f` (no call) is distinct: a
`.`-postfix is UFCS only when followed by a call (β, Grammar §3.4), so these built-ins do
not occupy field space.

`ptr` addresses a **data place** and yields a `ptr(T)` because a data place's
name, in value position, yields its **loaded value** — so getting its address needs
an operation. A **code** entity (a function, or a code-point label, Control Flow §2.1)
is the opposite: its name in value position **already is** its address (code has no
"value to load"), so it needs **no** address-of operation — there is no `ptr` for
code. (A function's address is a proper, callable **code pointer**; a code-point
label's is a **raw code address**, not callable; a structured loop/block label has no
address at all — Control Flow §2.1; decisions CF-5.)

#### 4.4 Monotonic permissions

The permission produced by address-of MUST be **less than or equal to** the write
permission of the place:

- an immutable place yields only `ptr(T)`;
- a mutable place yields `ptr(mut T)` **or** `ptr(T)` (a read-only pointer
  to a mutable place is permitted and useful).

A permission may only be **dropped**, never gained: the coercion
`ptr(mut T)` → `ptr(T)` is allowed; the reverse is ill-formed. Taking
`ptr(mut ...)` to `c.f` additionally requires the whole AND-path to `c.f` to
be writable (§3.3).

Pointers to distinct named target types (e.g. `ptr(Meters)` versus
`ptr(Seconds)`) are incompatible without an explicit `bitcast` (nominal
identity; Type System chapter).

#### 4.5 Pointer arithmetic and integer↔pointer

Pointer arithmetic and fabricating a pointer from an integer (`bitcast`) are
capability-requiring operations: they are legal only inside an `unchecked` grant
(or the assembly surface), and there they have hardware-defined behavior (I11).
Outside such a grant they are ill-formed. Checked code traverses by
**bounds-checked indexing** through typed arrays and slices; therefore checked
code contains no pointer-arithmetic UB.

#### 4.6 Lowering contract

- Address-of a memory-resident place lowers to the target's effective-address
  computation (e.g. a load-effective-address or address-materialization
  sequence), per the Codegen chapter.
- Dereference lowers to a load (value context) or store (assignment context) at
  the addressed location, honoring the place's type, alignment, and the target's
  endianness.
- `Option(ptr(T))` lowers identically to a raw pointer-width value; the niche
  sentinel `0` denotes `None`, and a non-`None` value is the address.

No address operation introduces hidden control flow or allocation (I3).

---

### 5. Lifetime, ownership, and resource management

#### 5.1 The problem

**Temporal safety:** no place may be accessed through an address after the
place's lifetime has ended (use-after-scope, use-after-free, use-after-realloc).
By I11 this MUST be guaranteed in checked code, either statically or by a
deterministic trap; under an `unchecked` grant / assembly the behavior is
hardware-defined and is the programmer's responsibility.

Alatyr does **not** adopt a borrow checker or a mutable-XOR-shared discipline
(§5.7). Temporal safety is provided by a small **menu of lifetime mechanisms**.

#### 5.2 The mechanism menu

The language has **one** reference: the **scoped** (second-class) reference (§5.3) — thin, zero-cost,
non-escaping (the only first-class storage is the value itself). Longer-lived allocation-and-lifetime
is **not a second language reference kind** but a **library allocator-provider menu** (MEM-1): a
provider (region/arena, generational, manual, …) is an ordinary library allocator value, selected per
site through `@alloc(value)` (§2.4) by the allocator value's type, and it hands out a
**first-class reference token** — for a region, the ordinary library handle type
`Handle(T)`, a number (§5.4), not a language reference. Each provider carries its own
validity check and independently upholds I11; the cost is visible because the token and
the access (`get`) are explicit (I1/I2). The **`manual`** provider is the raw-pointer escape: forming, freeing, or
dereferencing its pointer requires an enclosing `unchecked` grant (§5.6 rule 5; §4.5) and is forbidden
under `no_unchecked` — the one provider whose "check" is the programmer's responsibility, so checked
code never reaches it silently. The table below summarises each provider's reference representation; only
**scoped** is a language reference, the rest are library providers over the allocator `Ref` protocol (Stdlib
appendix §5.1).

| Mechanism | Reference representation | Validity check | Cost |
|-----------|--------------------------|----------------|------|
| **scoped** (default) | thin pointer | static (cannot escape) | zero |
| **region / arena** | **handle** (an index, not a pointer) | checked at access against the owning arena | bulk-free; per-access check only for the handle |
| **generational** | fat (pointer + generation) | generation compared at dereference; stale → trap | one comparison per deref |
| **manual** | raw pointer | none; `unchecked`-only | zero; hardware-defined |

Additional mechanisms (e.g. reference counting) MAY be added in later versions
without breaking existing programs (I10); the menu is fixed within a version.

#### 5.2.1 The ambient allocator (`alloc::with`; MEM-5)

An allocator is passed as an ordinary parameter of the library protocol **`alloc`** (the
region protocol, Stdlib appendix §5.1) — `(A : alloc, in a : ptr(mut A))`,
monomorphized per concrete allocator (zero-cost, allocator-polymorphic). Its direction is
**`in`** (a pointer to mutable), never `in out`: the grammar forbids a default / optionality
on `in out` (Functions §5.1), and an `in` pointer is the canonical by-reference input (§2.2)
— allocation mutates the arena **through** the pointer, explicitly. An allocator parameter
is distinguished **only by its type** (no marker, OP-1) and carries two extra powers: within
the body it **is** the ambient allocator, and at a call site it is **elidable** (Functions
§5.5).

`alloc::with(ar) { … }` — a prelude form recognized by the compiler — establishes (or, when
nested, overrides) the **ambient allocator** `ar` for the enclosing region; the value is an
ordinary place (a binding in scope), referenced by name for explicit/manual use (there is no
reflective accessor — it would duplicate a visible name, OP-1). The **outermost**
`alloc::with` (in `main`) is the program's de-facto default; there is **no manifest or
process-static global** allocator (no silent static state; I3 / STD-3). An allocating site with
**no** resolvable ambient and no default is a **compile error**, never a hidden heap.

`alloc::with` is a **runtime** allocation scope, so it is **ill-formed inside a `comptime`
context** (a `comptime` block / `comptime if` / a comptime function body): comptime evaluation
allocates nothing observable (its values are erased, I7), so there is no ambient to establish —
a comptime allocation uses the evaluator's own working store, not a program allocator. Under the
**`no_alloc`** limit, `alloc::with` — like any allocating site — is a **compile error** (allocation
is a forbidden capability there). **Nesting** is plain **shadowing**: an inner `alloc::with`
overrides the ambient for its block; the outer is restored on exit (lexical, no combining).

A function relates to allocation in one of three shapes:

- **(A)** **no allocator parameter** — self-contained (it owns and frees its own arena) or
  non-allocating;
- **(B)** **`in a : ptr(mut alloc)`** — a required, ambient-elidable allocator; the shape
  for an **escaping** producer (a container constructor returns storage backed by the caller's
  arena, which outlives the call);
- **(C)** **B with a Functions §5.1 default** — always short-callable, but the default is a
  call-site temporary, so it is sound **only when the allocations do not escape** (scratch);
  the escape checker (§5.4 / §5.6) enforces this. An escaping producer therefore uses B, not a
  create-fresh default.

#### 5.3 Scoped references (second-class)

A scoped reference is a thin pointer bound to a lexical scope or region. It is
**second-class**: it **MUST NOT** escape — it cannot be stored into a place that outlives the
referent, nor returned upward **except** through a declared **`scoped` return** (§5.3.1), whose result
is itself second-class at the call site. This is checked statically; a scoped reference therefore
cannot dangle. Borrowing a scoped reference does **not** consume an owning value (§5.9).

#### 5.3.1 The escape rule (second-class flow discipline)

A scoped reference flows only **downward** — into nested scopes and into callees —
never **upward**. The check is **structural and local**: lexical scope / region
nesting plus the function signature, with **no lifetime variables and no
cross-function inference** (this is why scoped references need no borrow checker, MEM-1).

**The "outlives" relation.** Places are ordered by **lexical scope / region nesting**:
P₁ *outlives* P₂ iff P₁'s scope strictly encloses P₂'s. A `static` or
allocator-managed place outlives every automatic place; an outer block outlives an
inner one; a caller's activation outlives the callee's. A scoped reference borrowing
referent R may be stored only into a place that does **not** outlive R.

**Permitted flows** (well-formed):

- dereference and read/write through it (subject to mutability, §3);
- store into a binding or aggregate field of the **same or an inner** scope;
- **pass as an argument** to a callee (the callee receives a scoped parameter, below);
- take a further scoped borrow of a sub-place (downgrade only, §5.6 rule 1).

**Forbidden flows** (a compile-time diagnostic):

- appearing in `return` / `-> T`, or being assigned to an `out` parameter (escape
  upward) — **unless** the result is itself declared a **second-class return** (below);
- storing into a `static` / allocator-managed place, or into a field of any aggregate
  that outlives R;
- being **captured by a closure that outlives R** — capturing a scoped reference makes
  the closure itself scoped (§6.3), bound by these same rules; a **`dyn`** closure
  (§6.3 / FN-11) likewise borrows its env storage and MUST NOT outlive it.

**Second-class returns (the dual of a scoped parameter, MEM-3).** A function MAY return a scoped
reference when its result type carries the **`scoped`** marker (`-> scoped ptr([mut] T)`).
Such a result is, **at the call site**, an ordinary second-class reference: the caller may deref it
and pass it further down, but may **not** store it where it outlives the call, return it (unless its
own result is likewise `scoped`), assign it to an `out`, or capture it in an outliving closure — the
same escape rule, now applied to the *result*. The marker is **local and unparameterized**: it says
only "this result does not escape the call," **not** "scoped to argument *X*" — there is **no**
binding of the result's scope to a particular parameter's lifetime, so it introduces **no lifetime
variable and no cross-function inference** (MEM-1 holds). The referent must, of course, outlive the
call — which the body guarantees by construction (it returns a borrow of a parameter or of
allocator-managed storage live across the call); the call site treats the result as scoped to **its
own** activation. This is exactly the surface by which `get` (§5.4) is an ordinary function, not a
privileged intrinsic.

The `scoped` marker is **meaningful only on a function result type**. Every reference is
already second-class by default (§5.3), so the qualifier carries information **only** where it
overrides the would-be-escaping default — the return. A `scoped` qualifier written in **any
other** type position — a parameter slot, a field type, a local binding's type, a nested type
argument — is **ill-formed (a Semantic error)**, *not* silently ignored (I3: a token that reads
as meaningful must be): the diagnostic directs the author to drop it (references are scoped
already) or, for a parameter that must *not* escape into the callee's outer state, to rely on the
default scoped-parameter contract (below). The grammar admits the qualifier in any type-atom
(Grammar §3.3) purely so this rejection is a semantic rule with a clear message, not a parse
error.

**Scoped parameters (the callee contract).** A parameter that is a scoped reference is,
inside the callee, a scoped reference whose referent's scope is **the call**: it does
not outlive the callee's activation. The callee may deref it, store it into its own
inner scopes, and pass it further down, but may **not** return it, assign it to an
`out`, or store it where it outlives the call. Nothing crosses the signature beyond
"this parameter is a scoped borrow," so caller and callee are each checked **locally**
and the analysis composes.

**Examples.**

- well-formed: `{ p := ptr(local); read(p) }` (used within the scope); passing
  such a `p` to a helper that only reads it.
- ill-formed: `fn leak() -> ptr(u64) { x := 0 ; ptr(x) }` (returns a scoped
  reference upward); storing `ptr(local)` into a module-level `mut` global;
  capturing `ptr(local)` in a closure that is stored beyond the scope.

Storing a long-lived link uses a **first-class** mechanism instead (a handle or a
generational reference) — crossing to it is **explicit** (§5.6 rule 4); escaping to a
raw pointer requires an `unchecked` grant (§5.6 rule 5).

#### 5.4 Regions and handles

A **region** (arena) is the allocation-and-lifetime unit for long-lived data; its
storage is freed in bulk when the region ends. Long-lived links into a region use
**handles** (plain index values), not pointers: a handle has no lifetime problem
because it is just a number; its validity is checked at access against the owning
arena. This expresses graphs and intrusive structures without dangling pointers.

The concrete v1 surface (MEM-3): a handle is the ordinary library type `Handle(T)` (a number
— distinct per `T` because a type-function result is nominal by (function, args), Type
System §4.1 — not a language construct, and carrying **no** region name or lifetime
variable). Allocation uses the `@alloc` storage attribute (§2.4):
`@alloc(a) x := init` allocates through `a`'s allocator protocol (Stdlib appendix §5.1),
writes `init`, and binds `x : Handle(T)`; capacity exhaustion **traps** (a defined
failure, I11), and the recoverable path is to call `a.allocate(T, …)?` directly — its
result is typed by the mechanism, so for a region it already yields `Handle(T)` (§5.1). A handle is
**not** dereferenced (no `deref` on a handle; §5.6 rule 4): the value is reached by
exchanging *handle + arena* for a **scoped reference** with `get(a, h) -> scoped ptr(mut T)`,
which bounds-checks the handle against the arena (out-of-range → trap) and returns a pointer the
caller then `deref`s. `get` is an **ordinary prelude function** (MEM-3): its result is declared a
**second-class scoped reference** (the `scoped` return marker, §5.3.1), so the escape checker applies
its **generic** second-class rule — the result cannot be stored or returned past the call site — with
**no** arena-specific compiler contract and **no** lifetime variable (the result is scoped to the
**call site**, not tied to the arena *argument*; a user could write the same signature). Temporal
safety rests on the arena's **linearity** (§5.9 — the arena is consumed by close, so `get` cannot be
called after it, and a get-result cannot be stored to outlive the call) plus that bounds check, not on
the handle's type. A get-result held across an arena **growth/realloc** within the same scope is the
residual unchecked-flavoured hazard (a logic error, hardware-defined under §4.5, never UB) — the
handle is the safe long-lived form; the get-result is transient, used at once.

#### 5.5 Generational references

A generational reference is a fat value (pointer plus a generation tag). The
allocation slot carries a generation counter; a dereference compares the
reference's generation against the slot's, and a stale reference **traps** (a
defined failure, I11). This provides a freely-storable, self-validating reference
at the cost of one comparison per dereference.

#### 5.6 Seam rules between mechanisms

Interactions across mechanisms are governed by these principles (not by an
unbounded matrix):

1. **Downgrade only.** A reference of greater validity is usable where lesser-or-
   equal validity is required; a **scoped borrow** of any mechanism's referent is
   always permitted.
2. **No escape / no upgrade.** A scoped reference MUST NOT be stored into a longer-
   lived place; a permission/validity may only be dropped, never gained.
3. **First-class vs second-class.** Handles and generational references are
   first-class (storable anywhere; self-validating at access). Scoped references
   are second-class.
4. **Distinct representations do not silently convert.** Handle, generational
   reference, and thin pointer have different representations; crossing between
   them is explicit (re-acquire, checked, or `unchecked`).
5. **`unchecked` is the escape.** Conversions to and from a raw pointer are explicit
   and require an `unchecked` grant.

#### 5.7 No mutable-XOR-shared discipline

Because the optimizer is **semantically-preserving** and makes **no aliasing assumptions** (CG-5), the
optimization rationale for a mutable-XOR-shared aliasing discipline does not apply; Alatyr does not impose one.
Safety against invalidation is provided by scoped references (which cannot
outlive) and handles (validated at access). Data races have hardware-defined,
not undefined, behavior (Concurrency chapter); race-freedom is a matter of
discipline, not a static guarantee.

#### 5.8 Cleanup — `defer`

Cleanup is **explicit**, not via implicit destructors (I3). A `defer` registers a
cleanup action (an expression or block):

- deferred actions run in **LIFO** order;
- they run when control leaves their scope by **any normal means** — fall-through,
  `break`, **`continue`** (which exits the loop body for that iteration), `return`,
  and the `?` early-exit — and so discharge linear obligations (§5.9) on all such
  paths;
- they do **not** run on abort, trap, or panic — the process terminates and there
  is no stack unwinding (unwinding would be hidden control flow, I3);
- they MUST NOT change the computed result of the function.

#### 5.9 Copy, move, and linearity

**Ordinary values are copied by default.** Assignment and parameter passing copy
the bits; an aggregate is copied byte-for-byte by a documented rule (its cost is
therefore visible, I1). Large values are passed by reference **explicitly** (a
pointer parameter); there are no hidden by-reference copies beyond the documented
ABI lowering (Functions chapter). There is no implicit move for copyable types.

For a raw union `U`, a whole-value assignment, argument, or result copies exactly
`U.size()` bytes, including member-tuple padding and union tail padding (Type System
§6.3/TYP-11). The definitely-active analysis facts transfer only between trackable
places/value expressions as specified there; they have **no run-time representation** and
add no copied byte or ABI refinement. A raw union carrying the `@owning` root marker or
containing an owning payload transitively is ill-formed in v1: overlap cannot be used to
copy, reinterpret, or partially move an owning resource.

**What makes a value owning — the `@owning` marker (MEM-2).** A value is **owning**
iff its type is owning, and a type is owning iff **either** it is a type declaration
carrying the **`@owning`** attribute — the **root** marker that says "a value of this
type stands behind a real resource" — **or** it is an aggregate one of whose fields is
of an owning type (linearity is **contagious**, below). `@owning` is a **semantic marker
only**: it changes **no** layout, size, or alignment (I2 — it is not a machine lever,
Type System §8) and adds **no** run-time data (I3); it is local and visible at the type's
declaration, so an implementation classifies owning-ness **structurally** with no
whole-program inference (the FND-3 bar). There is **no** other way a v1 type becomes owning
— a bare pointer, handle, or integer is **not** owning (a handle is just a number, §5.4);
a resource is owning **because its type is declared so**. (`@owning` attaches to a type
declaration; applying it elsewhere is a Semantic diagnostic — Declarations §2.3.)

**Owning values** (a resource: a file handle, a region, an allocation — each an
`@owning` type) obey stricter rules, because copying a resource handle would permit a
double free (UB, forbidden by I11):

- an owning value is **non-copyable**;
- it is **transferred** (moved); after a transfer the source is invalid, and using
  it is ill-formed;
- a scoped borrow (§5.3) of it does **not** consume it;
- it is **consumed exactly once** — **strict linearity**: an owning value MUST be
  released or transferred exactly once on every normal exit path; the compiler
  enforces this (catching double-release, use-after-release, and leaks). `defer`
  is the idiom that discharges this obligation on all paths.
- The escape valve `forget(x)` discharges the obligation **without** releasing
  (a deliberate leak; a leak is not UB).
- On abort/trap/panic the obligation is moot (the process terminates).
- An owning value whose place has **`@static`** storage (whole-program extent), or that is held by a
  function which **never returns** (result `-> Never`, the bottom type — Stdlib appendix §3.7), carries **no** finalize
  obligation: its extent has no normal exit — it ends only with the program — so there is nothing to
  discharge (the same reasoning as termination). This is the bare-metal "own a peripheral forever" idiom,
  with no `forget` noise.

**Moves are whole-value.** An owning value is moved or consumed **as a whole**; there
is no **partial move** of an individual field out of an owning aggregate (it would
leave the aggregate half-owned, and linearity tracking per path would no longer be a
simple consumed-once obligation). To hand out part of a resource, **restructure** (split
it into separately-owned values) or **borrow** (§5.3). This is the deliberate companion
to **field-sensitive definite-assignment** (Type System §9.4): a value may be *built up*
field by field, but once it owns a resource it is *moved whole*.

Linearity is **contagious**: an aggregate that owns a resource is itself an owning
value. Generic code is polymorphic over ownership.

**What consumes an owning value — `in` is the move (MEM-2).** An owning value is
**consumed** (moved) by being **passed to an `in` parameter** (Functions §2.1): since an
owning value is non-copyable, the §4.1 "`in` large aggregate → by reference to a *copy*"
rule cannot apply, so passing it `in` **transfers ownership into the callee** and
**invalidates the source** (using it afterward is ill-formed). This reuses the existing
direction axis — **no new "move"/"consume" keyword** (OP-1): `os::free(in a)` consumes its
argument; a constructor `T(in r : Owned)` that stores `r` into an owning aggregate
consumes it. A **`return`** of an owning value (or a `-> T` tail, or filling an `out`
result) likewise consumes it — ownership moves to the caller. What does **not** consume:
an **`in out`** parameter (a mutable borrow — the place is read and written in place, the
caller still owns it, §2.1) and a **scoped borrow** via `ptr` (§5.3). Thus the three
directions split cleanly for an owning value: `in` = move, `in out` = mutable borrow,
`ptr` = scoped borrow.

This consume rule is why a validity-decorated type `@require(pred) U` requires its
immediate underlying `U` to be **non-owning** in v1 (Type System §8.1): checked
construction must preserve the result while supplying `pred` an ordinary by-value
`in U`, which is a copy for an ordinary value but would consume an owning one.

A **read accessor** on an owning value is therefore written as a function taking a
**`ptr([mut] R)`** scoped reference (not `in R`, which would move it); the UFCS
**auto-ref** form (Type System §4.5 / MOD-3) supplies the address, so `a.len()` ≡
`len(ptr(a))` reads `a` without consuming it. A container's `free`/`close` takes the
value `in` (the consume); its inspectors and element accessors borrow by scoped reference.

The **dual** coercion, **auto-deref** (Type System §4.5 / OP-5), reads *through* a pointer: when
the receiver/base is already a scoped reference `ptr([mut] R)` and the position wants the
pointee `R`, it supplies `deref(p)` — `p.field`, `p[i]`, `p.m(args)` reach the pointee one
level. Like auto-ref it is a **borrow/read** path: `deref` of a scoped reference inspects the
pointee (§5.3) and never moves it, so an owning pointee is read, not consumed; a by-value move
still requires an explicit `in` argument.

---

### 6. Closures and function values

#### 6.1 Non-capturing function value

A function value that captures nothing is a **code pointer** (pointer width):
zero cost, called indirectly.

#### 6.2 Static closure

A closure that captures is a value of a **concrete type** — an anonymous aggregate
of the captures together with the known function. Its environment is stored where
declared (by §2/§5; on the stack or a region by default — visible, no hidden
allocation). A call to a statically-known closure is direct or inlinable. Generic
code that takes a callable **monomorphizes** over the concrete closure type
(Type System / Comptime chapters); there is no `dyn` indirection unless requested.

#### 6.3 Type-erased (`dyn`) closure

When heterogeneous callables must be stored together, a **type-erased** value is
used: a fat value of a code pointer plus an environment pointer. Its environment
MUST live in **explicit** storage (a region, arena, or allocation, §5) — **never**
a hidden box (I3). The call is indirect (a visible cost). Its **type** is
`dyn fn(T…) -> R` (Functions §1.6, FN-11); it is **constructed** by the prelude
`dyn_over(ptr(mut store))` over a **named place** `store` holding a static closure
(§6.2), and **called** `d(args)` (indirect through `code`, passing `env`). The `dyn`
value **borrows** its env storage and is bounded by the **escape rule** (§5.3.1) — it
MUST NOT outlive that storage (a stack env → scoped to the frame; an arena/allocation
env → valid as long as that region).

#### 6.4 Capture mode

Captures are **by value by default** (a copy is placed in the environment, so the
closure is independent). Capturing a **reference** explicitly (taking `ptr` of
a place in the capture) makes the closure **scoped** (§5.3): it MUST NOT escape.
When a closure is supplied as an `@require` predicate (Type System §8.1), the narrower
contract applies: every capture MUST be comptime-known and is specialized away. A
runtime value or reference capture is rejected, so that predicate call carries no
runtime environment.

---

### 7. Syntax (grammar fragments)

The forms this chapter introduces are given here as **grammar fragments**; the
**consolidated EBNF** and lexical rules are assembled in the Grammar chapter, and
these fragments conform to the unified declaration model (Declarations chapter).
Nonterminals not defined here (`ident`, `expr`, `type-expr`, `place`, `block`,
`binding`, `string`, `int`) come from those chapters. The `@`-layout attributes
(`@repr(T)` / `@packed` / `@align` / `@offset` / `@endian(...)` / `@niche(...)`) are
specified in the Type-System chapter; the storage/placement attributes (`@section`
included, §2.3) appear below.

```ebnf
(* mutability — bare prefix keyword, on a binding or a struct field (§3) *)
mut-binding   ::= "mut" binding                       (* mut x : u64 = 0 *)
(* a field's `mut` is part of the struct field grammar (Type-System chapter) *)

(* storage / lifetime specifiers — @-attributes on a binding (§2.4) *)
storage-attr  ::= "@reg" [ "(" ident ")" ]            (* @reg  |  @reg(rax) *)
                | "@stack"
                | "@static"
                | "@section" "(" string ")"           (* @section(".boot") *)
                | "@scoped"                            (* non-allocating, default lifetime *)
                | "@alloc" "(" expr ")"                (* @alloc(arena) — allocator value selects mechanism *)

(* pointer types (§4.1) — pointee-mutability is inside the constructor *)
pointer-type  ::= "ptr" "(" [ "mut" ] type-expr ")"   (* ptr(T) | ptr(mut T) *)
(* absence is the ordinary type Option(ptr(T)) — no null literal (§4.2) *)

(* cleanup and linearity (§5.8) — a bare keyword statement *)
defer-stmt    ::= "defer" ( expr | block )            (* LIFO; runs on normal exits *)
```

The **address operations** (§4.3) and `forget` (§5.9) add **no grammar productions**:
each is a call to a prelude identifier (`postfix-expr` in the Grammar chapter). The
following are **form schemas** (operand shapes and meaning), not productions:

```text
ptr( place )      — take an address; ≡ place.ptr()
deref( expr )     — load, or a place when assigned; ≡ expr.deref()
forget( expr )    — discharge linearity without release
```

Normative notes:

- `ptr` and `deref` are flat prelude word-functions (§4.3); a bare
  field access `x.f` is distinct from the call `x.ptr()`, so they do not occupy
  field-name space.
- `deref(p)` is a **place**: it loads in value context and stores as the left
  operand of an assignment, with write permission equal to `p`'s pointee mutability
  (§3.3, §4.3).
- A pointer's alignment override uses the Type-System `@align(N)` attribute (§4.1);
  a static binding's section uses `@section` (§2.3), whose default derivation is
  §2.3.

---

### 8. Conformance (normative summary)

A conforming implementation MUST:

1. classify expressions as place- or value-expressions and reject a store to a
   value-expression (§1.6);
2. place bindings by storage class per §2, derive static sections per §2.3, and
   document and obey a deterministic placement strategy per §2.5;
3. forbid taking the address of a register-resident place under `no_abstractions`,
   and otherwise spill by a documented rule (§2.6);
4. enforce mutability — immutable by default, the AND rule, immutable-field
   definite assignment — at compile time (§3);
5. represent pointers as non-null `ptr(T)`/`ptr(mut T)`, represent absence
   as a niche-`0` `Option(ptr(T))`, enforce monotonic permissions, and confine
   pointer arithmetic and integer→pointer to an `unchecked` grant / assembly (§4);
6. guarantee temporal safety in checked code via the §5 mechanisms (statically for
   scoped, by access check for handles, by generation trap for generational), reject
   `manual`-mechanism reference creation/use outside an `unchecked` grant (forbidden
   under `no_unchecked`), and never silently convert between mechanism representations (§5.6);
7. classify owning values structurally (the `@owning` marker plus contagion), treat an
   `in`-pass / `return` / `out`-fill of an owning value as a **move** that invalidates the
   source, and enforce strict linearity of owning values on every normal exit path (no
   leak, no double-consume, no use-after-move; `@static` / `-> Never` carry no obligation),
   running `defer` actions LIFO on normal exits and not on abort, and provide `forget`
   (§5.8–5.9); reject an owning immediate underlying `U` for `@require`, whose predicate
   needs a by-value copy while preserving the result (Type System §8.1);
8. compile non-capturing functions to code pointers (type `fn(T…)->R`, Functions §1.5),
   static closures to monomorphized concrete-type values, and `dyn` closures (type
   `dyn fn(T…)->R`, constructed by `dyn_over` over a named place) to `{code, env}` fat
   values with explicit environment storage, bounding each `dyn` by its env storage's
   extent via the escape rule; for an `@require` predicate, accept only comptime-known
   captures specialized away and reject a runtime environment (§6; §5.3.1; Type System
   §8.1; FN-11);
9. introduce **no** hidden control flow, allocation, copy, or run-time type
   information in any of the above (I3).

Where this chapter says an operation lowers to a machine effect, the exact
per-target emission is normative in the Codegen and Assembly-Correspondence
chapters; the meaning and the required effect are normative here.
