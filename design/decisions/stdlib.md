# Decisions — Standard library / prelude

**Why the prelude and standard library are the way they are.** Each entry records the
*rationale* and the *rejected alternatives*; the normative *what* lives in the spec (the
Stdlib appendix, `spec/160-appendix-stdlib.md`), the *how* in the compiler. IDs are
theme-prefixed (`STD-N`) and **append-only** — assigned in articulation order, never
renumbered. The `provenance:` line records what an entry refines and where it lands in
`spec/`.

---

## Definitions, tiers, built-ins

### STD-1. Chapter sets the framework; the appendix is normative v1 content
*provenance: the per-domain Concurrency families enumerated in the appendix · spec: Stdlib
§1–§4 (tiering, DCE, built-in catalog), Overview §6; Stdlib appendix §4.3, §4.4*

**Why.** The standard library is **not** a place to defer decisions under the banner of
"additive". The Stdlib *chapter* fixes the **framework** — the tier ordering (base <
alloc < std), dead-code elimination, the structure of built-ins, the runtime semantics
of `panic`/`exit`/`assert`, and the `@alloc` model — while the **definitions** (the
`Vec`/`HashMap`/`String` implementations, the protocol shapes, the catalog of built-ins)
are **normative v1 content in the appendix**, exactly as the per-arch tables are. The
distinction matters because of the FND-3 bar: two independent implementations must produce
compatible results without guessing, so the v1 surface must be *fully enumerated*
somewhere. "Additive" (FND-6) means **future growth** — new types/functions that do not
break v1 programs — **not** license to under-specify v1.

The **tiering + DCE** design lets one prelude serve the kernel/firmware niche and the
hosted application without a ladder of profiles: the **base prelude** (freestanding, no
allocation: `bitsN`, the scalar families, `Option`/`Result`, slice, `str`, core
operators) is injected everywhere; the **alloc tier** (`Vec`/`HashMap`/`String`) appears
only given an allocator; the **std tier** (process/I/O/time) is reached only by
qualified or explicit import. All three are **compiler-provided**, not manifest
dependencies. Because DCE is automatic — monomorphization emits nothing for the
uninstantiated, and the normative reachability rule (MOD-13) keeps everything the roots do
not reach out of the artifact, with the linker's `--gc-sections` as an additional,
non-contractual strip — the binary cost of a rich prelude is **zero**; the only price is namespace clutter, so the menu can
be generous without penalising the freestanding target.

