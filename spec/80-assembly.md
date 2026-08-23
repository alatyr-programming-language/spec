# Alatyr Language Specification

## Chapter — Assembly Correspondence

> **Status — draft (under review).** This chapter is a draft and is not yet
> accepted; the specification as a whole is under development. See the status note
> at the top of the Overview chapter.

This chapter defines the **assembly-correspondence surface**: instructions as
per-architecture intrinsics, the operand and register model, memory and SIMD
operands, data definitions, the raw GAS escape, and the architecture matrix. This
surface is what makes the language **lower-layer-complete** — everything expressible
in GAS for a supported target is expressible here (I4) — and **byte-exactly
transparent** under the `no_abstractions` limit (I1).

This chapter is the **register-ISA backend's** assembly-correspondence (CG-4): registers, instruction
intrinsics, raw transfer, and `asm` are a **target-capability** of the register ISAs. A structured/VM
backend (e.g. **WASM** — only structured `br`/`br_table`, no raw goto, no code addresses, no registers)
has its **own**, narrower lowest form and is **additive** (FND-6); there, these raw constructs are a
compile error (gated like an unavailable instruction). The **portable core** (types, functions/ABI,
structured control flow, data, comptime) lowers to every backend.

It builds on the Type-System chapter (`bitsN`, arrays, constructors), the Functions
chapter (ABI, naked functions), the Control-Flow chapter (raw labels/jumps and the
`unchecked` requirement for raw transfer), and the Comptime chapter (`when`-gating,
immediates). The exact **per-architecture tables** (registers, instructions, operand
forms, relocations) are **normative v1 content** delivered as the **per-architecture
appendix** (`170-appendix-arch.md`) — one section per architecture — which is **required** for v1 and is
**drafted** (its table format, the supported-target set, and all six architecture
sections — `x86_64`, `i386`, `aarch64`, `aarch32`, `riscv32`, `riscv64` — are written,
under review). This chapter fixes the **schema** (§2–§9) that appendix populates; **"additive" (FND-6)
governs only *future* growth** (new instructions or architectures without breaking a
v1 program) — it does **not** make v1's tables optional or unspecified (Overview §6).
The precise GAS emission is the Codegen chapter.

Requirement keywords (**MUST**, **MUST NOT**, **SHOULD**, **MAY**) follow RFC 2119
(Overview §6). Syntactic forms are given as grammar fragments (§11); the consolidated
EBNF is in the Grammar chapter.

---

### 1. The assembly surface (I4)

Everything expressible in GAS for a supported target **MUST** be expressible on this
surface (I4); assembly-correspondence code is **self-sufficient** — kernels,
firmware, and boot code can be written entirely in it. The surface is available
**everywhere** (one language; Overview §3); the **`no_abstractions`** limit restricts
a translation unit to *only* this 1:1 surface (no structured abstractions), where
byte-exact prediction holds (I1).

#### 1.1 What `no_abstractions` admits (normative)

Under `@limits(no_abstractions)` the operative rule (Overview §3; decision CG-12) is that
**every emitted runtime instruction MUST be one the programmer wrote, 1-to-1 with GAS**. A
construct is admitted **iff** it emits exactly the instruction(s) it names, or emits **no**
runtime instruction (it is erased). Concretely, a conforming implementation MUST admit and
MUST forbid exactly:

- **Admitted** — instruction intrinsics (§2) and the raw escape `asm(…)` (§4); operands as
  ordinary expressions (§5) — registers (§6), immediates (comptime values), the `at(…)`
  memory operand (§7), labels + a label address as `bitsN` data, reloc/TLS and SIMD
  decorations (§8); **raw control transfer** (`jmp`/branch/indirect), which is **native**
  here and needs no `unchecked` grant (Control Flow §2.3); data definitions (§9); the
  **function frame** of §2.2 — `@abi(naked)` **or** ABI-framed — over local operand bindings
  whose initializers are surface expressions, with the destination `return`ed by the
  enclosing function; and **comptime** in full (permitted and erased, I7 / FND-10).
- **Forbidden** (a compile error) — the library value-operators as surface syntax
  (`+ - * / %`, comparison, bitwise, `and`/`or`, unary `-`/`not`: a guarded operation is not
  1:1, §3); structured control flow (`if`/`while`/`for`/`loop`/`match`/`break`/`continue`);
  the `?` tryable; calls to non-instruction functions; structured data access (`s.f`, enum
  construction/`match`, tuple projection, `a[i]`, range-slices); and the pointer word-functions
  `ptr`/`deref` and pointer arithmetic. The raw substitutes are the instruction intrinsics,
  labels + raw transfer, and the `at(…)` memory operand.

