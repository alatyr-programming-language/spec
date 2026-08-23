# Decisions — Types, layout, branding, construction

**Why the language is the way it is** for the type system. Each entry records the
*rationale* and the *rejected alternatives*; the normative *what* lives in the spec
(linked per entry), the *how* in the compiler. IDs are theme-prefixed (`TYP-N`) and
**append-only** — assigned in articulation order, never renumbered (a later sub-theme
appends higher numbers, it does not shift these). The `provenance:` line records what an
entry refines and where it lands in `spec/`.

---

## Kernel, branding, construction

### TYP-1. Types are values; one comptime logic
*spec: Types §1, §5*

**Why.** A separate machinery for generics, constraints, "traits", and type identity
would multiply the language. Instead there is **one comptime logic**: `type` is an
ordinary value; a **generic is a comptime function** from values (types included) to a
type; functions are derived types; constraints/"traits" are **comptime predicates over
types**, not separate machinery. Every type **decomposes into the data blocks of the
machine model**. This single foundation lets the stdlib build the whole derived type
vocabulary in ordinary library code (TYP-5) rather than as privileged compiler entities.

### TYP-2. The kernel is raw `bitsN`; `u`/`i`/`f` are interpretations
*spec: Types §2, §3*

**Why.** The kernel must be exactly the *assembler-level* vocabulary and nothing more, so
that codegen stays transparent (I1) and the cost of every type is visible. The kernel
primitive is therefore a **raw block of bits `bitsN`** at **native widths only**
(`bits8/16/32/64`, the set chosen by the target, I6) — a block has no arithmetic, only
interpretation-independent operations (bitwise, shift/rotate, bit-equality, load/store,
`bitcast`). `uN`/`iN`/`fN` are **numeric interpretations** of one block, defined in the
prelude on top of the kernel: a single name (`u64`), a target-dependent realization, a
visible cost. **Signedness lives in the operations, not the storage** (LLVM `iN`
precedent): which instruction the operator body selects (`divq` vs `idivq`, `setb` vs
`setl`) *is* the signedness, so the compiler holds **no built-in numeric/signedness
table** — `bitcast` is identity on the block, zero-cost by construction. The
interpretation a brand *names* also fixes its ABI register class (an `fN` → floating-point
regs), keyed on the named interpretation, **not** on peeling to the `bitsN` base (peeling
`fN` to `bitsN` would mis-class a float as an integer). Widths wider than native are
**derived** as multi-word library values via the prelude type-function **`uint(N)`** (v1:
`N` a positive multiple of 64, `N/64` little-endian words — TYP-10); sub-native and
non-multiple-of-64 (partial-top-word) widths are additive (TYP-10). So the kernel stays
minimal while the surface is uniform. Consequence: the stdlib is the kernel's first and most demanding consumer,
which forces the construction machinery (TYP-5) to be genuinely expressive.

**Rejected.** Putting basic `u`/`i` into the kernel too (less clean — V2); numbers as the
kernel without a raw block (V3); a `brand`-carried numeric-domain descriptor the compiler
reads (keeps operator lowering inside the compiler — rejected in favor of library
operators).

### TYP-3. Layout primitives — a finite, complete, closed set
*spec: Types §5, §6*

**Why.** Memory layout needs primitives that are **complete for machine memory** yet
closed, so there is no research-level risk of open user-defined layout kinds. The set is
fixed: scalar leaves (`bitsN`) + product (struct/tuple) + sum (tagged enum) + array + raw
union — all further shapes come from **composition + comptime**, not new primitives. The
layout *rules* are chosen for **predictability** (I1): struct/tuple fields in declaration
order with **no auto-reorder** and standard padding; this default already coincides with
the platform C struct layout, which removes the need for a "match a named ABI" type lever
(see TYP-7). One general enum rule governs the tag (smallest fitting `uN`, discriminants
from 0 in declaration order, **placed first at offset `0`** with the payload after it —
the position is fixed, since §8's closed lever set contains nothing that moves it and only
`@repr(T)` pins its *type*; an earlier wording said the position "MAY be overridden",
which named a capability with no surface); a **zero-variant** enum is **ill-formed** (uninhabited — the
named bottom type `Never` is the language's only uninhabited type, so an empty `enum {}` would
be a redundant entity, OP-1). Niche folding is via the constructive `niche` lever, so
`Option` is **unprivileged** — `Option(ptr)` is pointer-width zero-cost because the
pointer supplies a null niche, not because `Option` is special. The positional `.N`
projection is **tuple-only**: arrays use `a[i]` (a `.0` would merely duplicate `a[0]`,
OP-1), and an enum payload is reached by `match` because a positional `e.N` would be
unsound across variants (I11). A raw union (untagged, all members at offset 0) is kept
distinct from a tagged enum; its complete payload, construction, layout, projection,
and definitely-active contract is TYP-11.

