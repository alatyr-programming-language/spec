# Alatyr Language Specification

## Chapter — Declarations and Bindings

> **Status — draft (under review).** This chapter is a draft and is not yet
> accepted; the specification as a whole is under development. See the status note
> at the top of the Overview chapter.

This chapter defines how names are **introduced** and **bound**: the single
declaration form, the modifiers that apply to it, type annotation and inference,
initialization and definite assignment, and the **scope**, **shadowing**, and
**order** rules. It builds on the Memory chapter (value / place / binding /
address, mutability), the Type-System chapter (a type is a value), and the
Comptime-and-Generics chapter (the `comptime` marker).

Requirement keywords (**MUST**, **MUST NOT**, **SHOULD**, **MAY**) follow RFC 2119
(Overview §6). This chapter gives the **concrete syntactic forms** it introduces as
grammar fragments (§9); the *consolidated* EBNF and lexical rules are assembled in
the Grammar chapter.

---

### 1. The unified declaration form

#### 1.1 One form

There is **one** declaration form:

> `[ modifiers ] name : type = value`

with two variations:

- `name := value` — declare and **infer** the type from `value` (§3.2);
- `name : type` — declare **without initialization** (§4).

A declaration **binds a name to a place** holding the value (Memory §1.3).

#### 1.2 Declaration versus assignment

`name := value` and the typed forms **introduce** a new binding. `place = value` is
an **assignment** — a store into an **existing** place (Memory §1.6) — **not**
a declaration. The token `:=` (introduce) versus `=` (store) distinguishes the two
syntactically (Go-like): a bare `name = value` where `name` is not already bound is
ill-formed, and `name := value` where `name` is already bound in the same scope is a
redeclaration error (§6.2).

#### 1.3 No separate declaration keywords

There are **no** declaration keywords (`let` / `const` / `fn` / `struct` / `type`).
Because types and functions are **values** (Type System §1), the right-hand side of
a declaration is an ordinary **value-expression**, and the **value-introducers** are
how type/function/module values are written:

| Introducer | Produces | Detailed in |
|------------|----------|-------------|
| `struct { … }` / `enum { … }` / `union { … }` | a type | Type System §5–§6 |
| `fn(…) { … }` | a function value (result via `-> T` or named `out`; a **generic type** is `fn(…) -> type`, Comptime §3) | Functions |
| `mod { … }` | a module (namespace value) | Modules |

`brand(…)` is **not** an introducer — it is an ordinary builtin **call expression** that
mints a branded type (`Meters := brand(u64)`; Type System §5.4, TYP-4), like any other
type-returning call.

A **generic type** is **not** a separate introducer — it is an ordinary type-returning
function `fn(…) -> type` whose body-expression (a type-literal) fills the projected
`-> type` output (SYN-6; there is **no** `type(…)` introducer — a clean kernel, OP-1/FN-1).

A declaration simply binds a name to whatever its RHS evaluates to. Examples:

```
Point   := struct { x : u64, y : u64 }
Meters  := brand(u64)
add     := fn(a : u64, b : u64) -> u64 { a + b }
Vec     := fn(T : type) -> type { struct { ptr : ptr(T), len : usize } }
comptime PI : f64 = 3.14159
mut count : u64 = 0
```

#### 1.4 What a name may denote

A bound name may denote a **runtime value/place** (a variable), a **comptime value**
(a constant, a type, a function value used at comptime), a **type**, a **function**,
or a **module**. Which it is follows from the RHS and the modifiers; there is no
separate syntactic category per kind.

---

### 2. Modifiers

A declaration MAY carry **bare prefix keyword** modifiers and **`@`-attributes**.
The three bare keywords are **distinct axes** (Modifier classification, decisions
TYP-7), combined per §2.1:

| Modifier | Axis | Default | Owned by |
|----------|------|---------|----------|
| `pub` | visibility | private | Modules chapter |
| `mut` | mutability (write permission of the place) | immutable | Memory §3 |
| `comptime` | phase | runtime | Comptime §1 |

#### 2.1 Combining the axes

The three are distinct axes, and `pub` combines freely with either of the others:

```
pub comptime MAX : u64 = 4096      # exported, compile-time-known, immutable
pub mut errno : i32 = 0            # exported, runtime, mutable static
```

The one constraint between them: **`comptime` entails immutable** (Comptime §1.4),
so **`comptime mut` is ill-formed** — a comptime value is already fixed, and the
combination is contradictory, not a "comptime-mutable" binding. Mutation *during*
comptime evaluation is a **separate** thing: an ordinary `mut` **local inside** a
comptime block or comptime function (Comptime §2.5), whose mutation is local and
erased — it is **not** a `comptime`-modified declaration.

#### 2.2 No `const`

