# Alatyr Language Specification

## Appendix — Manifest syntax

> **Status — draft (under review).** This appendix is a draft and is not yet
> accepted; the specification as a whole is under development. See the status note
> at the top of the Overview chapter.

This appendix fixes the **concrete syntax of the package manifest** — the normative v1
content the Tooling chapter defers to "a v1 tooling appendix" (Tooling §1). The Tooling
chapter is normative for what the manifest **means** (configuration; the field semantics,
§2; build profiles, §3; `target.*`/`build.*`, §2.7); this appendix is normative for
how a manifest is **written** and **read**.

The manifest is **a single `Package` value** bound in the package's **root module**
(TOOL-3): one ordinary declaration `‹name› := Package(…)`, whose **binding name is the
source-visible handle** — source reads the manifest as the comptime constant
`‹name›.*` (Tooling §2.7). It is **not** a second grammar: `Package` is a prelude config
struct (§3) and its field values are written in a **data subset of Alatyr itself**
(OP-1/TOOL-3: nothing expressible by existing mechanisms is re-introduced) — literals,
aggregates, arrays, and config-type constructors, evaluated as **pure, host-independent**
compile-time data. This reuses the lexis and grammar of the Grammar chapter, does not
compromise reproducible builds (Tooling §6.2), and yields the `target.*` / `build.*`
comptime constants directly from the resolved value.

---

### 1. The manifest file

- A package has **at most one** `Package` value. By default configuration finds it in the
  file **`package.al`**, searched for from the **current working directory upward** to the
  filesystem root — the **first** one found is the manifest and its directory is the
  **package root** (TOOL-14); the CLI **`--manifest <path>`** names the file explicitly
  and its directory becomes the root (Tooling §4). Zero `Package` values synthesize the
  default package (§3.7); more than one `Package` value in that module is a **Config
  diagnostic**.
- That file **is** the **anonymous package-root module** (MOD-1/MOD-3): it carries the
  `Package` value **and** the root module's ordinary code and declarations, so a
  **single-file package** — manifest and code in one file — is normal; submodules live
  by path under `source_dir`. The `Package` value configures the build; it does not
  itself export symbols.
- The binding is an **ordinary declaration of that root module** (TOOL-15): non-`pub`, so
  it is visible to the whole package by **down-tree privacy** (Modules §3) and to nothing
  outside it. **`pub` on it is a Config diagnostic** — its type belongs to the
  configuration prelude (§3), which is not part of the language's ordinary namespace, so a
  library publishes metadata by re-exporting fields (`pub VERSION := ‹name›.version`). Its
  name shares the root module's scope with the root's other declarations **and with
  `source_dir`'s child modules**, so a same-named module (`‹source_dir›/‹name›.al`) is a
  duplicate-name error (Modules §5).
- It is **UTF-8** and uses the same lexis as source (Grammar §2): identifiers,
  literals, comments (`#` / `##`), and the separator rules (§2.6) — it **is** source.
- The `Package` value is **evaluated first**, at **configuration** (Tooling §1; Codegen
  §1.1), before the rest of the program is lowered. Its evaluation is **pure** and
  **host-independent** (§5).

---

### 2. The data subset

The **`Package` value and every binding it transitively references** are restricted to
a **data subset** — they are pure configuration data, even though the surrounding root module
is ordinary code. A data value is:

```ebnf
manifest-decl ::= ident ":=" data-value                  (* the Package value, or a helper it references *)
                | ident ":" type-expr "=" data-value      (* optional type ascription *)
data-value    ::= literal                                (* int / float / bool / char / str (Grammar §2.4) *)
                | array-ctor                              (* [a, b, c] / [v; N] *)
                | tuple-ctor                              (* (a, b) *)
                | struct-ctor                             (* Type(field = value, …) — incl. Package(…) *)
                | variant-ctor                            (* Enum.Variant | Enum.Variant(value) *)
                | type-name                               (* a prelude type used as a value — for profile-flag types (§3.6) *)
                | ident                                   (* a reference to another manifest binding *)
                | path                                    (* a prelude config name: Arch.x86_64, Kind.executable *)
```

The root module's **other** declarations (functions, types, the program's code) are
unrestricted ordinary Alatyr; only the `Package` value's own argument expressions — and
any binding they reference — must lie in this data subset.

**Call-shaped forms.** `struct-ctor` and `variant-ctor` are *syntactically* a call
(`Target(…)`, `DepSource.Git(…)`) — the same shape as a function call (Grammar §3.4).
The boundary is **resolution-checked**, not syntactic: a call-shaped `data-value` is
accepted **only if its callee resolves to a schema/prelude data-type constructor** (a
`struct` constructor or an `enum` variant of §3) **or to one of the aggregate forms
above** (`array-ctor`/`tuple-ctor`). A call-shaped form whose callee resolves to a
**function value** (or to anything else) is **not data** and is a Config diagnostic.
Equivalently: the only callables the `Package` value may invoke are the §3 config type
constructors; there is no other evaluation.

Within the `Package` value's arguments (and any binding they reference) the data subset
**forbids** (a Config-stage diagnostic, Tooling §5):

- value introducers with bodies — `fn(…){…}` (including a generic type `fn(…) -> type`),
  `struct{…}`/`enum{…}`/`union{…}` **definitions**, `mod{…}` (the manifest **uses** the prelude
  config types of §3, it does not define types);
- executable statements (`assignment`, `return`/`break`/`continue`, `while`/`for`,
  `defer`) — a data value is an expression, not a statement;
- `comptime` blocks, attributes (`@…`), `ptr`/`deref`/`asm`/instruction calls, and
  any **function call** whose callee is not a §3 config type constructor (per the
  call-shape rule above);
