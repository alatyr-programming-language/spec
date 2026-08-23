# Alatyr Language Specification

## Chapter — Control Flow

> **Status — draft (under review).** This chapter is a draft and is not yet
> accepted; the specification as a whole is under development. See the status note
> at the top of the Overview chapter.

This chapter defines control flow at **two levels** — the raw assembly level (labels
and jumps) and the structured level (`if` / `match` / loops / `break` / `continue` /
`return`) — how structured forms **lower** to labels and jumps, **exhaustiveness**,
the **safety analyses** (linearity, definite assignment), the **short-circuit logical
operators**, and the **`?` try operator**. It builds on the Memory chapter
(`defer`, linearity §5), the Type-System chapter (`bool`, tagged enums), the
Declarations chapter (the `@label` attribute), the Functions
chapter (`return` / `out`, naked functions), and the Comptime chapter (`comptime if`
/ `comptime for`,
the truthy/tryable protocols).

Requirement keywords (**MUST**, **MUST NOT**, **SHOULD**, **MAY**) follow RFC 2119
(Overview §6). This chapter gives the **concrete syntactic forms** it introduces as
grammar fragments (§11); the *consolidated* EBNF is assembled in the Grammar chapter.

---

### 1. Two levels in one language

Raw (assembly-level) control flow and structured control flow belong to the **one**
language (Overview §3) and **may be freely mixed** within a unit. The structured
level **lowers to the raw level** by documented rules (§10; I1). The raw level is
unstructured (goto-like) and **bypasses** the safety analyses (§9); the structured
level is **analyzed** for linearity, definite assignment, and exhaustiveness.

---

### 2. Raw control flow

#### 2.1 Labels

A **label** names a code location with the **`@label(name)`** attribute (placement
per the attribute rule — the attribute precedes what it names; decisions SYN-7/CF-4). It
is **not** a value-introducer: naming a location is **orthogonal** to value binding,
so a labeled loop can still yield a value (§6, §7.2). There are **two kinds** with
different semantics:

> **Backend scope (CG-4):** the **structured** kind below is portable across all backends; the
> **code-point** kind and raw transfer are a **target-capability** (register ISAs only) — on a
> structured/VM backend (e.g. WASM: no raw goto, no code addresses) they are a compile error.

**Code-point label** — `@label(name) <instruction>` (named, not GAS-numeric). It is a
`jmp` target, and its **name in value position yields a raw code address** — the
target's **pointer-width raw block** (`bits64` on a 64-bit target; the kernel `bitsN`,
Type System §2 — `bitsN` is the family meta-notation, source names the concrete width;
there is **no** dedicated code-pointer type, Type System §7). It is for `jmp(name)` and
computed-goto **jump tables** (`tbl : [bits64; N] = [l1, l2, l3]` on a 64-bit target),
storable and copyable. It is a **raw code address, not a function pointer**: a `jmp`
target only, **not callable** (it has no ABI/frame entry contract — a `call` target is
a function value, Functions §1.2), and a jump through it is **raw** (it bypasses the
safety analyses, §9.2), meaningful only **within its own function's activation**. Being
a `bitsN` it is **not opaque** — it carries the ordinary raw-block operations (Type
System §2.2).

**`unchecked` requirement (the transfer, not the address).** Producing a label's
address and storing/copying it (building a jump table) is **plain pointer-width
`bitsN` data — not** unchecked. What requires an `unchecked` grant is the **raw
control transfer itself** — see §2.3.

**Structured label** — `@label(name) loop { … }` / `while … { … }` / `for … { … }` /
`{ … }`. It is the target of `break name` (loop or block) and `continue name` (loop
only, §7). It is a **structural target, not a value**: it has **no address**, and
using its name in value position or as a `jmp` operand is **ill-formed** — a
structured construct has no single address (entry, header, condition, back-edge, and
exit differ), and a jump into its middle would break its setup and `defer` handling.

*Why a code name needs no address-of:* a code entity (a function or a code-point
label) *is* its address when named, because code has no "value to load" — unlike a
**data place**, whose name yields the loaded value, so its address needs `ptr`
(Memory §4.3; decisions CF-5).

