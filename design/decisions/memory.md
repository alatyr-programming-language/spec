# Decisions — Memory, lifetime, ownership, allocation

**Why the language is the way it is** for the memory model. Each entry records the
*rationale* and the *rejected alternatives*; the normative *what* lives in the spec
(linked per entry), the *how* in the compiler. IDs are theme-prefixed (`MEM-N`) and
**append-only** — assigned in articulation order, never renumbered (a later sub-theme
appends higher numbers, it does not shift these). The `provenance:` line records what an
entry refines and where it lands in `spec/`.

---

## Lifetime, ownership, allocation

### MEM-1. One language reference + a library provider menu
*spec: Memory §5.1–§5.2*

**Why.** Temporal safety could have been a set of *language* reference kinds (scoped vs
generational vs refcounted as different pointer families). That path multiplies the
language: every pair of mechanisms needs a defined join (the N² problem), and the
reference *type* must encode the mechanism. Instead the language has exactly **one**
reference — the **scoped** (second-class) reference — and longer-lived
allocation-and-lifetime is **not a second language reference** but a **library
allocator-provider menu** (region/arena, generational, manual, …): ordinary library
values, each handing out a **first-class reference token** (a `Handle(T)` number for
regions, another representation for another provider), not a language reference. The
"menu" is thus a menu of *library providers over one allocator-reference protocol*, not
of language mechanisms.

**Rejected.** *Reference-type-encodes-mechanism* (MEM-1's original guardrail 2) — dropped:
it forces the family explosion and a typed-pointer zoo. Cross-mechanism *joins* (MEM-1
guardrail 3) evaporate because there is only one reference plus library handles.

### MEM-2. Owning values and linearity — `in` is the consume
*spec: Memory §5.9, §3; Types §8.1*

**Why.** Ordinary values copy by value (visible cost, I1); there is no implicit move and
no RAII. But a *resource* (a file, an arena) must be released exactly once — so an
**owning** value is **linear**: non-copyable, consumed exactly once, contagious. A resource
is an OS object **or an exclusive right over storage**: an arena owns its right to allocate
from and reclaim its buffer, so even an arena over a caller's buffer is owning — a copy would
be a second cursor handing out the same addresses (MEM-3). Two mechanisms had to be pinned to
make this implementable without guessing: how a type is *classified* owning (the **`@owning`**
marker — the same `Arena` type was otherwise used both as a borrowed bump-view and as an
owning OS arena, so no implementation could decide which values are linear) and what
*consumes* one (passing it **`in`** is the move). Release is explicit (`defer`/`close`), never
hidden (I3). A `@static` owning value has no finalize obligation and is never consumed (its
extent is the program's; a release reachable from every function could not be tracked
locally).

Consequently TYP-12 rejects an owning immediate underlying `U` for `@require`: checked
construction must preserve the result and also supply an ordinary by-value `in U`
predicate argument, which would require an impossible copy or would consume the result.

**Rejected.** Implicit move for copyable types (no double-drop motive without RAII);
inferring owning-ness from use (ambiguous — the dual-role `Arena` proved it); "owning" meaning
only "behind a releasable OS resource" (**superseded** — it left the caller-buffer arena
copyable and its `close` non-consuming, so temporal safety had nothing to rest on); a dedicated
`move`/`consume` parameter direction or marker (a new grammar axis for what `in` +
non-copyability already express — OP-1); consume **only** via recognized prelude intrinsics
(too narrow — a user could not write an ownership-taking function, and `Vec`/`HashMap` could
not express their own drop).

### MEM-3. The region/arena surface; `get` is an ordinary function
*spec: Memory §5.2.1, §5.3.1, §5.4, §5.9; Stdlib appendix §5.2.1*

**Why.** MEM-1's menu deferred each mechanism's concrete surface to "stdlib, additively",
which left the region/handle mechanism with no implementable v1 form. The region surface
(`Arena` / `Handle(T)` / `arena_over` / `get` / `close`) is fixed as **ordinary library
code**, and `get(ptr(a), h)` is de-privileged from a compiler-recognized intrinsic to an
**ordinary function** with a **second-class (`scoped`) return** — a result the caller may
use at the call site but not store or re-escape. Temporal safety then rests on three
already-present things, no new reference type: the arena's **linearity** (MEM-2), `get`'s
**bounds check**, and the **second-class** result. Two refinements make that real:

- **Every `Arena` is an owning value** (`@owning`), like an open file — the caller-buffer
  arena of `arena_over` and the `@static` fixed pool included. It cannot be copied, and it is
  consumed exactly once, by `close(in self : Arena)` (a consuming parameter) or `forget`; a
  `@static` arena is never consumed. Use after `close` and a missing `close` are then errors
  of the existing linearity checker, at no run-time cost. Allocation (`in out`) and `get`
  (a `ptr(Arena)` borrow) do not consume it. The OS-backed arena stays a distinct `@owning`
  type holding an `Arena` over `mmap`'d pages, whose release closes it and unmaps the pages.
- **A `scoped` result is transient in checked code** (Memory §5.3.1): it is used within the
  full expression of the call — dereferenced, field/element-accessed, or passed down — and
  never bound, stored, or captured; the call's `in out` / pointer-borrowed places are frozen
  in that full expression (no other `in out`, `ptr(mut …)`, or consuming use). So a get-result
  cannot outlive a growth, reallocation, or close: excluded statically, not "a residual
  hazard". The rule is **generic** to `scoped` returns, so `get` still has no arena-specific
  compiler contract and a user could write the same signature. Inside `unchecked { … }` the
  result may be bound to a local; its staleness is then hardware-defined (I11).
- **`arena_over` requires `unchecked`.** The arena over a caller buffer keeps `buf`'s
  address inside an owning value that can be moved, returned, or stored past `buf`'s scope,
  and the local, unparameterized escape rule cannot tie it to `buf`. So a call to
  `arena_over` is a raw operation (CG-7): the author asserts "`buf` outlives the arena" at a
  visible `unchecked`, and everything after construction stays checked. A violation is
  hardware-defined, never UB (I11).
- **The arena is an explicit argument**, by address: `get(ptr(a), h)`. `get`'s
  `in a : ptr(Arena)` is not an allocator parameter, so no ambient fills it (MEM-5), and a
  free-call argument is never auto-addressed — only the generic UFCS receiver form
  `a.get(h)` takes the address, as for any method (MOD-3).

The `scoped` marker carries information
**only on a result type** (every reference is second-class by default, so the qualifier means
something only where it overrides the escaping default — the return); a `scoped` written in any
other type position (parameter / field / local / nested arg) is **ill-formed**, not silently
ignored (I3), with a diagnostic to drop it.

**Rejected.** `get` as an *unchecked raw* primitive (loses store-safety — a stored result
dangles on arena growth); tying the result's scope to the arena *argument* (a lifetime
variable, which the one-reference model forbids); a handle **typed by its enclosing region's
name** (taking the region into the type — reintroduces lifetime variables and creeps toward a
borrow-checker); a caller-buffer `Arena` that is **not owning** (superseded — `b := a` gave two
cursors over one buffer, and a non-consuming `close(in out self)` let a handle used after
`close` silently reach a newer object); treating a get-result held across growth as a
"residual hardware-defined hazard" in checked code (superseded — it contradicted I11 and
Memory §5.1); a **stored** get-result with a non-mutation rule while it lives (a local borrow
check, against Memory §5.7 — no borrow checker); a **runtime generation check** on `get` (a
cost per access and a fat handle, and it does not stop arena copies); an **implicit or ambient
arena** for `get`, or an auto-address-of on its free-call argument; for the `arena_over`
escape, a result **scoped to `buf`** (a lifetime variable tying a result to an argument — the
same alternative rejected above), a `scoped Arena` result (a transient value cannot be bound, so
the arena would be unusable), or a checked `arena_over` restricted to a `@static` buffer (an
arena-specific argument check — deferred: admitting it later only accepts more programs, FND-6) (hidden data flow into an
access that the reader must see, I3 — the arena is named at every `get`). The second-class return
de-privileges `get` *and* keeps store-safety *and* introduces no lifetime variable.

