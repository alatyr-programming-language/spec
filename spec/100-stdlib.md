# Alatyr Language Specification

## Chapter — Standard Library and Built-ins

> **Status — draft (under review).** This chapter is a draft and is not yet
> accepted; the specification as a whole is under development. See the status note
> at the top of the Overview chapter.

This chapter defines the **prelude tiers** (base / alloc / std), automatic **dead-code
elimination**, the **built-ins** (compiler intrinsics vs ordinary prelude functions),
the **`panic` / `exit` / `assert`** runtime failure operations (ordinary prelude
functions, §4 — not compiler intrinsics), and **allocators** (`@alloc`).
It fixes the **tiering**, the **built-in catalog structure**, and the **runtime-builtin
semantics**; the concrete library type/function **definitions** are **v1 content
delivered as the Stdlib appendix** (§6) — the same rule as the per-arch
tables (Assembly §intro): required for v1, "additive" (FND-6) only for future growth.

It builds on the Type-System chapter (`Option`/`Result`/slice/`str`/`bool`/`char` are
prelude types built by the machinery), the Comptime chapter (monomorphization,
intrinsics, protocols), the Memory chapter (allocators, `@alloc`, lifetime
mechanisms), and the Modules chapter (the prelude is auto-available). It introduces
**no** new surface syntax. Requirement keywords follow RFC 2119 (Overview §6).

---

### 1. Prelude tiers