### TYP-4. Branding — separate nominal identity over a shared layout
*spec: Types §4.1, §5*

**Why.** Type identity is **nominal by declaration**: each named declaration is a
distinct type even with a coinciding layout, so distinct concepts (`u64` vs `i64` vs
`Meters`) cannot be silently confused. **Branding** is the single primitive that mints
such separate nominal identity over a *shared* layout (`u64`/`i64` are brands over
`bits64`; `Meters` is a brand over `u64`) — this is exactly what TYP-2 relies on to build
the numeric interpretations as library brands. **Structural identity is reserved for
anonymous layouts** (tuples, unnamed blocks), where there is no name to be nominal by.

### TYP-5. Construction machinery — constructors are comptime functions; operators are library functions
*spec: Types §5, §10*

**Why.** Because the stdlib itself builds `u/i/f`, `usize`, `char`, `Vec`, `Option` over
the kernel (TYP-2), the construction machinery must be expressive enough to be the
language's own dogfood — which validates it. So the surface syntax **unfolds into the
layout constructors**, and the constructors are **callable in comptime** as
type-producing functions (`fn Vec(T: type) type { return struct{…} }`, TYP-1).
**Operators are sugar over overloadable library functions** (builtins, OP-1), uniform for
the kernel and for derived types: a native operator's one-instruction body uses the
instruction intrinsic as a statement (destination-first) and the compiler **inlines** it
to that single instruction (I2 zero-cost), so the kernel keeps **no** built-in numeric
knowledge — signedness/overflow are entirely in the operator bodies and guard code.

**Rejected.** A `brand`-carried numeric-domain descriptor the compiler reads to lower
operators — it keeps operator lowering inside the compiler instead of in the library.

### TYP-6. Conversions — a lattice of explicit classes; only widening is implicit
*spec: Types §4*

**Why.** Open user-defined implicit coercions (C++ style) are a footgun and explode spec
complexity, so conversions form a fixed **lattice of classes** (reinterpret / widen /
narrow / numeric / brand), and the implicit↔explicit line is **tied to the limits**:
under `no_abstractions` nothing is implicit; otherwise **only lossless,
meaning-preserving widening** is implicit, and narrow/numeric/reinterpret/brand are
**always explicit**. The explicit form is uniform — `T(value)` — and **`bitcast` is a
distinct operation, not a spelling of `T(value)`**: `f32(x)` is a numeric conversion that
preserves the *value* and emits an instruction, while `bitcast(f32, x)` preserves the
*bits* and costs ~0 instructions (so the costs stay visible, I1/I2). User-declared
conversions plug into the **same `T(value)` form** via a function marked **`@convert`**
(single source parameter, target = its qualified return type), resolved by ordinary
in-scope name resolution — never a global registry or an orphan rule. The marker is the
**attribute, not the binding name**, so two conversions to one target can coexist and
foreign/deeply-nested targets are expressible. A `@convert` is always explicit and cannot
override a builtin lattice class; multiple matches at a site are an ambiguity error (FND-3,
no guessing). This is what lets the language **privilege no `bool`↔integer convention**:
that conversion becomes a library `@convert` (the author fixing checked-narrow vs
truthiness-fold by choice), while the base tier still ships the idioms (`n != 0`, `if b
{1} else {0}`), not the conversions.

TYP-12 adds one non-lattice constructor dispatch before `@convert`: for a required target
`R = require(U, pred)`, an argument assignable to the immediate `U` selects the contract
gate. A non-`U` source may still select a **direct** `@convert S`→`R`, but dispatch never
composes `S`→`U` with the gate and a competing `@convert U`→`R` cannot override it.

