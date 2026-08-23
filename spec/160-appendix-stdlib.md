# Alatyr Language Specification

## Appendix — Prelude and standard library

> **Status — draft (under review).** This appendix is a draft and is not yet
> accepted; the specification as a whole is under development. See the status note
> at the top of the Overview chapter.

This appendix fixes the **normative v1 definitions** the Stdlib chapter defers
to "the Stdlib appendix" (Stdlib §6): the **core protocols**, the **base-tier
type operations** (`Option` / `Result` / slice `[T]` / `str` / `bool` / `char`), the
**built-in catalog** (each built-in's signature and contract), the **allocator
protocol** and provided allocator surface(s), the **alloc-tier** types, and the **std** surface.
The Stdlib chapter is normative for the **tiers** (§1), **DCE** (§2), **built-in
structure** (§3), **runtime-builtin semantics** (§4), and the **`@alloc` model** (§5);
this appendix enumerates the definitions those sections promise.

These definitions are **ordinary library code** written with the construction machinery
(Type System §5) and the comptime protocols (Comptime §6.2) — the language eats its own
dogfood (Type System §7). Their **cost is visible** (I2); none is privileged syntax.
The set here is the **curated v1** set; **"additive" (FND-6) governs only future growth**,
not whether v1 is specified (Overview §6).

---

### 1. Conventions

- Definitions use the language's own surface: a function value `name := fn(…) { … }`, a
  generic type `Name := fn(T : type) -> type { … }` (SYN-5), results via anonymous projected `-> T` or named `out`
  (Functions §3), and protocols as comptime predicates `fn(type) bool` (Comptime §6.2).
- Operations are shown in **UFCS** form (`x.len()` ≡ `len(x)`; TYP-5). Receiver direction
  is explicit: `in self` (read), `in out self` (mutate).
- A **tier** tag — base / alloc / std (Stdlib §1) — is given per group. Base-tier items
  are freestanding and need no allocator.
- `ptr(mut bits8)` is the raw byte-pointer type (Memory §4.1); `usize` is the
  pointer-width unsigned interpretation (Type System §7).

---

### 2. Core protocols (comptime shapes)

A protocol is a **structural** comptime predicate (Comptime §6.2): a type satisfies it
by **providing the listed operations** (checked with `resolves`/`compiles`), not by
naming it. The load-bearing protocols:

#### 2.1 `Truthy` — conditions (`if` / `while` / `and` / `or` / `not`)

A type is **truthy** if it provides `is_true(in self) -> bool`. `bool` satisfies it by
identity. Conditions (Control Flow §4) and the logical operators (Control Flow §8) take
a truthy operand.

The **numeric scalars** (`bitsN`/`uN`/`iN`/`fN`) and `char` **do not** provide `is_true`
and are therefore **not truthy**: a numeric condition is written with a comparison that
yields `bool` — `if n != 0 { … }`, not `if n { … }` (the latter is ill-formed). This
keeps `bool` distinct from the integers (Type System §7) and removes the C `if (n)`
footgun; the truthiness of an integer is never implicit.

#### 2.2 `Tryable` — the `?` operator (Control Flow §8.2)

A type is **tryable** if it both **decomposes** into **success** / **failure** and can
**reconstruct** a failure (the latter so `?` can rebuild the *enclosing* function's failure
when its type differs from the operand's — CF-10):

```
is_success(in self) -> bool        # is this the continue case?
success_value(in self) -> S        # the unwrapped value on success
failure_value(in self) -> F        # the failure payload on failure
from_failure(in f : F) -> Self     # construct the failure case carrying f (the inverse of failure_value);
                                    #   Self = the tryable type itself
```

`expr?` evaluates to `success_value` and continues on success; on failure it converts
`failure_value` into the enclosing function's `out` failure type by a **declared** conversion
`OutErr(failure_value)` (the `T(v)` lattice — matching types pass directly, a declared conversion is
applied, else a compile error; declared-and-visible, not a hidden `From`, I8), then **wraps** that
failure into the enclosing tryable via its **`from_failure`**, and early-exits (running `defer`s, Memory
§5.8). Three cases of the wrap (all the same rule — build the enclosing tryable's failure):
**(a)** when the enclosing return is the operand's **own** type, `?` returns the operand verbatim (no
rebuild — an impl fast path); **(b)** `Option(T)` / `Result(T, E)` use their **built-in** failure
constructors (`None` / `Err`) — their `from_failure` need not be written; **(c)** a **user** tryable
returned from a function whose `?`-operand is a **different** tryable is rebuilt by the enclosing type's
`from_failure`, selected by that (known) return type. `Option(T)` and `Result(T, E)` satisfy the protocol
(§3.3/§3.4) — by **shape**, not by name; a user type satisfies it by providing all four operations.

#### 2.3 `Optional` — "a value or empty" (iteration)

A type is **optional** if it decomposes into **present (a value)** / **absent**:

```
is_present(in self) -> bool        # is there a value?
value(in self) -> T                # the value, when present
```

The absent branch carries **no** payload — the distinction from `Tryable` is **error-with-data**
(tryable's failure) vs **bare absence** (optional). `Option(T)` satisfies it (`Some` = present,
`None` = absent) **and** `Tryable`; `Result(T, E)` satisfies only `Tryable`. `for` (§2.4) depends on
this **shape**, not on the name `Option`.

#### 2.4 `Iterator` — `for x in …` (Control Flow §6)

A type is an **iterator** if it provides `next(in out self) -> Opt` where `Opt` satisfies the
**optional** shape (§2.3) — a **present** result yields the value and continues, **absent** ends the
loop. A type is **iterable** if it provides `iter(in self) -> I` where `I` is an iterator **or is
itself iterable** — in particular a container whose `iter` returns a **slice `[T]`** (e.g. `Vec`)
is iterable: `for` iterates the returned slice directly (the built-in counted loop), so no `next`
iterator is required. Ranges (`a..b`, `a..=b`), slices `[T]`, and arrays satisfy iterable from the
prelude. (`Option(Item)` satisfies
optional, but `for` depends on the **shape**, not the name.) Iterator **adapters**
(`map` / `filter` / `fold` / `zip` / `enumerate` / …) compose on this protocol and are
**additive** (FND-6) — v1 fixes the `next` / `iter` shape and `for`.

#### 2.5 `Allocator` — `@alloc(value)` (§5)

A type is an **allocator** if it provides the allocation shape of §5 and declares the
**lifetime mechanism** it supplies (scoped/region/generational/manual; Memory §5.2),
which fixes the reference representation of `@alloc(it)` allocations.

#### 2.6 `Eq` / `Ord` / `Hash` — equality, ordering, hashing (derived structurally)

The comparison operator-functions (`==` `!=` `<` `<=` `>` `>=`, eager → `bool`; Control
Flow §8 / OP-2) and `HashMap` keys (§6) rest on operations the prelude **derives
structurally** over `typeinfo(T)` (Comptime §5.3), so a type provides them **by
default** and may **override** per type (TYP-5). The derive walks each kind by its `typeinfo`
shape: a **struct**'s fields via `a.(f)` (Comptime §5.4), an **array**'s elements by index,
and an **enum**'s variants via the comptime variant pattern `T.(v)` in a `comptime
for`-unrolled `match` (Comptime §5.5 / CT-9) — a scalar leaf compares directly. So the
derives are library code over `typeinfo` for **every** type, the compiler keeping no
built-in structural-comparison knowledge. The signatures below are the **conformance
shape** — what satisfies `Eq` / `Ord` / `Hash` **for a given type `T`**:

- **`Eq`** — `eq(in a : T, in b : T) -> bool` (backs `==` / `!=`); default = componentwise
  field equality.
- **`Ord`** — `lt(in a : T, in b : T) -> bool` (backs `<`; `>` / `<=` / `>=` derive from
  `lt` and `eq`); default = lexicographic by declaration order.
- **`Hash`** — `hash(in self : T) -> u64`, **consistent with `Eq`** (equal values hash
  equally); default folds the field hashes. A `HashMap` key satisfies `Eq` **and** `Hash`.

The **default** is a single **generic** prelude function — `eq := fn(T : type, in a : T,
in b : T) -> bool` (likewise `lt` / `hash`), with the comptime type parameter `T` **inferred
from the operands** (TYP-1/CT-2; Comptime §5.4 shows the body) — so `a == b` applies it without
naming `T`. (The conformance shape above is that generic instantiated at one `T`; there is
no implicit-type-parameter form — every generic names `T : type`, Functions §1.4.) A type
**overrides** the structural default by defining its own **concrete** `eq` / `lt` / `hash`
for itself (e.g. `eq := fn(in a : Money, in b : Money) -> bool { … }`): overload resolution
(FN-7) prefers the **more specific** concrete definition over the generic default at a call
on that type. Derivation is **comptime + monomorphized** → zero-cost and fully erased
(I2/I7), no RTTI; satisfaction is structural (`resolves(eq, …)`, …), not by name. The v1 derived core is
`eq` / `lt` / `hash`, and the prelude ships the **v1 `Ord` consumers** over it:

- **`min`** / **`max`** — `min(in a : T, in b : T) -> T` / `max(…) -> T` (base); a tie
  returns the **first** argument.
- **`clamp`** — `clamp(in x : T, in lo : T, in hi : T) -> T` (base); requires `lo <= hi`,
  a **checked precondition** (a violation traps, I11).
- in-place **`sort`** on a mutable slice `[mut T]` (§3.5).

Broader structural derives (`clone`, a three-way total-order `compare`, a streaming
`Hasher`, serialization) build the **same** way and are **additive** (FND-6).

#### 2.7 `Writer` — a byte sink (output)

A type is a **writer** if it provides:

```
write(in out self, in bytes : [u8]) -> Result(usize, E)   # bytes accepted, or the writer's own failure
```

returning the number of bytes consumed (a **short write is legal** — the caller loops) or a failure
of the writer's **own error type `E`**. Per tier discipline (Stdlib §1), `E` is the implementer's
error, of the **writer's tier or below** — there is **no** protocol-fixed error type (a fixed
std-tier `IoError` here would be a tier inversion for a lower-tier sink, STD-2): the std byte streams
`stdout` / `stderr` and `File` (§7, std) fail with the std-tier **`IoError`**, while an **in-memory
buffer** (`String` / `StrBuf`, alloc) fails with the base-tier **`AllocError`** (§5.1) — its only
real failure is exhausting its allocator (`OutOfMemory`). A `Writer` is the **sink** output composes
against: formatting, serialization, and logging target *any* writer rather than a fixed stream, and
compose **generically over `E`**. `?`-propagation works because the result is `Tryable` (§2.2) for
every `E`; the structural predicate is "`write` returns `Result(usize, _)`".

The v1 commitment is the **protocol shape** plus a **minimal, alloc-free rendering layer**: the
std renders a **scalar** to its textual bytes and writes them to any `Writer` (or into a
caller-provided `[mut u8]` buffer, returning the byte count) — an **integer** in **base-10** (a
leading `-` for a negative two's-complement value, minimal digits, `0` for zero), a **`bool`** as
`true`/`false`, a **`char`** as its UTF-8 bytes. Output **composes by sequential writes** (write
the label, then the value, then a newline) — no allocation and no format string, so it works on a
freestanding target. This is the layer that lets a program print a **computed** value (TOOL-5 note:
distinct from a compile-time-known string, which is just a literal).

The conveniences over this shape are **additive** (FND-6), each gated on a capability the minimal
layer does not need — adding them changes neither the protocol shape nor the minimal layer:
- **`write_all`** (loop until the whole slice is consumed) — rides the shape directly;
- **`format(w, "…", args…)`** with a **comptime-checked** template — needs **comptime-variadics**
  (a heterogeneous argument pack, Functions §7.1). The template is a **comptime string literal**:
  `{}` is a **hole** (filled, in order, by the trailing arguments, each rendered by the scalar/
  `Display` layer below); `{{` and `}}` are **escapes** for a literal `{` and `}`; a lone `{` or
  `}` is a **comptime error**. The number of `{}` holes **MUST equal** the argument count — a
  mismatch is a comptime error (never a silently empty hole or a dropped argument). A non-literal
  template is outside v1. A `float` hole renders in base-10 with a bounded fractional part (an
  interim layer; a shortest-round-trip formatter is additive);
- a **`Display`** protocol + a **structural derive** rendering an aggregate field-by-field — needs
  **`typeinfo`** field introspection (Comptime §5);
- **`format(…) -> String`** (rendering into a returned growable string instead of a sink) — needs
  the **`alloc` tier** (§6).

---

### 3. Base-tier types

**Mode-polymorphism guarantee (the prelude operators, CT-11).** Every **prelude checked
operator and conversion** — the arithmetic `+` `-` `*` (overflow), `/` `%` (div-by-zero),
the comparisons, `char(n)` (code-point), narrowing conversions, indexing/sub-slicing bounds —
is, by **prelude guarantee**, a **mode-polymorphic** library function (Type System §4.5 / CT-11):
its guard is comptime-present in a checked context and comptime-absent inside an `unchecked`
scope. So although operators are ordinary library functions (TYP-2/TYP-5 — the compiler keeps no
built-in numeric knowledge), `unchecked (a + b)` still **wraps** and `unchecked xs[i]` still
**skips the bounds check** — the guard drops in the unchecked instantiation (Concurrency §3 /
CG-8). This is a **convention of the prelude**, relied on by users: a checked operation from the
prelude responds to `unchecked`. A *user-defined* function is **mode-opaque** unless it itself
reads `verify.checked` — a caller's `unchecked` does not reach into it (CG-7); a library that
wants its own checked operation to be `unchecked`-responsive opts in the same way the prelude
does (`comptime if verify.checked { … }`).

#### 3.1 `bool` (base)

The two-valued type (`false` = 0, `true` = 1; Type System §7) — a prelude **brand**
over `bits8` (`bool := brand(bits8)`, TYP-2/TYP-4; the kernel carries no built-in two-valued
type). Result of comparisons; operand of `and`/`or`/`not` (Control Flow §8). Satisfies
`Truthy`. Does **not** arithmetically interconvert with integers implicitly. As a brand that
marks a scalar interpretation, its `typeinfo` reifies as `Scalar{bits = 8, kind = Bool}` (§4.1,
not a `Brand`), on which a structural derive dispatches by `kind` (e.g. `fmt` renders `true`/`false`).

`bool` is **outside the numeric conversion lattice** (Type System §4.2), so no
bool↔integer transition is a *lattice* `T(value)` conversion and none is **implicit**.
Such a conversion is instead a **library conversion-constructor** (`@convert`, Type
System §4.6) — always explicit, the library author fixing its meaning (the old
ambiguity is thus resolved *by choice*, not by the language):

- **Truth value from a number:** the canonical, allocation-free spelling stays the
  comparison `n != 0` (which yields `bool`). A library *may* expose an explicit
  `@convert` to `bool`, but it must commit to one meaning — a *checked narrow* (`5 ∉
  {0,1} ⇒ trap`) or a *truthiness fold* (`5 != 0`) — and at most one such `@convert`
  may be in scope at a site (else an ambiguity error, §4.6); `n != 0` stays the
  recommended idiom.
- **Integer from a bool:** the direct spelling stays `if b { 1 } else { 0 }`. Since
  `b` is unambiguously `0`/`1`, a library may also expose `@convert`s such as `u64(b)`;
  the base tier ships the **idiom**, not the conversions — the language no longer
  forbids them (TYP-6), it leaves the choice to the library.

#### 3.2 `char` (base)

A Unicode code point — a prelude **brand** over `u32` (`char := brand(u32)`, TYP-2/TYP-4;
the kernel carries no built-in code-point type). Conversions to/from `u32` are explicit
(`u32(c)` — the zero-cost brand/numeric crossing; `char(n)` — a **library conversion-
constructor** (`@convert`, §4.6) whose code-point validity check (≤ 0x10FFFF, not a
surrogate; I11) is library code, dropped under `unchecked` per the verification mode
(CT-11)). `char` is **not** a byte. As a brand that marks a scalar interpretation, its `typeinfo`
reifies as `Scalar{bits = 32, kind = Char}` (§4.1, not a `Brand`), on which a structural derive
dispatches by `kind` (e.g. `fmt` renders it UTF-8).

Classification predicates (curated v1 — the **ASCII** tests a lexer/parser needs; each
reads the code point via `u32(c)` and tests an ASCII range):

```
is_digit(in c : char) -> bool         # '0'–'9'
is_alpha(in c : char) -> bool         # 'a'–'z' or 'A'–'Z'
is_alnum(in c : char) -> bool         # is_alpha or is_digit
is_whitespace(in c : char) -> bool    # space, tab, newline, CR, VT, FF
is_hex_digit(in c : char) -> bool     # '0'–'9', 'a'–'f', or 'A'–'F'
to_lower(in c : char) -> char         # ASCII 'A'–'Z' → 'a'–'z'; other code points unchanged
to_upper(in c : char) -> char         # ASCII 'a'–'z' → 'A'–'Z'; other code points unchanged
hex_value(in c : char) -> Option(u32) # '0'–'9'/'a'–'f'/'A'–'F' → 0–15, else None
```

These are ordinary base-tier library functions, not intrinsics. **Unicode-general**
classification (the full property tables) is **additive** (FND-6) — deliberately not in
v1: ASCII covers the v1 compiler's own source text.

#### 3.3 `Option(T)` (base)

```alatyr
Option := fn(T : type) -> type { enum { None, Some(T) } }      # niche-optimized where T provides one (Type System §7)
```

Operations (curated v1): `Some(v)` / `None` constructors; `is_some` / `is_none`;
`unwrap(in self) -> T` (a `None` is a **defined-failure** `panic`, §4, never UB);
`unwrap_or(in self, in default : T) -> T`; `map` / `and_then` (monomorphized);
`get(in self) -> Option(ptr(T))` for in-place access. **Satisfies `Tryable`**
(success = `Some`, failure = `None`); `x?` on an `Option` requires the function's `out`
to be `Option`-compatible (Functions §3).

#### 3.4 `Result(T, E)` (base)

```alatyr
Result := fn(T : type, E : type) -> type { enum { Ok(T), Err(E) } }
```

Operations (curated v1): `Ok(v)` / `Err(e)`; `is_ok` / `is_err`; `unwrap` (an `Err` →
`panic`, §4) / `unwrap_or`; `map` / `map_err` / `and_then`; `ok(in self) -> Option(T)`.
**Satisfies `Tryable`** (success = `Ok`, failure = `Err`); a mismatched `Err` at a `?` site is converted
by a **declared** conversion `OutErr(e)` (§2.2).

#### 3.5 Slice `[T]` (base)

A **pointer + length** pair over a contiguous run of `T` (Type System §7; the layout is
that library pair, not a primitive).

```alatyr
Slice := fn(T : type) -> type { struct { ptr : ptr(T), len : usize } }   # spelled [T]
```

Operations (curated v1): `len(in self) -> usize`; indexing `self[i]` (**bounds-checked**;
out-of-range traps, I11 — the index operator's **compiler-emitted** built-in read for a
slice/array, OP-3 / Type System §4.5); sub-slicing `self[a..b]` (bounds-checked; the
built-in `index_range`); `first` / `last`
→ `Option(...)`; `iter` (satisfies `Iterator`, §2.4, yielding `ptr` or value per
mutability); on a **mutable** slice `[mut T]`, in-place `sort()` — by `Ord` (`lt`, §2.6),
**O(n log n) worst-case**, **not guaranteed stable**, with **no allocation** (an
introsort-class algorithm; a quadratic worst case is **non-conforming**, and the bound is
worst-case so it holds on adversarial input — available under `no_alloc`). (A
**stable** sort and `sort_by` / key-comparator variants are **additive**, FND-6.) A mutable
slice is `[mut T]` (element mutability via the pointer, Memory §4.1).

#### 3.6 `str` (base)

A slice of **UTF-8 bytes** (`[u8]`) with the **validated-UTF-8 invariant** (Type System
§7; Grammar §2.4). Length is reported with **explicit units** (byte ≠ code point is
named, not hidden):

```
byte_len(in self) -> usize             # O(1)
codepoint_count(in self) -> usize      # O(n)
chars(in self) -> CharIter             # iterate code points (Iterator, §2.4)
bytes(in self) -> [u8]                 # the underlying bytes
str_at(in p : usize, in n : usize) -> str   # raw view: `n` bytes at address `p` (unchecked inverse of `bytes`)
```

Indexing by **byte** offset is checked to land on a code-point boundary (a mid-code-point
index traps, I11). A growable string is `String` (alloc tier, §6).

`str_at(p, n)` is the **raw inverse of `bytes`**: it reinterprets the `n` bytes at
address `p` as a `str` (`usize → ptr(u8) → [u8] → str`) in one step, so a scanner
walking a byte buffer by address (a lexer reading `base + offset`) names a lexeme without
the three-line pointer dance. It is **unchecked** — the caller guarantees the bytes are
valid UTF-8 and outlive the view (the view aliases the storage, no copy); the unsafety is
encapsulated in the one function rather than repeated at each call site.

#### 3.7 `Never` (base)

The **bottom type** (the named divergence type fixed by CF-7): it has **no value**, so an
expression of type `Never` cannot produce one — it only **diverges**. It is the result
type of `panic` / `exit` (§4.2) and of any function that never returns, **including an
`@extern` one** (`abort := @extern fn() -> Never`) — so no separate `@noreturn`
attribute is needed (OP-1: nothing expressible by `-> Never` is re-introduced). A
`Never` result is a **terminator** for flow analysis (definite-assignment and
block-value rules treat code after it as unreachable, Control Flow §9; Grammar §3.1).
`Never` is **not** `Option`/`Result`'s failure type — it is the type of *no result at
all*.

---

### 4. Built-in catalog

Built-ins are **prelude identifiers** (no `@`, not keywords; Stdlib §3) — flat
singletons (pointer ops `ptr` / `deref`, layout word-functions `T.size()` / `T.align()`),
or members of the prelude modules `atomic` / `volatile` reached by `::`. There is **no**
`mem` module: addressing/layout are flat word-functions, and the addressing operand `at`
lives on the arch surface (Assembly §4). Two kinds.

#### 4.1 Compiler intrinsics (cannot be written in the language)

| Built-in | Signature (UFCS receiver where applicable) | Contract |
|----------|--------------------------------------------|----------|
| `typeinfo`  | `typeinfo(comptime T : type) -> TypeInfo`            | comptime type introspection; no RTTI (Comptime §5) |
| `bitcast`   | `bitcast(comptime T : type, in v : U) -> T`          | reinterpret bits; **equal width** required (Type System §4.4) |
| `size`   | `T.size() -> usize` (prefix `size(comptime T : type)`) | size in bytes; compiler-folded over the machine model (Memory §4.3) |
| `align`  | `T.align() -> usize` (prefix `align(comptime T : type)`) | alignment in bytes; compiler-folded (layout = machine model, I6) |
| `ptr`   | `ptr(place) -> ptr([mut] T)`                 | take an address (Memory §4.3); on a *type* `ptr(T)` / `ptr(mut T)` it is the pointer **type** (`mut` on the pointee), on a *place* its address — UFCS `place.ptr()` |
| `deref`    | `deref(in p : ptr([mut] T)) -> T`               | load, or a **place** when assigned (UFCS `p.deref()`); the place's write permission follows `p`'s pointee mutability — `ptr(mut T)` is assignable, `ptr(T)` read-only (Memory §4.3/§4.4) |
| `resolves`  | `resolves(callee, comptime args) -> bool`            | does this call type-resolve? (comptime; Comptime §6.1) |
| `compiles`  | `compiles(comptime expr) -> bool`                    | does `expr` typecheck? (not evaluated) |
| `embed`     | `embed(comptime path : str) -> [u8; N]`              | reproducible file embed → byte array (Comptime §2.4) |
| `forget`    | `forget(in v : T)`                                    | discharge linearity without release (Memory §5.9) |
| `typeof`    | `typeof(in v : T) -> type`                            | comptime: the **static type** of `v` as a `type` value (no RTTI; distinct from `typeinfo`, which is introspection *data*) |

(The per-arch **instruction** intrinsics are also compiler intrinsics, but belong to the
Assembly per-domain family of §4.3 — they are not relisted here, so the partition stays
exact.)

`typeinfo`'s result type **`TypeInfo`** is a base-prelude type — a comptime-only tagged
enum (no RTTI is emitted; Comptime §5.1/§5.2):

```alatyr
TypeInfo := enum {
  Scalar  ( struct { bits : usize, kind : ScalarKind } )     # width + numeric interpretation
  Struct  ( struct { fields : [Field] } )
  Enum    ( struct { variants : [Variant] } )
  Union   ( struct { variants : [Variant] } )                # raw members use Variant.payload; no active-member RTTI/tag (TYP-11)
  Array   ( struct { elem : type, n : usize } )
  Pointer ( struct { pointee : type, mutable : bool } )      # ptr(T) / ptr(mut T)
  Function( struct { params : [Param], results : [Param] } ) # by direction (Functions §3)
  Brand   ( struct { underlying : type, marker : str } )     # nominal brand (Type System §5.4); marker = the brand's name
  Str     ( struct { } )                                     # the `str` validated-UTF-8 byte view (§3.6) — an opaque base type, no introspectable structure
}
Field   := struct { name : str, type : type, offset : usize, mutable : bool }
ScalarKind := enum { Bits, Uint, Int, Float, Bool, Char }   # a scalar leaf's numeric interpretation
Variant := struct { name : str, payload : Option(type) }   # the variant's whole payload as one type: `None` for a unit variant, the component type for one, the **tuple** of component types for several (`Add(Expr, Expr)` → `Some((Expr, Expr))`) — bound whole by the comptime variant pattern `T.(v)(p)` (CT-9; Comptime §5.5)
Param   := struct { name : str, type : type, dir : ParamDir }
ParamDir := enum { dir_in, dir_out, dir_in_out }   # the parameter directions (Functions §3)
```

(`ParamDir`'s variants are spelled `dir_in`/`dir_out`/`dir_in_out` rather than
`in`/`out`/`in_out` because the lowercase `in`/`out` are keywords (SYN-2) — a bare `in`
variant would collide with the keyword.)

**How a brand reifies (TYP-2/TYP-4/CT-6).** A brand whose name marks a **scalar interpretation** —
the prelude `uN`/`iN`/`fN`/`usize`/`isize`/`bool`/`char` — reifies as `Scalar{bits, kind}`, its
`kind` naming the interpretation (`Uint`/`Int`/`Float`/`Bool`/`Char`); the compiler keeps no
per-brand knowledge beyond that kind. A **user** nominal brand (`Meters := brand(u64)`, TYP-4)
reifies as `Brand{underlying, marker}`, `marker` the brand's name as a **`str`** — on which a
library consumer dispatches (`m == "Meters"`) or recurses on `underlying`. (So `typeinfo(bool)`
is `Scalar{bits = 8, kind = Bool}`, not a `Brand`; `typeinfo(Meters)` is `Brand{underlying = u64,
marker = "Meters"}`.) `marker` is a `str`, never a `type` — a derive matches it against a string
literal.

A `TypeInfo` value is inspected by ordinary comptime code (`match`, iteration); size and
alignment are derivable from it (and also via `T.size()`/`T.align()`). It is fully erased
before runtime (I7). Exposing user **comptime annotations** on a field — for library
derives such as serialization renames — is **additive** (FND-6; CT-6): v1 `Field` carries
exactly `{ name, type, offset, mutable }`.

#### 4.2 Ordinary prelude functions

| Built-in | Signature | Contract |
|----------|-----------|----------|
| `panic`  | `panic(in msg : str) -> Never` | defined failure (never UB); no unwinding; hosted = stderr + exit, freestanding = hook or trap (Stdlib §4) |
| `exit`   | `exit(in code : usize) -> Never` | hosted = OS exit; freestanding = trap/hook (Stdlib §4); code per §100 (the chapter governs the appendix) |
| `assert` | `assert(in cond : bool)`        | runtime → `panic` when false; in a comptime context → **compile error** (Stdlib §4.3); always emitted at runtime |
| `get`    | `get(in a : Arena, in h : Handle(T)) -> scoped ptr(mut T)` | region-handle access: bounds-checks `h` against `a` (out-of-range → trap); the result is a **`scoped` second-class return** (Memory §5.3.1) — the escape checker's generic rule, **no** built-in arena contract, **no** lifetime variable. An ordinary prelude function, not a compiler intrinsic (MEM-3). Stdlib §5.2.1; Memory §5.4 |

`panic` and `exit` **diverge**: their result type is **`Never`** (§3.7), the bottom
type, which has no value — so control does not return and a call is a **terminator** for
flow analysis (like `return`; definite-assignment and block-value rules treat the
following code as unreachable, Control Flow §9; Grammar §3.1). (`Never` is the named
divergence type fixed by CF-7; an `@extern` function that never returns is declared
`-> Never`, so no `@noreturn` attribute is needed — §3.7.)

`assert` has **no result type** (no `->`): it is a **statement-level effector** — a
`bool`-guarded `panic`, run for effect — so, unlike `panic`/`exit`, it is **not** `-> Never`
(on a true condition control falls through and continues). It does **not** yield a value and
**may not appear in value position** (there is no unit type to produce; Types §6); a call is an
ordinary statement. On a false condition it diverges through `panic` (a `Never` terminator);
when `cond` is a comptime-known constant the check folds (a comptime-false `assert` is a
compile error, §4.3).

#### 4.3 Per-domain built-ins (defined in their feature chapters)

The **full built-in catalog** (Stdlib §6) is the core built-ins above **plus** the
per-domain families whose signatures and contracts are **normative in their feature
chapters** — they are listed here for completeness, not redefined:

- **Assembly** (Assembly §4, §7): `asm(template : str, operands…)` (raw GAS escape,
  requires an `unchecked` grant) and `at(base=, index=, scale=, disp=, offset=)` (the
  addressing operand on the **arch surface** — in scope for the chosen target, qualified
  `<arch>::at(…)` outside an arch gate); the per-arch **instruction** intrinsics and
  **operand decorators** (per-arch appendix).
- **Concurrency** (Concurrency §2): the **atomic** family on `ptr` —
  `atomic::load` / `atomic::store` / the fetch-RMW ops (`atomic::fetch_add`,
  `atomic::fetch_sub`, …) / `atomic::cas_strong` / `atomic::cas_weak` (→ `(T, bool)`), each
  taking an `Ordering` (comptime enum); **`fence(Ordering)`**; the **`volatile`** access
  ops (`volatile::load` / `volatile::store`).
- **Overflow** (Concurrency §6): the per-operation variants
  `wrapping_*` / `saturating_*` / `checked_*` (→ `Option`) / `overflowing_*`
  (→ `(T, bool)`) on the integer interpretations.
- **Bit shift / rotate** (Types §2.2, §3.2; OP-6): the **named** operation-functions
  `shl` / `shr` / `rotl` / `rotr`, each `op(in v : T, in n : usize) -> T` for `T` a
  `bitsN`/`uN`/`iN` — call/UFCS form (no glyph). `shr` selects logical (`uN`/`bitsN`) vs
  arithmetic (`iN`) by the operand's interpretation; an over-width shift (`n ≥ N`) traps
  (Concurrency §6, I11), rotation is total (count `mod N`). The glyph operators
  `+ - * / % & | ^ ~` and the comparisons stay covered by Types §4.5/§5.3 (not relisted).

So the catalog is **partitioned**, not duplicated: §4.1/§4.2 here are normative; the
families above are normative in the cited chapters; together they are the complete v1
built-in set.

---

### 5. Allocators and the allocator protocol

#### 5.1 The protocol

A value is an **allocator** (usable as `@alloc(value)`, Memory §2.4; Stdlib §5) if its
type provides:

```
mechanism(in self) -> Mechanism          # comptime accessor; Mechanism := enum { region, generational, manual }
allocate(in out self, comptime T : type, in size : usize, in align : usize) -> Result(Ref(typeof(self), T), AllocError)
free(in out self, comptime T : type, in r : Ref(typeof(self), T), in size : usize, in align : usize)
```

The reference is **typed by the mechanism**, not always a byte pointer (MEM-1 — the mechanism
is reflected in the reference type; Memory §5.2). `Ref(A, T)` is a comptime type-function
that reads allocator type `A`'s `mechanism()` and yields the representation that mechanism
gives a `T`:

```
Ref := fn(A : type, T : type) -> type    # region → Handle(T) ; generational → GenRef(T) (additive) ; manual → a raw unchecked pointer type (additive — NOT ptr(T), which is always-valid/non-null/checked, Memory §4.1)
```

So a **direct** (recoverable) `a.allocate(T, size, align)?` already yields the mechanism's
reference — e.g. `Handle(T)` for a region — with **no** separate "form the reference"
step; `@alloc` (Memory §2.4) is the trapping convenience over the same call. `free` takes
that same reference back. (`scoped` is not allocator-managed, so it has no `Ref` case.)
Only the **region** case is concrete in v1; `GenRef(T)` and the `manual` **raw unchecked
pointer type** are **additive** (specified with their providers, §5.2 / reports). Crucially
the manual case is **not** `ptr(T)` — that type is always-valid, non-null and checked
(Memory §4.1), whereas a manual reference is a raw, `unchecked`-only pointer (Memory §5.2),
so its distinct type is what records the `unchecked` requirement (MEM-1); pinning that spelling
waits until the manual provider is built (OP-1 — not introduced before needed).

The comptime accessor `mechanism` is how the allocator **declares the lifetime mechanism**
it supplies — a UFCS function returning a comptime-known value, alongside `allocate`/`free`
(the UFCS model has no member-constant slot); `@alloc(self)` evaluates it to pick the
reference representation (MEM-4), so a new mechanism is additive (FND-6). The mechanism fixes
the **reference representation and per-access cost** of `@alloc(self)` allocations (Memory §5.2):

| Mechanism      | Reference representation | Temporal safety                    | Per-access cost |
|----------------|--------------------------|------------------------------------|-----------------|
| `region`/arena | a **handle** (index)     | checked against the owning arena   | a handle check  |
| `generational` | fat (pointer + generation)| generation compared on deref; stale → trap | one comparison |
| `manual`       | raw pointer              | none; **`unchecked`-only**         | zero; hardware-defined |

(`scoped` — the default, a static stack lifetime — is **not** allocator-managed; it
needs no `@alloc`. Memory §5.2.) A user allocator "just works" by providing the shape
above and a mechanism. If that mechanism is `manual`, using its `allocate`/`free` result
or `@alloc(value)` over it requires an enclosing `unchecked` grant and is rejected under
`no_unchecked` (the raw-pointer escape, Memory §5.2 / §5.6 rule 5). **`AllocError`** is a
base-prelude enum with a **closed v1 set**:

```
AllocError := enum { OutOfMemory, BadAlignment, SizeTooLarge }
```

`OutOfMemory` — no capacity for the request; `BadAlignment` — the requested alignment is
invalid or unsupported (e.g. not a power of two); `SizeTooLarge` — the size (after
alignment rounding) exceeds the allocator/implementation limit or overflows `usize`. The
`no_alloc` limit forbids `@alloc` entirely (Stdlib §5.3).

The protocol above is named **`alloc`**. A function takes an allocator as an ordinary
parameter of this protocol — `in a : ptr(mut alloc)` (an `in` pointer to a mutable
allocator, monomorphized per concrete type; never `in out`, Functions §5.1/§2.2). Within the
body that parameter **is** the function's ambient allocator, and at a call site it is
**elidable** (Functions §5.5): the omitted argument is filled from the caller's ambient,
established by the prelude form **`alloc::with(ar) { … }`** (MEM-5 / Memory §5.2.1). This is a
convenience over passing the allocator explicitly, not a hidden default — the ambient is an
explicit lexical binding, and an allocating site with no resolvable ambient is a compile error.

#### 5.2 Provided allocators

In v1 exactly one allocator **provider surface** is fully specified:

- **`arena`** — a region/arena allocator surface (`@alloc(arena)`), freed in bulk at region
  end; its v1 surface — `Arena` / `Handle(T)` / `arena_over` / `get` / `close` — is §5.2.1 (MEM-3).

An explicit `@alloc(value)` is required **per site**, on **every** target: v1 has **no
zero-config default provider** and **no** package-wide `allocator` manifest field (deferred,
additive — Manifest §3.7). The region allocator is constructed over a **caller-supplied
buffer** (`arena_over`, §5.2.1), so there is nothing to install implicitly, and a hosted
heap provider is not yet specified. The **`alloc::with` ambient** (MEM-5) does not change this:
it lets a function's allocator argument be elided within an **explicit lexical scope**, but
the scope (and its allocator binding) is written in the source — it is neither zero-config nor
a manifest default.

The following are **additive** (FND-6) — reserved provider names whose surfaces are specified
when first built, each fixing its concrete types, `mechanism` accessor, allocate/free
behavior, reference representation, and finalization rules (MEM-4 fixes the shared shape):

- **`gen`** — a generational allocator (a stale-reference deref traps, Memory §5.5);
- **`manual`** — a raw allocator (`unchecked`; no temporal safety);
- **`system`** — on a hosted target, a heap over the platform `malloc`/`free` (requires an
  allocator-bearing OS).

##### 5.2.1 The region mechanism — `Arena` and `Handle(T)` (v1 surface, MEM-3)

The region mechanism is two ordinary library types plus a prelude access function — the
language adds only the meaning of `@alloc` (Memory §2.4 / §5.4):

```
Handle := fn(T : type) -> type { ... }   # an index into its arena; copyable, not owning. Distinct per T because a
                                         # type-function result is nominal by (function, args) (Type System §4.1),
                                         # though the layout (usize) does not mention T.
Arena  := struct { ... }                 # the bump allocator
mechanism(in self : Arena) -> Mechanism { return Mechanism.region }   # the comptime accessor (§5.1)
# the protocol allocate/free (§5.1), typed by the region mechanism → Handle(T):
allocate(in out self : Arena, comptime T : type, in size : usize, in align : usize) -> Result(Handle(T), AllocError)
free(in out self : Arena, comptime T : type, in h : Handle(T), in size : usize, in align : usize)   # no-op; bulk close

# construct over a caller-supplied buffer (freestanding; no OS):
arena_over(in buf : ptr(mut bits8), in cap : usize) -> Arena
# access: exchange handle + arena for a scoped pointer, bounds-checked (out-of-range traps):
# `get` is an ordinary prelude function with a second-class return (Memory §5.3.1);
# the result is scoped to the call site, with no arena-specific compiler contract.
get(in a : Arena, in h : Handle(T)) -> scoped ptr(mut T)
# bulk reclamation:
close(in out self : Arena)
```

`@alloc(a) x := init` (Memory §2.4) bumps `a`, writes `init`, and binds `x : Handle(T)`;
`OutOfMemory` **traps** (the recoverable path is `a.allocate(T, …)?` directly, §5.1, which
already yields `Handle(T)`). `Arena.free(…)`
is a **no-op** — storage is reclaimed in bulk by `close`. The caller-buffer `Arena` is
**not owning** (it borrows the buffer); a **`@static`** fixed-pool arena likewise holds no
releasable resource and carries **no** finalize obligation (Memory §5.9 — "own forever").
An **OS-backed** arena holds a releasable resource (its `mmap`'d pages), so it is a
**distinct `@owning` type** (MEM-2) — not the same `Arena` — that wraps/holds an `Arena` over
those pages; its release (`munmap`) **consumes** it, so it is a linear owning value
enforced by the linearity checker (Memory §5.9 / MEM-2). Mixing a handle with the wrong arena
is a logic error, not UB (the access stays in-bounds of a live arena); an arena MAY carry
an identity tag to promote it to a trap (additive). The `generational` and `manual`
**mechanisms** are v1 protocol cases of the fixed menu (the `Mechanism` enum, §5.1; Memory
§5.2); what is **additive** is their concrete **provider surfaces** (the `gen` / `manual`
allocators), specified when first built (MEM-4 fixes the shared shape).

---

### 6. alloc-tier types

Available **only given an allocator** (Stdlib §1); forbidden under `no_alloc`. Each is
parameterized by an allocator (via `@alloc` on its storage), and its growth cost is
visible (I2).

- **`Vec(T)`** — a growable contiguous sequence. v1 operations: `new` / `with_capacity(n)`;
  `len` / `capacity` / `is_empty`; `push(v)` → `Result(usize, AllocError)` / `pop` → `Option(T)`;
  `get(i)` → `Option(...)` / indexing `self[i]` (bounds-checked, traps) / `set(i, v)`;
  sub-slice `self[a..b]` → `[T]`; `iter` (Iterator); `clear` / `truncate(n)` / `reserve(n)`.
  Reallocation cost is visible (I2). The indexing surface is the **index operator** (OP-3 /
  Type System §4.5): `self[i]` is `self.at(i)` (the bounds-checked trapping read — `Vec`'s
  established named read `at`, distinct from `get`'s `Option`; no separate `index` shim, OP-1),
  `self[i] = v` is `self.index_set(i, v)`, and `self[a..b]` is `self.index_range(a, b)`. So
  the `[]` notation is uniform with a built-in array's, dispatched to `Vec`'s own functions.
- **`String`** — a growable UTF-8 buffer (the alloc-tier counterpart of `str`). v1
  operations: `new` / `with_capacity(n)` / `from_str(s)`; `push(c: char)` / `push_str(s)`
  (each → `Result(usize, AllocError)`); `byte_len` / `is_empty`; `as_str` → `str` / `chars`
  (Iterator) / `clear`. Maintains the UTF-8 invariant.
- **`HashMap(K, V)`** — a hash map. v1 operations: `new` / `with_capacity(n)`;
  `insert(k, v)` → `Result(Option(V), AllocError)` / `get(k)` → `Option(...)` /
  `remove(k)` → `Option(V)` / `contains(k)` → `bool`; `len` / `is_empty` / `iter` / `clear`.
  Requires `K` to satisfy the **`Hash` and `Eq`** protocols (§2.6).

These operation sets are the **closed v1 surface** of each type; a further method is a
versioned addition (FND-6), not silent growth. A fallible operation returns `Result` (no
hidden failure, I11/I3); an allocation failure surfaces as `AllocError` (§5.1). Because the
language has **no unit type** (Grammar §3.1), a fallible mutator with no natural result
carries a **`usize` count** in its `Ok` arm — the new length for `push`/`push_str` (matching
the count convention of `write` → `Result(usize, …)` §4.2 and `StrBuf::push_*`); this fixes
the success type unambiguously for an independent implementation (FND-3).

---

### 7. std tier

Explicitly imported (Stdlib §1); forbidden under `freestanding`. The v1 std surface:

- **process** — `args(allocator) → [str]`; `env(allocator, name) → Option(str)` /
  `set_env(name, val)`; `exit(code)` (diverging, also §4.2); `abort()`. **`args`/`env` take an
  allocator** (STD-3): the OS delivers the command line / environment as a transient byte image, and
  returning owned `[str]`/`str` from it needs storage — under the no-implicit-allocator model (MEM-4)
  that storage is an explicit **allocator argument**, and the returned views are **region-backed**
  (MEM-1), valid for the allocator's extent (the caller owns and frees it, as for any container). The
  strings are *not* `@static`/process-static: that would need a lifetime outside MEM-1's menu and a
  fragile capture of the OS stack layout. `set_env` mutates an in-memory environment (a follow-up).
  **They require an entry that carries the state** (FN-12): the command line and environment are
  reachable only from the process entry, so `args`/`env` are available when the program's entry is
  `@abi(entry)` (the prologue records the state; ABI appendix §3.3) or when `startup = libc` (libc's
  start-up does). Calling them in a program whose entry is `@abi(naked)` under `startup = raw` is a
  **Semantic** diagnostic naming that entry — never a silent empty result.
- **I/O** — `stdin` / `stdout` / `stderr` byte streams (`read(buf)` / `write(buf)` /
  `flush()`; `stdout` / `stderr` and `File` satisfy the **`Writer`** protocol, §2.7);
  files: `open(path, mode) → Result(File, IoError)`, then `File.read(buf)` /
  `File.write(buf)` / `File.seek(pos)` / `File.close()`;
- **time** — `now() → Instant` (wall clock); `monotonic() → Instant` (steady, for
  durations); `Instant.elapsed() → Duration`. No hidden global clock.

All std calls that can fail return `Result(…, IoError)` (no hidden failure, I11/I3).
**`IoError`** is the error of the **std-tier byte streams** (`stdout`/`stderr`/`File`) when they
satisfy `Writer` (§2.7) — it is *not* the `Writer` protocol's fixed error: a lower-tier sink supplies
its own tier-appropriate error (an in-memory buffer fails with the base-tier `AllocError`; tier
discipline, Stdlib §1 / STD-2). `IoError` is a std-tier enum with a **closed v1 set**:

```
IoError := enum {
  NotFound, PermissionDenied, AlreadyExists, InvalidInput, UnexpectedEof,
  Interrupted, WouldBlock, BrokenPipe, TimedOut, ConnectionRefused,
  Other(i32),                              # any other failure; carries the raw OS code
}
```

The set is **closed in v1**: a failure not separately classified maps to **`Other(i32)`**
(the raw `errno` / `GetLastError`), **not** to a new variant — so an exhaustive `match`
stays valid across the frozen version (a new named variant would be a versioned, breaking
change — Overview §6; FND-6). `TimedOut` and `ConnectionRefused` are included ahead of the
networking surface (additive), though the v1 std surface is process / file / I/O / time only.

---

### 8. Conformance

A conforming implementation MUST:

1. provide the **core protocols** of §2 (`Truthy`, `Tryable`, `Optional`, `Iterator`,
   `Allocator`, `Eq`/`Ord`/`Hash`, `Writer`) as structural comptime predicates (Comptime
   §6.2) — make `bool` truthy, `Option`/`Result` tryable, ranges/slices/arrays iterable,
   the std byte streams writers (§2.7), **derive `eq`/`lt`/`hash` structurally** over
   `typeinfo`, overridable per type, and provide the v1 `Ord` consumers `min` / `max` /
   `clamp` and in-place slice `sort` (§2.6/§3.5);
2. provide the **base-tier types** of §3 — `bool`, `char`, `Option(T)`, `Result(T, E)`,
   slice `[T]`, `str`, and **`Never`** (the bottom/divergence type, CF-7) — with the
   enumerated operations, **bounds/UTF-8/validity checks trapping** as defined failures
   (never UB, I11), and visible cost (I2);
3. provide the **built-in catalog** of §4 with each built-in's signature and contract —
   compiler intrinsics (§4.1) and ordinary prelude functions (§4.2), with `panic`/`exit`
   returning **`Never`** and `assert` per Stdlib §4.3 (always emitted at runtime;
   compile error in a comptime context) — and honor the **per-domain families** (§4.3)
   normative in their feature chapters (Assembly `asm`/`mem`/instructions; Concurrency
   atomics/fence/`volatile`/overflow ops), the catalog being **partitioned** between
   this appendix and those chapters, not duplicated;
4. accept any value satisfying the **allocator protocol** of §5 as `@alloc(value)`,
   represent references per its declared lifetime mechanism (region/generational/manual),
   require an `unchecked` grant for `manual`-mechanism allocation/reference use (rejected
   under `no_unchecked`), keep `scoped` non-allocating, and forbid `@alloc` under `no_alloc`;
5. provide the **alloc-tier** types of §6 only given an allocator (forbidden under
   `no_alloc`) and the **std** surface of §7 only when imported (forbidden under
   `freestanding`), with all fallible std operations returning `Result` (no hidden
   failure);
6. treat every definition here as **required v1 content** — additive (FND-6) only for
   future growth, not optional (Stdlib §6; Overview §6) — and charge **zero binary
   cost** for unused prelude items (monomorphization + the normative reachability rule of
   Modules §6.4; a linker GC pass is additional, not contractual — Stdlib §2).