**Namespace and scope (labels) — function-scoped (C-style).** All labels (code-point
and structured) are **function-scoped**: one flat label namespace per function. A
label name is **unique within its function** and **visible throughout it** (forward
*and* backward `jmp`); there is **no block-level shadowing of labels** (unlike
ordinary block-scoped bindings, Declarations §6–§7 — this is what removes the earlier
"visible-everywhere vs cross-scope-shadowing" conflict). Labels share the name
namespace with bindings: a label name MUST NOT coincide with a binding name in the
same function, so `name` in value position and `break`/`continue name` resolve
unambiguously. Resolution is **not** order-dependent (forward `jmp` works). `break
name` / `continue name` additionally require `name` to be a **lexically enclosing**
loop/block (the name resolves function-flat, but the target must enclose the
statement; §7.1).

#### 2.2 Jumps

`jmp` and conditional branches are **instructions** taking a label operand (Assembly
chapter); raw inline assembly is available everywhere (Overview §3). There is **no**
`snippet` construct (removed): inline raw assembly serves everywhere, and a named
raw region — such as an interrupt handler, or a freestanding entry point (`Os.none`,
where no platform contract exists; ABI appendix §3.3) — is a **naked function**
(`@abi(naked)`, Functions §6.4; decisions CF-5).

#### 2.3 Raw control transfer bypasses safety — and requires `unchecked`

A **raw control transfer** — `jmp` or a conditional branch, **direct** (`jmp(label)`)
or **indirect** (`jmp(table[i])`, through a raw code address) — creates a control-flow
edge that the structured safety analyses (§9) do **not** model: it can skip an
initialization, re-enter past a linear release, or leave an `out` unwritten. Its
behavior is **hardware-defined**, never C-style UB (I11) — the compiler does not
miscompile surrounding code on the assumption the jump did not occur — but the
guarantee of §9 holds only for the structured edges; the raw edge is the
**programmer's responsibility**.

Because such a transfer silently bypasses the §9 analyses, it requires an **`unchecked`
grant** in structured code (lexically visible, decisions CG-6) — so the bypass is never
invisible. Direct and indirect raw transfers are treated **identically** in this
respect. (Straight-line raw instructions such as `movq` do **not** bypass §9 and need
no grant; CG-2.) Under the **`no_abstractions`** limit there is no structured control
flow and the §9 analyses do not apply, so raw transfer is the unit's **native** surface
and needs no per-site grant.

---

### 3. Blocks and expressions

A block `{ … }` is an **expression**: its value is its **final expression**
(decisions SYN-6); a block used in statement position simply
discards that value. Statements within a block are separated by a **newline or `;`**
(§11; decisions SYN-4). The structured forms `if`, `match`, and `loop` are
**expressions** (value-producing) and are equally usable in statement position.

**Result typing.** A block's value and type follow its **final element**:

| Final element | Block value / type |
|---|---|
| an **expression** `e` | the value and type of `e` |
| a **declaration / assignment / `while` / `for` / `defer`** | **no value** — there is no unit type (Grammar §3.1); using the block for a value is ill-formed |
| a **diverging** statement (`return` / `break` / `continue`, or a call of type **`Never`** — `panic`/`exit`, Stdlib appendix §3.7) | the block **diverges**: its type is **`Never`** and code after it is unreachable (§9) |

For the structured expressions: an `if`/`match` consumed for a value takes the
**common type of its non-diverging arms** — an arm that diverges (type `Never`) imposes
no constraint; if **every** arm diverges, the construct's type is **`Never`**. A
**`loop` with no reachable `break`** likewise has type **`Never`** (it never yields).
A `loop` consumed for a value takes the **common type of all its reachable
`break`-with-value exits** (§7.2) — a diverging path imposes no constraint, exactly as
for `if`/`match` arms; break-values of incompatible type are **ill-formed**.
(`Never` is the bottom type, Stdlib appendix §3.7; it unifies with any type, so a diverging arm
fits any branch.)

---

### 4. `if` / `else`

`if cond { … } else { … }`:

- the condition `cond` is a **truthy** value (`bool`, or a type satisfying the truthy
  protocol; Comptime §6.2) — **no parentheses** around it (decisions CF-1) — and each
  body is **braced**. A **numeric scalar is not truthy** (it provides no `is_true`,
  Stdlib §2.1): a numeric condition uses an explicit comparison yielding `bool`
  (`if n != 0 { … }`), never a bare `if n { … }`;
- as an **expression**, the branches MUST have a **compatible type** and the `if`
  yields that type; as a **statement**, the branches may be value-less;
- `else if` chains (`else` followed by another `if`).

A missing `else` is permitted only in **statement position** (the `if` produces no
value and is not consumed for one). An `if` **consumed for a value** MUST have an
`else`, so that every path produces a value of the common type.

---

### 5. `match`

`match v { pattern => body … }` matches a value of a **tagged enum** (Type System
§6.2) or a **scalar**.

#### 5.1 Exhaustiveness

A `match` MUST be **exhaustive**: every variant (or scalar case) is covered, or a
`_` default is present. A non-exhaustive `match` is a **compile error** — there is no
fall-through to undefined behavior (I11).

#### 5.2 Patterns

Patterns are a **shared mechanism** with destructuring binding (decisions FN-4):

- **variant + payload binding** — `Some(v)`, `Ok(x)`;
- **literal** — `0`, `'a'`;
- **range** — `1..10` (half-open) or `1..=10` (inclusive), §5.4;
- **destructuring** — `Point(x = px, y = py)`;
- **wildcard** — `_`;
- **OR-pattern** — `1 | 2 | 3`, alternatives sharing one arm body (§5.4).

A variant pattern names the variant **bare** (`Some`, `Red` — the scrutinee's enum
type supplies the context) and may **qualify** it either as a `::`-path (`Color::Red`)
**or** with the constructor `.`-spelling (`Color.Red`, `Option.Some(v)`) — a pattern
may mirror exactly how the variant is built (Grammar §3.5). All three forms denote
the same variant.

An **OR-pattern** writes several alternatives separated by `|` ahead of one `=>`
(`p1 | p2 | … => body`): the arm matches when **any** alternative matches, and is
**exactly equivalent** to repeating that arm once per alternative, in order, with the
same body (Grammar §3.7). It is pure surface sugar — it adds no new pattern kind, and
first-match-wins (§5.3), exhaustiveness (§5.1/§5.4), and the unreachable-arm error all
apply to the expansion unchanged. Alternatives are ordinary patterns of the same kinds
above; v1 alternatives do **not** bind (an alternative that introduces a binding is
ill-formed, since the alternatives would bind inconsistently).

#### 5.3 Arms and match order

Arms are separated by a **newline or `;`** (decisions SYN-4); each arm is
`pattern => ( expr | block )`. `match` is an **expression**: the arm bodies MUST have
a compatible type. Arms are tested **top-to-bottom and the first matching arm wins**;
there is **no implicit fall-through** — after the chosen arm runs, control leaves the
`match`. An arm that is **wholly unreachable** (every value it would match is already
covered by earlier arms) is a **compile error** (dead code).

#### 5.4 Scalar matches and ranges

For a `match` over a **scalar**:

- the scrutinee's type MUST be an **integer** (`uN` / `iN`), **`char`**, or **`bool`**.
  **Floating-point** scrutinees are **rejected** (a compile error): equality and range
  tests on IEEE-754 values are error-prone and excluded.
- A **range pattern** uses comptime-constant endpoints of the scalar's type:
  **`a..b`** is **half-open** — it matches `a ≤ x < b` — and **`a..=b`** is
  **inclusive** — `a ≤ x ≤ b`. Both `..` forms have the **same meaning** here as in
  iteration (Loops §6), so the operator does not change sense between contexts. The
  inclusive form `..=` is what covers a type up to its **maximum** (e.g. `0..=255` for
  a `u8`, which `0..256` cannot express).
- **Exhaustiveness** (§5.1) for a scalar is checked over the type's **finite value
  range** `[min, max]`: the union of the arms' literals and ranges MUST cover every
  value in `[min, max]`, or a `_` arm MUST be present. Coverage is computed by the
  compiler; a gap with no `_` is a compile error.
