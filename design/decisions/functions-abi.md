# Decisions — Functions, parameters, ABI

**Why the language is the way it is** for declarations, functions, their results and
parameters, function-values, overloading, foreign linkage, and the ABI. Each entry records
the *rationale* and the *rejected alternatives*; the normative *what* lives in the spec
(linked per entry), the *how* in the compiler. IDs are theme-prefixed (`FN-N`) and
**append-only** — assigned in articulation order, never renumbered. The `provenance:` line
records what an entry refines and where it lands in `spec/`.

---

## Declarations

### FN-1. One declaration form — types and functions are values
*spec: Declarations §1*

**Why.** Because types and functions are themselves values (a type is a comptime value, a
function is a code-pointer value), declarations need no per-kind keyword. There is a single
form `[modifiers] name : type = value` (with `name := value` for inference and `name : type`
for the no-initializer case). `struct{…}` / `enum{…}` / `union{…}` / `fn(…){…}` are
**value-expressions** — literals of types and functions — placed on the right-hand side, so
a binding to a type, an alias/brand, a function, a generic type (a type-returning function),
or a constant is all the *same* mechanism. There is no special declaration grammar: one
kernel, minimum keywords.

**Rejected.** Variant 2 — sugar keywords (`fn` / `struct` / `let` / `const` / `type`): a
clean kernel was preferred, and any such sugar is purely *additive*, not load-bearing.

### FN-2. Scope, shadowing, order — clarity over convenience
*spec: Declarations §6*

**Why.** Names obey lexical block scope (block, function, module/file, comptime block), an
inner scope sees the outer. Cross-scope **shadowing is allowed** (an inner name shadows an
outer within its bounds), but **same-scope re-declaration is forbidden** — there must be no
duplicate names and no implicit winner. There is no separate `const`: a "constant" is just a
`comptime`-binding, because phase (comptime) and mutability (immutable-by-default) are
orthogonal axes — keeping keywords to a minimum. Declaration **order** splits by level:
module/top-level is **order-independent** (mutual recursion without forward declarations);
local/in-block is **order-dependent** (visible from the point of declaration, which definite
assignment needs).

**Rejected.** Same-scope shadowing (Rust-style) — rejected for the sake of clarity.

### FN-3. Declaration placement + export-name mirroring
*spec: Declarations §6–§7, Functions §1; Types §8.1*

**Why.** Module level and function level admit different things, and the split follows from
what each can safely hold. At **module level** (order-independent, exportable via `pub`):
bindings of *values* — types, functions, naked-functions (code with no frame/ABI/return, the
entry-point / pure-asm case that replaced the former `label{}` blocks), modules,
comptime-constants, ABI values, import/re-export bindings, and static data initialized
**only by comptime constants** (or uninit/zero). **Forbidden at module level:** free
statements and runtime-computed static initialization — to avoid the static-init-order
fiasco and to keep nothing hidden; runtime initialization of globals is done by explicit
init-functions. Modules may be declared only at module level; types/functions/constants/code
may appear at both. **Name mirroring:** the exported symbol is the *spelling from the
declaration* qualified by the module path (`::`→`__`, C-compatible), so the linker name is
predictable from source; overloads add the deterministic signature suffix (FN-7). A library's
`pub` surface is auto-exported; non-`pub` names need not become symbols. `@export` /
`@export("name")` give an exact linker name for entry points and C-interop.

A checked module-level `@require` construction follows that same rule (TYP-12): its
predicate executes once in the comptime evaluator, false is a Semantic diagnostic, and
true contributes only the resulting static bytes, never a runtime initializer.

**Rejected.** A separate `@symbol(name)` attribute for the linker name — folded into
`@export` / `@export("name")` (one attribute carries both the export and its exact name).

---

## Functions, parameters, results

### FN-4. A function's result — one output-place model, `-> T` or named `out`
*spec: Functions §3, §4*