### MEM-4. The allocator protocol `alloc`
*spec: Stdlib §160 §5.1*

**Why.** An allocator is an ordinary value satisfying a structural protocol
(`mechanism`/`allocate`/`free`), named **`alloc`** — never a language entity. A function
takes one as a parameter of this protocol, monomorphized per concrete allocator
(zero-cost, allocator-polymorphic, I2). Distinguishing it by **type alone** (no marker)
keeps the surface minimal (OP-1); the reference an allocation yields is **typed by the
mechanism** (region → a handle, etc.), so cost stays visible (I1/I2).

**Rejected.** `@alloc` itself yielding `Result(Handle(T), AllocError)` — an attribute that
silently changes the binding's type and forces a near-ubiquitous trailing `?`; the
recoverable form is the explicit `allocate(…)?` call instead (the trapping `@alloc` is the
convenience over it; its failure is `trap()` in every verification mode, CG-13).

### MEM-5. The ambient allocator — `alloc::with`
*spec: Memory §5.2.1, Functions §5.5, Overview §4.1*

**Why.** Threading an explicit allocator through every constructor is noise where the
allocator is obvious. So an allocator **parameter** (an `in` pointer to mutable —
**not** `in out`, which the grammar bars from defaults/optionality) is the function body's
**ambient** allocator and is **elidable** at call sites; **`alloc::with(ar) { … }`** (a
scope form, sibling of `unchecked`) establishes it for a region. This is *not* hidden:
the dependency is in the signature and the source is a named binding in scope — the
package fully determines it (the closed-space reading of "no magic", Overview §4.1 / I3).
The outermost `alloc::with` (in `main`) is the de-facto default; there is **no manifest or
process-static global** (no silent static state). An allocating site with no resolvable
ambient is a **compile error**, never a hidden heap. `alloc::with` is a **runtime** scope:
ill-formed inside a `comptime` context (comptime values are erased, I7 — no program ambient
to establish), a **compile error** under `no_alloc` (allocation is forbidden there), and
**nesting is lexical shadowing** (inner overrides, outer restored on exit — no combining).
Shapes: **A** no allocator parameter
(self-contained) / **B** required, ambient-elidable (escaping producers) / **C** B with a
call-site default (scratch-only — a create-fresh default is a call-site temporary, sound
only for non-escaping allocations; the escape checker enforces it).

