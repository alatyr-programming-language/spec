# Alatyr Language Specification

## Chapter — Concurrency, Overflow, and Floating Point

> **Status — draft (under review).** This chapter is a draft and is not yet
> accepted; the specification as a whole is under development. See the status note
> at the top of the Overview chapter.

This chapter covers three cross-cutting areas where behavior must be **defined and
predictable** (I1/I11):

- **Concurrency** — the primitives (atomics, fences, `volatile`), the no-runtime
  stance, and how a programmer obtains data-race freedom **by construction**;
- **Integer overflow** — trap by default, wrap inside an `unchecked` scope,
  explicit policy operations;
- **Floating point** — deterministic comptime and runtime semantics.

It builds on the Memory chapter (`ptr`, linearity, scoped references), the
Type-System chapter (the `u`/`i`/`f` interpretations, `bool`), the Control-Flow
chapter (the `unchecked` grant), the Assembly chapter (atomic/fence instructions are
per-arch), and the Comptime chapter (comptime enum orderings, protocols).

The atomic / overflow-op / fence / `volatile` forms below are **ordinary builtin
calls** — no new surface syntax (their signatures are shown inline, §2–§6); the only
new named value is the comptime `Ordering` enum (§2). Requirement keywords follow
RFC 2119 (Overview §6).

---

## Part A — Concurrency

### 1. No built-in runtime

The language has **no** built-in threads, async runtime, or scheduler (I3): threads
come from the **OS via FFI**. The language provides **primitives** — atomics (§2),
fences (§3), and `volatile` (§4). `async` / `await` are **reserved keywords** with
**additive** machinery, and light threads (fibers) are an **additive library**
(decisions CC-6, CC-7); neither is built into v1.

### 2. Atomics — operation-based builtins

Atomic access is expressed by **builtins on a `ptr(mut T)`**, each taking a
**comptime memory-ordering** argument — **atomicity is a property of the operation,
not of a type**. There is **no** `Atomic(T)` type (consistent with §5: atomicity is
per-access discipline, not a static guarantee). The operations live in the **prelude
module `atomic`** (`atomic::load`, …); the UFCS form is `p.atomic::load(ord)` (qualified
UFCS, CT-10/MOD-3).

The v1 operation set:

```
atomic::load(p, ord)                                       # → T
atomic::store(p, v, ord)
atomic::swap(p, v, ord)                                    # → old T
atomic::cas_strong(p, expected, new, ok_ord, fail_ord)     # → (T, bool): (current value, succeeded?)
atomic::cas_weak(p, expected, new, ok_ord, fail_ord)       # → (T, bool); MAY fail spuriously
atomic::fetch_add(p, v, ord)                               # likewise sub / and / or / xor → old T
```

- **Orderings** are a comptime prelude enum `Ordering`: `relaxed`, `acquire`,
  `release`, `acq_rel`, `seq_cst`. The value MUST be comptime-known (it selects the
  instruction).
- **Legal orderings per operation** (otherwise a compile error): `atomic::load` takes
  `relaxed` / `acquire` / `seq_cst`; `atomic::store` takes `relaxed` / `release` /
  `seq_cst`; read-modify-write ops (`atomic::swap` / `atomic::fetch_*`) take any ordering;
  for CAS the **failure** ordering is one of `relaxed` / `acquire` / `seq_cst` and
  **MUST NOT be stronger than** the success ordering. (Same constraints as C11/C++/Rust.)
- **Compare-and-swap** has two named forms, both returning `(T, bool)` — the
  **current** value (so a failed CAS can retry) and whether the swap happened — and
  each takes a **success** and a **failure** ordering: **`atomic::cas_strong`** never
  fails spuriously; **`atomic::cas_weak`** MAY fail spuriously (cheaper on LL/SC
  architectures, for retry loops).
- **Atomic-capable widths/types are per-arch data** (the **table in the per-arch
  appendix §2**): typically up to the pointer width, plus a feature-gated wider one
  (e.g. `x86_64` adds 128-bit under `cx16`; RISC-V atomics require the `a` extension).
  An atomic op on a width unavailable for the selected target is a **compile error**
  (gate it with `when target…`).
