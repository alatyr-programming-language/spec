# Alatyr Language Specification

## Chapter — Tooling

> **Status — draft (under review).** This chapter is a draft and is not yet
> accepted; the specification as a whole is under development. See the status note
> at the top of the Overview chapter.

This chapter defines the **manifest** (configuration → build plan), the **CLI**, the
**diagnostic** model, the **cross-toolchain** handoff, and **reproducible builds**.
The manifest is read **first** and is the **single source of truth for the package's
configuration**; the full **build input** is the manifest **plus the build invocation**
(the selected target and profile, §1). The manifest declares the **`targets`** list and
the **`default_target`**; the **selected** target — once chosen — fixes the **machine
model** the rest of the language is defined over (Overview §3; I6).

It builds on the limits model (Overview §3; decisions FND-10), the comptime budget
(Comptime §2.2), the allocator model (Stdlib §5), dependencies and symbols
(Modules), and the codegen handoff (Codegen §6).

The manifest is **a single `Package` value** bound in the package's root module — by
default the file `package.al` (CLI `--manifest` overrides), which **is** the anonymous
package-root module and so carries the package's code too (TOOL-3; manifest appendix §1).
Its concrete syntax is a tooling detail (v1 tooling appendix). This chapter is normative
for the **field set and their meaning**, the CLI **behavior**, and the diagnostic/build
**contracts**. Requirement keywords follow RFC 2119 (Overview §6).

---

### 1. The manifest is the configuration phase

The manifest is processed in the **configuration** phase (Codegen §1.1) — the first
phase of the one compilation pipeline, and the **`Config`** diagnostic stage (§5) —
before parsing any source it fixes the build configuration. It is the **same Alatyr**
(one grammar/AST, FND-10/FND-9), a data subset of it (TOOL-3), **evaluated first** (as restricted
comptime) because the machine model it selects is an input to every later phase. It declares the **target**(s) — a triple
`arch + os + env + container` — which fixes the **machine model** (native widths,
registers, endianness, ISA features; I6) the type system and codegen are defined over;
and the unit's **limits ceiling**, the **comptime budget**, the **dependencies**, and
the **output**. It is the **single source of
truth for the package's configuration** — fields are not duplicated elsewhere, and a
field is added only when it is implemented (decisions TOOL-4). It is **at most one
`Package` value** (zero synthesizes the default package; more than one is a Config
diagnostic; TOOL-3); the binding name is the source-visible handle (§2.7) — an ordinary
non-`pub` root-module declaration, so `pub` on it is a Config diagnostic (TOOL-15).