**Why.** A result is *not* a second mechanism beside parameters — it is the same
**output-place** model (places + mutability + pointers + ABI). Parameters carry directions:
`in` (default input), `out` (a place the callee writes), `in out` (the caller's place read
and written in place). Large aggregates, multiple results, and in-place update are therefore
all **explicit**, with no hidden return-via-pointer. The result then has exactly two surface
spellings of that one model, never both at once: **`-> T`** — the single anonymous projected
output, the convenient surface for the common one-result case (`r := add(2,3)`,
body-expression, `return expr` all fill it); it excludes named `out` but composes with
`in`/`in out` so the parameter list stays **uniform** (every entry is `name : type`). And
**named `out r : T`** (one or more) — the same output-place with source-visible names,
written by assignment and projected by signature order into a destructuring or onto caller
places. Composition `f(g(x))` works via that projection. One model, assembler-honest,
composable.

**Rejected.** An anonymous `out T` mixed into the parameter list (the earlier form) — it
made the list non-uniform (a bare `out T` beside `name : T`) and forced an `out T` vs
`out r : T` disambiguation; `-> T` is cleaner for the single anonymous case and leaves `out`
for the named/multiple/in-place cases. A *second semantic result mechanism* — rejected:
`-> T` is only the anonymous-projected spelling of the same output-place model.

### FN-5. Functions / ABI core — ABI is a value; modes, defaults, named args, variadics
*spec: Functions §4, §5, §6; Types §8.1; ABI appendix (150)*

**Why.** Four design choices keep the call surface transparent and the cost visible.
**(1) ABI is a comptime value** — a description of the convention (argument/return
registers, who saves, alignment), defaulted from the target/env, overridable per-site with
`@abi(value)`, and user-definable as ordinary struct construction over the appendix ABI
schema; the prelude supplies `c`/`sysv`/`syscall`/… as values. Treating the ABI as data, not
a fixed compiler fact, is what lets conventions be specified and extended without guessing.
**(2) Passing modes are a documented per-target lowering, not a hidden mechanism** — a small
`in` goes by-value in a register, a large aggregate by-ref to a copy; `out`/`in out` go
by-ref to the caller's place. The value is *logically* by-value; an explicit by-ref is just a
pointer parameter, so there is no separate "mode modifier". **(3) Defaults and named
arguments reuse existing mechanisms** — `in`-parameter defaults are visible in the signature
(the omitted expression is supplied at the call site); named arguments use the *same* `=`
label mechanism as struct construction, giving order-independence that combines with
defaults; and argument expressions **evaluate left-to-right in call-site source order** so
that named→positional reordering affects binding, not evaluation (nothing hidden). **(4)
Variadics** are a single trailing-rest parameter, with the comptime-variadic (`args : ...`,
a monomorphized heterogeneous comptime tuple) as the main form, a typed rest (`args : ...T`)
for the homogeneous runtime-slice case, and the C-variadic reserved for FFI under an
`@abi(c)` context (and forbidden in checked code).

TYP-12 applies the ordinary `in` rule literally to a checked `@require` predicate while
preserving the constructor result: one distinct logical by-value argument, classified as
usual (large aggregate = one caller-owned copy). An owning immediate `U` is rejected
rather than copied or consumed.

### FN-6. Function-values and closures — zero-cost-or-explicit, three levels
*spec: Functions §1, §7; Memory §6; Types §8.1*

**Why.** A function should cost what it looks like it costs, and any captured environment
must be visible — never a hidden box. So a non-capturing function is simply a code-pointer
(pointer width, its type spelled `fn(T…)->R` — FN-10), a zero-cost indirect call, and capturing is graded into three levels, each
either zero-cost or explicit: (1) the bare code-pointer; (2) a **static closure** — a value
of a concrete type (an anonymous aggregate of captures plus a known function) whose
environment lives where declared (stack/region, visible, no hidden allocation), called
directly/inlined, and monomorphized by generics → zero-cost with no `dyn` until asked for;
(3) **type-erased `dyn`** — opt-in, a fat value (code-pointer + environment-pointer) whose
environment lives in *explicit* storage (region/arena/heap), an indirect call with visible
cost. Captures are by value by default (a copy into the environment, so the closure is
independent); an explicit reference capture makes the closure scoped and unable to escape.
"Callable" as a generic constraint is just a comptime-structural predicate — no special
function trait.

