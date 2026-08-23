# Decisions — Tooling: manifest, build, packages, tests, CLI

**Why the language is the way it is** for the toolchain — manifest, build plan, packages,
tests, the CLI, diagnostics. Tooling is a *first-class subject of the specification*, not
an addendum: its contract is specified normatively on a par with language semantics. Each
entry records the *rationale* and the *rejected alternatives*; the normative *what* lives
in the spec (linked per entry), the *how* in the compiler. IDs are theme-prefixed
(`TOOL-N`) and **append-only** — assigned in articulation order, never renumbered. The
`provenance:` line records what an entry refines and where it lands in `spec/`.

---

## Manifest, build, packages, tests, CLI

### TOOL-1. The toolchain contract: manifest → build-plan, CLI, diagnostics, reproducibility
*spec: Tooling §1–§7*

**Why.** A systems toolchain must be specified as precisely as the language so that two
independent implementations behave compatibly without guessing (the FND-3 bar). The contract
is fixed in four parts so nothing in the build is left implicit. (1) **The manifest is the
configuration phase** — it declares, *declaratively*, the full build plan: package
metadata, the build **targets** (each a machine model, stated as a `Machine` variant — TOOL-18), the
orthogonal **limits**, the comptime budget, **dependencies** with a hashed lockfile, and
output — and the toolchain turns it into a plan. (2) **The CLI** is a fixed verb set
(`new`/`build`/`run`/`test`/`check`/`fmt`) plus build parameters; **the full build input =
the manifest + the invocation** (the chosen target + profile), so a build is a pure
function of stated inputs. `--target` is a *legitimate build parameter*, not a debug-only
override — the manifest's `targets` list is the allowed set and `default_target` the
default. Budget/limit overrides exist **only for debugging**: a package's success must not
depend on them (a package must build on any conforming toolchain from its declared inputs
alone). (3) **Diagnostics** separate the portable contract from presentation: the
**conformance point is well-formedness** (accept/reject must agree across conforming
implementations — a program is ill-formed iff a conforming impl rejects it), while
diagnostic *presentation* (order, count, wording) is QoI — deterministic per-impl but not
portable. (4) **Reproducible builds**: the toolchain shells out into the host `as`/`ld`
(it does not re-implement assembly/linking — CG-1) and uses a hashed lockfile whose
*on-disk format is itself normative* (UTF-8, LF, tab-separated, sorted), so two conforming
toolchains emit a byte-identical lock for the same resolved graph.

**Rejected.** Profiles that change observable behavior — a profile may carry only a
**non-semantic** optimization level (it changes cost, never behavior); behavior differs
across builds only through source-visible `build.*`-gating, never silently per profile.
Treating `--target` as a debug override — it is a first-class build parameter. A toolchain
that owns assembly/linking itself — rejected in favour of shelling out to `as`/`ld` (CG-1).

### TOOL-2. `fmt` is a single non-configurable canonical form
*spec: Tooling §4.3*

*(Extended: §4.3.3's wrapping rule now also covers a `match` arm list, a `comptime for` arm template and an
inline value-`if` — the three overflowing forms it originally omitted. A single canonical form means the
wrapped spelling of **every** construct that can overflow is the specification's to fix, not the
formatter's: with the rule absent, an implementation had to invent one, which is a formatter deciding
language surface. The arm rule is element-per-line with a trailing comma on expression arms (matching lists)
and none after a block arm; an OR-pattern breaks before each `|`; the template stays a block even when it
would fit, because a template and a plain arm mean different things; and an expression `if` reuses the
statement `if` layout so no construct has two shapes.)*

**Why.** Formatting style is a perennial source of churn and per-project divergence.
Following the gofmt model, `fmt` rewrites source to **one** canonical form with **no style
option anywhere**: fixed 2-space indent, a 100-column soft width, semantics- and
comment-preserving, idempotent, and **byte-identical across conforming implementations**
(the FND-3 bar applied to formatting). A single canonical form removes the configuration
surface entirely and makes formatted output portable.

**Rejected.** Any style option or configurable formatter — it would reintroduce the
divergence the canonical form exists to eliminate.

### TOOL-3. The manifest is data-subset Alatyr — a single `Package` value, optional
*spec: Tooling §1–§2, Manifest appendix §3.7*

**Why.** A separate manifest format (TOML/JSON) would duplicate machinery that already
exists: the lexis and grammar of literals, declarations, and aggregates. By minimalism
(OP-1), the manifest is therefore **written in Alatyr itself** as a data-subset — pure
configuration data (literals, array/tuple/struct/variant constructors, references to other
manifest bindings) in ordinary qualified Alatyr, with no manifest-only sugar. The manifest
is a **single `Package` value** bound in the package's root module (`package.al` by
default, `--manifest` overridable). That root file **is** the anonymous package-root
module, so it carries the `Package` value *and* the root module's ordinary code — a
**single-file package** (manifest + code in one file) is normal. The manifest is
**optional** and **everything defaults**: zero `Package` values → a default package is
synthesized; an omitted `version` → `"0.1.0"`; omitted/empty `targets` and an omitted
triple → the build host's whole `Machine` variant (executable, `_start`; TOOL-18) — so a trivial
single-file program needs no
manifest at all (OP-1 — no ceremony for the common case). The `Package` binding is
**comptime by nature**, and its **name is the source-visible handle** read as
`package_name.*` — which is why there is **no `Package.name` field** (the binding name is
the handle; the artifact name is `Target.output`). The data must be **self-contained
comptime**: it cannot read `target.*` because it *defines* the targets. An unconfigured
build is thereby host-dependent (a developer convenience, like default `cargo`/`zig
build`); a reproducible or cross build states the target explicitly, restoring the
host-independent configuration that feeds the reproducibility tuple.

**Rejected.** A separate manifest format (TOML/JSON) — duplicates existing
lexis/grammar/aggregate machinery (OP-1). A `Target` as a triple-string — a string would
need a second mini-grammar; a struct-of-enums is data already expressible and directly
yields `target.*`. A `Package.name` field — the handle is the binding name, the artifact
name is `Target.output`. Manifest-only sugar — rejected for ordinary qualified Alatyr.

### TOOL-4. The manifest field catalog — one source of truth, no orphan fields
*spec: Tooling §2, Manifest appendix §3.7*

**Why.** Manifest fields breed faster than the implementation behind them:
declared-but-unimplemented knobs accumulate and rot. So the field catalog is fixed
as **one source of truth**, every field tied to a real build effect. The catalog spans:
**targets** (`targets : [Target]` is the allowed set, `default_target` the default; each
`Target` carries a **`Machine` variant** — the platform, with that platform's own
`arch`/`env`/`subsystem`/`startup` inside it (TOOL-18 superseded the flat
arch/os/env/container fields) — plus build fields like `output`, `entry`, `kind`,
`features`, `vector_length`, `code_size`, `arm_mode`);
**linking** for FFI (`libs : [Lib]` — each external library with a per-library **link mode**,
`static`/`dynamic`, MOD-9; `linker_script`, `linker_flags`, `as_flags`); **build/project**
controls (build-profiles that configure only non-semantic things, profile-flags surfaced as
the comptime constant `build.<name>`, workspaces, paths, a content-addressable `.s`/`.o`
cache); and **metadata** (`license`/`authors`/`repository`/`description`). A field that is
inapplicable or inconsistent with a target's machine model is a Config-diagnostic. One
careful distinction: **`code_size` (16/32/64) is a codegen knob, NOT part of the machine
model** — pointer/register width and endianness come from the arch/triple (I6), while
`code_size` is the x86 encoding mode (arch-native default; `Container.com` ⇒ `b16`
mandatory for real-mode); conflating the two would let an encoding knob masquerade as a
machine-model fact.