The limit binds the **unit's own source** (per-translation-unit, FND-12; not transitive over
the call graph) and is **orthogonal to verification** (CG-6). Its full construct-by-construct
rationale and rejected alternatives are decision **CG-12**.

---

### 2. Instructions are intrinsics

#### 2.1 Function-call form

An instruction is a **per-architecture intrinsic** — a kind of builtin (Type System
§4.5; a prelude identifier, not `@`-marked — CG-2/SYN-7) — written in **function-call form**:

```
movq(rax, 60)
syscall()
```

Instructions are **unified** with the numeric-operator intrinsics: `u64`'s `+`
lowers to the target add (`add` → `addq`) by the same mechanism (Type System §3.1).
They are available **directly** at the assembly surface.

#### 2.2 Destination-first operands

Operand order is **destination-first**: `movq(rax, 60)` (move `60` into `rax`), which
by UFCS is also `rax.movq(60)` (the instruction as a method on its destination; Type
System §4.5). Emission to GAS reorders to the assembler's convention (AT&T is
source→destination); **which** instruction is emitted is preserved 1:1 (§3).

An instruction is callable not only in a naked body but **as a statement in an
ordinary (ABI-framed) function over local variables**: each operand local is
materialized in a register, the instruction is emitted on those registers, and a
**read-modify-write** destination operand (`add`, `sub`, …) **mutates its first
operand's place** — the value written back to that local. This is the surface a
library uses to give a native operator a one-instruction body (Type System §3.1):
`mut out : u64 = a; add(out, b)` leaves `a + b` in `out`. (The instruction itself
yields no value; a result is the mutated destination, returned by the enclosing
function.)

---

### 3. One-to-one and guards

- A **raw instruction intrinsic emits exactly one GAS instruction** (I1) and carries
  **no guard** — there is nothing to check; it *is* the machine instruction.
- A **checkable operation** (division by zero; a shift ≥ the operand width; an
  unaligned load/store where the target faults) gets, by default (**checked**), a
  **documented runtime guard + inline trap** (the exact target instruction is the
  per-architecture appendix §2 table: `ud2` / `brk #0` / `udf #0` / `ebreak`) → a defined
  trap, never UB (I11). **Inside an `unchecked` scope** the guard is dropped (1:1)
  (the verification mode; Control Flow §2.3; decisions CG-7/CG-6/CG-8).
