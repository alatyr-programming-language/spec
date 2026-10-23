# Changelog

This specification is **living and versioned**: it does not stop once a compiler exists.
Implementations cite a version, so every change that an implementer could notice is
recorded here.

## Versioning policy

The version string is `MAJOR.MINOR.PATCH`, but the three digits do **not** all describe
the document. They describe two different things, and keeping them apart is the whole
point of the scheme:

- **MAJOR is the version of the language**, not of the text. It is `1`: this document
  specifies Alatyr v1. It moves only if a program valid under the previous language
  version stops being valid — which `I10` and `PRIN-2` forbid, since growth within v1 is
  additive by construction. It is therefore expected to stay `1`.
- **MINOR is a normative revision of the document** — a gap closed, a collision resolved,
  a rule made explicit, an entry added to an enumeration. An implementer must re-read.
  Such a change can alter what an implementation must *do* without invalidating any
  existing program; that is exactly the additive growth `I10` permits. During review this
  is the digit that moves.
- **PATCH is editorial only** — typos, wording, cross-references, added examples. No
  conforming implementation can observe the difference.

This is deliberately not plain library SemVer. A specification has no "0.x, anything may
change" phase in the SemVer sense: the language is specified in full, and what is in
motion is the precision of the text, not the shape of the language. Pinning MAJOR to the
language version is what keeps the number meaningful instead of making every substantive
correction look like a breaking release.

**Maturity is a separate axis, and it is not in the version string.** Whether the
document is *accepted* is stated where a reader actually meets it: every chapter carries
`Status — draft (under review)`, and the Status block in `README.md` and
`spec/00-overview.md` says the same. Acceptance is recorded by flipping those lines and
by an entry here — not by a version bump, and not by a `-draft` suffix. A suffix would
say a third time what the Status line already says, and would freeze MINOR/PATCH during
exactly the phase in which the most revisions happen.

So `1.0.0` does **not** claim the text is settled. It says: revision 1.0.0 of the
specification of Alatyr v1. Read the Status line for how far along it is.

**Revisions, not commits.** The number moves when a revision is *published*, not when a
commit lands. Entries accumulate under *Unreleased* and are released together; each
release is tagged `v<version>` in git, which is what makes a citation resolve to exact
text. The procedure is in `CONTRIBUTING.md`.

Each entry names the affected sources of truth, because a normative change touches
several at once (see `CONTRIBUTING.md`).

## Unreleased

**Normative — one checked-guard model: the compiler's inline trap.** The text placed the
guard of the checked-guard family in three incompatible places: at the operation site,
emitted by the compiler (CG-13, Concurrency §6.1); in the prelude operator body under
`comptime if verify.checked`, failing through `panic` (CT-11, Stdlib appendix §3, Types
§3.1/§4.5, Comptime §7.5, Assembly §3, CG-12); and on the instruction intrinsic itself (the
per-arch tables' guard column, against Assembly §3's "an intrinsic carries no guard"). The
operation site is now the only one: for the built-in numeric interpretations the compiler
emits the guard before the operation as the per-arch direct inline trap; a routed operator
body and an instruction intrinsic carry none. `comptime if verify.checked` (CT-11) stays,
narrowed to **library code** making its own checks respond to `unchecked`. TYP-2/TYP-5 now
say precisely what the compiler knows: the closed guard family, not the arithmetic. The
per-arch guard column describes the guard the compiler emits for that operator family.

**Normative — `trap() -> Never`.** A new prelude intrinsic emitting exactly the per-arch
checked-failure instruction — no message, hook, OS exit or unwinding, valid in every
context. A library-defined numeric type (`uint(N)`, and so `u128`) MUST fail a check that
mirrors the built-in family (overflow, division or remainder by zero) through `trap()`,
never `panic`, so `x / 0` fails identically for `u64` and `uint(256)`. `panic` stays the
failure of ordinary library logic (Stdlib §4.1; Stdlib appendix §4.1; CG-13).

**Normative — `MIN % -1` yields `0`.** In checked code it is not an overflow and does not
trap: the compiler guards the divisor `-1` case (the `MIN / -1` compare) so no target faults.
Under `unchecked` it is hardware-defined; the per-arch appendix §2 now tabulates unchecked
integer-divide behaviour for all six architectures. CG-7 no longer says `MIN ÷ -1` wraps
under `unchecked` — only `+ − *` and unary `−` wrap; division and shifts are
hardware-defined (Concurrency §6.2; CG-7/CG-8/CG-13; CT-12).

