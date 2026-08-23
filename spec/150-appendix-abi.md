# Alatyr Language Specification

## Appendix — ABIs per target

> **Status — draft (under review).** This appendix is a draft and is not yet
> accepted; the specification as a whole is under development. See the status note
> at the top of the Overview chapter.

This appendix fixes the **normative v1 ABI content** the Functions chapter defers to:
Functions §6 fixes that an ABI is a **comptime value** describing a calling convention,
that the **default** ABI comes from the target, that `@abi(value)` overrides it per
site, that the prelude provides ABI **values** (`c`/`sysv`/`syscall`/…), and that
`@abi(naked)` emits no frame. This appendix enumerates **what those values are** for
each v1 target (the 6 architectures of Assembly §10) — register assignments, saved-set,
stack discipline, aggregate passing, variadics, and the syscall convention.

Each ABI here is the platform's **canonical published convention** (System V AMD64,
AAPCS64, the RISC-V psABI, the Windows x64 convention, the System V IA-32 convention,
and the ARM AAPCS); this appendix records the part an implementation needs to lower
Alatyr's passing modes (Functions §4) **without guessing**, and adopts the canonical
document by reference for the remaining encoding detail.

Register **names** are the arch-surface names of the per-arch register tables (Assembly
§6, per-arch appendix); this appendix references them.

---

### 1. The ABI value

An ABI value (a prelude ABI, or a user-defined one built by `Abi(...)` struct
construction, Functions §6.3) has the following **schema**. It is the data a call site
and a function prologue/epilogue are lowered against (Functions §4; I1).

```alatyr
Abi := struct {
  name              : str                       # "sysv" / "c" / "win64" / "aapcs64" / "syscall" / …
  int_arg_regs      : [Reg]                      # integer/pointer argument registers, in order
  fp_arg_regs       : [Reg]          = []        # floating-point / vector argument registers, in order
  int_ret_regs      : [Reg]                      # integer/pointer result registers
  fp_ret_regs       : [Reg]          = []        # floating-point result registers
  indirect_result   : Option(Reg)    = Option.None  # register holding the address of a memory result
  number_reg        : Option(Reg)    = Option.None  # syscall-number register — set only for a syscall convention (§5); None = an ordinary call convention
  callee_saved      : [Reg]                      # registers the callee MUST preserve
  caller_saved      : [Reg]                      # registers the caller MUST assume clobbered
  stack_align       : u32                        # required stack alignment (bytes) at a call boundary
  stack_grows_down  : bool           = true      # true on every v1 target
  red_zone          : u32            = 0         # bytes below SP usable without adjustment (0 = none)
  shadow_space      : u32            = 0         # bytes the caller reserves for the callee (Win64 = 32)
  aggregate_max_bytes : u32                      # an aggregate larger than this is passed by reference (Functions §4.1)
  varargs           : VarargsRule    = VarargsRule.none
}
Reg         := str                   # an arch-surface register name (Assembly §6)
VarargsRule := enum { none, sysv_al, win64_dual, stack_after_named, aapcs }
```

`Reg` is a register **of the target's architecture** (an ABI is target-specific). The
passing-mode lowering of Functions §4 (small `in` by value in `int_arg_regs`/
`fp_arg_regs`; large `in` by reference to a copy; `out`/`in out` by reference; small
`out` in result registers) reads exactly these fields; `aggregate_max_bytes` is the
"size threshold" Functions §4.2 leaves to the target ABI.

**No `name_mangling` field — by design.** The `Abi` value describes only the **calling
convention** (register/stack passing), which is orthogonal to how a function's **linker
symbol** is spelled. Symbol naming is a **Modules** concern: an exported declaration's symbol
is its declaration spelling qualified by its module path, each `::` rendered as `__` (a valid C
identifier — Modules §6.1), and importing a foreign symbol under an exact name is
`@extern("exact_symbol")` (Modules §7.2). Platform symbol decorations (a macOS leading `_`, a
Windows `@N` stdcall suffix) are an **object-format / target** detail applied uniformly at
emission, not a per-`Abi` knob — so C interop needs no mangling field on the ABI value.

**Reading the per-target tables (§4).** Each table lists only the fields whose value
**differs from the schema default** above; an unlisted field takes its default
(`stack_grows_down = true`, `red_zone = 0`, `shadow_space = 0`,
`indirect_result = Option.None`, `fp_arg_regs`/`fp_ret_regs = []`,
`varargs = VarargsRule.none`). So, e.g., `i386`/`aarch32`/`riscv*` have no
`shadow_space` row because it is `0`, and every target has `stack_grows_down = true`
without a row. Every ABI value is nonetheless **fully determined** (table ∪ defaults).

---

### 2. Default ABI selection