- A **library-defined** operator/conversion guard (the kernel/prelude split makes
  `u64 +`, `char(n)`, … library functions, TYP-2/TYP-5) is **not** recognized by the
  compiler. The library makes its guard mode-dependent by reading the verification mode
  as a comptime fact — `comptime if verify.checked { <guard> }` (Comptime §7.5 / CT-11) — so the
  guard is present in the function's **checked** instantiation and comptime-absent in
  its **unchecked** one. `unchecked(x + y)` thus drops the guard by selecting the
  unchecked instantiation (leaving the intrinsic's wrap), not by the compiler stripping
  a guard it understands.
- **Compile-time checks are always applied**: a constant out of range, a write
  through an immutable path, operand-kind validity (§5), and arity.
- **`no_abstractions` is orthogonal to verification** (CG-6): it restricts a unit to
  the 1:1 form — a guarded checkable operation is *not* 1:1, so it is unavailable
  there; what remains are raw instructions (already guardless) — but it does **not**
  change verification (the `checked`/`unchecked` axis).

---

### 4. The instruction set — data-driven and additive

The instruction **schema** — the operand kinds (§5), the destination-first form
(§2.2), the 1:1 emission contract (§3) — is **fixed by this specification**. The
**catalog** of instructions is the **per-architecture instruction table** — normative
v1 content **enumerated in the per-arch appendix** (`170-appendix-arch.md`), required for v1.
**"Additive" (FND-6)** means a *future* version MAY add instructions or architectures
without breaking a v1 program; it does **not** leave v1 unspecified (§intro).

**Raw GAS escape — `asm(…)`.** An arbitrary mnemonic or assembler string is emitted
with the builtin **`asm(template, operands…)`** (§11), where `template` is a
comptime `str` of literal GAS text. It **requires an `unchecked` grant** (Control
Flow §2.3), is validated **only by `as`** (no Alatyr checks or guards apply), and is
the **fallback** that guarantees I4 even where the typed tables are incomplete. Typed
instructions are the **curated, checked** set; `asm(…)` is the **uncurated** path. (It
generalizes the old `bytes(…)` / `snippet`.)

**Operand substitution (normative).** Operands are bound into the template by
**positional placeholders** `{0}`, `{1}`, … (zero-based): placeholder `{i}` is
replaced by the **emitted operand text** of the `i`-th operand — a register name, an
immediate, a memory operand (§7), or a symbol with its reloc (§8.2), i.e. an operand
kind of §5. A literal brace is written `{{` / `}}`. It is a **compile error** if a
placeholder index has no operand, an operand is unreferenced, or an operand is not a
valid operand kind. The text outside placeholders is passed to `as` **verbatim**. The
only part deferred to Codegen is the **per-target textual spelling** of a substituted
operand — which is exactly the spelling typed instructions already use — so the
substitution scheme itself is fully defined here.

#### 4.1 Synthetic destination-first intrinsics (implicit-result instructions)

A few hardware instructions produce their result in an **implicit** register pair rather
than a written destination — on x86_64, `div`/`idiv` and `mul`/`imul` use `rdx:rax` and
leave the **quotient/low product in `rax`** and the **remainder/high product in `rdx`**.
The 1:1 typed catalog exposes the quotient/low forms directly (`idiv`/`div`, `imul`/`mul`),
but the library `%` operator and the `*`-overflow guard (Stdlib §2.6 / TYP-2) need the
**remainder** and the **high half**. For these the per-arch table MAY list **synthetic
destination-first intrinsics** — `remq`/`iremq` and `mulhiq`/`imulhiq` on x86_64 — which:
emit the **same** one-operand `div`/`idiv` / `mul`/`imul` (the `rdx:rax` dance), but the
lowering **captures the implicit `rdx`** (remainder / high product) as the result. They
take the destination-first shape `remq(out, b)` ≡ `out := out % b`: one ISA source operand
(the divisor / multiplier), with the dividend/multiplicand and result the implicit
`rdx:rax` — so the form carries **one extra leading destination** over the bare hardware
instruction. They are still 1:1 in *which* hardware instruction is emitted (§3); the
synthesis is only the implicit-register **capture**, and each is enumerated as a normative
row in the per-arch appendix (§170 §3.2) like any other intrinsic. (A target without such a
synthetic form re-expresses `%`/high-multiply over the listed `idiv`/`mul` + `cqo` + a
register read; the synthetic intrinsic is the curated convenience.)

A second use of the synthetic destination-first form is the **fused compare → bool**. A
scalar comparison `lt`/`eq` (Stdlib §2.6) reads the flags an ISA compare sets and writes a
**`bool`** — on x86_64 the integer forms are the single `setcc` family (`setb`/`setl`/`sete`/…,
each `cmp` + `setcc` + zero-extend). For **floating-point** the unordered (NaN) rule
(Concurrency §7.5) cannot be read from a single `setcc`, so the per-arch table MAY list
**synthetic float-compare intrinsics** — `fltsd`/`fltss` (ordered `<`) and `feqsd`/`feqss`
(ordered `==`); `sd`=double / `ss`=single — which emit a `ucomisd`/`ucomiss` then the
NaN-aware sequence (`seta` for ordered-`<`; `sete`+`setnp`+`andb` for ordered-`==`) and
zero-extend the byte. Like the setcc forms they take the fused shape `fltsd(out, a, b)` ≡
`out := (a < b)` — the byte destination plus the two compared operands. They are the FP
counterpart of `setb`/`sete`: the curated lowering of the library float `lt`/`eq` operator
bodies, each a normative row in the per-arch appendix (§170 §3.2). The **non-strict** float
operators `<=`/`>=` are **not** new intrinsics — they derive (Stdlib §2.6) as the ordered
`lt || eq` (both `false` for a NaN, so the result is `false`), never the total-order
shortcut `not lt(swapped)` (which would be `true` for a NaN, violating §7.5).

---

### 5. Operands

Operands follow a **fixed kind schema**:

| Kind | What | Notes |
|------|------|-------|
| **register** | an arch register (§6) | from the machine model |
| **immediate** | a **comptime** value | range/validity checked at compile time (§3) |
| **memory** | an addressing operand (§7) | `at(…)` arch-surface operand |
| **label** | a code label | a `@label` code-point (Control Flow §2.1) |
| **reloc / TLS** | a symbol operand under a relocation/TLS decorator | per-arch decorators, §8.2 |
| **SIMD** (per arch) | lane/mask-decorated registers (§8) | additive per-arch data |

Which kinds a given instruction accepts is **part of its table** (§4); operand-kind
validity is **always** checked (§3). Operands are **ordinary expressions** (Type
System / Declarations) — there is **no** special operand grammar.

---

### 6. Registers

Registers are **arch-data identifiers, not keywords** (Grammar §2.3; CG-2 — they are
prelude/arch names, not reserved words). They come from the **machine model** (configuration): the
GP, FP/SIMD, and system register **classes**, each register named per the arch table.
A register name is an ordinary **case-sensitive** identifier — the canonical lowercase
spelling of the arch table (`rax`, not `RAX`; Lexis chapter). Pinning a binding to a specific register is `@reg(name)` (Memory-model
§2.4).

**Arch surface in scope.** The **selected target**'s instruction and register surface
is in scope directly (you write `movq`, `rax`). Inside a `when(target.arch == …)`
gate (Comptime §7), **that arch's** surface is in scope. Otherwise an arch's surface
is reached **qualified** (`x86_64::movq`) or by binding it (`a := x86_64`, then
`a::movq`) — there is **no** `use` (decisions MOD-4).

---

### 7. Memory operands

A memory addressing operand is the **arch-surface addressing operand `at`** — co-located
with the registers and instructions of the selected target (§6), in scope directly for
that target and reached **qualified** (`x86_64::at(…)`) outside an arch gate — with
**named arguments** (Functions §5.2):

```
at(base = rbp, index = rcx, scale = 8, disp = -16)
at(base = rip, offset = sym)
```

It yields an **operand value** usable wherever a memory operand is accepted (§5). The
**valid field combinations** (which of `base` / `index` / `scale` / `disp` / `offset`
an addressing mode allows) are **per-arch data** (§4). There is no special `[ … ]`
addressing grammar — `at(…)` is an ordinary named-argument call (decisions CG-2).

---

### 8. Operand decorations (SIMD, relocation/TLS)

Operand **decorations** are **arch-intrinsic decorators applied UFCS-style** (Type
System §4.5), composable, with **no** special suffix grammar — a decoration is an
ordinary UFCS call (decisions CG-2). The decorator **sets are per-arch data** (their
catalogs live in the per-arch appendix) and grow additively (FND-6). Validity (a
decorator must exist for the arch and be legal on its operand) is checked from that
table (§3).

#### 8.1 SIMD lane/mask decorations

```
v0.lanes(f32, 4)                       # interpret v0 as 4×f32 lanes
zmm0.mask(k1).zeroing()                # ≡ zeroing(mask(zmm0, k1))
```

(`.4s`, `{k1}{z}` suffix forms are rejected — they would need a separate grammar.)

#### 8.2 Relocation / TLS decorations

A **relocation / TLS operand** is a **symbol** (a `@label` code-point, a data symbol,
or an external `@extern` name) under a **relocation decorator** — the same UFCS form,
producing a reloc operand:

```
sym.plt()           # x86_64:  sym@PLT
sym.gotpcrel()      # x86_64:  sym@GOTPCREL
sym.got()           # aarch64: :got:sym
sym.pcrel_hi()      # riscv:   %pcrel_hi(sym)
```

- **Syntax**: a UFCS decorator on a symbol operand (no `@`/`:`/`%` surface syntax —
  those are the *GAS* spellings the decorator lowers to).
- **Which decorators exist, and on which operands they are legal, is per-arch data**
  (the per-arch reloc table, in the per-arch appendix); using an unknown or operand-illegal decorator is a
  compile error (§3).
- **Lowering**: each decorator lowers to the architecture's GAS relocation syntax
  (`sym@PLT`, `:got:sym`, `%pcrel_hi(sym)`, …) per the per-arch table; the exact
  emission is the Codegen chapter. (This is the same model as SIMD decorations — a
  per-arch UFCS decorator family — not a separate operand grammar.)

---

### 9. Data definitions

- **Values** use the ordinary constructors (Type System §9): `bitsN(v)`,
  `u64(0x1234)`, strings, arrays.
- **Raw bytes are `[u8; N]` values** — there is no separate `bytes(…)`: inline as a
  literal `[0x48, 0x65, …]`, or from a file via **`embed("path")`** (a reproducible
  comptime builtin → `[u8; N]`, Comptime §2.4). Either is placed into a section by an
  **ordinary data binding** (Memory §2.3). No data-level escape is needed — any
  byte pattern is expressible as `[u8; N]` (I4).
- **Sections** are derived from mutability × initialization (Memory §2.3);
  `@section("…")` overrides. **Endianness-aware** emission follows the target default
  with per-site `@endian(big)` / `@endian(little)` (Type System §8).

---

### 10. The architecture matrix

- **v1 architectures**: `x86_64`, `i386`, `aarch64`, `aarch32`, `riscv32`, `riscv64`
  — each **data-driven**: per-arch tables of registers (§6), instructions (§4),
  operand forms (§5), and ABI (Functions §6). New architectures are **additive**
  (FND-6) — a new arch is new data, not new language.
- The per-arch tables themselves (registers, instructions, operand forms,
  relocations, decorators) are the **per-architecture appendix**
  (`170-appendix-arch.md`); this chapter fixes only the schema they populate.
- The **target** is selected by the manifest (configuration; Tooling chapter); the
  **machine model** (Type System §1) is the per-arch data the rest of the language is
  defined over (I6).
- **Containers** (ELF / PE / Mach-O / COM) are chosen by the target triple; their
  emission is the Codegen chapter.

---

### 11. Syntax (grammar fragments)

This chapter introduces **no grammar productions**: an instruction is a call to a
per-arch intrinsic; `asm` is a prelude-identifier call and `at` is an arch-surface call; an operand decorator is a
UFCS call, and arch access is a `path` — all derived through ordinary call / UFCS /
path forms in the Grammar chapter (`postfix-expr`, `path`). `instr-name`, `reg`, and
the per-arch catalogs are arch data (§4, §10). The following are **form schemas**
(operand shapes and meaning), not productions:

```text
instr-name( arg-list )           — instruction, destination-first: movq(rax, 60), syscall()
expr . instr-name( arg-list )    — instruction, UFCS form: rax.movq(60)
at( mem-field, … )                   — addressing operand (§7); mem-field = base|index|scale|disp|offset = expr
expr . ident( arg-list )         — operand decorator (§8): v0.lanes(f32, 4); sym.plt(); composable
path                             — arch surface access (§6): x86_64::movq, or a::movq after `a := x86_64`
asm( string, expr, … )           — raw GAS escape (§4); requires an unchecked grant; only `as` validates
```

Normative notes:

- An instruction call is destination-first; its UFCS form is equivalent (§2.2). The
  emitted GAS reorders to the assembler's convention but preserves which instruction
  (§3; Codegen).
- `at(…)` is a named-argument call (no `[ ]` grammar); SIMD and reloc/TLS decorations
  are UFCS calls (no suffix/`@`/`:` grammar) — all are per-arch data (§7, §8).
- The raw escape is the builtin **`asm("…GAS…", operands…)`** (this chapter fixes the
  form **and** the positional-`{i}` substitution scheme, §4): it needs an `unchecked`
  grant and is validated only by `as`; only the per-target operand *spelling* (shared
  with typed instructions) is Codegen's.
- An immediate operand is a **comptime** value, range-checked at compile time (§5).

---

### 12. Conformance (normative summary)

A conforming implementation MUST:

1. provide the assembly-correspondence surface such that everything GAS-expressible
   for a supported target is expressible (I4), available everywhere and the sole
   surface under `no_abstractions` with byte-exact prediction (§1; I1);
2. expose instructions as per-arch intrinsics in destination-first call form (with
   the equivalent UFCS form), unified with the numeric-operator intrinsics, emitting
   exactly one GAS instruction per raw instruction (§2, §3; I1);
3. apply checked-default guards + inline traps to checkable operations, drop the
   guard inside an `unchecked` scope (CG-7), always
   apply the compile-time checks, and keep `no_abstractions` orthogonal to the
   verification mode (§3; CG-6/I11);
4. fix the instruction **schema** while enumerating the instruction **catalog** as
   normative per-arch appendices (v1-required; "additive" = future growth only), and
   provide the **raw GAS escape `asm(template, operands…)`** that requires an
   `unchecked` grant and is validated only by `as` (§4, §intro; I4/FND-6/FND-3);
5. accept the fixed operand-kind schema (register / immediate / memory / label /
   reloc / per-arch SIMD) as ordinary expressions, checking operand-kind validity,
   with immediates comptime and range-checked (§5);
6. treat registers as arch-data identifiers (not keywords) from the machine model,
   put the selected target's (or a `when`-gated arch's) surface in scope, and
   otherwise require qualification or a binding — **no `use`** (§6; MOD-4);
7. provide the arch-surface `at(…)` (named-argument memory operand) and **UFCS operand decorations**
   — SIMD lane/mask **and relocation/TLS** (a symbol under a per-arch reloc decorator
   lowering to the GAS reloc syntax) — with their valid forms as per-arch data,
   checked against that table, and **no** special bracket/suffix/`@`/`:` grammar
   (§7, §8);
8. define data as ordinary constructors and `[u8; N]` (inline or `embed`) placed by
   data bindings into derived sections with target endianness — no data-level escape
   (§9; I4);
9. implement the six v1 architectures as data-driven per-arch tables, additive for
   new arches, with the machine model and container chosen by the target (§10; I6/FND-6).