The **built-in catalog** is partitioned, not duplicated: compiler **intrinsics** that
cannot be written by hand (`typeinfo`/`bitcast`/`ptr`/`deref`/`size`/`align`/`resolves`/`compiles`/`embed`/
`forget`/`trap`/instructions) versus ordinary prelude functions (`panic`/`exit`/`assert`). The
catalog core lives in the appendix; the per-domain families (Assembly, Concurrency,
Functions' `dyn_over`) are normative in their feature chapters, and the full catalog is the
union. The Concurrency families — `atomic::*`, `fence`, `volatile::*`, the 20 overflow policy
functions, `fma`, `saturating_convert` — have their semantics in that chapter but their
**names and signatures enumerated** in the appendix (§4.4): a family named only by a pattern
(`wrapping_*`, `fetch_*`, "…") is not an enumeration, and two implementations would guess
different sets. `assert` is an
**ordinary function** whose phase comes from context (a comptime context turns a false
assert into a compile error) — there is no separate `comptime assert` keyword (OP-1:
nothing new where context already decides).

The appendix also pins the **core protocols** as structural comptime predicates rather
than language entities: `Truthy`, `Tryable`, `Optional`, `Iterator`, `Allocator`,
`Eq`/`Ord`/`Hash`, and `Writer`. Two are load-bearing in their own right. **Numeric
scalars are not `Truthy`** — a numeric condition must be an explicit comparison
(`n != 0`), never a bare `if n` — keeping `bool` distinct from integers. And `Eq`/`Ord`/
`Hash` are **derived structurally over `typeinfo`** and overridable per type: the v1 core
ships the derived `eq`/`lt`/`hash` plus real consumers (`min`/`max`/`clamp`, in-place
slice `sort`) so the `Ord` shape is not a dangling predicate. A **minimal, alloc-free
scalar-rendering layer over `Writer`** is v1 so a program can print a *computed* value on
a freestanding target, and so is the **`{}` template** over it — `format(w, "…", args…)` to any
`Writer`, and `print` as `format` to standard output on a hosted target (STD-5); the richer
catalogs over the shapes (iterator adapters, `write_all`, `Display`, `format->String`) are
explicitly additive and gated on the features they need. **Error enums** are **closed v1 sets**: they grow only through an `Other`
variant or a versioned change, **never by adding variants** — so an exhaustive `match` over
an error type stays valid across versions.

**Rejected.** Treating the library definitions as "additive, work them out later" — that
re-opens the FND-3 ambiguity the appendix exists to close. A tier *ladder* of profiles
instead of orthogonal tiers + DCE — superseded by one prelude whose unused parts cost
nothing. A separate `comptime assert` keyword and a separate `@noreturn` attribute — both
expressible by existing means (context-driven phase; the `Never` bottom type), so OP-1
bars them.

### STD-2. A `Writer`'s error type is the implementer's own, tier-bounded
*spec: Stdlib §1 (tier discipline), §2.7 (`Writer`)*

**Why.** The draft fixed `Writer.write` to return `Result(usize, IoError)` *and*
declared that an in-memory buffer (`String`, alloc tier) satisfies `Writer`. But
`IoError` is a **std-tier** enum and `String` is an **alloc-tier** type — and the alloc
tier is available on a freestanding target with an allocator, where the std tier (and so
`IoError`) **does not exist**. A buffer-satisfiable `Writer` whose error is `IoError`
therefore makes an alloc-tier API reference a std-tier type: a **tier inversion** the
base < alloc < std ordering forbids, and a self-contradiction with the spec's own
`String` growth operations (already `-> Result(_, AllocError)`).

The fix makes **the error follow the writer, not the protocol**: `write` returns
`Result(usize, E)` where `E` is the implementer's own error type, constrained to the
writer's own tier or below. The std byte streams (`stdout`/`stderr`/`File`) use
`IoError` — an OS write genuinely fails with an OS error; an in-memory buffer uses
`AllocError` (a **base-prelude** enum) — its only real failure is exhausting its
allocator. `write_all`/`format` compose **generically over `E`** (they ride `Result`'s
`Tryable`-ness, so `?`-propagation is unaffected), and the protocol predicate relaxes to
"`write` returns `Result(usize, _)`", satisfaction staying structural.

The general rule this establishes — **tier discipline** — is the real payload: a stdlib
item's public API, protocol implementations included, references **only** types of its
own tier or below. That is the *structural reason* a tier's absence under a limit is
sound: a present tier's surface never names an absent one.

**Rejected.** **Lower `IoError` to base/alloc** — drags OS-specific failure vocabulary
(`PermissionDenied`/`BrokenPipe`/raw `errno`) into the freestanding base tier where it is
meaningless and a buffer can never produce it. **Two separate protocols** (`fmt`-writer
vs `io`-writer, Rust-style) — duplicates the sink protocol and forces `format` to pick
one or be generic anyway, against OP-1 and the single-protocol ethos. **Keep `IoError`
and exclude the buffer from `Writer`** — contradicts §2.7's own "an in-memory buffer
satisfies it" and would push `format->String` into std, breaking the alloc-tier
`format->String` gating.

---

## Process inputs

### STD-3. `args`/`env` take an allocator; their strings are region-backed
*spec: Stdlib §7*

**Why.** The draft wrote `args() -> [str]` and `env(name) -> Option(str)` with **no
allocator**, implying the returned strings live *somewhere* by themselves. Under the
language's actual memory model that is underspecified: the OS delivers the command line
and environment as a **transient byte image** (initial-stack `argv`/`envp`, or
`/proc/self/cmdline`·`environ`), and producing owned `[str]`/`str` from it **requires
storage**. With **no implicit allocator** and **no `'static`-equivalent** in the
lifetime menu (scoped / region / generational / manual), there is nowhere for that
storage to come from except an **explicit allocator**.

So `args`/`env` take an allocator, exactly as `Vec`/`String` and every allocating
operation do: `args(allocator) -> [str]` / `env(allocator, name) -> Option(str)`. The
returned views are **region-backed** — they live for the allocator's extent, the caller
owns and frees it, and a fallible allocation surfaces `AllocError` or traps per the
container convention. This is uniform with the rest of the allocating tier and has a
checkable lifetime.

**Rejected.** **Process-static `[str]`** — keeping the bare `args() -> [str]` by treating
the strings as whole-program/`@static` (like a `str` literal in `.rodata`). Two reasons:
(1) it needs a lifetime *outside* the lifetime menu — the program does not *allocate*
process inputs, so reusing the literal/`.rodata` story for OS-provided memory is a
distinct, unspecified lifetime; (2) realizing it portably means either capturing the OS
stack layout at entry (arch/OS-specific, fragile) or holding the bytes in mutable static
storage — neither is in the v1 implementable surface, and both add a hidden
whole-program-lifetime resource the explicit-allocator form avoids.

### STD-4. Comparison over `[T]` and `str` is by content, not by the `{ptr, len}` derive — compiler#1443
*provenance: refines STD-1 (eq/ord/hash derived structurally over `typeinfo`) and the walk it
  rests on, TYP-9/CT-9 (field and variant projection) · spec: Stdlib appendix §3.5, §3.6,
  §2.6, §8; Types §7; Comptime §5.4*

**Question.** Appendix §2.6 derives `Eq`/`Ord` **structurally over `typeinfo(T)`**, and a
slice is the library pair `struct { ptr : ptr(T), len : usize }` (§3.5). Applied to a
slice, that walk compares **a pointer's address bits and a length** — so two slices of
equal content at different addresses are unequal, and `a == b` becomes a question about where
the bytes live. §3.5's operation list does not name comparison at all, and §3.6 (`str`)
defines no equality anywhere in the specification. Four compiler fixtures sat on the gap —
three over `[T]` (`reject_slice_float_compare`, `reject_slice_ordering`,
`reject_slice_struct_elem_compare`) and one over arrays of `str` (`reject_tuple_str_compare`) —
and a fifth, `reject_agg_elem_compare`, compares a struct with a `str` field, the
view-as-field case. `slice_eq`, `slice_field_compare`, `str_field_eq` and
`review_slice_str_eq` already expect content comparison.

**Decision.** **Comparison over `[T]` is elementwise: `==` holds when the lengths are equal
and every pair of elements is `eq`; `<` is lexicographic over the elements, a shorter equal
prefix being less. Over `str` it is the same rule at `T = u8` — UTF-8 byte-sequence
equality and lexicographic byte order — so a literal and a computed `str` of the same bytes
are equal while their `ptr`s differ.** Each element pair goes through `T`'s own `eq`/`lt`,
never a bytewise compare of the runs (wrong for a float or a padded element). §2.6 gains a
**view arm**: wherever its walk meets a slice or a `str` — as an operand, or as a field,
element or component of an aggregate — the default `eq`/`lt`/`hash` are these content rules,
never a descent into the `{ptr, len}` pair. The rule itself is written in §3.5 and §3.6 where
the operations are enumerated. The cost is **O(n)** and it is stated in the text, because the
operator hides it; a caller who wants identity compares `.ptr` explicitly, an ordinary pointer
comparison that needs no grant (Memory §4.5).

**Rejected.** *Leaving comparison undefined over `[T]` and `str` and refusing at the site.*
It is the cheaper rule and keeps §2.6 uniform, but it makes ordinary code — "does this text
equal that text" — a compile error, and the fixtures that wanted it are programs authors
write. *The structural derive as written* was rejected outright: pointer identity as the
meaning of `==` on a view is a trap, and it is not what any caller means.

**Consequences.** §3.5 and §3.6 gain the rule and §2.6 the view arm. It is an arm of the
structural default, not a per-type override: FN-7 allows one generic definition per name, so a
second generic `eq` over `[T]` could not carry it. The five fixtures become
content-comparisons, and an implementation that compares the pair answers a wrong value.
`Hash` over `[T]`/`str` follows content too (equal values hash alike), which §2.6 already
requires of any `Eq`/`Hash` pair and now has a defined equality to be consistent with. The
owning containers `Vec(T)` and `String`, which hold a data pointer without being views, take the
same content rule through an arm of their own (STD-6).

### STD-5. The `{}` template is v1; its trigger is the first fixed `str` parameter; `print` is `format` to standard output — spec#38
*provenance: refines STD-1 (which listed the `format` template as additive) and FN-13 (the
  template trigger of Functions §7.1); defines the `print` that §7.1 and FN-13 named · spec:
  Functions §7.1; Stdlib appendix §2.7, §7*

**Question.** Three defects around one feature. Functions §7.1 called the `{}` template "the
single, normative trigger" that `format`/`print` use, while Stdlib appendix §2.7, STD-1 and the
aspect map's deferred list called the `format` template an **additive** convenience — v1 and
not v1 at once. §7.1's trigger was "a comptime-variadic's **first fixed parameter is `str`**",
but §2.7 spells `format(w, "…", args…)`, whose first fixed parameter is the writer `w`, so the
function said to use the trigger did not meet it. And `print`, named by §7.1 and FN-13's
example, was defined nowhere: no signature, tier or sink.

**Decision (owner, 2026-11-08).** **(a) The `{}` template is v1**, not additive. **(b) The
trigger is "the first fixed parameter of type `str`"**, wherever it stands among the fixed
parameters, which covers `format(w, "…", args…)`. **(c) `print` is defined in Stdlib appendix
§2.7 as `format` to standard output** — `print(t, args…)` ≡ `format(stdout, t, args…)`, with the
same template rules and result — **hosted only**: a std-tier facility, absent under
`freestanding`. The other conveniences (`write_all`, `Display` + derive, `format->String`) stay
additive.

**Why.** The template is already fully specified (holes, escapes, the count check, FN-13's
non-literal rule) over two v1 mechanisms — comptime-variadics and the scalar rendering layer —
so calling it additive left a specified, implementable feature in limbo and two answers to
"must an implementation provide it?". Naming the trigger by the parameter's **type** rather than
its **position** lets the writer come first, which is the shape every sink-generic output takes,
without reshaping `format`. Defining `print` through `format` adds no mechanism: it fixes the
sink (standard output, a std stream), and so the tier, and nothing else (PRIN-1).

**Rejected.**
- *Keep the template additive and drop the "normative trigger" wording from §7.1*: it leaves the
  most common output form unspecified for v1 and every program that prints a value non-portable.
- *Reshape `format` to `format("…", w, args…)`* so that "first fixed parameter is `str`" holds
  literally: the template would stop being next to its holes' arguments only to satisfy a
  wording, and every writer-generic signature would have to put the writer second.
- *Drop `print` from §7.1 instead of defining it*: the most-written call in example code would
  remain a name with no definition, and FN-13's example would have to change.

**Consequences.** Functions §7.1 states the trigger by type and points at §2.7; Stdlib appendix
§2.7 lists the template and `print` as v1 and the rest as additive; §7 lists `print` among the
std I/O; STD-1's summary and the aspect map's deferred list drop the template. A comptime-variadic
that must take a `str` without templating takes it in a fixed parameter after its first `str`
one, or in the pack. This changes the meaning of the shape the former §7.1 advice recommended:
a comptime-variadic whose first `str` fixed parameter is not its first fixed parameter
(`f(x : u32, s : str, args : ...)`) now templates a literal there, so such a function moves that
`str` into the pack or after another `str` parameter. A second, non-template `print(s : str)`
beside the template one does not conform (a call `print("{}")` would have two readings);
compiler#1515 tracks the compiler's. `format`'s own result type and failure behaviour, which
`print` inherits, and the v1 set of types a hole renders were not decided here (spec#46; decided
by STD-7); `format`'s tier (base, so injected unqualified) follows from existing text (Stdlib §1;
appendix §2.7); `print`'s module
path is spec#74.