There is **no** `const` keyword. A "constant" is a **`comptime` binding** (the phase
axis); immutability is the **default** (the mutability axis). The two are separate
(decisions FN-2): `comptime` means *known and fixed before run time*; `mut`/its
absence means *whether the place may be rewritten*.

#### 2.3 Attributes

`@`-attributes decorate not only a binding or field but **any annotatable entity**
(decisions SYN-7) — a binding, field, type, function, file, or **code construct** —
always in prefix position (the attribute precedes what it modifies). They cover
storage and lifetime (`@reg` / `@stack` / `@static` / `@scoped` / `@section` / `@alloc`,
Memory §2.3–2.4), layout (`@repr` / `@packed` / `@align` / `@offset` / `@endian(...)` /
`@niche(...)`, Type System §8), calling convention (`@abi`, on a function declaration —
Functions §6), **inlining** (`@inline`, on a function declaration — substituted at its
direct call sites, Codegen §3.5 / CG-10), linkage (`@extern` / `@export`, Modules), **linearity** (`@owning`, which
marks a type as a root owning value — Memory §5.9), and **control
flow** (`@label(name)`, which names a code point / loop / block as a `jmp` /
`break` / `continue` target — Control Flow §2.1). Their exact positions are fixed in
the Grammar chapter; their semantics are owned by those chapters. (Attributes are the
**open** representation/identity set; the bare keywords `pub`/`mut`/`comptime` are the
**fixed** role axes — decisions TYP-7.)

The v1 attributes group into a fixed set of **families** by *what they affect* — the
canonical organization for discoverability and diagnostics (the syntax stays a **flat**
`@name`; the family is **not** written, SYN-7). The table below lists each by family,
**target kind**, and **validation stage** (form in Grammar §3.11; semantics in the cited
chapter; diagnostic stages per Tooling §5):

| Family | Attribute(s) | Attaches to | Validated at | Semantics |
|---|---|---|---|---|
| **storage** | `@reg`/`@reg(r)` · `@stack` · `@static` · `@scoped` · `@section("…")` · `@alloc(v)` | binding (a place) | Semantic | Memory §2.3–2.4 |
| **layout** | `@repr(T)` · `@align(N)` · `@packed` · `@offset(N)` · `@endian(…)` · `@niche(…)` | type / field | Semantic → Codegen (lowering §5.1) | Type System §8 |
| **contract** | `@require(pred)` | type | Semantic (comptime) | Type System §8 (`require`) |
| **contract** | `@owning` | type (declaration) | Semantic | Memory §5.9 — marks a **root owning value** (strict linearity: consumed exactly once; non-copyable) (MEM-2) |
| **contract** | `@convert` | function (one parameter) | Semantic (comptime) | Type System §4.6 — conversion-constructor for `T(v)` (TYP-6) |
| **abi** | `@abi(value)` | function | Semantic | Functions §6; ABI appendix |
| **codegen** | `@inline` | function | Semantic | Codegen §3.5 — substituted at every direct call site (no `call`, no symbol unless its address is taken/`@export`ed); ill-formed on a recursive function (CG-10) |
| **linkage** | `@extern` · `@extern("name")` | binding (body-less decl) | Semantic + link | Modules §7 |
| **linkage** | `@export` · `@export("name")` | binding / function | Semantic + link | Modules §6 |
| **build** | `@limits(…)` | file / module | Config + Semantic | Composition (FND-11/FND-10); Tooling |
| **control** | `@label(name)` | code construct (instruction / loop / block) | Semantic (name resolution) | Control Flow §2.1, §7.1 |
| **test** | `@test("description")` | a **name-less** top-level function `fn()` | Semantic | Tooling §4.1 — a runtime test collected by `alatyr test`, ignored by `build` (TOOL-5) |

The families (storage / layout / contract / abi / codegen / linkage / build / control /
test) are **documentation and diagnostic** structure only — there is **no** `@family.name`
syntax (SYN-7/3B): isolation is provided by the **diagnostic**, not the namespace. An unknown
`@name`, or one applied to a target kind not listed for it, is a Semantic diagnostic that
**names the family** — *"unknown layout attribute `@foo`"*, *"`@reg` is a storage attribute,
not valid on a type"* — so the relevant family is discoverable from the error without a
syntactic prefix. Each family is **extensible additively** (I10/FND-6). The set of layout
machine levers (`@repr`…`@niche`) is **closed** (CT-10); other `@name` are
**comptime-function effectors** (prelude or library, indistinguishable — CT-10), attached
per their function shape. `@` is always a **compile-time prefix on a construct** and
**never wraps an expression** (CG-6); the equivalent UFCS surface is `construct.name(args)`.

---

### 3. Type annotation and inference

#### 3.1 Annotated — `name : T = value`