**Rejected.** Letting fields be declared without an implemented effect — the prior
iteration's failure mode, replaced by a single implemented catalog. `code_size` as part of
the machine model — it is a codegen knob; the machine model owns pointer/register width and
endianness (I6). (Supersedes the earlier `target`+`allowed_targets` shape in favour of
`targets`+`default_target`.)

### TOOL-5. The test construct and runner — `@test`, isolated, trap-or-`Err`
*spec: Tooling §4 (the `test` command) + the Tests subsection*

**Why.** The Tooling chapter listed a `test` CLI command, but the language had **no test
construct** for it to run — a FND-3 gap this closes by pinning both the construct and the
runner's contract. A test is an **anonymous runtime function carrying
`@test("description")`** as a top-level item — the *one* name-less top-level item form: it
has no binding name, the **description string is its label**. By minimalism (OP-1) this
reuses the existing attribute + a name-less item rather than inventing syntax, and the
free-form description avoids forcing an identifier per test. A `@test` function takes no
parameters, returns nothing (or a soft-fail result), and is **not** comptime/generic — it
is a *runtime* function built into the test artifact, deliberately distinct from a
compile-time `comptime { assert(…) }` check. **Discovery is explicit, not magic**:
`build`/`run` ignore `@test` items (zero binary cost, I2, like any unused item); `alatyr
test` collects every `@test` item across the package's modules (a test sees its module's
private items by ordinary down-tree visibility, so it can test internals), builds a test
artifact, and runs them; an optional substring filters by description. **Isolation and
outcome rest on I11**: each test runs in isolation (one process per test) so a failing test
does not block the others, and a **trap** (failed `assert`, overflow/bounds trap, or
`panic`) is observed as *that* test's failure. A void test passes on normal return; a
**soft-fail** test (result `Result(usize, str)`) passes on `Ok` and fails on `Err(message)`
— reporting a failure *without trapping the process*, so one test may run several checks and
report detail. The result is thus either nothing or `Result(usize, str)`; any other result
type is rejected.

**Rejected.** A dedicated `test "name" { … }` block — new keyword/syntax for what the
attribute + a name-less item already express (against OP-1). `@test` on a *named* function —
forces inventing an identifier per test and cannot carry a free-form description.
Discovery by name convention (`test_*`) — magic, with a silent drop on a typo (against I3).

### TOOL-6. Per-module compilation + an output-neutral content-addressable cache (incremental build)
*provenance: TOOL-1, TOOL-4, MOD-7 · spec: Tooling §build-model*

**Why.** A whole-tree build — concatenate every module, one AST arena, one monolithic GAS
buffer, emit in a single pass — is simple and reproducible but does not scale: it is
**memory-unbounded** (a large tree overruns a fixed build arena) and **O(whole-tree) on
every build**, so a one-line change recompiles everything and the dev cycle is minutes.
TOOL-4 already reserves a **content-addressable `.s`/`.o` cache** as a build control; this
decision fixes its *contract* so it can be implemented without leaving anything implicit,
and pins the **module** as the unit of separate compilation. Three properties are settled:

1. **Per-module compilation.** Each module compiles independently to its own `.s`/`.o` in a
   per-module arena, then a deterministic link produces the artifact. This bounds compile
   memory to one module (not the tree — retiring the whole-tree overflow class), enables
   **parallelism** across independent modules, and is the substrate for incrementality.

2. **The module INTERFACE is the invalidation unit, hashed as ONE value.** A module `M`'s
   cache entry is keyed by `hash(M's own source + the transitive INTERFACE hashes of M's
   dependencies)`. A module's **interface** is the single combined hash over everything a
   dependent can observe: exported signatures; **struct/enum LAYOUT** (offsets/size/align —
   `@packed`/`@offset`/`@align`/`@repr` make layout part of the ABI, so a field reorder in a
   dependency must invalidate a dependent even with an unchanged API); generic signatures +
   the instantiations demanded; **`@inline` bodies** (inlined across module boundaries); and
   comptime-observable facts (`when`-predicates, `typeinfo`-visible properties). It is ONE
   hash, not a split API/ABI pair: in a **transparent-struct** language nearly all
   cross-module struct use depends on layout, so a split buys little while adding a
   correctness failure mode (a missed sub-hash = a silent stale-cache miscompile).

3. **The cache is OUTPUT-NEUTRAL and orthogonal to reproducibility.** The `.cache/` is a
   **local, disposable** performance artifact (gitignored, never committed, never shipped).
   Its hard invariant: a build **using** the cache is **byte-identical** to a build after
   `rm -rf .cache/` — the cache may only make a build *faster*, never change its *output*
   (verifiable: a `--no-cache`/clean build must reproduce a cached one). **Reproducibility
   comes from the lockfile + source (TOOL-1/MOD-7), not from the cache** — the two are
   separate concerns, so the cache is NOT part of the lockfile (folding it in would make the
   lockfile machine-specific and break "same lockfile → same build across machines"). The
   TOOL-1 fixpoint / whole-build reproducibility check is unaffected by the cache.