For an `@require` predicate (TYP-12), that callable constraint is exactly the
post-specialization interface `fn(in U) -> bool`: naming is orthogonal to representation,
so both non-capturing function values and static closures are admitted whether named or
inline, but every capture must be comptime-known and specialized away. Runtime captures
and `dyn fn` are rejected, leaving no hidden environment argument.

### FN-7. Overloading by signature — the minimal carve-out protocols need
*spec: Functions §1.4, §10; Declarations §6.2*

**Why.** A function is a value bound by name, and same-scope re-declaration is normally an
error — but a *protocol realized by free functions* needs each participating type to supply
its own implementation under a **shared name** (the iterator protocol's `iter`/`next` per
type, a type's structural `eq`/`hash`). With one-binding-per-name a second `iter` is both a
redefinition error and, worse, a symbol collision (the symbol is the name verbatim). So a
name **may** bind several function values in one scope when they differ in their ordered
value-parameter signature — an overload set, the *only* exception to one-binding-per-name. A
call resolves by **argument types only** (results/`out` do not participate, so a call is
resolved without knowing its result type), with **exactly one** match required; each overload
lowers to a **distinct symbol** mangled by the parameter signature — transparently, with no
runtime dispatch, exactly as monomorphization mints per-instance symbols. UFCS method
dispatch is the same resolution keyed on the receiver. A name may also combine **one generic
definition** with concrete overloads (the generic-default + concrete-override pattern the
structural derives need): resolution prefers the *more specific* — a concrete match wins, and
only on concrete-miss does the generic apply (monomorphized at the inferred type). That is
the *only* mixing allowed; two same-concrete-signature functions, or two generic definitions,
remain forbidden.

**Rejected.** Overloading on the **result** type — a call could not be resolved in isolation,
breaking left-to-right typing. An explicit per-type **dispatch-table** value — redundant with
the symbol the compiler already mangles, and not zero-cost at the call site. A new
**`overload` keyword/attribute** — the signature distinction is already visible, so a marker
only adds ceremony.

---

## Foreign linkage and ABI

### FN-8. FFI — importing external symbols via `@extern`
*spec: Functions §6; ABI appendix (150)*

**Why.** Calling foreign code needs only the symbol's contract, not its body. So an
`@extern` declaration (a linkage-cluster *attribute*, not a keyword) gives a signature plus
ABI with no body — telling the compiler the symbol is external, to be called with this
ABI/signature and resolved at link time. The external name defaults to the declared name
(by the same mirroring as FN-3) or is set exactly with `@extern("name")` for differing or
foreign-mangled (e.g. C++) symbols; `@export("…")` is reserved for *our* exports, never for
import, keeping the two directions distinct. `@abi(c)` is the common denominator of FFI —
other languages are reached through the C ABI, C++ via mangled-name import. Linking itself
(static/dynamic libraries, paths, flags) is driven by the **manifest** — where the
static-vs-dynamic choice is a first-class **per-library link mode** (MOD-9), not an opaque
flag — and relocation/TLS
operands appear at the asm level as per-arch UFCS decorators (`sym.plt()`, `sym.gotpcrel()`,
…) that *lower to* the GAS spellings rather than being surface syntax — so there is no
`@`/`:`/`%` in the source. Header/bindgen import is left to future tooling.

### FN-9. ABI per-target — the convention is data, total over supported targets
*spec: Functions §6; ABI appendix (150)*

**Why.** The ABI-as-value choice (FN-5) is only implementable without guessing if the
*data* it reads is fully pinned. So the appendix fixes the **`Abi` schema** (the argument/
return registers, callee/caller-saved sets, stack alignment and growth direction, red zone,
shadow space, aggregate-by-value cutoff, the syscall number register, varargs variant, …) —
data against which the passing modes of FN-5 are lowered — and the **default-ABI as a total
function of `arch+os+env+features`** in three steps (per-arch base, container/OS exceptions,
the FP subvariant from env/features). Totality is deliberately scoped to the **supported
targets** (the canonical triples), not the whole Cartesian product: an unsupported
combination is a Config diagnostic, never an improvised convention. `@abi(…)` takes an ABI
*value by name* — **string shortcuts are rejected** (strings remain only for external/linker
symbol names like `@extern("…")`/`@export("…")`), and the selector is `Conv(Abi) | Naked`,
where `naked` is a *sentinel* "no-lowering" mode (not a convention, fills no schema). The
schema carries defaults so per-target tables list only differences. The **syscall ABI** is a
bodyless declaration (the trap is the implementation, like `@extern`, and only under
`unchecked`) with a fixed positional parameter→register mapping and register-width
integer/pointer operands only — its result is `isize` because the Linux convention returns
**`−errno`** in the result register, so a signed register-width value is the natural carrier;
it is specified per-arch for Linux plus BSD/macOS, while
Windows (no stable public syscall ABI) and freestanding/unlisted OSes reject it — further
OSes being additive.

---

## Function-value types and type erasure

### FN-10. The function-value type — `fn(T…) -> R` (a code-pointer type)
*spec: Functions §1.5; Grammar §3.3, §3.6*

**Why.** FN-6 graded function values into three representations — bare code-pointer,
static closure, `dyn` — and Functions §1.2 fixed the non-capturing case as a pointer-width
**code pointer**. But the grammar had **no type spelling** for a function value:
`type-atom` admitted `ident` / `path` / the aggregate literals / `T(...)` / `ptr(...)` /
`brand(...)`, and `fn` was a **value** introducer only (`fn-value ::= fn-sig block`). So a
program could not annotate a binding, parameter, or struct/array/tuple field as "a function
of signature S" — a callable could travel only through generic monomorphization (a
`T : type` parameter). That blocks **storing** a function value in a data structure, and it
is the missing prerequisite for the type-erased `dyn` (FN-11), which must name a function
signature to erase over. This entry adds the type (a FND-3 gap: a representation was fixed
with no way to write it).