- Overlapping arms are permitted (first-match-wins, §5.3); a fully-shadowed arm is the
  unreachable-arm error of §5.3.

---

### 6. Loops

- **`loop { … }`** — an unconditional loop; **`break value`** turns it into a
  **loop-expression** yielding `value` (§7.2).
- **`while cond { … }`** — `cond` truthy (§4), no parentheses, braced body.
- **`for x in iter { … }`** — iterates a **structural iterator protocol** (a comptime
  shape — the general protocol mechanism is Comptime §6; the `Iterator` shape itself is
  defined in the Stdlib appendix §2.4); ranges and arrays satisfy it from the
  prelude (Type System §7). A **range** is written `a..b` (**half-open**, `a ≤ x < b`) or `a..=b`
  (**inclusive**, `a ≤ x ≤ b`) — a prelude `Range` value; the same two forms are the
  range **patterns** of §5.4, so `..` does not change meaning between iteration and
  matching.

Any loop may be **labeled** with `@label(name)` (§2.1) to be the target of
`break name` / `continue name` from nested loops; its value (from `break name value`,
§7.2) flows to the binding of the loop expression — the **label** and the **value
binding** are orthogonal (`total := @label(outer) loop { … }`).

Compile-time loops (`comptime for` unrolling, `comptime if` selection) are defined in
the Comptime chapter §8 and are **not** runtime loops.

---

### 7. `break` / `continue` / `return`

#### 7.1 Targets

Unlabeled `break` and `continue` target the **nearest enclosing loop**.

`break name` targets a **labeled loop or a labeled block** (`@label(name) …`, §2.1)
and **exits** it (a block-`break` may carry a value, §7.2).

`continue name` targets a **labeled loop only** — it resumes that loop's next
iteration. A `continue` whose target is a **non-loop** labeled block is
**ill-formed**: a block has no back-edge or next iteration. (Unlabeled `continue`
inside a non-loop labeled block targets the nearest enclosing **loop**, skipping the
block.)

There is **no** `:outer` label syntax; the target is the named label.

The two label kinds (§2.1), the structured one shown by its loop and block forms, differ as targets:

| Label kind | In value position | `break name` | `break name value` | `continue name` |
|---|---|---|---|---|
| **code-point** `@label(n) <instr>` | **is a value** — raw code address (`bitsN`) | — (not a target) | — | — |
| **structured loop** `@label(n) loop`/`while`/`for` | **no value, no address** | yes | yes for `loop`; `while`/`for` are statement-only | yes (loop only) |
| **structured block** `@label(n) { … }` | **no value, no address** | yes | yes | ill-formed (no next iteration) |

**Target-vs-value disambiguation.** Since a code-point label is a **value** but never
a `break`/`continue` target, and a structured label is a **target** but never a value
(§2.1), the two roles do **not** overlap. The rule: an identifier immediately after
`break` / `continue` that resolves to an **in-scope structured label** is the
**target**; otherwise the operand is a **value** expression. Thus `break outer`
(structured `outer`) breaks to `outer`; `break l1` (code-point `l1`) breaks the
nearest loop **with value** `l1` (its raw code address); `break 42` breaks with value `42`;
`break outer 42` breaks to `outer` with `42`.

#### 7.2 `break` with a value

`break value` yields `value` from the enclosing `loop` expression; `break name value`
yields it from the named target when that target is a value-bearing `loop` expression
or labeled block (§7.1 disambiguation). A `while`/`for` target is statement-only, so a
value-bearing `break` to it is ill-formed. `continue` carries no value.

#### 7.3 `return`

`return` exits the function (Functions §3.4): a bare `return` (no value — or with
named/multiple `out`s written in the body), or `return expr` for a **single anonymous
projected output** (`-> T`). `defer` actions run on the `return` path (§9.3).

---

### 8. Control-flow operators

Two operators are **control flow**, not value-functions — they are **special syntax**
(keywords / a postfix form), because they affect *which* code runs, and this chapter
is their normative home. (The eager value-operators — bitwise, comparison, arithmetic
— are overloadable functions, Type System §10.)

