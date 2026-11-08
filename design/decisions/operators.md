# Decisions — Operators, indexing, member access, conversions

**Why the language is the way it is** for the operator surface. Each entry records the
*rationale* and the *rejected alternatives*; the normative *what* lives in the spec
(linked per entry), the *how* in the compiler. IDs are theme-prefixed (`OP-N`) and
**append-only** — assigned in articulation order, never renumbered. The `provenance:` line
records what an entry refines and where it lands in `spec/`.

---

## The operator model

### OP-1. Operations are overloadable builtins, not keywords
*spec: Types §4.5, §5.3; spec/20-types.md*

**Why.** Reserved words are spent only on what **cannot** be expressed as a function —
control flow and declarations. Everything else — operators (`+`, `==`), `bitcast`,
`panic`/`exit`/`assert`, `T.size()`/`T.align()` — is an ordinary **built-in function
in the prelude**. This keeps the grammar small (operators do not each grow new syntax),
makes operations **overridable per type**, and through UFCS gives both spellings for free
(`f(a, b)` ≡ `a.f(b)`). An operator glyph is therefore just a name for a library
function; signedness, multi-word arithmetic and user vector types all reuse the one
"operator = sugar over a function" mechanism rather than each carving out compiler magic.
This is the cross-cutting operator model the rest of this theme builds on (and the
minimalism rule — do not introduce what existing mechanisms already express — that
governs the whole spec).

---

## Operator families and compound assignment

### OP-2. Bitwise with symbols, logical with words; compound assignment is pure sugar
*spec: Types §4.5, §5.3; spec/20-types.md*

**Why.** Two operator families are split **by essence**, not by an arbitrary either-or:

- **Bitwise** (`&` `|` `^` `~`) on `bitsN`/integers are **eager** operation-functions
  (overridable; on `bitsN` they are kernel operations).
- **Logical** (`and` / `or` / `not`) on `bool` **short-circuit** — they are lazy
  **control flow**, *not functions*, so they are keywords (OP-1 permits words for
  control flow). Short-circuiting is *visible* control, so nothing is hidden.

Drawing the line by eager-function-vs-lazy-control removes the C `&`/`&&` footgun: there
is one bitwise `&` and one logical `and`, never two glyphs that silently differ in
evaluation. Comparisons (`==` `!=` `<` `>` `<=` `>=`) are eager functions yielding
`bool`; `!=` is its own token and does not clash with the word `not`.

