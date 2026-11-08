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
- **`arena_over` requires `unchecked` — except over a static buffer.** The arena over a caller
  buffer keeps `buf`'s address inside an owning value that can be moved, returned, or stored past
  `buf`'s scope, and the local, unparameterized escape rule cannot tie it to `buf`. So a call to
  `arena_over` is a raw operation (CG-7): the author asserts "`buf` outlives the arena" at a
  visible `unchecked`, and everything after construction stays checked. A violation is
  hardware-defined, never UB (I11). **The exception:** when the operand is `ptr(mut X)`/`ptr(X)`
  of a place declared `@static` (directly, or through field/element steps), the call is **checked** —
  a static place lives for the whole program, so the fact the author would vouch for is guaranteed by
  the declaration, and the test stays syntactic (I1, FND-3). Without it a `freestanding` program over
  a static pool could not allocate in checked code, which PRIN-2 does not permit.
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
the arena would be unusable), or a checked `arena_over` restricted to a `@static` buffer **by an
arena-specific argument check** (rejected as framed — no arena-specific compiler contract, MEM-3;
**adopted in the end as a general syntactic storage-class test**, not an arena rule: the operand is
checked for `@static` the same way any raw-operand argument would be, and PRIN-2 requires the
freestanding static-pool path to be writable in checked code) (hidden data flow into an
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
§5.1 default naming a **static** allocator place.

**An omitted allocator argument names an existing place that outlives the call** — the
ambient (a binding in scope) or, for a §5.1 default, `ptr(mut P)` with `P` of static storage
reached without a pointer step. A default that *creates* an allocator is ill-formed: under
owning arenas (MEM-3) a fresh `Arena` must be `close`d, and nothing in the source would close
it. Temporary memory is therefore explicit — a local arena, `defer close(a)`, then pass it or
`alloc::with(a) { … }` (Memory §5.2.1 shows the idiom). The elided argument is a use of the
ambient place, so `alloc::with(a) { close(a) ; f() }` is a linearity error, as the written
`f(ptr(mut a))` after `close(a)` is.

**Rejected.** A fresh temporary allocator with **no closer** (the former "create-fresh
default"): linearity cannot hold for an owning `Arena` that nothing consumes. *Rejected (for
now):* a call-site temporary arena **closed by a compiler-inserted `close` at the end of the
full expression**. It could be admissible under CG-11 as a specified language convention,
since the explicit form stays available; it is not adopted in 1.1 because it is a hidden
allocation plus a hidden call, both of which I3 lists, and it is additive if wanted later
(FND-6). A manifest/process-static global default (invisible in source, static state);
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

### MEM-9. A pointer meeting an integer is grant-only
*provenance: refines MEM-8 (§4.1 non-nullness, §4.2 optional pointer) · spec: Memory §4.5; cites Concurrency §2 · owner decision, #1030*

**Why.** The instrument already sees the shape and the checker accepts it: `cmp_ptr_zero :=
fn(p : ptr(u8)) -> bool { return p == 0 }` passes `check` with rc=0 on both the tree compiler
and the frozen seed, while `scripts/ptrint_census.sh` reports the OP-CMP row for it (#1029
proved the hook fires). So detection and enforcement had come apart. The spec was silent: §4.5
named *fabricating* a pointer from an integer as capability-requiring, and said nothing about
comparing one. Silence here is not neutral — two implementations could accept and reject the same
source, and FND-3's bar — two independent developers must build **compatible** compilers **without
guessing** — leaves a form that is neither confined nor permitted with no answer at all.

**The rule.** Comparing a pointer with an integer is legal **only inside an `unchecked` grant**,
where it tests the pointer's representation, the integer operand being `usize` and no conversion
being performed; outside a grant it is ill-formed. What is target-dependent is the representation
(a pointer's width, the bits a fabricated address holds — I6, I11 as §4.5 already uses them), not
the comparison: given two `usize` values every conforming target returns the same answer. This is the grant model §4.5 already applies to the neighbouring
operation, not a new kind of rule — which is why it was chosen over the alternatives. It is also
what MEM-8 makes evident: outside a grant every `ptr(T)` is non-null and valid, so a checked
`p == 0` cannot be true, and the author's actual question — "is there nothing here?" — is
`Option(ptr(T))`, whose `None` is the all-zero niche and which a `match` cannot leave out.

**Refined by the same owner decision — both directions.** A follow-up read showed the rule as first
written was bypassable in two tokens: `bitcast(usize, p) == 0` outside a grant compares two integers, so
confining only integer→pointer left the comparison reachable. The owner chose to confine **integer↔pointer
in either direction**, which is what §4.5's own title and Grammar §3.7's raw list already said; the body
now matches them. The measured cost is stated rather than discovered later, and it is **not** the census's
`places=3`: that record counts the **implicit** half only — it excludes explicit bitcasts by construction —
so quoting it as the cost of a both-directions rule was a category error, and the earlier text that did so
is corrected here. The cost is in the **explicit** half: **at least 14 sites in 8 tracked fixtures** read
a pointer out with `bitcast(usize, …)` outside any grant and become illegal, enumerated line by line in the
CHANGELOG entry for this revision (a floor: a lexical sweep cannot tell a pointer operand from an integer
one). They are fixtures, not stdlib — `lib/` and `src/` hold zero such sites — and the compiler repo files
the fix as its own change with its own gate.

**Rejected.** *Refuse in every context, grant included* — contradicts §4.5, under `unchecked` an
address may be fabricated or already invalid and testing its bits is precisely what the grant is
for; it would also refuse code written correctly by today's rules. *Keep detecting without
refusing, and let the census ratchet be the pin* — cheapest, breaks nothing, and the goal permits
the `review-only` label explicitly; rejected only because the grant model already exists in the
checker and reporting a form as enforced while accepting it is the gap this decision closes. The
census's OP-CMP ratchet stays as the measurement either way (#1029).

**Downstream.** The compiler conforms when the pin moves (workspace rule: a spec change binds the
compiler only after a release is cut and the pin moves in its own explicit compiler commit). #1034
implements the refusal on the comparison path and waits on that pin, not on review — and, once the
pin moves, on a rebase onto the new `main`, the **expected-delta** update it forces (the census and
`xbackend_diff.baseline` counts move when the refusal starts firing), and a **green authoritative gate**
before it lands. Its seam is
kind-based (`TyInt × TyPtr`, either order), so it enforces the **grant boundary**; the rule that the
integer operand is `usize` is NOT yet enforced there — inside a grant `p == u32(0)` stays accepted, and
closing that is a follow-up on #1034, not a claim of this record.

### MEM-10. Comparison is a relation over an operand pair; the grant attaches to the pair, not to a name
*provenance: refines MEM-9 · spec: Memory §4.5, Stdlib appendix §2.6; cross-cuts Functions §1.4 (FN-7) · owner decision, spec#20*

**Why.** MEM-9 confines a pointer meeting an integer, and a fourth independent read of §4.5 found the
confinement bypassable: the spec nowhere requires a comparison's operands to share a type, while
Functions §1.4 (FN-7) **permits** heterogeneous bindings verbatim — "a name MAY be bound to more than one
function value in the same scope when the bindings differ in their parameter signature"
(`spec/50-functions.md:59`) — and resolution "prefers the more specific definition" (`:90-93`). So an
author may declare `eq(in a : ptr(T), in b : usize)`, and an implementer following §1.4 lets `p == 0`
bind to it, reaching the site MEM-9 refuses. The answer first written was an **ordering** — Stdlib
appendix §2.6 said the comparison is refused "before overload resolution" (the words were never in Memory
§4.5, and are removed there), which is the wrong instrument: an ordering between two rules is invisible to
a second implementation, so two conforming compilers could disagree on whether a source is well-formed —
the exact failure FND-3 exists to prevent.

**The owner's direction.** Expressing one's own types with a comparison relation is needed, and it must
follow the language's principles. A blanket same-type rule would delete that expressiveness, so it is not
adopted; an ordering rule leaves the hole open for every other type pair.

**The rule.** A comparison site is well-formed when a **comparison relation is established for the
operand pair** — the pair, not the name:
- **Same-type pairs** are established by the structural derive, exactly as today (`eq(in a : T, in b :
  T)`, `lt(in a : T, in b : T)`; Stdlib appendix §2.6).
- **Heterogeneous pairs of the author's own types** are established when the author provides a concrete
  `eq(in a : A, in b : B) -> bool` (likewise `lt`). This is **structural**, in the same sense as every
  other protocol: satisfied by *providing the operation*, not by naming an interface (Stdlib appendix §2,
  Comptime §6.2). It is **additive** (FND-6): nothing that resolves today stops resolving.
- **The pointer↔integer pair is confined by the pair itself.** The grant attaches to the operand types at
  the site, **independent of what is declared**: `p == 0` with `p : ptr(T)` requires an enclosing
  `unchecked` grant whatever relation exists, because what the site asks — whether a machine address holds
  a bit pattern — is the same knowledge fabricating an address requires (MEM-9's own argument).
  **A declaration cannot supply the grant, and neither can its body.** A relation over this pair may be
  established — even by a body whose comparison sits inside its own `unchecked { … }` — and the site still
  requires its own enclosing grant, because the requirement is a property of the operand types at the site
  and not of where some function places a block. So declaring `eq(in a : ptr(T), in b : usize)` never
  makes `p == 0` legal in checked code; inside a grant both are legal, and what is hardware-defined
  there is the operand — the fabricated address its bits name (Memory §1.4) — never the comparison,
  which is determined by the two values it is given.
  *An earlier draft of this rule argued that the declaration's body "must compare the two, and is therefore
  confined". That argument was wrong and is retracted here rather than edited out: `unchecked { bitcast(usize, a) } == b`
  inside the body satisfies it, so the bypass would have stayed open behind a sentence that read as a
  closure. The rule belongs to the site, which is why it holds whatever the declaration does.*

**Why this and not the alternatives.** It closes the bypass with a rule about the program text rather than
a rule about the order of two checks, so FND-3 holds by inspection; it keeps heterogeneous comparison
expressible for ordinary types, which the owner asked for and a same-type rule would have removed; and it
costs no new syntax, because FN-7 already admits the binding and the protocol convention already means
"satisfied by providing the operation".

**Diagnostics.** The compiler reports at the site: `seam_cmp_outside_grant_err` fires in the comparison
arm above `ty_eq`, guarded by the kind pair and `PTRINT_GRANT == 0` (compiler#1034). That location stays;
when the pair rule lands the same marker covers a declared heterogeneous pair, because the condition is the
same kind pair at the site.

### MEM-11. A value-expression gets a temporary only in a read-only receiver slot; a store into one is ill-formed — spec#47
*provenance: refines MEM-6 (places and values) and the receiver auto-ref of MOD-3/OP-5; reads OP-8 and
  Memory §5.3.1 · spec: Memory §1.6, §5.9, §8; Type System §4.5; Functions §1.4; Modules §2; Comptime
  §5.4, §11; Grammar §3.4 · owner decision 2026-11-08, spec#47*

**Question.** Memory §1.6 listed "a field access `s.f`" among the place-expressions with no condition on
`s`, so nothing said whether `f().x = 1` (with `f` returning a struct by value) stores into a discarded
copy or is ill-formed. And Type System §4.5's auto-ref passes the receiver "by its address" at a read-only
`ptr(R)` slot without requiring a place, so `make_pair().sum()`, or `vv[i].len()` and `vv[i][j]` after
OP-8 made `vv[i]` a value-expression, had no stated meaning: a temporary (with what extent), or ill-formed.

**Decision (owner, 2026-11-08).** **A value-expression in a read-only `ptr(R)` receiver slot is
materialized in a caller-owned temporary that lives to the end of the full expression, and its address
is passed** (`vv[i].len()`, `make_vec().len()`, `vv[i][j]`). **A field or index of a value-expression is a
value-expression, so a store into it (`f().x = 1`) is ill-formed.**

**Derived (not the owner's wording).** The temporary is an automatic placed by the Memory §2.5 strategy as a memory-resident
one, like a spilled binding (§2.6); "full expression" is §5.3.1's. A slice index stays a place whatever its base — the
element lies behind the view's pointer — and so does a field through a pointer base: OP-8, an earlier
decision, already rules both, and the owner's sentence is read alongside it, so "index" here is an
array's or a library aggregate's. The base is classified after the access coercion of Type System §4.5, so a
pointer-typed base auto-derefs and `make_ptr().x` is a field of a dereference, a place. A
`ptr(mut R)` slot still requires a mutable place; the rule creates no temporary for any other slot.

**Not decided here.** The **owning** case was decided afterwards by MEM-12: an owning value-expression
at a read-only receiver slot gets no temporary and the call is ill-formed, and `Vec(T)` and `String`
are `@owning` — so this decision's `make_vec().len()` example is now ill-formed. A call-rooted place (`make_slice()[0]`, `make_ptr().x`) is no store target: Grammar §3.3 derives
none, so the store is a syntax error (Grammar §5 item 3; Memory §1.6; spec#84). (An owning element of a `Vec` is not read by value at all, STD-10, so `vv[i]` with
an owning element — over a `Vec(Vec(T))` too, MEM-12 — never reaches this case.)

**Why.** A read-only receiver only needs *an* address to read through; the caller can always supply one
for a value it already holds, at the cost of one stack slot it can see in the source (I1), and the
extent is the one §5.3.1 already gives anything scoped. A store into a copy that no one can read again is
a silent no-op — the program says it writes and nothing observable is written (I3) — so it is rejected
rather than given a place.

**Rejected.** *(b) As (a), but a store into a field of a temporary is a place and is allowed* — the write
is discarded, a silent no-op. *(c) No temporaries* — `vv[i].len()` and `make_pair().sum()` ill-formed,
the program binding first (`e := vv[i]; e.len()`) for a read that needs no name.

**Consequences.** Memory §1.6 conditions the field/tuple/array-index place forms on a place base, lists
their value-base forms among the value-expressions, rejects `f().x = 1` and states the temporary rule;
Type System §4.5's auto-ref bullet, Functions §1.4's and Modules §2's auto-ref equivalences and Memory
§5.9's `a.len()` example point to it; the glossary entries follow.

### MEM-12. An owning value-expression gets no temporary; `String` and `Vec(T)` are `@owning`, released by `free` — spec#73
*provenance: completes MEM-11 (its owning case) on MEM-2's linearity; makes STD-6's "owning containers" owning
  in Memory §5.9's sense and answers part of spec#96's first item (STD-10) · spec: Memory §1.6, §5.9, §8; Type
  System §4.5; Functions §1.4; Modules §2; Comptime §5.4; Stdlib appendix §1, §2.4, §2.6, §6, §8 · owner decision 2026-11-08, spec#73*

**Question.** MEM-11 gives a value-expression in a read-only `ptr(R)` receiver slot a temporary that lives to
the end of the full expression. For an owning `R`, nothing at that end releases, transfers or `forget`s the
temporary, so Memory §5.9 rejected the call as a leak without saying so, and whether the end should
discharge the obligation was open. MEM-11's own example, `make_vec().len()`, depended on a second question:
neither `Vec(T)` nor `String` was declared `@owning`, so under §5.9 both were copyable — a copy duplicates
the buffer pointer — and neither had a release in its closed v1 surface.

**Decision (owner, 2026-11-08).** **`String` and `Vec(T)` are `@owning` types with an explicit release,
`free`, in their v1 surface** (Stdlib appendix §6). **A method call on an owning value-expression
(`make_vec().len()`) is ill-formed** — no hidden release at the end of the full expression (I3), no hidden
leak. The explicit form is `v := make_vec(); n := v.len(); v.free()`. MEM-11's temporary rule stands for
non-owning values.

**Derived (not the owner's wording).** "A method call" is read at the slot where MEM-11 makes a temporary,
the read-only `ptr(R)` receiver: a `ptr(mut R)` slot already requires a mutable place, and a by-value `in`
receiver (`make_vec().free()`) is a move, not a temporary — read literally, the sentence would refuse
`make_vec().free()` while its UFCS equivalent `free(make_vec())` (TYP-5) stays well-formed. `Vec(T)` is `@owning` for every `T` — the owner's
"`Vec(T)`" names no condition on `T`. `free` takes the container `in` (`free(in self : Vec(T))`,
`free(in self : String)`; `in` is the consume, MEM-2) and returns the buffer to the container's allocator.
The other operations that take a receiver borrow it, by `ptr(Self)` for a read and `in out` or `ptr(mut Self)` for a
change, as Memory §5.9 requires of any owning type's accessors (Stdlib appendix §1 says so beside its
`in self` convention); so does `iter`, the iterable shape's operation, as §5.1 already reconciles an
owning allocator's `mechanism`. The diagnostic is located at the receiver and names the explicit form.

**Why.** Each of the other answers hides something. Releasing the temporary at the end of the full
expression is a destructor the source does not show, and the spec has no uniform release function to
name. `forget`-ing it is a silent leak per call. Making the containers owning, rather than leaving them
copyable, is what lets the linearity checker see a buffer released twice or never: a copyable `Vec`
duplicated its buffer pointer, and with no release in its surface it could not be released at all.

**Rejected.**
- *Option 2 of spec#73: the temporary is released at the end of the full expression by its type's
  release* — a compiler-inserted destructor (I3), with no uniform release function to name; MEM-5
  likewise rejects an allocator default that nothing in the source would `close`.
- *Option 3: the temporary is `forget`-ed* — a deliberate leak that nobody wrote.
- *`Vec(T)` owning only when `T` is* (spec#96's option (a)) — derived, not the owner's stated
  rejection: his "`Vec(T)`" names no condition on `T`, and a vector of copyable elements still owns its
  buffer. The cost spec#96 listed for the unconditional form — `Vec(u8)` linear, every existing `Vec`
  use changed — is reported to him with this record.

**Consequences.** Memory §1.6 states the refusal beside the temporary rule; §5.9's auto-ref example and
§8 item 1 follow; Type System §4.5, Functions §1.4 and Modules §2 qualify their restatements of the
temporary as non-owning. Stdlib appendix §6 marks both containers `@owning`, adds `free` to both surfaces
and states the receiver directions, §1 qualifies its `in self` convention, §2.4 states the owning
iterable's `iter` receiver, §2.6 and Comptime §5.4 point to spec#106, and §8 item 5 gains the clause.
A type with a `Vec` or `String` field is owning by contagion; neither may be the underlying type of a
`@require` type (Type System §8.1) or a payload of a raw union (Memory §5.9). `Vec(String)` and
`Vec(Vec(U))` now have an owning element, so STD-10's refusals apply to them: MEM-11's `vv[i].len()` over
a `Vec(Vec(T))` is ill-formed by STD-10.

**Left open.** Three questions block decisions already made and go to the owner: what `free`, `clear` and
`truncate` do with owning elements still in the vector (spec#107); how `==` / `<` / `hash`, whose
operands are `in`, take an owning operand — STD-6's comparison of two `String`s (spec#106); and how
`for x in v` applies `iter` to its operand, so whether `for` over a `Vec` borrows it (spec#115). Latent:
whether `HashMap(K, V)` is owning and how it is released (spec#108); views into a container's buffer
across reallocation or `free` (spec#109).