The prelude is **tiered**, mirroring the freestanding → allocating → hosted spectrum
(like Rust's core / alloc / std), so the language fits the kernel/firmware niche:

- **Base prelude** — **freestanding, no allocation**, auto-available everywhere:
  `bitsN`, the numeric interpretations `u`/`i`/`f`, `usize`/`isize`, `bool`, `char`,
  `Option` / `Result`, the slice `[T]`, `str`, `Never` (the bottom/divergence type,
  CF-7), the core built-ins (§3), and the base operators. (A code address is a
  pointer-width `bitsN`, not a dedicated type;
  decisions CF-5.) No allocator is required.
- **alloc tier** — available **only when an allocator is present** (a hosted target,
  or a freestanding one with an explicit allocator): `Vec`, `HashMap`, `String`, and
  the like — anything that allocates.
- **std tier** — **explicitly imported**: I/O and other OS-facing facilities.

**How the tiers are rooted (the import surface).** All three tiers are
**compiler-provided** — they are part of the language implementation, **not** manifest
dependencies (Modules §8 governs *external* path/git packages; the stdlib is neither).
They differ only in **how their names enter scope**:

- the **base** prelude is injected **unqualified** (auto-available, above);
- the **`alloc`** and **`std`** tiers are provided as **ambient root-level modules**
  named exactly **`alloc`** and **`std`** (alongside the package root, sharing the one
  module tree, Modules §1). "**Explicitly imported**" means their names are **not**
  injected unqualified the way base is: a program reaches them **qualified**
  (`std::io::stdout`, `alloc::Vec`) or binds a local name with the ordinary import form
  (`out := std::io::stdout`, Modules §4.1). There is no `use` and no new syntax — an
  import is the declaration `name := path` (Modules §4), the path now rooted at the
  ambient `alloc`/`std`.

The tiers respect **limits** (Overview §3): `no_alloc` forbids the alloc tier;
`freestanding` forbids the std/OS facilities. A name from a higher tier than the
unit's limits allow **does not exist** for that unit — the ambient `alloc` root is
absent unless an allocator is present (and is removed entirely by `no_alloc`); the
ambient `std` root is absent under `freestanding`. (Being absent, a reference to it is
an ordinary unresolved-name error — there is no special-cased tier diagnostic.)

**Tier discipline (no inversion).** A stdlib item's **public API** — its signatures and
the types named there, **including a protocol implementation** — references **only** types
of its **own tier or below** (base < alloc < std). A higher-tier type appearing in a
lower-tier API is a **tier inversion** and is **ill-formed**. This is the structural reason
a tier's absence under a limit is sound: a *present* tier's surface never names an *absent*
one, so removing the std (or alloc) root by a limit can never strand a name that remains.
A protocol whose v1 shape would otherwise fix a higher-tier error type therefore lets the
**implementer** supply the error of its own tier — e.g. the `Writer` byte sink (appendix
§2.7): the std streams fail with the std-tier `IoError`, an in-memory buffer with the
base-tier `AllocError` (STD-2).

---

### 2. Automatic dead-code elimination

Being in the prelude has **zero binary cost**: an unused item never reaches the
binary. This holds via three mechanisms already defined elsewhere:

- **Monomorphization** — an uninstantiated generic emits **no** code (Comptime §3.4);
- **Reachability emission** — an artifact holds only what its **roots** reach, per the
  artifact-`kind` rule (Modules §6.4): the roots and the reachability closure are
  normative, so an unused prelude item is absent by specification, not by luck;
- **Linker GC** — `ld --gc-sections` may additionally drop sections nothing references
  (Codegen §6; CG-1). This is a **belt-and-braces** strip, **not** part of the emission
  contract: an artifact's specified contents never depend on it (Modules §6.4), and it
  can only remove what the reachability rule already left unreferenced.

So the **only** cost of a large prelude is namespace surface, **not** binary size — a
freestanding program pays nothing for the parts of the prelude it does not use.

---

### 3. Built-ins

Built-ins are **prelude identifiers**, not keywords (Comptime §6; the `@` sigil is
**not** used for them — `@` is only attributes, decisions SYN-7). There are **two
kinds**:

- **Compiler intrinsics** — cannot be written in the language; the compiler provides
  them: `typeinfo`, `bitcast`, `T.size()` / `T.align()` (Memory-model
  §4.3; field offsets via `typeinfo(T).fields`), `ptr` / `deref`, `resolves` / `compiles` (Comptime §6.1), `embed`
  (Comptime §2.4), `forget` (Memory §5.9), and the instruction intrinsics
  (Assembly chapter).
- **Ordinary prelude functions** — could be written in the language but are provided:
  `panic`, `exit`, `assert` (§4), and the library operations.

The built-in **catalog** is **v1 content** (§6): fully specified for v1, additive for
future growth (FND-6) — not left unspecified.

---

### 4. `panic` / `exit` / `assert`

#### 4.1 Defined failure, not UB

`panic` and a `trap` are a **defined failure** (a controlled termination), never UB
(I11). There is **no** stack unwinding (cleanup is `defer` on normal exits only;
Memory §5.8) — a panic/trap terminates the process.

#### 4.2 Hosted versus freestanding

The behavior of `panic` / `exit` is a **target/limits property** (configuration; manifest):

- **hosted** — `panic` writes a message to stderr and the process exits; `exit(code)`
  is the OS exit;
- **freestanding** — `panic` invokes a **programmer-supplied hook**, or, with none, a
  **trap instruction** (the per-architecture appendix §2 checked-failure instruction:
  `ud2` / `brk #0` / `udf #0` / `ebreak`); there are **no** OS calls.

`exit` is **not** a compiler builtin: like `assert` (§4.3) it is an **ordinary prelude
function** — on a **hosted** target a thin wrapper over the OS exit primitive
(`exit := fn(in code : usize) -> Never { unchecked exit_group(code) }` over an
`@abi(syscall)` declaration — the **process**-wide exit call, Linux `exit_group`, not the
thread-local `exit`, or a multi-threaded program would not terminate; the syscall
**numbers** are per-OS/arch data the programmer supplies, not fixed by this
specification, ABI appendix §5 / FN-9), on **freestanding** the programmer hook or trap.
Its result is `Never` (§3.7), so a call is a flow terminator. The compiler keeps **no**
built-in knowledge of process exit — consistent with the kernel/prelude split (the kernel
carries no built-in operations either; TYP-2/TYP-5).

**Hosted does not mean libc.** "Hosted" here is a property of the **target** (`os` is not
`Os.none`), not of the link: the default `startup = raw` links **no** libc and **no**
platform start-up objects (TOOL-12; Manifest appendix §3.2), while `startup = libc` is the
opt-in that links libc and lets its start-up own the process entry.

Under `raw` the **OS primitive** the hosted `exit`/`panic` path uses is **per container**
(ABI appendix §3.3): on **ELF** it is the exit syscall (`@abi(syscall)`, ABI appendix §5);
on **PE** and **Mach-O** there is no usable syscall ABI — `@abi(syscall)` is rejected there
— so the primitive is the platform library's (`kernel32::ExitProcess`, `libSystem::exit`),
which the toolchain links **dynamically** by the fixed table of ABI appendix §3.3, with the
usual non-hermetic Config note (Tooling §2.5). So `exit`/`panic` are defined on every
supported hosted triple, and what they cost is visible per target.

#### 4.3 `assert`

`assert(cond)` is an ordinary prelude function. At **run time** it lowers to `panic`
when `cond` is false. Evaluated in a **comptime context** — e.g. inside a
`comptime { … }` block, or in a comptime function (Comptime §8.1) — it is a
**compile-time** assertion: a false `cond` is a **compile error** (the comptime
evaluation fails; there is no runtime `panic`). There is **no** `comptime assert`
keyword-form — it is the same `assert`, evaluated at compile time by its context. Both
outcomes are defined failures (I11), never UB.

A runtime `assert` is **always emitted** — there is **no** hidden, build-mode strip
(no `NDEBUG`-style removal): silently dropping it would change behavior (a false `cond`
would continue past a violated invariant instead of trapping), which is **semantic**
and therefore not a configurable build policy (I3; decisions FND-7). A **debug-only**
check is gated **explicitly** with a comptime build flag —
`comptime if build.debug { assert(expensive_invariant(x)) }` (Comptime §8.2; the
`build.*` flags are the manifest's, Tooling chapter) — visible in the source, not a
silent mode. (So no `debug_assert` is needed either: `comptime if` + a flag expresses
it.)

---

### 5. Allocators — `@alloc`

#### 5.1 Allocation is selected by `@alloc(value)`

Dynamic allocation is requested with the attribute **`@alloc(allocator-value)`**
(Memory §2.4): the **allocator is a value** (Declarations §1), and its **type
carries the lifetime mechanism** — the reference representation follows from it
(Memory §5.2). This parallels `@abi(value)` (Functions §6): one extensible,
value-parameterized attribute.

#### 5.2 Unified and extensible

`@alloc(value)` **unifies** the earlier `@heap` / `@region` / `@generational` into one
attribute (decisions STD-1). A **user allocator just works**: any value of an allocator
type is a valid `@alloc(…)` argument. The **non-allocating** storage specifiers
remain separate: `@reg` / `@stack` / `@static` / `@scoped` (Memory §2.4).

#### 5.3 The allocator interface is a comptime protocol

What makes a type an allocator is a **comptime protocol** (Comptime §6.2) — a shape
(allocate / free / the lifetime-mechanism it provides) checked structurally; the
**exact protocol** is v1 content (§6). The `no_alloc` limit forbids `@alloc` entirely.

---

### 6. v1 content (the Stdlib appendix)

The **concrete definitions** are **normative v1 content** delivered as the
**Stdlib appendix** (the Prelude-and-Stdlib appendix, now drafted
and under review), **required** for v1. They include:

- the base-tier type operations — `Option` / `Result` (incl. the **tryable** protocol,
  Control Flow §8.2), slice `[T]`, `str` / text;
- the alloc-tier types — `Vec`, `HashMap`, `String`, …;
- the **allocator protocol** (§5.3) and the provided allocator surface(s);
- the **std** surface — I/O, etc.;
- the full **built-in catalog** (§3) with each built-in's signature and contract.

This chapter is normative for the **tiers** (§1), **DCE** (§2), the **built-in
structure** (§3), the **runtime-builtin semantics** (§4), and the **`@alloc` model**
(§5); the appendix enumerates the definitions. "**Additive**" (FND-6) governs only
*future* growth — it does **not** make the v1 definitions optional (Overview §6).

---

### 7. Conformance (normative summary)

A conforming implementation MUST:

1. provide the **base / alloc / std** prelude tiers as **compiler-provided** (not
   manifest dependencies, Modules §8): the base tier injected **unqualified**
   (auto-available, freestanding); the **`alloc`** and **`std`** tiers as **ambient
   root-level modules** of those names, reached **qualified** or via an ordinary
   `name := path` import (Modules §4.1) — "explicitly imported", not unqualified — with
   the `alloc` root present only given an allocator (removed by `no_alloc`) and the
   `std` root removed under `freestanding`, an absent root being an ordinary
   unresolved-name error (§1);
2. ensure **zero binary cost** for unused prelude items via monomorphization and the
   normative reachability rule (Modules §6.4) — a linker GC pass may strip further but is
   not part of the contract — so the only cost of breadth is namespace (§2);
3. provide the built-ins as **prelude identifiers** (not keywords; no `@`), split into
   compiler intrinsics (un-writable) and ordinary prelude functions, with the catalog
   as enumerated v1 content (§3, §6);
4. make `panic` / a trap a **defined failure** (never UB), with **no** unwinding;
   implement hosted (stderr + OS exit) versus freestanding (programmer hook or trap
   instruction, no OS calls) per the target/limits; and treat `assert(cond)` as lowering
   to `panic` at run time and as a compile error when evaluated in a comptime context
   (no `comptime assert` keyword-form), **always emitting** a runtime `assert` (no
   hidden build-mode strip — a debug-only check is an explicit `comptime if` on a build
   flag) (§4; I3/I11/FND-7);
5. select dynamic allocation **only** via `@alloc(allocator-value)` (allocator a value
   whose type carries the lifetime mechanism), accept user allocators satisfying the
   allocator protocol, keep `@reg`/`@stack`/`@static`/`@scoped` as the non-allocating
   storage, and forbid `@alloc` under `no_alloc` (§5);
6. treat the standard-library definitions (base operations, alloc-tier types, the
   allocator protocol, std, the built-in catalog) as **required v1 content** in the
   Stdlib appendix — additive only for future growth, not optional (§6;
   I10/FND-6/FND-3).