### STD-6. `Vec(T)` and `String` compare by content, not by buffer identity — spec#40
*provenance: extends STD-4 (content comparison over views) to the alloc-tier owning containers;
  rests on STD-1's structural derive and FN-7 (one generic definition per name) · spec: Stdlib
  appendix §2.6, §6, §8; Comptime §5.4*

**Question.** STD-4 made `==`/`<`/`hash` over `[T]` and `str` compare content, through a view arm
in §2.6's structural derive, because comparing the `{ptr, len}` pair answers by address — a trap
no caller means. The alloc-tier `String` and `Vec(T)` (appendix §6) are not views, so they took
the plain field walk; their fields include a data pointer, so `==` over two `String`s with equal
text reached the `Pointer` leaf and compared **buffer addresses**: the same trap, one tier up.

**Decision (owner, 2026-11-08).** **`String` and `Vec(T)` compare by content: `==`/`<`/`hash`
over their contents as STD-4 does for views** — a `Vec(T)` as the `[T]` of its `len` elements, a
`String` as the `str` of its bytes; capacity, allocator and buffer address take no part. **`Vec(T)`
requires `T`'s relation** (`eq` for `==`, `eq` and `lt` for `<`, `hash` for `hash`). The cost, **O(n)**, is
stated in the text because the operator hides it. **Identity stays an explicit comparison of the
data pointer** (`v[0..v.len()].ptr`, `bytes(s.as_str()).ptr`).