**Rejected.** Open user-defined implicit coercions (C++ footgun); literal suffixes like
`5u8` (would work only for a privileged set, against uniform `T(v)` — see TYP-8);
return-type-directed selection of any plain `(S) -> T` as a silent conversion (implicit,
un-greppable, breaks I3); binding-name-equals-target as the `@convert` marker (fragile for
foreign/qualified/same-simple-name targets); the name `parse` for the conversion builtin
(conversions are `T(v)` / `bitcast` / `@convert`, not a `parse` function). *Note:* the earlier flat prohibition on
`bool(n)` was dropped by TYP-6 — the language now privileges no such conversion.

### TYP-7. Layout control — closed per-site levers, not `@abi`
*spec: Types §8*

**Why.** Any layout must be constructible **by hand and visibly** (it is part of the type
contract, I1/I3), so the levers are a **closed** explicit set of overrides on the TYP-3
defaults: `@packed`, `@align(N)`, `@offset(N)` (MMIO/register maps; overlapping fields are
union-like), and per-field/type `@endian(big|little)` for wire formats. Because the
default struct rule already coincides with the platform C layout (TYP-3), **no type-level
"match a named ABI" lever is needed** — `@abi` is reserved for *calling conventions* on
functions, **not** type layout in v1. These levers also illustrate the general modifier
classification: a small **fixed core of role-changing keywords** (`pub`, `mut`,
`comptime`, `unchecked`) vs an **open set of representation/conformance `@`-attributes**
(storage, allocation, layout, external ABI) — interface/role as keywords, representation
as attributes.

**Rejected.** `@abi(c) struct{…}` as a data-layout (C-layout) lever — dropped: the `Abi`
value schema carries only calling-convention fields, so it had no meaning an
implementation could lower without guessing (a FND-3 gap). A type-layout ABI is additive
(FND-6) once a data-layout schema is specified.

### TYP-8. Value construction — context-typed literals, explicit defaults, definite assignment
*spec: Types §9*

**Why.** Literals are **comptime numbers typed from context** (capacity checked at
compile time, out-of-range is an error, I11), with a documented default when there is no
context — refined via the uniform constructor `T(v)` or an annotation rather than
**suffixes** (which would privilege a fixed set, against uniformity). Aggregate
constructors fill the bytes by the layout recipe (TYP-3) and also run in comptime.
Crucially, **the language imposes no implicit default/zero-init on an uninitialized
binding** (it never silently makes an omitted value) — but a documented operation may
write fixed representation bytes, as the raw-union constructor explicitly clears its
backing storage before component stores (TYP-11). **Programmer-declared defaults are
explicit and visible**: a struct field
or array element may carry a type-level default, applied *at construction* when omitted,
never auto-on-declaration. Reading uninitialized storage is forbidden (I11), enforced by
**field-sensitive definite-assignment analysis** (definitely-assigned = assigned on every
control-flow path; a diverging arm contributes none); partial init is permitted, a
declared default counts as initialization, and `uninit` is the explicit opt-out.

**Rejected.** Literal suffixes (`5u8`) — privilege a set, against the uniform `T(v)`
form. Implicit language-imposed zero/default initialization of a binding — the language
does not invent an omitted value; this is distinct from an explicit constructor's
documented representation writes (TYP-11).

### TYP-9. Comptime field projection — `a.(f)` reads a field by descriptor
*spec: Types §6.1 (unchanged); Comptime §5.4*

**Why.** The v1 structural derives (`Eq`/`Ord`/`Hash`) were declared as **library code
over `typeinfo(T).fields`**, yet the only field-access form was `a.ident` — a *source
literal* name — so a derive iterating the field list **could not read the corresponding
field** of a runtime value. That is a FND-3 gap (declared-but-inexpressible). `a.(f)`
bridges it minimally: it projects the field identified by a comptime descriptor (a `Field`
or its `str` name), **resolved at comptime** to the named access `a.<f.name>` — the same
place, type, and per-field write permission, emitting **identical** code to `a.name`
(zero-cost, I2; no runtime by-name lookup, no RTTI, I3). It composes the existing
`.`-postfix with the comptime universe — **no new keyword** (OP-1) — so the structural
derives become ordinary library code.

