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

**Normative — a deferred action with a value is ill-formed (Memory §5.8; Grammar §3.7; Control Flow
§9.3; SYN-10).**  Nothing said what happened to the value of `defer expr`, or of a deferred block ending
in an expression. Such a `defer` is now ill-formed, as a procedure body ending in a value is: write
`defer { _ := print("bye\n") }` (SYN-9). Rejected: evaluating the action for effect and dropping its
value silently. (spec#113.)

**Normative — `free` / `clear` / `truncate` over a `Vec` of an owning `T` require the elements gone,
else trap (Stdlib appendix §6, §8; STD-11).**  `free` returned the buffer and said nothing of owning
elements still held. For an owning `T`, `free` and `clear` now require `len() == 0` and `truncate(n)`
requires `len() <= n`; otherwise each fails through `trap()` in every verification mode. The caller
drains the vector with `pop`, consuming each element. For a non-owning `T` the three are unchanged.
Rejected: a `free_with` taking a per-element release; refusing the three and `forget`-ing the drained
vector, which leaks the buffer. (spec#107.)

**Normative — `for x in e` applies `iter` to `e` as a receiver, with auto-ref (Control Flow §6;
Type System §4.5; Stdlib appendix §2.4; CF-11).**  No passage said how `for` applies `iter` to its
operand, and once `Vec(T)` became owning that decided whether a loop over a `Vec` compiles. The
operand is now `iter`'s UFCS receiver (`e.iter()`): `for x in v` borrows `v` and does not consume
it, and `for x in make_vec()` is ill-formed by Memory §1.6 (bind it first). Rejected: passing the
operand by value (a move, or `for x in ptr(v)`); refusing `for` over an owning operand and
iterating its view. (spec#115.)

**Normative — the comparison relations borrow an owning operand (Stdlib appendix §2.6, §6, §8;
Memory §4.5, §5.9; Functions §1.4; Comptime §5.4; MEM-13).**  `eq` / `lt` / `hash` take their operands
`in`, which moves an owning value, so `s == t` over two `String`s consumed both. For an owning `T`,
`==` `!=` `<` `<=` `>` `>=` and `hash` now pass each operand by its address, as a receiver's auto-ref,
and consume neither; an owning value-expression operand is ill-formed (bind it first). The one generic
structural default is instantiated at `T` and reads through the addresses, and the `ptr(T) × ptr(T)`
pair this produces is not a pointer comparison. Open: how a concrete relation is declared over an
owning type (spec#126); the enum arm over a borrowed operand, `Option(String) == Option(String)`
(spec#128). Rejected: refusing the comparison and comparing views; consuming the operands and
`forget`-ing them. (spec#106.)

**Normative — more than one `@convert` accepting the operand is an ambiguity error, whatever the
sources (Type System §4.6; TYP-19).**  A `T(v)` site where more than one in-scope `@convert` to `T`
accepts `typeof(v)` is ill-formed, and never falls through to the builtin lattice step; before, only
two conversions with the same (source, target) were named. An untyped literal is accepted by every
candidate whose parameter can represent it (Declarations §3.4), so with conversions to `B` from `u32`
and from `u64`, `B(42)` is ambiguous; write `B(u64(42))` or an annotated local. Rejected: falling
through to the brand crossing, which runs neither guard; giving the literal its default type first.
(spec#122.)

**Normative — `_ := expr` is the explicit discard; the lone `_` is not a name (Grammar §2.2, §3.1;
Functions §3.3; Control Flow §3; Stdlib appendix §2.7; SYN-9).**  Since STD-7 made `print(…)` return a
`Result`, a function with no result ending in `print(…)` was ill-formed with no way to drop the value
but an unused named binding. `_ := expr` now evaluates `expr` for effect and binds nothing; as a block's
last element it leaves no value, so `main := fn() { _ := print("hello\n") }` is well-formed. The lone
`_` is no longer an identifier: it cannot be bound or read and may repeat in one scope; it remains the
wildcard pattern. A final `if`/`else` or `match` whose non-diverging arms all yield a value takes
the discard on it or in each arm. An owning value cannot be discarded. `for _ in`, `(q, _) :=`, a `_` parameter and a
module-scope discard are refused (spec#114); `defer print(…)` is ill-formed (SYN-10); open:
a final `if`/`match` with partly value-less arms (spec#119).
Rejected: an unused named binding as the only form; discarding a body's final expression implicitly.
(spec#81.)

**Normative — `String` and `Vec(T)` are `@owning`, released by `free`; a call on an owning
value-expression at a read-only receiver slot is ill-formed (Memory §1.6, §5.9, §8; Type System
§4.5; Functions §1.4; Modules §2; Comptime §5.4; Stdlib appendix §1, §2.4, §2.6, §6, §8; MEM-12).**  Both containers
were copyable under Memory §5.9, which duplicated the buffer pointer, and neither could be
released. Each is now `@owning` (`Vec(T)` for every `T`) and gains `free(in self)`; its other
operations borrow the receiver. MEM-11's temporary is for non-owning values only:
`make_vec().len()` is a located error, and the explicit form is
`v := make_vec(); n := v.len(); v.free()`. `Vec(String)` and `Vec(Vec(U))` now have an owning
element, so STD-10's refusals apply to them.
Rejected: releasing or `forget`-ing the temporary at the end of the full expression; (derived) `Vec(T)`
owning only for an owning `T`. (spec#73.)

**Normative — `==` over `HashMap(K, V)` compares entries; `<` and `hash` over it are ill-formed
(Stdlib appendix §2.6, §6, §8; Comptime §5.4; STD-9).**  `HashMap`, the third owning container,
took the plain field walk, so `==`, `<` and `hash` compared its table address. `a == b` now holds
when the lengths are equal and every key of `a` maps in `b` to a value equal by `V`'s `eq`
(O(n) expected lookups); `<` / `<=` / `>` / `>=` and `hash` over a `HashMap`, directly or through
an aggregate's walk, are a Semantic diagnostic at the site, so a map cannot be a `HashMap` key.
Open: a concrete user `lt` / `hash` over a `HashMap` type itself (spec#102); the order and early
exit of `V`'s `eq` calls (spec#103). Rejected: an order-independent `hash`; refusing `==` too.
(spec#50.)

**Normative — the owning-container comparison arm lives in the compiler-provided prelude, built
per translation unit (Stdlib appendix §2.6; Comptime §5.5; STD-8).**  STD-6's content arm for
`Vec(T)` / `String` sits in the base-tier generic `eq` / `lt` / `hash`, whose body must name
alloc-tier types that do not exist under `no_alloc`. The arm is part of the compiler-provided
prelude, which is built per translation unit with that unit's limits: a unit with the alloc
tier gets a default body with the arm, one without gets a body without it. No tier-presence
predicate is added. Open: a unit without the alloc tier using a type whose layout reaches an
alloc-tier type (spec#101). Rejected: a comptime tier predicate; an extension point in the base
walk. (spec#55.)

**Normative — `format` returns `Result(usize, E)` and stops at the first failed write; the v1
hole set is closed (Stdlib appendix §2.7; Functions §7.1; STD-7).**  `format(w, t, args…)` had
no stated result or failure behaviour, and `print`, being `format` to stdout, inherited the
gap. It now returns `Result(usize, E)` (`E` the writer's own error; `IoError` for `print`):
each fragment is written completely, `write` called again on the unconsumed remainder after a
short write (a writer that keeps returning `Ok(0)` keeps being called); the first failed write
is returned unchanged and nothing is written after it; on success, the total byte count. A hole
renders an integer (base-10), a `bool`, a `char` (UTF-8), a `str` (its bytes) or a float (the
shortest decimal that reads back to the same value); any other type is a comptime error until
`Display` is added. Because a procedure body may not end in a value-expression (Functions
§3.3), `print(…)` as the last element of a function with no result is now ill-formed there;
discard the result (`_ := print(…)`, SYN-9), bind it or handle it. Open: the exact text of a float (spec#75);
the module paths (spec#74); a writer
over-reporting its count, and a total past `usize` (spec#82); a user brand or `@require` type
in a hole (spec#99); which `write` calls carry the fragments (spec#100). Rejected: a count-less
result; a `print` that traps or ignores a stdout failure; `Display` in v1; a fixed-fraction float. (spec#46.)

**Normative — a value-expression gets a temporary in a read-only receiver slot; a store into a
field or index of one is ill-formed (Memory §1.6; Type System §4.5; MEM-11).**  Memory §1.6 made
any field access a place and Type System §4.5's auto-ref did not say what a read-only `ptr(R)`
slot does with a value, so `f().x = 1` and `make_pair().sum()` / `vv[i].len()` had no stated
meaning. A field, tuple component or array index of a value-expression is now a value-expression,
so `f().x = 1` is ill-formed; a value-expression in a read-only `ptr(R)` receiver slot is
materialized in a caller-owned temporary living to the end of the full expression, and its
address is passed. A slice or `str` index stays a place whatever its base. Whether an owning temporary's
end discharges its linearity obligation is open as spec#73. Rejected: allowing the store into a
temporary's field (a discarded write); no temporaries at all. (spec#47.)

**Normative — under `no_abstractions` a constant expression is admitted (Assembly §1.1, §5;
CG-17).**  CG-12 admitted comptime in full but forbade the value-operators as surface syntax, so a
constant expression such as `disp = -16` (comptime by its form since CT-13) had two answers in a
`no_abstractions` unit. It is admitted: it emits no instruction and carries no guard, and the
forbidden list governs run-time operator sites only. Rejected: forbidden regardless of phase (a
`comptime` binding would be required); only unary `-` on a literal admitted. (spec#59.)

**Normative — an integer constant expression is evaluated exactly and its result checked; a
floating-point one per operation (Comptime §1.1, §2.6, §11; Type System §2.3, §9.1; Declarations
§3.4; Memory §4.5; Concurrency §6.5, §8; CC-8; CT-14).**  CT-13 checked a constant expression "like a literal" without saying
whether each operator runs at the context type or the exact result is checked, and the two
disagree on `-1 : u8`, `-128 : i8` and `1 - 2 + 3 : u64`. An integer constant expression is now
computed over exact integers and only its result is range-checked: `-1 : u8` and `~0 : u8` are
rejected, `-128 : i8` is accepted, `1 - 2 + 3 : u64` is 2, `300 - 100 : u8` is 200; an unsigned
context does not select the unsigned operations, and the mask `~7` at
`usize` is rejected (write `~usize(7)`). `unchecked` does not change a constant expression. A
floating-point constant expression — one containing a floating-point literal — is evaluated per
operation in its context's IEEE format (`f64` with no context), so it means the value run time
would give (overflow is ±∞). Open: an integer-literal expression more than a single literal
(unary `-` included) at a floating-point context (spec#87); the operands of a constant comparison (spec#88); `/`, `%` with a negative operand
and the bitwise operators over negative values (spec#91).
Memory §4.5's `p == -1` is now ill-formed; the all-ones sentinel is written `p == ~usize(0)`.
Rejected: per-operator evaluation at the context type; exact evaluation with a negative result
accepted at an unsigned type as its two's complement; floating-point evaluated exactly with one
rounding and range-checked. (spec#62.)

**Normative — `Vec(T)` admits an owning `T`; its element is exchanged or lent by a scoped
reference, never copied out or overwritten unseen (Stdlib appendix §6, §8; Types §4.5; Memory §1.6,
§5.9; STD-10).**  Nothing said whether `Vec(T)`'s `T` may be `@owning`, nor what the by-value `at`/`v[i]` or
the overwriting `index_set`/`v[i] = x` do with a linear element. `Vec(Fd)` for `Fd := @owning struct { fd : i32 }` is now well-formed;
for an owning `T`, `at`/`v[i]` (and a read through it), `index_set`/`v[i] = x`/`set`, `v[i] ⊕= x`
(OP-7) and OP-8's explicit form are ill-formed, with a diagnostic naming the explicit operations.
`Vec(T)` gains, for every `T`, `replace(i, x)` → `T`, `swap(i, j)`, and the `scoped` element
references `at_ptr(i)` → `ptr(T)` / `at_mut(i)` → `ptr(mut T)` (transient, the vector frozen, as
the arena's `get`), each bounds-checked like `at`; for a copyable `T` they are added and nothing else changes. `pop`
stays. Open: whether such a `Vec` is itself owning and how it is released, `push`'s failure path,
`get`/`iter`/`clear`/`truncate` and `==`/`<`/`hash` (spec#96); a whole-value move out of or store
over an owning place through a pointer or a sub-slice (spec#97).

**Normative — a `@reg(name)` binding is a register operand; a register the source names is
reserved from the function's placement (Assembly §4, §5, §6, §12; Memory §2.5, §2.6; Functions
§6.4; Codegen §3.3, §3.4; CG-19).**  A naked body may only use `@reg(name)`-pinned locals, but
nothing said such a binding is a register operand, so neither its instructions nor
`asm("… {0} …", x)` had an answer; nor did anything stop the placement from reusing a register the
programmer wrote. A `@reg(name)` binding now **is** that register as an operand (naked body,
instruction intrinsic, `asm(…)`); a bare-`@reg` binding is not — an instruction intrinsic
materializes it, `asm(…)` rejects it. A register named by an instruction intrinsic, an `asm(…)`
operand or `@reg(name)` is reserved: the placement (S2, or the fixed `no_abstractions` rule) gives it
to nothing else in that function, nor any register sharing storage with it (`al` → `rax`, `ymm3` →
`xmm3`). Open: a reserved register across a call, names in a comptime-removed branch, and names in
an `@inline` body (spec#93). Rejected: both kinds of binding as
register operands; neither; reservation left to the programmer; per statement span. (spec#52.)

**Normative — an immediate slot and an `at(…)` field take the native signed integer as context;
the arch table alone decides encodability (Assembly §3, §5, §7, §10, §11, §12; Comptime §1.1; arch
appendix §2(f), §3.6–§8.6, §9; CG-18).**  Nothing named the type of an immediate slot or of `disp`,
so since CT-13 nothing fixed whether `movb(al, 200 + 100)` or `disp = -0x80000001` is well-formed.
An immediate now takes the target's native signed integer as its context type (a constant
expression is exact, then checked, CT-14), and each slot has an **encodable set** in the arch
table — a signed range, an unsigned range, a bitmask pattern or a rotated immediate — which alone
decides whether the value is encodable. Every slot of all six architectures has its set
(`movb` −128 … 127, a vector number of `int` 0 … 255, RISC-V `imm12` −2048 … 2047, aarch64 logical
immediates as a bitmask pattern, aarch32 modified immediates as a rotated one). A value above the
native signed maximum is written as its two's-complement negative (`movq(rax, -1)`). Superseded:
a signed per-width slot type per slot (this cycle's first answer, spec#63), which no v1 type could
give sub-native, zero-extended or pattern-encoded fields. Rejected: the operand width's signed
type plus an encodable-set column; x86 types with the native integer elsewhere; sub-native types
for immediates. (spec#63, spec#80.)

**Normative — a `str` index is the built-in slice read plus the code-point-boundary check, a
read-only place (Stdlib appendix §3, §3.6; Memory §1.6; Type System §4.5; Stdlib §4.1; OP-9).**
The `str` entry admitted byte access as either the built-in slice read or library code, and
since OP-8 that choice decided whether `s[i]` is a place, so whether `ptr(s[0])` is
well-formed. `s[i]` is now the built-in slice read over the bytes plus the code-point-boundary
check: a read-only place like a `[u8]` element, so `ptr(s[0])` is well-formed (a `ptr(u8)`) and
`s[i] = b` is ill-formed. The boundary check is part of the compiler's one bounds guard at the
site, which `unchecked` drops. The library-check examples no longer list `str`/`String` byte
access. Rejected: `str` as a library aggregate, `s[i]` ≡ `at(s, i)` a value-expression. (spec#53.)

**Normative — a named-component variant's payload is the anonymous struct of its fields;
patterns bind it by name, a subset (Types §6.2; Comptime §5.1, §5.5; Control Flow §5.2;
Grammar §3.5; Stdlib appendix §4.1; TYP-18).**  `Variant.payload` of a variant whose
components are declared by name is now `Some(struct { … })` — names, types, defaults and order
as declared — and `T.(v)(p)` binds that struct; positional payloads and every layout are
unchanged. A pattern over such a variant, and a struct destructuring, binds **by name only**
and may list a subset, unlisted components ignored; a positional argument there, an unknown
name or a repeated one is a Semantic diagnostic. `E.N()` is ill-formed as a pattern, as it is
as a constructor. Open: the empty list and positional-pattern arity (spec#68); a reflected
field default (spec#69); the identity of an anonymous struct (spec#79). Rejected: the tuple payload; listing every component; positional
binding of named components. (spec#44.)

**Normative — a trailing comment on a line `fmt` expands trails the first output line (Tooling
§4.3.4; TOOL-26).**  A trailing comment suppresses re-wrapping of its line, but an always
multi-line construct (a `match`, a `comptime for` template, a value-`if`, a non-empty block, a
list or chain holding one) has no one-line form to keep, and nothing said which of the output
lines the comment then trails. It trails the first: `y := match x { A => 1; B => 2 }  # why`
becomes `y := match x { # why` followed by the arms; a line holding only some arms keeps the
comment on its first arm's line. Rejected: the last output line; an own line above the
statement. Settled under the standing delegation for formatter layout. (spec#57.)

**Normative — `fmt` wraps a list holding an always multi-line element one element per line
(Tooling §4.3.3; TOOL-25).**  §4.3.3's outermost-construct rule assumed every construct has a
one-line form; a `match`, a `comptime for` template, a value-`if` and a non-empty block (a `fn`
value's body included) have none, so `f(a, match x { … })` had no single layout. A construct
whose canonical layout breaks whatever the width is now *always multi-line*, never "fits"; a
call argument list, array/list literal, constructor field list or parameter list holding one is
wrapped one element per line with the trailing comma, and a binary-operator chain holding one
breaks before each operator; the rule climbs to the enclosing list or chain. Rejected: the list
staying on its opening line with `)` after the element's `}`. Settled under the standing
delegation for formatter layout. (spec#58.)

**Normative — `fmt` lays out every value-`if` wrapped (Tooling §4.3.3, §4.3.4; TOOL-24).**
§4.3.2 makes `if` a block header whose `}` stands on its own line, while §4.3.3 kept a fitting
value-`if` (`x := if c { a } else { b }`) on one line, so a short value-`if` had two canonical
answers. A value-`if` is now always laid out with the statement-`if` shape, like `match`
(TOOL-23); a one-line source value-`if` is reformatted that way, and a trailing comment does not
keep it on one line. The grammar is unchanged. Rejected: one line when it fits. Settled under the
standing delegation for formatter layout. (spec#60.)

**Normative — a constant expression is comptime by its form (Comptime §1.1, §1.6, §2.6, §11;
Type System §2.3, §9.1; Declarations §3.4; Memory §4.5; Assembly §1.1, §5, §7;
CT-13).**  Evaluation is runtime by default with no phase inference, and only a *literal* was a
compile-time number, so `-16` (unary `-` over `16`; there is no negative literal) and `0 - 1`
read as runtime values, against the specification's own `disp = -16` immediates. A constant
expression — scalar literals combined by the built-in scalar operators, unary `-` included — is
now comptime by its form, a rule about the text and not inference. It is checked against its
context type like a literal, a failure being a compile-time diagnostic (`x : u8 = 200 + 100` is
rejected; how the check reads, `unchecked` included: CT-14, above). `disp = -16` and `0 - 1` are
immediates; CG-15 and CG-16 no longer leave the phase of `0 - 1` open. How the check reads where
per-operator and mathematical evaluation differ (`-1` at an unsigned type, `-128 : i8`,
`unchecked`), and how a floating-point constant expression is evaluated, are open as spec#62
(since settled by CT-14, above); the context type of an immediate slot as spec#63 (since settled by
CG-18, above). Whether a constant expression is admitted
under `no_abstractions` is open as spec#59 (since settled by CG-17, above). Rejected: only a literal and unary `-` on a literal
comptime; every non-literal runtime, with negative immediates spelled through a `comptime`
binding. (spec#39.)

**Normative — `fmt` lays out every `match` wrapped, one arm per line (Tooling §4.3.3, §4.3.4;
TOOL-23).**  §4.3.3 put any construct that fits in 100 columns on one line and wrapped a
`match` arm list only when it did not fit, while §4.3.2 puts a block's `}` on its own line;
neither said which governs a short `match`, nor how a one-line arm list would be spelled. A
`match` arm list is now always wrapped, like the `comptime for` arm template: each arm on its
own line, the `}` at the `match`'s indent; a one-line source `match` with `;` between arms is
reformatted that way, with no `;` left between arms, and a trailing comment does not keep two
arms on one line (§4.3.4; the comment then trails the first output line, TOOL-26). The
grammar is unchanged. Rejected: a fitting `match` on one line with arms joined by `; `.
(spec#35.)

**Normative — `Vec(T)` and `String` compare by content, not by buffer identity (Stdlib
appendix §2.6, §6, §8; Comptime §5.4; STD-6).**  STD-4 made comparison over `[T]` and `str`
compare content, but the owning containers took the plain field walk, which reaches their data
pointer and compared buffer addresses. `==`/`<`/`hash` over a `Vec(T)` are now §3.5's rule over
its `len` elements (requiring `T`'s `eq`/`lt`/`hash`), and over a `String` §3.6's rule over its
bytes; capacity, allocator and address take no part, the cost is O(n), and identity is an
explicit comparison of the data pointers. `HashMap` is decided by STD-9 (spec#50).

**Normative — a floating-point literal with no context takes `f64` in Declarations §3.4 too
(Declarations §3.4; TYP-13).**  §3.4 gave "an integer/float literal" with no context "the
documented default (the target's native signed integer)", which read literally gives
`x := 1.5` an integer type; Types §9.1, TYP-13 and the Stdlib appendix give it `f64`. §3.4's
parenthetical now names both defaults — an integer literal the target's native signed
integer, a floating-point literal `f64` — so the chapters give one answer; §3.2 says the same
for a bare float binding (`x := 1.5` is `f64`), and Types §9.1's integer paragraph now says
"an integer literal" where it said "a literal". (spec#33.)

**Normative — a write into a library aggregate's element through its index is ill-formed
(Types §4.5; Memory §1.6; OP-8).**  Types §4.5 gave a library aggregate such as `Vec(T)` a
by-value read `a[i]` and a write `a[i] = v`, but said nothing of `a[i].f = x` when the element
is a struct. It is now refused with a located diagnostic naming the explicit form
`mut e := a[i]; e.f = x; a[i] = e`, as are `a[i].f ⊕= x` and any longer path into the element's
own storage; a path that leaves it through a dereference (a pointer- or slice-typed element
or component) writes the pointee and is unaffected. A mutating method on the element such as
`a[i].push(x)`, and `vv[i][k] = x` on a `Vec(Vec(T))`, are refused by §4.5's existing receiver
rule, since the element is not a mutable place. `a[i] = v` and `a[i] ⊕= rhs` (OP-7) stay.
Types §6.4 now names the library functions `at`/`index_set`/`index_range` (it said `index`).
Rejected: read-modify-write sugar; a place-returning index operation (deferred). (spec#23.)

**Normative — a compound assignment on a library aggregate's index is `index_set(a, j,
at(a, j) ⊕ rhs)` (Memory §1.6; Types §4.5; Grammar §3.2; Stdlib appendix §6; OP-7).**  Memory
§1.6 lowered `place ⊕= rhs` by taking the place's address once, but on a library aggregate such
as `Vec(T)` an index is the by-value call `at(a, i)` and has no address, so `v[i] += 1` had no
specified meaning; §1.6 also listed every index `a[i]` as a place-expression, against Types
§4.5. On a library aggregate `a[i] ⊕= rhs` now lowers to `index_set(a, j, at(a, j) ⊕ rhs)`,
with `a` and `i` each evaluated once (`j` holds `i`), in the order `a`, `i`, the read, `rhs`;
`a` meets `index_set`'s receiver slot as for `a[i] = v`. §1.6's place list now says an index
on an array or slice, and an index on a library aggregate is a value-expression. Arrays and
slices are unchanged. Rejected: refusing the form; a place-returning index operation
(deferred). (spec#34.)

**Normative — the `{}` template is v1, its trigger is the first fixed `str` parameter, and
`print` is `format` to standard output (Functions §7.1; Stdlib appendix §2.7, §7; STD-5).**
The template was called the normative trigger in Functions §7.1 but an additive convenience in
Stdlib appendix §2.7, STD-1 and the aspect map; it is v1. §7.1's trigger, "the first fixed
parameter is `str`", missed `format(w, "…", args…)`, whose first parameter is the writer; it is
now the first fixed parameter **of type** `str`, wherever it stands. `print`, named but never
defined, is `format(stdout, …)` with the same template rules and result, a std-tier facility
absent under `freestanding`. `write_all`, `Display` and `format->String` stay additive. A
comptime-variadic whose first `str` fixed parameter is not its first fixed parameter — the shape
the former §7.1 advice recommended for an untemplated `str` — now templates a literal there.

**Normative — variant components may take the struct field form, and a nullary variant has
one spelling (Types §6.2, §9.3, §9.4; Grammar §3.3, §3.4; manifest appendix §3.1, Tooling §2.2,
ABI appendix §2, per-arch appendix §1; TYP-17).**  The manifest appendix's `Machine` enum
declared named, defaulted variant components (`Linux(arch : Arch, env : Env = …)`) and built
them by name (`Machine.Bare(arch = Arch.aarch64)`), which the `variant` production could not
express. A variant's components may now be declared in the struct field form
`ident : type-expr [= expr]` — all of one variant's components positional or all named — and
such a variant is constructed by named arguments exactly as a struct is, a defaulted component
omittable; positional components stay valid. A variant without components is written `N` and
constructed `E.N`; `E.N()` is a Semantic diagnostic, as `m()` is for a raw-union member. The
manifest's `Com ( )` / `Machine.Com()` are now `Com` / `Machine.Com` everywhere. The payload
type and pattern form of a named-component variant are open as spec#44.

**Normative — `asm(…)` emits exactly its template: an operand not of a §5 kind is a compile
error, never materialized (Assembly §4, §5, §11, §12; CG-16).**  §4 restricts an
`asm(template, …)` operand to an operand kind of §5, but did not say whether a runtime
operand — a plain local `x`, or a non-comptime expression — is placed in a register as an
instruction intrinsic's operand is (§2.2, CG-15). It is not: each operand must already be of
a §5 kind, as in a naked body, and one that is not is a compile error in any function frame;
the register materialization is for instruction intrinsics only. The explicit form is to
move the value into a register with an instruction intrinsic first. Materializing it was
rejected because the escape would then emit instructions, and pick a register, that its
template does not show (I1). Whether a `@reg` / `@reg(name)` binding is a register operand is
open as spec#52 (since settled by CG-19, above).

**Normative — a backend that cannot lower an expression instruction-intrinsic operand
refuses, and that refusal is its limit, not the language's (Assembly §5; CG-15).**  §5 says operands are
ordinary expressions and there is no special operand grammar, yet five compiler fixtures
refused expression operands such as `movq(rbx, 0 - 1)` on the ground that ch. 80 fixes an
operand to a register name or an immediate. In an ABI-framed function the form is
well-formed (where its operator is admitted): a comptime operand is an immediate, and any other
is materialized in a register exactly as an operand local is (§2.2); operand-kind validity
stays checked against the instruction's table.
An implementation that cannot yet lower the operand refuses with a located diagnostic rather
than emit a wrong value, and publishes that refusal as a known deviation (Overview §6) — never
as a rejection this chapter mandates.

**Normative — a non-literal first argument to a comptime-variadic is a valid call, rendered
as one fragment (Functions §7.1; Stdlib appendix §2.7; FN-13).**  §7.1 said a non-literal first argument "is not a
template (outside v1)" without saying whether the call was callable at all, and the compiler
refused it. With no template there are no holes to interleave, so the leading value renders as
one fragment through the same `Display`/scalar layer and the rest follow in order; what stays
forbidden is the template reading — no hole-splitting, no brace unescaping, no hole-count
check. Refusing was rejected because it would make an ordinary program — printing a computed
string — ill-formed when the call has one obvious meaning; the call is never silently dropped (I3).

**Normative — comparison over `[T]` and `str` is by content, not by the `{ptr, len}`
derive (Stdlib appendix §3.5, §3.6, §2.6; STD-4).**  §2.6 derives `Eq`/`Ord` structurally
over `typeinfo(T)`, and a slice is the library pair `{ ptr, len }`: applied to a view that
walk compares an address and a length, so two slices of equal content at different addresses
were unequal and `==` asked where the bytes live. §3.5 never listed comparison and §3.6
defined no equality for `str` anywhere in the specification. `[T]` now compares elementwise
through `T`'s own `eq`/`lt`, with `<` lexicographic, and `str` compares its bytes by the same
rule at `T = u8`; `hash` follows content. §2.6 gains a view arm so the rule holds wherever
the walk meets a view, including inside an aggregate. The O(n) cost is stated because the
operator hides it; identity stays an explicit `.ptr` comparison, which needs no grant. This
changes the result 1.8.0's derive gave `==` over a view: code that relied on pair identity must
now compare `.ptr` explicitly.

**Normative — an enum variant constructor takes exactly its declared component count (Types
§9.3; Grammar §3.4; TYP-15).**  The arity rule was stated once, for **a raw-union
member** constructor; the enum bullet in §9.3 named the constructor form with no arity rule,
and §9.4's no-implicit-default rule names no constructor arity. `Pay.B(7, 8, 9)` for
`B(u64, u64)` therefore had no answer in the text, and a compiler that dropped the surplus
argument was not contradicting anything. An enum variant constructor now takes no argument
beyond its declared components — a surplus argument is a Semantic diagnostic and is never
silently discarded (I3, FND-3) — and positionally declared components exactly, in order.

**Normative — a bare literal binding takes the documented default (Declarations §3.2;
Memory §5.3.1; TYP-16).** Declarations §3.2 called `x := 5` ill-formed ("a bare integer
literal with no context … MUST be annotated"), while Declarations §3.4 and Types §9.1 give a
literal with no context the documented default, the target's native signed integer. The
default stands and §3.4 and §9.1 are unchanged; §3.2's example now reads "a literal that
needs a non-default type", and §3.2 states the consequence: `mut i := 0` is the native signed
integer, a later use never re-types it, and passing it where `usize` is expected is a
signed→unsigned numeric conversion, refused unless written `usize(i)` (Types §4.2, §4.3;
I8). Memory §5.3.1's escape example annotates `x : u64` so it shows only the escape error.
Rejected: annotating every bare literal binding; inferring the type from the first use.
(spec#25.)

**Normative — `match` arms take no comma (Grammar §3.7; Control Flow §5.3; Tooling §4.3.3;
SYN-8).** Grammar §2.6 separates `match` arms by a newline or `;`, while Tooling's canonical
layout ended every expression arm with a trailing comma, so the formatter's own output did
not parse. The grammar stands: the canonical layout puts no comma after any arm, and a comma
after an arm, expression, block (`E::B => { return 4 },`) or `comptime for` template,
including after the last arm, is a Parse diagnostic located at that comma. TOOL-2's
arm-wrapping note is corrected to match. Rejected: making `,` an arm separator. (spec#24.)

## 1.8.0 — 2026-11-03

**Normative — a `@convert` is the operation a site denotes, checked or `unchecked` alike
(Types §4.2, §4.6, §5.4, §7, §12; TYP-14 as amended).** 1.7.0 made an in-scope `@convert`
precede the brand crossing, but wrote the exception as *dispatch takes the builtin crossing
instead* inside an `unchecked` scope. For a source type outside the numeric lattice that
leaves the site with **no** operation: `to_u64 : bool -> u64` is §4.6's own example, step (3)
cannot match `bool`→`u64`, so `unchecked u64(b)` compiled checked and refused unchecked — an
observable defect, not a wording slip. The exception is now what Stdlib appendix §3.2 already
said and what the decision's own `(CT-11)` named: the conversion is selected at every site,
and its guard — library code under `comptime if verify.checked` — is comptime-absent inside
`unchecked`, where the body is an identity crossing that costs nothing. `unchecked` drops a
guard inside the operation a site denotes and never substitutes another one: each guarded
operation takes its hardware-defined behaviour, the same operation and never another (CG-7),
and the guard itself is absent by CT-11 (I1, I11). Added: dispatch terminates — step (2) never re-selects the conversion whose
body is being compiled, and inside that body `T(v)` over the directly-branded type
resolves at step (3) to the brand crossing — still `T(v)`, since `bitcast` is
reinterpret's only form (§4.4). §4.2's intro and brand row, §5.4's brand bullet and §12 item 5 now state that the
table fixes cost and whether a crossing may be implicit, not denotation; §7 calls `char`
what it is, a brand over `u32`, not an alias. (spec#22; compiler#364, #1244.)

**Normative — an alias is transparent to visibility: the resolved entity is what is checked
(Modules §3, §4.1, §11; MOD-15).** A comptime alias binds a *name* to an **entity**; it
creates no new entity, carries no visibility different from what it names, and launders
nothing. Creating an alias requires the site to be allowed to name the entity — which is why
§4.1's own `vec := math::vec` … `vec::length(p)` stays legal, `math` being `pub` and
reachable — and every later use through the alias is judged against **that entity's**
visibility, never against the shape of the head; a member's own visibility still governs it.
Measured on native x86_64 at `compiler@c00eae90` before this change: with `hid` a private
sibling module, the spelled `hid::secret()` was refused while `h := hid` then `h::secret()`
was accepted and **ran** (exit 42) — the same leak for a private type, a private global, an
aliased nested module and an alias declared in a third module. Enforcing against the spelled
path is a hole, not a loophole: privacy that one line of source defeats is not privacy.
`pub use other::private_item` therefore does not re-export: the alias is refused at its
declaration (spec#27). (spec#27; compiler#1429, #1427.)

## 1.7.0 — 2026-11-03

**Normative — an in-scope `@convert` precedes the brand crossing (Types §4.2, §4.6; TYP-14).**
`char := brand(u32)` with the prelude's `to_char := @convert fn(n : u32) -> char` was answered twice
and wrongly. §4.6 put the builtin lattice before the in-scope `@convert`, and §4.2 called a brand
crossing over the same numeric domain zero-cost; read together `char(n)` was the crossing, so the
conversion's body never ran, its code-point guard never fired, and `char(1114112)` answered
`1114112` silently. §4.6's dispatch for a single-argument `T(v)` is now: the required-type gate; then
**the one in-scope `@convert`**, except in a body carrying the `unchecked` grant; then a builtin
lattice conversion. In an `unchecked` body dispatch takes the crossing, so the conversion's body — and
any guard written in it — does not run, which is what CT-11 and Stdlib appendix §3.2's "no check under
`unchecked`" already required. §4.2's "cost: zero" is read as a statement of cost, not of precedence;
the clause that a `@convert` "cannot override a builtin lattice class" is narrowed to the **numeric**
classes, which is what it always covered. Consequence for the library: `to_char` becomes `pub` — an
unimportable conversion is in scope for no one — and guards with `trap()`, not `panic`.
(spec#22; compiler#364, #1244.)

**Normative — a target revision with published deviations (refines FND-13 and TOOL-21).**
TOOL-21 required `version` to report "the revision of this specification it conforms to", and
Overview §6 defines conformance strictly. An implementation still in development conforms to no
revision completely, so the rule left it two untruthful choices: print a revision it does not
conform to, or fall back to an older one whose gaps are no fewer, only unlisted. Overview §6 now
lets such an implementation name a **target revision** instead. It must publish its known
deviations (a list for the naming implementation version, kept current, publicly readable where
the implementation says; each entry the rule, by section or decision ID, and the observable
difference), and it must not present the target as conformance. An implementation that does
conform to a revision reports that conformance. Under TOOL-21, `version` reports the target
marked as such, with where the deviations are published. (Overview §6; Tooling §4, §7 item 4;
FND-13; TOOL-21; glossary; aspects map; README.)

## 1.6.0 — 2026-10-27

**Normative — the derived `Pointer` arm is granted library code, not a mechanism outside the lattice**
(Stdlib appendix §2.6, Memory §4.5; MEM-9). 1.5.0 named the derive's pointer step a "compiler-known rule"
with "no source expression at that point to confine". That was wrong in the direction it mattered, and it
invented a mechanism: the derives **are** library code in this language — `lib/base/derive.al` is a
hand-written `comptime match` over `typeinfo(T)` — so there *is* a source expression at that point, and
§4.5 does reach it. The correct shape is the one the same stdlib already uses for a pointer at library
level: `Option::get` matches `typeinfo(T)` for `Pointer(_)` and then spells its crossings as explicit
`unchecked bitcast` (`lib/base/option.al`). The derive's `Pointer` arm therefore expands to
`unchecked bitcast(usize, …)` **inside the generated body**: every confined crossing is named and granted
where it is written, and the author's checked `hash(p)` / `p == q` still need no grant of their own,
because the grant is not theirs to write. Nothing observable to a user moves — checked code keeps hashing
pointer-bearing keys, an author's own `bitcast(usize, self)` outside a grant stays ill-formed — but the
mechanism is now one the language already has instead of a new one, and §4.4's "bitcast is the only
reinterpret form" survives intact. The chapter text is corrected in both places it was stated.

**Follow-up this creates in the compiler:** `lib/base/derive.al` has no `Pointer` arm today and falls
through to `return u64(v)` at `:149` — a `T(value)` conversion for which the lattice has no class, since
§4.4 makes `bitcast` the only reinterpret form. It needs the granted arm the text now describes. Filed
against `compiler/`.

## 1.5.0 — 2026-10-27

**Normative — the derived walk over a `Pointer` leaf is a compiler-known step, and what a declared
heterogeneous relation's body must spell** (Stdlib appendix §2.6, Functions §1.4; MEM-9, MEM-10). 1.4.0
said the derive compares a `Pointer` leaf "by its address bits" and needs no grant, while also calling the
derives "library code over `typeinfo` for **every** type" — but a library body must spell its reads, and
every source spelling that reads address bits out of a pointer is `bitcast`, which §4.5 confines. The
reference implementation showed the gap: `lib/base/derive.al` has no `Pointer` arm and falls through to
`u64(v)`, a `T(value)` conversion for which the lattice has no class (§4.4 makes `bitcast` the only
reinterpret form), so two implementations diverge on `hash(p)`. The chapter now says the `Pointer` arm is a
**compiler-known rule** with no source expression at that point to confine, and scopes the derives to every
type **whose leaf the lattice can walk**. It also closes a hole the same paragraph left: a body writing
`a == b` inside `eq(in a : ptr(T), in b : usize)` re-enters the pair it defines and recurses, so the body
must spell its own crossing inside its own grant — `bitcast(usize, a) == b`, a same-type pair.

**Editorial — one collision resolved and four citations corrected** (Functions §1.4, Memory §4.5, Stdlib
appendix §2.6, MEM-9). Functions §1.4 answered its own question twice: "admissible for every pair of types
**except** pointer↔integer at a comparison site" against "may exist and **be selected by this mechanism**".
Resolved in the direction MEM-10 decided — the binding is admissible and selectable, the **site** is what
is refused — and the false claim that §1.4's mechanism is unchanged, made inside a paragraph that narrows
§1.4, is removed. Memory §4.5's "any other integer type at this position is ill-formed" was unconditional
and collided with §1.4's permission for every other pair; scoped to this operator position. MEM-9's decision
record still said the granted comparison is "hardware-defined", which §4.5, §2.6 and the CHANGELOG all deny
— scoped to the operand. §2.6 attributed the pair rule to MEM-9; it is MEM-10's. The CHANGELOG repeated the
§7 overclaim that 0a1aa16 had already fixed in the chapter.

## 1.4.0 — 2026-10-27

**Normative — the granted comparison site, and what a grant covers** (Memory §4.5, Stdlib appendix
§2.6; MEM-9, MEM-10). Two passages of 1.3.0 contradicted each other on the half of the rule that
proves it is a grant and not a ban. Stdlib appendix §2.6 stated the well-formedness condition
universally — a site is well-formed when a comparison relation is **declared** for its operand pair —
and carved out only that pointer↔integer cannot be made well-formed in **checked** code, without ever
saying what stands in for a declaration **inside** a grant. Memory §4.5 said the site is legal inside
a grant on its own. Read literally, §2.6 refuses `unchecked { p == 0 }`, which is exactly the program
§4.5 exists to allow, and the two chapters then license different verdicts on one source — the FND-3
divergence, in the paragraph written to prevent it. §2.6 now names the compiler-known operator relation
of Memory §4.5 as **standing in for a declaration** inside a grant, and says why no author-declared
`eq` is needed or possible there (`eq`'s parameters are the author's own types). The reference
implementation already reads §4.5 this way: #1034's accept fixture runs the granted comparison with no
`eq(ptr(T), usize)` declared anywhere, and its refusal is keyed on the operand types plus the grant,
never on a declared relation.

The same review moved a premise into the text that was resting on it. Whether `unchecked` reaches a
**sibling subexpression of the same statement** was decided nowhere: the glossary says granularity is
the author's choice (one line or a block), and the answer lived only in a changelog parenthetical —
yet nine of the sites this revision counts depend on it. Memory §4.5 now states it: a grant covers the
expression it is written on, not the statement that contains it; the grant is raised where the crossing
value is walked, and a sibling does not inherit it. So in
`deref(unchecked bitcast(ptr(mut usize), base)) = bitcast(usize, s.ptr)` the left-hand address
computation is granted and the right-hand read of `Slice.ptr : ptr(T)` out as an integer is **not**,
and is ill-formed; a block grant covers every crossing in its block.

**Editorial — the measured cost of 1.3.0 was inflated by three sites it does not touch, and four
citations were wrong** (CHANGELOG, MEM-9, MEM-10). `subword_ptr_deref.al:5`,
`issue345_ptr_target_spacing.al:13` and `deref_subword_field_write_raw.al:11` read
`bitcast(usize, r)` where `r` is the result of `sys_mmap`, declared `-> isize`: an integer→integer
**reinterpret inside the numeric domain** (Type System §4.2), neither direction of integer↔pointer, not
capability-requiring, legal after this rule. The 1.3.0 entry counted them and prescribed a grant for
them, which would have added dead `unchecked` scopes. The cost is **at least 14 sites in 8 tracked
fixtures**, not 17 in 11, corrected here and in MEM-9's own record where the inflated figure was
recorded twice. The unreproducible "225 occurrences in 108 fixtures" becomes the figure that
reproduces, with its command stated (`git grep -o 'bitcast(usize' -- 'test/*.al'` → 358 occurrences in
144 fixtures on `main`). I1 was cited in four places for the bar that two implementations must not
diverge on whether a source is well-formed; that bar is **FND-3** — I1 is codegen transparency, the
programmer predicts the emitted assembly — and this correction had already been made once at 6b05dbf
before coming back. The quotation "refused before overload resolution" was attributed to Memory §4.5;
it was written in Stdlib appendix §2.6 and removed there, and is now attributed where it stood. The
normative chapter no longer cites the reference compiler's internal kind names as spec vocabulary; the
glossary reconciles the decision record's word **established** with its own two terms.

## 1.3.0 — 2026-10-27

**Normative — a pointer meeting an integer is grant-only** (Memory §4.5; MEM-9, #1030).
Comparing a pointer with an integer — `p == 0`, `p != 0`, or an ordering against one such
as `p < 4096`, the three forms §4.5 itself lists — was unnamed: §4.5 made *fabricating* an
address from an integer capability-requiring and said
nothing about comparing one, so the checker accepted `return p == 0` with `rc=0` while
`scripts/ptrint_census.sh` already reported the shape as an OP-CMP row (#1029 proved the hook
fires). It is now ill-formed outside an `unchecked` grant, and inside one it tests the
pointer's representation — the integer operand being `usize`, the pointer's own width, with **no**
conversion performed (the lattice, Type System §4.2, gains no edge; implicit conversion stays confined
to widen, §4.3). What is target-dependent is the representation, not the comparison: given two `usize`
values every conforming target answers alike (I6; I11 as §4.5 uses it for address arithmetic), and
`p == -1` has exactly one reading — the all-ones address of the pointer's own width, `2^N − 1`
(`usize` is a prelude alias for the target's pointer width, Type System §7 — which states the width,
not the number; `2^32 − 1` follows on i386/aarch32/riscv32), by CC-8. The rule is the grant
model §4.5 already applies to its neighbouring operation, and MEM-8 makes the checked case
vacuous rather than merely discouraged: every `ptr(T)` is non-null and valid, so a checked
`p == 0` cannot be true and the question the author meant is `Option(ptr(T))`, whose `None`
is the all-zero niche (§4.2). Found by measurement, not by reading: the tracked corpus reports
`op-cmp=0`, so the **comparison** half of the rule changes no verdict the census measures — 12 tracked `lib/` modules remain
blind to it (`blind-lib=12` in the same record), which is stated rather than glossed. The gap closed
is between detecting a form and refusing it. The same decision confines an integer↔pointer conversion in
**either direction** (§4.5's title and Grammar §3.7 already said `int↔ptr`; the body confined only
integer→pointer, leaving `bitcast(usize, p) == 0` outside a grant as a two-token bypass). Measured cost,
re-measured three times because both earlier figures were wrong: **at least 14 sites in 8 tracked
fixtures** write an explicit `bitcast(usize, …)` over a **pointer** outside any grant and become illegal.
Five of them sit on four lines, one line twice (`hashmap_str_key.al:52` compares two):
`require_ptr_ok.al:3`, `require_ptr_trap.al:2`, `linked_walk.al:9`, `hashmap_str_key.al:52` (×2).
**The first two** are `@require`-style non-null predicates over `ptr(u64)`, so the fix is not purely
mechanical. A first re-count missed a further **9 sites in 4 fixtures** by counting lines that contain
`unchecked` as granted: a grant scopes **one expression**, the one it is written on, and does not reach a
sibling subexpression of the same statement (§4.5, "What a grant covers"), so in
`deref(unchecked bitcast(ptr(mut usize), base)) = bitcast(usize, s0.ptr)` the grant covers the LHS and the
RHS — reading `Slice.ptr : ptr(T)` — is ungranted. Those are `slice_len_field.al:19,21`,
`slice_local_arg.al:26,28`, `slice_str_read.al:21,23`, `str_join.al:18,20,22`. A second re-count then
removed **3 sites in 3 fixtures** that this rule does not touch at all:
`subword_ptr_deref.al:5`, `issue345_ptr_target_spacing.al:13` and `deref_subword_field_write_raw.al:11`
all read `base := bitcast(usize, r)` where `r` is the result of `sys_mmap`, declared `-> isize` — an
integer→integer **reinterpret inside the numeric domain** (Type System §4.2, reinterpret row), neither
direction of integer↔pointer, so it is not capability-requiring and stays legal. The earlier text prescribed
a grant for them; that instruction was wrong and would have added dead `unchecked` scopes. The figure is a
**floor**: separating a pointer operand from an integer one at `bitcast(usize, …)` needs the operand's
type, so a lexical sweep over `test/` cannot prove completeness — `git grep -o 'bitcast(usize' -- 'test/*.al'`
counts **358 occurrences in 144 fixtures** on `main`, and only the ones whose operand is a pointer are
affected. The typed count is the gate's, on the run that lands this. `lib/` and
`src/` hold **zero** such sites, so the self-host build is untouched — the cost is confined to fixtures,
which the compiler fixes under its own gate. The census could not have reported this: it excludes
explicit bitcasts by construction ("Every integer/pointer crossing below is EXPLICIT, so the seam decides
nothing and the census must be empty", `scripts/ptrint_census.sh:462-463`), so its `places=3` measures the
implicit half only, and quoting it as the cost of a both-directions rule was a category error. Owner decision B of #1030.

**Normative — a comparison resolves over its operand pair, and the grant attaches to the pair** (Stdlib
appendix §2.6, Functions §1.4; MEM-10, refining MEM-9; spec#20). The spec never required a comparison's
operands to share a type, while Functions §1.4 permits heterogeneous bindings verbatim — so an author could
declare `eq(in a : ptr(T), in b : usize)` and, on §1.4's "prefers the more specific definition", bind
`p == 0` to it, reaching the site MEM-9 refuses. The answer first written was an **ordering** — Stdlib
appendix §2.6 said the site is refused "before overload resolution" (the words were never in Memory §4.5,
and are removed there), which is the wrong instrument: an ordering between two rules is invisible to a second
implementation, and FND-3 requires the rule to be in the text. The owner's direction was that
expressing one's own types with a comparison relation is needed **and** must follow the language's
principles, so neither a blanket same-type rule nor an ordering was adopted. A site is now well-formed when
a comparison relation is **established for its operand pair**: same-type pairs by the structural default;
heterogeneous pairs of the author's own types by a concrete `eq(in a : A, in b : B) -> bool`, structural in
the existing protocol sense (satisfied by providing the operation, not naming an interface) and additive
(FND-6). The pointer↔integer pair is confined **by the pair itself**: the grant attaches to the operand
types at the site whatever overload set exists. **A declaration cannot supply the grant, and neither can
its body**: a relation over that pair may be established even by a body whose comparison sits inside its own
`unchecked { … }`, and the site still requires its own enclosing grant, because the requirement is a property
of the operand types at the site and not of where a function places a block. No new syntax; §1.4's mechanism
is unchanged and is named in §4.5 as a permission, not the refuser.

## 1.2.0 — 2026-10-24

Two gaps, both found by measuring the reference implementation rather than by reading the
text, and both silent: the counter could not say which backend was wrong because the text
did not decide. Each is normative, so MINOR moves (I10: no enumerated v1 construct is
removed, MAJOR stands).

**Normative — signedness is a property of the value, never of the expression's shape**
(Concurrency §6.5; CC-8). The result type of unary `−` is its **operand's** type: over
`u64`, `-a` is the negation modulo 2^64 — `2^64 − 5` for `a = 5`, a representable value —
and **not** the signed `-5`. Because the operand types are unchanged, an operation's checked
guard is selected by its **operand types** and by nothing else: unsigned tests the carry out
of the result, signed tests signed overflow. An operand that *arrives* from a negation, a
cast, a call or a load does not make the operation signed; the shape of a subexpression is
not a type (I11). This also fixes what §12 item 3's "signedness in the operations" means —
the operand's interpretation selects the operation; it is not permission for an operator to
mint a new signedness for its result. Found as `u64(30 + -a)` with `a : u64 = 5`: x86_64
emitted a signed guard and answered `25` where the unsigned sum overflows and must trap.
Readings B (unary `−` yields signed) and C (forbid the form) are recorded with their costs
in CC-8; B was rejected because it makes the inference that #546, #764 and #766 are made of.

**Normative — `unchecked` is written by the programmer, never introduced by a pass**
(Concurrency §6.6; CC-9). No compiler pass — lowering, IR builder or peephole — may open an
`unchecked` scope the source does not contain. A grant the compiler invents moves the
checked-by-default decision of §6.1 to a level where it is invisible in the program, and it
makes every grant-reporting instrument false: a tool that answers "was this place inside a
grant?" from the lexical nesting of a walk reports `in` for a place no author granted. A pass
needing an unguarded wrap says so with an operation mode or a distinct IR operation, and the
provenance of every grant in its output stays distinguishable — source-written or
pass-introduced, never conflated.

## 1.1.0 — 2026-10-23

The 2026-10-23 coherence audit: the checked-guard family, region/arena temporal safety, the
function-value ABI, the built-in catalogue, per-OS syscall availability and the grammar gaps.
Every entry below is normative; none removes a construct the v1 language enumerated, so the
MAJOR digit stands still (I10).

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
context (hosted, `freestanding`, bare; not under `no_abstractions`). A library-defined numeric type (`uint(N)`, and so `u128`) MUST fail a check that
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

**Normative — the grammar catches up with the chapters.** The consolidated EBNF lacked five
forms the chapters already used. A `match` arm list now derives `comptime-for-arm` (`comptime
for x in <comptime collection> { arms }`, unrolled), which was prose only. `type-atom` names
`dyn-type` instead of repeating it. A range may omit its lower bound — `a[..hi]`, `a[..]`,
index position only, half-open only (there is no `..=hi`). `scoped` is a contextual keyword,
only before `ptr` in a type position, and `brand` is a prelude identifier whose `brand(T)` is an
ordinary call, so the dedicated `brand` type production is gone. A bracket form in an argument
or expression is a type when its first component denotes a type, so `Option([u8])` is an
option of a byte slice; an array of type values is written `[type; 1](u8)` (SYN-2; Grammar
§2.3, §3.3, §3.4, §3.5, §3.7; Types §11; Control Flow §5.4, §11).

**Normative — `trap()` is not admitted under `no_abstractions`.** `trap()` emits one
instruction, but one the target chooses, not one the source names, so it fails the
`no_abstractions` allow-list ("emits exactly the instruction(s) it names"): a `no_abstractions`
unit writes the per-arch checked-failure instruction itself (`ud2` / `brk #0` / `udf #0` /
`ebreak`) (CG-12; CG-13; Assembly §1.1; Stdlib §4.1; Stdlib appendix §4.1; glossary).

**Normative — an unchecked divide without a hardware divider returns the ISA family's
architectural values.** On an `aarch32` core without `idiv` or a RISC-V core without `m`, `/`
and `%` lower to a runtime helper whose `unchecked` result was unspecified. The helper now
returns exactly the values of that ISA family's architectural divide (the per-arch appendix
§2 table: `aarch32` `0` / `x` / `MIN` / `0`; RISC-V all-bits-set / `x` / `MIN` / `0`) and never
faults, so the result is deterministic across implementations (CG-13; Concurrency §6.2;
per-arch appendix §2, §6.2, §7.2).

**Normative — a `naked` function has no Alatyr-level interface.** `@abi(naked)` meant "no
frame, no prologue, no return sequence", but nothing said what its parameters or result were,
or how an Alatyr call to it would place arguments. A naked function now declares **no
parameters and no result** (`fn() { … }`), is **not callable** from Alatyr code (a located
Semantic diagnostic), and is **not** a function value; it is reached only by a raw `call` /
`jmp` from assembly, as the entry, or by the hardware. Its frameless body has no stack slots,
so its locals are `@reg(name)`-pinned. On `Container.com` (default `naked`) a function called
from Alatyr code declares a user `Abi(...)` (FN-9, FN-10, CG-12; Functions §6.4, §10; ABI
appendix §2(b), §3.1; Assembly §1.1; glossary).

**Normative — an `@extern @abi(c)` function may be taken as a function value.** Modules §7.1
said an `@extern` "cannot itself be used as a first-class function value", which left C
callbacks (a `qsort` comparator) without a form. Under FN-10's one rule a function value has the
target's default convention, and `c` names that default wherever it exists, so a non-variadic
`@extern` with `@abi(c)` (or no `@abi`) may now be taken as a value — the external symbol's
address; any other selector is FN-10's diagnostic, and a C-variadic `@extern` has no fn-type
(FN-8, FN-10; Functions §1.5; Modules §7.1; glossary).

**Normative — `clamp` with `lo > hi` and an `@alloc` that cannot allocate fail through
`trap()`.** Both said only "traps". `clamp`'s `lo <= hi` is an operand-domain check (like
division by zero), made under `comptime if verify.checked`, failing through `trap()`, never
`panic`; its body is now specified (`lo` if `x < lo`, else `hi` if `hi < x`, else `x`), so the
unchecked result is defined. `@alloc`'s exhaustion (`OutOfMemory`) fails through the direct
inline trap in every verification mode — there is no value to bind, so `unchecked` cannot drop
it; the recoverable path stays `allocate(…)?` (CG-13; MEM-4; Memory §2.4, §5.4; Stdlib §4.1;
Stdlib appendix §2.6, §5.2.1; glossary).

**Normative — a call to `arena_over` requires `unchecked`.** `arena_over(buf, cap)` keeps the
address of a caller buffer inside an owning `Arena` that can be moved, returned or stored past
`buf`'s scope, and the local escape rule cannot tie the result to `buf` without a lifetime
variable — so checked code could use an arena over dead storage, against I11. A call to
`arena_over` is now a raw operation (CG-7): ill-formed in checked code and under `no_unchecked`,
written `a := unchecked arena_over(ptr(mut buf), cap)`, the author vouching that `buf` outlives the
arena; all later use stays checked. **Over a place declared `@static` the call is checked**: the
declaration supplies the fact the author would have to vouch for, the test stays syntactic (I1,
FND-3), and PRIN-2 will not accept forcing a `freestanding` program over a static pool into
`unchecked` just to allocate (MEM-3; Memory §5.4, §8; Stdlib appendix §5.2.1, §8; Grammar §3.7;
CG-7; glossary; aspects map).

**Normative — no call-site temporary allocator; an omitted allocator argument names an existing
place.** Allocator shape C let a §5.1 default create a fresh allocator per call, but an `Arena` is
owning and nothing would `close` it. An omitted allocator argument now always names a place that
outlives the call: the ambient (a binding in scope), or a §5.1 default `ptr(mut P)` with `P` of
static storage reached without a pointer step; a default that creates an allocator is ill-formed.
Temporary memory is a local arena, `defer close(a)`, then an explicit argument or
`alloc::with(a) { … }`. An elided argument is a use of the ambient place, so
`alloc::with(a) { close(a) ; f() }` is a linearity error. A compiler-closed call-site temporary is
recorded as rejected for now, additive later (MEM-5; Memory §5.2.1; Functions §5.5; glossary).

**Normative — a call to `str_at` requires `unchecked`.** `str_at(p, n)` builds a `str` from an
integer address and was documented as "unchecked, encapsulated in the one function"; but
`unchecked` is non-transitive, so checked code could call it and read arbitrary memory, against
I11. A call to `str_at` is now a raw operation (CG-7), like `arena_over`: ill-formed in checked
code and under `no_unchecked`, written `unchecked str_at(p, n)`, the caller vouching for valid
UTF-8 that outlives the view. The `bitcast` catalog row now states that a `bitcast` to a pointer
type is the same `unchecked`-only fabrication Memory §4.5 already required (CG-7; Memory §4.5, §8;
Stdlib appendix §3.6, §4.1, §8; Grammar §3.7; glossary).

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
