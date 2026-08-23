# Decisions — Control flow, labels, error handling (`?`/`!`, Tryable)

**Why the language is the way it is** for control flow. Each entry records the *rationale*
and the *rejected alternatives*; the normative *what* lives in the spec (linked per entry,
Control Flow chapter `60`), the *how* in the compiler. IDs are theme-prefixed (`CF-N`) and
**append-only** — assigned in articulation order, never renumbered. The `provenance:` line
records what an entry refines and where it lands in `spec/`.

---

## Control flow

### CF-1. Two levels in one language — raw jumps under structured constructs
*spec: Control Flow §1, §2, §10*

**Why.** Control flow is **two-level** so the language can keep its assembler-equal
expressiveness ceiling (I1) without giving up structured ergonomics. The
**assembly-correspondence surface** is raw and unstructured: named labels, jumps
(`jmp`/conditional branches as instructions with a label operand), raw inline asm —
available everywhere. Raw jumps **bypass** the safety analyses; the programmer is
responsible, hardware-defined rather than C-UB (consistent with I11). The **structured
surface** (`if`/`match`/loops/`break`/`continue`/`return`) is then defined as ordinary
sugar that **lowers into labels + jumps by documented rules** (I1) — it is not a separate
machine, just the readable spelling of the same primitives. Conditions are written without
parentheses (`if cond { … }`), bodies always in braces, and a block is itself an
expression (its value = its final expression); `if` and `match` are value-expressions, not
just statements, so control flow composes as values.

### CF-2. `match` is exhaustive; loop/branch values share one common-type rule
*spec: Control Flow §4, §5, §6, §7*

**Why.** `match` over a tagged enum or scalars is **exhaustive** — all variants or a
`_`-default — because an unhandled case would be a path with no defined behavior, i.e. UB,
which I11 forbids. Pattern matching is the **one common mechanism** shared with binding
destructuring (one machinery, not two): variant+payload, literal, range, `_`,
destructuring. The value of a value-yielding control construct follows a **single rule**
across the family: an `if`/`match` expression's type is the common type of its arms, and a
`loop` becomes an expression whose type is the **common type of all reachable
`break`-with-value exits** (no reachable break → `Never`). `for x in <iterator>` ranges
over the **structural iterator protocol** from the prelude — a protocol form, not a
hardwired set of types — so iteration is library-extensible.

### CF-3. Structured constructs are checked: definite assignment, linearity, no UB
*spec: Control Flow §9*

**Why.** The structured level carries the safety the raw level deliberately drops, so that
ordinary code is correct-by-construction (I11) without paying for it where it is bypassed
(CF-1). Flow analysis ensures **definite assignment** of outputs (every `out`/binding is
written on all normal-exit paths, TYP-8) and **linearity** (each path consumes owning values
exactly once, MEM-2); `match` exhaustiveness closes the last hole (no undefined path). These
analyses see only the structured edges; raw transfers are excluded from them by design
(CF-5), which is precisely why a raw transfer needs an explicit grant.

---

## Labels

### CF-4. Labels are an attribute `@label(name)`, not a value-introducer
*spec: Control Flow §2.1*

**Why.** Naming a code location is **orthogonal to producing a value**, so it is an
**attribute** (`@label(name)`, the `@`-side = representation/identity, TYP-7) prefixing the
labeled thing — not a value-introducer. The earlier design made `label` a value-introducer
(`name := label { … }`), which (a) overloaded the `name :=` slot, which should be only
about values, and (b) left a hole: a labeled loop could not also yield a value. Making the
label an attribute removes both problems at the root — the value of a labeled loop flows
into an ordinary binding (`total := @label(outer) loop {…}`) because the `name :=` slot is
now free. The family of introducers consequently **loses `label`** (leaving
`struct/enum/union/fn/mod`, all of which produce a value).

**Rejected.** The `label` keyword/introducer (overloads `name :=`; blocks the
labeled-loop-as-value case). Also: a module-level named raw region is now a
**naked-function** (`@abi(naked)`), not a label-block (a revision of FN-3/CF-4), keeping one
rule — a module element is a declaration `name := value`.

### CF-5. Two kinds of label — code-point vs structured — with disjoint roles
*spec: Control Flow §2.1, §2.2, §2.3, §7.1*

