# Alatyr Language Specification

## Chapter — Functions, Parameters, and ABI

> **Status — draft (under review).** This chapter is a draft and is not yet
> accepted; the specification as a whole is under development. See the status note
> at the top of the Overview chapter.

This chapter defines **functions** as values, **parameter directions** and how
results are produced by one output-place model (via anonymous `-> T` or named `out`),
**passing modes** and their ABI
lowering, **defaults** and **named arguments**, **variadics**, and the **ABI** model.
It builds on the Memory chapter (places, mutability, pointers, function-value
representations §6), the Type-System chapter (types are values), the
Comptime-and-Generics chapter (comptime parameters, monomorphization), and the
Declarations chapter (the `fn` introducer).

Requirement keywords (**MUST**, **MUST NOT**, **SHOULD**, **MAY**) follow RFC 2119
(Overview §6). This chapter gives the **concrete syntactic forms** it introduces as
grammar fragments (§9); the *consolidated* EBNF is assembled in the Grammar chapter.

---

### 1. Functions are values

#### 1.1 The `fn` introducer

A function is a **value**, written with the `fn(…) { … }` introducer (Declarations
§1.3) and bound by an ordinary declaration:

```
add := fn(a : u64, b : u64) -> u64 { a + b }
```

There is no `function` declaration keyword; `fn(…){…}` is a value-expression like
`struct{…}` (Declarations §1.3).

#### 1.2 Representations

A function value's runtime representation is defined in the Memory chapter §6
and is **chosen by what it captures**, never hidden:

- **non-capturing** → a **code pointer** (zero cost, indirect call);
- **capturing, statically known** → a **static closure** (a concrete anonymous
  aggregate of captures + the function), **monomorphized**, direct/inlinable;
- **type-erased** → a **`dyn`** fat value (code pointer + explicit environment
  pointer), indirect call, environment in explicit storage — never a hidden box (I3)
  (the type `dyn fn(T…)->R`, built by `dyn_over` over explicit storage — §1.6, FN-11).

#### 1.3 Generic functions

A function with **comptime parameters** is generic and **monomorphized** per
comptime-argument set (Comptime §3): zero-cost, no runtime dispatch. A `T : type`
parameter is comptime by nature (no `comptime` marker needed); a non-`type` comptime
parameter is marked (`comptime N : usize`) (Comptime §1.3).

#### 1.4 Overloading by signature (FN-7)

A name MAY be bound to **more than one function value** in the same scope when the
bindings differ in their **parameter signature** — the ordered list of value-parameter
types. The bindings form an **overload set**. This is the single exception to the
one-binding-per-name rule (Declarations §6.2): rebinding a name to a **non-function**,
or to a function with the **same** parameter signature, remains a redefinition error.

A call `f(args)` resolves to the overload whose value parameters **accept the argument
types** — **exactly one** must match (zero → unresolved-call error; more than one →
ambiguous-call error). Resolution is by the **argument types only**; the result / `out`
types do not participate (a call site is resolved without knowing its result type). UFCS
method dispatch `recv.m(args)` (MOD-3) is the same resolution keyed on the receiver — the
first value parameter — and composes with it. When that first parameter is a **scoped
reference** `ptr([mut] R)` and the receiver has type `R`, the receiver is passed **by
its address** (**auto-ref**, MOD-3): `recv.m(args)` ≡ `m(ptr(recv), args)` — the way a
method **borrows** its receiver (reads it without consuming, for an owning receiver; Memory
§5.9). A by-value first parameter is matched and passed unchanged. The **dual** (**auto-deref**,
OP-5) applies when the receiver is already a pointer `ptr([mut] R)` and the resolved method's
first parameter is the pointee `R`: the receiver is dereferenced, `recv.m(args)` ≡
`m(deref(recv), args)`. The coercion is resolved **exact match → auto-ref → auto-deref**, one
level (Type System §4.5), so a method whose slot is `ptr(R)` matches a pointer receiver
**as-is** and is preferred over reaching it by auto-deref.

Each overload lowers to a **distinct symbol** — its name **mangled by the parameter
signature** — transparently (I1/I3), like a monomorphized generic instance (Comptime §3);
there is **no runtime dispatch** (I2). Overloading exists for **distinct concrete
functions that share a role**: the Iterator protocol's per-type `iter` / `next`
(Stdlib §2.4), a type's own structural `eq` / `hash`.

