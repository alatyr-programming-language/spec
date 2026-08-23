# Decisions — Concurrency

**Why the language is the way it is** for concurrency. Each entry records the *rationale*
and the *rejected alternatives*; the normative *what* lives in the spec (linked per
entry), the *how* in the compiler. IDs are theme-prefixed (`CC-N`) and **append-only** —
assigned in articulation order, never renumbered. The `provenance:` line records what an
entry refines and where it lands in `spec/`.

---

## Concurrency

### CC-1. No built-in runtime — the language gives primitives, the OS gives threads
*spec: Concurrency §1, §6*

**Why.** A built-in thread or async model would mean a hidden scheduler/runtime, which
violates "nothing hidden" (I3). So the language ships **primitives** only: threads come
from the OS via FFI/syscalls, not from a language-managed runtime. There is no built-in
thread model and no built-in async model. Higher-level machinery (executors, schedulers,
lightweight threads) is built additively on top of these primitives as ordinary
libraries, keeping the language minimal and its cost transparent (I1/I2).

### CC-2. Atomics, fences, and `volatile` are operation modules, not new types
*spec: Concurrency §2, §3, §4*

**Why.** Correct concurrency needs atomic load/store/RMW (CAS, fetch-add, …) with the
standard **memory orderings** (`relaxed`/`acquire`/`release`/`acq_rel`/`seq_cst`), plus
explicit barriers and MMIO accesses. These are exposed as **prelude modules**
(`atomic::`, `volatile::` — qualified UFCS, per MOD-3), each lowering directly to the
corresponding atomic instructions and barriers — transparent (I1) and zero-cost (I2),
not a new pointer family or runtime abstraction. Per-operation ordering legality is
enforced at compile time (load ∈ relaxed/acquire/seq_cst; store ∈ relaxed/release/seq_cst;
RMW any; CAS-failure ∈ {relaxed,acquire,seq_cst} and ≤ success), matching C11/Rust, so
nonsensical orderings are rejected rather than silently accepted.

**Rejected.** Allowing a **`relaxed` fence** — C++ permits it as a no-op, but a relaxed
fence has no effect, so (like Rust) it is a compile error to avoid a meaningless form.
Folding `volatile` into atomics — `volatile` is a distinct guarantee (the compiler emits
exactly as many accesses as written, no elimination/reorder) needed for MMIO, separate
from atomicity. A **volatile-qualified pointer** (`ptr(volatile T)`, a path
qualifier like `mut`) is left **additive** (FND-6); v1 uses the `volatile::` operations.

### CC-3. Data races are hardware-defined, not undefined behavior
*spec: Concurrency §5.1*

**Why.** The language does not statically prevent data races, because preventing them
generally requires a borrow-checker, which is deliberately not adopted (FND-5). But "not
prevented" must not mean "undefined behavior": per I11 a data race is
**hardware-defined**, not UB. A torn or stale value is an ordinary logic bug, not a
license for the compiler to miscompile — there is no UB to exploit, so the optimizer
never exploits one (I11/CG-5). Correct concurrency is achieved through atomics;
race-freedom is a matter of discipline (and, optionally, the additive `Send`/`Sync`
checks of CC-5).

### CC-4. Race-safety by construction — three paths over existing mechanisms
*spec: Concurrency §5.2, §5.3*

**Why.** A programmer needs a realistic way to guarantee the absence of races without the
language adopting a borrow-checker (FND-5). The insight is that race-freedom can be reached
through **three paths, each enforced by mechanisms that already exist**, with no new
machinery:
1. **Share-nothing** — communicate by moving data through a channel. **Linearity** (MEM-2)
   guarantees exactly one owner, so after `send` the source is unavailable
   (use-after-move is an error); and **scoped references do not escape** (MEM-1), so only
   self-sufficient owned data crosses the boundary, never a reference. No shared cell
   means no race.
2. **Immutable-share** — share only deeply-immutable data (MEM-7, immutability enforced);
   concurrent reads cannot race.