**Why.** A code location is needed in two genuinely different senses, and conflating them
would create ambiguity, so they are kept as two kinds with **non-overlapping** roles. A
**code-point** label (`@label(retry) <instruction>`) is a raw code-point: its name in a
value-position is a **raw code-address** (pointer-width) usable for `jmp` and jump-tables,
storable, but **not** a function-pointer (no frame/ABI, not callable) — jumping to it is
raw, meaningful only within its own activation (hardware-defined responsibility, I11). A
**structured** label (`@label(outer) loop/while/for/{block}`) is the target **only** of
`break`/`continue`; it is **not a value, has no address, is not a `jmp` target** — a
structured construct has no single "address" (entry/header/cond/back-edge/exit differ, and
a jump into its middle would skip setup/`defer`). Because the roles are disjoint
(structured = target-not-value, code-point = value-not-target), `break`/`continue`
disambiguation is unambiguous: an identifier resolving to a structured-label in scope is a
**target**, otherwise the operand is a **value**. Labels are **function-scoped** (C-style):
one flat namespace per function, unique within and visible throughout it
(forward+backward `jmp`), **no block-shadowing** — which resolves the "visible everywhere
vs cross-scope shadow" contradiction and lets `break/continue name` resolve flatly while
still requiring a lexically enclosing target. The *code-vs-data* principle ties this back
to memory: a code name in value-position **is** its address (code has no value to load, so
no address-of operation), whereas a data place needs `ptr` for its address (MEM-7).

**Rejected.** A separate code-address **type** — a code-address is just a pointer-width
raw `bitsN` block (no data dereference exists for code, so a rich type like `ptr(T)`
is unneeded); the honest price is that it carries ordinary bitsN-operations.

### CF-6. `unchecked` gates the raw transfer, not the address
*spec: Control Flow §2.3, §9.2*

**Why.** What the safety analyses (§9) cannot see is the **edge** a raw jump creates, not
the bits of a label's address — so the grant is required at the **transfer**, not at the
address. A direct `jmp(label)` or indirect `jmp(table[i])` both create an unseen control
edge and therefore both require an `unchecked` grant in structured code; taking and
storing a code-address (a jump-table `[bits64; N]`) is **ordinary data** (just bits) and
needs no grant. Straight-line raw instructions (`movq`) do not bypass §9 and need no grant
(CG-2). Under `no_abstractions` there is no structured layer/analyses, so a raw transfer
is native without a grant. *Backend scope (CG-4):* the **structured** label kind is
**portable** across all backends; the **code-point** kind and raw transfer are a
target-capability (register ISAs only) and a compile error on a structured/VM backend
(WASM has no raw goto / code addresses). Relatedly, `snippet` was **removed** as redundant
(FND-10): inline raw asm is available everywhere, a frame/ABI-less function is `@abi(naked)`,
and a named raw region is a naked-function — one fewer entity.

---

## Error handling (`?` / `!`, Tryable)

### CF-7. Glyph distribution — `?` is control-flow syntax, `!` gains no new role
*spec: Control Flow §8*

**Why.** Glyphs are distributed by a criterion (continuing OP-2, bitwise-with-symbols /
logical-control-with-words): **a symbol for a value-operation or comparison; a word/named
builtin for semantics or loss; special syntax for control flow.** `?` is control flow, so
it is **special syntax, not a builtin** (like `and`/`or`/`not` are words). `!` correspondingly
gets **no new role** — only the token `!=`.

**Rejected.** A unary logical "not" via `!` (there is `not`, OP-2); a force-unwrap `!`/`!!`
(unwrapping with a trap is the **named** builtin `.unwrap()`/`.expect(...)`, UFCS — explicit
loss, I8); macros (none — comptime instead); `never`/`noreturn` as a symbol (it is the type
`Never`, a name, STD-1 — clarity over terseness; no separate `@noreturn` attribute, OP-1).

### CF-8. `?` — visible, zero-cost try/propagation over the *tryable* protocol
*spec: Control Flow §8.2*