The type of a **non-capturing** function or non-capturing lambda is written **`fn(T0, T1,
…) -> R`** in **type position** — the bodyless signature, with no `{ … }` block (which is
exactly what distinguishes it from the `fn(…){…}` value-expression of §1.1). Its
**representation is one machine word — a code address** (I1/I2): a value of this type can
be **bound, passed, returned, stored** (in structs, arrays, tuples), and **called**
(`f(args)` = one indirect call), with **no** environment and **no** allocation. Taking a
top-level function or a non-capturing lambda yields such a value. It fits the one-language /
no-magic model: an ordinary type (no tier), `@`-free (it is a type, not an attribute),
visible cost (one word + one indirect call).

Sub-choices (resolved the Alatyr-consistent way — FND-3, no guessing):
- **Parameter directions are part of the type.** A parameter entry is an optional direction
  (`in` default / `out` / `in out`) followed by a **type**, with **no name** (a type needs
  no parameter names): `fn(u64, u64) -> u64` is the all-`in` common case,
  `fn(u64, out u64)` types a function with a named-`out` result. Two function values share
  a type only when their directions, parameter types, and result all match — directions
  drive the ABI (FN-4/FN-5), so they belong to the type.
- **`-> R` is optional** — present projects an anonymous output `R`; absent is a
  **procedure** (no result), mirroring the optional `->` on a `fn-sig` (Functions §3.5). A
  named-`out` result is expressed by an `out`-directed parameter entry.
- **This is not the generic *callable* constraint.** `fn(T…)->R` is a concrete one-word
  code-pointer *type*; the generic-`callable` predicate (FN-6) is a comptime structural
  check that also admits **static closures** for monomorphization. The two coexist — the
  fn-type stores a homogeneous code pointer, the predicate abstracts over any callable.

