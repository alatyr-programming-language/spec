# Alatyr Language Specification

## Appendix — Per-architecture tables and supported targets

> **Status — draft (under review).** This appendix is a draft and is not yet
> accepted; the specification as a whole is under development. See the status note
> at the top of the Overview chapter. **All sections are now drafted: the table format
> (§2), the supported-target set (§1), and all six architectures — x86_64 (§3), i386
> (§4), aarch64 (§5), aarch32 (§6), riscv64 (§7), riscv32 (§8).**

This appendix fixes the **normative v1 per-architecture content** the Assembly chapter
defers to "the per-arch appendix" (Assembly §intro, §4, §6, §8) and the **closed set of
supported targets** referenced by the Manifest appendix (§3.1) and the ABI appendix
(§2). The Assembly chapter is normative for the **schema** — the instruction model
(§2–§4), the operand kinds (§5), the register classes (§6), memory operands (§7), and
the relocation/SIMD decorator model (§8); this appendix **populates** that schema with
each architecture's data.

Per CG-2, the instruction set per architecture is a **curated v1 set** (the common
integer / memory / control / system / baseline-SIMD instructions, each 1:1 with GAS and
carrying its checked-guards); **everything else on the ISA is reachable through
`asm(…)`** (Assembly §4), the uncurated escape that guarantees lower-layer completeness
(I4). "Additive" (FND-6) governs only *future* growth of the curated set — it does not
make the v1 set optional (Overview §6).

---

### 1. Supported targets (the closed v1 set)

A **supported target** is one of the `(Machine` variant`, arch)` pairs below — the
manifest states the platform as a `Machine` variant (Manifest §3.1; TOOL-18), and the
`os`/`env`/`container` columns are that variant's `target.*` **projections**. A `Target`
outside this set is a **Config diagnostic** (Manifest §3.1; ABI §2). The default ABI for
each is the ABI appendix §2 procedure.

| `machine` variant | `arch` | → `os` | → `env` | → `container` | Notes |
|---|---|---|---|---|---|
| `Machine.Linux`   | `x86_64`  | `linux`   | `gnu` / `musl`    | `elf`   | the reference Linux target |
| `Machine.Windows` | `x86_64`  | `windows` | `none`            | `pe`    | Win64 ABI (ABI §4.2); `subsystem` is the variant's |
| `Machine.Macos`   | `x86_64`  | `macos`   | `none`            | `macho` | System V variant |
| `Machine.Freebsd` | `x86_64`  | `freebsd` | `gnu`             | `elf`   |  |
| `Machine.Bare`    | `x86_64`  | `none`    | `none`            | `elf`   | freestanding (kernel/firmware) |
| `Machine.Linux`   | `i386`    | `linux`   | `gnu` / `musl`    | `elf`   |  |
| `Machine.Bare`    | `i386`    | `none`    | `none`            | `elf`   | freestanding; 32-bit boot |
| `Machine.Com`     | `i386`    | `none`    | `none`            | `com`   | DOS/real-mode flat image (`code_size = CodeSize.b16`) |
| `Machine.Linux`   | `aarch64` | `linux`   | `gnu` / `musl`    | `elf`   |  |
| `Machine.Macos`   | `aarch64` | `macos`   | `none`            | `macho` | Apple AAPCS64 variant |
| `Machine.Bare`    | `aarch64` | `none`    | `none`            | `elf`   | freestanding |
| `Machine.Linux`   | `aarch32` | `linux`   | `eabi` / `eabihf` | `elf`   | soft / hard float (ABI §4.5) |
| `Machine.Bare`    | `aarch32` | `none`    | `eabi` / `eabihf` | `elf`   | freestanding |
| `Machine.Linux`   | `riscv64` | `linux`   | `gnu`             | `elf`   |  |
| `Machine.Bare`    | `riscv64` | `none`    | `none`            | `elf`   | freestanding |
| `Machine.Linux`   | `riscv32` | `linux`   | `gnu`             | `elf`   |  |
| `Machine.Bare`    | `riscv32` | `none`    | `eabi`            | `elf`   | freestanding/embedded |
| `Machine.Android` | `aarch64` | `android` | `bionic`          | `elf`   | NDK `arm64-v8a`; PIE/PIC; `api ≥ 21` |
| `Machine.Android` | `aarch32` | `android` | `bionic`          | `elf`   | NDK `armeabi-v7a`; soft-float **ABI** with `vfp`/`neon` instructions available (ABI §2(c)) |
| `Machine.Android` | `x86_64`  | `android` | `bionic`          | `elf`   | NDK `x86_64` (emulator/devices) |
| `Machine.Android` | `i386`    | `android` | `bionic`          | `elf`   | NDK `x86` |

The **minimum supported `api`** for `Machine.Android` is **21** (the level at which PIE is
required and the bionic surface this specification assumes is present); it is that field's
default, and a lower value is a Config diagnostic (Manifest §3.1). Android's **kernel** ABI
is Linux's, so `@abi(syscall)` uses the Linux table (ABI appendix §5); its **userland** is
bionic, which is what `target.env` reports.

`Machine.Com()` takes no `arch` argument — it **is** the `i386` real-mode image — and
`Machine.Windows` / `Machine.Macos` take no `env`: their ABI environment is fixed by the
platform, which is why neither `msvc` nor a `pe`+`gnu` combination is expressible at all
(TOOL-18).

The `com` container (Assembly §10) is the **`Machine.Com()`** target (the row above), for
DOS/real-mode flat boot images: a platform variant of its own rather than a container field
combined with an arch. It **requires `code_size = CodeSize.b16`** — that is its
**default** there and any other value is a Config diagnostic (Manifest §3.2) — selecting the
**16-bit instruction-encoding mode** (a codegen mode, §4); it does **not** change the
machine model, whose pointer width is the arch's 32-bit (Manifest §3.2). New triples are
**additive** (FND-6).

---

### 2. Table format (the schema each architecture section fills)

Every architecture section (§3–§8) provides **five tables**, all populating the Assembly
chapter schema:

**(a) Registers** (Assembly §6) — by **class** (GP / FP-SIMD / system), each row a
register name and its width(s)/sub-register names:

> `class | name(s) | width | notes`

**(b) Instructions** (Assembly §4) — the curated v1 set, each row: the Alatyr
**intrinsic name**, the **operand kinds** it accepts (the §5 kinds: reg / imm / mem /
label / reloc / SIMD), the **GAS mnemonic** it lowers to (the I1 1:1 contract), and any
**checked-guard** (Assembly §3; e.g. division-by-zero, over-width shift):

> `intrinsic | operands | GAS | checked-guard`

A curated intrinsic MAY map to an assembler **pseudo-instruction** (e.g. RISC-V
`call`/`ret`/`j`/`li`/`nop`): the 1:1 contract (§9) is **at the GAS-line level** — one
intrinsic emits one GAS line (the pseudo) — and the assembler's own expansion of that
pseudo into machine instructions is **below the GAS-text level** (consistent with
"conformance = rule + cost, not byte-identical GAS", Codegen §3). Such rows name the GAS
pseudo and note "(pseudo)".

**(c) Addressing modes** (Assembly §7) — which `at(…)` field combinations
(`base`/`index`/`scale`/`disp`/`offset`) the architecture accepts:

> `form | fields | example`

**(d) Relocation / TLS decorators** (Assembly §8.2) — each UFCS decorator and the GAS
relocation syntax it lowers to:

> `decorator | GAS reloc | legal on`

**(e) Operand decorators** (Assembly §8) — the SIMD lane/mask decorators **and any
arch-specific operand decorator** (e.g. an x86 segment override), with their meaning:

> `decorator | meaning`

An instruction or decorator **absent** from the curated tables is reached with `asm(…)`
(Assembly §4); the tables are the curated set, not the whole ISA.

**Canonical feature names.** Each architecture section fixes the arch's **canonical
ISA-feature names**. Two disjoint sets: the **opt-in feature names** a manifest's
`features` / `FeatureAlias.members` may name (Manifest §3.7; an unknown opt-in name is a
Config diagnostic), and the **baseline query names** that are queryable in
`target.features` but **forbidden** in a manifest feature list. By role:

- **baseline** — architecturally mandatory, **always present**, not opt-in (and not
  disablable): **`sse2`** on `x86_64`, **`neon`** on `aarch64`. `features = []` still
  has these, and they appear in `target.features` under those canonical names
  (**query-only**: source may test `target.features.has("sse2")`, but listing a baseline
  name in the manifest `features` is a Config diagnostic — Manifest §3.7).
- **instruction/register gates** — opt-in features that enable curated table rows or
  registers: `x86_64` `avx`/`avx512`/`fma`, `i386` `sse` (its `avx`/`avx512` are **additive**,
  not v1), `aarch64` `sve`, `aarch32` `vfp`/`neon`/`idiv`, RISC-V `m`/`f`/`d`/`v`. Some
  also select the ABI float sub-variant (RISC-V `f`/`d`, ABI §2(c)).
- **capability flags** — opt-in features that change codegen or enable builtins without
  a dedicated curated table row: `x86_64` `cx16` (the 128-bit `CMPXCHG16B` atomic, §2
  atomic-widths), RISC-V `a` (atomics, used by the Concurrency builtins) and `c`
  (compressed-instruction encoding, a codegen choice).

So the canonical set is **broader** than the table gates (it also names baseline and
capability features); the gates are the subset that appears as a feature condition in
the curated tables.

Each architecture section also fixes the arch's **feature implication graph** — the
edges under which a declared feature pulls in others when `features` is resolved to its
closure (Manifest §3.7). The v1 edges are **arch-specific**: `aarch32` `neon` ⇒ `vfp`;
RISC-V `d` ⇒ `f`; `x86_64` `avx512` ⇒ `avx`, `fma` ⇒ `avx` (SSE2 baseline, always present — no `sse`
opt-in); `aarch64` `sve` ⇒ NEON (baseline, already present). `i386` has only the `sse`
opt-in with **no** edges (its `avx`/`avx512` are additive, not v1). `target.features`
(Tooling §2.7) is this **closure**, and all gating/ABI decisions read the closure.

**Atomic-capable widths (all six arches).** The widths on which the atomic builtins
(`atomic::*`, Concurrency §2) lower to a hardware atomic; an atomic op on a width **not**
available for the **selected target's features** is a **compile error** (Concurrency §2).

| arch      | native atomic widths | wider / feature-gated                              |
|-----------|----------------------|----------------------------------------------------|
| `x86_64`  | 8 / 16 / 32 / 64     | **128** with feature `cx16` (`CMPXCHG16B`)         |
| `i386`    | 8 / 16 / 32 / 64     | 64 via `CMPXCHG8B` (baseline)                      |
| `aarch64` | 8 / 16 / 32 / 64     | **128** via LSE `CASP` or `LDXP`/`STXP`            |
| `aarch32` | 8 / 16 / 32 / 64     | 64 via `LDREXD`/`STREXD` (ARMv6K baseline)         |
| `riscv64` | 32 / 64              | requires the `a` extension; 8 / 16 not native (additive: `Zabha`) |
| `riscv32` | 32                   | requires the `a` extension; 8 / 16 / 64 not native |

The common case is native widths **≤ pointer width**; a double-pointer-width (128-bit)
atomic is always feature- or instruction-gated. On RISC-V the atomics are the `a`
extension — without it every atomic op is a compile error.

**Checked-failure trap (all six arches).** Every checked-guard failure — the
checked-overflow set, **division by zero**, an **over-width shift**, the bounds /
alignment / narrowing guards, and a false `@require` constructor predicate (Types §8.1) —
is the language's **direct inline trap** and emits exactly the GAS instruction below
(CG-13). It does **not** call `panic`, an OS exit, or a programmer hook and performs no
unwinding. Where the hardware would fault on its own (`x86_64` `idiv` with a zero
divisor), the compiler-inserted guard runs **first**, so the observable failure is this
instruction, never the machine's own fault — the mechanism is identical on every target.
A future architecture added to this appendix MUST define one corresponding instruction
before it is a supported target.

| arch | GAS instruction |
|------|-----------------|
| `x86_64` | `ud2` |
| `i386` | `ud2` |
| `aarch64` | `brk #0` |
| `aarch32` | `udf #0` |
| `riscv64` | `ebreak` |
| `riscv32` | `ebreak` |

---

### 3. `x86_64` (reference exemplar)

Machine model: 64-bit, little-endian; pointer width 64. Default ABI: `sysv` (ELF/Mach-O)
or `win64` (PE) — ABI appendix §4.1/§4.2.

#### 3.1 Registers

| Class | Name(s) | Width | Notes |
|-------|---------|-------|-------|
| GP | `rax` (`eax`/`ax`/`al`) | 64/32/16/8 | accumulator; `sysv` int result |
| GP | `rbx` (`ebx`/`bx`/`bl`) | 64/32/16/8 | callee-saved |
| GP | `rcx` (`ecx`/`cx`/`cl`) | 64/32/16/8 | 4th `sysv` int arg / shift count (`cl`) |
| GP | `rdx` (`edx`/`dx`/`dl`) | 64/32/16/8 | 3rd `sysv` int arg; `rdx:rax` for wide mul/div |
| GP | `rsi` (`esi`/`si`/`sil`) | 64/32/16/8 | 2nd `sysv` int arg |
| GP | `rdi` (`edi`/`di`/`dil`) | 64/32/16/8 | 1st `sysv` int arg |
| GP | `rbp` (`ebp`) | 64/32 | frame pointer (callee-saved) |
| GP | `rsp` (`esp`) | 64/32 | stack pointer |
| GP | `r8 … r15` (`r8d`/`r8w`/`r8b` …) | 64/32/16/8 | `r8`/`r9` = 5th/6th `sysv` int args |
| PC | `rip` | 64 | instruction pointer (RIP-relative addressing only) |
| flags | `rflags` | 64 | status flags (set by `cmp`/`test`/arithmetic) |
| FP/SIMD | `xmm0 … xmm15` | 128 | SSE vector registers — **SSE2 is baseline** on `x86_64` (always present) |
| FP/SIMD | `ymm0 … ymm15` | 256 | AVX vector registers — **only with the `avx` feature** |
| FP/SIMD | `zmm0 … zmm31` | 512 | AVX-512 vector registers — **only with `avx512`** |
| mask | `k0 … k7` | 64 | AVX-512 opmask registers — **only with `avx512`** |