**Why.** `x?` yields the success value and continues, or **early-exits** the enclosing
function with the failure (running `defer`s along the path, MEM-1). It is **visible sugar over
`match` + early-return** — neither exceptions nor stack unwinding — therefore zero-cost (I2)
and visible (I3), which is exactly why it qualifies as control syntax. Crucially, `?`
dispatches over the **comptime `tryable` protocol (a form), not over the names
`Option`/`Result`**: the language fixes the *structure* (ask success/failure, extract the
success value, assemble the early-exit value); `Option`/`Result` merely **implement** it in
the prelude (STD-1), not privileged. This is the same principle by which `if`/`while`/`and`/
`or` already depend on the *form* of `bool` (a prelude type), not its name — a condition
needs a *truthy* form, `?` needs a *tryable* form. Consequence: user-defined error types and
freestanding/custom-prelude all get `?`, resolved in comptime + monomorphization → zero-cost,
no RTTI (I7/I2). The failure is converted into the result's failure type by a **declared,
visible** conversion (`OutErr`, reusing the `T(v)` lattice) — "explicit" means declared and
visible (I8/I3), **not** a hidden `From` inferred from a registry; under `no_abstractions`
auto-application is off (exact match or an on-site `OutErr(x)?`).

**Rejected.** Hardcoding the names `Option`/`Result` (Rust-style lang-items — magic, breaks
freestanding/custom types); building the error type into the language (the Zig path — against
"Option is built with the same machinery", TYP-5).

### CF-9. No `T?` Option-sugar — one glyph, one meaning
*spec: Control Flow §8.2*

**Why.** A `T?` type-sugar is rejected **on principle**, not deferred. The `?` *operator*
applies to Option **and** Result (and any tryable), whereas `T?` would mean Option
specifically — one glyph would be general in value-position but Option-privileging in
type-position, with no analog for Result. That misleads about `?`'s generality and
privileges one of two equal tryables (against the single-constructor canon `Option(T)` /
`Result(T, E)`, TYP-5/TYP-1, and the protocol-not-name rule, CF-8). Without `T?`, `?` keeps a
**single meaning** — postfix try on a value — there is no `?` type-position, which also
removes the types-vs-values comptime ambiguity (TYP-1).

### CF-10. `Tryable` gains `from_failure` — `?` can *rebuild* a user tryable's failure
*spec: Control Flow §8.2, Stdlib §2.2*

**Why.** The `Tryable` protocol was originally **decomposition-only**
(`is_success`/`success_value`/`failure_value`) — enough to *inspect* a tryable, but `?`
must also **build** the enclosing function's failure value to early-exit with it. For
`Option`/`Result` the impl wraps the converted failure in built-in `None`/`Err`, and when
the enclosing function returns the operand's own type the operand passes verbatim. But when
the enclosing function returns **one** user tryable and the `?`-operand is a **different**
tryable, there was **nothing to reconstruct the enclosing type's failure with** — TYP-6 had
noted the wrap target "may be an aggregate whose variant wraps the source" yet provided no
way to declare that wrap (a FND-3 gap: disassemble but not reassemble). The fix adds a fourth
operation, the inverse of `failure_value`: `from_failure(in f : F) -> Self`. The three wrap
cases collapse to **one rule** (build the enclosing tryable's failure): (a) enclosing ==
operand type → verbatim; (b) `Option`/`Result` → built-in `None`/`Err` (implicit
`from_failure`); (c) a user tryable → its `from_failure`, **selected by the statically
known enclosing return type** at the `?` site. That selection is return-type-directed but
**scoped to the `?`/`OutErr` mechanism and keyed on the fixed name** `from_failure` — it is
**not** the rejected "any `(S) -> T` is silently a conversion", and it is distinct from
`@convert` (declared-and-visible, I3/I8).

**Rejected.** **Structurally guessing the failure variant** (the non-`is_success` arm of a
2-variant enum) — works only for enum-shaped tryables, forces the compiler to comptime-decide
which variant is the failure, and a wrong guess is a **silent** miscompilation (worse than an
explicit op); `from_failure` is uniform over any tryable shape and greppable (I3). **Reusing
`@convert` `F -> Self`** — a `@convert` target is a single (source, target) constructor, but a
tryable failure target is a *variant wrap* of an aggregate; folding them would blur two
distinct mechanisms (OP-1).