A name MAY combine **one generic definition** with concrete overloads — the
**generic-default + concrete-override** pattern the structural derives rely on (`eq` / `lt` /
`hash`: a single generic structural default `fn(T : type, in a : T, in b : T)`, plus a type's
own concrete `eq(in a : K, in b : K)`; Stdlib §2.6). Resolution prefers the **more specific**
definition: a call whose arguments are **exactly accepted by a concrete overload** resolves to
that concrete symbol; **only when no concrete overload matches** does the **generic** apply
(monomorphized at the inferred type). The generic is thus the **fallback default**, a concrete
overload the **per-type override** — `eq(K, …)` / `a == b` on a `K` with a concrete `eq` calls
the override, on any other type the structural default. (Among the **concrete** overloads the
exactly-one-match rule still holds; the generic never causes ambiguity — it is consulted only
on concrete-miss. This is the sole mixing allowed: a name still may not bind two functions of
the *same* concrete signature, nor two generic definitions.)

*(Rejected: overloading on the **result** type — a call would not be resolvable in
isolation, breaking left-to-right typing; an explicit per-type **dispatch table** value —
redundant with the symbol the compiler already mangles, and not zero-cost at the call
site. Generic monomorphization already mints per-instance symbols; overloading reuses
that mechanism for distinct concrete bodies rather than introducing a parallel one, OP-1.)*

#### 1.5 The function-value type (FN-10)

The **type** of a **non-capturing** function value — a top-level function or a
non-capturing lambda — is written, in **type position**,

```
fn(T0, T1, …) -> R
```

the **bodyless** signature: `fn`, a parenthesized list of parameter **types**, and an
optional `-> R` result. It is distinguished from the `fn(…){…}` **value**-expression
(§1.1) by the **absence of a `{ … }` block**, and from a type-returning generic function
by appearing where a **type** is expected. Its **representation is one machine word — a
code address** (Memory §1.2 / §6.1): a value of this type is **bound, passed, returned,
stored** (in a struct, array, or tuple field), and **called** `f(args)` — one **indirect
call**, with no environment and no allocation (I2/I3). Taking a top-level function or a
non-capturing lambda yields such a value.

- A parameter entry is an optional **direction** (`in` — the default — / `out` / `in out`,
  §2.1) followed by a **type**, with **no name**: `fn(u64, u64) -> u64` is the all-`in`
  common case, `fn(in out [u8]) -> Status` an in-place one, and `fn(u64, out u64)` types a
  function with a named-`out` result. Two function values share a type only when their
  parameter directions and types **and** their result match — directions drive the ABI
  (§4), so they are part of the type.
- `-> R` is **optional**: present, the function projects an anonymous output `R`; absent,
  it is a **procedure** (no result), mirroring the optional `->` on a `fn-sig` (§3.5). A
  named-`out` result is typed by an `out`-directed parameter entry.
- `fn(T…)->R` is the concrete **code-pointer type**; it is **not** the generic *callable*
  constraint (§7 / FN-6 — a comptime structural predicate that also admits **static
  closures** for monomorphization). The two coexist: this type stores a homogeneous code
  pointer, the predicate abstracts over any callable. Storing a heterogeneous set of
  **capturing** closures under one type is the type-erased **`dyn`** (§1.6; FN-11), built
  over this signature.

**Required-type predicates.** The Type System §8.1 contract
`@require(pred) U` uses that generic callable-interface rule, requiring the post-
specialization interface `fn(in U) -> bool`. Whether the source is a named binding or an
inline function literal is orthogonal to representation: a non-capturing predicate is a
value of the concrete function type above, while one with comptime-known captures is a
static closure whose concrete type is monomorphized. Both are called with the predicate's
declared conventional ABI. Runtime captures and `dyn fn` are rejected for this use, so
the call has no hidden environment argument.

#### 1.6 Type-erased `dyn` closures (FN-11)

When **different capturing closures** must share **one** runtime type — stored together,
dispatched dynamically — the type-erased **`dyn`** form is used. Its type is written

```
dyn fn(T0, T1, …) -> R
```

the **`dyn` keyword** prefixing a function-value type (§1.5). Its representation is a
**two-word fat pair** `{code, env}` — a code pointer plus a pointer to the captured
environment (Memory §6.3). Visible cost: **two words + one indirect call**. `dyn` is a
**keyword type-constructor** (it switches the representation from a thin code pointer to the
fat pair), **not** an `@`-attribute (which is compile-time only, never a representation).