Not every `(Machine` variant`, arch)` pair is a real platform: e.g.
`aarch64` on `Machine.Windows`, or `riscv64` on `Machine.Macos`, are **not supported
targets**. The closed **set of supported targets** — the `(Machine` variant`, arch)` pairs
— is enumerated in the **per-arch appendix §1** (Manifest §3.1); selecting a pair outside
it is a **Config diagnostic**. Combinations that used to need a rule of their own (`pe`
with `gnu`, `msvc` on `aarch64`) are no longer expressible: the platform is a variant, and
its ABI environment is the variant's (TOOL-18). The default ABI is **total over the
supported targets** — every supported combination,
**including** freestanding `Os.none`/`Env.none` targets, has a default. It is determined
in three steps:

**(a) Base convention — per `arch`.** Each architecture has one **standard psABI**, used
regardless of `os`/`env` (so `Os.none`/`Env.none`/`musl`/`freebsd` all get a default):

| `arch`    | Base ABI (`name`) | Canonical document          |
|-----------|-------------------|-----------------------------|
| `x86_64`  | `sysv`            | System V AMD64 psABI        |
| `i386`    | `sysv32`          | System V IA-32 psABI (cdecl)|
| `aarch64` | `aapcs64`         | AAPCS64                     |
| `aarch32` | `aapcs32`         | AAPCS (ARM)                 |
| `riscv64` | `lp64[f\|d]`       | RISC-V psABI                |
| `riscv32` | `ilp32[f\|d]`     | RISC-V psABI                |

**(b) Convention-level exceptions.**
- `x86_64` on **`Machine.Windows`** (container `pe`): the default base convention becomes
  **`win64`** (§4.2).
- **`Machine.Com()`** (the `i386` real-mode image, `code_size = b16`;
  per-arch appendix §1): **no standard C calling convention applies** — `sysv32` is a
  32-bit protected-mode convention and is **not** the ABI of a 16-bit real-mode image.
  The default selector is therefore **`naked`** (`AbiSelector.Naked`, §3.1): the
  programmer supplies the frame/return directly (assembly-correspondence; entry points
  and far-call conventions are hand-written). `@abi(c)` / `@abi(sysv32)` on a
  `Container.com` target is a **diagnostic** (there is no C ABI to select).

No other `os`/`env`/`container` selects a different base convention.

**(c) Sub-variant — from `env` and `features`.** Within the base convention, the
floating-point sub-variant is fixed deterministically:

- **`aarch32`**: `Env.eabihf` → AAPCS **hard-float** (FP args in `s`/`d`; **requires the
  `vfp` feature** — `eabihf` without `vfp` is a Config diagnostic, per-arch appendix §6);
  `Env.eabi` → **soft-float** (FP args in core registers/stack). (Every supported
  `aarch32` target is `eabi` or `eabihf`, or `bionic` below; there is no `Env.none`
  `aarch32` target, per-arch appendix §1.)
- **`aarch32` on `Machine.Android`** (`Env.bionic`): the **soft-float ABI** — FP arguments
  pass in core registers/stack, exactly as `Env.eabi` — while `vfp`/`neon` **instructions**
  remain available as ISA features. The ABI and the instruction set are independent here,
  which is the NDK `armeabi-v7a` convention; treating `bionic` as hard-float would silently
  break every call into a platform library.
- **`riscv64`/`riscv32`** — by the widest enabled FP `feature`: the **`d`** extension →
  `lp64d`/`ilp32d` (double FP args in `fa*`); **`f`** without `d` → `lp64f`/`ilp32f`
  (single FP args in `fa*`); **neither** → `lp64`/`ilp32` soft-float (FP args in integer
  registers/stack). (The FP register file existing on the target is exactly what
  `features` records, Manifest §3.7.)
- **`os`** never changes the **base convention**; it tunes only a few fields:
  **`red_zone`** (`os = none`, i.e. freestanding/kernel, sets it to `0`, §4.1), the **varargs**
  variant (the Apple AArch64 stack variant, §4.4), and the **AArch64 platform register
  `x18`** — a caller-saved temporary on Linux, **reserved** on Apple, **reserved** on
  **Android** (bionic uses it as the shadow-call-stack register), and **reserved**
  on freestanding (`Os.none`) for forward compatibility (§4.4).

The prelude name **`c`** resolves to the default C ABI of the selected target computed
by (a)–(c); **`sysv`** names the System V convention explicitly; **`syscall`** is the
OS/arch syscall convention (§5). `@abi(…)` selector resolution is §3.1.

---

### 3. Common rules

#### 3.1 `@abi(…)` — value selector

`@abi`'s argument is an **ABI selector** — an **`Abi` value** (§1), or one of the two
distinguished sentinels **`naked`** / **`entry`**:

```alatyr
AbiSelector := enum { Conv( Abi ), Naked, Entry }   # a convention, "no lowering", or the process entry
```

- **`Conv(abi)`** — an ordinary calling convention (the §1 schema). The prelude binds the
  identifiers **`c`**, **`sysv`**, **`sysv32`**, **`win64`**, **`aapcs64`**, **`aapcs32`**,
  **`lp64`**/**`lp64f`**/**`lp64d`**, **`ilp32`**/**`ilp32f`**/**`ilp32d`**, and
  **`syscall`** to such values, and
  a user-defined `Abi(...)` value (Functions §6.3) is one too. Canonical spelling: `@abi(c)`,
  `@abi(syscall)`, `@abi(MyConv)`.
- **`naked`** is **not** an `Abi`: it is a **sentinel** meaning *no compiler-generated
  frame/prologue/epilogue/return* (Functions §6.4). It carries **no** register/stack/
  varargs schema and is never read field-by-field; it is a lowering **mode**, not a
  convention. `@abi(naked)` selects it.
- **`entry`** is likewise a sentinel, not an `Abi`: it marks **the process entry** and
  selects the **per-target entry prologue/epilogue** of §3.3 (FN-12). The function is
  written as an ordinary function but declares **no parameters** (`fn()` / `fn() -> i32`,
  §3.3): the platform's entry state is captured by the prologue, not passed as arguments,
  and the body's result becomes the process exit status. `@abi(entry)` selects it.

`@abi` does **not** accept string-literal shortcuts: `@abi("c")` and
`@abi("syscall")` are ill-formed. ABI selection follows the language's ordinary
value-resolution model; strings remain for exact external/linker names such as
`@extern("printf")` and `@export("_start")`.

#### 3.2 Stack, frame, and aggregates

- **Stack grows down** on every v1 target (`stack_grows_down = true`).
- **Alignment** is enforced at every call boundary (`stack_align`); a misaligned call is
  an ABI violation the compiler does not produce.
- **`@abi(naked)`** (Functions §6.4): none of §1's prologue/epilogue lowering applies —
  the body is emitted verbatim; the programmer supplies frame and return (typically with
  assembly-correspondence constructs).
- **Aggregate passing** is by the per-target rule (§4 tables); an aggregate over
  `aggregate_max_bytes` (or one that does not fit the remaining argument registers) is
  passed **by reference to a copy** for `in` (Functions §4.1), visible in the emitted
  code (I1).
- **Result by memory:** a result too large for the result registers is returned via a
  caller-provided address; where the arch dedicates a register to that address it is
  `indirect_result` (AArch64 `x8`), otherwise the address is the first integer argument
  (System V AMD64, Win64).

#### 3.3 Process entry — `@abi(entry)` and the per-target contract

`@abi(entry)` (§3.1; FN-12) is available under **`startup = raw`** (Manifest appendix §3.2) and
marks the function the process starts in. Its **declared form is portable**: `fn()` or
`fn() -> i32` — no parameters. The platform's entry state is not a parameter list but a
per-target fact the prologue captures, and the command line and environment are read
through the library (`args`/`env`, Stdlib appendix §7), so one source form is correct on
every target.

The compiler emits, in this order: **capture** the platform entry state, **record** it for
`args`/`env`, **establish** a call-ready frame (stack alignment per §3.2), **call** the body
under the target's default convention (§2), and **terminate** the process with the body's
result (`0` when the body has no result). What "capture" and "terminate" mean is fixed per
container:

| container / os | state on entry | terminate with |
|---|---|---|
| `elf` (`Os.linux`, `freebsd`) | `sp` → `argc`, then `argv[]`, `NULL`, `envp[]`, `NULL`, `auxv[]`; stack aligned per §3.2; no return address — the entry **cannot** `ret` | the OS exit syscall (`@abi(syscall)`, §5; Stdlib §4.2) |
| `macho` (`Os.macos`) | `dyld` calls the entry **as a function**: `argc`, `argv`, `envp`, `apple[]` in the argument registers of §4 | `libSystem`'s `exit` (see the platform-library rule below) |
| `pe` (`Os.windows`) | the loader passes **no** arguments and no command line; `argv` does not exist at entry — the library obtains it on demand | `kernel32`'s `ExitProcess` (see below); never a plain `ret` |

**Platform libraries under `startup = raw`.** `raw` links no start-up files and no libc
(TOOL-12), but two containers have **no syscall ABI to fall back on**: Windows has no
stable public syscall numbering and macOS routes calls through `libSystem`, so `@abi(syscall)`
is **rejected** on both (§5). For them the toolchain therefore adds the **platform library**
the entry contract needs, as a **`dynamic`** dependency (MOD-9), and nothing else:

| container | added library | the entries used |
|---|---|---|
| `pe` | `kernel32` | `ExitProcess` (process exit); `GetCommandLineW` + `GetEnvironmentStringsW` when `args`/`env` are reachable |
| `macho` | `libSystem` | `exit`; `_NSGetArgv` / `_NSGetEnviron` when `args`/`env` are reachable |
| `elf` | *(none)* | the syscalls of §5 |

Because those libraries are `dynamic`, such a build is **non-hermetic** and the toolchain
surfaces the ordinary Config note (Tooling §2.5; MOD-9) — a `raw` build on PE or Mach-O is
self-contained only up to the OS's own always-present library, which is what "static" means
on those platforms. The additions are **fixed by this table**, not discretionary: an
implementation adds exactly these, so the link graph stays predictable from the manifest
plus this appendix (I3).

Two targets have **no** platform entry contract, so `@abi(entry)` is **not applicable** and
its use is a diagnostic — `@abi(naked)` is the form there:

- **`Os.none`** (freestanding): the entry address comes from the linker script or a reset
  vector, and the stack pointer, `gp`-style base registers, `.bss` and `.data` are the
  program's own business (the freestanding triples of per-arch appendix §1). No generic
  prologue can be correct.
- **`Container.com`** (real-mode flat image): the default selector is already `naked`
  (§2(b)); there is no C-level convention to build a frame in.

**The recorded entry state.** "Record it for `args`/`env`" is a **named static**, not hidden
state: the prologue stores the captured values into a compiler-emitted `@static` of **three
pointer-width words** — `argc`, `argv`, `envp` (a word is zero where the container supplies
no such value) — whose linker symbol is **`alatyr_entry_state`**, placed in the target's
default writable data section (Memory §2.3). It is an ordinary emitted symbol: subject to
the package-wide uniqueness rule (Modules §6.7) and visible in the artifact (I3). It is
emitted, and the recording step executed, **only when `args` or `env` is reachable** in the
program (Stdlib appendix §7); otherwise the prologue captures nothing and the static does
not exist — an unused facility costs nothing (I2).

**Frame and unwind.** The prologue's frame is an ordinary non-leaf frame (§3.2): on PE the
function receives the same unwind data (`.pdata`/`.xdata`) any other non-leaf function does,
and `.cfi_*` call-frame directives follow `Target.auto_cfi` (Tooling §2.2) exactly as for a
declared function. `@abi(entry)` is **not** admitted under `Limit.no_abstractions`, whose
allow-list requires every emitted runtime instruction to be one the programmer wrote
(Assembly §1.1): the entry form there is `@abi(naked)`.