**Rejected.** Reusing the named `fn-sig` (with parameter *names*) as the type — a type has
no use for names, and printing them invites the reader to think they are significant. An
`@`-attribute spelling — `@` is **compile-time attributes only** and never denotes a
runtime representation (the code-pointer word *is* a representation), so a keyword-headed
type form is the correct vehicle. A pointer-to-function spelling `ptr(fn …)` — a function
value already **is** a code address (Memory §4 / §1.2), and wrapping it in `ptr(T)` would
imply a further data-dereference that code does not have (Types §7, the raw-code-address
discussion).

### FN-11. The `dyn` type-erased closure — `dyn fn(T…) -> R` over explicit storage
*spec: Functions §1.6; Memory §6.3, §5.3.1; Grammar §2.3, §3.3, §3.6*

**Why.** FN-6 named the third function-value level — a **type-erased `dyn`**, a fat value
`{code, env}` that lets **different capturing closures share one runtime type** (uniform
storage, dynamic dispatch) — and Memory §6.3 / Functions §1.2 fixed its **representation**
(a code pointer + an environment pointer, environment in explicit storage, an indirect
call). But the **surface was undefined**: `dyn` was not a lexeme, the grammar had **no**
`dyn` type spelling, no construction form, and no call form — so §6.3 was a representation
clause an implementer could not realize without **inventing** syntax (a FND-3 gap that
blocked storing heterogeneous closures at all). This entry pins the surface, building on the
function-value type (FN-10).

**Type.** `dyn fn(T0, T1, …) -> R` — the **`dyn` keyword** prefixing a function-value type
(FN-10). Its representation is a **two-word fat pair** `{code, env}`: a code pointer plus a
pointer to the captured environment. Visible cost: two words + one indirect call. `dyn` is
a **keyword type-constructor** (like `ptr` / `struct`), **not** an `@`-attribute — it
changes the **runtime representation** (a fat pointer + dynamic dispatch), and `@` is
compile-time attributes only (never a runtime representation), so a keyword is the correct
vehicle.

**Explicit storage (I3).** The environment lives in **explicit, user-provided storage** — a
user place, a region/arena slot, or an allocation the `dyn` value **borrows** — **never** an
implicit heap box (this deliberately differs from a `Box<dyn>`-style hidden allocation). The
`dyn` value's validity is **bounded by that storage's extent** and enforced by the
**existing escape / second-class machinery** (Memory §5.3.1): the `dyn` is a borrow of its
env storage and MUST NOT escape it — a **stack** env makes the `dyn` scoped to the frame; an
**arena/allocation** env lets it live as long as that region (and be stored where the region
outlives). No new lifetime discipline is introduced.