- references to the comptime constants **`target.*` / `build.*`** — the manifest itself
  *defines* the targets and profiles, so these are **published only after** configuration
  resolves the selected target/profile (§5; Tooling §2.7); they do not yet exist during
  the manifest's own evaluation, and a reference to them in manifest data is a Config
  diagnostic.

**Visible names (the resolution boundary).** When the `Package` value is evaluated, the
only names in scope are: literals; the §3 prelude **config types** and their
constructors/variants (plus `Option(T)` for manifest use, §3); the base scalar/type
names admitted as profile-flag type values (`bool`, integer interpretations, `str`, and
manifest-local comptime enum types); and **other manifest bindings** (resolved in
dependency order). Nothing else is visible — not the root module's ordinary declarations,
not the wider prelude, and not `target.*`/`build.*`.

(These restrictions bind only the manifest data, not the rest of the root module, which
is ordinary code.) A manifest binding **MAY reference another** manifest binding by name (for DRY); such
references are resolved as pure comptime data in **dependency order**, and a **cycle**
is a Config diagnostic (the same well-definedness as module-level comptime initializers,
Declarations §7.1; Comptime §2.5).

---

### 3. The schema (prelude config types)

The configuration phase validates the manifest against a **fixed schema**. The schema is expressed with
the **prelude configuration types** below (given in ordinary Alatyr type syntax). A **Package field** is a
named argument of the `Package(…)` constructor; the field set is §3.7. An **unknown** field
name or a **type mismatch** is a Config diagnostic; every field **defaults** (§3.7), so
an omitted field — or the whole manifest — is **not** an error.

**Two halves of the configuration prelude.** The types below split by who may name them:

- the **manifest-only structures** — `Package`, `Target`, `Dependency`, `DepSource`, `GitRef`,
  `Lib`, `Profile`, `FlagDecl`, `FlagSet`, `FeatureAlias` — live in a prelude **visible only to the
  manifest** (§2's resolution boundary). Ordinary source cannot name them, which is why the manifest
  handle is not `pub`-exportable (Tooling §2.7; TOOL-15): its type does not exist outside the manifest.
- the **configuration enums** — `Arch`, `Os`, `Env`, `Container`, `Kind`, `Startup`, `CodeSize`,
  `ArmMode`, `Subsystem`, `Endian`, `Limit` — are **also ordinary prelude names**, because the
  `target.*` constants they describe are compared in ordinary code: `when target.arch == Arch.x86_64`,
  `when target.kind == Kind.static_lib` (Tooling §2.7). They are plain comptime enums with no special
  status: source may name a variant, bind one (`host := Arch.x86_64`), or `match` over
  `target.arch` — it simply has nothing to construct with them, since the values that matter are
  published by configuration.

The configuration prelude also provides the **base-prelude `Option(T)`** (Type System
§7) — a base-tier type independent of the Stdlib appendix's enumeration — so the
optional override fields of §4 may use `Option.None` / `Option.Some(v)`. For the
call-shape rule (§2), `Option` counts as a config type: `Option.Some(…)` is an accepted
variant constructor.

#### 3.1 Target and machine model

```alatyr
Arch      := enum { x86_64, i386, aarch64, aarch32, riscv32, riscv64 }   # the v1 arches (Assembly §10)
#   `Arch` names REGISTER ISAs only. An additive non-ISA backend (WASM → WAT; FND-6, CG-4) arrives as a new
#   `Machine` variant, not as an `Arch` — see CG-14 and Tooling §2.7 for what `target.arch` means there.
Env       := enum { gnu, musl, eabi, eabihf, bionic, none }              # libc / ABI environment (eabihf = ARM hard-float;
                                                                         #   `bionic` is `Machine.Android`'s projection only)
Subsystem := enum { console, windows, native }                           # PE subsystem
Startup   := enum { raw, libc }                                          # start-up files & libc (§3.2; TOOL-12)

Machine   := enum {                          # THE PLATFORM — one variant per platform shape (TOOL-18)
  Linux  ( arch : Arch, env : Env = ‹arch-default›, startup : Startup = Startup.raw )   # ELF, hosted
  Freebsd( arch : Arch, startup : Startup = Startup.raw )                               # ELF, hosted
  Windows( arch : Arch, subsystem : Subsystem = Subsystem.console,
           startup : Startup = Startup.raw )                                            # PE, hosted
  Macos  ( arch : Arch, startup : Startup = Startup.raw )                                # Mach-O, hosted
  Android( arch : Arch, api : u32 = ‹min-supported›, startup : Startup = Startup.raw )   # ELF, bionic
  Bare   ( arch : Arch, env : Env = Env.none )                                           # ELF, freestanding
  Com    ( )                                                                            # i386 real-mode flat image
}

Target    := struct {
  name          : str       = ""              # selection key for `--target`; optional when there is a single target
  machine       : Machine   = ‹host›          # the platform (above); defaults to the BUILD HOST's variant, whole
  kind          : Kind      = Kind.executable # Tooling §2.2
  entry         : str       = "_start"        # entry point: an Alatyr PATH to the declaration (§3.2; TOOL-12)
  output        : str       = ‹base+affix›    # artifact file name; default = the manifest binding name plus
                                              # the kind×container affix (§3.2); explicit = verbatim (TOOL-11)
  features      : [str]     = []              # opt-in ISA features (§3.7); package feature-aliases expand here
  vector_length : u32       = 0               # SVE/RVV
  code_size     : CodeSize  = ‹arch-native›   # x86 ONLY (`x86_64`/`i386`) — the encoding mode (§3.2)
  arm_mode      : ArmMode   = ArmMode.arm     # `aarch32` ONLY
  auto_cfi      : bool      = false           # emit CFI directives
}
```

**The platform is a variant, not a bag of fields** (TOOL-18). Everything that is true of
*some* platforms only — the container, the OS, the libc/ABI environment, the PE
`subsystem`, whether start-up files exist at all — lives **inside** the `Machine` variant
that has it, so an inapplicable field cannot be written in the first place. There is no
`os`, `container` or `endian` field: `Machine.Windows(…)` **is** PE, `Machine.Bare(…)` **is**
freestanding ELF, and endianness follows the `arch` (FND-8). Two fields stay on `Target`
because they are **arch**-specific codegen knobs rather than platform facts, and both are
applicable only to their own architecture (an explicit value elsewhere is a Config
diagnostic): **`code_size`** on `x86_64`/`i386` and **`arm_mode`** on `aarch32`.

**Defaulting is by variant, not field by field.** `Target()` is the **build host's** whole
variant (`Machine.Linux(arch = Arch.x86_64, env = Env.gnu)` on such a host) — a developer
convenience that makes an unconfigured build host-dependent. A cross or reproducible build
**states the variant**: `Target(machine = Machine.Bare(arch = Arch.aarch64))`. A *partial*
deviation ("aarch64, else host") is deliberately no longer expressible — it was the shape
that made a manifest's meaning depend on where it was built.

**`target.*` are projections of the variant** (Tooling §2.7): `target.arch` is the
variant's `arch` (`i386` for `Com`), `target.os` is `linux`/`android`/`freebsd`/`windows`/
`macos`/`none`, `target.container` is `elf`/`pe`/`macho`/`com`, `target.env` is the
variant's `env` (`bionic` for `Android`, `none` where the variant has none), `target.startup` is the variant's (`raw` for `Bare`/`Com`,
which have no other mode), and `target.endian` follows the arch. So every existing gate —
`when target.arch == Arch.x86_64`, `when target.os == Os.none` — keeps working unchanged,
and `Os` / `Container` remain prelude enums for those comparisons even though neither is a
`Target` field.

Not every variant admits every `arch`: the **set of supported targets** is normative v1
content enumerated in the **per-arch appendix §1**, now one row per `(variant, arch)` pair.
A combination outside it — `Machine.Windows(arch = Arch.aarch64)`, `Machine.Macos(arch =
Arch.riscv64)` — is a **Config diagnostic**, in whichever `Target` of `targets` it appears.
What the variant model removes is the whole class of *unrepresentable* combinations that
used to need diagnostics of their own (`pe` with `gnu`, `msvc` on `aarch64`, an `endian`
contradicting the arch, a `subsystem` off PE, a `startup` on `Os.none`): they can no longer
be written. The default ABI is defined for every supported target (ABI appendix §2).

A target is a **structured value**, not a string triple: a string `"x86_64-linux-gnu"`
would need a second mini-grammar to parse, whereas a variant of a struct of enums is data
the grammar already expresses and maps **directly** to the `target.*` projections.

**Enum values use the variant-access form `Type.variant`** throughout (Grammar §3.4
variant-ctor): `Arch.x86_64`, `Machine.Linux(…)`, `Kind.executable`, `Limit.freestanding`.
There is **no** lowercase `arch`/`os`/… namespace; the comptime constant exposed to source
is `target.arch`, compared against the variant `Arch.x86_64` (so `when target.arch ==
Arch.x86_64`, Tooling §2.7).

#### 3.2 Output — kind, artifact name, entry, start-up

```alatyr
Kind      := enum { executable, static_lib, shared_lib, object, source }  # Tooling §2.2
CodeSize  := enum { b16, b32, b64 }                                       # 16/32/64 (bootloaders/real mode)
ArmMode   := enum { arm, thumb }
Os        := enum { linux, android, windows, macos, freebsd, none }      # NOT a field — the `target.os` projection
Container := enum { elf, pe, macho, com }                                # NOT a field — the `target.container` projection
Endian    := enum { big, little }                                         # NOT a field — follows the arch (FND-8)
```

`Os`, `Container` and `Endian` are **projection enums**: since TOOL-18 the platform is the
`Machine` variant (§3.1), so none of the three is a `Target` field — they remain prelude
names because `target.os` / `target.container` / `target.endian` are compared against them
in ordinary code (Tooling §2.7).

**The artifact name** (TOOL-11) has a **base** and a **per-target affix**. The base
`‹base›` is the **manifest binding's name** — `app := Package(…)` gives `app` (there is
no `Package.name`, §3.7; the binding is the handle, TOOL-3). Where there is **no binding**
— a manifest-less invocation (Tooling §4), or a root module carrying **zero** `Package`
values (§1, the synthesized default package) — the base is the **stem of the root source
file** (so a bare `package.al` builds `package`; state `output` to name it otherwise). An
**omitted** `output` is that base decorated per **`kind` × `container`**:

| `kind` | `elf` | `macho` | `pe` | `com` |
|---|---|---|---|---|
| `executable` | `‹base›` | `‹base›` | `‹base›.exe` | `‹base›.com` |
| `static_lib` | `lib‹base›.a` | `lib‹base›.a` | `‹base›.lib` | — |
| `shared_lib` | `lib‹base›.so` | `lib‹base›.dylib` | `‹base›.dll` | — |
| `object` | `‹base›.o` | `‹base›.o` | `‹base›.obj` | — |
| `source` | *(no artifact)* | *(no artifact)* | *(no artifact)* | — |

The columns are the **`target.container` projection** of the `Machine` variant (§3.1):
`Machine.Linux`/`Freebsd`/`Bare` → `elf`, `Windows` → `pe`, `Macos` → `macho`, `Com` →
`com`. A **non-ISA backend** (a structured/VM backend such as WASM → WAT; CG-4/CG-14)
arrives as a new `Machine` variant with no container projection, so its artifact naming is
fixed **with that backend** (additive, FND-6), not by this table.