The type `T` is explicit. `value` MUST be **assignable** to `T`: it is either of
type `T`, or implicitly convertible to it (only lossless **widening**, and only
outside the `no_abstractions` limit — under `no_abstractions` nothing converts
implicitly; Type System §4.3), or an untyped literal in range (§3.4). Otherwise the
declaration is ill-formed.

#### 3.2 Inferred — `name := value`

`T` is **inferred from the type of `value`**. Inference is **local** — from the
initializer alone — not bidirectional or whole-program. A `value` whose type is not
determinable on its own (e.g. a bare integer literal with no context, §3.4) MUST be
annotated.

#### 3.3 Uninitialized — `name : T`

`T` is explicit (there is no value to infer from); the binding is **uninitialized**
(§4). The annotation is mandatory in this form.

#### 3.4 Literals and context

An integer/float literal takes its type from **context** — the annotation `T`, or
the target place/parameter — and its representability is checked at compile time
(Type System §9.1). With **no** context it takes the documented default (the
target's native signed integer); a literal that needs a non-default type with no
context MUST be annotated or constructed (`T(v)`).

---

### 4. Initialization and definite assignment

- A `name : T` binding is **uninitialized**. It MUST be **definitely assigned**
  before any read — i.e. on every control-flow path to a use, a prior store is
  proven (Type System §9.4). Reading uninitialized storage is **forbidden** (it
  would be UB, against I11).
- **`uninit`** is the explicit opt-out (Type System §9.4): in checked code a read
  still must satisfy definite assignment; an unchecked read yields hardware-defined
  contents.
- **No implicit zero-initialization** (I3): the language never zeroes a binding on
  the programmer's behalf. Programmer-declared defaults (struct field / array
  element) apply at construction, not at declaration (Type System §9.4).
- **Immutable vs mutable initialization.** Initialization establishes the binding's
  value; the **mutability** axis governs what happens **after**: an immutable binding
  (no `mut`) is established **once** — by its initializer, or by a single definite
  assignment before first use — and MUST NOT be stored to again; a `mut` binding MAY
  be stored to repeatedly (Memory §3).

---

### 5. Scope

Scope is **lexical and block-structured**. The scopes are:

- a **block** `{ … }`;
- a **function** (its parameters and body);
- a **module / file** (§7.1);
- a **comptime block** (Comptime §8.1).

A name is visible in the scope where it is declared and in all **nested** scopes; a
nested scope sees the names of its enclosing scopes. **Scope is the *name* axis**
(Memory §1.5) — independent of the binding's storage class (space) and
lifetime (time).

---

### 6. Shadowing

#### 6.1 Across scopes — allowed

A declaration in an inner scope **may shadow** an outer name; within the inner
scope the inner name wins, and the outer name is again visible after the inner scope
ends.

#### 6.2 Same scope — forbidden

Re-declaring a name **already bound in the same scope** is a **compile error**. This
is the same rule as "no duplicate names, no implicit winner" for module scope
(Modules, decisions MOD-8). (Same-scope shadowing, as in Rust, is **rejected** — for
clarity: a name means one thing within its scope.)

The **one exception** is **function overloading by signature** (Functions §1.4 / FN-7):
a name MAY be bound to several **function values** in the same scope when they differ in
their **parameter signature**, resolved per call by the argument types. Two functions
with the **same** signature, or a function and a non-function, are still a redefinition
error — so a name still means **one role**, dispatched by argument type.

---

### 7. Declaration order

#### 7.1 Module / top level — order-independent

At module (top) level, declarations are **order-independent**: a name is visible
throughout the module regardless of textual position, so **mutual recursion** (types
referring to one another, functions calling one another) needs **no forward
declarations**.

A module-level data binding's initializer MUST be **compile-time-known** (no runtime
static initialization; Memory §2.2). Such comptime initializers are evaluated
in **dependency order** (an initializer may reference another binding's comptime
value); a **cycle** among comptime initializers is a compile error (purity and
memoization make this well-defined, Comptime §2.5).

For a module-level binding of required type `R = @require(pred) U`, a checked
`R(u)` initializer therefore executes `pred` exactly once in the comptime evaluator
after `u` is evaluated once (Type System §8.1). False is a Semantic diagnostic at that
construction site; true emits only the resulting static bytes. No runtime predicate
call, branch, trap, or hidden initializer is emitted. An `unchecked R(u)` initializer
evaluates `u` once but skips the predicate in the evaluator as well.

#### 7.2 Local — order-dependent

Inside a block or function, declarations are **order-dependent**: a name is visible
**from its point of declaration onward**. This is what makes definite-assignment and
use-before-declaration analysis well-defined (§4).

#### 7.3 Rationale

Module-level order-independence is required for ordinary mutual recursion;
local order-dependence keeps initialization and reading flow analyzable
(definite assignment, §4). The two regimes are deliberate, not an inconsistency.

---

### 8. Lowering