- **Lowering**: atomic ops emit the target's atomic instructions + barriers —
  transparent (I1), zero-cost (I2). The op **set** is v1 content; the per-arch
  instruction mapping is per-arch data (Assembly chapter).

### 3. Fences

`fence(ord)` is a barrier builtin lowering to the architecture's fence instruction.
Fences are **explicit** (no implicit barriers are inserted). **Legal orderings**
(otherwise a compile error, as in §2): `fence` accepts `acquire` / `release` /
`acq_rel` / `seq_cst`; **`relaxed` is a compile error** — a relaxed fence has no effect.

### 4. `volatile` — operation-based, distinct from atomic

The **prelude module `volatile`** provides `volatile::load(p)` / `volatile::store(p, v)`
for **MMIO**: each access is emitted
**exactly once** and is **not** eliminated or reordered relative to other volatile
accesses. **`volatile` is NOT `atomic`** — it provides **no** inter-thread ordering
or atomicity, only "do not optimize away, do not reorder." (A common confusion: use
atomics for synchronization, `volatile` for device memory.)

A **volatile-qualified pointer type** — `ptr(volatile T)`, making every access through it volatile
**by construction** (a path qualifier, like `mut`; visible in the type, so I3 holds) — is **additive**
(FND-6): v1 uses the explicit `volatile::load` / `volatile::store` operations, and the qualifier's form is
fixed now so it can be added later without reworking the model. (With the optimizer present, `volatile`
is no longer redundant — the optimizer may touch ordinary accesses, never volatile ones; CG-5.)

### 5. Data races and race-freedom

#### 5.1 Races are hardware-defined, not UB

Data races are **not** statically prevented in general (no borrow checker; decisions
FND-9). A data race has **hardware-defined** behavior, **never** UB (I11): a torn or
stale value is a logic bug, not license for the compiler to miscompile unrelated code
(no UB to exploit; I11/CG-5).

**Optimizer contract for ordinary memory.** Ordinary (non-`atomic`, non-`volatile`)
loads and stores are compiled under **single-thread sequential semantics**: the
optimizer may reorder, eliminate, duplicate, hoist or sink them **subject only to
preserving the thread's observable behavior** (results, traps, and the order of
`volatile`/`atomic`/IO effects — CG-5; glossary). **Inter-thread ordering and visibility
are established only by `atomic`/`volatile`/`fence`** (§2–§4): the optimizer never
reorders *those* (§4) and treats them as the synchronization edges. Consequently a
**data race on ordinary memory** yields **hardware-defined** values (whatever the
emitted, possibly-optimized, accesses produce on the target) — never UB, and it **never
affects the compilation of race-free code**. The compiler makes **no aliasing
assumptions** (Memory §5.7): the latitude above comes from preserving single-thread
observable behavior, not from assuming distinct pointers never alias. Two conforming
implementations may differ in how they optimize ordinary accesses, but **not** in the
observable behavior of a **race-free** program (the FND-3 bar).

#### 5.2 Race-freedom by construction (three routes)

A programmer obtains **100%** race-freedom **by construction**, via three routes,
each **enforced by existing mechanisms** — no borrow checker needed (decisions CC-4):

- **share-nothing** — communicate by **moving** values through channels; **linearity**
  (Memory §5.9) gives exactly one owner (after a move the source is unusable), and
  **scoped references cannot escape** (Memory §5.3) so only self-contained owned data
  can be sent — there is no shared cell to race on;
- **immutable-share** — share only deeply **immutable** data (immutable by default,
  Memory §3); concurrent reads do not race;
- **`Mutex`-guarded** — `lock()` returns a **scoped reference** (cannot escape)
  released by `defer`; access is only while the lock is held, and the mutex
  serializes holders — no two writers at once.