- **Explicit storage (I3).** The environment lives in **explicit, user-provided storage** —
  a user place, a region/arena slot, or an allocation — that the `dyn` value **borrows**;
  **never** a hidden heap box (this differs deliberately from a `Box<dyn>`-style hidden
  allocation). The `dyn` value is bounded by that storage's extent and checked by the
  **escape rule** (Memory §5.3.1): it MUST NOT outlive its env storage. A **stack** env
  makes the `dyn` scoped to the frame; an **arena/allocation** env lets it live as long as
  that region (and be stored where the region outlives). No new lifetime discipline is added.
- **Construction — `dyn_over`.** The prelude word-function `dyn_over(ptr(mut store))` (a
  builtin; the name echoes `arena_over`, Memory §5.4) pairs a **static closure** held in a
  **named place** `store` (Memory §6.2) with that place as its environment, yielding
  `{code = <the adapter monomorphized for `store`'s type>, env = ptr(store)}`, typed
  `dyn <store's call signature>` and assignable to a compatible `dyn fn(T…)->R`. The storage
  is **explicit in the source** (`store` and its address), so its location and lifetime are
  recoverable from the package (I3). The `code` adapter is a compiler-synthesized monomorphic
  wrapper (a documented lowering, I1); the env-pointer mutability follows the `ptr(mut …)` /
  `ptr(…)` form.
- **Call.** `d(args)` — the ordinary postfix call (§3.2) — lowers, for a `dyn`-typed callee,
  to an **indirect call through `code`** passing `env` as the leading environment pointer to
  the adapter. Nothing is hidden (I3).

```
store := fn(x : u64) -> u64 { x + base }          # a static closure; explicit env storage (a named place)
d : dyn fn(u64) -> u64 = dyn_over(ptr(mut store))  # erase: {code, env = ptr(store)}; d borrows `store`
r := d(3)                                          # indirect call through the fat pair
```

---

### 2. Parameters and directions

#### 2.1 The three directions

Every parameter has a **direction**:

| Direction | Meaning |
|-----------|---------|
| `in` (default) | an **input**; the callee reads it |
| `out` | a **place the callee writes** the result into |
| `in out` | the **caller's place**, read **and** written **in place** |

An unmarked parameter is `in`. A direction composes the existing axes: a parameter
is a **place** (Memory §1) with a **mutability** (Memory §3), passed by an **ABI**
lowering (§4, §6); `out`/`in out` denote the caller's place by reference.

#### 2.2 Why directions instead of a return type

Multiple results, large in-place updates, and "fill the caller's buffer" are all
expressed **explicitly** through `out` / `in out` — there is **no hidden
return-via-pointer** (I1/I3). Explicit by-reference input is just an ordinary
pointer parameter (`in p : ptr(T)`); there is **no** separate by-reference *mode*
modifier (§4.1).

#### 2.3 Argument binding at the call

What a parameter takes at a call site follows from its **direction**:

- an **`in`** parameter is bound by a **value argument** — any value-expression
  (Memory §1.6) — positional or named (§5), or by its default (§5.1);
- an **`in out`** parameter is bound by a **place argument** — a place-expression
  the callee reads and writes **in place**. It is **required**: it can be neither
  projected nor defaulted, because there is an existing value to update;