`Machine.Com()` is a real-mode flat image: only `Kind.executable` is defined for it, and
any other `kind` there is a **Config diagnostic**. Likewise the **freestanding**
`Machine.Bare(…)` has no dynamic loader, so `Kind.shared_lib` there is a **Config
diagnostic** — a freestanding target builds an `executable`, a `static_lib`, an `object`,
or `source`.

An **explicit** `output` is taken **verbatim** — no prefix, no suffix is added, whatever
the `kind` or `container` (what is written is what is produced, I3). It is an artifact
**file name**, not a path: **`/`**, **`\`**, and a **NUL** byte are rejected wherever the
build runs (so a manifest is portable across hosts), as is an **empty** `output` — all
Config diagnostics; the location is `target_dir` (§3.8). Under `kind = source` there is no
artifact, so an explicit `output` is a Config diagnostic there as well.

**`-o <path>`** (Tooling §4) supplies the artifact's **complete path and file name**: no
affix is added and `target_dir` is not consulted for it. Because it names **one** artifact,
`-o` together with a selection that builds more than one (`--target all`, or a multi-target
`--target` selection) is a **Config diagnostic**.

**The entry point** (TOOL-12). `entry` is an **Alatyr path** to a declaration, resolved
by the ordinary module rules (Modules §1, §4): the default `"_start"` names the
declaration `_start` in the **root module** (the manifest file itself, §1), while
`"boot::start"` names `start` in `‹source_dir›/boot.al`. The **linker symbol** is what
that declaration emits — its path-derived spelling (Modules §6.1) or the exact name of
its `@export("…")` (Modules §6.3) — and the toolchain **MUST** pass that symbol to the linker
explicitly (Codegen §6), never relying on the linker's own default entry symbol. A path
naming **no** declaration is a **Codegen-stage diagnostic** raised **before** the linker
runs, with the span on this `entry = …` (or, when the whole manifest is synthesized, the
start of the root source file). `entry` is applicable to `Kind.executable` and
`Kind.object`; on `static_lib` / `shared_lib` / `source` an explicit value is a Config
diagnostic.

**Start-up files —** `startup`, a field of the hosted `Machine` variants (§3.1; TOOL-12):

- **`Startup.raw`** (the default) — **no** platform start-up objects, **no** implicit
  libc; the process entry is the symbol `entry` names, and the entry function's own
  contract is per-target (ABI appendix §3.3). A `Lib(name = "c")` may still be linked
  for pure routines, but libc is then **uninitialized** (no TLS, no `errno`, no
  `.init_array`) — the toolchain surfaces this as a **Config note** (Tooling §5), not an
  error.
- **`Startup.libc`** — the platform's start-up objects **and** libc are linked; they own
  the process entry, so `entry` is **not applicable** (an explicit value is a Config
  diagnostic) and the program supplies the function the platform's start-up calls
  (`main` on ELF and Mach-O; on PE the CRT start-up selected by `subsystem`). libc's link
  mode is `Lib(name = "c")`'s if stated, else the ordinary `LinkMode.static` (§3.5).
- The **freestanding** variants carry no `startup` at all: `Machine.Bare(…)` and
  `Machine.Com()` have no start-up files and no libc to link, so `raw` is not a choice
  there but the only shape — which is why the field lives on the hosted variants only
  (TOOL-18) rather than being a `Target` field with a diagnostic attached.

`Env.gnu` / `Env.musl` are **ABI** parameters (calling conventions, libc-compatible
layouts; ABI appendix §2) — **not** an instruction to link libc. A hosted target with
`startup = raw` is a normal configuration, and it is the default one.

**Android** (`Machine.Android`, TOOL-19) is its own platform variant rather than a flavour
of `Machine.Linux`: the kernel ABI is Linux's (so `@abi(syscall)` uses the Linux table, ABI
appendix §5) but the userland is **bionic**, not glibc/musl — a different libc, a different
dynamic linker, and a platform whose available libc surface depends on the **API level**.
Hence:

- **`api`** — the target **API level** (Android's `minSdkVersion`). It selects the platform
  sysroot the toolchain links against and is reported in the build plan (Tooling §4.2), so
  a packaging step can state the same level it was built for. Its default is the lowest
  level this specification supports (per-arch appendix §1); a lower value is a Config
  diagnostic.
- **`target.env` projects to `Env.bionic`**, so source can gate on the libc it actually has
  (`when target.env == Env.bionic`) rather than inferring it from `os`.
- Artifacts are **position-independent**: an Android `executable` is PIE and a `shared_lib`
  is PIC — the platform's loader requires it, so it is not a knob.
- `startup = raw` (the default) reaches the OS through Linux syscalls, exactly as on
  `Machine.Linux`; `startup = libc` links bionic and its start-up objects, and the
  start-up-called function is `main` (ABI appendix §3.3, the ELF row).

The JNI shape is the ordinary one: `kind = shared_lib` produces `lib‹base›.so` (§3.2), which
an APK later carries. Packaging itself is **not** a `Kind` (TOOL-20).

**Pointer width vs `code_size`.** The machine model's **pointer width is the
architecture's native width** (from `arch`, per-arch appendix §1): `i386`/`aarch32`/
`riscv32` → 32-bit, `x86_64`/`aarch64`/`riscv64` → 64-bit. It is **not** taken from
`code_size`.

`code_size` is the **x86 instruction-encoding mode** (16/32/64-bit code, for
bootloaders/real mode) — a **codegen** knob, **not** part of the machine model: it does
**not** change the pointer width, register widths, or endianness (those are the
triple's, Tooling §1; I6). Its **default is the architecture's native encoding**
(`x86_64` → `b64`, `i386` → `b32`). A **`Machine.Com()`** target (the `i386` real-mode
flat image) **MUST** have `code_size = CodeSize.b16` (its default there; any other value is
a Config diagnostic), emitting 16-bit-encoded instructions; real-mode `segment:offset` addressing
is then a hand-written (`naked`/`asm`) concern below the typed surface (per-arch appendix
§4), not a change to `ptr(T)`. On a **non-x86** architecture `code_size` is not applicable
at all: an explicit value is a Config diagnostic (as is `arm_mode` off `aarch32`).

#### 3.3 Limits and language policy

```alatyr
Limit     := enum { no_abstractions, no_alloc, freestanding, no_unchecked, no_comptime, no_opt }  # Overview §3; FND-10/CG-6/CG-5
```

#### 3.4 Dependencies

```alatyr
GitRef := enum {
  Commit( sha : str )                     # an immutable commit SHA — fully reproducible
  Tag( name : str )                       # a tag; resolved to a commit and pinned by the lockfile
  Branch( name : str )                    # a branch (a moving ref); resolved-and-pinned by the lockfile
}
DepSource := enum {
  Path( dir : str )                       # a path dependency (the on-disk package, used as-is)
  Git( url : str, ref : GitRef )          # a git dependency, selected by ref
}
Dependency := struct {
  name   : str                            # a non-empty valid identifier (Grammar §2.2), unique in this
                                          # manifest, and distinct from `alloc`/`std` and from every
                                          # root-scope name of this package (§3.4 below; MOD-14).
                                          # The LOCAL namespace name in this manifest (Tooling §2.4):
                                          # this dependency's items live under `‹name›::‹module›::…`.
                                          # Package-local naming, never identity — the graph and the
                                          # lockfile key a dependency by its `source` (MOD-10).
  source : DepSource
}
```

**`Dependency.name` is a root-scope name.** A dependency's items are reached as
`‹name›::‹module›::…` (Modules §8), so the name occupies the **package root's** namespace
alongside the root module's own declarations, the child modules of `source_dir`, and the
**ambient `alloc` / `std`** roots (Stdlib §1). It MUST therefore be a **non-empty valid
identifier** (Grammar §2.2), and it MUST be distinct from:

- **another dependency's** name in this manifest — a **Config diagnostic** naming the name
  and both sources (MOD-14);
- **`alloc`** or **`std`** — a **Config diagnostic**: those roots exist for every package,
  so the clash is known before any source is read;
- a **root-module declaration** (including the manifest handle) or a **child module** of
  `source_dir` — the ordinary duplicate-name error (Modules §5), a **Semantic** diagnostic
  naming both sides, since neither side is known at configuration.

The name is still **package-local and never an identity** (MOD-10): these rules constrain
one manifest's own namespace, not the graph.

#### 3.5 Linking (for FFI)

```alatyr
LinkMode := enum { static, dynamic }                     # per-library link mode (MOD-9)
Lib      := struct {
  name : str                                             # the system library name (no `-l` prefix)
  link : LinkMode = LinkMode.static                      # default: absorb the `.a` archive (hermetic)
}
```

`libs : [Lib]` (§3.7) names external libraries, **each with its own link mode** (MOD-9;
Modules §7.5): `LinkMode.static` (the default — absorb the `.a` archive into the binary) or
`LinkMode.dynamic` (reference the `.so`/`.dll`, resolved by the OS loader at run time). A bare
system library thus links **statically by default** — `Lib(name = "m")` — while
`Lib(name = "ssl", link = LinkMode.dynamic)` opts a single library into dynamic linking.

**Binary link-mode rule.** The produced binary is **fully static** (self-contained, no
runtime interpreter, no `.so` dependency) **unless at least one linked library (directly or
transitively) is `dynamic`**, in which case the binary becomes **dynamic**; a `static`
library in a dynamic binary is still absorbed as an archive. The **hermetic-static** build
is the default. A build linking **only `static`** libraries is **hermetic** (byte-for-byte
reproducible with the archive pinned by the lockfile); any **`dynamic`** library makes the
build **non-hermetic**, a status **determinable from the manifest** that the toolchain
**surfaces** (a Config note, Tooling §2.5).

The remaining linking fields — `linker_script`, `linker_flags`, `as_flags` — are plain
strings / string arrays (§3.7); `linker_flags` stays the low-level escape (Tooling §2.5;
FN-8).

#### 3.6 Build profiles and profile flags

```alatyr
FlagDecl := struct {
  name    : str                           # a valid identifier, unique in the manifest (Tooling §2.6)
  type    : type                          # a comptime scalar: bool | an int interpretation | str | a comptime enum
  default : ‹flag-type›                    # a comptime constant of the declared `type` (metavariable, see below)
}
Profile  := struct {
  name        : str                       # "debug" / "release" / custom
  debug_info  : bool = false
  strip       : bool = false
  as_flags    : [str] = []                # assembler flags (non-semantic, §3)
  linker_flags : [str] = []               # linker flags (non-semantic, §3)
  flags       : [FlagSet] = []            # per-profile overrides of declared profile flags
}
FlagSet  := struct { name : str, value : ‹flag-type› }   # set a declared flag (Tooling §2.6)
```

**Meta-notation.** `‹flag-type›` is a **metavariable**, not language syntax: it stands
for "a comptime constant whose type is the value of the same record's (or the named
flag's) `type` field". This is a **dependent** field type — its concrete type is fixed
only once `type` is known — which is why it cannot be written as a plain `type-expr`.
`FlagDecl.type` is itself a **type-valued** field (a type is a comptime value, Comptime
§1, so `type = bool` is data); this is the one place the data subset admits a type name
as a value (§2). A `FlagSet`'s `value` MUST match the `type` of the `FlagDecl` it names
(by `name`), else a Config diagnostic. The profile's `as_flags`/`linker_flags` are the
**per-profile** counterparts of the top-level linking fields (§3.5/§3.7); they feed the
same `as`/`ld` invocation (Codegen §6).

#### 3.7 `Package` field catalog

```alatyr
Package := struct {                            # the manifest type — one value per package (§1)
  version         : str        = "0.1.0"       # default version when omitted
  targets         : [Target]   = ‹[host]›      # default: one target for the build host (§4)
  default_target  : str        = ""            # selection default; "" = the first target (§4)
  license         : str        = ""
  authors         : [str]      = []
  repository      : str        = ""
  description     : str        = ""
  feature_aliases : [FeatureAlias] = []        # package-level alias → opt-in feature set (Tooling §2.2)
  limits          : [Limit]    = []            # the package limits ceiling (FND-11)
  comptime_budget : u64        = ‹impl›        # reproducible step ceiling (Comptime §2.2)
  # (no package-wide `allocator` field in v1: there is no zero-config default provider —
  #  the region allocator needs a caller-supplied buffer (`arena_over`, Stdlib §5.2.1/MEM-3).
  #  Allocation is selected per site via `@alloc(value)`. A default-provider field is
  #  additive (I10/TOOL-4 — a field is added only once implemented), reinstated then.)
  dependencies    : [Dependency] = []
  libs            : [Lib]      = []            # external libraries, each with a link mode (§3.5; MOD-9)
  linker_script   : str        = ""
  linker_flags    : [str]      = []
  as_flags        : [str]      = []
  profiles        : [Profile]  = ‹debug, release›  # the two built-ins exist implicitly
  profile_flags   : [FlagDecl] = []            # manifest-wide flag declarations (§3.6)
  default_profile : str        = "debug"       # used absent `--profile`/`--release`
  # (no `members` / workspace field in v1: multi-package workspaces are deferred post-v1
  #  (TOOL-9) — member discovery, command fan-out, target/profile propagation, output
  #  layout and lockfile ownership are unspecified, and a field without them is an orphan
  #  (TOOL-4). The field is additive and reinstated with those semantics.)
  source_dir      : str        = "src"          # relative to the package root; §3.8 (TOOL-13)
  target_dir      : str        = "target"       # relative; `--target-dir` relocates per invocation
  # (no `vendor_dir` in v1: vendoring is unspecified — nothing says how a dependency's
  #  source is materialized on disk, or how a vendored tree is checked against the
  #  lockfile — so the field would have no implemented effect (TOOL-4/TOOL-16). Where a
  #  git dependency is checked out is an implementation detail until vendoring is
  #  specified; the field and `--vendor-dir` are additive and reinstated with it.)
}
```

**Every field defaults** — the manifest is optional (TOOL-3): an omitted `version` → `"0.1.0"`,
and an omitted/empty `targets` → a single target for the **build host** — its whole
`Machine` variant, with `kind = executable`, `entry = "_start"` and `startup = raw`
(TOOL-12/TOOL-18) — so a trivial program needs no `Package` at all. An *explicit* `targets`
is what a reproducible / cross build states (or `--target`); the host default is a developer
convenience that makes an unconfigured build host-dependent. There is
**no** `Package.name`: the source-visible handle is the binding name (`‹name›.*`), and
the artifact name is each `Target.output` (§3.1) — whose **default base is that same
binding name**, decorated per `kind` × `container` (§3.2; TOOL-11). So the binding
names both the handle and the artifact, and no second name field is needed. The **per-target** fields (`machine`,
`kind`, `entry`, `output`, `features`, `vector_length`, `code_size`, `arm_mode`,
`auto_cfi`) live on each `Target` (§3.1), not on `Package` — and the platform's own fields
(`env`, `subsystem`, `startup`) live inside its `machine` variant (TOOL-18). The `‹…›` defaults
above are metavariables for impl-/context-derived values, not literals.

```alatyr
FeatureAlias := struct { name : str, members : [str] }   # name → set of canonical opt-in feature names
```

**Feature names are validated, not arbitrary.** Each `features` string MUST be a
**canonical opt-in ISA-feature name of the selected architecture**, enumerated in the
per-arch appendix; a name that is **unknown** — or a **baseline query name** (`sse2`/
`neon`, …), which is not an opt-in — is a **Config diagnostic**. A `FeatureAlias` is a
package-local name expanding to a set of **opt-in** features (its `members` are validated
the same way); an alias name MUST NOT collide with any canonical feature name (opt-in or
baseline). The canonical opt-in v1 names include: RISC-V `m`/`f`/`d`/`v`/`a`/`c`;
`i386` `sse`; `x86_64` `avx`/`avx512`/`fma`/`cx16`; `aarch32` `vfp`/`neon`/`idiv`; `aarch64` `sve`.
(`i386` `avx`/`avx512` are **additive** future features, not v1 — per-arch appendix §4.) **Baseline** capabilities (architecturally mandatory — **`sse2`** on
`x86_64`, **`neon`** on `aarch64`) are **not** `features` opt-ins: they are always
present, so `features = []` still has them (per-arch appendix §2). They **are**
present in `target.features` under those canonical names (so source may
`when target.features.has("sse2")`), but they are **query-only** — listing a baseline
name in the manifest `features` is a **Config diagnostic** (it is not an opt-in). Features are **load-bearing**:
they gate instruction/register availability (per-arch tables) **and** select the ABI
float sub-variant (RISC-V `f`/`d` → `lp64f`/`lp64d`; ABI §2(c)).

**Feature closure.** The declared `features` set is resolved to its **transitive
closure** under the architecture's **implication graph** (per-arch appendix; the edges
are **arch-specific**): declaring a feature pulls in the features it implies — e.g.
`aarch32` `neon` ⇒ `vfp`; RISC-V `d` ⇒ `f`; `x86_64` `avx512` ⇒ `avx`, `fma` ⇒ `avx` (SSE2 is baseline,
always present — not an `sse` opt-in). `i386` v1 has only the `sse` opt-in with **no**
implication edges (its `avx`/`avx512`, and their edges, are additive — not v1). The
resolved set **is** `target.features`
(Tooling §2.7) — the comptime set queryable by name — and **every** consumer (instruction
gating, ABI float sub-variant, `eabihf`-requires-`vfp`) reads the **closure**. So
`features = ["neon"]` on an `aarch32` `Target` yields `target.features ⊇ {neon, vfp}`, which
satisfies a hard-float (`eabihf`) target's `vfp` requirement.

#### 3.8 Project paths

`source_dir` / `target_dir` (§3.7) default to **`"src"` / `"target"`**, each **relative to
the package root** — the directory holding the manifest file (§1). A single-file package
needs neither to exist (TOOL-13). (There is no `vendor_dir` in v1 — §3.7; TOOL-16.)

A `*_dir` written in the manifest **MUST** be relative and **MUST** stay inside the
package tree: an absolute path, or a `..` that escapes the root after lexical
normalization (as in MOD-10 — segments folded as text, never through the filesystem), is
a **Config diagnostic**. The manifest is part of the reproducibility tuple (Tooling
§6.2), so a machine-specific path in it would make the package unbuildable elsewhere.

`source_dir` **MUST NOT** contain the manifest file or `target_dir` (lexical containment)
— otherwise a Config diagnostic. Without this check `source_dir = "."` would make the
manifest file a module by its own stem, and would sweep whatever the toolchain writes or
checks out below the package root into *this* package's module tree (Modules §1).

**Relocation is the caller's.** `--target-dir <path>` (Tooling §4) overrides that location
for one invocation and MAY point outside the package. Like `-o`, it changes **where** files
are written, not **what** is built, so it is **not** part of the build input (Tooling
§6.2).

---

### 4. Targets and selection

**`Package.targets` is a non-empty list of fully-specified `Target` values** (§3.1).
Each `Target` carries its **`machine` variant** — the platform, with the fields that
platform actually has — **and** its own build fields (`kind`, `entry`, `output`,
`features`, `vector_length`, `code_size`, `arm_mode`, `auto_cfi`); there is **no** separate
override table — multitarget configuration is expressed by **listing one `Target` per
platform**, each complete. A field omitted on a `Target`, or on its variant, takes its
default (§3.1).

**The list is the allowed set.** A `--target <name>` argument selects the `Target` whose
`name` equals `<name>`; `--target all` builds every `Target` in the list; with neither,
the **`default_target`** is built (absent or `""` → the **first** `Target`). Selecting a
name not in `targets` is a Config diagnostic. `Target.name` is optional only when there
is a **single** target; with two or more, each MUST have a distinct non-empty `name`
(Tooling §2.2).

**Validity, per `Target`.** Its `(machine variant, arch)` pair MUST be a **supported
target** (per-arch appendix §1; Assembly §10). Of the old validity checks only the
**arch-specific** ones remain, because the rest are no longer expressible: an explicit
`code_size` off x86 or `arm_mode` off `aarch32` is a Config diagnostic (Tooling §2.2), and
a **defaulted** inapplicable field is ignored. For example, two targets — a default x86_64
Linux build and an `aarch32` Linux build in Thumb mode:

```alatyr
targets = [
  Target(name = "host",  machine = Machine.Linux(arch = Arch.x86_64, env = Env.gnu)),
  Target(name = "embed", machine = Machine.Linux(arch = Arch.aarch32, env = Env.eabi),
         arm_mode = ArmMode.thumb),
]
```

---

### 5. Evaluation and determinism

- The manifest is evaluated at **configuration** as **pure comptime data** (Comptime §2): no
  wall-clock, environment, randomness, or I/O other than reading the manifest file and,
  transitively, dependency manifests pinned by the lockfile (Tooling §2.4, §6.2).
- Field references (§2) are resolved in **dependency order**; cycles are a Config
  diagnostic.
- The resolved fields fix the **build configuration** (Tooling §1) and publish the
  `target.*` and `build.*` comptime constants (Tooling §2.7) for source to inspect.
- Because evaluation is pure and host-independent, the manifest contributes to a
  **reproducible build**: it is the `manifest` component of the
  `(source, manifest, target, profile, lockfile, toolchain)` tuple (Tooling §6.2).

---

### 6. Example

```alatyr
# package.al — the anonymous package-root module: the manifest value + the code