The concrete **`Mutex(T)`** type is **additive** (it ships with the threading surface —
threads/tasks come from the OS via FFI, §1, and beyond the v1 primitives are deferred, like
`Send`/`Sync` in §5.4); v1 is single-threaded by default, so it has no standalone v1
definition. What v1 **fixes** is its **contract**, so two implementations agree when it lands
and so it is provably expressible as ordinary library code (no new language entity): `Mutex(T)`
**wraps a `T`** (the guarded datum, owned); `lock(in m : ptr(Mutex(T))) -> scoped ptr(mut T)`
acquires and yields a **scoped reference** to the inner `T` — second-class, so it **cannot
escape** the holding scope (Memory §5.3.1) — and the lock is **released by `defer`** at scope
exit; the mutex **serializes** holders. The scoped-reference + `defer` mechanisms are exactly
what makes this a library type, not a privileged one.

#### 5.3 The discipline point

What the compiler does **not** catch in v1 is **unsynchronized shared mutable state**
— notably a `mut` module-level global reachable from two threads. For 100%, route
every cross-thread mutable datum through an atomic or a `Mutex` (§5.2), and do **not**
drop to raw/`unchecked` for cross-thread access.

#### 5.4 `Send` / `Sync` (additive — a library pattern)

Thread-safety capabilities (`Send` / `Sync`) are **additive and library-level**, not in v1:
they are bounds on a thread/channel API, and **both that API and the predicates are ordinary
libraries** — threads come from the OS via FFI (§1), and `Send(T)` / `Sync(T)` are ordinary
**comptime predicates** (`fn(type) bool`, Comptime §4 / §6.2), **not** language entities and
**not** a new mechanism. Because a data race is **hardware-defined, not UB** (§5.1), they are
an **optional discipline check**, not a correctness necessity: when such an API requires the
bound (`when Send(T)`), the compiler rejects sharing non-thread-safe data across a thread
boundary, turning §5.3's discipline into a compile-time check (decisions CC-5/CC-4). Full
borrow-checker-level static proof of race-freedom is deliberately **not** provided (FND-9).

The language **guarantees they are expressible without guessing** (I10/FND-6) — the v1 surface
already provides everything a library needs:

- **structural inference via `typeinfo`** — `Send(T)` / `Sync(T)` recurse over the type's
  fields; a type qualifies when all its fields do. The bottom is the non-thread-safe leaves:
  a raw `ptr(mut T)` under sharing, and a scoped reference (second-class — cannot be sent
  at all, Memory §5.3); a `Mutex`-guarded or atomic-accessed datum qualifies;
- **nominal opt-out via a brand** (Comptime §4.4) — where structural inference is wrong or beyond
  introspection (a type holds a raw pointer yet is logically thread-safe), a brand marker
  overrides it and the predicate checks the marker; a wrong assertion is a logic bug, not UB (§5.1);
- **`Send(T)`** = `T` may be **moved** across a thread boundary; **`Sync(T)`** = a `ptr(T)`
  may be **shared**; bounds use `when` (Comptime §4.3), localizing a violation to the call site.

(If structural inference needs a type property `typeinfo` does not yet expose — e.g. pointee
mutability — that exposure is itself additive; the brand opt-out covers the gap meanwhile.)

---

## Part B — Integer overflow

### 6. Overflow is defined

By I11, integer overflow is **defined behavior, never UB** (decisions CG-8); it extends
the checked-guard family (Assembly §3) to arithmetic.

#### 6.1 Default: trap

By default (**checked**), an arithmetic operation that overflows **traps** — a defined
failure (Stdlib §4), not UB — **everywhere except inside an `unchecked` scope** (§6.2).
The trap is an inline check; its cost is the checked-guard family (I2).

The **checked-overflow set** is: `+`, `−`, `*`, **unary `−`** (negating the minimum
value), and **`MIN / -1`** (division overflow). Separate checked-guards (which also
**trap**, but are not "overflow") are **division by zero** and a **shift amount ≥ the
width** (Assembly §3).

