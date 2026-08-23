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
*spec: Stdlib §1–§4 (tiering, DCE, built-in catalog), Overview §6*

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
`forget`/instructions) versus ordinary prelude functions (`panic`/`exit`/`assert`). The
catalog core lives in the appendix; the per-domain families (Assembly, Concurrency) are
normative in their feature chapters, and the full catalog is the union. `assert` is an
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
a freestanding target; the richer catalogs over the shapes (iterator adapters, the
`format` template, `Display`, `format->String`) are explicitly additive and gated on the
features they need. **Error enums** are **closed v1 sets**: they grow only through an `Other`
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