**Normative — a library container's checked access fails through `trap()`.** The text
contradicted itself: the Stdlib appendix said `Vec` indexing and the arena's `get` *trap*,
while CG-13, Concurrency §6.1 and the Stdlib appendix §3 called a container's own bounds
check an ordinary `panic`. A library check that mirrors a member of the built-in guard
family — bounds, alignment, narrowing — now fails through `trap()`, never `panic`, and is
made under `comptime if verify.checked` so it drops under `unchecked`: `Vec`'s `at` /
`index_set` / `index_range` / `set`, a library-implemented `str` / `String` byte access,
the arena's `get`, and every such guard in the prelude/stdlib catalogue. So `xs[10]` fails
identically for an array, a slice and a `Vec`, in every context including `freestanding`.
`panic` / `assert` stay for program logic — an explicit assertion, `unwrap` on `None` /
`Err` (CG-13; CT-11; OP-3; Stdlib §4.1; Stdlib appendix §3, §3.6, §4.2, §5.2.1, §6, §8;
Types §4.5; Memory §5.4; Concurrency §6.1).

**Normative — the grammar gains the slice type `[T]` / `[mut T]`.** The slice view was
used as a type throughout (Type System §7, Stdlib appendix §2.4/§3.5, the `Writer`
protocol's `[u8]`, `sort` on `[mut T]`), but `type-atom` had no production for it, so the
normative grammar rejected every such program. `slice-type ::= "[" [ "mut" ] type-expr
"]"` joins `type-atom` (Grammar §3.3; Type System §11). It does not collide with
`array-type`: after `"[" type-expr` a `"]"` ends a slice type, while `";"` or `"="`
continues an array type, and a leading `mut` occurs only in a slice type.

**Normative — arenas are owning values; a `scoped` result is used at once in checked code.**
Memory §5.1 requires temporal safety in checked code (I11), and §5.4 rested it on the arena's
linearity — but `close(in out self : Arena)` did not consume, the caller-buffer `Arena` was
declared not owning (so `b := a` gave two cursors over one buffer), and a get-result held across
growth was called a "residual hardware-defined hazard" even in checked code.

**Every `Arena` is now `@owning`** (MEM-2/MEM-3; Memory §5.9, Stdlib appendix §5.2.1) — the
`arena_over` caller-buffer arena and the `@static` fixed pool included: what it owns is the
exclusive right to allocate from and reclaim its storage. It is non-copyable and consumed exactly
once, by `close(in self : Arena)` (now consuming) or `forget`; use after `close` and a missing
`close` are linearity errors. `get` now borrows the arena as `in a : ptr(Arena)` and `mechanism`
as `in self : ptr(Arena)` (an `in` owning value would be consumed); allocation keeps `in out`. A
`@static` owning place carries no obligation and is never consumed. The OS-backed arena remains a
distinct `@owning` type holding an `Arena` over its pages.

**A `scoped` call result is transient in checked code** (Memory §5.3.1, generic, not
arena-specific): it is used within its full expression — dereferenced, field/element-accessed,
or passed down — never bound to a name, stored, or captured, and the call's `in out` /
pointer-borrowed places may not be passed `in out`, `ptr(mut …)`, or consumed elsewhere in that
expression. `deref(get(ptr(a), h)).x = 5` is well-formed, `p := get(ptr(a), h)` is not. Inside
`unchecked { … }` the result may be bound to a local, with staleness hardware-defined (CG-7;
glossary; Grammar §3.7).

**Clarified — `get` takes its arena explicitly, by address: `get(ptr(a), h)`.** There is no
implicit or ambient arena for `get` (its `ptr(Arena)` parameter is not an elidable allocator
parameter), and a free-call argument is never auto-addressed, so `get(a, h)` with `a : Arena` is a
type error; the examples now all write `get(ptr(a), h)` (MEM-3; Memory §5.3.1, §5.4; Stdlib
appendix §5.2.1; glossary).

**Normative — a call to `arena_over` requires `unchecked`.** `arena_over(buf, cap)` keeps the
address of a caller buffer inside an owning `Arena` that can be moved, returned or stored past
`buf`'s scope, and the local escape rule cannot tie the result to `buf` without a lifetime
variable — so checked code could use an arena over dead storage, against I11. A call to
`arena_over` is now a raw operation (CG-7): ill-formed in checked code and under `no_unchecked`,
written `a := unchecked arena_over(ptr(mut buf), cap)`, the author vouching that `buf` outlives the
arena; all later use stays checked. A checked form over a `@static` buffer is deferred, additive
(MEM-3; Memory §5.4, §8; Stdlib appendix §5.2.1, §8; Grammar §3.7; glossary; aspects map).

**Normative — a function value has the target's default calling convention.** Functions §6.2
said a call uses "the calling convention carried by the callee's type", but the function-value
type `fn(T…) -> R` carries no convention and `@abi` on a type is forbidden, so an indirect call
through a value taken of an `@abi(naked)`, `@abi(entry)`, `@abi(syscall)` or user-`Abi(...)`
function was undefined. **FN-10** now admits a function value — and, per **FN-11**, the `code`
of a `dyn` — only of a function with the target's default convention (ABI appendix §2: no
`@abi`, `@abi(c)`, or that convention's own prelude name); any other selector is a located
Semantic diagnostic, and on `Container.com`, whose default is `naked`, no function value can be
taken. §6.2 now reads: a direct call uses the callee's declared convention, an indirect call
the default. Low-level code reaches other conventions through the raw surface (a symbol operand
plus raw `call` / `jmp`). A convention carried in the type (`@abi(X) fn(…)`) is recorded as
deferred, additive growth; carrying it now, and silently calling with the default, are
rejected. Functions §1.5, §1.6, §6.2, §8, §9 and §10 items 1 and 7; glossary; aspects map.

**Normative — `@abi(syscall)` is rejected on macOS, as on Windows.** ABI appendix §5 listed
`x86_64-macos` and `aarch64-macos` syscall rows and allowed a direct-syscall program
"targeting a specific OS version", while §3.3 and Stdlib §4.2 said `@abi(syscall)` is
rejected on Mach-O. The rejection now holds: the macOS rows are removed, and on
`Os.macos` (container `macho`) `@abi(syscall)` is a located diagnostic, worded in parallel
with `Os.windows`. Apple's syscall numbers are not a stable public ABI, and the platform's
entry and exit already go through `libSystem` (§3.3); programs call `libSystem` through
`@extern` / `@abi(c)`. The raw trap stays writable as an instruction intrinsic under
`unchecked` — `syscall()` on `x86_64`, `svc(0x80)` on `aarch64` — so nothing expressible
in the assembler is lost (I4). §7 conformance item 4 names the rejected OSes; **FN-9** records
the decision and the rejected alternative.

**Normative — the v1 built-in catalogue is complete.** The Stdlib appendix called its
catalogue the complete v1 set while naming families only by pattern. It now enumerates
them (Stdlib appendix §4.4). **Atomics:** `load`/`store`/`swap`/`cas_strong`/`cas_weak`/
`fetch_add`/`fetch_sub`/`fetch_and`/`fetch_or`/`fetch_xor`, all on `ptr(mut T)`, `load`
included; `T` an integer, pointer or `Option(ptr(U))` type of an atomic-capable width (`fetch_*`: integers
only); natural alignment is required, guarded in checked code and hardware-defined under
`unchecked`; `fetch_add`/`fetch_sub` wrap and never trap. `fence`, `Ordering` and the
`volatile` operations get signatures. **Overflow policies:** exactly 20 functions,
`wrapping_`/`saturating_`/`checked_`/`overflowing_` × `add`/`sub`/`mul` (on `uN`/`iN`) and
`neg`/`div` (on `iN`). They are mode-opaque; on a zero divisor `checked_div` is `None`
and the other `*_div` trap (below). **Float:**
`fma(a, b, c)` on `f32`/`f64` is rounded once; with no fused instruction it is a
correctly-rounded software routine. The saturating float→int conversion is named
`saturating_convert(T, f)`: NaN → 0, ±∞ clamped. `dyn_over` joins the catalogue partition.
The conformance list no longer names a nonexistent `mem` module (CG-8; CG-9; CC-2; STD-1;
Concurrency §2, §4, §6.3, §7.3, §7.7, §8; Types §4.2, §10; Stdlib appendix §4.1, §4.3,
§4.4, §8; glossary).

**Normative — atomics admit `Option(ptr(T))`, `checked_div` reports a zero divisor as
`None`, and `NaN` / `Infinity` join the catalogue.** **Atomics:** `load`/`store`/`swap`/
`cas_strong`/`cas_weak` accept `Option(ptr(U))` / `Option(ptr(mut U))` as `T` — it has the
pointer's representation (the all-zero niche, Memory §4.2), and without it lock-free code
needs an untyped `ptr == 0` test; `fetch_*` stays integer-only. **`checked_div`:**
`checked_div(a, 0)` is `None`, like `checked_div(MIN, -1)` — the `checked_` family never
traps; `wrapping_div`/`saturating_div`/`overflowing_div` still trap on a zero divisor.
**`NaN` / `Infinity`:** the Grammar §2.4 prelude constants are typed as an untyped float
literal (a float context, else `f64`); `NaN` is the canonical quiet NaN (`f32`
`0x7FC00000`, `f64` `0x7FF8000000000000`), `Infinity` is +∞ (`0x7F800000`,
`0x7FF0000000000000`), `-Infinity` is unary minus (CC-2; CG-8; CG-9; TYP-13; Concurrency
§2, §6.3, §8; Types §9.1; Grammar §2.4; Stdlib appendix §4.4, §8; glossary; aspects map).

**Normative — the CLI gains `help`, `version`, and a verbosity contract.** Tooling §4 specified
the command set, discovery, target and profile selection, and artifact placement, but said
nothing about how a toolchain reports on *itself*. Two conforming implementations could
therefore disagree on whether `alatyr --version` works at all, and nothing stopped a
`--verbose` flag from changing the artifact it was supposed to be merely describing.

**TOOL-21** makes `help` and `version` introspection commands: available as both a command and
a flag, forbidden to read a manifest, discover a package, resolve dependencies or produce an
artifact, hence valid from any directory; written to stdout with a successful exit; `version`
must report the **revision of this specification** the implementation conforms to, since an
implementation's own version and the revision are separate axes (Overview §6). A bare invocation with no
arguments reports the same material on **stderr** with a **failure** exit — help asked for is an
answer, the same text volunteered after a mistake is a diagnostic — so a script that lost its
arguments still fails. What either prints is quality-of-implementation.

**TOOL-22** makes output verbosity an observation rather than a build parameter: such a flag
must not change the artifact, the diagnostics, the accept/reject outcome or the exit status,
and the same input must build byte-identically with and without it (§6.2). This is stricter
than the debug-only budget/limit overrides, whose effect a package's success merely must not
depend on.

**TOOL-14** is clarified: an unrecognised command or flag is an invocation-level Config
diagnostic naming the offending argument, and an implementation must not silently reinterpret
an argument it does not recognise as a source path — which reports a fault in a file the user
never named and hides a mistyped flag behind a file-system error.

§7 conformance item 4 now carries all three.

**Normative — conformance cites a revision, not the language version.** Overview §6 now
states that `v1` (the language) and `1.0.0`, `1.1.0`, … (revisions of this document) are
separate axes: program validity holds across the whole major (I10), conformance does not,
because a MINOR revision may pin behavior an earlier one left unstated. "Conforms to `v1`"
is therefore not a well-formed claim.

`v1` was load-bearing throughout the text — including inside the conformance rules — and
defined nowhere. It now has a glossary entry, alongside **revision**, and a decision entry
(FND-13).

## 1.0.0

First public release of the complete first draft. The whole document is **under review**
and not yet accepted.

**Specification (`spec/`)** — all 14 narrative chapters, `00-overview` through
`130-grammar`, plus 4 appendices: `140-appendix-manifest` (manifest schema),
`150-appendix-abi` (ABI per target), `160-appendix-stdlib` (prelude and standard
library), `170-appendix-arch` (per-architecture tables for all six v1 architectures and
the closed supported-target set).

**Design record (`design/`)** — invariants `I1–I11`; the decisions organized by theme
under stable theme-prefixed IDs (`MEM-`, `TYP-`, `FN-`, …), indexed by
`design/decisions/README.md`; the glossary; the aspect map. Every open question from the
design phase is closed.

**Implementation** — none of this text has been validated against a toolchain. The
reference compiler, which carries the standard library with it, is in development in the
sibling `compiler` repository; its feedback will arrive as refinements here.