3. **Mutex-guarded** — `m.lock()` yields a **scoped reference** `ptr(mut T)` (which
   cannot escape, MEM-1) plus a linear guard unlocked via `defer`. T is reachable only
   while the lock is held and the Mutex serializes access, so two `&mut T` can never
   coexist — the scoped reference plays the role a borrow-checker would, pointwise along
   the do-not-escape axis. The concrete **`Mutex(T)`** type is **additive** (it lands with
   the threading surface; threads are FFI/OS, single-threaded by default), but its
   **contract** is v1-fixed (`lock(in m : ptr(Mutex(T))) -> scoped ptr(mut T)`, released by
   `defer`) so it is provably an ordinary library type over the scoped-ref + `defer`
   mechanisms — no new language entity (spec §5.2).

The compiler enforces these **local** invariants (single ownership, non-escape,
immutability, lock scope) but does **not** globally prove race-freedom (FND-5). The known
gap is **unsynchronized shared mutable** state (notably a `mut` global, FN-3): for a 100%
guarantee the programmer routes any cross-thread `mut` through an atomic/Mutex (path 3),
shares only moved or immutable data (paths 1–2), and stays out of raw/`unchecked` for
cross-thread access. This is documented as discipline, not a guarantee.

### CC-5. `Send`/`Sync` are additive library predicates, not language entities
*spec: Concurrency §5.4*

**Why.** A capability discipline that *checks* thread-safety — Rust's `Send`/`Sync` —
is desirable, but it need not be a new language mechanism. Both the thread/channel API
**and** the `Send(T)`/`Sync(T)` predicates can be **ordinary libraries**: `fn(type) bool`
over comptime-protocols (CT-4/CF-8), **structurally inferred** via `typeinfo` (CT-6 — if all
fields qualify, the type does) with a **nominal brand** opt-out (TYP-4) where inference is
wrong or beyond introspection. Roles: `Send` = movable across a thread boundary;
`Sync` = `ptr(T)` shareable across threads; the non-thread-safe leaves are a raw
`ptr(mut T)` under sharing and a scoped reference (which cannot be sent, MEM-1). When
such an API states the bound (`when …`), the compiler rejects sharing non-thread-safe
data across a thread boundary — turning CC-4's gap-discipline into a real compiler check,
approaching Rust's "the compiler catches it" without a borrow-checker. Because a race is
not UB (CC-3), the check is an **optional discipline**, not a correctness necessity. The
v1 surface (typeinfo + predicates + brand + scoped-ref/linearity) makes it expressible
without guessing (I10/FND-6); if structural inference ever needs a type property `typeinfo`
does not expose, that exposure is itself additive (CT-6) and the brand covers the gap.

**Rejected.** A **borrow-checker** for full static race-freedom (FND-5) — the price would
be that advanced aliasing-`&mut` patterns require a lock/copy/move, which is **not less
safety** for the disciplined paths above, only more language.

### CC-6. `async`/`await` reserved now; the machinery is additive
*spec: Concurrency §1*

**Why.** Whether async ever ships, the question of *adding it later without breaking the
language* must be settled now, because the direction of breakage is asymmetric: **adding
a keyword later breaks programs; freeing a reserved one later does not**. So `async`/`await`
are **reserved keywords now** (SYN-2) with the semantics deferred — a reservation of a
syntactic slot, not a feature; if async never materializes the words are removed
harmlessly. They are bare keywords, **not** `@`-attributes, because they change a
function's role and control flow (an `async fn` returns a future-automaton; `await` is a
suspension point), and by the TYP-7 test `@async` would misclassify role as representation.
When added, the `async fn → automaton` transform is an **additive lowering** (a new
node/pass that does not touch the existing one, documented per I1); a future is a concrete
sized type (zero-cost) or `@alloc` (heap-pinned), targeting a comptime-protocol
`Future`/pollable (a form, not a name — CF-8), with the executor and storage left to a
configurable library. The door is kept open for an additive immovability/`Pin` protocol
for zero-cost self-referential stack-futures (the heap-pinned path does not need it).

### CC-7. Lightweight threads (fibers) are fully a library
*spec: Concurrency §1*

**Why.** Fibers require no language change: a context switch (save/restore callee-saved
registers, swap the stack pointer, `jmp`) is already expressible with the existing
Assembly chapter — a naked function `@abi(naked)` plus instructions/registers including
`sp` and `asm(...)`; the stack comes from `@alloc`, and the scheduler/I/O is a library or
FFI. This lives at the raw level (§9.2) with a safe API layered on top. Together with the
poll-future being a library and async/await-sugar being additive (CC-6), this passes the
I10/FND-6 test: the concurrency direction grows additively on the current language without
underspecifying v1.