**One failure mechanism** (CG-13). Every failure in this family — the checked-overflow
set, division by zero, and an over-width shift — is the language's **direct inline trap**:
the guard is emitted **before** the operation, and the trap is the per-architecture
instruction fixed in the per-arch appendix §2 (`ud2` on `x86_64`/`i386`, `brk #0` on
`aarch64`, `udf #0` on `aarch32`, `ebreak` on RISC-V). It never calls `panic`, an OS exit,
or a programmer hook, and it does **not** defer to the target's own fault behaviour: on a
target whose divide instruction faults by itself on a zero divisor, the inserted guard has
already trapped, so the observable failure is the same instruction on every architecture
and in every context (a bare/`freestanding` program included). `panic` stays an ordinary
library operation (Stdlib §4) that library code may call; it is never the mechanism of a
built-in guard.

**The guard belongs to the operation, not to whatever implements the operator.** Operators
are ordinary library functions (Type System §4.5, §5.3), so an implementation may route
`+` / `−` / `*` / `/` / `%` through a prelude operator body — and that routing changes
nothing here: the compiler emits the family's guard at the **operation site** whether or
not the operator is routed, and a routed operator body carries **no guard of its own**.
Otherwise the same expression would fail one way with the prelude operator in scope and
another way without it, which is exactly the implementation fork CG-13 removes. `panic` and
`assert` inside ordinary **library logic** — a container's own bounds check, an explicit
precondition — are untouched: that is library behaviour, not a checked guard of the
language.

#### 6.2 Inside an `unchecked` scope: wrap

Inside an `unchecked` scope (a verification **mode**, CG-7 — `unchecked expr` for one
expression, `unchecked { … }` for a block), `+` / `−` / `*` / unary `−` **wrap** — plain
two's-complement, the same defined value on every target. The remaining members of the
family are dropped to **hardware behaviour**, which is *not* the same thing and *not*
uniform: **division by zero**, an **over-width shift**, and **`MIN ÷ -1`** do whatever the
target does — `x86_64` faults on `MIN ÷ -1` and on a zero divisor (`#DE`), `aarch64`
yields `MIN` and `0` respectively, and an over-width shift is masked on some ISAs and not
others. Writing them under `unchecked` is therefore target-specific by construction (I11:
hardware-defined, never UB — but never portable either). The whole
scope is affected (not a single operation), and the choice of granularity — one line or
a block — is the programmer's; correctness inside the scope is the author's
responsibility (Control Flow §2.3; CG-7/CG-6).

#### 6.3 Explicit policy operations

The prelude provides explicit operations, available **everywhere** (no grant), as
ordinary functions (Type System §4.5):