#### 3.2 Instructions (curated v1)

| Intrinsic | Operands | GAS | Checked-guard |
|-----------|----------|-----|---------------|
| `mov` (`movq`/`movl`/`movw`/`movb`) | reg/mem, reg/mem/imm | `mov` | — |
| `movzx` / `movsx` | reg, reg/mem | `movz*`/`movs*` | — (zero/sign extend) |
| `lea` | reg, mem | `lea` | — (address compute) |
| `add` / `sub` | reg/mem, reg/mem/imm | `add`/`sub` | overflow set (Concurrency §6) on the checked op |
| `imul` / `mul` | reg/mem (, reg, imm) | `imul`/`mul` | overflow |
| `idiv` / `div` | reg/mem | `idiv`/`div` | **division by zero** (and `MIN/-1`) → the inserted guard traps *before* the instruction (§2) |
| `iremq`/`remq` · `imulhiq`/`mulhiq` (**synthetic**, §80 §4.1) | reg/mem (divisor / multiplier) | `idiv`/`div` capturing `rdx`; `imul`/`mul` capturing high `rdx` | div-by-zero (remainder) / overflow (high-multiply) |
| `andq`/`andl`/`andw`/`andb` · `orq`/`orl`/… · `xor` | reg/mem, reg/mem/imm | `and`/`or`/`xor` | — (binary bitwise; keyword-collision rule §3.2) |
| `notq`/`notl`/… · `neg` | reg/mem | `not`/`neg` | — (unary; `neg` overflow) |
| `shl` / `shr` / `sar` | reg/mem, `cl`/imm | `shl`/`shr`/`sar` | **over-width shift** → trap (Concurrency §6) |
| `cmp` / `test` | reg/mem, reg/mem/imm | `cmp`/`test` | — (sets `rflags`) |
| `cdq` / `cqo` | — | `cdq`/`cqo` | — (sign-extend `rax`→`rdx:rax` before `idiv`) |
| `push` / `pop` | reg/mem/imm | `push`/`pop` | — |
| `jmp` | label / reg/mem | `jmp` | — (raw transfer needs an `unchecked` grant, Control Flow §2) |
| `je`/`jne`/`jl`/`jle`/`jg`/`jge`/`jb`/`jae`/`jbe`/`ja` | label | `j*` | — |
| `set{cc}` | reg/mem (8-bit) | `set*` | — |
| `call` | label / reg/mem | `call` | — |
| `ret` | — | `ret` | — |
| `syscall` | — | `syscall` | — (`@abi(syscall)`, ABI §5; `unchecked`) |
| `int` | imm | `int` | — (`int 0x80` legacy syscall) |
| `nop` | — | `nop` | — |
| FP arith (SSE2) `addps`/`subps`/`mulps`/`divps`/`minps`/`maxps`/`sqrtps`; `addpd`/`subpd`/`mulpd`/`divpd`/`sqrtpd`; scalar `addss`/`addsd`/`mulss`/`mulsd` | SIMD reg/mem | same | — (lane-decorated, §3.5) |
| FP move/compare/shuffle `movaps`/`movups`/`movss`/`movsd` · `cmpps`/`cmppd` · `shufps`/`pshufd` | SIMD reg/mem, imm | same | — |
| `fltsd`/`fltss` · `feqsd`/`feqss` (**synthetic** fused float compare → `bool`, §80 §4.1) | byte dst, two FP operands | `ucomisd`/`ucomiss` + `seta` (ordered `<`) / `sete`+`setnp`+`andb` (ordered `==`) + zero-extend | — (NaN unordered → `false`, §110 §7.5; FP counterpart of `setb`/`sete`) |
| int packed `paddb`/`paddw`/`paddd`/`paddq` · `psubd` · `pand`/`por`/`pxor` | SIMD reg/mem | same | — |
| convert `cvtps2dq`/`cvtdq2ps`/`cvtss2sd`/`cvtsd2ss`/`cvttps2dq` | SIMD reg/mem | same | — |
| FMA `vfmadd213ps`/`vfmadd213pd` (**`fma`** feature) | SIMD reg | same | — (further vector ops — AVX-512 masked/broadcast, gather, crypto — via `asm()`, §intro/CG-3) |

**Keyword-collision rule.** An instruction intrinsic is an ordinary call/path identifier
(Assembly §4), so it **cannot** be a keyword. Where a bare GAS mnemonic would collide
with a keyword — `and` / `or` / `not` (the logical operators, Grammar §2.3) — the
**size-suffixed** name is the intrinsic (`andq`/`andl`/`andw`/`andb`, `orq…`, `notq…`);
these are never keywords. The bare GAS mnemonic is still what is **emitted** (the 1:1
contract holds at the GAS level, not the spelling level). Non-colliding mnemonics
(`xor`, `neg`, `add`, …) are used as-is.

Anything outside this set (string ops, BMI, full AVX-512, `cpuid`, fences such as
`mfence`, atomics — the latter via the Concurrency builtins) is reached with `asm(…)`
or the relevant builtin (Concurrency §2).

#### 3.3 Addressing modes (`at(…)`)

| Form | Fields | Example |
|------|--------|---------|
| base | `base` | `at(base = rbp)` |
| base + disp | `base`, `disp` | `at(base = rbp, disp = -16)` |
| base + index×scale + disp | `base`, `index`, `scale`∈{1,2,4,8}, `disp` | `at(base = rdi, index = rcx, scale = 8, disp = 0)` |
| RIP-relative | `base = rip`, `offset = sym` | `at(base = rip, offset = msg)` |
| absolute | `disp` | `at(disp = 0x1000)` |

#### 3.4 Relocation / TLS decorators

| Decorator | GAS reloc | Legal on |
|-----------|-----------|----------|
| `sym.plt()` | `sym@PLT` | a call/jmp symbol operand |
| `sym.gotpcrel()` | `sym@GOTPCREL` | a RIP-relative memory symbol |
| `sym.got()` | `sym@GOT` | a GOT-relative symbol |
| `sym.tpoff()` | `sym@TPOFF` | a thread-local symbol (local-exec TLS) |
| `sym.tlsgd()` | `sym@TLSGD` | a thread-local symbol (general-dynamic TLS) |

#### 3.5 Operand decorators (SIMD)

| Decorator | Meaning |
|-----------|---------|
| `v.lanes(T, N)` | interpret vector register `v` as `N` lanes of `T` (e.g. `xmm0.lanes(f32, 4)`); SSE baseline (`ymm`/`zmm` need `avx`/`avx512`) |
| `v.mask(k)` | apply AVX-512 opmask `k` (merging) — **requires `avx512`** |
| `v.mask(k).zeroing()` | apply opmask `k` with zeroing of masked-off lanes — **requires `avx512`** |

---

### 4. `i386`