**Diagnostics** (all **Semantic**, per build): a declared form other than `fn()` /
`fn() -> i32`; **more than one** `@abi(entry)` function reachable in one built artifact; an
`@abi(entry)` function that `Target.entry` does **not** name; `@abi(entry)` under
`startup = libc` (the platform's start-up owns the entry, so there is no prologue to emit);
and `@abi(entry)` in a build whose `kind` is neither `executable` nor `object`.
`Target.entry` (Tooling §2.2) still names which declaration is the entry, and it may equally
name an `@abi(naked)` one — the selector is about who writes the prologue, not about who is
the entry.

**Under `startup = libc`** the platform's start-up files own the process entry and call a
function the program must supply, by exact name and form:

| container / `subsystem` | the program supplies | form |
|---|---|---|
| `elf`, `macho` | `main` | `fn() -> i32`, or the C form `fn(in argc : i32, in argv : ptr(ptr(u8))) -> i32` |
| `pe`, `subsystem = console` | `main` | as above |
| `pe`, `subsystem = windows` | `WinMain` | `fn(in inst : usize, in prev : usize, in cmd : ptr(u8), in show : i32) -> i32` |
| `pe`, `subsystem = native` | — | `startup = libc` is a **Config diagnostic** (no CRT start-up exists) |

That declaration MUST be **root-level** so its symbol is unprefixed (Modules §6.1), or carry
an exact `@export` of the name above; it is emitted and is the artifact's reachability root
exactly as `entry`'s declaration is under `raw` (Modules §6.4). The wide (`w`-prefixed)
CRT entry forms are **additive** (FND-6): v1 fixes the narrow ones.

---

### 4. Per-target calling conventions

Registers are listed in **argument order**. "Callee-saved" lists the registers a callee
must preserve (the frame/stack pointers and link register are saved by the standard
prologue and not relisted as general saves).

#### 4.1 `x86_64` — System V AMD64 (`sysv`; ELF/Mach-O; `c` on Linux/macOS/BSD)

| Field                 | Value                                                        |
|-----------------------|--------------------------------------------------------------|
| `int_arg_regs`        | `rdi, rsi, rdx, rcx, r8, r9`                                  |
| `fp_arg_regs`         | `xmm0 … xmm7`                                                 |
| `int_ret_regs`        | `rax, rdx`                                                    |
| `fp_ret_regs`         | `xmm0, xmm1`                                                  |
| `callee_saved`        | `rbx, rbp, r12, r13, r14, r15`                                |
| `caller_saved`        | `rax, rcx, rdx, rsi, rdi, r8, r9, r10, r11, xmm0 … xmm15`     |
| `stack_align`         | `16`                                                          |
| `red_zone`            | `128` (leaf functions; **0** when `os = none` — freestanding/kernel)  |
| `shadow_space`        | `0`                                                           |
| `aggregate_max_bytes` | `16` (two eightbytes, classified INTEGER/SSE; else by reference) |
| `varargs`             | `sysv_al` (`al` = number of vector registers used)           |

#### 4.2 `x86_64` — Microsoft x64 (`win64`; PE; `c` on Windows)

| Field                 | Value                                                        |
|-----------------------|--------------------------------------------------------------|
| `int_arg_regs`        | `rcx, rdx, r8, r9`                                           |
| `fp_arg_regs`         | `xmm0, xmm1, xmm2, xmm3` (positional — shared slots with int)|
| `int_ret_regs`        | `rax`                                                        |
| `fp_ret_regs`         | `xmm0`                                                       |
| `callee_saved`        | `rbx, rbp, rdi, rsi, r12, r13, r14, r15, xmm6 … xmm15`        |
| `caller_saved`        | `rax, rcx, rdx, r8, r9, r10, r11, xmm0 … xmm5`                |
| `stack_align`         | `16`                                                         |
| `red_zone`            | `0`                                                          |
| `shadow_space`        | `32` (caller reserves; callee may spill the 4 register args) |
| `aggregate_max_bytes` | `8` (larger → by reference)                                  |
| `varargs`             | `win64_dual` (FP varargs in **both** the int and xmm slot)   |

#### 4.3 `i386` — System V IA-32 (`sysv32`/cdecl; ELF)

| Field                 | Value                                                        |
|-----------------------|--------------------------------------------------------------|
| `int_arg_regs`        | *(none — integer args on the stack, right-to-left)*          |
| `fp_arg_regs`         | *(none — on the stack)*                                      |
| `int_ret_regs`        | `eax, edx`                                                   |
| `fp_ret_regs`         | `st0`                                                        |
| `callee_saved`        | `ebx, esi, edi, ebp`                                         |
| `caller_saved`        | `eax, ecx, edx`                                              |
| `stack_align`         | `16` (modern GCC/SysV at a call)                             |
| `red_zone`            | `0`                                                          |
| `aggregate_max_bytes` | `0` (aggregates by reference / on the stack; caller cleans, cdecl) |
| `varargs`             | `stack_after_named`                                          |

#### 4.4 `aarch64` — AAPCS64 (`aapcs64`; `c`)

| Field                 | Value                                                        |
|-----------------------|--------------------------------------------------------------|
| `int_arg_regs`        | `x0 … x7`                                                    |
| `fp_arg_regs`         | `v0 … v7`                                                    |
| `int_ret_regs`        | `x0, x1`                                                     |
| `fp_ret_regs`         | `v0, v1`                                                     |
| `indirect_result`     | `x8`                                                         |
| `callee_saved`        | `x19 … x28`, low 64 bits of `v8 … v15`                       |
| `caller_saved`        | `x0 … x17`, `v0 … v7`, `v16 … v31` (+ `x18` on Linux)         |
| `stack_align`         | `16`                                                         |
| `red_zone`            | `0`                                                          |
| `aggregate_max_bytes` | `16` (HFAs in `v` registers; larger → by reference)          |
| `varargs`             | `aapcs` (Apple variant passes varargs on the stack)          |

`x18` is the **platform register**: a caller-saved temporary **on Linux only**;
**reserved** (not used by the compiler) on the Apple variant **and on freestanding
`Os.none`** (where a host platform may later claim it — reserved for forward
compatibility) (per-arch appendix §5.1).

#### 4.5 `aarch32` — AAPCS (`aapcs32`; `c`)

Two sub-variants (selected per §2(c)): **hard-float** (`aapcs32-hf`, `Env.eabihf`) and
**soft-float** (`aapcs32-sf`, `Env.eabi`). Differing fields:

| Field                 | hard-float (`eabihf`)            | soft-float (`eabi`)               |
|-----------------------|----------------------------------|-----------------------------------|
| `fp_arg_regs`         | `s0 … s15` / `d0 … d7`           | `[]` (FP in `r0 … r3` / stack)    |
| `fp_ret_regs`         | `s0` / `d0`                      | `[]` (FP in `r0`/`r0,r1`)         |
| `callee_saved` (FP)   | adds `d8 … d15`                  | (none — no FP regs used)          |

Shared fields:

| Field                 | Value                                                        |
|-----------------------|--------------------------------------------------------------|
| `int_arg_regs`        | `r0, r1, r2, r3`                                             |
| `int_ret_regs`        | `r0, r1`                                                     |
| `callee_saved` (int)  | `r4 … r11`                                                  |
| `caller_saved`        | `r0, r1, r2, r3, r12`                                        |
| `stack_align`         | `8` (at a public interface)                                  |
| `aggregate_max_bytes` | `4` (larger → by reference; small aggregates split across `r0 … r3`) |
| `varargs`             | `stack_after_named` (variadic FP always in core registers)   |

#### 4.6 `riscv64` / `riscv32` — RISC-V psABI (`lp64[f|d]` / `ilp32[f|d]`; `c`)

Three sub-variants (selected per §2(c) by the widest enabled FP `feature`): **`d`**
(`lp64d`/`ilp32d`, double hardware FP), **`f`** without `d` (`lp64f`/`ilp32f`, single
hardware FP), and **soft** (`lp64`/`ilp32`, no FP feature). Differing fields:

| Field               | `d` (`lp64d`/`ilp32d`)   | `f`-only (`lp64f`/`ilp32f`) | soft (`lp64`/`ilp32`)          |
|---------------------|--------------------------|-----------------------------|--------------------------------|
| `fp_arg_regs`       | `fa0 … fa7` (double)     | `fa0 … fa7` (single)        | `[]` (FP in `a0 … a7` / stack) |
| `fp_ret_regs`       | `fa0, fa1`               | `fa0, fa1`                  | `[]` (FP in `a0, a1`)          |
| `callee_saved` (FP) | adds `fs0 … fs11`        | adds `fs0 … fs11`           | (none)                         |

Shared fields:

| Field                 | Value                                                        |
|-----------------------|--------------------------------------------------------------|
| `int_arg_regs`        | `a0 … a7` (`x10 … x17`)                                      |
| `int_ret_regs`        | `a0, a1`                                                     |
| `callee_saved` (int)  | `s0 … s11` (`x8, x9, x18 … x27`)                            |
| `caller_saved`        | `t0 … t6`, `a0 … a7`, `ra`; `ft0 … ft11`, `fa0 … fa7` (any hard-float `f`/`d`) |
| `stack_align`         | `16`                                                         |
| `aggregate_max_bytes` | `2 × XLEN` (16 on `riscv64`, 8 on `riscv32`; larger → by reference) |
| `varargs`             | `stack_after_named` (varargs in `a`-registers then stack; FP varargs in int regs) |

---

### 5. Syscall ABIs (`@abi(syscall)`)

The syscall convention is **OS- and arch-specific** and distinct from the C ABI. It is
selected with `@abi(syscall)` and is writable only inside an `unchecked` grant
(raw-level interface, Functions §7.3 precedent). Each table gives the **convention**
(the number register, the argument/result registers, how an error is signalled, and the
trap instruction); the syscall **numbers** are OS-version data the programmer supplies,
not fixed here.

**Linux** — an error is returned **in the result register** as a negative value
(`−errno`):

| Arch      | Number reg | Argument regs                      | Result | Error signal      | Clobbers   | Trap        |
|-----------|------------|------------------------------------|--------|-------------------|------------|-------------|
| `x86_64`  | `rax`      | `rdi, rsi, rdx, r10, r8, r9`       | `rax`  | `−errno` in `rax` | `rcx, r11` | `syscall`   |
| `i386`    | `eax`      | `ebx, ecx, edx, esi, edi, ebp`     | `eax`  | `−errno` in `eax` | —          | `int 0x80`  |
| `aarch64` | `x8`       | `x0 … x5`                          | `x0`   | `−errno` in `x0`  | —          | `svc #0`    |
| `aarch32` | `r7`       | `r0 … r6`                          | `r0`   | `−errno` in `r0`  | —          | `svc #0`    |
| `riscv64` | `a7`       | `a0 … a5`                          | `a0`   | `−errno` in `a0`  | —          | `ecall`     |
| `riscv32` | `a7`       | `a0 … a5`                          | `a0`   | `−errno` in `a0`  | —          | `ecall`     |

**FreeBSD and macOS** (BSD-style) — an error is signalled by the **carry flag** (`CF` on
x86, the `C` bit of `PSTATE` on AArch64); on error the result register holds the
**positive `errno`**:

| Target            | Number reg                         | Argument regs                | Result | Error signal     | Trap        |
|-------------------|------------------------------------|------------------------------|--------|------------------|-------------|
| `x86_64-freebsd`  | `rax`                              | `rdi, rsi, rdx, r10, r8, r9` | `rax`  | carry flag set   | `syscall`   |
| `x86_64-macos`    | `rax` (+ BSD class `0x2000000`)    | `rdi, rsi, rdx, r10, r8, r9` | `rax`  | carry flag set   | `syscall`   |
| `aarch64-macos`   | `x16`                              | `x0 … x5`                    | `x0`   | carry flag set   | `svc #0x80` |

- On **macOS** the syscall *numbers* are **not a stable public ABI** (Apple routes calls
  through `libSystem`); a direct-syscall program therefore targets a specific OS version
  — consistent with `@abi(syscall)` being an `unchecked`, OS-coupled interface. The
  `x86_64` number additionally carries the BSD class bits (`0x2000000`).
- **Windows** (`x86_64-windows-msvc`) has **no stable public syscall ABI** (NT syscall
  numbers are unstable); `@abi(syscall)` is **rejected** — call the platform libraries
  via `@extern` / `@abi(c)` instead.
- A **freestanding** target (`os = none`) defines no syscalls; `@abi(syscall)` is
  **rejected** with a diagnostic. Further operating systems are **additive** (FND-6): the
  same schema, populated with that OS's documented convention.

A syscall convention is the one whose `Abi.number_reg` is set (§1); its **call lowering
is this section's table**, not §4 — none of `§1`'s prologue/epilogue/frame lowering
applies. A `@abi(syscall)` function is a **bodyless declaration** (the trap *is* the
implementation, like `@extern`); every call to it emits the table's **trap instruction**,
not an ordinary `call`.

**Parameter and result mapping.** Parameters map **positionally**:

- the **first** parameter supplies the **syscall number** → the **Number reg**
  (`Abi.number_reg`);
- each **remaining** parameter, in declaration order, is a syscall **argument** → the
  **Argument regs** (`Abi.int_arg_regs`) in the order listed;
- the function's single result (`-> T`) receives the **Result reg** (`Abi.int_ret_regs[0]`)
  verbatim.

The function MUST have at least one parameter (the number); the argument count MUST NOT
exceed the available Argument regs (a type/Config diagnostic otherwise). Every parameter
and the result MUST be a **register-width integer or pointer** type — no aggregate is
passed through a syscall (`ptr(T)` and an `isize`/`usize`-width integer are the
intended forms). The **Clobbers** are assumed caller-saved across the trap; every other
register is preserved (I1).

**Error model.** The result is the Result reg verbatim — interpreting it is the caller's
job. On a **Linux** target a negative result is `−errno`, so the natural result type is
`isize` (a library wraps it into a `Result`); on a **BSD/macOS** target the **carry
flag** signals error (the Result reg then holds the positive `errno`).

The **normative surface** for the carry condition is a **second named result** on the
`@abi(syscall)` declaration: `out errored : bool` (the `out`-result mechanism, Functions
§3.1 — no new entity, OP-1). When declared, the implementation populates it from the
carry flag immediately after the trap (`true` ⇒ error) on a carry-flag target, and as a
constant `false` on a negative-errno target (where the sign of the Result reg carries the
condition instead). A declaration that omits it observes only the Result reg (the Linux
form). This makes the condition observable identically across implementations — two
toolchains built to this appendix agree (FND-3) — without a target-specific ad-hoc channel.
On macOS the syscall numbers are OS-version data (§5); the row fixes the calling
mechanism, not a stable public numbering contract.

---

### 6. Scalar and aggregate passing, variadics (summary)

- **Scalar classification.** A scalar argument/result's register **class** follows its
  **numeric interpretation**, not its raw-block *representation*. An `fN` — the
  floating-point interpretation (Types §3 / §2; a prelude **brand** over `bitsN`,
  TYP-2/TYP-4) — classifies **floating-point** and uses `fp_arg_regs` / `fp_ret_regs`;
  `uN` / `iN` / `bitsN` / `bool` / `char` and every pointer classify **integer** and use
  `int_arg_regs` / `int_ret_regs`. The class is therefore a property of the
  *interpretation a brand names*, **not** of `peel`-to-base: peeling `fN` to its `bitsN`
  block would mis-class a float as an integer eightbyte, so an implementation MUST key the
  class on the interpretation (the `fN`/`uN`/`iN` family the brand is), recognizing the
  `fN` brand as floating-point even though it shares `bitsN`'s representation. A
  **non-numeric** brand (`Meters := brand(u64)`) takes its base interpretation's class
  (here integer). This scalar class is the input to the per-target field-classification
  below (e.g. which System V eightbyte is SSE vs INTEGER, which AArch64 field is an HFA
  lane).
- **Aggregate passing** follows each target's `aggregate_max_bytes` and the platform's
  field-classification rule (e.g. System V AMD64 eightbyte classification, AArch64
  HFA/HVA). The lowering is the **per-target table's** rule; the *cost* (a register
  group, or a by-reference copy) is visible in the emitted code (I1). A raw union is
  classified from its complete declared union layout under that platform rule, never
  from which member is active at a call site; every carrier preserves the complete
  `U.size()` object representation, including padding, and carries no active-member
  metadata (Type System §6.3/TYP-11).