#### 8.1 Logical `and` / `or` / `not`

`and` / `or` are **short-circuiting**: the right operand is evaluated **only when
needed** (`and` stops at the first false, `or` at the first true); `not` is prefix
logical negation. Their laziness is **visible** (I3). Operands are truthy (§4). The
words-not-symbols choice (decisions OP-2) removes the `&`/`&&` footgun: the bitwise
`& | ^ ~` are the eager functions, the words are the lazy control operators.

#### 8.2 The `?` try operator

`x?` is a **postfix** operator on an `Option` / `Result` — or any type satisfying the
**tryable** protocol (Comptime §6.2):

- on **success** (`Some v` / `Ok v`) it evaluates to the unwrapped `v` and execution
  continues;
- on **failure** (`None` / `Err e`) it performs an **early return** from the enclosing
  function with that failure, running `defer` actions on the way out (§9.3).

`?` is sugar for a `match` + early return; there are **no** exceptions and **no**
stack unwinding. It is **zero-cost** (a branch + the normal return path) and
**visible** (I3). The enclosing function's **result** (`-> T` or named `out`) MUST be a
**compatible tryable type**, else the `?` is ill-formed (Functions §3); the failure is
converted into the result's failure type by a **declared** conversion `OutErr(failure)` (the `T(v)` conversion lattice,
Type System §4), applied by `?` — matching failure types pass directly, a declared conversion is applied,
otherwise a compile error. This is **declared and visible** (I8/I3), not a hidden `From`; under
`no_abstractions` auto-application is **off** (exact match or an on-site conversion `OutErr(x)?` required).
The converted failure is then **wrapped** into the enclosing tryable: for `Option`/`Result` by their
built-in failure case (`None`/`Err`), for a **user** tryable by its **`from_failure`** constructor (the
fourth `Tryable` operation, CF-10 / Stdlib §2.2) — except where the enclosing type **is** the operand's
own type, in which case the operand is returned verbatim.

---

### 9. Safety analyses (structured level)

Flow analysis over the **structured** control-flow graph enforces, before run time:

- **Linearity** (Memory §5.9): on **every normal exit path**, each owning value is
  consumed (released or transferred) **exactly once** — double-release,
  use-after-release, and leaks are compile errors;
- **Definite assignment** (Type System §9.4): a binding (and every `out` parameter) is
  **definitely assigned at a point iff it is assigned on every control-flow path that
  reaches that point**; for `if`/`else` and `match` that means assigned in **every**
  arm, where an arm that **diverges** (ends in a trap / `return` / `Never`) contributes
  no path and is excluded. The analysis is **field-sensitive** (tracked per field;
  partial init is permitted, moves are whole-value — Type System §9.4, Memory §5.9).
  Reading possibly-unassigned storage is forbidden (I11);
- **Exhaustiveness** (§5.1): every `match` is exhaustive.

#### 9.1 Normal exit paths

A "normal exit path" includes fall-through, `break`, `continue`, `return`, and the
`?` early-exit (§8.2). The analyses cover all of them.

#### 9.2 Raw edges are excluded

Raw control transfers (§2.3) are **outside** the structured control-flow graph and are
**not** analyzed: along a raw edge the programmer guarantees linearity and definite
assignment, and behavior is hardware-defined (I11). The structured guarantees hold for
the structured edges; a raw transfer into or out of a structured region voids them
only along that edge. Because such a transfer silently bypasses these analyses, in
structured code it requires an **`unchecked`** grant (§2.3) — making the bypass
lexically visible (CG-6); under `no_abstractions` the analyses do not apply and it is
native.

#### 9.3 `defer`

`defer` actions run **LIFO** on every **normal** exit of their scope — fall-through,
`break`, `continue`, `return`, `?`-early-exit — and so discharge linear obligations
on all of them (Memory §5.8). They do **not** run on abort / trap / panic (the
process terminates; no unwinding).

---

### 10. Lowering

Each structured form lowers to labels + jumps by a documented rule (I1), so the
emitted branch structure is predictable:

- `if` → a conditional branch over the two block bodies;
- `match` → a documented dispatch (a branch chain or a jump table on the tag/scalar);
- `loop` / `while` / `for` → a loop label with a back-edge and an exit branch;
  `break` / `continue` → jumps to the loop's exit / back-edge label (or the named
  label's, §7.1); `break value` writes the result place first only for a
  value-bearing `loop` expression or labeled block;
- `?` → a branch on success/failure with the failure path taking the function's
  normal return route (running `defer`s);
- `and` / `or` → a short-circuit branch (the right operand's code is reached only on
  the non-deciding outcome).

The raw level is already labels + jumps (1:1; Assembly chapter). No structured form
introduces hidden control flow beyond its documented lowering (I3).

---

### 11. Syntax (grammar fragments)

The forms this chapter introduces are given here as **grammar fragments**; the
**consolidated EBNF** is assembled in the Grammar chapter. Nonterminals not defined
here (`ident`, `expr`, `block`, `pattern`, `type-expr`) come from the Declarations,
Type-System, and Grammar chapters.

```ebnf
(* labels — the @label(name) attribute precedes what it names (§2.1; CF-4) *)
label-attr    ::= "@label" "(" ident ")"
labeled-point ::= label-attr postfix-expr              (* postfix-expr is an instruction/intrinsic call — no `instruction` production (§11; Grammar §3.10); code-point: jmp(name); name in value pos = raw code address *)
labeled-loop-expr ::= label-attr loop-expr             (* structured: break/continue name; value-bearing loop *)
labeled-loop-stmt ::= label-attr ( while-stmt | for-stmt )  (* structured: break/continue name; NOT a value *)
labeled-block ::= label-attr block                     (* structured: break name only (§7.1); block expression remains value-bearing *)
(* only a code-point label's name is a value (a raw code address, for jump tables); structured *)
(* label names are break/continue targets only — no address, not usable in value position (§2.1)   *)

(* structured (§4–§7) — conditions unparenthesized, bodies braced *)
if-expr       ::= "if" expr block [ "else" ( if-expr | block ) ]
match-expr    ::= "match" expr "{" match-arm { (";" | newline) match-arm } "}"
match-arm     ::= pattern { "|" pattern } "=>" ( expr | block )   (* OR-pattern: alternatives share one body (§5.4) *)
loop-expr     ::= "loop" block
while-stmt    ::= "while" expr block
for-stmt      ::= "for" ident "in" expr block
break-stmt    ::= "break" [ struct-label ] [ expr ]   (* struct-label resolved semantically, §7.1 *)
continue-stmt ::= "continue" [ struct-label ]         (* struct-label must be a loop (§7.1) *)
(* struct-label = a leading identifier that RESOLVES to an in-scope structured label (§7.1); if a leading *)
(* identifier is not a structured label (e.g. a code-point label or any value), it is part of `expr`.     *)
return-stmt   ::= "return" [ expr ]                    (* expr only with a single anonymous projected output (§7.3) *)

(* control-flow operators (§8) *)
logical-expr  ::= expr ( "and" | "or" ) expr | "not" expr   (* short-circuit *)
try-expr      ::= expr "?"                                  (* postfix unwrap-or-propagate *)

(* ranges — used both as iterators (§6) and as scalar match patterns (§5.4) *)
range-expr    ::= [ expr ] ".." [ expr ] | [ expr ] "..=" expr   (* half-open [a,b) | inclusive [a,b]; in index position an omitted lower bound = 0 and an omitted upper = the base's length (`a[lo..]`/`a[..hi]`/`a[..]`, §5.4) *)
```

Normative notes:

- Conditions are **unparenthesized**; every body is a **brace block**; a block is an
  expression whose value is its final expression (§3).
- `match` arms are separated by newline or `;`, are tested first-match-wins, must be
  **exhaustive** (§5.1), and share the destructuring **pattern** grammar (Type System
  / Grammar). Scalar scrutinees are integer / `char` / `bool` only (not float), and a
  range pattern is `a..b` (half-open) or `a..=b` (inclusive) over comptime endpoints
  (§5.4).