**Why.** STD-4's reasoning applies unchanged: equality that depends on where a value happens to
live is not what `==` on text or on a sequence means, and a program that compares two `String`s
expects their text compared. Taking the view arm's rule keeps one meaning for "these hold the same
elements" across `[T]`, `str`, `Vec(T)` and `String`.

**Rejected.**
- *Comparison over an owning container is ill-formed in v1, refused at the site; the author
  compares the views explicitly.* Safe, but it makes the most common comparison of owned text a
  compile error with a mechanical workaround, which is ceremony rather than information.
- *Leave identity as the defined meaning.* It contradicts STD-4's rationale outright.
- *A concrete override per container.* Possible for `String`, impossible for the generic
  `Vec(T)` under FN-7, so the two containers would be specified by two mechanisms.

**Consequences.** The rule is carried by an arm of §2.6's structural default keyed on the
prelude's `Vec` and `String`, beside the view arm, not by a per-type override: FN-7 admits one
generic definition per name, and `Vec(T)` is generic, so a second generic `eq` over it could not
exist (the mechanism follows from "as STD-4 does for views" and FN-7; it is not a separate
choice). The arm exists only where the alloc tier does, and the base-tier `eq` signature names
no alloc type, so there is no tier inversion; how the base body names the alloc types was
open as spec#55 and is decided by STD-8 (the arm lives in the compiler-provided prelude built per
translation unit). Appendix §2.6 gains the containers' arm, §6 the rule with its cost and the
identity spelling, §8 item 1 the conformance clause; Comptime §5.4's note and the glossary follow.
`HashMap(K, V)`, the third owning container of §6, is not covered by this decision; its comparison
was filed as spec#50 and is decided by STD-9. MEM-12 later made `Vec(T)` and `String` `@owning`;
since `eq` / `lt` / `hash` take their operands `in`, which moves an owning value, how a comparison
takes an owning operand was spec#106, decided by MEM-13: the relations borrow an owning operand.