- **A checked `@require` constructor predicate** is an ordinary one-parameter call
  `fn(in U) -> bool` (Types §8.1). The constructor's result representation is not the
  argument representation: for a scalar/small aggregate, copy the value into the ordinary
  classified argument registers/stack slots while preserving the result; for an aggregate
  larger than `aggregate_max_bytes`, make exactly one full caller-owned byte-copy and pass
  its address under Functions §4.1. Preserve the result through ordinary S2 placement and
  caller-save rules; that save/reload is not another predicate-argument aggregate copy.
  `unchecked` emits none of that argument-copy/call path.
- **`VarargsRule`** values: `none` (no C variadics); `sysv_al` (System V AMD64 — `al`
  carries the vector-register count); `win64_dual` (FP variadic args duplicated into the
  integer slot); `aapcs` (AAPCS64, with the Apple stack variant); `stack_after_named`
  (named args in registers, variadic args on the stack). A C variadic is legal only
  under `@abi(c)` within an `unchecked` grant (Functions §7.3; I11).

---

### 7. Conformance

A conforming implementation MUST:

1. reject an **unsupported target** (a combination outside the supported-target set,
   §2; Manifest §3.1) with a Config diagnostic, and for every **supported target**
   (including freestanding `Os.none`/`Env.none`) compute the **default ABI** by the
   procedure of §2 (per-arch base; the Windows/PE and `Container.com`→`naked`
   exceptions; the `env`/`features`-derived float sub-variant; `os` tuning only
   `red_zone`, the varargs variant, and the AArch64 `x18` status — Linux temp vs
   Apple/freestanding reserved):
   - for a **`Container.com`** target the default selector is **`AbiSelector.Naked`**
     (§3.1) — **no §1 ABI lowering applies** (the programmer supplies the frame/return);
     any **conventional `Conv(Abi)` selector** (`@abi(c)`, `@abi(sysv32)`, …) on such a
     target is a **diagnostic** unless a convention is explicitly defined for COM;
   - for **every other** supported target, treat `c` as that target's C convention and
     **lower Functions §4 passing modes against the §1 `Abi` value's fields**, filling
     unlisted table fields from the §1 **defaults**;