- An **open-ended** upper bound `a[lo..]` (slice/index position only) means *to the
  end* — the upper bound is the indexed base's **own length** (a `[T; N]` array's
  constant `N`, a slice / `str` / `Vec`'s `len`), so the count is never hand-written
  (`keywords[0..]` slices the whole array; no magic number to drift out of sync). It is
  **half-open** by nature; an iteration `for x in lo..` (no bound) and a match pattern
  with an open bound are ill-formed (a range pattern / iterator needs both ends).
- An **open-ended** lower bound `a[..hi]`, and the fully-open `a[..]`, are the dual: an
  omitted lower bound is **0** (`a[..hi]` ≡ `a[0..hi]`, `a[..]` ≡ `a[0..]` — to the end).
  Both are **index position only** (a standalone range still needs its lower bound);
  half-open like `a[lo..hi]`.
- `..` / `..=` build the same **range** in iteration (§6) and in patterns (§5.4) —
  half-open and inclusive respectively.
- a label is the `@label(name)` attribute (§2.1), **not** an introducer. A
  **code-point** label's name in value position is a **raw code address** (jump
  tables; not a callable function pointer; raw `jmp` use, §9.2). A **structured**
  (loop/block) label is a `break`/`continue` target **only** — not a value, no address.
  `break name` targets a loop or block, `continue name` a **loop only** (§7.1);
  `break` with an `expr` yields a value (§7.2). Labels share the ordinary namespace,
  are function-wide and forward-referenceable (§2.1).
- `and` / `or` / `not` are **keywords** (lazy control flow), distinct from the eager
  bitwise functions `& | ^ ~` (Type System §10); `?` is a postfix control operator,
  not a builtin function.

---

### 12. Conformance (normative summary)

A conforming implementation MUST:

1. provide both a raw level (`@label(name)` code-point and structured labels, jumps;
   a code-point label's name in value position is a raw code address — not a callable
   function pointer — while a structured loop/block label is a break/continue target
   only, with no address; labels share the namespace and are function-wide /
   forward-referenceable) and a structured level in one language, lowering structured
   forms to labels + jumps by documented rules (§1, §2, §10; I1);
2. exclude raw control transfers from the safety analyses (their hazards are
   hardware-defined and the programmer's responsibility, no miscompilation of
   surrounding code), and **require an `unchecked` grant** for a raw transfer (direct
   or indirect) in structured code — native under `no_abstractions` (§2.3, §9.2;
   CG-6/I11);
3. treat blocks, `if`, `match`, and `loop` as expressions whose value is well-defined,
   require compatible branch/arm types where a value is consumed, and use
   unparenthesized conditions with braced bodies (§3–§6);
4. test `match` arms first-match-wins with no implicit fall-through, reject a
   non-exhaustive `match` and a wholly-unreachable arm, restrict scalar scrutinees to
   integer/`char`/`bool` (not float), interpret `a..b` half-open and `a..=b`
   inclusive, and verify scalar coverage over the type's `[min, max]` (§5; I11);
5. support `loop` (+ `break value`), `while`, and `for` over the structural iterator
   protocol; target unlabeled `break`/`continue` at the nearest loop, `break name` at
   a named loop or block, and `continue name` at a named **loop only** (rejecting
   `continue` to a non-loop labeled block); and permit `return` / `return expr` per
   the function's `out` (§6–§7);
6. implement `and` / `or` / `not` as short-circuiting control keywords (distinct from
   the eager bitwise functions) and `?` as a postfix early-return over the tryable
   protocol with a **declared** failure conversion (`OutErr(err)`, off under `no_abstractions`) and a
   tryable-compatible enclosing `out` (§8; I3/I8);
7. enforce, over the structured control-flow graph, linearity (each owning value
   consumed exactly once per normal exit), definite assignment (every out/binding
   written before read), and match exhaustiveness, with `defer` running LIFO on every
   normal exit including `break`/`continue`/`return`/`?` and not on abort (§9; I11);
8. introduce **no** hidden control flow beyond the documented lowering (§10; I3).