---

### STD-7. `format` returns `Result(usize, E)`, writes every fragment, stops at the first failure; the v1 hole set — spec#46
*provenance: refines STD-5 (`format` / `print` are v1) and STD-2 (`Writer` and its own error
  `E`); rewords FN-13's rendering clause · spec: Stdlib appendix §2.7; Functions §7.1*

**Question.** With the `{}` template v1 (STD-5), `format(w, …)` had no stated result type, and
nothing said whether it loops over short writes or stops at a failed one — so a hosted `print`
to a closed stdout had no defined outcome. And the set of types a hole renders was the scalar
layer (integers, `bool`, `char`) "or the `Display` protocol", which is additive: `str` was not
listed though FN-13 renders a non-literal `str` as one fragment, a float rendered "with a
bounded fractional part" without the bound, and `print("{}", p)` for a struct had no answer.

**Decision (owner, 2026-11-08).** (1) `format(w, …) -> Result(usize, E)`: it writes every
fragment completely, looping over short writes, stops at the first failed write and returns it;
on success, the total byte count. `print` is the same, with `E = IoError`. (2) The v1 hole set:
integers, `bool`, `char`, `str` as its bytes, and floats as the shortest decimal that reads
back to the same value; any other type is a compile-time error until `Display` is provided.
Consequences, not separate decisions: after a short write, the unconsumed remainder is offered
again (a writer that keeps returning `Ok(0)` keeps being called); a result over an owning `E`
is consumed like any owning value, not discarded (Memory §5.9); and, by Grammar §3.1 and
Functions §3.3, a call to `print` as the final element of a procedure body is a
value-expression there and makes the body ill-formed — the result is discarded explicitly
(`_ := print(…)`, SYN-9), bound or handled. Left open: whether a user brand or `@require` type over a hole type is in the set
(spec#99); which `write` calls carry the fragments (spec#100).

**Why.** The result mirrors `Writer.write` and STD-2's `?` story, so a failure is a value the
caller sees (I3, I11) and the count is kept at no cost; `print` stays exactly `format` to
stdout, as STD-5 made it. A closed hole set fixes the bytes of every v1 call (FND-3) and
leaves `Display` additive without changing any v1 program's output (I10).

**Rejected.** *A result without the count* — loses information for no gain. *`print` alone
traps or ignores a stdout failure* — a hidden failure policy (I3/I11), and `print` would stop
being `format` to stdout. *`Display` in v1* — it needs the structural derive over `typeinfo`
and was kept additive by STD-5. *A float with a fixed number of fractional digits* — loses
values or prints noise; shortest round-trip is exact and minimal.

**Left open.** The exact text of a float hole (notation, ties, integral values, non-finite and
signed zero): spec#75. The module paths of the §7 std surface: spec#74 (`format`'s tier,
base, follows from existing text). The explicit discard form for a value in final position:
settled by SYN-9 (spec#81). A writer
returning `Ok(k)` beyond its slice, and a total count past `usize`: spec#82. Whether a user
brand or `@require` type over a hole type is in the hole set: spec#99. Which `write` calls carry
the fragments (an empty fragment, joining fragments): spec#100.

---

### STD-8. The owning-container comparison arm lives in the compiler-provided prelude, built per translation unit — spec#55
*provenance: refines STD-6 (its mechanism) on STD-1's tiering (Stdlib §1: the tiers are
  compiler-provided) and FND-10/FND-11 (limits are per translation unit) · spec: Stdlib appendix
  §2.6; Comptime §5.5*

**Question.** STD-6 puts the content arm for `Vec(T)` / `String` inside the base-tier generic
`eq` / `lt` / `hash`, described as ordinary library code over `typeinfo` with "no built-in
comparison in the compiler". That body must name `alloc::Vec` / `alloc::String`, but under
`no_alloc` the `alloc` root does not exist (Stdlib §1), so the name would be an unresolved-name
error. The observable rule was settled; the mechanism a conforming base library uses to name an
absent tier's type conditionally was not.

