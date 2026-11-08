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
`print` inherits, its tier, the module paths of `format` and `print`, and the v1 set of types a
hole renders are not decided here (spec#46).

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
no alloc type, so there is no tier inversion; how the base body names the alloc types is
open as spec#55. Appendix §2.6 gains the containers' arm, §6 the rule with its cost and the
identity spelling, §8 item 1 the conformance clause; Comptime §5.4's note and the glossary follow.
`HashMap(K, V)`, the third owning container of §6, is not covered by this decision; its comparison
is filed as spec#50.