**Rejected.** A manifest/process-static global default (invisible in source, static state);
a compiler-*derived* implicit allocator parameter threaded by call-graph fixpoint
(caller-context-dependent codegen, derived ABI) — replaced by a *declared* parameter +
call-site elision; a general "ambient value by type" (implicit-resolution footgun) — kept
narrow to the `alloc` protocol; an `alloc::current()` accessor (duplicates a visible name).

---

## Value model, storage, pointers

### MEM-6. Values, places, bindings, addresses
*spec: Memory §1*

**Why.** A precise vocabulary is the foundation the rest of the model attaches to: a
**value** is a typed datum with no identity/location/scope; a **place** is a storage
location (type + storage class + mutability + lifetime, addressable only when
memory-resident); a **binding** names one; an **address** is a place's location taken as a
value. Separating the four lets mutability attach to the *path* to a place (not to values),
lifetime attach to places, and the dangling problem be stated precisely — without them the
later rules (mutability, lifetime, pointers) would have nothing to hang on.

### MEM-7. Storage classes, mutability; the dissolved `mem` cluster
*spec: Memory §1.6, §2–§3*

**Why.** Places live in one of four **storage classes** — register (not addressable),
stack (LIFO activation), static (whole program), heap (allocator-bound, **always explicit**
per I3) — fixed by declaration site, so lifetime and cost are predictable from the source
(I1). **Mutability is immutable-by-default**, a visible `mut` opt-in (I3/I8) that is a
*permission on the path* to a place (binding / field / pointer), **not** a property of a
value — so aliasing and write-permission compose cleanly. The earlier **`mem::` cluster
dissolves by nature** — it had grouped three different kinds of operation under one module.
Runtime pointer/place ops are the flat prelude word-functions **`ptr` / `deref`** (MEM-8);
compile-time **layout** (`size` / `align`) is *derived from* the machine model and exposed as
prelude word-functions `T.size()` / `T.align()` (CT-6), and a field's offset is read from
`typeinfo(T).fields` — not a parallel `mem::*` query surface; the **addressing operand**
`at(…)` belongs to the **arch surface** beside registers (CG-2). There is no `mem` module —
each operation lives in its true home.

**Rejected.** Mutable-by-default (hides writes, against I3); value-level mutability (MEM-6 —
mutability is about the place/path, not the datum); **per-binding-only** mutability
(Rust-style) — simpler, but underdelivers expressiveness (I10): mutability is carried **per
field** too, so a type can guarantee a forever-immutable field regardless of the owner's
`mut`. A single **`mem::` module for all three** kinds (superseded) — it mis-prefixed the
comptime layout queries as "memory access" and duplicated `typeinfo` (the CT-6 "one reify,
no scatter of query-builtins").

### MEM-8. Pointers — `ptr(T)`
*spec: Memory §4*

**Why.** A pointer is **`ptr(T)`** / **`ptr(mut T)`** — carrying the target type, the
**pointee's** mutability, and alignment (natural for `T`, with a pointwise override). It is
**always non-null and valid** (absence is `Option`, not a null pointer), so a checked
dereference needs no null guard and rights are **monotonic** (a pointer cannot widen
write-permission). The names are **word-functions** (OP-1): `ptr` is polymorphic — on a
*type* it is the pointer type, on a *place* `ptr(x)` / `x.ptr()` is that place's address —
and **`deref`** is its inverse, `deref(p)` / `p.deref()` (a place on the LHS, `p.deref() = x`,
write right = pointee mutability). The pair reads as "make a pointer ↔ follow it", and both
forms phrase the same way ("pointer to X": value→address, type→pointer-type). The canonical
type spelling is **prefix** `ptr(mut T)` (UFCS cannot carry `mut`); `T.ptr()` is the free
UFCS dual, idiomatic in chains (`typeof(v).ptr()`). `ptr(mut T)` is also shorter than the
former `mem::addr(mut T)` — without a sigil.

**Rejected.** Nullable pointers (absence is `Option`, preserving the always-valid
invariant). **`ref`** for the pointer — collides with the one *scoped reference* and the
stdlib **`Ref(arena, T)`** type (MEM-1/MEM-3), conflating pointer / scoped reference /
handle. **`val`** for the dereference (superseded) — too polysemous; `deref` says what it
does and pairs with `ptr`. The **`T.addr` / `T.mut.addr` suffix** and the **`mem::addr` /
`mem::val` module** spellings (both superseded) — a sigil is barred (`&`/`*`/`~` are taken,
OP-2; a new prefix marker would break the one-marker `@` model, SYN-7/CT-10), and the flat
word-function `ptr(mut T)` is shorter than `mem::addr(mut T)` without one.
