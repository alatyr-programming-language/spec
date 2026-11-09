# Decisions — Syntax: lexis, separators, grammar, `@` attributes

**Why the language is the way it is** for its surface form. Each entry records the
*rationale* and the *rejected alternatives*; the normative *what* lives in the spec (the
Grammar chapter, `spec/130-grammar.md`, linked per entry), the *how* in the compiler. IDs
are theme-prefixed (`SYN-N`) and **append-only** — assigned in articulation order, never
renumbered. The `provenance:` line records what an entry refines and where it lands in
`spec/`.

---

## Lexis

### SYN-1. Identifiers — case-sensitive, underscore-significant, ASCII in v1
*spec: Grammar §2.2, §2.6*

**Why.** Identifiers are compared by their code points **as written** — case-sensitive
and underscore-significant — so `my_func`, `myfunc`, `MyFunc`, `_tmp` are all distinct.
Case-sensitivity is the C/Rust/Zig norm and the simpler one: it removes any case-folding
barrier entirely, which is what lets non-ASCII identifiers be added later (NFC + UAX #31)
as a clean additive step (FND-6) without a locale- or Unicode-version dependency. v1
identifiers are ASCII (`[A-Za-z]`, `[0-9]`, `_`); exported symbols stay ASCII even after
that future addition, because the linker symbol is the spelling from the declaration
(paths `::`→`__`, C-compatible) or an explicit `@export("…")`. The `_` is significant in
identifiers; it is a non-significant *formatting* separator only inside numeric literals
(`1_000` = `1000`).

**Rejected.** Case- and underscore-**insensitivity** (the earlier Ada-style "typo
protection"): it broke the Type/value casing convention, hurt greppability, caused
surprising import clashes, and would have forced locale-/Unicode-version-dependent
case-folding.

### SYN-2. A minimal keyword set; built-ins are prelude names, not keywords
*spec: Grammar §2.3*

**Why.** Keywords are kept to the minimum the grammar genuinely needs (OP-1), and many
things one might expect to be keywords are deliberately *not*. The verification axis
contributes only `unchecked` — `checked`/`runtime` are the unnamed defaults, not words.
Parameter directions `in`/`out`/`in out` are **contextual** (keywords only in the
parameter position; `in` doubles in `for x in …`). The value-introducers
`struct`/`enum`/`union`/`fn`/`mod` are keywords because they denote distinct literal kinds
(see SYN-6). `type` is a keyword (the *kind* of types, used in `T: type`) but **not** an
introducer — a generic type is the ordinary `fn(…) -> type`. `abi` is reserved but is not
an introducer either (an ABI value is an ordinary `Abi(...)` struct). The built-ins
(`typeinfo`/`bitcast`/`panic`/…) and `true`/`false`/`NaN`/`Infinity` are **prelude
identifiers/constants, not keywords**, so the keyword surface does not grow with the
library (OP-1). `async`/`await` are reserved now for additive async (CC-6) — a safe
direction of breakage. Import/re-export is a plain declaration (`name := path`), so `use`
is gone; `@extern`/`@label` are attributes (SYN-7), so `extern`/`label` are not keywords.
`scoped` is **contextual** like the directions — a keyword only immediately before `ptr` in
a type position (`scoped ptr(T)`, MEM-1) — and `brand` is a **prelude identifier**: `brand(T)`
is an ordinary comptime call returning a type (TYP-4), with no production of its own.
*Rejected:* reserving `scoped` outright (the grammar needs it in one position only) and a
dedicated `"brand" "(" type-expr ")"` type production (it duplicated the call form and made
`brand` a keyword the keyword list never named).

### SYN-3. Literals, text types, and comments — explicit, no hidden block forms
*spec: Grammar §2.1, §2.4, §2.5*

**Why.** Literals are conventional and explicit: integers in dec/`0x`/`0b`/`0o` (with the
`_` separator), floats `1.5`/`1e10`/hex-float, a string `"…"` is `str` (UTF-8 bytes, with
the invariant of valid UTF-8), a char `'a'` is a Unicode codepoint, and exact bytes are
spelled `\xNN` / `[u8; N]`. Text is split into `str` (fixed, base-prelude) and `String`
(growing, alloc-tier), and length APIs carry **explicit units** (`.byte_len` O(1) /
`.codepoint_count` / `.graphemes` O(n)) so that byte ≠ char is named in source rather than
hidden behind an ambiguous "length". Comments are single-line only — `#` line, `##` doc —
with adjacent lines merging into one logical block and **no block-comment form**, keeping
the lexer simple; a `##`-block immediately before a declaration is picked up as its
documentation for tooling and the spec. Sources are UTF-8.

---

## Separators

### SYN-4. Newline-or-delimiter separators with a deterministic continuation rule
*spec: Grammar §2.6*

**Why.** Sequence elements (call arguments, fields/elements, aggregate literals) are
separated by **a comma OR a newline** (one-per-line, trailing comma allowed); block
statements and `match` branches by **a newline OR `;`**. This gives the comma-free
one-per-line style without making commas mandatory. To keep newline-as-separator
unambiguous it follows a **deterministic Go/Swift-style continuation rule**: a newline does
*not* terminate when the line is explicitly incomplete (last token cannot end a
statement — a trailing binary operator/`=`/`,`/open bracket/`.`, or the next line starts
with a continuing token); inside `(...)`/`[...]` newlines are always free. Determinism is
the point — the rule never needs lookahead beyond the line boundary.

**Rejected (carve-out).** The one disambiguation needed: a line-leading *non-prefix*
operator glyph (`/`, `%`, `==`, `<=`, …) normally continues the previous line, but when
**immediately followed by `:=`/`:`** it is an operator-name *binding* (the glyph is the
declaration's name, e.g. `/ := fn(…)`), so the newline stays a separator. This is needed
only for the non-prefix operators; the prefix-capable `+`/`-`/`*`/`&`/`~` already start a
fresh line unambiguously.

### SYN-8. `match` arms take no comma — spec#24
*provenance: refines SYN-4 (separators) and corrects TOOL-2's arm-wrapping extension ·
  spec: Grammar §2.6, §3.7; Control Flow §5.3; Tooling §4.3.3*

**Question.** Grammar §2.6 separates `match` arms by `sep ::= newline | ";"`. Tooling's
canonical layout (§4.3.3) ended every expression arm with a trailing comma, including the
last, and no block arm. A line ending in `,` is incomplete under §2.6, so `E::A => 3,`
followed by a newline joined the next arm onto it, and the formatter's own output did not
parse. Is `,` an arm separator, or does Tooling drop the comma?

**Decision (owner, 2026-11-07).** **Tooling drops the comma, and the grammar stands.**
`match` arms are separated by a newline or `;` only (Grammar §2.6 unchanged). The canonical
layout puts no comma after any arm. A comma after an arm, whether its body is an expression
(`E::A => 3,`) or a block (`E::B => { return 4 },`) or the arm is a `comptime for` template,
and including after the last arm before `}`, is a **Parse** diagnostic located at that comma
(Grammar §3.7). It is located at the comma, not at whatever the continuation rule then
joins from the next line. The trailing separator §2.6 permits before a closing bracket is,
in an arm list, a `sep`, never `,`.

**Why.** Arms use the separator statements use, so the grammar does not grow (PRIN-1) and
the trailing-`,` line-continuation trap is gone. Practice already agrees: at the time of
the decision the compiler's own sources wrote 108 expression arms without a comma and 5
with one.

**Rejected.** *Make `,` an arm separator* (for expression arms, or for all arms). That
would give one thing two spellings, `;` and `,`, and blur §2.6's split between the
statement/arm separator and the sequence-element separator.

---

## Declarations and expressions

### SYN-5. Declaration vs assignment tokens; conventional precedence
*spec: Grammar §3.2, §3.4, §4*

**Why.** The four binding/assignment forms are distinguished by token, Go-like:
`name : T = value` (declare + initialize), `name := value` (declare + infer),
`name : T` (declare without initialization), `name = value` (assign to an existing place).
The `:=` token is what separates a *declaration* from an *assignment*, so neither needs a
keyword. Parameters are typed; genericity is expressed *only* by `T: type` (a marker like
`any`/`?` was removed — it gives no capability beyond `T: type`, only saves names, against
OP-1/PRIN-1). Operator precedence is **conventional** (arithmetic > comparison > logical, `*`
> `+`, …) with explicit parentheses for override/clarity, so the grammar matches existing
intuition instead of inventing a novel table.

### SYN-6. Results are always declared; the introducer-admission criterion
*spec: Grammar §3.2, §3.4, §3.6, §3.8*

**Why.** A function's result is **always declared in the signature** — the single
anonymous projected output `-> T` or a named `out r: T` — so the contract is visible and
cannot silently change (a footgun). A body-expression `{ expr }` is sugar that fills the
single anonymous output (instead of `{ return expr }`); named/several outs are written in
the body; **no result declared plus a trailing expression is an error**, not an implicit
return. `return expr` fills the single anonymous output and exits (guard clauses); with
named outs `return` carries no value; with no out, `return expr` is an error. Because the
result is just a declared `type`-valued output, a **generic type is an ordinary
type-returning function** (`Vec := fn(T: type) -> type { struct{…} }`), and type
identity/memoization keys on *the result being `type`*, not on any keyword. From this
falls the **introducer-admission criterion**: a value-introducer is admitted **iff** it
denotes a genuinely distinct *kind of value-literal* not expressible as an ordinary
call/expression. That admits exactly the type-literals (`struct/enum/union{}`), the
function-literal (`fn(){}`), and the module-literal (`mod{}`); it never admits an
introducer that merely shortens or annotates a function whose result is of some type.

**Rejected.** A `type(...)` introducer — it is just sugar for `fn(…) -> type` and fails
the criterion; the kernel keeps only the ordinary form (memoized identically). An `abi{}`
introducer — an ABI value is an ordinary `Abi(...)` struct value (like `Package`/`Target`
in the manifest), so it *is* expressible as an ordinary call and fails identically;
conventions are `MyConv := Abi(...)`, `@abi(value)` selects one. `pred`/`attr`
introducers (a predicate `fn(K) bool`, a lever `fn(K) K`) — neither is a new literal-kind,
and the value's role is already fixed by its **consumption slot** (`when`/`comptime if`
for predicates, `@`/UFCS for levers, CT-10), so a definition-site introducer would only save
names while losing the value's role-agnosticism.

### SYN-9. `_ := expr` is the explicit discard; the lone `_` is not a name — spec#81
*provenance: refines SYN-1 (identifiers) and SYN-6 (no silent result); reads STD-7 (`print` returns
  `Result(usize, IoError)`) and MEM-2 (strict linearity) · spec: Grammar §2.2, §3.1; Functions §3.3;
  Control Flow §3; Stdlib appendix §2.7*

**Question.** A block's value is its final expression (Grammar §3.1) and a function with no result
whose body ends in a value-expression is ill-formed (Functions §3.3). Since STD-7 `print(…)` returns
`Result(usize, IoError)`, so `main := fn() { print("hello\n") }` is ill-formed, and the only way to drop
the value was an unused named binding (`r := print(…)`). No form said "discarded on purpose".

**Decision (owner, 2026-11-08).** **`_ := expr` is an explicit discard: `_` is no longer a bindable
name, may be repeated in one scope, and cannot be read.** It is the form for a procedure body ending in
`print(…)`, and covers the branching forms of the same question consistently.

**Derived (not the owner's wording).** The lone `_` is not an identifier; it stays the wildcard pattern
and is the discard's target. The discard is a block statement, not an expression, so a block ending in
it has no value; at module scope it is not in the grammar. A final `if` with `else`, or a final `match`,
whose non-diverging arms all yield a value is a value-expression, and the discard is written on it or as each arm's last
element; an `if` without `else` yields no value already (Control Flow §4). An owning value is not
discarded — `_ := e` with `e` owning leaves it neither released nor transferred, so it is ill-formed
(Memory §5.9), as STD-7 already said of a result over an owning error type.

**Why.** The discard is visible at the site (I3), has one spelling, and reuses `_` and `:=` with no new
keyword (PRIN-1). Functions §3.3's "no silent result" stays: a function that meant to return a value but
declares none still fails to compile.

**Rejected.** *Keep the rule and write a named unused binding* — a dummy name in every such procedure and
no way to say the drop is intended. *Discard the final expression of a body with no result* — brings
back the silent result §3.3 forbids.

**Left open.** The lone `_` in other binding positions — `for _ in`, a projection target, a parameter, `_ : T = e`, module scope — is refused
for now: spec#114. A final `if`/`match` whose arms are partly value-less: spec#119. The value of a
deferred expression (`defer print(…)`, spec#113) is settled by SYN-10.

### SYN-10. A deferred action with a value is ill-formed — spec#113
*provenance: follows SYN-9 (the explicit discard) and SYN-6 / Functions §3.3 (no silent result) · spec:
  Memory §5.8; Grammar §3.7; Control Flow §9.3 · owner decision 2026-11-09, spec#113*

**Question.** `defer-stmt ::= "defer" ( expr | block )`, and Memory §5.8 said nothing of the value of
`defer expr` or `defer { … expr }` when `expr` has one — `defer print("bye\n")`, with `print` returning a
`Result` since STD-7.

**Decision (owner, 2026-11-09).** **A deferred expression that has a value is ill-formed; write
`defer { _ := print(…) }` (SYN-9).** Same rule as a procedure body ending in a value (Functions §3.3).

**Derived (not the owner's wording).** A block is an expression whose value is its final element
(Grammar §3.1), so `defer { …; print(…) }` is the same case; a block ending in a discard has no value.

**Why.** Nothing receives the value, so dropping it is written (I3); a forgotten `Result` in cleanup does
not go unseen, and the rule is the one procedure bodies already follow.

**Rejected.** *A deferred expression is evaluated for effect and its value discarded* — a silent discard,
the one place besides a non-final statement where a value disappears unseen.

---

## `@` attributes

### SYN-7. `@` means exactly one thing — a prefix attribute on any markable entity
*spec: Grammar §3.11*

**Why.** Attributes are spelled `@name` / `@name(args)`, always as a **single prefix**
before what they modify (one placement rule, TYP-7). The target is **any markable entity**,
not just a binding/field: a binding (`@reg x`), a field/type (`@align(8)`, `@packed
struct{…}`), a function (`@abi(c)`), a file (`@limits(…)`), or a code construct
(`@label(outer) loop {…}`). Crucially, **`@` means exactly one thing — an attribute**.
Intrinsics carry no `@` marker: `typeinfo`/`bitcast`/`T.size()`/`compiles`/`embed`/… are
ordinary prelude identifiers (OP-1). The Zig-style of `@`-intrinsics is rejected because it
would overload `@` with two meanings.

**Rejected.** A dotted `@family.name` spelling. Attributes are organized into fixed
*families* by what they affect — storage, layout, contract, abi, codegen, linkage, build,
control, test — but that family structure is **documentation and diagnostic structure, not
syntax**. The spelling stays flat (`@name`), and the objection that ~20 attributes share
one flat namespace is answered by the **diagnostic**, not a namespace: an unknown or
mis-targeted `@name` is reported *by its family* ("unknown layout attribute `@foo`",
"`@reg` is a storage attribute, not valid on a type"), so the family is discoverable at
zero syntactic cost — a dotted form would add churn and verbosity for a cosmetic gain.
