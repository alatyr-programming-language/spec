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
now matches them. The measured cost is stated rather than discovered later: `scripts/ptrint_census.baseline`
records its 3 places as "all OUTSIDE an `unchecked` grant", 2 of them pointer→integer (`RESULT-TAIL p2i`,
in `test/`), and those 2 become illegal — they are fixtures, not stdlib, and the compiler repo files the
fix as its own change with its own gate.

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
bind to it, reaching the site MEM-9 refuses. §4.5 had answered this with an **ordering** ("refused
before overload resolution"), which is the wrong instrument: an ordering between two rules is invisible to
a second implementation, so two conforming compilers could disagree on whether a source is well-formed —
the exact failure I1 and FND-3 exist to prevent.

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
  makes `p == 0` legal in checked code; inside a grant both are legal and hardware-defined.
  *An earlier draft of this rule argued that the declaration's body "must compare the two, and is therefore
  confined". That argument was wrong and is retracted here rather than edited out: `unchecked { bitcast(usize, a) } == b`
  inside the body satisfies it, so the bypass would have stayed open behind a sentence that read as a
  closure. The rule belongs to the site, which is why it holds whatever the declaration does.*

**Why this and not the alternatives.** It closes the bypass with a rule about the program text rather than
a rule about the order of two checks, so I1 holds by inspection; it keeps heterogeneous comparison
expressible for ordinary types, which the owner asked for and a same-type rule would have removed; and it
costs no new syntax, because FN-7 already admits the binding and the protocol convention already means
"satisfied by providing the operation".

**Diagnostics.** The compiler reports at the site: `seam_cmp_outside_grant_err` fires in the comparison
arm above `ty_eq`, guarded by the kind pair and `PTRINT_GRANT == 0` (compiler#1034). That location stays;
when the pair rule lands the same marker covers a declared heterogeneous pair, because the condition is the
same kind pair at the site.