**Rejected.** A `mem::field(in p, comptime f)` pointer-flavored builtin — reads every
field through a raw address, markedly more verbose at each derive site. Field-by-index
`a.{i}` — positional, less legible than by-name and redundant with the descriptor, which
already carries its `name`.

### TYP-10. `uint(N)` — fixed-width multiword unsigned integers (multiples of 64)
*spec: Types §7*

**Why.** TYP-2 fixed that widths wider than native are **library recipes** (not kernel
types) and named the prelude type-function `uint(N)` as their producer — but left `N`'s
admissible set and the multiword value's **representation** and **operations**
unspecified, so two implementations could not agree on what `uint(192)` *is*, nor on how
its `+` / `/` / `<` behave (a FND-3 gap). This entry pins the v1 form. `uint(N)` is a
**prelude type-function**: given a comptime `N` that is a **positive multiple of 64**, it
yields an unsigned integer stored as a **library multiword value of exactly `N/64` machine
words, little-endian** (word 0 = least significant) — the same recipe the wider-than-native
named integer already carries, so **`u128` ≡ `uint(128)`**. It is a **library recipe in the
prelude, not a kernel/builtin type** (TYP-2 — the kernel is raw `bitsN` at native widths
only, and every wider width must carry visible cost): a `uint(N)` value visibly occupies
`N/64` words. Its arithmetic and comparison are **library operator-functions** (OP-1)
generated from the word count, so the compiler holds **no** multiword numeric knowledge:
ripple-carry `+` / `-` (`O(words)`), schoolbook `*` keeping the low `N` bits (`O(words²)`),
binary long-division `/` / `%` (`O(words²)`), and a hi-word-to-lo-word lexicographic
**unsigned** comparison (`O(words)`). This is the concrete library multiword value TYP-2/TYP-10
demand so that a non-native width (`u64` on a 32-bit target, a `usize` decomposition) is an
ordinary `{word…}` value — **never** backend register-pair magic (the words are visible,
I1/I3).