Machine model: little-endian; **pointer width 32** (the architecture's native width;
Manifest §3.2 — *not* derived from `code_size`), so `ptr(T)` and a code address are
**32-bit on every i386 target, including the COM one**. `code_size` is the **instruction
encoding mode**: its default is **`b32`** (32-bit protected mode); the **`Container.com`**
target requires **`b16`** (emit 16-bit-encoded instructions). The tables below
(registers, instructions) target **32-bit protected mode**. Real-mode `segment:offset`
addressing is **not** a typed-surface concept (it does not change `ptr(T)`'s width);
a real-mode COM program works **below the typed surface** — `naked` functions, `asm`,
and the segment decorator (§4.5) — because the curated typed surface targets protected
mode.

Default ABI: **`sysv32`/cdecl** (ABI §4.3) — **except a `Container.com` target**
(`i386-none-none-com`, real mode), whose default selector is **`naked`** with no ABI
lowering (ABI §2(b)/§3.1); there C calling conventions are unavailable and the
programmer writes the frame/return.

#### 4.1 Registers

| Class | Name(s) | Width | Notes |
|-------|---------|-------|-------|
| GP | `eax` (`ax`/`al`/`ah`) | 32/16/8 | accumulator; int result (with `edx`) |
| GP | `ebx` (`bx`/`bl`/`bh`) | 32/16/8 | callee-saved |
| GP | `ecx` (`cx`/`cl`/`ch`) | 32/16/8 | shift count (`cl`) |
| GP | `edx` (`dx`/`dl`/`dh`) | 32/16/8 | `edx:eax` for wide mul/div; 2nd return word |
| GP | `esi` (`si`) | 32/16 | callee-saved |
| GP | `edi` (`di`) | 32/16 | callee-saved |
| GP | `ebp` (`bp`) | 32/16 | frame pointer (callee-saved) |
| GP | `esp` (`sp`) | 32/16 | stack pointer |
| PC | `eip` | 32 | instruction pointer |
| flags | `eflags` | 32 | status flags |
| x87 | `st0 … st7` | 80 | x87 FP stack (scalar FP / `sysv32` float result in `st0`) |
| FP/SIMD | `xmm0 … xmm7` | 128 | SSE (when the `sse` feature is present) |
| segment | `cs`/`ds`/`es`/`ss`/`fs`/`gs` | 16 | segment selectors (real-mode / segmented; `gs` for TLS) |

#### 4.2 Instructions (curated v1)

| Intrinsic | Operands | GAS | Checked-guard |
|-----------|----------|-----|---------------|
| `mov` (`movl`/`movw`/`movb`) | reg/mem, reg/mem/imm | `mov` | — |
| `movzx` / `movsx` | reg, reg/mem | `movz*`/`movs*` | — |
| `lea` | reg, mem | `lea` | — |
| `add` / `sub` | reg/mem, reg/mem/imm | `add`/`sub` | overflow set (Concurrency §6) |
| `imul` / `mul` | reg/mem (, reg, imm) | `imul`/`mul` | overflow |
| `idiv` / `div` | reg/mem | `idiv`/`div` | **division by zero** (and `MIN/-1`) → the inserted guard traps *before* the instruction (§2) |
| `andl`/`andw`/`andb` · `orl`/… · `xor` | reg/mem, reg/mem/imm | `and`/`or`/`xor` | — (binary bitwise; keyword-collision rule §3.2) |
| `notl`/`notw`/`notb` · `neg` | reg/mem | `not`/`neg` | — (unary; `neg` overflow) |
| `shl` / `shr` / `sar` | reg/mem, `cl`/imm | `shl`/`shr`/`sar` | **over-width shift** → trap |
| `cmp` / `test` | reg/mem, reg/mem/imm | `cmp`/`test` | — (sets `eflags`) |
| `cdq` | — | `cdq` | — (sign-extend `eax`→`edx:eax` before `idiv`) |
| `push` / `pop` | reg/mem/imm | `push`/`pop` | — |
| `jmp` | label / reg/mem | `jmp` | — (raw transfer needs an `unchecked` grant, Control Flow §2) |
| `je`/`jne`/`jl`/`jle`/`jg`/`jge`/`jb`/`jae`/`jbe`/`ja` | label | `j*` | — |
| `set{cc}` | reg/mem (8-bit) | `set*` | — |
| `call` | label / reg/mem | `call` | — |
| `ret` | — | `ret` | — |
| `int` | imm | `int` | — (`int 0x80` legacy syscall, ABI §5; `unchecked`) |
| `nop` | — | `nop` | — |
| SSE FP (**`sse`**) `movaps`/`movups`/`addps`/`subps`/`mulps`/`divps`/`sqrtps`; scalar `addss`/`mulss` | SIMD reg/mem | same | — (lane-decorated, §4.5) |
| SSE compare/shuffle/convert (**`sse`**) `cmpps` · `shufps` · `cvtps2dq`/`cvtdq2ps` | SIMD reg/mem, imm | same | — |
| x87 scalar `fld`/`fst`/`fadd`/`fmul`/`fdiv`/`fsqrt` | `st(i)`/mem | same | — (x87 FP stack; default scalar FP without `sse`) |

Segment overrides, far jumps/calls, string ops, BMI, x87 control, and 16-bit real-mode
specifics are reached with `asm(…)` (Assembly §4). The bare-mnemonic / keyword-collision
rule of §3.2 applies (`and`/`or`/`not` → `andl`/`orl`/`notl`).

#### 4.3 Addressing modes (`at(…)`)

| Form | Fields | Example |
|------|--------|---------|
| base | `base` | `at(base = ebp)` |
| base + disp | `base`, `disp` | `at(base = ebp, disp = -8)` |
| base + index×scale + disp | `base`, `index`, `scale`∈{1,2,4,8}, `disp` | `at(base = edi, index = ecx, scale = 4, disp = 0)` |
| absolute (immediate) | `disp` | `at(disp = 0x7C00)` |
| absolute (symbol) | `offset` | `at(offset = msg)` |

(No RIP-relative addressing — that is `x86_64` only; i386 uses absolute or
base-relative.) The `at(…)` field set is the fixed schema of Assembly §7
(`base`/`index`/`scale`/`disp`/`offset`) — a **segment register is not an addressing
`base`**. A **segment override** (e.g. `gs`-relative TLS) is a **per-arch operand
decorator** (Assembly §8), `op.seg(gs)`, applied to a `at(…)` operand and composing
with a TLS relocation decorator (§4.4):

```
at(offset = tls_var).seg(gs)         # %gs:tls_var ; with tls_var.tpoff() → %gs:tls_var@NTPOFF
```

#### 4.4 Relocation / TLS decorators

| Decorator | GAS reloc | Legal on |
|-----------|-----------|----------|
| `sym.plt()` | `sym@PLT` | a call/jmp symbol operand |
| `sym.got()` | `sym@GOT` | a GOT-slot symbol |
| `sym.gotoff()` | `sym@GOTOFF` | a GOT-relative data symbol |
| `sym.tlsgd()` | `sym@TLSGD` | a thread-local symbol (general-dynamic TLS) |
| `sym.tpoff()` | `sym@NTPOFF` | a thread-local symbol (local-exec, `gs`-relative) |

#### 4.5 Operand decorators (SIMD, segment)

| Decorator | Meaning |
|-----------|---------|
| `v.lanes(T, N)` | interpret `xmm` register `v` as `N` lanes of `T` (e.g. `xmm0.lanes(f32, 4)`); requires the `sse` feature |
| `op.seg(s)` | apply a **segment override** `s` ∈ {`cs`,`ds`,`es`,`ss`,`fs`,`gs`} to a memory operand `op` (e.g. `gs`-relative TLS, §4.3); lowers to the `%s:` prefix |

(AVX-512 opmask decorators `mask`/`zeroing` exist only with the AVX-512 feature, as on
`x86_64` §3.5 — additive per-arch data, FND-6.)

### 5. `aarch64`

Machine model: 64-bit, little-endian; pointer width 64. Default ABI: **`aapcs64`**
(ABI §4.4; the Apple variant differs in varargs and **reserves `x18`**, §5.1). Supported
on Linux, macOS, and freestanding (v1 does **not** include Windows-on-ARM, §1). A **load/store architecture**
— memory is touched only by `ldr`/`str`-family instructions, never by arithmetic
operands.

#### 5.1 Registers

| Class | Name(s) | Width | Notes |
|-------|---------|-------|-------|
| GP | `x0 … x7` (`w0 … w7`) | 64/32 | argument/result registers (AAPCS64) |
| GP | `x8` (`w8`) | 64/32 | indirect-result address (ABI §4.4) |
| GP | `x9 … x15` | 64/32 | caller-saved temporaries |
| GP | `x16`/`x17` (`ip0`/`ip1`) | 64 | intra-procedure-call scratch |
| GP | `x18` | 64 | **platform register**: caller-saved temporary on **Linux only**; **reserved** on macOS/Apple and on freestanding `Os.none` (ABI §4.4) |
| GP | `x19 … x28` (`w19 … w28`) | 64/32 | callee-saved |
| GP | `x29` (`fp`) | 64 | frame pointer |
| GP | `x30` (`lr`) | 64 | link register (return address) |
| GP | `sp` | 64 | stack pointer (16-byte aligned) |
| GP | `xzr`/`wzr` | 64/32 | zero register (reads 0; writes discarded) |
| PC | `pc` | 64 | program counter (PC-relative addressing only) |
| flags | `nzcv` | — | condition flags (set by `cmp`/`*s` instructions) |
| FP/SIMD | `v0 … v31` (`b`/`h`/`s`/`d`/`q` views) | 8/16/32/64/128 | **NEON — baseline** (mandatory in AArch64; always present, not a `features` opt-in); `v0…v7` FP args, `v8…v15` callee-saved (low 64) |
| SVE | `z0 … z31`, `p0 … p15` | scalable | vector + predicate registers (only with the `sve` feature) |

#### 5.2 Instructions (curated v1)

| Intrinsic | Operands | GAS | Checked-guard |
|-----------|----------|-----|---------------|
| `mov` / `movz` / `movk` / `movn` | reg, reg/imm | `mov`/`movz`/`movk`/`movn` | — (wide-immediate construction) |
| `ldr` (`ldrb`/`ldrh`/`ldrsw`/…) | reg, mem | `ldr*` | — (load; mem = §5.3) |
| `str` (`strb`/`strh`/…) | reg, mem | `str*` | — (store) |
| `ldp` / `stp` | reg, reg, mem | `ldp`/`stp` | — (load/store pair) |
| `add` / `sub` | reg, reg, reg/imm | `add`/`sub` | overflow set (Concurrency §6) on the checked op |
| `mul` / `madd` / `msub` | reg, reg, reg (, reg) | `mul`/`madd`/`msub` | overflow |
| `sdiv` / `udiv` | reg, reg, reg | `sdiv`/`udiv` | **division by zero** → trap (hardware returns 0; Alatyr inserts the guard, Concurrency §6) |
| `andx`/`andw` · `orr` · `eor` · `bic` · `orn` | reg, reg, reg/imm | `and`/`orr`/`eor`/`bic`/`orn` | — (binary bitwise; `and` keyword-collision → `andx`/`andw`, §3.2) |
| `mvn` | reg, reg/imm | `mvn` | — (unary bitwise NOT) |
| `lsl` / `lsr` / `asr` / `ror` | reg, reg, reg/imm | `lsl`/`lsr`/`asr`/`ror` | **over-width shift** → trap (Concurrency §6) |
| `cmp` / `cmn` / `tst` | reg, reg/imm | `cmp`/`cmn`/`tst` | — (sets `nzcv`) |
| `csel` / `cset` / `csinc` | reg, … , cond | `csel`/`cset`/`csinc` | — (conditional select/set) |
| `adr` / `adrp` | reg, label/sym | `adr`/`adrp` | — (PC-relative address; `adrp`+`:lo12:` §5.4) |
| `b` | label | `b` | — (unconditional branch; raw transfer needs `unchecked`, Control Flow §2) |
| `b.eq`/`b.ne`/`b.lt`/`b.le`/`b.gt`/`b.ge`/`b.hi`/`b.hs`/`b.lo`/`b.ls` | label | `b.*` | — (conditional branch) |
| `cbz` / `cbnz` / `tbz` / `tbnz` | reg (, imm), label | `cbz`/… | — (compare-and-branch) |
| `bl` | label | `bl` | — (branch-with-link = call) |
| `br` / `blr` | reg | `br`/`blr` | — (indirect branch / call) |
| `ret` | (reg) | `ret` | — (return; default `x30`) |
| `svc` | imm | `svc` | — (`svc #0` syscall, ABI §5; `unchecked`) |
| `nop` | — | `nop` | — |
| FP/NEON arith `fadd`/`fsub`/`fmul`/`fdiv`/`fmin`/`fmax`/`fsqrt`/`fabs`/`fneg`; fused `fmadd`/`fmsub` | SIMD/FP reg | same | — (lane-decorated, §5.5) |
| FP/NEON move/compare/convert `fmov`/`fcmp`/`fcvt`/`fcvtzs`/`scvtf` · `dup`/`ins` | SIMD/FP reg | same | — |
| NEON int (vector forms) `add`/`sub`/`mul`/`and`/`orr`/`eor` · `ld1`/`st1` | SIMD reg, mem | same | — (`ld1`/`st1` = structured load/store; further NEON via `asm()`, CG-3) |

Outside this set (atomics — via the Concurrency builtins; `dmb`/`dsb`/`isb` barriers;
full SVE; system-register `mrs`/`msr`; crypto) is reached with `asm(…)` (Assembly §4).
The keyword-collision rule (§3.2) applies to **`and`** only — AArch64 spells the others
`orr`/`eor`/`mvn`/`bic`, which are not keywords.

#### 5.3 Addressing modes (`at(…)`)

| Form | Fields | Example |
|------|--------|---------|
| base | `base` | `at(base = x0)` |
| base + imm | `base`, `disp` | `at(base = x0, disp = 16)` |
| base + reg | `base`, `index` | `at(base = x0, index = x1)` |
| base + reg, scaled (LSL) | `base`, `index`, `scale` | `at(base = x0, index = x1, scale = 8)` |

(Pre-/post-index forms `[Xn, #i]!` / `[Xn], #i` modify the base and are reached with
`asm(…)` in v1. AArch64 has **no** segment registers; PC-relative addressing uses
`adr`/`adrp` (§5.2), not a `at(…)` form.)

#### 5.4 Relocation / TLS decorators

| Decorator | GAS reloc | Legal on |
|-----------|-----------|----------|
| `sym.lo12()` | `:lo12:sym` | the `add`/`ldr` low-12 after an `adrp` |
| `sym.got()` | `:got:sym` | a GOT-page symbol (with `adrp`) |
| `sym.got_lo12()` | `:got_lo12:sym` | the GOT low-12 (`ldr` after `adrp`) |
| `sym.tprel()` | `:tprel_lo12:sym` | a thread-local symbol (local-exec TLS) |
| `sym.tlsdesc()` | `:tlsdesc:sym` | a thread-local symbol (TLS descriptor) |

#### 5.5 Operand decorators (SIMD)

| Decorator | Meaning |
|-----------|---------|
| `v.lanes(T, N)` | interpret `v` register as `N` lanes of `T` (e.g. `v0.lanes(f32, 4)` ≡ the `.4s` arrangement) |
| `z.pred(p)` | apply SVE predicate `p` (merging) to a scalable-vector op (only with the `sve` feature) |
| `z.pred(p).zeroing()` | apply predicate `p` with zeroing of inactive lanes (SVE) |

### 6. `aarch32`

Machine model: 32-bit, little-endian; pointer width 32. Default ABI: **`aapcs32`**, in
the **soft-float** (`Env.eabi`) or **hard-float** (`Env.eabihf`) sub-variant (ABI §4.5).
The hard-float ABI **requires the `vfp` feature** (FP args go in `s`/`d`, which exist
only with `vfp`); `Env.eabihf` without `vfp` is a Config diagnostic. Soft-float
(`Env.eabi`) uses no VFP registers.
Every supported `aarch32` triple carries `Env.eabi` or `Env.eabihf` (incl. freestanding
`Os.none`; §1) — there is no `Env.none` `aarch32` target. Instruction-encoding mode `arm` or `thumb` (`arm_mode`, Manifest §3.2). A
**load/store architecture**.

#### 6.1 Registers

| Class | Name(s) | Width | Notes |
|-------|---------|-------|-------|
| GP | `r0 … r3` | 32 | argument/result registers; caller-saved scratch |
| GP | `r4 … r11` | 32 | callee-saved |
| GP | `r12` (`ip`) | 32 | intra-procedure scratch (caller-saved) |
| GP | `sp` (`r13`) | 32 | stack pointer (8-byte aligned at a public interface) |
| GP | `lr` (`r14`) | 32 | link register (return address) |
| GP | `pc` (`r15`) | 32 | program counter |
| flags | `cpsr` | 32 | program status (condition flags `NZCV`) |
| VFP | `s0 … s31` | 32 | single-precision FP (hard-float FP args); **only with the `vfp` feature** |
| VFP | `d0 … d31` | 64 | double-precision FP (`d0…d15` alias `s0…s31`); `d8…d15` callee-saved; **only with `vfp`** |
| NEON | `q0 … q15` | 128 | 128-bit vector (aliases `d` pairs); **only with the `neon` feature** (`neon` implies `vfp`) |

#### 6.2 Instructions (curated v1)

| Intrinsic | Operands | GAS | Checked-guard |
|-----------|----------|-----|---------------|
| `mov` / `movw` / `movt` | reg, reg/imm | `mov`/`movw`/`movt` | — (immediate construction) |
| `ldr` (`ldrb`/`ldrh`/`ldrsb`/`ldrsh`) | reg, mem | `ldr*` | — (load; mem = §6.3) |
| `str` (`strb`/`strh`) | reg, mem | `str*` | — (store) |
| `ldm` / `stm` · `push` / `pop` | `[reg, …]` (, mem) | `ldm`/`stm`/`push`/`pop` | — (a register list is an ordinary **array of register operands**, Assembly §5 — no new operand kind) |
| `add` / `sub` / `rsb` | reg, reg, reg/imm | `add`/`sub`/`rsb` | overflow set (Concurrency §6) on the checked op |
| `mul` / `mla` / `mls` | reg, reg, reg (, reg) | `mul`/`mla`/`mls` | overflow |
| `sdiv` / `udiv` | reg, reg, reg | `sdiv`/`udiv` | requires the **`idiv`** feature (hardware integer divide; not universal across ARMv7 profiles) — **unavailable (a diagnostic) without it**; **division by zero** → trap (Concurrency §6) |
| `and32` · `orr` · `eor` · `bic` · `orn` | reg, reg, reg/imm | `and`/`orr`/`eor`/`bic`/`orn` | — (binary bitwise; `and` keyword-collision → `and32`, §3.2) |
| `mvn` | reg, reg/imm | `mvn` | — (unary bitwise NOT) |
| `lsl` / `lsr` / `asr` / `ror` | reg, reg, reg/imm | `lsl`/`lsr`/`asr`/`ror` | **over-width shift** → trap (Concurrency §6) |
| `cmp` / `cmn` / `tst` / `teq` | reg, reg/imm | `cmp`/`cmn`/`tst`/`teq` | — (sets `cpsr`) |
| `b` | label | `b` | — (branch; raw transfer needs `unchecked`, Control Flow §2) |
| `b.eq`/`b.ne`/`b.lt`/`b.le`/`b.gt`/`b.ge`/`b.hi`/`b.hs`/`b.lo`/`b.ls` | label | `b{cond}` | — (conditional branch) |
| `bl` | label | `bl` | — (branch-with-link = call) |
| `bx` / `blx` | reg / label | `bx`/`blx` | — (branch-exchange, ARM/Thumb interworking; `bx lr` = return) |
| `svc` | imm | `svc` | — (`svc #0` syscall, ABI §5; `unchecked`) |
| VFP (**`vfp`**) `vmov`/`vldr`/`vstr` · `vadd`/`vsub`/`vmul`/`vdiv`/`vsqrt`/`vabs`/`vneg` · `vcmp`/`vcvt` | FP reg, mem | same | — (lane-decorated, §6.5) |
| NEON (**`neon`** ⇒ `vfp`) vector `vadd`/`vsub`/`vmul` · `vld1`/`vst1` | SIMD reg, mem | same | — (further NEON via `asm()`, CG-3) |
| `nop` | — | `nop` | — |

The high-level integer division **operator** (`/`) is distinct from the `sdiv`/`udiv`
intrinsic: with the `idiv` feature it lowers to that intrinsic; **without** it, the
operator lowers to a **runtime helper** (a codegen/stdlib lowering, not an instruction —
the helper still enforces the div-by-zero trap). The instruction intrinsic itself is
simply **unavailable** on a core without `idiv`.

Return is `bx(lr)` (there is no dedicated `ret`). Conditional execution on arbitrary
instructions (ARM `{cond}` suffixes, Thumb `IT` blocks), barriers (`dmb`/`dsb`/`isb`),
coprocessor/system access, and atomics (via the Concurrency builtins) are reached with
`asm(…)` (Assembly §4). The keyword-collision rule (§3.2) applies to **`and`** only —
AArch32 spells the others `orr`/`eor`/`mvn`/`bic`/`orn`, which are not keywords; with a
single 32-bit width the escape is `and32`.

#### 6.3 Addressing modes (`at(…)`)

| Form | Fields | Example |
|------|--------|---------|
| base | `base` | `at(base = r0)` |
| base + imm | `base`, `disp` | `at(base = r0, disp = 8)` |
| base + reg | `base`, `index` | `at(base = r0, index = r1)` |
| base + reg, shifted (LSL) | `base`, `index`, `scale` | `at(base = r0, index = r1, scale = 4)` |

(Pre-/post-index forms `[Rn, #i]!` / `[Rn], #i` modify the base and are reached with
`asm(…)` in v1. PC-relative literal-pool loads (`ldr r0, =sym`) use the `adr`/literal
mechanism, not a `at(…)` form. AArch32 has **no** segment registers.)

#### 6.4 Relocation / TLS decorators

| Decorator | GAS reloc | Legal on |
|-----------|-----------|----------|
| `sym.plt()` | `sym(PLT)` | a call/branch symbol operand |
| `sym.got()` | `sym(GOT)` | a GOT-slot symbol |
| `sym.gotoff()` | `sym(GOTOFF)` | a GOT-relative data symbol |
| `sym.tlsgd()` | `sym(TLSGD)` | a thread-local symbol (general-dynamic TLS) |
| `sym.tpoff()` | `sym(TPOFF)` | a thread-local symbol (local-exec TLS) |

#### 6.5 Operand decorators (SIMD)

| Decorator | Meaning |
|-----------|---------|
| `v.lanes(T, N)` | interpret a `d`/`q` register as `N` lanes of `T` (e.g. `q0.lanes(f32, 4)` ≡ the `.4s` arrangement); requires the `neon` feature |

(AArch32 has no predicate registers and no segment override — its operand-decorator set
is lane decorations only.)

### 7. `riscv64`

Machine model: 64-bit, little-endian; pointer width 64 (XLEN = 64). Default ABI:
**`lp64`** (soft), **`lp64f`** (`f` only), or **`lp64d`** (`d`) — by the widest enabled
FP feature (ABI §2(c)/§4.6). Base ISA **RV64I**; common extensions are **feature-gated**
(`m` integer mul/div, `f`/`d` FP, `v` vector). A **load/store architecture** with **no condition-flags register**
(compare-and-branch instead).

#### 7.1 Registers

Register names are the **ABI names** (the `xN` numbers are the alternate spelling).

| Class | Name(s) | Width | Notes |
|-------|---------|-------|-------|
| GP | `zero` (`x0`) | 64 | hard-wired zero (reads 0; writes discarded) |
| GP | `ra` (`x1`) | 64 | return address |
| GP | `sp` (`x2`) | 64 | stack pointer (16-byte aligned) |
| GP | `gp` (`x3`) | 64 | global pointer |
| GP | `tp` (`x4`) | 64 | thread pointer (TLS base) |
| GP | `t0 … t2` (`x5…x7`), `t3 … t6` (`x28…x31`) | 64 | temporaries (caller-saved) |
| GP | `s0`/`fp` (`x8`), `s1` (`x9`), `s2 … s11` (`x18…x27`) | 64 | callee-saved (`s0` = frame pointer) |
| GP | `a0 … a7` (`x10…x17`) | 64 | argument/result registers (`a0`/`a1` = results) |
| PC | `pc` | 64 | program counter |
| FP | `ft0 … ft7`, `ft8 … ft11` (`f0…f7`, `f28…f31`) | FLEN (32 with `f`, 64 with `d`) | FP temporaries (caller-saved); **only with the `f`/`d` feature** |
| FP | `fs0 … fs11` (`f8/f9`, `f18…f27`) | FLEN (32/64) | FP callee-saved; **only with `f`/`d`** |
| FP | `fa0 … fa7` (`f10…f17`) | FLEN (32/64) | FP argument/result registers; **only with `f`/`d`** |
| vector | `v0 … v31` | scalable | vector registers (only with the `v` feature) |

#### 7.2 Instructions (curated v1)

| Intrinsic | Operands | GAS | Checked-guard |
|-----------|----------|-----|---------------|
| `lui` / `auipc` | reg, imm/sym | `lui`/`auipc` | — (upper-immediate / PC-relative) |
| `addi` / `add` / `sub` | reg, reg, reg/imm | `addi`/`add`/`sub` | overflow set (Concurrency §6) on the checked op |
| `addiw` / `addw` / `subw` | reg, reg, reg/imm | `addiw`/`addw`/`subw` | overflow (32-bit word ops on RV64) |
| `and64` · `or64` · `xor` · `andi` · `ori` · `xori` | reg, reg, reg/imm | `and`/`or`/`xor`/`andi`/`ori`/`xori` | — (bitwise; `and`/`or` keyword-collision → `and64`/`or64`, §3.2) |
| `not64` | reg, reg | `not` (pseudo `xori rd, rs, -1`) | — (unary; pseudo; keyword-collision §3.2) |
| `sll` / `srl` / `sra` (`slli`/…/`sllw`/…) | reg, reg, reg/imm | `sll`/`srl`/`sra`/… | **over-width shift** → trap (Concurrency §6) |
| `slt` / `sltu` / `slti` / `sltiu` | reg, reg, reg/imm | `slt`/… | — (set-less-than) |
| `lb`/`lh`/`lw`/`ld`/`lbu`/`lhu`/`lwu` | reg, mem | `lb`/… | — (load; mem = §7.3) |
| `sb`/`sh`/`sw`/`sd` | reg, mem | `sb`/… | — (store) |
| `beq`/`bne`/`blt`/`bge`/`bltu`/`bgeu` | reg, reg, label | `beq`/… | — (compare-and-branch; no flags) |
| `jal` / `jalr` | reg, label / reg, reg, imm | `jal`/`jalr` | — (jump-and-link = call; raw transfer needs `unchecked`) |
| `j` / `ret` / `call` / `tail` | label / — / sym | `j`/`ret`/`call`/`tail` | — (pseudos; `ret` = `jalr zero, ra, 0`) |
| `mul` / `mulh` / `div` / `divu` / `rem` / `remu` | reg, reg, reg | same | requires the **`m`** feature — **unavailable (a diagnostic) without it**; `div`/`rem` **div-by-zero** → trap (HW yields −1/dividend; Alatyr inserts the guard, Concurrency §6) |
| FP single (**`f`**) `fadd.s`/`fsub.s`/`fmul.s`/`fdiv.s`/`fsqrt.s`/`fmin.s`/`fmax.s` · `flw`/`fsw`/`fmv.s` · `fmadd.s` · `fcvt.w.s`/`fcvt.s.w` · `feq.s`/`flt.s`/`fle.s` | FP reg, … | same | — (float div0 = ±∞, Concurrency §7) |
| FP double (**`d`**) `fadd.d`/`fsub.d`/`fmul.d`/`fdiv.d`/`fsqrt.d` · `fld`/`fsd`/`fmv.d` · `fmadd.d` · `fcvt.*`/`feq.d`/`flt.d`/`fle.d` | FP reg, … | same | — (`d` ⇒ `f`; vector ops via the `v` ext through `asm()`, CG-3) |
| `ecall` / `ebreak` | — | `ecall`/`ebreak` | — (`ecall` syscall, ABI §5; `unchecked`) |
| `nop` | — | `nop` | — (pseudo `addi zero, zero, 0`) |

The high-level integer `*`/`/`/`%` **operators** use the `m`-extension intrinsics when
`m` is present; **without `m`**, they lower to **runtime helpers** (a codegen/stdlib
lowering, not an instruction). The helper preserves **all** checked guards of the
operator (Concurrency §6): multiplication overflow, the `MIN / -1` overflow case, and
division/remainder by zero — each traps exactly as the in-hardware path would. The
`mul`/`div`/`rem` **intrinsics** themselves are simply **unavailable** without `m`.
Atomics (`a` extension — via the Concurrency builtins), fences (`fence`), CSR access,
and the full vector (`v`) ISA are reached with `asm(…)` (Assembly §4). The
keyword-collision rule (§3.2) applies to **`and`/`or`/`not`** → `and64`/`or64`/`not64`;
`xor`/`andi`/`ori`/`xori` are not keywords.

#### 7.3 Addressing modes (`at(…)`)

| Form | Fields | Example |
|------|--------|---------|
| base + imm12 | `base`, `disp` | `at(base = sp, disp = -16)` |
| base | `base` | `at(base = a0)` (≡ `disp = 0`) |

RISC-V has **only** base + signed-12-bit-immediate addressing — **no** index, scale, or
segment. A symbol address is formed with `lui`+`addi` (`%hi`/`%lo`, medlow) or
`auipc`+`addi`/load (`%pcrel_hi`/`%pcrel_lo`, medany), §7.4 — not a `at(…)` form.

#### 7.4 Relocation / TLS decorators

| Decorator | GAS reloc | Legal on |
|-----------|-----------|----------|
| `sym.hi()` | `%hi(sym)` | the `lui` upper-20 (absolute, medlow) |
| `sym.lo()` | `%lo(sym)` | the `addi`/load lower-12 (absolute) |
| `sym.pcrel_hi()` | `%pcrel_hi(sym)` | the `auipc` upper-20 (PC-relative, medany) |
| `label.pcrel_lo()` | `%pcrel_lo(label)` | the lower-12 paired with a `pcrel_hi` label |
| `sym.tprel_hi()` / `sym.tprel_lo()` | `%tprel_hi(sym)` / `%tprel_lo(sym)` | thread-local (local-exec TLS) |
| `sym.tls_gd()` | `%tls_gd(sym)` | thread-local (general-dynamic TLS) |

#### 7.5 Operand decorators (SIMD)

| Decorator | Meaning |
|-----------|---------|
| `v.lanes(T, N)` | interpret a vector register `v` as `N` lanes of `T`; requires the `v` feature (the vector `vsetvl`/LMUL configuration is otherwise reached with `asm(…)`) |

(RISC-V has no segment registers; the operand-decorator set is lane decorations only.)

### 8. `riscv32`

Machine model: 32-bit, little-endian; pointer width 32 (**XLEN = 32**). Default ABI:
**`ilp32`** (soft) / **`ilp32f`** (`f`) / **`ilp32d`** (`d`) — by the widest enabled FP
feature (ABI §2(c)/§4.6). Base ISA **RV32I**; the same feature gates and implication
graph as `riscv64` (`m`/`f`/`d`/`v`/`a`/`c`; `d` ⇒ `f`). A load/store architecture with
no condition-flags register. **Identical to `riscv64` (§7) except for XLEN = 32**, with
the deltas below.

#### 8.1 Registers

The register file and ABI roles are **exactly those of `riscv64` §7.1**, except: GP
registers are **32-bit** (XLEN = 32; `sp` 16-byte aligned). FP registers (`ft`/`fs`/`fa`,
only with `f`/`d`) keep their **FLEN** width (32 with `f`, 64 with `d`) — on `riscv32`
with `d`, FLEN = 64 is **wider than XLEN = 32**. The vector file (`v0…v31`, only with
`v`) is unchanged.

#### 8.2 Instructions (curated v1)

The curated set is **`riscv64` §7.2 minus the 64-bit-only forms**:

- **removed** — the RV64 word ops `addiw`/`addw`/`subw` and word shifts `sllw`/`srlw`/
  `sraw` (on RV32 the base `add`/`addi`/`sub`/`sll`/… already operate on the 32-bit
  XLEN), and the 64-bit memory ops `ld`/`lwu`/`sd`. The widest load/store are `lw`/`sw`
  (XLEN = 32).
- **keyword-collision escapes** are `and32`/`or32`/`not32` (XLEN = 32; §3.2) — emitting
  `and`/`or`/`not`.
- everything else — `lui`/`auipc`, `addi`/`add`/`sub`, `andi`/`ori`/`xori`/`xor`,
  `sll`/`srl`/`sra`, `slt`/…, `lb`/`lh`/`lw`/`lbu`/`lhu`/`sb`/`sh`/`sw`, branches,
  `jal`/`jalr` + pseudos, `mul`/`div`/`rem` (under `m`; the `*`/`/`/`%` operator
  helper-fallback and the `div`-by-zero/overflow guards as in §7.2), FP under `f`/`d`,
  `ecall`/`ebreak`, `nop` — is **as in `riscv64` §7.2**.

#### 8.3 Addressing modes (`at(…)`)

**Identical to `riscv64` §7.3**: only base + signed-12-bit-immediate; no index, scale,
or segment; symbol addresses via `lui`/`auipc` + relocation (§8.4).

#### 8.4 Relocation / TLS decorators

**Identical to `riscv64` §7.4** (`%hi`/`%lo`/`%pcrel_hi`/`%pcrel_lo`/`%tprel_hi`/
`%tprel_lo`/`%tls_gd`).

#### 8.5 Operand decorators (SIMD)

**Identical to `riscv64` §7.5** (vector `v.lanes(T, N)` under the `v` feature; no segment
override).

---

### 9. Conformance

A conforming implementation MUST:

1. accept exactly the **supported targets** of §1 (the closed canonical-triple set) and
   reject any other `arch`×`os`×`env`×`container` `Target` with a Config diagnostic at
   configuration (Manifest §3.1; ABI §2); treat new triples as additive (FND-6);
2. for each supported architecture, provide the **five tables** of §2 — registers,
   curated instructions (each 1:1 with its GAS mnemonic and carrying its checked-guards),
   addressing modes, relocation/TLS decorators, and **operand decorators** (SIMD +
   arch-specific, e.g. x86 segment override) — populating the Assembly chapter schema
   (Assembly §4–§8);
3. emit each curated instruction **1:1** with the listed GAS mnemonic (I1; Assembly §3),
   apply the listed **checked-guards** (division-by-zero, over-width shift, overflow set;
   Concurrency §6), emit the exact §2 **checked-failure trap** for every direct-trap guard
   (including `@require`, Types §8.1), and require an `unchecked` grant for raw control
   transfer (Control Flow §2);
4. make any ISA construct **not** in the curated set reachable via `asm(…)` (lower-layer
   completeness, I4; Assembly §4) — the curated set is not the whole ISA;
5. treat every populated table here as **required v1 content** — additive (FND-6) only for
   future growth, not optional (Overview §6).