4. **Cross-module monomorphization: the USING module owns its instances.** A generic in `A`
   instantiated with a type from `B` and used in `C` is emitted by **`C`** (the using
   module) as a **weak/COMDAT symbol**; the **linker deduplicates** if several modules emit
   the same instance. This keeps per-module incrementality (each module emits what it uses
   and rebuilds independently) — the instance's demand is part of `C`'s interface-dependency
   set (a change to `A`'s generic signature or `B`'s layout invalidates `C`'s instance). The
   cost is some redundant *codegen* (two modules may compile the same instance) but the
   linked binary carries one copy.

**Staging (risk after value).** Step 1 = per-module EMIT with NO cache: delivers bounded
memory + parallelism at **zero stale-cache risk** (a mostly-mechanical refactor of the
monolithic emit into per-module `.o` + deterministic link + weak-symbol instances). Step 2 =
the content-hash cache on top, guarded by the output-neutrality invariant and **adversarial
cache-invalidation tests** (deliberately change a layout / an `@inline` body / a generic
signature and assert the dependent's entry invalidates) — this is where the
correctness-critical hashing lives, done second and carefully.

**Rejected.** Folding the cache into the lockfile — conflates a local speed artifact with the
reproducibility contract and makes the lockfile machine-specific. A **split** API-hash vs
ABI/layout-hash from the start — over-engineering for a transparent-struct language, buying
little finer-grained invalidation while adding a silent-stale-miscompile failure mode
(additive later, only if profiling shows layout-only changes over-rebuild). A shared
**mono-module** or **defining-module** owner for monomorphized instances — both need a
whole-program view of instantiation demand, re-centralizing what per-module compilation
decentralizes and fighting incrementality; the using-module + linker-dedup model keeps each
module independent. (A content-addressed **instance cache** — reuse one compiled instance
across modules to kill the redundant codegen — is a sound *additive* optimization on top of
the using-module model, deferred until measured worthwhile.)

### TOOL-7. The test artifact supplies its own entry; the program's entry is not linked into it
*provenance: refines TOOL-5 · spec: Tooling §4.1; Manifest appendix §3.2 (`Target.entry`)*

**Why.** TOOL-5 fixed one half of the separation — `build`/`run` ignore `@test` items — and
left the other half unstated: what the test artifact does with the *program's* entry. The
runner needs an entry of its own (it starts the process that runs one test), and a program
is free to declare one too (`entry = "_start"` is the default, and a freestanding program
may write the symbol itself). With nothing said, the two land in one artifact and the
assembler rejects the duplicate symbol — a program is unbuildable under `test` for a reason
that has nothing to do with its tests, and two implementations may "solve" it differently
(drop one, rename one, refuse). Symmetry decides it: `build` drops what belongs to `test`,
so `test` drops what belongs to `build`.

**The rule.** The artifact `alatyr test` builds is a distinct artifact whose entry point is
the **runner's**. The package's declared entry — the declaration `Target.entry` names (TOOL-12) and any entry the
program defines itself — is **not linked into it**. Everything else is unchanged and
unprivileged: `main` is an ordinary function, present in the test artifact only if a test
reaches it under the normal reachability rules. Conversely `build`/`run` keep the program's
entry and drop the `@test` items (TOOL-5), so neither artifact is the other with additions.

**Rejected.** Making the runner yield the symbol to the program (calling the program's entry
as the test process's start) — the entry's contract is to run the program, not a test.
Renaming one of the two — a hidden symbol rewrite, against I3, and it would surface in a
freestanding program's own linker script. Rejecting a program that declares an entry when
`test` runs — punishes exactly the low-level programs the language targets. Emitting both
and letting the assembler decide — the failure the rule exists to prevent.

### TOOL-8. The lockfile modes are two independent capabilities — writing the lock, using the network
*provenance: refines TOOL-1/TOOL-4 and MOD-7/MOD-10 · spec: Tooling §4 (CLI), §2.4*

**Why.** `--frozen` / `--locked` / `--offline` were **named** in the CLI surface with no
semantics attached, which is worse than absent: a user reaches for them precisely in CI, to
make a build fail rather than drift, and three implementations would each pick a different
read/write/fetch matrix. The three names also invite being treated as three policies on one
axis, which they are not.

**The rule.** There are exactly **two** capabilities a resolve may or may not have —
**writing `alatyr.lock`** and **using the network** — and the flags are their combinations:
`--locked` withholds writing, `--offline` withholds the network, `--frozen` withholds both
(it is their conjunction, not a third policy). Withholding a capability that the resolve
would have needed is a **Config diagnostic** naming what it would have done: the entry it
would have written, or the source it would have fetched. Nothing else about the build
changes — a build that succeeds under a mode is byte-identical to the same build without
it, so the modes are a *guard*, never a build parameter (§6.2). "The lock would have to
change" is defined concretely (a missing entry, a stale entry, a ref resolving to a
different commit, a malformed line) so two implementations agree on when a mode fires.

**Rejected.** One flag with a mode word (`--lock=strict|offline|frozen`) — the capabilities
are genuinely independent, so a single axis either loses a combination or invents a
meaningless one. `--locked` implying `--offline` — a locked build legitimately fetches the
*locked* commit on a fresh machine, which is the normal CI shape. Silently succeeding under
`--locked` when the lock is stale (just not writing the file) — the build would then differ
from what the lockfile records, which is the one thing the flag exists to prevent. Leaving
the modes additive/post-v1 — they are already named in the v1 CLI surface, and a named flag
without semantics is an orphan (TOOL-4).

### TOOL-9. Multi-package workspaces are post-v1; v1 has no `members` field
*provenance: refines TOOL-3/TOOL-4; parallel to MOD-7's registry deferral · spec: Tooling §2.6;
Manifest appendix §3.2*

**Why.** The manifest carried `members : [str]` and the configuration list carried a
"Workspaces" bullet, but nothing else: member discovery, how a command fans out over
members, whether targets and profiles propagate, where the outputs land, how member
dependency edges interact with the graph, failure ordering across members, and which
package owns the lockfile were all unwritten. That is a field with a name and no
semantics — precisely the orphan TOOL-4 forbids — and it is not a small gap: each of those
questions is a decision, and several interact with MOD-10 (graph identity) and TOOL-8 (who
may write the lock).

**The rule.** v1 has **no** workspace surface: the `members` field is removed from the
`Package` schema and the configuration list, exactly as MOD-7 defers a registry and version
constraints. A repository holding several packages builds them as several packages, each
with its own manifest, wired by ordinary **path dependencies** — which already work and are
already specified. Workspaces are **additive** (I10/FND-6): the field is reinstated together
with its semantics, and no v1 manifest breaks when it is.

**Rejected.** Specifying workspaces now — the surface must be designed against real
multi-package experience, and guessing it locks a shape into the frozen v1. Keeping the
field as a documented no-op — a manifest that accepts `members` and ignores it silently
misleads (I3), and a later implementation could not tell an intentional value from a stale
one. Making it implementation-defined — the FND-3 bar again.

### TOOL-10. Every artifact lands under `target_dir`; a manifest-less run cleans up after itself
*provenance: refines TOOL-3/TOOL-4 (`target_dir`) and TOOL-5/TOOL-7 (the test artifact) · spec:
Tooling §4; Manifest appendix §3.8*

**Why.** The manifest has a `target_dir` and the chapter says artifacts are built into it, but nothing said
whether that covers the files `run` and `test` produce on the way — and an implementation read that silence
the obvious way: it compiled into an OS temporary directory under a process-id name and never removed the
result. Three things follow, all observable. The toolchain accumulates files outside the package
indefinitely (thousands, in a working session). `target_dir` stops being the complete answer to "what did the
build make", so cleaning is no longer deleting one directory. And the **test artifact becomes uninspectable**,
which matters more than housekeeping: TOOL-7 makes a normative claim about what that artifact contains (the
runner's entry, not the program's), and a contract that cannot be observed cannot be gated — the exclusion had
to be verified through a probe of a temporary file that the next run overwrites.

**The rule.** For a package, every produced file — the executable or library, the intermediate `.s`/`.o`, and
the test artifact — is written under `target_dir` with a deterministic name and left there; `-o <path>`
overrides the location for the artifact it names. (**Refined by TOOL-13**, which fixes what `target_dir`
defaults to, the per-target/profile layout inside it, and the CLI-only relocation — so "under `target_dir`"
has a referent; and by **TOOL-11**, which fixes the deterministic name itself.) For a manifest-less invocation (a bare file list) an
implementation MAY use a temporary location, but MUST delete what it created before exiting. Reproducibility
is unaffected either way (TOOL-1/TOOL-6): where a file is written is not part of the build's output identity.

**Rejected.** Leaving the location implementation-defined — it is directly observable, so two toolchains
"conforming" would litter differently and neither could be scripted against. Deleting the package's
intermediates after a successful build — they are the material for `--emit`-style inspection and for the
`.s` a debugging session reads; `target_dir` is disposable by construction. Writing the test artifact
outside `target_dir` because it is "not the package's output" — it is exactly the package's output under
`test`, and hiding it is what made TOOL-7 ungateable.

### TOOL-11. The artifact name — the manifest binding is the base, the affix is per kind×container
*provenance: refines TOOL-3/TOOL-4 (no `Package.name`; `Target.output`) and TOOL-10 (a
deterministic name under `target_dir`) · spec: Tooling §2.2; Manifest appendix §3.1, §3.2*

**Why.** `Target.output` defaults to `""` and nothing said what an empty value names. That is not a
cosmetic gap: TOOL-10 requires every produced file to land under `target_dir` **with a deterministic
name**, so with `output` empty the normative claim has no referent, and two conforming toolchains would
write different file names for the same package — the FND-3 bar fails on the most visible thing a build
produces. It cannot be closed by adding a `name` field either: TOOL-3 removed `Package.name` on purpose
(the source-visible handle is the binding name), so the base has to come from something already in the
manifest. And the platform decoration is a second, independent question: a static library is
`libfoo.a` on ELF and `foo.lib` on PE, so "the name" is really "a base plus a per-target affix".

**The rule.** Two layers, so both the zero-ceremony case and the exact-control case are defined.

*The base* `‹base›` is the **manifest binding's name** — `app := Package(…)` gives `app`. Where there is
no binding — a manifest-less invocation (TOOL-14), or a root module carrying **zero** `Package` values
(the synthesized default package, TOOL-3) — the base is the **stem of the root source file**, so a bare
`package.al` builds `package` and anything else is stated as `output`. The base is thus host-independent and survives cloning the package into a differently-named
directory.

*The default name* is `‹base›` decorated per **`kind` × `container`**:

| `kind` | `elf` | `macho` | `pe` | `com` |   ← columns are the `machine` variant's container projection (TOOL-18)
|---|---|---|---|---|
| `executable` | `‹base›` | `‹base›` | `‹base›.exe` | `‹base›.com` |
| `static_lib` | `lib‹base›.a` | `lib‹base›.a` | `‹base›.lib` | — |
| `shared_lib` | `lib‹base›.so` | `lib‹base›.dylib` | `‹base›.dll` | — |
| `object` | `‹base›.o` | `‹base›.o` | `‹base›.obj` | — |
| `source` | *(no artifact)* | *(no artifact)* | *(no artifact)* | — |

A `Container.com` target is a real-mode flat image: only `Kind.executable` is defined for it, and any
other `kind` with that container is a Config diagnostic. `Os.none` is the other constrained case: with no
dynamic loader there is nothing for a `shared_lib` to be, so that combination is a Config diagnostic too
— the table would otherwise hand a freestanding target a `lib‹base›.so` no loader can use.

*An explicit `output`* is taken **verbatim** — the toolchain adds no prefix and no suffix, whatever the
kind or container. It is an artifact **file name**, not a path: `/`, `\`, NUL and an empty string are
Config diagnostics, rejected on every host so a manifest stays portable. `-o <path>` supplies a complete
path and name instead (no affix, `target_dir` unconsulted) and, naming one artifact, is a Config
diagnostic when the invocation would build more than one. With
`kind = source` there is no artifact, so an explicit `output` is a Config diagnostic there too.

**Rejected.** Reinstating a `Package.name` field — TOOL-3 settled that the binding name is the handle;
a second name for the same package is exactly the duplication the field catalog forbids (TOOL-4). The
**package directory name** as the base — it changes when the repository is cloned or renamed, so the
artifact name would depend on the checkout path rather than the package. The **manifest file's stem**
(`package.al` → `package`) — every package in the world would build `package`. Making `output`
mandatory for artifact-producing kinds — a trivial program would need a manifest, against TOOL-3/OP-1
(no ceremony for the common case). Always appending the platform affix, even to an explicit `output` —
then `output = "libfoo.a"` yields `liblibfoo.a.a`, and the written name is not the produced name (I3).

### TOOL-12. The entry point is a path, and `startup` says whether platform start-up files are linked
*provenance: refines TOOL-3/TOOL-4 (`Target.entry`) and TOOL-7 (whose entry the test artifact
carries); interacts with MOD-9 (link mode) and FN-9 (per-target ABI) · spec: Tooling §2.2, §4;
Manifest appendix §3.1, §3.2; Codegen §6; Stdlib §4.2*

**Why.** Three gaps met in one place. (a) The schema defaulted `entry` to `""` while the prose and TOOL-7
called `"_start"` the default — and neither said *what* the string is: a linker symbol, a declaration name,
or a file. (b) Nothing said whether the platform's **start-up objects** are linked. That is not a detail:
with glibc's `crt1.o` in the link, `_start` is **already defined** and `__libc_start_main` calls a symbol
named `main`, so a program defining its own `_start` collides, while without those objects nothing has
initialized TLS, `errno`, or `.init_array` and libc functions must not be called. The same package,
the same triple, two incompatible worlds — and the manifest could express neither. (c) The library promises
`args(allocator) → [str]` (Stdlib appendix §7), which is only implementable if **someone saved the process
entry state**; a raw `naked` entry drops it, so the promise had no mechanism behind it.

**The rule.**

*`entry` is an Alatyr path*, resolved by the ordinary module rules (MOD-1/MOD-4): `"_start"` is the
declaration `_start` in the **root module** (the manifest file), `"boot::start"` is `start` in
`‹source_dir›/boot.al`. The **linker symbol** is then whatever that declaration emits (Modules §6.1, or the
exact name of its `@export("…")`, Modules §6.3), and the toolchain **always passes `-e <symbol>` explicitly** — never
relying on the linker's own default. The default is `"_start"`. It is applicable to `kind = executable` and
`kind = object`; on `static_lib` / `shared_lib` / `source` an explicit `entry` is a Config diagnostic.

*A declaration named by `entry` that does not exist* is a **Codegen-stage diagnostic** raised **before** `ld`
is invoked, with the span on the manifest's `entry = …` (or, for a synthesized default, the start of the root
source file) — the link step never gets to report it, so the message and the exit status stay ours and
deterministic (§5).

*`Target.startup : Startup = Startup.raw`* selects whether platform start-up files are linked:

- **`raw`** (the default) — **no** start-up objects and no implicit libc; the process entry is the symbol
  `entry` names. This is the mode the library is written for (`exit` is a syscall wrapper on ELF, Stdlib
  §4.2), and on ELF it keeps the hermetic-static default (MOD-9). **PE and Mach-O have no usable syscall
  ABI** (ABI appendix §5 rejects `@abi(syscall)` on both), so there `raw` cannot mean "nothing linked":
  the toolchain adds exactly the platform library the entry contract needs — `kernel32` / `libSystem`,
  `dynamic`, enumerated in ABI appendix §3.3 — and the build carries the ordinary non-hermetic note. Left
  unstated, `exit`/`panic` had no implementation at all on a supported hosted triple under the default
  configuration. A `Lib(name = "c")` may still be linked for pure routines
  (`memcpy`, `strlen`), but libc is **uninitialized** — no TLS, no `errno`, no `.init_array` — which the
  toolchain surfaces as a **Config note**, not an error (freestanding code legitimately does this).
- **`libc`** — the platform's start-up objects **and** libc are linked. They own the process entry, so
  `Target.entry` is **not applicable** (an explicit value is a Config diagnostic); the program instead
  supplies the function the platform's start-up expects (`main` on ELF and Mach-O; on PE the CRT start-up
  selected by `subsystem`). libc's own link mode is `Lib(name = "c")`'s if stated, else the ordinary default
  `LinkMode.static` (MOD-9), so a `libc` build is still hermetic-static unless something is `dynamic`.
- The **freestanding** variants (`Machine.Bare`/`Com`) carry no `startup` field at all (TOOL-18): there
  are no start-up files there to link, so the mode is not a choice to be diagnosed.

*`Env.gnu` / `Env.musl` are ABI parameters, not a link instruction.* The triple's `env` fixes calling
conventions and libc-compatible layouts (FN-9); whether libc is linked at all is `startup`'s decision. A
hosted target with `startup = raw` is a normal, supported configuration — indeed the default one.

**Rejected.** Defaulting `entry` to `"main"` — `main` is meaningful only in the `libc` mode, where something
calls it; under the default `raw` mode it would be a name with no caller. `entry` as a **raw linker symbol**
— the manifest and the declaration would be tied by a string the compiler cannot check, duplicating what
`@export` already expresses, and the diagnostic would have no span. Looking the function up as
**`‹source_dir›/‹entry›.al`** — against MOD-1 (a file is a module by path) and MOD-6 (the symbol is mangled
by that path, so `src/_start.al` emits `_start__start`, not `_start`); it would need a special unmangled-
emission rule, and it is unnecessary — a file is part of the package by existing, so nothing has to be
"found". Deriving the mode from the triple (`Os.linux` ⇒ libc) — it would deny `linux` the right to be
libc-less, which is precisely the default. Relying on the linker's default entry symbol instead of passing
`-e` — an undefined `_start` then becomes a linker warning and an entry address of `0`, moving a normative
diagnostic outside the toolchain.

### TOOL-13. The project paths are fixed and package-local; the layout under `target_dir` is per target+profile
*provenance: refines TOOL-4 (the paths in the field catalog) and TOOL-10 (a deterministic name
under `target_dir`) · spec: Tooling §2.6, §4; Manifest appendix §3.7, §3.8*

**Why.** `source_dir` / `vendor_dir` / `target_dir` defaulted to "implementation-defined", which fails
the same way TOOL-10's silence failed: the paths are **directly observable**, they are what a script,
a `.gitignore` and a CI cache are written against, and two conforming toolchains would lay a package
out differently — so TOOL-10's promise ("deleting `target_dir` removes everything the toolchain made")
had no referent. Two more holes sat behind the defaults. Nothing constrained the **values**: an
absolute `source_dir` makes the manifest machine-specific although the manifest is part of the
reproducibility tuple (§6.2), and a `source_dir` that contains `vendor_dir` silently turns every
dependency file into a module of *this* package (`vendor::foo::src::bar`, MOD-1). And nothing fixed
the **layout inside** `target_dir`, so `--target all`, or two profiles, would overwrite one another's
artifacts whenever `output` matched.

**The rule.**

*Defaults:* `source_dir = "src"`, `target_dir = "target"`, each relative to the package root (the
directory holding the manifest). A single-file package needs neither to exist. (`vendor_dir` was part of
this rule and is withdrawn by TOOL-16 — vendoring is unspecified, so the field had no effect.)

*Values:* a `*_dir` in the manifest MUST be **relative** and MUST stay **inside the package tree**
(lexical normalization as in MOD-10; `..` escaping the root and any absolute path are Config
diagnostics). And `source_dir` MUST NOT contain the manifest file or `target_dir` — one containment
check instead of scanning exceptions, so `source_dir = "."` is an honest diagnostic rather than a
source of phantom modules.

*Relocation is the caller's, not the package's:* `--target-dir <path>` overrides that location per
invocation and MAY point anywhere, including outside the package. Like
`-o` (TOOL-10) they change **where** files are, never **what** is built, so they are not part of the
build input (§6.2). A shared build cache across packages is therefore a property of the workstation
that asks for it, not of the package that would impose it on every consumer.

*Layout:* when `Package.targets` has one element, artifacts and intermediates go to
`‹target_dir›/‹profile›/`; with two or more — where each `Target.name` is already mandatory and distinct
(TOOL-4) — to `‹target_dir›/‹target-name›/‹profile›/`. The switch is on the **manifest's** target count,
not on what an invocation selected, so a path never shifts with `--target`. The `.s`/`.o` of a build sit
beside the artifact they produce, named by the module path (`geometry__vec.o`, MOD-6), so they cannot
collide with a `kind = object` artifact. So `--target all` and a profile switch never collide, and the path is derivable from the
invocation without consulting the implementation.

**Rejected.** Keeping the paths implementation-defined — the FND-3 bar, on the most scriptable
surface the toolchain has. A manifest-settable `target_dir = "../shared"` — two packages would then
overwrite each other's artifacts (there is no package-name namespace under `target_dir`, and TOOL-3
removed `Package.name`), and a package would dictate where the machine's build output lands; Cargo's
`CARGO_TARGET_DIR`/Go's `GOBIN` live in the caller's configuration for exactly this reason. A flat
`target_dir` with no per-target/profile nesting — `--target all` silently overwrites. Nesting by a
canonical triple string instead of `Target.name` — v1 has no normative string spelling of a triple
(targets are structured values, TOOL-3), and inventing one here would create a second name for a
target that `--target` does not use.

### TOOL-14. Manifest discovery searches upward; a manifest-less invocation is fully specified
*provenance: refines TOOL-3 (the manifest is optional, `package.al` by default) and TOOL-10 (the
manifest-less run cleans up) · spec: Tooling §4; Manifest appendix §1*

**Why.** Two loose ends left by "the manifest is optional". First, *where* `package.al` is looked for:
the spec said "at the package root" without saying how the root is found, so `alatyr build` run from
`src/gfx/` is a coin flip — one toolchain searches upward and builds the package, another reports no
manifest. Second, TOOL-10 already grants a **manifest-less invocation** ("a bare file list") the right
to use a temporary directory, yet nothing said what such an invocation *is*: which file is the root
module, what the artifact is called, which kind and startup mode apply, whether submodules resolve.
A mode with normative cleanup rules but no normative semantics is the worst of both.

**The rule.**

*Discovery.* `package.al` is searched for starting at the **current working directory** and walking
**upward**, parent by parent, to the filesystem root; the **first** one found is the manifest, and the
directory holding it is the **package root** (which is what `source_dir`/`target_dir` are relative to,
TOOL-13). `--manifest <path>` names the file explicitly and its directory becomes the
package root. So a build behaves the same from any subdirectory of a package — the property a developer
already assumes from `cargo`/`go`.

*The manifest-less invocation.* When a command is given a **bare file list**
(`alatyr build a.al b.al …`), no discovery happens and no manifest is read — the file list *is* the
input. Configuration synthesizes the default `Package` (TOOL-3) with: the **first listed file** as the
root module and its directory as the **package root** — so `source_dir` is that directory (submodules
still resolve by path, MOD-1) and `target_dir` resolves beneath it — with that root file **excluded**
from module-path scanning, the same exception a manifest file gets (TOOL-13), so it is not also a module
by its own stem; `kind = executable`, `startup = raw`, `entry = "_start"` (TOOL-12); the artifact base =
the **stem of that first file** (TOOL-11); the host triple; the `debug` profile unless
`--profile`/`--release` says otherwise. Dependencies cannot be declared in this mode, so no lockfile is
read or written. Artifacts land under that `target_dir`, or where `-o` says; an implementation MAY use a
temporary location instead, which it MUST clean up (TOOL-10).

*Conflicts and emptiness.* A bare file list together with `--manifest` is a **Config diagnostic** (two
different answers to "what is being built"). An invocation with neither a discoverable manifest nor a
file list is a **Config diagnostic** naming that nothing was found — never a silent no-op. Both are
**invocation-level**: no source exists to point at, so they are the one exception to the mandatory-span
rule, naming the offending argument or the working directory instead (spec: Tooling §5).

**Rejected.** Looking only in the current directory (the `zig build` shape) — it makes the command's
result depend on the shell's location inside one package, which is exactly the accident a package root
exists to remove. Stopping the upward walk at a VCS boundary (`.git`) — a package need not be a
repository, and a repository need not be a package (TOOL-9 defers workspaces), so the boundary would be
a second, invisible rule. Dropping the manifest-less mode instead of specifying it — a single-file
program is the language's own smoke test, and TOOL-10 already made normative claims about it. Treating
extra listed files as additional roots — one module tree (MOD-1); the first file is the root and the
rest are reached through it, or they are not part of the build.

### TOOL-15. The manifest handle is an ordinary root-module declaration — package-wide, never `pub`, and it collides with a same-named module
*provenance: refines TOOL-3/TOOL-4 (the binding name is the handle) and TOOL-11 (the binding
names the artifact); interacts with MOD-2/MOD-8/MOD-12 · spec: Tooling §2.7; Manifest appendix
§1; Modules §5*

**Why.** "The binding name is the handle" left three consequences of *being an ordinary declaration*
unstated, and each fails observably. (a) **Its scope.** Tooling §2.7 lists the handle beside `target.*`
and `build.*`, which configuration publishes unconditionally to every module of every package in the
build — so the handle reads as if it were equally ambient, when in fact it is a root-module declaration
that reaches the rest of the package only by down-tree privacy (MOD-2) and reaches a dependency never.
(b) **`pub` on it.** Nothing forbade `pub app := Package(…)`, which would export a value whose **type
lives in the configuration prelude "visible only to the manifest"** (Manifest appendix §3): a consumer
could read its fields but could not name the type, declare one, or accept it as a parameter — a
half-exported entity that exists nowhere else in the language. (c) **A same-named child module.**
`mylib := Package(…)` in the manifest file with `‹source_dir›/mylib.al` present — the natural spelling
for a library — is a duplicate name in the root scope, yet Modules §5 lists only "declarations,
imports, re-exports" as the sources of names, and the file/directory pairing MOD-12 describes is the
**same-stem** one (`geometry.al` + `geometry/`), not the root's (the manifest file + `‹source_dir›/`).
The clash was therefore ill-formed by inference and undiagnosable by the letter.

**The rule.**

*Scope.* The handle is an **ordinary declaration of the root module**: private by default, hence visible
to the root module and — by down-tree privacy (MOD-2) — to **every module of the package**, and to
nothing outside it. That is exactly what distinguishes it from `target.*` / `build.*`.

*No `pub`.* `pub` on the manifest binding is a **Config diagnostic**. `Package` is one of the
**manifest-only** configuration structures (TOOL-17), not part of the language's ordinary namespace, so the
value cannot be part of a package's public API. A library that wants to publish metadata re-exports the fields it means — `pub VERSION :=
mylib.version`, an ordinary `str`. (The handle emits no symbol under any spelling, Manifest appendix §1,
so none of this changes an artifact's contents; MOD-13.)

*Child modules name a scope too.* A module's scope is named by its own declarations, imports and
re-exports **and by its child modules by path** (MOD-1/MOD-12); for the **anonymous package root** the
two halves are the **manifest file** and **`‹source_dir›/`**. So `mylib := Package(…)` beside
`‹source_dir›/mylib.al` is the ordinary MOD-8 duplicate-name error with no implicit winner. Because one
side of such a clash is **never written as a declaration**, the diagnostic MUST name **both** — the
declaration in the manifest file (here: the handle) and the file whose path became the module.

**Rejected.** Giving the handle a namespace of its own (a keyword such as `package.*`) so that it could
coexist with a same-named module — it would stop being an ordinary binding, which is precisely what
TOOL-3 chose, and it buys a name reuse worth little (a consumer reaches the package through its own
alias, MOD-10, so an internal module named after the package reads `hal::mylib::item`). Letting `pub` on
the handle mean "export the fields, not the type" — a structurally-accessible value of an unnameable
type; re-exporting the fields says the same thing in one line. Resolving the collision by shadowing
(the module wins, or the declaration wins) — one scope has no implicit winner (MOD-8), and the loser
would be silently unreachable. Treating the collision as a **Config** diagnostic — the manifest is
valid; the clash is a name-resolution fact of the module tree, so it belongs to the Semantic stage.

### TOOL-16. Vendoring is post-v1; v1 has no `vendor_dir`
*provenance: refines TOOL-4 (no orphan fields) and TOOL-13 (which had given the field a default,
validation and a CLI flag); parallel to TOOL-9's workspace deferral · spec: Tooling §2.6;
Manifest appendix §3.7, §3.8*

**Why.** `vendor_dir` sat in the field catalog with a default, a containment rule, a `--vendor-dir`
relocation flag and a conformance duty — and **no rule anywhere saying what it does**. Nothing states
that a dependency's source is materialized there, how a vendored tree is checked against the lockfile,
whether a vendored copy is preferred over fetching, or how the vendored form interacts with MOD-10's
source identity (a path dependency is keyed by its lexically-normalized absolute path — vendoring
changes that path). `DepSource` has no vendored variant, and §2.4 never consults the field. That is
precisely TOOL-4's failure mode ("a field that is declared without an implemented effect"), made worse
by TOOL-13: a **conformance requirement** to honour a flag pointing at a directory with no defined
role. And a half-specified `--vendor-dir` was actively wrong in a second way: `vendor_dir` would hold
build **inputs** (dependency sources), so relocating it changes *what* is built, contradicting the
"location only, never build input" clause TOOL-13 wrote for it.

**The rule.** v1 has **no `vendor_dir` field and no `--vendor-dir` flag**. Where a git dependency is
checked out is an **implementation detail** until vendoring is specified — the lockfile already fixes
*what* is built (MOD-7/MOD-10), and TOOL-10's cleanliness promise covers what the build *produces*, not
where a fetched source is cached. `source_dir`'s containment check accordingly names the manifest file
and `target_dir` only.

Vendoring is **additive** (I10/FND-6) and reinstated together with the semantics it needs: a
`DepSource`/manifest surface that selects a vendored tree, the lockfile check that validates it, its
place in MOD-10's identity rule, and its status in the reproducibility tuple — as an **input**, not a
location.

**Rejected.** Keeping the field as documented-but-inert — the orphan TOOL-4 forbids, and a reader
reasonably concludes vendoring works. Specifying vendoring here, inside a paths decision — it is a
dependency-resolution feature (MOD-7's registry/semver neighbourhood), not a directory-naming one, and
deciding it in passing is how underspecified fields get born. Keeping `--vendor-dir` alone as a cache
control — the flag would name a directory the specification does not otherwise mention, and it would
inherit the same input-versus-location confusion.

### TOOL-17. The configuration prelude has two halves; `target.*` is a closed set; `check` is defined
*provenance: refines TOOL-3/TOOL-4 (the schema and its visibility) and MOD-13 (`target.*`
widened to the build fields); closes the `check` command left unspecified by TOOL-1 · spec:
Tooling §2.7, §4; Manifest appendix §3*

**Why.** Three loose ends met once `target.kind` existed. (a) The manifest schema was declared to live in
"a configuration prelude **visible only to the manifest**", yet ordinary source has always been told to
write `when target.arch == Arch.x86_64`, and MOD-13 added `when target.kind == Kind.static_lib`. Both
cannot hold: either the enums are nameable in source or those gates are ill-formed. The distinction is
also load-bearing in the other direction — TOOL-15 forbids `pub` on the manifest handle *because* its type
is manifest-only — so leaving it implicit put two rules on an undefined foundation. (b) MOD-13 said
`target.*` exposes "every other per-target field", then listed six of the ten: whether `target.name`,
`target.entry`, `target.output` and `target.auto_cfi` exist was a coin flip, and a comptime surface must
not be. (c) `check` appeared in the command list and nowhere else, yet MOD-13's remedy for `kind = source`
("use `check`") and the new Codegen-stage `entry` diagnostic both depend on what it does.

**The rule.**

*Two halves.* The **manifest-only structures** (`Package`, `Target`, `Dependency`, `DepSource`, `GitRef`,
`Lib`, `Profile`, `FlagDecl`, `FlagSet`, `FeatureAlias`) remain visible only to the manifest — hence
TOOL-15's `pub` prohibition. The **configuration enums** (`Arch`, `Os`, `Env`, `Container`, `Kind`,
`Startup`, `CodeSize`, `ArmMode`, `Subsystem`, `Endian`, `Limit`) are **also ordinary prelude names**,
because they are the vocabulary of the `target.*` comparisons; they are plain comptime enums with no
special status.

*A closed `target.*`.* The published set — all of it **projections of the `machine` variant** since
TOOL-18 — is `arch`/`os`/`env`/`container`, the derived widths and
`endian`, `features`, and the build fields `kind`/`startup`/`subsystem`/`code_size`/`arm_mode`/
`vector_length`. **Not** published: `name` and `output` (they name the build and its file, not the
program), `entry` (a path into the program's own tree — gating on it would make code depend on which
declaration is the entry), and `auto_cfi` (a non-semantic codegen knob, like a profile's flags).

*`check`.* Configuration + parse + semantic analysis for the selected target and profile, reporting those
stages' diagnostics; no lowering, emission, assembly or linking, and **no artifact** even with `-o`. So it
is the command for a `kind = source` target and the fast well-formedness gate elsewhere — and, explicitly,
a **Codegen**-stage diagnostic does not surface under it.

**Rejected.** Making the enums manifest-only and giving source a parallel vocabulary (string comparisons,
or a second `arch`-like namespace) — a second name for one closed set, which OP-1 forbids, and
`target.arch == Arch.x86_64` has been the spec's own spelling throughout. Publishing every `Target` field
for symmetry — `target.entry` invites code that branches on which declaration is the entry, coupling
program logic to a build knob; `target.output` invites embedding the artifact's file name in the artifact.
Leaving `check` to "obviously means what it does in other toolchains" — the FND-3 bar, and the one
question that actually matters (does a Codegen-stage diagnostic appear?) has no obvious answer.

### TOOL-18. The platform is a `Machine` variant, not a bag of sometimes-applicable `Target` fields
*provenance: supersedes the flat-triple shape of TOOL-4 (`arch`/`os`/`env`/`container` as
fields) and the per-field host defaulting of TOOL-3; refines FND-8 (the machine model), TOOL-11
(the affix table), TOOL-12 (`startup`), TOOL-17 (`target.*`) and CG-14 (a non-ISA backend) ·
spec: Manifest appendix §3.1–§3.2, §4, §7; Tooling §2.2, §2.7; ABI appendix §2; per-arch
appendix §1*

**Why.** `Target` was a flat struct in which **half the fields are conditional**: `subsystem` means
something only on PE, `code_size` only on x86, `arm_mode` only on `aarch32`, `startup` only where an OS
provides start-up files, `endian` only as a restatement of the arch, and `env` only for the environments a
given OS has. The specification paid for that shape three times over. It needed a **rule per hole** —
"explicitly provided but not applicable is a Config diagnostic", "an `endian` contradicting the arch is
rejected", "`Startup.libc` on `Os.none` is rejected", "`Container.com` requires `CodeSize.b16`" — and every
new platform adds another. It made the **supported set** a subset of a cartesian product, so `pe` with
`gnu`, or `msvc` on `aarch64`, were *writable* and had to be diagnosed rather than being unsayable. And it
could not accommodate a backend that has no container at all: an additive non-ISA backend (WASM, CG-4/CG-14)
forced scope caveats onto rules written as unconditional, because there was no place in the shape for "a
platform that is not a triple". Per-field host defaulting made it worse in a subtler way:
`Target(arch = Arch.aarch64)` meant "aarch64, else whatever this machine is", so a manifest's meaning
depended on where it was read.

**The rule.** A `Target` carries a **`machine` field whose type is the `Machine` enum** — one variant per
platform shape, each holding exactly the fields that platform has:

```alatyr
Machine := enum {
  Linux  ( arch : Arch, env : Env = ‹arch-default›, startup : Startup = Startup.raw )   # ELF, hosted
  Freebsd( arch : Arch, startup : Startup = Startup.raw )                               # ELF, hosted
  Windows( arch : Arch, subsystem : Subsystem = Subsystem.console, startup : Startup = Startup.raw )   # PE
  Macos  ( arch : Arch, startup : Startup = Startup.raw )                                # Mach-O, hosted
  Bare   ( arch : Arch, env : Env = Env.none )                                           # ELF, freestanding
  Com    ( )                                                                            # i386 real-mode image
}
```

Consequences, each replacing a rule rather than adding one:

- **No `os` / `container` / `endian` / `subsystem` / `env` fields on `Target`.** The variant *is* the
  platform; endianness follows the arch (FND-8). A `subsystem` off PE or a `startup` on a freestanding
  target is not diagnosed — it cannot be written.
- **`target.*` are projections** of the variant (`target.os`, `target.container`, `target.env`,
  `target.startup`, `target.endian`), so every existing gate keeps its spelling and `Os`/`Container`/`Endian`
  stay prelude enums for those comparisons. The variant itself is **not** published as a value: source
  compares projections rather than matching platforms, so adding a platform does not oblige portable code
  to handle a new arm.
- **The supported set is `(variant, arch)` pairs** (per-arch appendix §1), and the ABI-selection exceptions
  read off the variant (`Machine.Windows` → `win64`, `Machine.Com()` → `naked`).
- **Two fields stay on `Target`** because they are **arch**-specific codegen knobs, not platform facts:
  `code_size` (x86) and `arm_mode` (`aarch32`). Their applicability rule is the one survivor of the old
  class.
- **Defaulting is by variant**: `Target()` is the build host's whole variant; a partial deviation
  ("aarch64, else host") is deliberately no longer expressible, so an unconfigured build is host-dependent
  in exactly one place instead of field by field.
- **A non-ISA backend arrives as a new variant** with no container projection — additively, without
  rewriting the rules that assume one (CG-14).

**Rejected.** A full enum over platforms *including* the build fields (`Target := enum { X86_64Linux(…),
… }`) — `name`/`kind`/`entry`/`output`/`features` are common to every target, so each variant would restate
them; in a data subset without spread that is copy-paste in every manifest. Keeping the flat struct and
living with the applicability rules — the shape was the cause and the rules were the symptom, and each new
platform or backend adds another symptom. Variants per **arch** rather than per platform — the conditional
fields are mostly platform-conditional (`subsystem`, `startup`, `env`), so arch variants would leave them
conditional anyway while multiplying the variant count by six. Publishing `target.machine` as a matchable
value — portable code would then owe every platform an arm, and a new platform would break existing
`match`es; projections keep the old gates working and the new platform invisible to code that does not care.
A string triple (`"x86_64-linux-gnu"`) — rejected already by TOOL-3, and a variant is strictly more precise.

### TOOL-19. Android is its own `Machine` variant, carrying the API level
*provenance: extends TOOL-18 (the platform is a variant) and TOOL-12 (`startup`); refines FN-9's
per-target ABI · spec: Manifest appendix §3.1; per-arch appendix §1; ABI appendix §2(c), §4.4;
Tooling §2.2*

**Why.** Android is a Linux **kernel** with a non-glibc userland, and every attempt to express it as a
flavour of `Machine.Linux` gets something wrong. Its libc is **bionic** — a different symbol surface, a
different dynamic linker — so `target.env` reporting `gnu` or `musl` would be a lie a program could act on.
Its available libc surface depends on the **API level**, which nothing in the manifest could state, yet a
packaging step must know it (`minSdkVersion`) and the toolchain must pick a sysroot by it. Its artifacts
must be position-independent (PIE/PIC) because the platform loader requires it, not because a knob asked.
And on `aarch32` the NDK convention is the **soft-float ABI with VFP/NEON instructions available**, which
is neither of `aarch32`'s existing `eabi`/`eabihf` readings taken literally — treating `bionic` as
hard-float would break every call into a platform library. Since TOOL-18 made the platform a variant, all
of this has an obvious home; before it, it had none.

**The rule.** `Machine.Android( arch : Arch, api : u32 = ‹min-supported›, startup : Startup = Startup.raw )`
— ELF, bionic. Its projections: `target.os` = `android` (a new `Os` variant), `target.env` = **`bionic`**
(a new `Env` variant, projection-only), `target.container` = `elf`. Supported arches are the four NDK ones
(`aarch64`, `aarch32`, `x86_64`, `i386`; per-arch appendix §1). `api` selects the platform sysroot, is part
of the build input, and is reported in the build plan (TOOL-20); the minimum — and the default — is **21**,
and a lower value is a Config diagnostic. Artifacts are position-independent by the platform's rule, not by
a field. `startup = raw` reaches the OS through **Linux** syscalls (the kernel ABI is Linux's, ABI appendix
§5); `startup = libc` links bionic, and the start-up-called function is `main`. On `aarch32` the ABI is
**soft-float** with `vfp`/`neon` available as ISA features (ABI appendix §2(c)), and on `aarch64` the
platform register **`x18` is reserved** (bionic's shadow call stack, §4.4).

**Rejected.** `Machine.Linux(env = Env.bionic)` — it would put an OS-level difference (loader, sysroot,
API level, reserved register) inside a field that means "which libc", and `api` would have nowhere to live
but `Target`, where it is inapplicable to every other platform — the shape TOOL-18 exists to remove.
Deriving `api` from a feature or a limit — it is neither an ISA capability nor a language restriction, and
its effect (which sysroot, which symbols) is a link-time fact. Making PIE/PIC a `Target` field for symmetry
with other platforms — nothing else on Android is possible, so the field would have exactly one legal
value. Treating Android's `aarch32` as `eabihf` because VFP exists — the instruction set and the calling
convention are independent, and the platform's is soft-float.

### TOOL-20. The build plan is the toolchain's output contract; packaging is not a `Kind`
*provenance: refines TOOL-10 (what a build produces) and TOOL-11 (artifact naming); parallel to
TOOL-9's and TOOL-16's deferrals · spec: Tooling §4, §4.2, §7*

**Why.** Two questions arrived together: can a `Kind` be a distribution package (`.deb`, an APK), and how
does anything downstream learn what a build produced? They have one answer, and it is not a new `Kind`.
A `Kind` is what the **link step** yields — one file whose contents follow from our emission rules
(MOD-13). A package is a different domain: it bundles **several** artifacts plus non-code (unit files,
icons, an `AndroidManifest.xml`), carries metadata the compiler has no opinion about (`Depends`,
maintainer scripts, permissions, `minSdkVersion`), needs foreign toolchains (`dpkg-deb`, `aapt2`,
`apksigner`, a JDK), and — for an APK — a **signing key**, i.e. a secret that cannot live in a manifest
without destroying both portability and hermeticity. Its archive format also embeds timestamps and file
order, which a bit-identical guarantee would have to legislate. And `.deb` is conventionally built from an
**installed tree**, a notion v1 does not have at all: everything lives under `target_dir` and nothing is
"installed". Meanwhile the actual gap is smaller and universal: today a packaging step, an installer, or a
CI attestation must scrape paths and re-derive the affix rules (TOOL-11) to learn what came out of a build.

**The rule.** The toolchain emits a **build plan**: `alatyr build --plan` writes it beside the artifacts,
`alatyr plan` writes only it (configuration and resolution, no codegen). It is **output only** — nothing in
a build reads it, and its presence changes no artifact, so it stays outside the build input (§6.2). Its
format is normative (a v1 tooling surface, like the lockfile): line records `meta` / `artifact` / `dylib`,
tab-separated, deterministically ordered, byte-identical for one build input, with a `plan-version` so a
reader can refuse what it does not understand. Each `artifact` record carries a **derived install path**
(`bin/…` for an executable, `lib/…` for a library, empty for `object`/`source`) — the one piece packaging
always needs and the only part of "installation" v1 commits to; there is no install section in the
manifest, and a tool wanting another layout maps it.

**Packaging stays outside this specification.** It is a separate tool — plausibly one per publication
target — or a future `publish`-style command, and either way the build plan is its **input**. That keeps
foreign toolchains, signing keys and archive-format quirks out of the conformance surface while making the
handoff precise instead of conventional.

**Rejected.** `Kind.deb` / `Kind.apk` — it would put foreign toolchains and a signing secret inside the
manifest's field catalog (TOOL-4) and inside the reproducibility tuple; a package is also not one link
step's output. Manifest-declared packaging **steps** (a Gradle-shaped `tasks` section) — the manifest is
pure data with no executable steps (TOOL-3); admitting steps re-opens exactly that. Emitting the plan as
JSON or TOML — v1 has no such format anywhere, and the lockfile already set the house pattern for a
normative machine-readable file. Emitting it as an Alatyr data value — elegant, but it obliges every
external packager to parse Alatyr. Making the plan an **input** a later stage consumes — then a build's
result would depend on a file a previous build wrote, which is precisely the incremental-build trap TOOL-6
avoids by keying the cache on content. A full **install** section in v1 — installation has its own
questions (prefixes, permissions, symlinks, man pages, `DESTDIR`) and none of them is decided; the derived
install path answers the packaging case without pretending to answer those.