2. honor the **per-target tables** of §4 — argument/result registers, callee-/caller-
   saved sets, `stack_align`, `red_zone`, `shadow_space`, `indirect_result`, and
   `aggregate_max_bytes` — together with the §1 defaults for unlisted fields, and the
   common rules of §3;
3. resolve **`@abi(…)`** per §3.1 (a prelude/user ABI **value**, or the `naked` /
   `entry` sentinels; string literals are rejected), implement **`@abi(naked)`** as
   emitting no compiler-generated frame/prologue/epilogue/return (Functions §6.4),
   **`@abi(entry)`** as the per-container entry prologue/epilogue of §3.3 — rejecting it
   on `Os.none`, `Container.com`, `startup = libc`, a non-`executable`/`object` build, a
   declaration `Target.entry` does not name, a second such function in one artifact, and
   any declared form other than `fn()` / `fn() -> i32`; emitting the recorded entry state
   as the documented `alatyr_entry_state` static only when `args`/`env` are reachable; and
   supporting the `startup = libc` start-up-called function per the §3.3 table — and
   `@abi(value)` per-site selection (Functions §6.2);
4. implement **`@abi(syscall)`** per the OS/arch table of §5 where the OS defines one,
   rejecting it with a diagnostic where it does not, and permit it only inside an
   `unchecked` grant — lowering a call to the table's **trap instruction** (not a §4
   call) with the **positional** parameter mapping of §5 (first parameter → the Number
   reg; the rest → the Argument regs in order; result ← the Result reg), and rejecting a
   non-register-width or aggregate parameter/result and an argument count exceeding the
   Argument regs;
5. accept a **user-defined `Abi(...)`** value populating the §1 schema, and apply it
   identically to a prelude ABI;
6. emit C variadics only under `@abi(c)` within an `unchecked` grant, per each
   target's `VarargsRule` (§6); reject C variadics in checked code (I11);
7. lower a checked `@require` predicate as the ordinary `fn(in U) -> bool` call of §6,
   preserving the result and making exactly the scalar/small-aggregate argument copy or
   one large-aggregate byte-copy specified there; omit the whole copy/call path under
   `unchecked` (Types §8.1).