- **`wrapping_*`** — wrap (two's-complement); equal in result to an unchecked op but
  available everywhere;
- **`saturating_*`** — clamp to the type's min/max;
- **`checked_*`** — `→ Option(T)` (`None` on overflow);
- **`overflowing_*`** — `→ (T, bool)` (the wrapped value and an overflow flag).

#### 6.4 Kernel has no arithmetic

`bitsN` has **no** arithmetic — only bit operations (Type System §2.2); overflow
concerns the numeric interpretations `u`/`i` only.

---

## Part C — Floating point

### 7. Determinism

#### 7.1 Comptime float

Compile-time floating-point is **strict IEEE-754**, **round-to-nearest-even**, and
**host-independent** (soft-emulated where needed; **no** x87 extended precision, **no**
FMA contraction) — so a comptime float result is **bit-exact reproducible** across
machines (Comptime §2).

#### 7.2 Runtime float

Runtime floating-point is the **target's IEEE-754**, with **no** hidden contraction or
reassociation (predictability I1; the optimizer is semantically-preserving — it never reorders or
contracts float ops in a way that changes results, CG-5). **Subnormal (denormal) numbers are preserved** — gradual
underflow per IEEE-754; flush-to-zero / denormals-are-zero modes are **never enabled
implicitly** (they are non-IEEE and target-dependent, and would break reproducibility).

#### 7.3 FMA is explicit

A fused multiply-add is **only** the explicit builtin **`fma(a, b, c)`**; `a * b + c`
does **not** auto-contract to an FMA.

#### 7.4 IEEE exceptions — and the integer contrast

Floating-point follows the **IEEE-754 default** (non-trapping) exception behavior:

- **division by zero → ±∞**, **invalid** (e.g. `0.0/0.0`) **→ NaN**, **overflow →
  ±∞** — **no trap**.

This is the key contrast with integers: **integer** division by zero **traps**
(§6.1), whereas **float** division by zero yields **±∞** per IEEE.

#### 7.5 NaN comparisons

NaN comparisons are **unordered** (IEEE-754): `NaN == x` is **false** (including
`NaN == NaN`); `<`, `>`, `<=`, `>=` involving NaN are **false**; `!=` involving NaN is
**true**.

#### 7.6 Rounding and v1 types

Rounding is **round-to-nearest-even** by default; there is **no** implicit dynamic
rounding-mode state in v1. The v1 float types are **`f32`** and **`f64`** (native
where the target has hardware, else soft-float; Type System §3.3); `f16` / `bf16` /
`f128` are **additive** (FND-6).

#### 7.7 Float-to-integer conversion

Converting a float to an integer (`i64(f)`, `u32(f)`, …) **rounds toward zero**
(truncates the fractional part: `3.9 → 3`, `-3.9 → -3`). The conversion is **checked by
default**: a result outside the integer's range, or a NaN, **traps** (I11 — the same
checked-default as narrowing, Type System §4.2). A **`saturating`** conversion (prelude,
available everywhere) instead **clamps** to the integer's minimum/maximum and maps
**NaN → 0**, never trapping. **Inside an `unchecked` scope** the conversion is
**hardware-defined** (§6.2; the out-of-range result depends on the ISA).

---

### 8. Conformance (normative summary)

A conforming implementation MUST:

1. provide **no** built-in threads / async runtime / scheduler (threads via OS FFI),
   only the primitives; treat `async`/`await` as reserved with additive machinery and
   fibers as an additive library (§1; I3/CC-1, CC-7);
2. provide atomics as **operation builtins** on a `ptr(mut T)` (`atomic::load`/
   `atomic::store`/`atomic::swap`/`atomic::cas_strong`/`atomic::cas_weak` (each → `(T,
   bool)`)/`atomic::fetch_*`) with a comptime `Ordering` argument and **no** `Atomic(T)`
   type, atomic-capable widths as per-arch data, lowering to atomic instructions +
   barriers, zero-cost (§2; I1/I2);
3. provide explicit `fence(ord)` and `volatile::load`/`volatile::store` (each access
   emitted exactly once, not reordered w.r.t. other volatile; **not** atomic) (§3, §4);
4. treat data races as **hardware-defined, never UB**, provide the three by-construction
   race-free routes enforced by linearity / scoped-refs / immutability / Mutex, leave
   unsynchronized shared mutable as the discipline point, and treat `Send`/`Sync` as an
   additive comptime-protocol capability (§5; I11/FND-9/CC-4, CC-5);
5. make integer overflow **trap** by default (the set `+ − * unary− MIN/-1`; div-by-0
   and over-width shift are separate guards), with one uniform failure mechanism — the
   direct inline trap (CG-13) — and inside an `unchecked` scope **wrap** `+ − * unary−`
   while `MIN/-1`, div-by-0 and an over-width shift take their target's hardware
   behaviour (CG-7), and provide `wrapping_*`/`saturating_*`/`checked_*`(→
   Option)/`overflowing_*`(→(T,bool)) everywhere; `bitsN` has no arithmetic (§6; I11/CG-8);
6. make comptime float **strict, host-independent IEEE-754 (RNE)** and runtime float
   the **target IEEE-754 with no hidden contraction/reassociation**, provide FMA only
   via explicit `fma`, follow IEEE non-trapping exceptions (float div-by-0 → ±∞, not a
   trap — unlike integer), unordered NaN comparisons, RNE rounding with no implicit
   mode state, and `f32`/`f64` as the v1 types (§7; I1/CG-9).