app := Package(                          # `app` is the source-visible handle: app.*
  version = "0.3.0",
  authors = ["Ada L."],
  license = "MIT",

  targets = [                            # a freestanding aarch64 kernel image
    Target(machine = Machine.Bare(arch = Arch.aarch64),   # freestanding ELF: no `startup`, no container field
           kind = Kind.executable, entry = "_start", output = "tiny-kernel",
           features = ["sve"],           # opt-in; NEON is baseline on aarch64 (per-arch appendix §5.1)
           auto_cfi = true),
  ],
  limits  = [Limit.freestanding, Limit.no_alloc],

  # a custom build flag, defaulted off, set per profile
  profile_flags = [
    FlagDecl(name = "verbose_panic", type = bool, default = false),
  ],
  profiles = [
    Profile(name = "release", strip = true,
            flags = [FlagSet(name = "verbose_panic", value = false)]),
    Profile(name = "debug", debug_info = true,
            flags = [FlagSet(name = "verbose_panic", value = true)]),
  ],
  default_profile = "debug",

  dependencies = [
    Dependency(name = "hal", source = DepSource.Git(url = "https://example/hal", ref = GitRef.Commit("9f1c2a"))),
  ],
  linker_script = "link/aarch64.ld",
)

# ordinary root-module code lives in the same file (single-file package).
# `entry = "_start"` is a PATH: this root-module declaration (§3.2). On a freestanding machine
# (`Machine.Bare`/`Com`) the entry writes its own frame — `@abi(naked)`; `@abi(entry)` is not
# applicable there (ABI appendix §3.3).
@abi(naked) pub _start := fn() {
  comptime if build.verbose_panic { … }  # the resolved profile flag for the chosen profile
  …                                       # app.version, app.license, … read the manifest's own fields
}
```

Source then reads, e.g., `when target.arch == Arch.aarch64 { … }` and
`when target.os == Os.none { … }` — projections of the selected `Machine` variant (§3.1), `comptime if build.verbose_panic { … }` (the chosen profile's flag),
and `app.version` (the manifest handle — the package's own declared fields; Tooling
§2.7).

---

### 7. Conformance

A conforming implementation MUST:

1. accept **at most one** `Package` value per package, found by searching **`package.al`
   upward** from the working directory (the first hit fixes the **package root**) or named
   by **`--manifest <path>`** (its directory becomes the root) — or, for a **bare file
   list**, none at all (Tooling §4; TOOL-14) — in the **anonymous package-root module**
   that also carries the root code, as a UTF-8 file in the lexis of Grammar §2, evaluating
   that value **first** at **configuration**; **zero** `Package` values → synthesize the
   **default package** (§3.7 — host target, `version "0.1.0"`); more than one → a Config
   diagnostic; and reject **`pub`** on the manifest binding (§1; TOOL-15);
2. accept the `Package` value (and any binding it references) only within the **data
   subset** of §2 — data values (literals, aggregate/array/tuple constructors, variant
   constructors, type names for profile-flag types, and references to other manifest
   bindings) — accepting a **call-shaped** value only when its callee resolves to a §3
   config type constructor (struct constructor or enum variant) or an aggregate form, and
   rejecting any other forbidden form (introducer bodies, executable statements,
   `comptime` blocks, attributes, pointer/layout word-function (`ptr`/`deref`/`size`/`align`)/`asm`/instruction calls, a call to a function
   value or any other callee) with a **Config** diagnostic, while leaving the rest of the
   root module as ordinary code (Tooling §5);
3. validate the value against the **schema** of §3 — the `Package` struct, the prelude
   config types, and the field catalog (§3.7) — rejecting an **unknown** field or a
   **type mismatch** with a Config diagnostic; **default** every omitted field (§3.7) —
   an absent `Package`, `version`, or `targets` is **not** an error (the host variant is
   the default, TOOL-4/TOOL-18);
4. resolve the built target(s) from **`targets`** per §4 — `--target <name>` selects by
   `Target.name`, `--target all` builds all, otherwise `default_target` (absent → the
   first); reject a `--target` name absent from `targets`, and require distinct non-empty
   `Target.name`s when there is more than one — take each `Target`'s platform from its
   **`machine` variant**, reject a `(variant, arch)` pair outside the supported set
   (per-arch appendix §1), publish the `target.*` **projections** of that variant
   (§3.1; TOOL-18), and enforce the **arch-specific** rules of §3.2: `Machine.Com()`
   requires `CodeSize.b16` (its default there), `code_size` off x86 and `arm_mode` off
   `aarch32` are rejected — with a Config diagnostic; **validate every `features` / `FeatureAlias.members` name against
   the selected architecture's canonical *opt-in* feature set** (per-arch appendix),
   rejecting with a Config diagnostic both an **unknown** name and a **baseline
   query-only** name (`sse2`/`neon`, …) that appears in a manifest-declared feature list
   — baseline names are queryable in `target.features` but are **not** opt-ins (§3.7);
   features gate instructions and select the RISC-V ABI float sub-variant (ABI §2(c));
   and reject the remaining per-`Target` inconsistencies of §3.2: a `Machine.Com()` target
   whose `kind` is not `executable`, `Kind.shared_lib` on `Machine.Bare(…)`, an explicit
   `entry` on a library kind or under `startup = libc`, and an `output` that is empty or carries
   a path separator (`/`, `\`, NUL) or that is stated at all under `kind = source`;
4a. resolve the **artifact name** per §3.2 — the base being the manifest binding's name, or
   the root source file's stem where there is no binding, decorated by the
   `kind` × `container` affix, with an explicit `output` taken verbatim (TOOL-11) — and the
   **entry** per §3.2 as an Alatyr **path** whose emitted symbol is stated to the linker,
   honouring `startup` (`raw` / `libc`, TOOL-12);
4b. enforce §3.8's **path** rules — `source_dir`/`target_dir` relative, inside the package
   tree, and `source_dir` containing neither the manifest file nor `target_dir` — with a
   Config diagnostic (TOOL-13);
5. resolve inter-field references in **dependency order**, rejecting cycles, and
   evaluate the manifest as **pure, host-independent** comptime data, publishing the
   manifest handle as an **ordinary non-`pub` root-module declaration** — `‹name›.*`,
   visible to the package by down-tree privacy and to nothing outside it (TOOL-15) — and
   publishing `target.*` and `build.*` **unconditionally** (§5; Tooling §2.7), and
   contributing the `manifest` component of the reproducible-build tuple (Tooling §6.2).