- A binding **introduces a place** of its (declared or inferred) type in its storage
  class — automatic (register/stack) for a local, static for a module-level binding
  (Memory §2). The storage class is unchanged by `comptime`/`mut`/`pub`, which
  affect phase/permission/visibility, not placement.
- **Initialization** lowers to a **store** of the value into the place. For a static
  binding with a compile-time-known initializer, the value is **emitted into the
  section** (Memory §2.3) — no runtime store. For `name : T` (uninitialized),
  no store is emitted until the definite assignment.
- A **`comptime` binding is erased** (Comptime §1.5); if its value crosses into a
  runtime use it is materialized into **static** storage, never the heap (Comptime
  §1.6).
- No declaration emits hidden initialization, zeroing, or control flow (I3).

---

### 9. Syntax (grammar fragments)

The forms this chapter introduces are given here as **grammar fragments**; the
**consolidated EBNF** and lexical rules (identifiers, the `:=` token, separators) are
assembled in the Grammar chapter, and these fragments conform to the value-introducer
and expression grammar of the Type-System, Comptime, and Functions chapters.
Nonterminals not defined here (`ident`, `type-expr`, `expr`, `place`, `attribute`)
come from those chapters.

```ebnf
declaration ::= { modifier } binding
binding     ::= ident ":" type-expr [ when-clause ] "=" expr   (* typed + initialized; when gates it (Comptime §7.1) *)
              | ident ":=" expr [ when-clause ]    (* inferred; when at the end *)
              | "(" ident { item-sep ident } ")" ":=" expr [ when-clause ]  (* multi-out destructuring (Functions §3.2) *)
              | ident ":" type-expr [ when-clause ]   (* uninitialized (§4) *)
modifier    ::= "pub" | "mut" | "comptime" | attribute
when-clause ::= "when" expr                       (* the general declaration guard (Comptime §7.1; CT-5): gates ANY declaration's existence; at most one per declaration — for a function value it sits in the fn-sig instead (Functions §9) *)

assignment  ::= place "=" expr                     (* store to an existing place — not a declaration (§1.2) *)
```

Separators (decisions SYN-4; consolidated in Grammar):

- declarations / statements are separated by a **newline or `;`** (`;` to put several
  on one line);
- a line is **not** terminated if it is syntactically incomplete — the last token
  cannot end a construct (a trailing binary operator, `=`, `,`, `.`, or an open
  bracket), or the next line begins with a continuing token (deterministic,
  Go/Swift-style); inside `( … )` / `[ … ]` newlines are free.

Normative notes:

- `:=` declares-and-infers; `:` …`=` declares-with-type; `:` alone declares
  uninitialized; bare `=` assigns to an existing place (§1.2).
- The RHS is any value-expression, including the value-introducers of §1.3.
- Parameters are typed; **generics are expressed only via `T : type`** — there is no
  `any` / `?` parameter marker (it would add no capability over `T : type`; decisions
  SYN-6).

---

### 10. Conformance (normative summary)

A conforming implementation MUST:

1. accept the single declaration form and its inferred/uninitialized variations, and
   distinguish declaration (`:=` / typed) from assignment (bare `=` to an existing
   place), rejecting a bare `=` to an unbound name (§1);
2. provide **no** declaration keywords; bind a name to the value of an ordinary RHS
   value-expression, including the `struct`/`enum`/`union`/`fn`/`mod`
   introducers (§1.3; a generic type is an ordinary `fn(…) -> type`, and `brand(…)` an
   ordinary builtin call — neither is an introducer);
3. treat `pub`, `mut`, and `comptime` as distinct modifier axes — `pub` combining
   freely with either, but `comptime` entailing immutable so that `comptime mut` is
   ill-formed — provide **no** `const` keyword (a constant is a `comptime` binding),
   and apply `@`-attributes per the owning chapters (§2);
4. type-check an annotated initializer for assignability, infer the type locally for
   `:=`, require an annotation where the value's type is not self-determined, and
   take literal types from context with the documented default otherwise (§3);
5. enforce definite assignment before any read, forbid reading uninitialized storage
   in checked code, impose no implicit zero-initialization, and allow an immutable
   binding to be established exactly once (§4; I3/I11);
6. implement lexical block scope as the name axis (§5);
7. allow shadowing across scopes and reject same-scope redeclaration with a
   diagnostic (§6);
8. make module-level declarations order-independent (mutual recursion, no forward
   declarations) with compile-time-known module initializers evaluated in dependency
   order (cycles an error), evaluate a checked module-level `@require` gate entirely at
   comptime (false a Semantic diagnostic, true only static bytes, no runtime initializer),
   and make local declarations order-dependent (§7; Type System §8.1);
9. lower a binding to a place in its storage class, lower initialization to a store
   (or to emitted section contents for a static comptime-known initializer), erase
   `comptime` bindings, and introduce no hidden initialization or control flow
   (§8; I3).