The full **build input** is the **manifest plus the build invocation**: the target is
a build parameter — the manifest gives the `targets` list and `default_target`, and
`--target` (§4) selects one (or `all`) per build. This is legitimate
cross-compilation: the **output varies by target by design** (different machine model
→ different `when`-gated code, widths, container). It is **not** a case of a package
"depending on a flag": the package's *configuration* is the manifest, and the target
is which valid build of it you ask for. (Contrast the **debug-only** budget/limits CLI
overrides, §4, which a package's success must **not** depend on.)

---

### 2. Manifest fields

#### 2.1 Package metadata

`version`, `license`, `authors`, `repository`, `description`. There is **no** package
`name` field: the source-visible handle is the binding name (§2.7), and the artifact
name is each target's `output` — whose **default base is that same binding name**
(TOOL-11; Manifest §3.2), so the one binding names both.

#### 2.2 Targets and output

- **`targets`** (defaulted; non-empty after defaulting) — a list of **`Target`**
  values; the **list is the allowed set** of builds. Each `Target` carries a
  **`machine` variant** — the platform, and with it the machine model (FND-8; TOOL-18):
  `Machine.Linux(arch, env, startup)` / `Freebsd` / `Windows(arch, subsystem, startup)` /
  `Macos(arch, startup)` / `Android(arch, api, startup)` / `Bare(arch, env)` / `Com()`, so a field that belongs to one
  platform (the PE `subsystem`, the libc `env`, `startup`) exists only in the variant that
  has it, and there is no `os` / `container` / `endian` field at all (they are `target.*`
  projections, §2.7). Defaulting is **by variant**: `Target()` is the build host's whole
  variant, and a cross build states another. Beside `machine`, each `Target` carries its
  own build fields:
  - **`name`** — the `--target` selection key (optional only when there is a single
    target; with two or more, each MUST be distinct and non-empty);
  - **`kind`** — `executable` / `static_lib` / `shared_lib` / `object` / `source`
    (where `source` merges this package's sources into the consumer, so it produces **no**
    artifact: `build`/`run` on such a target is a Config diagnostic pointing at `check`).
    The `kind` also fixes **which symbols land in the artifact** — in particular the
    program's entry is excluded from a library (Modules §6.4; MOD-13). Not every `kind`
    fits every target: `Container.com` admits only `executable`, and `Os.none` — having no
    dynamic loader — admits everything but `shared_lib` (Manifest §3.2);
  - **`entry`** — the entry point, given as an **Alatyr path** to a declaration
    (default `"_start"`, the root module's; `"boot::start"` names `start` in
    `‹source_dir›/boot.al`). The toolchain derives the **linker symbol** that
    declaration emits (Modules §6.1, or its exact `@export("…")`, Modules §6.3) and passes it
    to the linker explicitly; a path naming no declaration is a **Codegen**-stage
    diagnostic raised before the linker runs, spanned on the manifest's `entry = …`
    (TOOL-12);
  - **`startup`** — a field of the **hosted** `machine` variants: `raw` (the default:
    **no** platform start-up objects and no implicit libc; the process entry is what
    `entry` names) or `libc` (the platform's start-up objects **and** libc are linked; they
    own the process entry, so `entry` is not applicable and the program supplies the
    function that start-up calls, by exact name and form — `main` on ELF and Mach-O, the
    `subsystem`-selected form on PE, enumerated in ABI appendix §3.3; that function is the
    artifact's root exactly as `entry` is under `raw`, Modules §6.4). The freestanding
    variants (`Bare`/`Com`) have no such field — there are no start-up files there to
    link. A variant's **`env` is an ABI parameter, not a link instruction** (TOOL-12);
  - **`output`** — the artifact **file name**, not a path; omitted, it is the manifest
    binding's name plus the `kind` × `container` affix, and stated, it is taken verbatim
    (TOOL-11, Manifest §3.2);
  - **`features`** — per-arch ISA extensions (the package-level **`feature_aliases`**
    field — a `Package` field, Manifest §3.7 — expands here), **`vector_length`**
    (SVE/RVV), and the two **arch-specific** codegen knobs **`code_size`** (16/32/64,
    x86 only — bootloaders/real mode) and **`arm_mode`** (`arm`/`thumb`, `aarch32` only),
    plus **`auto_cfi`** (emit control-flow-integrity directives);
- **`default_target`** — names the `--target`-less default; absent or `""` → the **first**
  `Target`. `--target <name>` selects one; `--target all` builds every target (§4).

Each `Target` is **fully self-contained** — there is no separate per-target override
table; multitarget configuration is just **multiple `Target` values**, one per platform.
Since the platform is a variant, most old "inapplicable field" cases can no longer be
written (a `subsystem` off PE, an `endian` contradicting the arch, a `startup` on a
freestanding machine, `pe` with `gnu`). What remains is checked: an **explicitly provided**
`code_size` off x86, `arm_mode` off `aarch32`, or `entry` on `kind = static_lib` /
`shared_lib` / `source` or under `startup = libc` (TOOL-12) is a **Config-stage
diagnostic** (§5), as is a `(machine variant, arch)` pair outside the supported set
(per-arch appendix §1). A **defaulted** inapplicable field is ignored for that target.

#### 2.3 Language policy

- **`limits`** — the package's limits ceiling (`no_abstractions` / `no_alloc` /
  `freestanding` / `no_unchecked` / `no_comptime` / `no_opt` / …; Overview §3); a file's
  `@limits(…)` may only be **stricter** (FND-11);
- the default **comptime budget** (a reproducible step ceiling; Comptime §2.2);
- **no** package-wide default allocator in v1; allocation is selected explicitly at each
  `@alloc(value)` site (Stdlib §5).

#### 2.4 Dependencies

A dependency is selected entirely by its **source**: a **path** (the on-disk package,
used as-is) or a **git** source with a **`GitRef`** — `Commit` / `Tag` / `Branch`. There
is **no per-dependency version constraint in v1** (a path has nothing to select against;
a git dep selects by ref); version *constraints* and a registry-based resolver are
**additive** (FND-6). A `Tag` or `Branch` ref is **resolved to a commit and pinned by the
hashed lockfile** (Modules §8) — so the build is reproducible (§6.2) — while a `Commit`
ref is already immutable. A dependency's items live under its **local namespace name**
(`<name>::<module>::…`), not flatly merged — one naming field, `Dependency.name`, which is
this manifest's own choice and never an identity (MOD-14). **The graph is acyclic** (MOD-11): a dependency chain that returns to a package already
on it is a **Config diagnostic** naming the chain, never a silently deduplicated edge.

**Identity in the graph is the source** (MOD-10): a git dependency is the URL exactly as
written, a path dependency is its lexically-normalized absolute path. "Lexically" is exact: the
absolute base is the process's working directory (which the OS has already resolved), while the
path's own segments — `.`, `..` — are folded as **text**, never through the filesystem. So two
spellings that differ by a symlink *below* the working directory are two distinct sources, by the
same deliberate choice that makes a git URL byte-comparable rather than normalized. A
dependency's local namespace name is package-local and never identifies a package — two manifests may call one source differently,
or one name may cover two sources, without affecting resolution — that is a statement about the graph
being keyed by source ACROSS manifests. **Within one manifest the name is a root-scope
name**: it MUST be a non-empty valid identifier and MUST differ from another dependency's
name, from the ambient `alloc`/`std` roots (Stdlib §1), and from any root-module
declaration or `source_dir` child module of this package — the first three are Config
diagnostics, the last a Semantic one (Manifest §3.4; Modules §5). Two
dependencies sharing it would put two distinct declarations under the same `<name>::<module>::item`
path, which is MOD-8's no-duplicate-names rule applied to that namespace, so it is a **Config
diagnostic naming the name and both sources**. Without the rule the collision surfaces as an assembler
error about a duplicate symbol, which names neither the name nor either dependency. If one **source** resolves
to two different commits across the graph (a `Branch`/`Tag` pinned differently by two
consumers), that is a **Config diagnostic** naming both requesting packages and both
resolved commits — there is no automatic version unification in v1.

**The lockfile** (`alatyr.lock`, beside the root manifest) records each git
**source**'s **resolved commit** — one entry per source, its `<url>` then the resolved
`<commit>` SHA, in a stable order (sorted by `<url>`, deduplicated). The **dependency's name never
appears**: that name is one manifest's local naming choice, while the lock is graph-wide,
so the graph key is the **source** (MOD-10). When a lockfile is present, a `Tag`/`Branch` ref is checked out at
its **locked commit** rather than re-resolved, so the build is reproducible (§6.2)
until the lock is regenerated; a `Commit` ref is already immutable, and a `path`
dependency (local, on-disk) is not locked. The resolved commit SHA **is** the
content hash — a git commit is content-addressed — so the lock pins the exact
dependency tree.

The on-disk **format is normative** (so two conforming toolchains write a
byte-identical lock for the same resolved graph, FND-3): the file is **UTF-8** text
with **`LF`** (`U+000A`) line terminators. A line whose first non-whitespace
character is `#` is a **comment**, and a blank line is **ignored** (the writer
emits a single leading comment line; readers MUST tolerate any comment / blank
lines). Each **entry line** is two fields separated by a single **horizontal tab**
(`U+0009`) and terminated by `LF`:

```
<url>\t<commit>\n
```

`<url>` is the git URL **verbatim** as written in the manifest and `<commit>` the **full
resolved commit SHA** (lowercase hex). Entry lines are sorted **ascending by `<url>`**
(byte-wise over the UTF-8 encoding) and **deduplicated by `<url>`** (one entry per source
— a diamond shares one, whatever each consumer called it). On read, a line is **trimmed**
of surrounding whitespace, then split on tabs; a line with fewer than two fields is
**skipped** (the lock is regenerated on the next resolve) and a field MUST NOT contain a
tab or `LF`. A graph with **no** git dependencies writes
**no** lockfile (a path-only graph needs none).

#### 2.5 Linking (for FFI)

`libs` (external libraries — a list of **`Lib`** values, each a `name` (no `-l` prefix)
plus a **link mode** `static`/`dynamic`; MOD-9, manifest appendix §3.5), `linker_script`,
`linker_flags`, `as_flags`; the link graph is **de-duplicated**. These feed the `as`/`ld`
invocation (Codegen §6; Modules §7.5).

**Link mode and hermeticity.** Each library's link mode is **`static`** by default (its
`.a` archive is absorbed into the binary) or **`dynamic`** (the `.so`/`.dll` is referenced
and resolved by the OS loader at run time). The produced binary is **fully static** —
self-contained, no runtime interpreter — **unless** some linked library (directly or
transitively) is `dynamic`, which makes the binary **dynamic** (Modules §7.5). A build
linking **only `static`** libraries is a **hermetic build** (byte-for-byte reproducible with
the archive pinned by the lockfile, §6.2); any **`dynamic`** library makes it
**non-hermetic** (its runtime behavior depends on the host's installed library). This status
is **determinable from the manifest**, and the toolchain **surfaces it** as a **Config
note** (§5) whenever a `dynamic` library is linked — so the reproducibility guarantee's
*scope* is explicit, never silently eroded.

#### 2.6 Build and project

- **Build profiles** (`debug` / `release` + custom) — tune **non-semantic** policy
  only: `debug_info`, `strip`, and toolchain flags (§3);
- **Profile flags** — declared in **two parts**:
  - a manifest-wide **declaration** of each flag: a **name** (a valid identifier,
    unique within the manifest; it MUST NOT shadow the built-in
    `build.debug`/`build.profile`), a **type** (a comptime scalar — `bool`, an integer
    interpretation, `str`, or a comptime `enum`), and a **default** value (a
    comptime-constant literal of that type);
  - a **per-profile override**: a profile MAY set any declared flag to another
    comptime-constant value of the declared type. A profile that does not override a
    flag uses its declared default.

  Each declared flag is exposed as the comptime constant `build.<name>` (§2.7); its
  value is resolved from the **selected profile** at configuration and is constant for the
  build. (Conceptually: a top-level `profile_flags` declares name/type/default, and
  `profiles.<name>` supplies the overrides.);
- **`default_profile`** — the profile used when the CLI gives no `--profile`/`--release`
  (§4); absent, it is **`debug`**;
- **Paths** — `source_dir` / `target_dir`, defaulting to **`"src"` / `"target"`** relative
  to the package root. A `*_dir` in the manifest MUST be relative and stay inside the
  package tree, and `source_dir` MUST NOT contain the manifest file or `target_dir` —
  otherwise a Config diagnostic. Relocation outside the package is the **caller's**
  choice, via `--target-dir` (§4), which changes where files are written and never what is
  built (TOOL-13; Manifest §3.8). There is **no** `vendor_dir` in v1: vendoring is
  unspecified, so the field would have no implemented effect (TOOL-16);
- a content-addressable **`.s`/`.o` cache** (an implementation detail).

#### 2.7 Comptime-visible configuration

The configuration phase publishes read-only **comptime constants** that source may inspect (e.g. in
`when` guards and `comptime if`). They are the *contract* between the manifest and the
language for build-conditional code; the standard library and user code depend on them,
so they are defined here normatively.

In addition to the two namespaces below, the **manifest binding itself** is in scope as
a comptime constant under its **binding name** — the source-visible handle `‹name›.*`
(TOOL-4): `‹name›.version`, `‹name›.targets`, … expose the package's **own declared
fields**. This is distinct from `target.*` (the *resolved* selected target) and
`build.*` (the *selected* profile): the handle is the static manifest data as written.

The handle's **scope** is that of an **ordinary root-module declaration**, not that of
`target.*`/`build.*` (TOOL-15): being non-`pub` it is visible to the root module and — by
down-tree privacy (Modules §3) — to **every module of the package**, and to nothing
outside it, whereas `target.*`/`build.*` are published unconditionally to every module of
every package in the build. **`pub` on the manifest binding is a Config diagnostic**:
`Package` is a configuration-prelude type (Manifest §3), so the value cannot be part of a
package's public API — a library publishes metadata by re-exporting the fields it means
(`pub VERSION := ‹name›.version`, an ordinary `str`). The handle's name shares the root
module's scope with `‹source_dir›`'s child modules, so a same-named module is a
duplicate-name error (Modules §5).

- **`target.*`** — the **selected `Target`**, exposed as comptime values. The **machine
  model** first, as **projections of the `machine` variant** (§2.2; TOOL-18):
  `target.arch`, `target.os`, `target.env`, `target.container`, the native
  widths (e.g. `target.ptr_width`), `target.endian`, and the resolved ISA `target.features`
  (a set queryable by name — the **transitive closure** of the declared features under
  the per-arch implication graph, **plus baseline capabilities** under their canonical
  query names, e.g. `sse2`/`neon`, which are query-only and not manifest opt-ins;
  per-arch appendix §2, Manifest §3.7). `target.features` is a **comptime feature set**
  (a `FeatureSet` comptime value); its sole query is **`has`** —
  `has(comptime name : str) -> bool`, used as `target.features.has("sse2")` (UFCS, CT-10)
  — a comptime `bool` true iff `name` is in the closure (an unknown / misspelled name is
  simply `false`, not an error; the set is closed by the per-arch table). Membership is the
  only operation (no enumeration / negation surface in v1 — additive, FND-6). These are the same facts the type system and codegen are
  defined over (I6); a different target produces different `target.*`, hence the output
  legitimately varies by target (§1). Typical use: `when target.arch == Arch.x86_64`.
  **A non-ISA backend has no `Arch` identity** (CG-14). `Arch` enumerates v1's register ISAs; an additive
  structured/VM backend such as WASM is not one of them and is not selectable as an `arch` of any
  `Machine` variant (Manifest §3.1; TOOL-18). When
  such a backend is driven directly, `target.arch` therefore equals **no** `Arch` variant: every
  `target.arch == Arch.<v>` folds **false** and every `!=` folds **true**, which keeps ordinary portable
  code (`when target.arch == Arch.x86_64 { … } else { … }`) meaningful there. The consequence is explicit:
  a `match` over `target.arch` is exhaustive only for ISA targets, so portable code MUST give it a default
  arm — without one it is ill-formed on a non-ISA backend. **`target.container` behaves the same way**
  there (it matches no `Container` variant, since the four containers are the register-ISA ones, Manifest
  §3.2), and **`target.startup`** is inapplicable — a backend with no start-up objects and no libc has
  neither mode. Such a backend arrives as a new **`Machine` variant** (Manifest §3.1; TOOL-18) whose
  projections are fixed together with it (CG-14; FND-6).

  Beyond the machine model, `target.*` exposes the selected target's **build fields**
  (§2.2; MOD-13), and the set is exactly: **`target.kind`**, **`target.startup`**
  (`raw` on a freestanding variant, which has no other), `target.subsystem` (the PE
  variant's, `console` elsewhere), `target.code_size`, `target.arm_mode`,
  `target.vector_length`. So
  source gates on the **artifact** with the same construct it uses for the machine —
  `when target.kind == Kind.static_lib { … }` — which is how a package that builds both a
  program and a library expresses anything finer than the emission table of Modules §6.4.
  The remaining fields are **not** published: `name` and `output` name the build and its
  file rather than describing the program, `entry` is a path into the program's own module
  tree (gating on it would let code depend on which declaration is the entry), and
  `auto_cfi` is a non-semantic codegen knob, like the profile's flags (§3). The
  **`machine` variant itself** is likewise not published as a value: source compares the
  projections (`target.os`, `target.container`) rather than matching a platform variant, so
  a new platform does not change what portable code must handle. The comparison enums
  (`Kind`, `Startup`, `Arch`, `Os`, `Container`, …) are ordinary prelude names, not
  manifest-only types (Manifest §3).
- **`build.*`** — non-semantic build policy from the selected profile (§2.6, §3):
  - `build.debug : bool` — true under the `debug` profile (used by `assert`'s
    debug-only form, Stdlib §4.3);
  - `build.profile : str` — the profile name (`"debug"` / `"release"` / custom);
  - each **profile flag** declared in the manifest (name/type/default + per-profile
    override, §2.6) is exposed as a `build.<name>` comptime constant of its declared
    type, resolved from the selected profile.

  `build.*` is **non-semantic** policy: the compiler and the selected profile **never**
  strip the checked-guard family or an emitted `assert` (§3, Stdlib §4.3).
  The only sanctioned semantics-affecting use is a **source-visible**
  `comptime if build.debug { assert(…) }` written by the program author — the
  conditional is in the source and the author owns its effect; this is not the profile
  removing a guard or an assert. No `build.*` value otherwise changes a program's
  observable semantics (FND-10/FND-7).

Both namespaces are fixed by configuration and are constant for the whole build; they are
part of the build input (§1), so they do not compromise reproducibility (§6.2) or
diagnostic determinism (§5).

---

### 3. Build profiles do not change semantics

A build profile (`debug`/`release`/…) tunes only **non-semantic** policy — the **optimization level**
(semantically-preserving, CG-5 — it changes cost, never observable behavior), debug information, symbol
stripping, and `as`/`ld` flags. (The byte-exact floor is `no_abstractions`; un-optimized output is the
`no_opt` limit — Codegen §3.)

A profile **MUST NOT** *itself* change program **semantics** (FND-10/FND-7): in particular,
the **checked-guard family** (overflow, bounds, alignment, narrowing — Concurrency §6,
Type System §4.2 (narrowing) and §6.4 (bounds)) and an emitted `assert` (Stdlib §4.3) are **not** stripped
by a profile — stripping them would change behavior (a violated invariant would
continue instead of trapping). The compiler never makes an observable-behavior
difference between profiles on its own.

The **one** way a profile can affect runtime behavior is when the **source author
explicitly gates on `build.*`** — e.g. `comptime if build.debug { assert(…) }` (§2.7,
Stdlib §4.3). Then the difference is **in the source**, owned by the author
and visible at the gate; the profile is not silently stripping anything. So: absent
any source-visible `build.*` gating, **observable behavior is identical across
profiles** (profiles then differ only in artifacts — debug info, symbols — and
toolchain flags); where such a gate exists, behavior differs exactly as the source
prescribes.

---

### 4. The CLI

Commands: **`new`** / **`build`** / **`run`** / **`test`** / **`check`** / **`plan`** /
 **`fmt`**.

- **`check`** runs configuration, parse and **semantic analysis** for the selected target
  and profile and reports the diagnostics of those stages (§5) — it does **not** lower,
  emit, assemble or link, and produces **no artifact** (not even under `-o`). It is
  therefore the command for a `kind = source` target, which has no artifact to build
  (§2.2), and the fast well-formedness gate for any other kind. Two consequences are
  explicit: a **Codegen**-stage diagnostic (an `entry` path that names no declaration,
  §2.2) is **not** reported by `check`, since that stage does not run; and `check` writes
  nothing under `target_dir` except what the `.s`/`.o` cache (§2.6) may already hold.

- **`--manifest <path>`** names the manifest file explicitly; its directory becomes the
  package root. Without it, **`package.al` is searched for from the current working
  directory upward** to the filesystem root, and the first one found fixes the package
  root — so a command behaves the same from any subdirectory of a package (TOOL-14).
- **A manifest-less invocation** — a command given a **bare file list**
  (`alatyr build a.al b.al …`) — reads **no** manifest and performs no discovery.
  Configuration synthesizes the default `Package` (TOOL-3) with the **first listed file**
  as the root module and its directory as the **package root** — so `source_dir` is that
  directory (submodules still resolve by path, Modules §1) and `target_dir` resolves
  beneath it — while that root file is **excluded** from module-path scanning, exactly as a
  manifest file is (Manifest §3.8), so it is not also a module by its own stem. Then:
  `kind = executable`, `startup = raw`, `entry = "_start"`, the artifact base = that file's
  **stem** (TOOL-11), the host triple, and the `debug` profile unless `--profile` /
  `--release` says otherwise. Artifacts land under that `target_dir` like any other build
  unless `-o` names one; an implementation MAY instead use a temporary location, which it
  MUST clean up (TOOL-10). Dependencies cannot be declared
  in this mode, so no lockfile is read or written. A bare file list **together with**
  `--manifest` is a Config diagnostic, and an invocation with neither a discoverable
  manifest nor a file list is a Config diagnostic — never a silent no-op (TOOL-14). Both are
  **invocation-level**: there is no source to point at, so they are the one exception to the
  mandatory-span rule (§5).
- **`--target <name>`** selects one of the manifest's `targets` by its `Target.name`;
  **`--target all`** builds every target; absent, the `default_target` is built (absent
  or `""` → the first). A name not in `targets` is a Config diagnostic. This is a
  **legitimate build parameter** (cross-compilation), not a debug-only override; the
  output varies by target by design (§1).
- **`--profile <name>`** selects the build profile (§2.6); it is likewise a
  **legitimate build parameter** (it is part of the build input, §6.2, and fixes
  `build.*`, §2.7), not a debug-only override. **`--release`** is shorthand for
  `--profile release`. The **default** profile when none is given is **`debug`** for
  every command (`build`/`run`/`test`/`check`); a manifest MAY declare a different
  default. The selected profile name is reported in `build.profile` (§2.7).
- **Temporary overrides** of the comptime **budget** or **limits** are for **debugging
  only**: a package's *success* **MUST NOT** depend on such a flag (that would make it
  non-self-contained; Comptime §2.2, FND-7) — the manifest is authoritative for these.
- **Cross-run**: for a non-host target, `run`/`test` execute under **QEMU**.
- **Where artifacts land** (TOOL-10, TOOL-13). Every file a command produces for a package — the
  executable or library, the intermediate `.s`/`.o`, and the **test artifact** `alatyr test` builds — is
  written under the package's **`target_dir`**, with a deterministic name, and is left there for
  inspection. Deleting `target_dir` therefore removes everything the toolchain made, and a test artifact
  is an ordinary inspectable artifact rather than a hidden one (I3). The **layout** inside it is
  `‹target_dir›/‹profile›/` when **`Package.targets` has one element**, and
  `‹target_dir›/‹target-name›/‹profile›/` when it has **two or more** (where each `Target.name` is then
  already mandatory, §2.2) — the layout follows the **manifest**, not which target this invocation
  selected, so a path does not shift with `--target`, and `--target all` and a profile switch never
  overwrite each other. A build's `.s`/`.o` sit beside the artifact they produce, named by the **module
  path** (`geometry__vec.s` / `.o`, the mangling of Modules §6.1), so they cannot collide with a
  `kind = object` artifact. `-o <path>` names the artifact's **complete path and name** — no affix added,
  `target_dir` not consulted, and a Config diagnostic when the invocation builds more than one artifact
  (Manifest §3.2); **`--target-dir <path>`**
  relocates that directory for the invocation and MAY point outside the package (the manifest itself may
  not, Manifest §3.8). Neither affects what is built, so both are outside the build input (§6.2). For an invocation with **no** manifest (a bare file list), an
  implementation MAY use a temporary location instead — but it MUST remove everything it created before
  exiting, so repeated invocations do not accumulate files.
- **Lockfile modes** — `--locked` / `--offline` / `--frozen` (TOOL-8). They restrict two
  independent capabilities of a resolve, **writing the lockfile** and **using the
  network**; nothing else about the build changes, and a build that succeeds under a mode
  produces the **same** artifact it would without it:

  | mode | may write `alatyr.lock` | may use the network | a needed source only reachable by fetching | the lock would have to change |
  |------|------------------------|---------------------|--------------------------------------------|-------------------------------|
  | *(none)* | yes | yes | fetched | rewritten |
  | `--locked` | no | yes | fetched | **Config diagnostic** |
  | `--offline` | yes | no | **Config diagnostic** | rewritten |
  | `--frozen` | no | no | **Config diagnostic** | **Config diagnostic** |

  "The lock would have to change" is any difference between the resolved graph and the
  lockfile on disk: a source with no entry, an entry for a source no longer in the graph,
  a `Branch`/`Tag` that resolves to a different commit, or a malformed line that would be
  regenerated (§2.4). The diagnostic names what would have changed. `--frozen` is exactly
  `--locked` **plus** `--offline`, not a third policy. A path-only graph writes no
  lockfile, so `--locked` and `--frozen` are satisfied trivially by it.
- **Job controls** (additive): `-j` (parallelism) / `-k` (keep-going).

#### 4.1 Tests (`alatyr test`)

A **test** is an anonymous runtime function carrying the **`@test("description")`**
attribute as a top-level item (Declarations §2.3; Grammar §3.2; TOOL-5):

```alatyr
@test("adds two numbers") fn() {
  assert(add(2, 3) == 5)
}
```

- It has **no name** — the **description string is its label** (in the report and for
  selecting one test). It takes **no parameters** and is **not** `comptime`/generic;
  otherwise it is a Semantic diagnostic. Its result is either **nothing** (a hard-fail test:
  a trap is the only failure signal) or **`Result(usize, str)`** (a **soft-fail** test, below);
  any other result type is a Semantic diagnostic. It is a **runtime** function, distinct from
  a compile-time `comptime { assert(…) }` check (which asserts at build time and needs no
  test framework).
- **Discovery.** `alatyr build` / `run` **ignore** `@test` items — they are not emitted
  (zero binary cost, I2; like an unused library item). `alatyr test` **collects every
  `@test` item across the package**, builds a test artifact, and runs them. A test sees
  its module's private items by ordinary down-tree visibility (Modules §3), so it can test
  internals. `alatyr test [<substring>]` runs all tests, or those whose description
  contains `<substring>`; descriptions need not be unique. A cross-target `test` runs
  under QEMU (§4, §6.1).
- **The test artifact owns its entry** (TOOL-7). It is an artifact of `kind`
  **`executable`** whose root is the runner's entry, and the `@test` items are additional
  roots (Modules §6.4). What `alatyr test` builds is a
  **separate artifact**, not the package's executable with tests bolted on: the runner
  supplies the **entry point**, and the package's own entry — the declaration the manifest's
  `Target.entry` names (§2.2) and any entry the program declares itself — is **not linked
  into it**. This
  mirrors `build`/`run` ignoring `@test` items: each artifact excludes what belongs to the
  other, so a program that declares its own entry is testable without a symbol collision.
  Nothing else changes: `main` and every other item are ordinary functions, linked when a
  test (or something a test calls) reaches them under the normal reachability rules.
- **Isolation and outcome** (I11). Each test runs **in isolation** so a failing one does
  not prevent the others (one process per test): a **trap** — a failed `assert`, an
  overflow/bounds trap, or `panic` — is observed as **that** test's **failure**. A void test
  that returns normally **passes**. A **soft-fail** test (result `Result(usize, str)`)
  **passes** on `Ok` and **fails** on `Err(message)`, where the **message** is reported as the
  failure detail — so a single test may run several checks and report a failure (the first, or
  a joined message) **without trapping** the process. The runner reports each test's
  description, outcome, and detail (the soft-fail `Err` message, or a trapping test's stderr,
  e.g. a `panic` message), plus a passed/failed summary.

#### 4.2 The build plan (`--plan`)

A build **produces a machine-readable description of what it produced**: the **build plan**
(TOOL-20). `alatyr build --plan` writes it beside the artifacts (`‹target_dir›/…/plan.tsv`,
per the layout of §4) in addition to building; `alatyr plan` writes only the plan, doing the
configuration and the resolution but no codegen. It is **output**, never input: nothing in a
build reads it, and its presence or absence changes no artifact.

It exists because everything downstream of the compiler — packaging (`.deb`, an APK, a
container image), installation, CI attestation — needs to know *what was built, for what,
and where*, and the only alternative is scraping paths and re-deriving the affix rules.
Packaging itself is **not** a `Kind` and not a command of this specification (TOOL-20): it
belongs to a separate tool, or to a future `publish`-style command, and this file is its
input.

**Format.** UTF-8 text, `LF` terminators, one **record per line**; a line whose first
non-whitespace character is `#` is a comment and a blank line is ignored. A record is a
**record type** followed by tab-separated (`U+0009`) fields:

```
meta	‹key›	‹value›
artifact	‹target-name›	‹profile›	‹kind›	‹path›	‹install-path›
dylib	‹target-name›	‹library-name›
```

- **`meta`** — one line per fact, keys fixed by this specification: `plan-version`
  (`1` for v1), `package-version` (`Package.version`), `machine`, `arch`, `os`, `env`,
  `container`, `api` (only where the platform has one, TOOL-19), `profile`,
  `hermetic` (`yes`/`no`, §2.5), `toolchain` (the `as`/`ld` identification of §6.1).
- **`artifact`** — one line per produced artifact: its target's `name` (empty for a
  single unnamed target), the profile, its `kind`, its path **relative to `target_dir`**,
  and its **install path** — the relative location a packaging or install step should place
  it under: `bin/‹file›` for an `executable`, `lib/‹file›` for a `static_lib` /
  `shared_lib`, and empty for `object` (an intermediate for someone else's link) and for
  `source` (no artifact). The install path is **derived**, not configured: v1 has no
  install section in the manifest, and a packaging tool that needs another layout maps it.
- **`dylib`** — one line per dynamically-linked library the artifact needs (MOD-9), so a
  packager can state runtime dependencies without inspecting the binary.

Records are sorted by type in the order above, then field by field (byte-wise over the
UTF-8 encoding); the file is therefore **byte-identical for the same build input**, like the
lockfile (§2.4). `plan-version` is bumped when a record type or field is added, so a reader
can refuse a plan it does not understand rather than guessing.

#### 4.3 Canonical source format (`alatyr fmt`)

`alatyr fmt` rewrites Alatyr source to **the** canonical form. The format is **a single
non-configurable canonical** (the gofmt model, TOOL-2): there is **no** style option, in
the manifest or on the CLI — one well-formed program has exactly **one** formatted text.
This is **normative** (FND-3): two conforming `fmt` implementations produce **byte-identical**
output for the same input. `fmt` is **semantics-preserving** (the reformatted text parses to
the same AST modulo formatting) and **comment-preserving** (§4.2.4), and it is
**idempotent** — `fmt(fmt(x)) == fmt(x)`. It operates per `.al` file; with no path argument
it formats every `.al` file under the package root. A file that does not parse is **left
unchanged** and reported as a diagnostic (`fmt` never emits ill-formed source).

##### 4.3.1 Lexical frame

The output is **UTF-8** with **`LF`** (`U+000A`) line terminators, **exactly one** trailing
`LF`, and **no** trailing whitespace on any line. Indentation is **two spaces** per nesting
level — **never** tabs, and never any other width; one level is added inside each
brace-delimited block and each continuation (§4.3.3). There are **no leading** blank lines;
**at most one** consecutive blank line anywhere; **one** blank line between top-level items
(none is inserted before the first item or after the last). A blank line is **not** emitted
immediately after an opening `{` or before a closing `}`.

##### 4.3.2 Spacing (single-line form)

- **One** space on each side of a binary operator, of `:=` / `=`, of `:` in a binding or
  parameter type (`x : T`), and of `->` in a result type; **one** space after `,` and after
  `;`, and **none** before them.
- **No** space between a callee/array and its `(` / `[`, and **no** space just inside
  `(` `)` `[` `]` (`f(a, b)`, `xs[i]`, `a[lo..hi]`); a unary prefix operator hugs its operand
  (`-x`, `not p`); `..` in a range carries **no** surrounding space.
- A block `{` sits on the **same line** as its header (`if`, `while`, `for`, `match`, a `fn`
  value, a `struct`/`enum`), preceded by **one** space; the matching `}` is on **its own
  line** at the header's indent. An **empty** block is `{}` (no inner space).
- `match` arms are `pattern => body`, one space on each side of `=>`.

##### 4.3.3 Wrapping (the 100-column rule)

The **soft maximum line width is 100 columns** (counting Unicode scalar values, not bytes).
A construct that fits within 100 columns at its indent is written on **one** line. One that
does **not** is wrapped by the following deterministic rule, so the choice to wrap is a pure
function of width (never of the input's original line breaks):

- A **call argument list**, an **array/list literal**, a **struct/enum constructor's
  fields**, and a **parameter list** wrap with **each element on its own line**, indented
  **one** level beyond the opening line, the closing `)` / `]` at the opening line's indent.
  A wrapped list carries a **trailing comma** after its last element (a single-line list does
  **not**); this keeps element-wise diffs minimal and is itself canonical.
- A **binary-operator chain** that overflows breaks **before** each operator at the
  continuation indent (one level), the operator beginning the continuation line.
- A **`match` arm list** wraps with **each arm on its own line**, indented one level beyond
  the `match` line, the closing `}` at the `match`'s indent. An arm whose body is an `expr`
  ends with a **trailing comma** — including the last (as for a wrapped list above); an arm
  whose body is a `block` ends with its `}` and takes **no** comma. An **OR-pattern**
  (`p1 | p2 | …`, Grammar §3.5) that still overflows breaks **before** each `|` at one
  further level, the `|` beginning the continuation line, with `=>` and the body following
  the last alternative. An arm **body** that still overflows wraps by its own construct's
  rule; it is never moved onto the pattern's line or split from its `=>`.
- A **`comptime for` arm template** inside a `match` arm list (Grammar §3.5; CT-9) wraps as
  a block: `comptime for x in ‹collection› {` on the arm line, its **arm** bodies one level
  deeper by the arm rule above, and `}` at the template's indent. It is never collapsed to
  one line when the `match` is wrapped, even if it would fit — a template and a plain arm
  are visually distinct because they mean different things.
- An **inline value-`if`** (`if c { a } else { b }` in expression position, Grammar §3.4)
  that overflows wraps with the condition on the `if` line and **each branch's block
  brace-delimited on its own lines**, one level deeper, `else` on the line closing the
  previous branch (`} else {`) — the same shape a statement-position `if` has, so an
  expression `if` never grows a second layout. A trailing `else if` chain continues at the
  same indent (`} else if c2 {`), never nesting one level per link.
- Wrapping is applied at the **outermost** construct that makes the line fit; an inner
  construct wraps only if it still overflows once the outer one has. A construct is never
  wrapped if it fits — **except** the `comptime for` arm template, above.

##### 4.3.4 Comments

Comments are **preserved verbatim** (text and kind) and kept **attached** to the construct
they document: a comment on its own line(s) immediately above an item stays immediately
above it at that item's indent; a trailing same-line comment stays on that line, separated
from the code by **one** space. `fmt` never deletes, merges, or rewrites the **content** of a
comment (only its surrounding indentation/whitespace is normalized). A comment **suppresses**
re-wrapping of the line it trails (the line is kept as the author wrote it modulo the spacing
rules) so a deliberately-laid-out, commented line is not reflowed.

---

### 5. Diagnostics

A diagnostic is **a stage + a mandatory source span + a message**:

- the **stage** is one of **Config / Lex / Parse / Semantic / Codegen**;
- the **span** (a source location/range) is **mandatory** — every diagnostic points
  to source. The **one exception** is an **invocation-level** Config diagnostic, where no
  source exists to point at (no manifest and no file list; a file list together with
  `--manifest`; §4): it names the offending argument, or the working directory, in the
  span's place (TOOL-14);
- diagnostics are **deterministic** over the **same build input** — the tuple
  `(source, manifest, target, profile, lockfile, toolchain)` (the same one that fixes a
  reproducible build, §6.2) — yields the same diagnostics (consistent with the pure
  pipeline, Codegen §3). Semantic/Codegen-stage diagnostics legitimately depend on the
  resolved dependencies (lockfile), the target, and the selected profile (which fixes
  `build.*`, §2.7), so they are deterministic *given* that tuple, not for
  source+manifest alone.
- The **debug-only** budget/limits overrides (§4) are **not** part of this tuple,
  because a package's success must not depend on them (§4, Comptime §2.2). Applying one
  is a debugging action that **may** itself change diagnostics — e.g. a tighter budget
  yields a *budget-exhausted* diagnostic (Comptime §2.2) — and that is **by design**,
  not a determinism violation. The guarantee is: with the tuple fixed **and** the same
  overrides (none, in a normal build), the diagnostics are identical.

**What conformance requires (and what is QoI).** The cross-implementation conformance
point is **agreement on well-formedness**: a program is **ill-formed iff a conforming
implementation rejects it**, so all conforming implementations agree on **accept /
reject** for a given build input (this is part of "compatible results without
guessing", FND-3). The **presentation** of diagnostics — their **order**, **how many** are
reported (one vs all), and the message wording — is **quality-of-implementation**, not
portable: it is only required to be **deterministic per implementation** (the bullets
above), not identical across implementations. A conforming implementation **SHOULD**
order diagnostics by ascending source span (start line, then column), tie-broken by
stage then message, and **SHOULD** report at least the first — but tooling **MUST NOT**
assume two implementations select the same "first error".

---

### 6. Cross-toolchain and reproducible builds

#### 6.1 Toolchain handoff

For a **register-ISA backend** (the v1 instance), the compiler emits **GAS text** and
shells out to **`as`** then **`ld`** (CG-1/CG-4; Codegen §6) — it does not encode or link
itself. Other backends use their own documented toolchain handoff (CG-4).

**Cross-toolchain prefix detection (normative).** The tool prefix is derived from the
target's machine model as the **GNU triple** `<arch>-<os>-<env>` (the canonical spellings of
appendix §170; e.g. `aarch64-linux-gnu`). The compiler resolves the assembler/linker in this
order, on `$PATH`:

1. when the target **equals the host** machine model — the bare **`as`** / **`ld`**;
2. otherwise the **triple-prefixed** tools **`<triple>-as`** / **`<triple>-ld`** (e.g.
   `aarch64-linux-gnu-as`); a per-arch appendix MAY list **known-alias** triples tried in a
   fixed, documented order (deterministic — no first-found-on-`PATH` ambiguity);
3. if neither resolves — a **Tooling diagnostic** naming the exact tool(s) expected for the
   target (never a silent fallback to the host `as`/`ld`, which would mis-encode).

The resolved toolchain is part of the reproducible-build input (§6.2).

**QEMU cross-run (normative).** For a **non-host** target, `run` and `test` execute the
linked binary under **QEMU user-mode emulation** — the per-arch binary **`qemu-<arch>`** (e.g.
`qemu-aarch64`; the `<arch>` spellings of §170), resolved on `$PATH`. It is invoked as
`qemu-<arch> <output-binary> <args…>`, the program's own argv passed through unchanged, and
the program's exit status / stdout / stderr are propagated verbatim (so `test`'s pass/fail is
the emulated process's, §4). `qemu-system` (full-system emulation) is **not** used. If
`qemu-<arch>` is not found, `run`/`test` fail with a **Tooling diagnostic** ("QEMU not found
for `<target>`"), **never** a silent skip or a spurious pass. A **host** target runs the
binary directly (no QEMU).

#### 6.2 Reproducible builds

A build is **reproducible**: the same
`(source, manifest, target, profile, lockfile, toolchain)` produces **bit-identical**
output. (The target and selected profile are part of the build input, §1/§2.6 — a
different target or profile legitimately produces different output: different machine
model, or different artifacts and `build.*` constants.) This follows from the **pure, deterministic**
pipeline — comptime evaluation is pure and host-independent (Comptime §2; float by
Concurrency §7.1), codegen is deterministic (Codegen §3), and dependencies are pinned
by the hashed lockfile (§2.4). No wall-clock, environment, or randomness enters the
build (Comptime §2.4).

#### 6.3 Incremental build and the module cache

Separate compilation is per **module**: each module compiles independently to its own
`.s`/`.o`, and a deterministic link produces the artifact — so compile memory is bounded
per module (not the whole tree) and independent modules may compile in parallel. A build
MAY cache compiled module outputs in the content-addressable `.s`/`.o` store (§2.6). A
module's cache entry is keyed by the hash of its own source together with the transitive
**interface hashes** of its dependencies. A module's **interface** is everything a
dependent can observe — exported signatures; struct/enum **layout** (offsets/size/align,
ABI-affecting under the layout attributes, Types §8); generic signatures and demanded
instantiations; `@inline` bodies; and comptime-observable facts — hashed as **one** value.

The cache is **output-neutral**: a build that reuses cached outputs MUST produce
**bit-identical** output to a build with an empty cache — the cache may only make a build
*faster*, never change its *output*. It is therefore a local performance artifact only: it
is **not** part of the build input (§1), **not** part of the lockfile, and cannot affect
reproducibility (§6.2), which rests on `(source, manifest, target, profile, lockfile,
toolchain)` alone. A monomorphized generic instance is emitted by the module that **uses**
it, as a mergeable (weak) symbol deduplicated at link, so a module stays independently
compilable and cacheable. (Decision TOOL-6.)

---

### 7. Conformance (normative summary)

A conforming implementation MUST:

1. process the manifest at **configuration** as the single source of truth for the
   **package configuration** — **at most one `Package` value** in the root module
   (zero synthesizes the default package; >1 is a Config diagnostic), declaring the
   `targets` list and `default_target` (the selected
   target, via `default_target` or `--target`, fixes the machine model), the limits
   ceiling, comptime budget, dependencies, and output; the full
   **build input** is the manifest **plus the build invocation** (selected target and
   profile) (§1, §2; I6/TOOL-4/TOOL-3);
2. support the manifest field set of §2 — package metadata (no `name`); **`targets`**
   (defaulted to a non-empty list of `Target`, each carrying a **`machine` variant** —
   `Linux`/`Freebsd`/`Windows`/`Macos`/`Bare`/`Com`, holding that platform's own
   `arch`/`env`/`subsystem`/`startup` — plus the build fields
   `name`/`kind`/`entry`/`output`/`features`/`vector_length`/`code_size`/`arm_mode`/`auto_cfi`,
   and publishing the `target.*` projections of the variant; TOOL-18)
   and `default_target`; limits/budget; dependencies + hashed lockfile; linking
   fields; build profiles + profile flags (name/type/default + per-profile
   override)/`default_profile`/paths (§2) — applying each `Target`'s fields to
   the **selected** target and rejecting inapplicable/inconsistent ones with a Config
   diagnostic (§2.2, §5);
3. treat build profiles as **non-semantic** in themselves — never stripping the
   checked-guard family or an emitted `assert`, and never making an observable-behavior
   difference between profiles on the compiler's own initiative; the only
   profile-dependent behavior is what the source author explicitly gates on `build.*`
   (e.g. `comptime if build.debug`) (§2.7, §3; CG-5/FND-7);
4. provide the CLI commands (`new`/`build`/`run`/`test`/`check`/`plan`/`fmt`), `--manifest`,
   `--target <name>`/`--target all` (selecting from `targets`, else `default_target`),
   `--profile`/`--release` (default profile `debug`, or the manifest's
   `default_profile`), `-o`/`--target-dir` (location only, never build
   input), debug-only budget/limit overrides that a package's success does
   **not** depend on, QEMU cross-run, and the lockfile/job modes (§4); and run **`@test`**
   items as **isolated runtime tests** under `alatyr test` (a trap = a failure), while
   `build`/`run` ignore them (§4.1; TOOL-5);
5. discover `package.al` by searching **upward** from the working directory (the first
   found fixes the package root), or accept a **bare file list** as a fully-specified
   manifest-less build (first file = root module, `kind = executable`, `startup = raw`,
   `entry = "_start"`, artifact base = that file's stem, host triple, `debug`), rejecting
   a file list combined with `--manifest`, and rejecting an invocation with neither (§4;
   TOOL-14);
6. name each artifact per **TOOL-11** — the manifest binding's name (or the root file's
   stem) plus the `kind` × `container` affix, an explicit `output` **verbatim** — and
   write every produced file under `target_dir` in the `‹profile›` /
   `‹target-name›/‹profile›` layout, with the project paths defaulting to
   `src`/`target`, relative and package-local (§2.6, §4; TOOL-13);
7. resolve `Target.entry` as an **Alatyr path**, state the derived symbol to the linker
   explicitly, and diagnose a path that names no declaration at the **Codegen** stage
   before the linker runs; honor `startup` (`raw` = no start-up files and no implicit
   libc; `libc` = start-up files + libc own the entry, so `entry` is not applicable;
   `Os.none` admits only `raw`), and support **`@abi(entry)`** where the container defines
   an entry contract (§2.2; TOOL-12/FN-12; ABI appendix §3.3);
8. write the **build plan** on `--plan` (and as the sole output of `plan`) in the normative
   record format of §4.2 — `meta` / `artifact` / `dylib`, deterministically ordered and
   byte-identical for one build input, with each artifact's derived **install path** — as
   **output only**: no build reads it and its presence changes no artifact (TOOL-20);
9. emit every diagnostic as **stage + mandatory span + message** — the one exception being
   an invocation-level Config diagnostic (§5) — deterministically (§5);
10. emit GAS and shell out to `as`/`ld` with cross-toolchain prefix detection, and
   guarantee **reproducible, bit-identical** builds for the same
   `(source, manifest, target, profile, lockfile, toolchain)` (§6; CG-1).