**Compound assignment** `place ⊕= rhs` rides this same glyph surface as **pure sugar**,
not a new mechanism: it lowers to `place = place ⊕ rhs` with the **place evaluated once**
(its address taken once into a fresh temporary, so a side-effecting `a[f()] ⊕= e` runs
`f()` exactly once; a library aggregate's index has no address, and its compound form is
OP-7's `index_set`/`at` lowering). `⊕` ranges over **exactly the binary glyph operators**
`+ - * / % & | ^` — the compound set **mirrors** the operator glyphs, it does not extend
them. Hence by construction: **no** `and=`/`or=` (those are short-circuit control, not
functions); **no** `<<=`/`>>=` (shifts have no glyph — call/UFCS form only); **no** clash
with `== != <= >=`. Because `⊕` is an overloadable operator-function, `⊕=` works for
**any** type defining `⊕` (multi-word `u128`, a user vector) with **no separate overload
point**, and stays a **statement** like `=` (so no `x = (y += 1)`).

**Rejected.** A dedicated in-place hook `op_assign(in out a, in b)` — a second overload
surface per type, against single-mechanism minimalism; the in-place optimization is left
to the compiler over the desugared form. Also rejected: making compound assignment an
**expression** — inconsistent with `=`, and it buys nothing.

---

### OP-6. Bit shifts and rotations are the named operation-functions `shl`/`shr`/`rotl`/`rotr`
*spec: Types §2.2, §3.2; spec/20-types.md; spec/160-appendix-stdlib.md §4.3*

**Why.** OP-2 fixed that shifts and rotations have **no glyph** (there is no `<<`/`>>`; the compound set
mirrors only `+ - * / % & | ^`). This entry pins **what they are** — the piece OP-2 deferred — so two
implementations agree without guessing (FND-3). They are ordinary **operation-functions** in **call/UFCS
form** (`shl(v, n)` ≡ `v.shl(n)`), overridable per type (TYP-5), `@inline` over the target's shift/rotate
**instruction intrinsic** (§3.1's operator-lowering mechanism) — exactly the `+`-is-library-over-`add`
model, with **no glyph and no new mechanism**.

- **Names + arity.** `shl` / `shr` (shift left / right) and `rotl` / `rotr` (rotate left / right), each
  `op(in v : T, in n : usize) -> T`, where `T` is `bitsN` or a numeric interpretation `uN`/`iN` of it. The
  count `n` is a **`usize`** — Alatyr's pervasive count/index type (lengths, `at`), not a new width. There is
  **no** `sar`/`asr` surface name: signedness is not spelled into the operator (see below).
- **Signedness selects the intrinsic, not the name (§3.2).** `shl` fills vacated low bits with 0 for every
  interpretation. `shr` **selects** its intrinsic from the operand's interpretation — `bitsN`/`uN` → a
  **logical** right shift (vacated high bits ← 0); `iN` → an **arithmetic** right shift (vacated high bits ←
  the sign bit, so `shr(i8(-8), 1) == -4`). This is the same "signedness lives in operations" rule that makes
  `u32 +`/`<` and `i32 +`/`<` select different intrinsics over the shared block — **no** separate `sar`
  operator, so a value's type alone fixes the shift's meaning.
- **Over-width shift → trap (I11).** A shift count `n ≥ N` (the operand's bit width) is a member of the
  **checked-guard family** already fixed by Concurrency §6 / Assembly §80 (alongside div-by-zero): checked →
  **trap**; inside an `unchecked` scope it drops to the target's hardware behavior (never UB — I11). Nothing
  new is added to the checked family; shifts join the guard that already names them.
- **Rotation is total — no over-width trap.** `rotl`/`rotr` lose no bits (vacated bits ← the bits shifted out
  the other end), so **every** count is meaningful; the count is taken **`mod N`** (`rotl(x, N) == x`). Hence
  rotation carries **no** checked guard — the mod is defined for all `n`.

**Rejected.** A `<<`/`>>` glyph pair (OP-2's C-footgun + a fourth precedence tier the layered grammar
avoids). A signed-shift **name** (`sar`/`asr`) — it re-spells into the operator what the operand type already
carries, against the §3.2 "signedness lives in operations" rule. A saturating/wrapping count instead of the
over-width trap — inconsistent with I11's correct-or-trap default (the wrap is exactly what `unchecked` buys).

---

## Indexing

### OP-3. Indexing `a[i]` is an overloadable operator — sugar over `at` / `index_set` / `index_range`
*spec: Types §4.5, §5.3; spec/20-types.md*

**Why.** The stdlib promised `v[i]` and `v[a..b]` on library `Vec(T)` (a `str` index is the
built-in slice read, OP-9), but the
operator material specified `[]` only for **built-in arrays/slices** — so an implementer
could not tell **which function `v[i]` desugars to** for a library type (a
reproducibility defect: the surface was mandated, the dispatch unspecified). The fix
makes `[]` one of the **overloadable operators**, uniform with `+`/`==`: the postfix
index forms are sugar over ordinary prelude functions resolved by UFCS over the base's
type — `a[i]` ≡ `at(a, i)` (the bounds-checked element read; out-of-range traps — for
arrays and slices by the compiler's operation-site guard (CG-13), for a library aggregate
by its own mode-dependent check (CT-11), failing through `trap()` like the built-in guard
(CG-13)); `a[i] = v` ≡
`index_set(a, i, v)`; `a[lo..hi]` ≡ `index_range(a, lo, hi)` (the sub-slice view).
Arrays and slices supply these as **compiler-known built-ins** (bounds check + pointer
arithmetic emitted directly — the floor, exactly as a native operator's one-instruction
body is); a library aggregate supplies its **own** `at`/`index_set`/`index_range`, so
`v[i]` is `v.at(i)` (receiver auto-ref borrows `v`). The read arm standardizes on the
container's **already-exposed trapping read `at`** (distinct from `get`'s `Option`), so no
new entity is minted and **one indexing surface spans built-in and library sequences** —
reusing the existing "operators are sugar over functions" mechanism (OP-1), naming no new
entity beyond the two range/write prelude names, with no grammar growth.

**Rejected.** **`[]` built-in-only** (drop `self[i]` from `Vec`) — a real loss of
the uniform-operator surface already committed to for `+`/`==`; the overloadable-operator
model makes `v[i]` free. **A dedicated `Index` protocol type** (a nominal interface) —
Alatyr's protocols are *structural* comptime predicates and operators dispatch by plain
overloadable-function names, so a nominal interface would be a redundant entity. **One
read function for both read and write** (return a mutable place) — Alatyr has no
first-class return-a-place form for a library type (second-class references), so read
(`at` → value) and write (`index_set`) stay distinct, matching `get`/`set`. **A separate
`index` read name** distinct from `at` — pure duplication (identical trapping-read body),
so the operator resolves directly to `at` and the per-element `index → at` call vanishes.

---

### OP-7. Compound assignment on a library aggregate's index is `index_set(a, j, at(a, j) ⊕ rhs)` — spec#34
*provenance: refines OP-2 (compound assignment, place evaluated once) on OP-3 (the index
  operator over `at`/`index_set`) · spec: Memory §1.6; Types §4.5; Grammar §3.2; Stdlib
  appendix §6*

**Question.** Memory §1.6 and OP-2 lower `place ⊕= rhs` by taking the place's address once
into a temporary. On a library aggregate (`Vec(T)`), `a[i]` is the by-value call `at(a, i)`
(OP-3): the element has no address, so that lowering cannot apply, and `v[i] += 1` on a
`Vec(i64)` had no specified meaning. Memory §1.6 also listed "an index `a[i]`" among the
place-expressions without qualification, contradicting Types §4.5's by-value read.

**Decision (owner, 2026-11-08).** **Defined sugar.** On a library aggregate,
`a[i] ⊕= rhs` ≡ `index_set(a, j, at(a, j) ⊕ rhs)`, with `a` and `i` each evaluated exactly
once into temporaries (`j` for `i`; `a`, when it is a place, by its address as in Memory
§1.6), in the order `a`, `i`, the `at` read, `rhs` (Functions §5.4). The calls take their
receiver as any call does, so `a` must meet what `index_set`'s receiver slot requires, exactly
as for `a[i] = v` (this follows from Types §4.5; it is not part of the owner's choice).
Memory §1.6's "an index `a[i]`" is qualified to arrays and slices; an index on a library
aggregate is listed among the value-expressions, as the by-value call it is. On an array or slice nothing changes: the element is a place and the
address-temporary rule applies.

**Why.** It costs exactly what the hand-written `a[i] = a[i] ⊕ rhs` costs — the same `at`
and `index_set` calls (I2) — and it evaluates `i` once where the hand-written form evaluates
it twice, which is what "the place evaluated once" promises for arrays (OP-2). The lowering
is written in the specification, so it is not hidden (I3; an implicit convenience with its
explicit form still expressible, CG-11). And it keeps one indexing surface across built-in
and library sequences (OP-3): `xs[i] += 1` means the same for an array and a `Vec`. Unlike
a field write through the index (refused, OP-8), it copies nothing beyond what the
hand-written form copies, so OP-8's I2 argument does not apply here.

**Rejected.**
- *Refuse `a[i] ⊕= rhs` on a library aggregate*, leaving `a[i] = a[i] ⊕ rhs`. It splits the
  indexing surface between arrays and library sequences for no saving, and the explicit form
  evaluates `i` twice.
- *A place-returning index operation* (`index_ptr`) to make the element addressable so OP-2's
  rule applies unchanged. It enlarges OP-3's function set and needs its own rule for the
  pointer's lifetime across reallocation; it is deferred as additive growth (OP-8).

---

### OP-8. A write into a library aggregate's element through its index is refused — spec#23
*provenance: refines OP-3 (the index operator over `at`/`index_set`) with OP-7's
  classification of the index as a value-expression; the method case follows from MOD-3/OP-5's
  receiver rule · spec: Types §4.5, §6.4; Memory §1.6; Grammar §3.3, §3.4*

**Question.** Types §4.5 gave a library aggregate three index forms — the read `a[i]` ≡
`at(a, i)` (by value), the write `a[i] = v` ≡ `index_set(a, i, v)`, and the range — but said
nothing about a field write **through** the index, `a[i].f = x`, when the element is a
struct. On an array the element is a place and the write lands in place; on a `Vec` the
element read is a call result. Three readings were open: refuse; desugar to
`mut e := a[i]; e.f = x; a[i] = e`; or add a place-returning index function.

**Decision (owner, 2026-11-07).** **Refused.** On a library aggregate, a store or compound
assignment whose target reaches into the element's own storage — a field or raw-union member, tuple component or
array index of the element at any depth: `a[i].f = x`, `a[i].f ⊕= x`, `a[i].f.g = x` — is
ill-formed, with a located diagnostic that names the explicit form
`mut e := a[i]; e.f = x; a[i] = e`. A mutating method on the element itself, such as
`a[i].push(x)` with a `ptr(mut R)` slot for the element type `R`, is refused too; that
already follows from Types §4.5's receiver rule (auto-ref to `ptr(mut R)` requires a
mutable place, and the element is a value). The whole-element forms `a[i] = v` and
`a[i] ⊕= rhs` (OP-7) stay. A path that leaves the element through a dereference — an
explicit `deref`, the auto-deref of a pointer-typed element or component, an index on a
slice-typed element or component — writes the pointee, a place by Memory §1.6, not the element, so it is outside the
refusal: `vs[i].f = x` on a `Vec(ptr(mut S))` copies nothing and is well-formed.

**Why.** Nothing implicit happens to the program. Read-modify-write sugar would copy the
whole element twice to change one field, above what hand-written code costs (I2): a program
that batches several field changes into one `e`, or already holds the element, pays less,
and the sugar would hide the two copies behind a one-field assignment. And refusal can
later be relaxed additively (I10); a defined sugar could never be withdrawn.

**Rejected.**
- *Read-modify-write sugar* (`a[i].f = x` ≡ `mut e := a[i]; e.f = x; a[i] = e`) — the cost above,
  hidden behind a field assignment's syntax.
- *A place-returning index operation* (`index_ptr` → `ptr(mut T)`, a third function, distinct
  from OP-3's rejected single read-write function) — deferred as additive growth: it enlarges OP-3's function set and needs its own decision on how long the pointer
  stays valid across a reallocation.

### OP-9. A `str` index is the built-in slice read plus the code-point-boundary check — a read-only place — spec#53
*provenance: refines OP-3 (indexing) and OP-8 (the place/value split for an index); reads Stdlib appendix
  §3 / §3.6 and CG-13 · spec: Stdlib appendix §3, §3.6; Memory §1.6; Type System §4.5; Stdlib §4.1*

**Question.** The Stdlib appendix's `str` entry said "where `str` byte access is library code rather
than the built-in slice read", admitting both. Since OP-8, Memory §1.6 makes an index on an array or
slice a place and an index on a library aggregate a value-expression (the by-value `at`), so whether
`ptr(s[0])` is well-formed depended on that implementation choice (FND-3).

**Decision (owner, 2026-11-08).** **A `str` index `s[i]` is the built-in slice read over the `str`'s
bytes plus the code-point-boundary check: a read-only place, like a `[u8]` element.** `ptr(s[0])` is
well-formed and yields a `ptr(u8)`; a store `s[i] = b` is ill-formed, a `str` view admitting no write.
The bounds and code-point-boundary checks are the compiler's guards at the operation site, as the
built-in index's bounds guard is (CG-13) — the boundary condition is part of the index's bounds
guard, so every list of the guard family that names bounds covers it — and `unchecked s[i]` drops it,
as `unchecked` drops every checked guard (CG-7). The "where … rather than …" alternative is deleted.

**Why.** A `str` *is* a slice of UTF-8 bytes (Type System §7); reading one of its bytes is the slice
read, and making it a library call would give the same bytes a second classification and leave the
address of a byte reachable only through `s.bytes()`. One extra check on the same built-in keeps the
access 1:1 with what a reader sees (I1).

**Rejected.** *`str` as a library aggregate*, `s[i]` ≡ `at(s, i)` a value-expression, with
`ptr(s[i])` ill-formed and a byte's address reached through `s.bytes()` — a second classification for
the bytes of a slice, for no capability gained.

**Consequences.** Stdlib appendix §3.6 states the rule; §3 lists the `str` index among the
compiler-guarded built-ins and drops `str` byte access from the library-check examples, as do Stdlib
§4.1, CG-13, CT-11 and the glossary (`String` defines no index of its own, §6; its bytes are read
through `as_str()`, which is this built-in). Memory §1.6 lists a `str` index among the
place-expressions; Type System §4.5 names it beside the array/slice built-ins.

---

## Member access on unions

### OP-4. Raw-union member access reuses constructor, assignment, and projection syntax
*spec: Types §6.3; spec/20-types.md · completed by TYP-11*

**Why.** A raw union needs no union-specific operator. TYP-11 completes this decision:
`U.m(args)` is the existing **type-qualified member constructor**, `u.m = payload` uses
ordinary assignment syntax for the whole typed member place, and `u.m` is the named
projection; TYP-11 specifies that activation's clear-plus-component-copy lowering.
The earlier `u.m(value)` write spelling is **superseded** because Grammar §3.4 makes a call
on a value `u` unconditionally UFCS (`m(u, value)`); retaining it would make the same token
sequence mean two things. Checked projection requires the member to be definitely active;
`unchecked u.m` admits raw reinterpretation. The complete arity, tuple-payload, layout,
padding, active-state, comptime/static, and unsupported-representation contract is TYP-11.

**Rejected.** `bitcast`-only reads — heavier than a typed named projection once the
checked/unchecked active-member rule is explicit. A new union-specific keyword — the
constructor, assignment, projection, and `unchecked` scope already express every operation.

---

## Receiver / access coercions

### OP-5. Auto-deref — a pointer in a value-access position reaches the pointee (dual of auto-ref)
*spec: Types §4.5, §6.1, §6.4; spec/20-types.md*

**Why.** Pointer-heavy programs (trees, graphs, allocators) read fields and call methods
*through* a pointer constantly; writing `deref(p).field` / `deref(p).m(args)` at
every access is needless verbosity. **Auto-deref** is the symmetric **dual of the receiver
auto-ref**: where auto-ref lifts a value to a scoped reference (`a` → `ptr(a)`) when
a slot wants a pointer, auto-deref lowers a pointer to its pointee (`p` → `deref(p)`)
when a **value-access position** wants the pointee. The two are one coercion mechanism
chosen by base-type vs. wanted-type, resolved **exact match → auto-ref → auto-deref**, and
applied **exactly one level** — a pointer-to-a-pointer derefs once, and reaching further
is an explicit `deref`, so deref *depth* is never hidden. It applies at the three
value-access positions: field `p.field` (unconditional — a pointer has no fields of its
own), index `p[i]` (then the index operator resolves on the pointee, OP-3), and UFCS
method `p.m(args)` (the receiver passes through as-is when the method's slot is itself a
pointer — e.g. `p.free()` — else auto-derefs; a method matching the pointer as-is wins
over one reached by deref, so an explicit pointer-receiver method is never silently
bypassed). On **owning** values, auto-deref **reads/borrows** through the pointer
(`deref` of a scoped reference inspects) — it never moves or consumes; it is the read
path, the mirror of auto-ref's borrow purpose. It borrows **no operator glyph** (every
punctuation glyph is already an operator naming a library function, whereas deref is the
`deref` primitive) and adds **no syntax** — it is a type-directed postfix coercion in
the same family as auto-ref and UFCS.

**Rejected.** Treating auto-deref as "merely saving a name" (the minimalism objection) —
rejected because, by exact parity with the already-accepted auto-ref sugar, it is the
**missing half of one coercion mechanism**, not a second mechanism. A dedicated deref
operator glyph (`.*` / `p^`) — rejected: it would collide with the `mul`/`xor` operator
glyphs, and deref is a primitive (`deref`), not a library operator-function.
