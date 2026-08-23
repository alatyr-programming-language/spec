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
`f()` exactly once). `⊕` ranges over **exactly the binary glyph operators**
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

**Why.** The stdlib promised `v[i]` and `v[a..b]` on library `Vec(T)`/`str`, but the
operator material specified `[]` only for **built-in arrays/slices** — so an implementer
could not tell **which function `v[i]` desugars to** for a library type (a
reproducibility defect: the surface was mandated, the dispatch unspecified). The fix
makes `[]` one of the **overloadable operators**, uniform with `+`/`==`: the postfix
index forms are sugar over ordinary prelude functions resolved by UFCS over the base's
type — `a[i]` ≡ `at(a, i)` (the bounds-checked element read; out-of-range traps, the guard
being library code, mode-dependent like any checked operator); `a[i] = v` ≡
`index_set(a, i, v)`; `a[lo..hi]` ≡ `index_range(a, lo, hi)` (the sub-slice view).
Arrays and slices supply these as **compiler-known built-ins** (bounds check + pointer
arithmetic emitted directly — the floor, exactly as a native operator's one-instruction
body is); a library aggregate supplies its **own** `at`/`index_set`/`index_range`, so
`v[i]` is `v.at(i)` (receiver auto-ref borrows `v`). The read arm standardizes on the
container's **already-exposed trapping read `at`** (distinct from `get`'s `Option`), so no
new entity is minted and **one indexing surface spans built-in and library sequences** —
reusing the existing "operators are sugar over functions" mechanism (OP-1), naming no new
entity beyond the two range/write prelude names, with no grammar growth.

**Rejected.** **`[]` built-in-only** (drop `self[i]` from `Vec`/`str`) — a real loss of
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