**Construction — `dyn_over`.** A prelude word-function (a builtin, OP-1; the name echoes
`arena_over`, MEM-3 — "*X* over caller-provided storage"): `dyn_over(ptr(mut store))`, where
`store` is a **named place** holding a **static closure** (Memory §6.2 — a concrete
aggregate of captures + the known function). It yields the fat pair
`{code = <the adapter monomorphized for `store`'s concrete type>, env = ptr(store)}`, typed
`dyn <store's call signature>` and assignable to a compatible `dyn fn(T…)->R` binding. The
**storage is explicit in the syntax** (the named place and its address `ptr(mut store)`), so
the environment's location and lifetime are recoverable from the closed package
(I3 / MEM-5's closed-package reading). The `code` adapter is a compiler-synthesized
monomorphic wrapper — a documented, transparent lowering (I1); the env-pointer mutability
flows from the `ptr(mut …)` / `ptr(…)` form.

**Call — `d(args)`.** The ordinary postfix call (Grammar §3.4), no new form: for a
`dyn`-typed callee it lowers to an **indirect call through `code`**, passing `env` as the
leading environment pointer to the adapter — two-word cost, one indirect call, nothing
hidden (I3).

Fits: I3 (explicit storage, no hidden box), one language (an ordinary type, no tier),
visible cost. **Resolves the §6.3 SPEC-BLOCKED gap** (representation was decided, surface was
not).

**Rejected.** A `Box<dyn>`-style **implicit heap allocation** of the environment — hides an
allocation (I3) and forces an allocator where none is wanted. A `dyn(fn(A)->B)` prelude
**type-function** (parsed via `type-atom "(" args ")"`) instead of a `dyn`-keyword prefix —
it would read as an ordinary generic instantiation yet denote a distinct runtime
representation (fat vs thin), blurring the code-pointer/fat-pair distinction the cost model
must keep visible; the keyword makes the representation switch **loud**. An `@dyn` attribute
— `@` is compile-time attributes only, never a runtime representation. Deriving the env
storage implicitly (a hidden box, or a compiler-chosen arena) — against I3; the storage is
named by the programmer.

### FN-12. `@abi(entry)` — the platform entry prologue is a selector, not a hidden crt0
*provenance: refines FN-5/FN-9 (ABI is a value; per-target conventions) and TOOL-12 (`startup =
raw` owns the process entry) · spec: ABI appendix §3.1, §3.3; Manifest appendix §3.2; Stdlib
appendix §7*

**Why.** Under the default `startup = raw` (TOOL-12) nothing runs before the entry function, and the
per-platform entry contracts do not resemble each other: on ELF the kernel leaves `argc`, `argv`, `envp`
and `auxv` **on the stack** and there is nowhere to return to; on PE the loader passes **nothing** and the
command line must be fetched (`GetCommandLineW`) while exit goes through `ExitProcess`; on Mach-O `dyld`
calls the entry **like a function** with `argc, argv, envp, apple[]` in registers and uses its return value
as the exit status. So "write `_start` yourself" means writing three different programs, and the library's
`args`/`env` (Stdlib appendix §7) are unimplementable in the general case because a `naked` entry drops the
state that carries them. The alternative — a stdlib-supplied crt0 linked in silently — is exactly the hidden
code I3 forbids, and it would make every program pay for a runtime it may not want.

**The rule.** A third **ABI selector** joins `Conv(abi)` and `Naked`:

```alatyr
AbiSelector := enum { Conv( Abi ), Naked, Entry }
```

`@abi(entry)` marks a function as **the process entry** and makes the compiler emit the **per-target entry
prologue/epilogue** enumerated normatively in the ABI appendix (§3.3): recover the platform's entry state,
record it where `args`/`env` can read it, establish a call-ready frame, call the body, and terminate the
process with its result. The function is written as an **ordinary** function that declares **no
parameters** — `fn()` or `fn() -> i32`, whose result is the process exit status — because the entry state is
a per-target fact the prologue captures rather than an argument list that would differ per platform. One
source form therefore works across ELF, PE and Mach-O, and what the prologue does is documented per target
rather than inferred (I1: the cost and the shape are predictable from the source plus the table). What it
records is a **named, documented static**, emitted only when `args`/`env` are reachable, so the cost is
visible and zero when unused (I2/I3).

The selector is **opt-in and visible at the declaration**: `@abi(naked)` remains available and remains the
right choice for freestanding entries, where the reset vector, the stack pointer and the linker script are
the programmer's business and no generic prologue could be correct. Because `@abi(entry)` is what saves the
entry state, `args`/`env` are available exactly when the program's entry carries it — an `@abi(entry)` entry
or `startup = libc`; calling them otherwise is a Semantic diagnostic naming the entry, not a silent empty
result.

**Rejected.** A stdlib-linked crt0 that calls a magic `main` — hidden code in the artifact (I3), and it
forces a runtime on programs that want none; it also re-creates the `_start`/`main` name coupling TOOL-12
removed. Leaving every entry `naked` and dropping `args`/`env` to additive — the platform state is available
only at entry, so dropping the prologue drops the capability permanently, and each program would re-derive
three platform contracts by hand. A dedicated `@entry` **attribute** instead of an ABI selector — the thing
being selected *is* a lowering mode for a function's frame and return, which is precisely what `@abi`
already selects (`naked` is one); a second attribute for the same axis is the duplication OP-1 rejects.