**Out of scope (additive, deferred).** Non-multiple-of-64 widths — an arbitrary `N` with a
**partial top word** — and their masking / carry / overflow semantics (the earlier
`u3` / `u100` sketch), **sub-native** widths (a byte-plus-mask `u3`), and the **signed**
multiword `int(N)` (two's-complement wider integers, the signed counterpart of this recipe)
are **not** v1: each needs its partial-word or sign rules pinned before it meets the FND-3
bar. All are addable without breaking v1 programs (I10 / FND-6). This scope boundary is
recorded so the deferral is deliberate, not an omission.

**Rejected.** Putting multiword integers in the kernel / compiler (against TYP-2 — the
kernel is native `bitsN` only, and hiding the word count would hide cost, I1). A backend
register-pair primitive for wider-than-native widths (an un-greppable representation — the
whole point of the library value is that the constituent words are visible, I1/I3).
Admitting **arbitrary** `N` in v1 without the partial-top-word semantics (would not meet
FND-3 — the mask / carry / overflow rules would be left to the implementer to guess).

### TYP-11. Raw-union members have one payload; 2+ components form an anonymous tuple
*spec: Types §6.3, §9.3–§9.4, §11; Grammar §3.3–§3.4; Memory §5.9; Functions §4; ABI appendix
§6*

**Why.** The grammar admitted any number of components in a raw-union member while the
type chapter and OP-4 defined only one payload type and even gave a write spelling
(`u.m(v)`) that the grammar classifies as UFCS. That left construction, projection,
padding, constants, copying, and active-member legality to each implementation — a direct
FND-3 blocker. The least-surprising completion reuses the already-specified enum/
`Variant.payload` model: zero components have unit payload `()`, one has `T`, and two or
more have **one anonymous tuple payload** `(T0, T1, ...)`, with ordinary tuple order,
offsets, alignment, padding, identity, and `.N` projections. Different members overlap at
offset `0`; union alignment is the maximum payload alignment and size is the aligned
maximum payload size. The zero-field tuple's value spelling is `()`, making whole-member
assignment to a zero-component member expressible. No tag or active-member byte exists.

Construction is the existing **type-qualified member constructor** (`U.none`, `U.word(x)`,
`U.pair(x, y)`), with positional left-to-right arguments, never a value-call;
`u.m(args)` remains UFCS. Whole-member assignment (`u.m = payload`) evaluates the RHS
before clearing and activates a mutable place. Both operations clear the complete union
storage then copy each component's complete representation in order, leaving padding
introduced by the member tuple and the union tail at zero. Runtime, comptime, and static
materialization therefore share one reproducible operation and visible cost ceiling.
Whole-union copy is exactly `U.size()` bytes and carries no hidden metadata.

With no tag, checked access cannot be a runtime test. A fixed dataflow analysis tracks
only roots and fixed non-pointer subobject paths, extends ordinary definite assignment with
an orthogonal exact-member/unknown fact, and permits `u.m` only when its base is a
trackable place or value expression that is initialized and exact-`m`. Unequal merges,
pointer-derived places, ABI/function boundaries, mutable aliases, and mutable-static entry
are active-unknown. `unchecked`
admits inactive/uninitialized reinterpretation with hardware-defined results (I11). This
is deliberately conservative and identical across implementations rather than allowing
implementation-defined proof strength. Raw unions also reject duplicate member names,
empty declarations, zero-component `m()`, named constructor arguments, member modifiers,
discriminants, an `@owning` root or transitive owning payload, and union-level
`@repr`/`@packed`/`@offset`/`@endian`/`@niche`; `@align` is their sole union-level layout
lever. ABI classification uses the declared union layout, never the active member.

**Rejected.** Treating each component as an independently overlapping member — it loses the
declared member grouping and contradicts `Variant.payload`. A hidden discriminant or runtime
active-member check — violates I2/I3 and turns a raw union into a tagged enum. Making every
read legal in checked code — contradicts CG-7's raw-operation boundary. Requiring every read
to be `unchecked` even immediately after a visible activation — unnecessarily discards the
existing definite-assignment mechanism. `u.m(args)` as a special write — collides with the
language-wide UFCS rule; whole-member assignment and the type-qualified constructor already
express it. Leaving padding unspecified — makes inactive unchecked reads and static bytes
implementation-defined, below FND-3.

### TYP-12. `require` is an explicit constructor gate with an ordinary predicate call
*provenance: refines CT-10/TYP-5/TYP-6/TYP-8/MEM-2/FN-3/FN-5/FN-6 · spec: Types §4.1, §4.6,
§8.1, §9.4; Memory §5.9, §6.4; Comptime §3.1;
Declarations §7.1; Functions §1.5, §4.1; Codegen §4; ABI appendix §6; per-arch appendix §2*

**Why.** The earlier validity-contract sketch fixed the intent — a checked construction
must reject a value for which the predicate is false — but did not fix enough mechanics
for two implementations to agree on aggregate values, copies, callable identity,
mutation, stacking, or global construction. `require(U, pred)` is therefore a **type
constructor whose only injected behavior occurs at its explicit `R(u)` gate**. It yields
a distinct, layout-identical, ownership-neutral type `R` over a non-owning `U`; after
specialization, `pred` is a comptime-known callable with exactly the ordinary interface
`fn(in U) -> bool`.
Named and inline callables obey the same signature, capture, call, and failure rules;
their source spelling does not affect lowering. Callable identity *does* remain part of
the type-function instantiation, so separately written lambdas are not equated by body
comparison.

**The gate.** In checked runtime code, `R(v)` evaluates `v` exactly once into the result
representation, invokes `pred` exactly once through its ordinary conventional ABI on a
**distinct logical by-value argument**, and delivers the preserved result only after
`pred` returns true. A false result emits the target's per-architecture **direct inline
trap**, not `panic`, `assert`, a hook, or unwinding. The ordinary ABI fixes the cost:
small aggregates are copied into their classified argument register/stack slots; large
aggregates get one full byte-copy into the ABI's caller-owned argument temporary and a
pointer to that copy. This is the documented unoptimized cost ceiling (I1); a
semantics-preserving optimization may coalesce storage or inline the call only when it
does not duplicate, reorder, or observably erase the predicate evaluation. In
`unchecked`, the whole predicate path — argument copy, call, branch, and trap — is absent;
only the once-evaluated value is delivered. Checked comptime construction runs the same
predicate once in the evaluator and rejects false at the construction site; module-level
initialization therefore emits no runtime initializer.

**Identity, compatibility, and aggregates.** `R` is identified by the immediate
underlying type `U`, the callable identity, and its comptime capture values. The layout,
size, alignment, and ABI field classification are exactly `U`'s, but `U` and `R` are not
assignment-compatible. The built-in contract constructor accepts one expression
assignable to the **immediate** `U`; it is never implicit, never inherited as
`R(field = ...)`, and never searches conversion chains. Build an aggregate `U` first,
then write `R(u)`. The gate takes precedence over a competing `@convert U`→`R`; when a
source `S` is not assignable to `U`, an ordinary direct `@convert S`→`R` may still match,
but its body must construct `R` and the compiler does not compose `S`→`U` with the gate.
A checked `R` place cannot be initialized field-by-field or mutated through a component
place, because either would bypass the complete-value gate; it may be replaced whole by
another `R`. `unchecked` is the explicit contract escape for component mutation otherwise
permitted by `U`'s mutability, and `bitcast` remains the explicit raw-bit escape.

**Conservative v1 boundaries.** An owning `U` is rejected: preserving the result while
also passing an ordinary by-value predicate argument would duplicate or consume an owning
value, contradicting MEM-2. Predicate captures must be comptime-known and are specialized
away; a runtime capture or `dyn fn` environment is rejected, so the runtime call has no
hidden environment argument. Repeated `@require` decorators are nested nearest-first:
`@require(p) @require(q) T` is `require(require(T, q), p)`; each predicate takes its
immediate underlying type and each explicit constructor gate runs exactly once, so
constructing from `T` is written as `Outer(Inner(t))` rather than via an implicit flattened
chain.

**Rejected.** Lowering false through `assert`/`panic` (adds target- and limits-dependent
IO/hooks instead of the fixed checked trap); permitting owning underlying values (no
coherent preserve-and-copy rule); runtime closure captures (a hidden environment argument);
an implicit `U`→`R` coercion or conversion-chain search (hidden meaning change, I8/I3);
field-by-field construction or checked component mutation of `R` (bypasses the gate);
predicate-body equivalence for type identity (requires undecidable semantic comparison);
flattening stacked requirements (loses the immediate-underlying signature and ordering).

### TYP-13. A float-spelled literal is a float literal — never an integer initializer
*provenance: refines TYP-1/TYP-4 (§9.1 literal typing) · spec: Types §9.1, §9.2; Grammar §2.4*

**Why.** §9.1 fixed integer-literal typing (inferred from context, representability
checked, no silent wrap) but said nothing about the float literal's context type. That
left one shape genuinely open: `x : u64 = 1.0`, whose *value* is integral even though its
*spelling* is a float. An implementation could read "the literal takes its type from
context" as licence to accept it (and one does, silently), while another rejects it —
identical source, two answers, below the FND-3 bar. Any rule that accepts it must also
answer where acceptance stops (`1.0e3`? `1.5`? a comptime float that happens to be
integral?), and every such boundary is a fresh place for two implementations to differ.

**The rule.** A float-spelled literal is a floating-point literal, full stop: its context
type must be a floating-point type, so `x : u64 = 1.0` and `x : u64 = 1.5` are equally
diagnostics — the integer is written `1`. The converse direction stays open in the useful
case: an integer literal in a float context is accepted when it is **exactly**
representable in that format, and is a diagnostic when it is not (`x : f32 = 16777217`),
which is the same representability check §9.1 already applies to integers. Rounding a
decimal float literal to the nearest binary value is representation, not loss; a literal
outside the format's finite range is an error rather than a silent infinity.

**Rejected.** Accepting an integral-valued float literal in an integer context — one
character (`.0`) then decides nothing, the spelling stops carrying information, and the
accept/reject boundary has to be redrawn for exponents, comptime-computed floats and
narrowing. Rejecting integer literals in float contexts — `x : f64 = 1` is the common,
unambiguous case and forcing `1.0` buys nothing. Silent truncation or rounding of a float
literal to an integer type — a silent value change, contradicting I11 and §9.1's
no-silent-wrap rule.