**Decision (owner, 2026-11-08).** Option 1. **The container arm lives in the compiler-provided
prelude, built per translation unit with its limits**: a unit with the alloc tier gets a default
body with the arm, a unit without it gets one without. The aspects-map line "no built-in
comparison in the compiler" is reworded accordingly. **No tier predicate is introduced** (PRIN-1).

**Why.** Stdlib §1 already makes all three tiers compiler-provided and already builds what a
unit sees from its limits (the `alloc` root is absent under `no_alloc`), so choosing the prelude
body per unit is the same act, not a new entity. The comparison stays prelude code over
`typeinfo`; the compiler's checker and code generator gain no comparison knowledge.

**Rejected.** *A comptime tier-presence predicate* (`comptime if tier(alloc)`) usable in library
code — explicit and general, but a new language entity for one arm (PRIN-1). *The alloc tier
contributes the arm through an extension point in the base walk* — needs an extension
mechanism that FN-7's one-generic-definition rule does not obviously admit, and v1 has none.

**Left open.** Whether a unit without the alloc tier may use, and so compare, a type from
another unit whose layout reaches an alloc-tier type (STD-6's "no `Vec` for it to meet" has no
rule behind it yet): spec#101.

---

### STD-9. `==` over `HashMap(K, V)` compares entries; `<` and `hash` over it are ill-formed — spec#50
*provenance: extends STD-6 (content comparison over the owning containers) to the third
  container of appendix §6, with STD-8's arm · spec: Stdlib appendix §2.6, §6, §8; Comptime §5.4*

**Question.** STD-6 covers `String` and `Vec(T)` only. Over a `HashMap(K, V)`, `==` / `<` /
`hash` took the plain field walk, which reaches the `Pointer` leaf of its table and compares
**addresses** — the trap STD-4 and STD-6 rejected — also when the map is a field of a struct
being compared. A map has no element order, so "content" means its set of entries, and `<` has
no natural meaning.

**Decision (owner, 2026-11-08).** Option 1. **`==` / `!=` over `HashMap(K, V)` compare
content**: equal `len`, and every key of one maps in the other to an `eq` value — which needs
`V`'s `eq`. **`<` and `hash` over a `HashMap` are ill-formed at the site.** Consequences, not
separate decisions: "over a `HashMap`" covers a map reached by the structural walk of an
aggregate, as STD-6's arm does, so a struct holding a map compares by `==` and refuses `<` and
`hash` unless it overrides them; the derived `<=` / `>` / `>=` go with `lt`; and a map cannot be
a `HashMap` key (it fails §6's `Hash` requirement, since the default `hash` does not resolve
for it). The cost (O(n) expected lookups) is stated, as STD-6 does.

**Why.** It mirrors STD-6 where an order exists neither in the map nor in its meaning: the
useful relation is defined by content, and the meaningless ones are refused rather than
answered by address. One direction suffices once the lengths match, because keys are unique.

**Rejected.** *Content `==` plus an order-independent `hash`* (a commutative fold of entry
hashes) — weaker against collisions, and its exact definition would have to be fixed for FND-3.
*All comparison over a `HashMap` ill-formed* — smallest, but equality of two maps is a
reasonable question to ask.

**Left open.** Whether a concrete user `lt` / `hash` declared over a `HashMap` type itself lifts
the refusal: spec#102. The order of `V`'s `eq` calls over the entries and early exit: spec#103.

---

### STD-10. `Vec(T)` admits an owning `T`; its element is exchanged or lent, never copied out or overwritten unseen — spec#54
*provenance: applies Memory §5.9's linearity (MEM-2) to the `Vec(T)` surface of STD-1's appendix;
  restricts OP-3's index operator, OP-7's compound and OP-8's explicit form for an owning `T`; the
  scoped accessors take the arena `get`'s `scoped` pattern (Memory §5.3.1, §5.4) · spec: Stdlib
  appendix §6, §8; Types §4.5; Memory §1.6, §5.9*

**Question.** An `@owning` value is linear: non-copyable, consumed exactly once (Memory §5.9).
Appendix §6 gave `Vec(T)` a by-value read (`self[i]` ≡ `at`), an overwriting store
(`self[i] = v` ≡ `index_set`, and `set`) and `pop` → `Option(T)`, and never said whether `T` may
be owning. For `v : Vec(Fd)` with `Fd := @owning struct { fd : i32 }` the spec had no answer to whether the type is well-formed, whether
`v[i]` copies, moves or is refused, and whether `index_set` drops the element it replaces; OP-7's
`index_set(a, j, at(a, j) ⊕ rhs)` and OP-8's `mut e := a[i]; e.f = x; a[i] = e` inherit it.