- an **`out`** parameter is bound **either** by a **place argument** (the callee
  writes the result into the caller's place) **or** by **projection** (§3.2) — the
  `out` is omitted from the argument list and becomes (part of) the call's value in
  expression position.

A **place argument** (for `out` / `in out`) MUST be a **mutable** place (Memory §3)
of the matching type; it may be passed positionally or named, exactly like a value
argument (§5). Each `out` / `in out` parameter is bound **exactly once** — by a place
argument or, for `out` only, by projection — never both. If **any** `out` is
projected, the call MUST appear in **expression / binding position** so its value is
consumed; if **every** `out` / `in out` is bound by a place (or there are none), the
call MAY stand as a **statement**.

```
r := add(2, 3)                 # add(in, in) -> u64: the result projects → r
(q, rem) := divmod(10, 3)      # two named outs → destructured projection
divmod(10, 3, qp, remp)        # same fn: the two outs bound to mutable places qp, remp
swap(a, b)                     # swap(in out x, in out y): a, b are mutable places, updated in place
fill(buf)                      # fill(out buf : [u8]): buf is a mutable place, written in place
```

Whether an argument must be a **value** or a **place** is set by the matched
parameter's **direction**, not by the argument's syntax — the call grammar is
uniform (§9), and this is a **static-semantic** rule. Passing a non-place (e.g. a
literal) to an `out` / `in out`, or an insufficiently-mutable place, is ill-formed.

---

### 3. Results — one output-place model

#### 3.1 A result is an anonymous projected output or named `out`

A function declares its result with the **same output-place mechanism** in one of two
surface forms, **never both**:

- a **single anonymous projected output** as **`-> T`** after the parameter list
  (`add := fn(a : u64, b : u64) -> u64 { a + b }`). `->` is the convenient anonymous
  surface for the one-result case: body-expression and `return expr` fill this output,
  and calls project it into expression position. `->` **excludes** named `out`-results
  but **composes** with `in` / `in out` (e.g. `fn(in out buf : [u8]) -> Status`), so the
  parameter list is **uniform** — every entry is `name : type`;
- one or more **named `out r : T`** parameters, the same output-place mechanism with
  source-visible output names, written in the body by assignment.

The result is therefore **visible in the signature** either way; there is no silent
contract change (a footgun avoided). There is **no** anonymous `out T` form — the
anonymous surface is `-> T`, and named/multiple results use `out`.

#### 3.2 Expression projection

When a call appears in **expression position**, its result is **projected** into that
position:

- a **`-> T`** anonymous output (or a **single named `out`**) projects to a **value**:
  `r := add(2, 3)`;
- **several** named `out`s project to a **destructuring** `(t₁, …, tₙ) := call`: the target
  count MUST equal the call's `out` count, and the **i-th target binds the i-th `out` in
  signature order** (positional; Grammar §3.2) — e.g. `(q, rem) := divmod(10, 3)`;
- composition works through projection: `f(g(x))`.

#### 3.3 Body-expression sugar

For a **`-> T`** anonymous output, a body that **is** an expression — `{ expr }` — fills it (sugar
for `{ return expr }`). With **named** `out`s, the results are written in the body by
assignment. A function with **no** result whose body ends in a value-expression is
**ill-formed** (no silent result).

#### 3.4 `return`

- With a **`-> T`** anonymous output, `return expr` fills it and exits — for early returns / guard
  clauses, on a par with the fall-through final expression.
- With **named** `out`s, `return` carries **no** value (the `out`s are written in the
  body); a bare `return` just exits.
- With **no** result, `return expr` is **ill-formed** (nothing to fill).

#### 3.5 Procedures

A function with **no result** (no `-> T`, no `out`) yields nothing; using such a call in
value position is ill-formed. `defer` actions run on **every** normal exit, including each
`return` (Memory §5.8).

---

### 4. Passing modes and ABI lowering

#### 4.1 The value model is logically by-value

A parameter is logically a **value**; the **passing mode** is a **documented,
per-target ABI lowering** (I1, not hidden):

- `in`, **small** → **by value** in register(s);
- `in`, **large aggregate** → **by reference to a copy** (so the callee cannot mutate
  the caller's value; the copy's cost is visible, I1);
- `out` / `in out` → **by reference to the caller's place**; a **small `out`** is
  returned in register(s) and projected (§3.2).

For an **owning** value (Memory §5.9, an `@owning` type), there is **no copy** — it is
non-copyable — so passing it `in` is a **move**: ownership transfers into the callee and
the source is invalidated (Memory §5.9 / MEM-2). The lowering is unchanged (small → by
value in register(s); large → by reference); only the static *consume* semantics differ.
`in out` remains a mutable borrow and `ptr` a scoped borrow — neither consumes.

A checked `@require` constructor predicate is one concrete use of ordinary `in`
passing: the preserved constructor result and the predicate's logical by-value argument
are distinct, with small/large aggregate lowering fixed by the ABI appendix §6. Because
an owning `in U` would move rather than copy, v1 rejects an owning immediate underlying
`U` for `@require` (Type System §8.1).

There is **no** by-reference *mode* modifier: to pass an explicit pointer, declare a
pointer parameter (`in p : ptr(T)`, §2.2).

A raw union is an aggregate for every passing mode. Its selected ABI classifies the
**declared union type and complete layout**, never the compile-time active-member fact;
small register carriers or a large by-reference copy preserve all `U.size()` object bytes,
including padding. Active-member facts are not part of a signature or ABI: an incoming
parameter and a normally returned union are active-unknown in checked code (Type System
§6.3/TYP-11).

#### 4.2 Predictability

The size threshold and the specific registers are the **target ABI's** (§6); the
lowering rule is **documented and deterministic**, so the cost of each call is
predictable (I1). No copies occur beyond this documented lowering (I3).

---

### 5. Defaults and named arguments

#### 5.1 Defaults

An `in` parameter MAY carry a **default** (`in x : u64 = 0`): when the argument is
omitted at the call, the default expression is supplied, and it is **visible in the
signature**. `out` and `in out` parameters have **no** defaults (a result place must
be provided).

#### 5.2 Named arguments

Arguments MAY be **named** at the call: `f(x = 1, y = 2)`. This is the **same `=`
label mechanism** as struct construction (Type System §9.3): order-independent and
combinable with defaults (omit an argument that has a default; name the rest).

#### 5.3 Mixing positional and named

Positional and named arguments may be combined per the Grammar rules (positional
arguments bind by position, then named arguments bind by name); an argument MUST NOT
be given both positionally and by name.

#### 5.4 Evaluation order

Argument expressions are evaluated **left-to-right in the order written at the call
site**, independently of any named→positional reordering (§5.3) — that reordering fixes
only which parameter each argument binds to, never **when** it is evaluated. There is
**no** implicit reordering of side effects (I1/I3); a different evaluation order is
obtained explicitly, by binding arguments to temporaries in the desired order
beforehand.

#### 5.5 Allocator elision — the ambient allocator (MEM-5)

An **allocator parameter** — an `in` parameter whose type is the library protocol
**`alloc`** (Memory §5; concretely `(A : alloc, in a : ptr(mut A))`) — gains, beyond
ordinary defaults, the right to be **elided**: when omitted at the call, it is filled from
the caller's **ambient allocator** rather than from a `§5.1` default expression. The two
mechanisms are distinct and ordered — an elided allocator resolves as: **(1)** the nearest
enclosing `alloc::with` scope (Memory §5), **(2)** the enclosing function's own `alloc`
parameter (within a function body an `alloc` parameter **is** the ambient), **(3)** a §5.1
default if the parameter declares one, otherwise **(4)** a **compile error** ("allocator
required"). There is **no** silent fallback to a hidden heap (I3). Because the parameter is
**declared in the signature** and the ambient is a **named binding in scope**, the elision
is determinate from the closed package (CG-11), not hidden.

A §5.1 default on an allocator parameter is evaluated as a **call-site temporary** (§5.1);
it is therefore sound only when the allocations it backs **do not escape** the call (the
escape checker enforces this, Memory §5.4/§5.6). An **escaping** producer (a container
constructor) therefore declares a *required* allocator parameter and relies on `alloc::with`
above it, not on a create-fresh default.

**UFCS interaction.** An elided allocator parameter does **not** participate in receiver
binding: `recv.f(…)` binds `recv` to the first **written** (non-elided) parameter, and the
allocator is supplied by the ambient; the fully explicit form is the free call `f(a, …)`.

---

### 6. ABI

#### 6.1 An ABI is a comptime value

An **ABI** is a **comptime value** describing a calling convention: which registers
carry arguments and results, who saves what, and alignment. Being a value, it is
nameable, passable, and selectable per site (Comptime §1).

#### 6.2 Default and override

The **default** ABI comes from the **target / environment** fixed at configuration. A
per-site override uses the **`@abi(value)`** attribute (decisions TYP-7/SYN-7) on a
**function** (the override site is the function declaration, not the call — `@` never
wraps a call expression, CG-6): it sets the **calling convention** (`@abi(c)`,
`@abi(syscall)`). A call uses the calling convention carried by the **callee's type**.
`@abi` is a calling-convention attribute only; a **type's data layout** is fixed by the
default plus the closed layout levers (Type System §8), **not** by `@abi`.

#### 6.3 Provided and user-defined ABIs

The prelude provides `c` / `sysv` / `syscall` / … as ABI **values**. A user-defined
convention is an ordinary **`Abi`** value (the appendix §1 schema), built by **struct
construction** like any other struct value: `MyConv := Abi(name = "...",
int_arg_regs = [...], ...)` (an ordinary value declaration, Declarations §1.3).

#### 6.4 Naked functions — and the process entry

`@abi(naked)` declares a function with **no compiler-generated frame, prologue,
epilogue, or return sequence** — the body is exactly what is emitted. It is for entry
points, interrupt handlers, and trampolines, where the programmer supplies the
machine-level frame/return directly (typically with assembly-correspondence
constructs).

**`@abi(entry)`** is the third selector alongside a convention and `naked` (ABI appendix
§3.1; FN-12): it marks **the process entry** and emits the **per-target entry
prologue/epilogue** (ABI appendix §3.3) — capture the platform's entry state, record it
for `args`/`env` in the documented `alatyr_entry_state` static (emitted only when those
are reachable), establish a frame, call the body, terminate the process with its result.
The declared form is `fn()` or `fn() -> i32` — **no parameters**: the entry state is a
per-target fact the prologue captures, so one source form is portable across containers.
It is **not applicable** on `Os.none`, `Container.com`, under `startup = libc`, or in a
build whose `kind` is neither `executable` nor `object`; `@abi(naked)` is the entry form
where no platform contract exists. Which declaration *is* the entry is named by the
manifest's `Target.entry` (Tooling §2.2) — independently of which selector it carries, and
an `@abi(entry)` function it does not name is ill-formed (ABI appendix §3.3).

---

### 7. Variadics

#### 7.1 Comptime variadic — the primary form

Spelled as a trailing parameter **`name : ...`** (a bare rest, **no** element type —
comptime by nature, like a `T : type` parameter, §9). Trailing arguments are collected
into a **comptime tuple**: the function is **monomorphized** over their types, so it is
**type-safe and heterogeneous**, and the body walks them with `comptime for`
(Comptime §8.3). This is the primary, checked-code variadic.

**Format-template shape (the specified trigger).** When a comptime-variadic's **first
fixed parameter is `str`** and its argument is a **comptime string literal**, that literal
is a `{}`-**template**: it is split on `{}` holes and **interleaved** with the remaining
(pack) arguments — the pack the body walks becomes `[seg0, arg0, seg1, …, segN]`, each
piece rendered by the `Display`/scalar layer (Stdlib §2.7). This is the **single, normative
trigger** that `format`/`print` use and that **any** user-defined comptime-variadic with a
leading `str` parameter inherits — a documented rule, **not** a hidden reinterpretation
(I3). `{{`/`}}` escape a literal brace; a lone `{`/`}`, and a hole-count ≠ argument-count
mismatch, are **comptime errors**; a **non-literal** first argument is **not** a template
(outside v1). A comptime-variadic that must take a leading `str` *without* templating
therefore takes it as a non-first parameter (or a runtime, non-literal value).

#### 7.2 Slice variadic — the homogeneous case

For a **homogeneous** variadic, a trailing parameter **`name : ...T`** (a rest with a
concrete element type `T`) receives the arguments gathered into one runtime slice `[T]`.

#### 7.3 C variadic — FFI only

A **C variadic** exists only for FFI: spelled as the bare trailing-rest **`name : ...`**
(no element type) in an **`@abi(c)`** function — the `@abi(c)` context is what selects
the C form over the comptime tuple (§7.1). It is writable only inside an **`unchecked`
grant** (decisions FN-5/CG-6), **forbidden** in checked code (I11). It is for interop with
C functions such as `printf`.

---

### 8. Lowering

- A **call** lowers per the function's ABI (§6): arguments are placed in
  registers/stack per the convention (§4), `out` / `in out` as references to the
  caller's places, and the result is projected (§3.2).
- A **generic** function is **monomorphized** (Comptime §3.4): each comptime-argument
  set yields one concrete function; no comptime parameter survives to run time.
- **Defaults** are supplied at the **call site** (the default expression is evaluated
  there, §5.1); **named arguments** are reordered to positional at compile time
  (§5.2).
- No call introduces hidden control flow, allocation, or copies beyond the documented
  ABI lowering (§4; I3); `defer` runs on all normal exits (§3.5).

---

### 9. Syntax (grammar fragments)

The forms this chapter introduces are given here as **grammar fragments**; the
**consolidated EBNF** is assembled in the Grammar chapter, and these fragments
conform to the declaration and expression grammar of the Declarations, Type-System,
and Comptime chapters. Nonterminals not defined here (`ident`, `type-expr`, `expr`,
`block`, `when-clause`, `arg-list`) come from those chapters.

```ebnf
(* a function value = a signature + a body (Declarations §1.3) *)
fn-value     ::= fn-sig block
fn-sig       ::= "fn" "(" [ param-list ] ")" [ "->" type-expr ] [ when-clause ]  (* -> T = single anonymous projected output (§3.1) *)

(* the function-value TYPE (§1.5, FN-10) — the bodyless signature in TYPE position, no names, no block *)
fn-type      ::= "fn" "(" [ fn-type-param { ("," | newline) fn-type-param } ] ")" [ "->" type-expr ]
fn-type-param ::= [ "in" | "out" | "in" "out" ] type-expr    (* direction (default in) + type, no name *)
(* the type-erased dyn closure TYPE (§1.6, FN-11) — `dyn` keyword over a fn-type; a {code, env} fat pair *)
dyn-type     ::= "dyn" fn-type
(* construction (§1.6) is the prelude word-function `dyn_over(ptr(mut store))` — a builtin, no grammar (OP-1);
   the call `d(args)` is the ordinary postfix call (§3.2). *)
param-list   ::= variadic-param
               | param { ("," | newline) param } [ ("," | newline) variadic-param ]  (* at most one variadic, last; may be the sole parameter (§7) *)
param        ::= in-param | place-param
in-param     ::= [ "in" ] [ "comptime" ] ident ":" type-expr [ "=" expr ]   (* default allowed (§5.1) *)
place-param  ::= ( "out" | "in" "out" ) ident ":" type-expr                 (* named out / in-place; no default; no comptime *)
variadic-param ::= [ "in" ] ident ":" "..." [ type-expr ]   (* trailing-rest (§7): `...T` slice / `...` comptime tuple / `...` under @abi(c) C-variadic *)
(* a T : type parameter is comptime by nature — no `comptime` marker (Comptime §1.3) *)
(* `-> T` (single anonymous projected output) excludes named `out`-results; it composes with `in`/`in out` *)

(* results — anonymous projected `-> T` or named `out`s in the signature (§3) *)
return-stmt  ::= "return" [ expr ]                    (* expr only with a single anonymous projected output (§3.4) *)

(* calls, defaults, named arguments (§3.2, §5) *)
call         ::= callee "(" [ call-args ] ")"
call-args    ::= arg { ("," | newline) arg }
arg          ::= expr                                 (* positional *)
               | ident "=" expr                       (* named — same `=` as struct ctor (§5.2) *)

(* ABI (§6) *)
abi-attr     ::= "@abi" "(" expr ")"                  (* on a function: calling convention; the value is an `Abi` (§6.3), `naked`, or `entry` *)
```

Normative notes:

- `fn-sig` is the **body-less signature**; a function value is `fn-sig` + a `block`.
  The same `fn-sig` is reused (without a body) by an `@extern` declaration (Modules
  §7).
- `fn-type` (§1.5) is the **type** of a non-capturing function value — the bodyless
  signature in **type position** (a `type-atom`, Grammar §3.3), carrying parameter
  **types** (with optional directions) but **no names** and **no block**. In a type
  position `fn(` unambiguously begins `fn-type` (`fn` is a keyword, so it is never a
  generic instantiation `ident(...)`); a `{ … }` block after the signature makes it the
  `fn-value` expression (§1.1) instead. Its value is one code-address word (Memory §1.2).
- A function's result uses one output-place model: **either** an anonymous projected `-> T`
  **or** named `out`s in the signature — never both (§3.1); `->` composes with
  `in`/`in out`. A `-> T` output is filled by a body-expression or `return expr`
  (§3.3–3.4). There is **no** anonymous `out T` form.
- The grammar admits a **default** (`= expr`) and the `comptime` marker **only** on
  an `in`-parameter, by construction; `out` / `in out` parameters can carry neither
  (§5.1). (No separate static rule is needed to forbid them.)
- The `arg` production is **uniform**: whether a given argument must be a
  **value-expression** (for `in`) or a **mutable place-expression** (for `out` /
  `in out`) is fixed by the matched parameter's **direction** as a static-semantic
  rule (§2.3), not by the grammar.
- A named argument uses the struct-construction `=` mechanism (§5.2), binds `in`
  values and `out`/`in out` places alike (§2.3), and MUST NOT also be passed
  positionally (§5.3).
- The **variadic** surface form is fixed in the Grammar chapter; semantically the
  three forms are comptime-tuple (primary), slice (homogeneous), and C-variadic
  (FFI, `unchecked`-grant only) (§7).
- `@abi(...)` selects the **calling convention** on a **function** declaration (a call
  uses the callee type's convention; `@` never wraps a call expression, CG-6); it is
  **not** a type-layout lever in v1 (type layout is the default plus the closed levers,
  Type System §8) (§6.2).

---

### 10. Conformance (normative summary)

A conforming implementation MUST:

1. treat a function as an ordinary value written with the `fn(…){…}` introducer, and
   represent it (code pointer / static closure / `dyn`) per Memory §6 with no
   hidden boxing (§1; I3); admit the **function-value type** `fn(T…) -> R` in type
   position — the bodyless signature (directions + types, no names) — as a one-word
   code-pointer type that a non-capturing value inhabits, storable and indirectly
   callable (§1.5; FN-10); admit the **type-erased** closure type `dyn fn(T…) -> R`
   (the `dyn` keyword over a fn-type) as a two-word `{code, env}` fat pair, construct it
   with the prelude `dyn_over(ptr(mut store))` over a **named place** holding a static
   closure (environment in explicit storage, never a hidden box), call it `d(args)` as an
   indirect call through `code` passing `env`, and bound its validity by its env storage's
   extent via the escape rule (§1.6; FN-11; Memory §5.3.1 / §6.3; I3);
2. monomorphize generic functions over comptime arguments, with no runtime dispatch
   and no surviving comptime parameter (§1.3, §8; Comptime §3);
3. support the parameter directions `in` (default) / `out` / `in out` with no
   separate by-reference mode (explicit by-ref is a pointer parameter); bind each
   `in` to a value argument (or its default), each `in out` to a required mutable
   place argument, and each `out` to **either** a mutable place argument **or**
   projection — exactly once, projected only in expression position — rejecting a
   non-place or insufficiently-mutable argument to `out`/`in out` (§2);
4. produce results through **either** anonymous projected `-> T` **or** named `out`s (never
   both; `->` composes with `in`/`in out`), project a `-> T` / single `out` to a value and
   several named `out`s to a destructuring, fill a `-> T` by a body-expression or
   `return expr`, and reject a value body / `return expr` where there is no result (§3);
5. lower passing modes by the documented per-target ABI (small `in` by value, large
   `in` by reference to a copy, `out`/`in out` by reference to the caller's place),
   with no copies beyond that lowering; for a checked `@require` predicate, preserve the
   constructor result while supplying one distinct logical by-value `in U` argument, and
   reject an owning immediate `U` rather than copying or consuming it (§4; Type System
   §8.1; I1/I3);
6. permit defaults only on `in` parameters (visible in the signature, evaluated at
   the call site) and named arguments via the struct-construction `=` mechanism,
   rejecting an argument given both positionally and by name (§5);
7. treat an ABI as a comptime value, default it from the target, allow per-site
   `@abi(value)` selecting the calling convention on a function, provide the prelude
   ABI values and accept a user-defined `Abi(...)` value, and support `@abi(naked)` and
   `@abi(entry)` (§6; ABI appendix §3.1, §3.3);
8. provide comptime-tuple variadics (primary), slice variadics (homogeneous), and
   C-variadics only under `@abi(c)` within an `unchecked` grant — forbidden in
   checked code (§7; I11);
9. run `defer` actions on every normal exit including `return`, and introduce no
   hidden control flow, allocation, or copies in calls (§8; I3);
10. permit a name to bind several function values that differ in **parameter
    signature** (an overload set), resolve each call to the unique overload whose value
    parameters accept the argument types — rejecting a zero-match (unresolved) or
    multi-match (ambiguous) call — and lower each overload to a distinct,
    signature-mangled symbol with no runtime dispatch; treat a same-signature or
    function-vs-non-function rebinding as a redefinition error; and allow **one generic
    definition** to coexist with concrete overloads of the name (the derive
    `generic-default + concrete-override` pattern), preferring an exactly-matching
    **concrete** overload and falling back to the **generic** only on concrete-miss
    (§1.4; FN-7; I1/I2/I3).