**Decision (owner, 2026-11-08; option (b) of spec#54).** **`Vec(T)` admits an owning `T`.** For
such a `T`, the by-value read `at`/`v[i]` (and so a read through it, `v[i].f`), the overwriting
store `index_set`/`v[i] = x`/`set`, and the OP-7/OP-8 forms over the element are **ill-formed**,
with a diagnostic located at the use and naming the explicit operation that fits it: no hidden
copy, no hidden drop. The refusal is keyed on the prelude's own `Vec`, as STD-6's arm is. Access
goes through four operations, defined for every `T`, whose signatures use only the existing
reference and linearity rules:

- `replace(in self : ptr(mut Vec(T)), in i : usize, in x : T) -> T` — moves `x` in, the old
  element out;
- `swap(in self : ptr(mut Vec(T)), in i : usize, in j : usize)`;
- `at_ptr(in self : ptr(Vec(T)), in i : usize) -> scoped ptr(T)` and
  `at_mut(in self : ptr(mut Vec(T)), in i : usize) -> scoped ptr(mut T)` — `scoped` results, so
  in checked code the reference is transient within the call's full expression and the vector is
  a frozen borrowed place there (Memory §5.3.1, the arena `get`'s pattern, Memory §5.4).

All four are bounds-checked like `at` (trap, CG-13). `pop` stays well-formed: it moves the last
element out. The names `at_ptr`/`at_mut` and the bounds-check behaviour follow the existing
surface (`at`'s check; the `ptr`/`ptr(mut)` pair) and are not part of the owner's choice.

**Not decided here** (each changes well-formedness or meaning, so each is filed for the owner):
whether `Vec(T)` of an owning `T` is itself owning and how it is released (decided afterwards by
MEM-12: `Vec(T)` is `@owning` for every `T`, released by `free`; what `free`, `clear` and
`truncate` do with owning elements still held was spec#107, decided by STD-11: they trap unless the
elements are drained), `push`'s failure path
for a moved-in element, and the owning case of `get`/`iter` (`clear`/`truncate` decided by STD-11) and of
`==`/`<`/`hash` (STD-6's arm calls `T`'s relation, whose operands are `in`) — spec#96, the
comparison decided by MEM-13 (spec#106: the relations borrow an owning operand); a
whole-value move out of, or store over, an owning place reached through a pointer or a view
(`deref(v.at_mut(i)) = x`, an element of the sub-slice `v[a..b]`) — spec#97.

**Why.** An owning-element container is the ordinary way to hold several resources (`Vec(Fd)`),
so refusing it (option (a)) would push every such program to a fixed array or a hand-rolled
container. Refusing only the operations that would copy or drop unseen keeps linearity
syntactic and local (FND-3): every element leaves through a named operation that hands it to the
caller (`replace`, `pop`), and every change in place goes through a reference whose lifetime the
existing `scoped` rule already bounds (I3, I11). Nothing new is added to the language: the four
operations are ordinary prelude functions over `ptr`/`ptr(mut)` slots, `in` moves and a
`scoped` result. Like `pop`, their bodies move elements within the vector's own storage, which
is the prelude's implementation, not a user-level move through a pointer (that is spec#97). For a copyable `T` every existing form keeps its meaning and the four operations
are added: confining them to an owning `T` would have needed a comptime ownership predicate,
which v1 did not have when this was decided. STD-12 has since added one (`typeinfo(T).owning`,
Comptime §5.1); it does not change this entry, and the four stay defined for every `T`. So OP-8's deferred
place-returning access now exists for every `T`, as the `scoped` `at_mut`.

**Rejected.**
- *(a) `Vec(T)` requires a copyable `T` in v1* (a refusal of an owning `T`; an owning-element
  container as additive growth) — safe, but it leaves the commonest owning collection inexpressible.
- *(c) `at` is a move-out that leaves a hole to be refilled* — needs a rule for the hole (a
  per-slot initialisation state the checker cannot see through indices), the flow analysis
  MEM-1 and Memory §5.3.1 exclude.

**Consequences.** Appendix §6 gains the four operations in `Vec`'s list, the paragraph defining
them and the owning-element rule with its explicit forms; §8 item 5 the conformance clause; Types
§4.5 the owning case of the index forms beside OP-8; Memory §1.6 the owning case beside OP-7's
lowering; Memory §5.9 the sentence beside the arena's
accessors. `at_mut` is the place-returning access OP-7 and OP-8 deferred, in its `scoped` form:
the reference cannot outlive the full expression, which answers their question of its validity
across a reallocation without a new rule; OP-3's index operator is unchanged and still lowers to
`at`/`index_set`. `String` was not declared `@owning` when this was decided, so `Vec(String)` had
a copyable element; MEM-12 has since made `String` and `Vec(T)` `@owning`, so `Vec(String)` and
`Vec(Vec(U))` are vectors of an owning element under this decision.

---

### STD-11. `free` / `clear` / `truncate` over a `Vec` of an owning `T` require the elements gone, else trap — spec#107
*provenance: completes MEM-12 (`Vec(T)` owning for every `T`, released by `free`) and STD-10 (an owning
  element) on Memory §5.9's linearity; refines CG-13 (a third every-mode `trap()`) · spec: Stdlib
  appendix §3, §6, §8; Stdlib §4.1 · owner decision 2026-11-09,
  spec#107*

**Question.** `free(in self : Vec(T))` returns the buffer. For an owning `T` (`Vec(String)`,
`Vec(Fd)`), nothing said what happens to elements still in the vector: releasing them needs a release
function per element type, which the spec does not have (a hidden destructor, I3); dropping them is a
leak the linearity checker cannot see, since the length is a run-time fact. `clear` and `truncate(n)`
raise the same question.

**Decision (owner, 2026-11-09).** **For an owning `T`, `free`, `clear` and `truncate` on a `Vec(T)`
require the affected elements to be gone (drained with `pop`, each consumed explicitly); otherwise they
fail through `trap()` in every verification mode. For a non-owning `T` they work unconditionally.**

**Derived (not the owner's wording).** The affected elements are all of them for `free` and `clear`
and those past `n` for `truncate(n)`, so the conditions are `len() != 0` and `len() > n`. "Every
verification mode" means `unchecked` does not drop the check, as for `@alloc` exhaustion (Stdlib
appendix §5.2.1).

**Why.** Every element's release is written in the source (I3), nothing new is added to the surface
(PRIN-1), and the trap fails at the operation rather than leaking silently (I11).

**Rejected.** *`free` takes the per-element release* (`free_with(in self, release : fn(in T))`) — a new
operation in the closed v1 surface, which the `pop` loop already expresses. *`free` / `clear` /
`truncate` over an owning `T` are ill-formed; drain and `forget` the empty vector* — `forget` as the
normal release leaks the buffer itself, so it would need a second, buffer-only release.

**Refined (STD-12, owner decision 2026-11-09).** Whether `T` is owning is decided at comptime by
`typeinfo(T).owning`, which guards the check, so `Vec(u8).free()` carries none (I2).

---

### STD-12. `typeinfo(T)` reports whether `T` is owning; STD-11's check is guarded by it at comptime — spec#133
*provenance: extends CT-6 (`typeinfo(T)`'s member surface) with Memory §5.9's classification (MEM-2);
  refines STD-11 (its check made per instantiation) · spec: Comptime §5.1, §11; Stdlib appendix §3.1,
  §3.2, §4.1, §6, §8 · owner decision 2026-11-09, spec#133*

**Question.** STD-11 makes `free` / `clear` / `truncate` over a `Vec` of an owning `T` trap while an
element is held, and lets them work unconditionally for a non-owning `T`. Nothing said how the prelude's
`Vec(T)` body tells the two apart: `typeinfo(T)` carried no ownership fact, and v1 had no other comptime
predicate for it (STD-10 noted its absence), so `Vec(u8).free()` could carry a run-time length check
that serves no purpose (I2).

**Decision (owner, 2026-11-09; option 1 of spec#133).** **`typeinfo(T)` exposes whether `T` is owning,
as a comptime `bool` member, `true` for an `@owning` type and for a type owning by contagion (Memory
§5.9). STD-11's emptiness check is guarded by it at comptime, so a `Vec` of a non-owning `T` carries
no run-time check.**

**Derived (not the owner's wording).** The member is named **`owning`**, the adjective Memory §5.9 uses,
as `Field` and `Pointer` name theirs `mutable`. `TypeInfo` is an enum whose member surface is its cases'
fields (Comptime §5.1), so `owning : bool` is a field of **every** case — `typeinfo(T).owning` then
reads for any `T`, a scalar, pointer, `str` or (well-formed) raw union always giving `false`. Its value is
the classification the linearity checker already makes, by reference to Memory §5.9; this entry adds
no rule for which types are owning. The guard is `comptime if typeinfo(T).owning` in the three bodies.

**Why.** The ownership fact is a comptime fact of the type, already computed structurally (FND-3);
exposing it through the one reify (CT-6) adds no query builtin, and selecting the check at comptime keeps
the non-owning instance free of it (I2). Any collection an author writes can make the same choice the
prelude's `Vec` makes, so the prelude's `Vec` is not a special case.

**Rejected.** *A compiler-private decision inside the prelude* (as for spec#55, STD-8, where the
compiler-provided prelude supplies the owning-container arm) — it would make `Vec` a special case an author's own collection
cannot express.
