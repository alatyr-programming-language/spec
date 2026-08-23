# Decisions — Modules, visibility, packages, paths, imports/re-export

**Why the language is the way it is** for the module system. Each entry records the
*rationale* and the *rejected alternatives*; the normative *what* lives in the spec
(linked per entry), the *how* in the compiler. IDs are theme-prefixed (`MOD-N`) and
**append-only** — assigned in articulation order, never renumbered (a later sub-theme
appends higher numbers, it does not shift these). The `provenance:` line records what an
entry refines and where it lands in `spec/`.

---

## Modules, visibility, packages

### MOD-1. One module tree — files, directories, and inline `mod`
*(Refined by TOOL-13/TOOL-14: file paths are read relative to the package's `source_dir` — a fixed
default, not implementation-defined — and its own name is not part of a module path.)*
*spec: Modules §1*

**Why.** One language (the single-tree consequence of FND-9) wants one namespace structure,
not several. So files become modules by their on-disk path (`geometry/vec.al` →
`geometry::vec`), directories form the hierarchy, and an inline `Name := mod { … }`
extends *the same* tree — `mod{…}` being an ordinary namespace-*value* expression (the
unification of FN-1), not a second construct. The package root is **anonymous**: by default
the file `package.al`, which carries **both** the single `Package` manifest value
(configuration, TOOL-3) **and** the root module's ordinary code — a single-file package is
normal. The result is one tree reached one way, with no special module-declaration syntax
distinct from ordinary value binding.

### MOD-2. Private by default; privacy flows down the tree
*spec: Modules §3*

**Why.** Encapsulation is the default: an element is **private** unless `pub` exports it
(a bare keyword, TYP-7), and reaching the root requires a **continuous chain of `pub`** (one
break stops it). The privacy *direction* is the load-bearing choice: a non-`pub` element
of a module `M` is visible to `M` and **all of its descendants** (nested modules,
file-based and inline) but not to ancestors, siblings, or strangers — a submodule may use
its parent's internal helpers without those helpers being exposed. `pub` opens **upward**
one level (to the parent), further by chain. Subtree-down (Rust-style) was chosen because
the language has only `pub`: with no `pub(super)` / `pub(crate)`, strict module-locality
would force `pub`-ing helpers merely so a submodule could see them — over-exposure. The
linker emission of a symbol is a **separate axis** (`@export`, MOD-5), not governed by this
visibility.

**Rejected.** Strict module-locality (no down-tree access) — forces over-exposing helpers
for a submodule's sake, given only one `pub` keyword.

### MOD-3. `::` navigates namespaces, `.` accesses values (and UFCS)
*spec: Modules §2*

**Why.** Two operators carve the two worlds cleanly. **`::`** navigates a **namespace**
(modules / types / associated items) at compile time — `geometry::vec::length` — and also
navigates a **binding that holds a module value** (`v := geometry::vec` → `v::length`).
**`.`** accesses a member of a *value* (fields, tuple index) and carries **UFCS** at
runtime — `s.field`, `value.method(x)`. UFCS accepts a qualified name
(`value.ns::method(x)` ≡ `ns::method(value, x)`), so a module's free functions are
UFCS-callable without importing bare names — `.` stays the UFCS dot, `::` stays the
namespace operator. This split removes the `geometry::length(x)` (qualified call) vs
`value.length(x)` (UFCS) ambiguity by **syntax**, not by case (SYN-1). A `.`-postfix
**followed by a call** is *always* UFCS even when the name also matches a field; a
function-valued field is called as `(a.f)(args)`. (Receiver auto-ref / auto-deref — the
call-site sugar that supplies the receiver's address or pointee — are recorded with
Functions/Pointers, MEM-2/OP-5.)

**Rejected.** Member-first with UFCS-fallback (D/Kotlin): it reads ambiguously and admits
the silent field-shadows-function hazard (I3). `$` as a path separator in symbols — not
C-compatible (see MOD-5).

### MOD-4. Imports, aliases, and re-export are ordinary declarations — no `use`
*spec: Modules §4*

**Why.** Because `::`-paths are *already values* (a module / type / function / constant is
a comptime entity, FN-1/MOD-1), a dedicated `use` construct adds nothing — it was the last
special construct with its own grammar. So an **import / alias** is an ordinary
declaration `vec := math::vec` (module alias; access `vec::length`) or
`len := math::vec::length` (a single name): a comptime alias, **erased, creating no
symbol**; the full qualified path is always available without it (an import is only
brevity, the final universalization of FN-1). A **re-export** is `pub name := path` (it was
`pub use`); renaming falls out of the `=` form itself, so there is **no `as` keyword**
(redundant given FN-1's declaration token). There is **no glob** (`X::*` / import-all) —
invisible imports stay out — but a **listed member projection** `(a, b, …) := M` is
allowed: it binds each *listed* name to `M::<name>`, pure sugar for N single declarations,
disambiguated from value-destructuring by RHS kind (a module path → projection by name; a
multi-output call → positional). This listed projection is an ergonomic name-saver admitted
**despite** OP-1's "merely saves names" test — weighed and allowed by analogy to compound
assignment (OP-2): the names stay explicit, so the explicitness motive holds while the
per-name `M::` repetition goes.

**Rejected.** A `use` primitive or a thin `use`-sugar; a `use(...)` function (a `::`-path
is already a value — a wrapper is superfluous). Consistent with "do not introduce what is
expressible by existing means" (OP-1).

### MOD-5. Re-export aliases a name, never duplicates a symbol; the export gate
*spec: Modules §4.3, §4.4*

*(Refined by MOD-13: the `pub`-chain gate decides **eligibility**; whether that export lands in a given
artifact is the artifact-`kind` rule of Modules §6.4 — an executable does not export the API for merely
being built.)*

**Why.** A re-export (`pub name := path`) is an **alias at the level of names, not a new
binary symbol**: the symbol is defined once at the declaration site, and a re-export merely
makes that *same* symbol reachable under another Alatyr-namespace path — no duplicates. It
controls **reachability**: a linker export happens iff a continuous `pub`-chain (including
`pub`-re-exports) reaches the root (MOD-2), while the symbol's *name* is the definition's
path (MOD-6) or `@export`. You may re-export **only `pub` names**: a descendant's down-tree
access to an ancestor's private (MOD-2) is for local use; re-exporting a private would
re-publish an ancestor's private helper upward and bypass the owner's privacy, so it is an
error. A facade module is a set of `pub x := …` lines, and a C-facing clean name is set by
`@export("clean")` on the declaration (a re-export does not rename the symbol).

**Rejected.** Re-export as a new binary symbol / duplicate (defeats single-definition).
Re-exporting a private name (bypasses the owner's privacy).

### MOD-6. Symbol mangling and FFI — the declaration spelling, C-compatible
*spec: Modules §6, §7*

**Why.** The export symbol is the **spelling from the declaration** (not normalized — `_`
and case as written, SYN-1), with path separators `::` → **`__`**, so
`geometry::vec::length` → `geometry__vec__length`: a **valid C identifier**, linkable and
callable from C directly (the root takes no prefix — `@export _start` → `_start`).
`@export("exact")` pins an exact C symbol. Imports are symmetric and scheme-flexible — an
external symbol is taken by exact name under any foreign scheme (C-flat, C++-mangled, Rust,
`__`-separated) via `@extern` (name by declaration) or `@extern("exact_external_name")`;
the symmetry is `@export`/`@export("...")` outward, `@extern`/`@extern("...")` inward. A
literal `__` inside a declared name is a lint, and demangling consults the module tree (not
a blind split) since for C `__` is part of a valid name. Symbol naming is **orthogonal to the
calling convention**: the `Abi` value (ABI appendix §1) carries **no `name_mangling` field** —
it describes register/stack passing only; platform symbol decorations (a macOS leading `_`, a
Windows `@N` stdcall suffix) are an object-format/target detail applied uniformly at emission,
not a per-`Abi` knob.

**Rejected.** `$` as the path separator — not C-compatible, so C and others could not link
by name. A `name_mangling` field on the `Abi` value — conflates symbol spelling with the
calling convention; mangling is a module/object-format concern (here), not an ABI knob.

### MOD-7. Packages and dependencies — selected by source, not by semver
*spec: Modules §8*

**Why.** A package is the unit of build and delivery (its manifest is the configuration).
Dependencies are selected **by source** in the manifest: a path (on-disk, used as-is) or
git by a `GitRef` (`Commit` / `Tag` / `Branch`, with `Tag`/`Branch` resolved-and-pinned
through the hashed lockfile). A dependency's elements live under its **local namespace name**
(`<name>::<module>::*`, without flat clutter — one naming field, MOD-14), reached qualified or via a
binding `m := <name>::<module>` (MOD-4). This **exact-pin, by-source** model is what the project's
**reproducibility-first ethos** wants (byte-for-byte reproducible builds gated by the
lockfile, TOOL-1): a build resolves to *exactly* the sources named, with no solver in the
loop to reinterpret a constraint into a different graph across machines or over time.

**A central registry + semver version-constraint resolution is deferred to post-v1**
(confirmed). It is **ecosystem-maturity** work — a version solver and a registry carry real
complexity and **supply-chain** surface (a resolver picking versions, a hosted index to
trust), and both are premature before an ecosystem exists to populate them. Exact-pin now
serves reproducibility directly; a solver + registry is **purely additive** (FND-6 — addable
later without breaking a v1 manifest, since a `version` field and a registry source are new
optional inputs, not changes to path/git resolution). So v1 resolves dependencies by
explicit **path / git-ref + a reproducible lockfile only**.

**Rejected.** A per-dependency semver `version` constraint in v1 — a path has nothing to
select and a git dep selects by ref; a version solver adds complexity and supply-chain risk
for no v1 benefit (there is no registry to resolve against), and it is additive (FND-6), not
v1. A central package registry in v1 — ecosystem infrastructure, premature before an
ecosystem, and in tension with the exact-pin reproducibility guarantee unless itself pinned
(which the lockfile-over-source model already achieves without it).

### MOD-8. No duplicate names in a module's scope
*spec: Modules §5*

**Why.** Within a module's scope, any two of declarations, imports, and re-exports that
yield the same resulting name are an **error**, with no implicit winner (an extension of
FN-2). The same rule covers a re-export whose name collides with an existing name in the
target module. Refusing to silently pick a winner keeps name resolution unambiguous and
predictable. **Child modules count too** (refined by TOOL-15): a module's scope is named by its own
declarations/imports/re-exports *and* by its child modules by path, so a declaration colliding with a
same-named file-module is this same error — including, at the anonymous root, the manifest handle against
a module of `source_dir`. Where one side is a file rather than a declaration, the diagnostic names both.

### MOD-9. Foreign-library link mode is first-class and per-library; hermetic-static is the default
*provenance: refines FN-8 (manifest-driven linking) and MOD-7 (reproducibility-first packages) · 
spec: Modules §7.5, §8; manifest appendix §3.5, §3.7*

**Why.** An external (C/system) library carries a **link mode** — `static` (absorb the
`.a` archive into the binary) or `dynamic` (reference the `.so`, resolved by the OS loader
at runtime) — and this is what decides whether a build is **hermetic**. FN-8 already routes
*linking* through the manifest ("static/dynamic libraries, paths, flags are driven by the
manifest"); this entry makes the static-vs-dynamic choice a **first-class, per-library**
field rather than an opaque `linker_flags` string, so **hermeticity is a manifest-
determinable, compiler-surfaced property** (I3 — nothing hidden). The schema replaces the
flat `libs : [str]` with a structured list: `LinkMode := enum { static, dynamic }` and
`Lib := struct { name : str, link : LinkMode = LinkMode.static }`. A bare system-lib name
thus links **statically by default** (`Lib(name = "m")`); `Lib(name = "ssl", link =
LinkMode.dynamic)` opts a single library into dynamic — precise, per-library, all-or-
nothing not forced. This stays consistent with the manifest being a pure comptime `Package`
value (MOD-1/TOOL-3): `Lib`/`LinkMode` are ordinary config types in the data subset.

**Binary link-mode rule.** The produced binary is **fully static** — self-contained, no
runtime interpreter, no `.so` dependency — **unless at least one linked library (directly or
transitively) is `dynamic`**, in which case the binary becomes **dynamic** (gains a runtime
interpreter/loader). A `static` library inside a dynamic binary is still absorbed as an
archive. The **hermetic-static** build is the **default** — reproducibility-first (MOD-7)
starts from a self-contained artifact and opts out only where the source explicitly asks.

**Hermeticity is legible (I3).** Our source→object mapping is reproducible regardless of what
is linked; the *scope of the runtime guarantee* is what link mode governs. A build linking
**only `static`** libraries is **hermetic** — self-contained and byte-for-byte reproducible
with the archive pinned by the lockfile (the TOOL-1/MOD-7 guarantee holds end to end). A
build with **any `dynamic`** library is **non-hermetic** — its runtime behavior depends on
the host's installed library — but that status is **determinable from the manifest**, and the
toolchain **surfaces it** (a Config note/diagnostic), so the guarantee's scope stays
**explicit** rather than silently eroded. The C boundary itself remains `unchecked`/unsafe
(FN-8 — foreign calls are forbidden in `checked` code): a linked external is not the
language's correctness concern; pin its version via the lockfile, or the host owns it.

**Why (OP-1).** A structured per-library link mode earns its place over the opaque
`linker_flags` escape because it makes **hermeticity first-class** — manifest-determinable
and compiler-surfaced (I3) — and reproducibility is a core value (MOD-7), so the
hermetic/non-hermetic split must be readable from the closed package, not buried in raw
linker flags. `linker_flags` remains for genuine low-level escapes. This changes only the
link-mode field, not dependency **selection** (still path/git + lockfile, MOD-7), so it is
consistent with MOD-7's post-v1 registry/semver deferral.

**Rejected.** Leaving static/dynamic to raw **`linker_flags`** — opaque; hermeticity would
not be legible from the manifest and the toolchain could not surface it (against I3). A
**global build-wide** static/dynamic switch instead of per-library — less precise, forcing an
all-or-nothing choice when a program wants (say) a static libm and a dynamic libssl. A
`link` field on the **output `Kind`** (executable/static_lib/shared_lib) — that is the
package's *own* output shape, **not** the per-dependency link mode; conflating them would
confuse "what we produce" with "how we consume an external".

---

## Planned sub-themes (to articulate; will append `MOD-10`+)

- *(none currently identified for this theme beyond the entries above)*

### MOD-10. A package's identity in the graph and the lockfile is its source, never its alias
*provenance: refines MOD-7 and TOOL-4 (the lock format) · spec: Modules §8; Tooling §2.4*

**Why.** The v1 lock was keyed by **alias** — one entry per alias, sorted and deduplicated
by it. But an alias is chosen by the *consuming* manifest, while the lockfile is
**graph-wide**: two transitive parents may legally pick the same alias for two different
repositories (they collapse into one lock entry, and one of them silently builds against
the wrong commit), or two aliases for one repository (it is fetched and pinned twice, and
the two pins may disagree). The same hole made the neighbouring rule unimplementable: "one
package name resolving to incompatible revisions is a Config diagnostic" had no definition
of *package name*, since a manifest deliberately has no `Package.name` field. Both problems
are the same missing thing — an identity that does not depend on who is doing the naming.

**The rule.** Identity is the **source**. A git dependency is identified by its **URL
exactly as written** in the manifest; a path dependency by its **lexically-normalized
absolute path**. Lockfile entries are therefore `<url>\t<commit>`, sorted and deduplicated
by `<url>`, with the alias absent from the file entirely. One source resolving to two
different commits anywhere in the graph is a **Config diagnostic** that names both
requesting packages and both commits. Aliases keep doing exactly one job — the local
namespace `<alias>::<module>::…` — and two consumers naming one source differently is
ordinary, not a conflict.

**One alias, one source — within a manifest.** Keying the graph by source means two *different*
manifests may reuse an alias freely, and that is the point. Inside a *single* manifest it is a collision:
`<alias>::<module>::item` would name two distinct declarations, which is exactly what MOD-8 forbids in a
scope, so a duplicate local namespace name is a Config diagnostic naming that name and both sources.
(MOD-14 merged the two naming fields: the package-local namespace name is `Dependency.name`, and
"alias" in this entry reads as that name.) Left unstated it surfaced as an assembler complaint about a duplicate symbol,
which names neither the alias nor either dependency.

**Lexical means lexical.** A path dependency's key takes its absolute base from the process's working
directory — already resolved by the OS — and folds `.`/`..` as text, never through the filesystem. Two
spellings differing by a symlink below that base are therefore two sources. This is the same choice as the
byte-wise URL rule below: an implementation that consulted the filesystem would have to decide what to do
about a broken link, a bind mount or a race, and two implementations would answer differently.

**Byte-wise URL comparison** is deliberate: two spellings of one repository (`https://…`
vs `git@…`, with or without a trailing `.git`) count as two sources, are fetched twice, and
pin independently. The alternative — a normalization model for git URLs — has to encode
host aliasing, credentials, redirects and default branches to be useful, and every
implementation would guess a slightly different equivalence, which is precisely the FND-3
failure this entry removes. A consumer that wants one package writes one spelling.

**Rejected.** Keying by alias — a package-local name deciding a graph-wide fact; silently
merges different packages and splits the same one. Inventing a `Package.name` to key on —
reintroduces the field TOOL-3/TOOL-4 deliberately omitted, and a self-declared name is not
an identity either (two packages may claim it). Content-hashing the checked-out tree
(a `b3:`-style digest) — a git commit is already content-addressed, so it adds a
second identity for the same fact, and it cannot key a path dependency that is edited in
place. URL normalization — see above.

### MOD-11. The package dependency graph is acyclic; a cycle is a diagnostic
*provenance: refines MOD-7 · spec: Modules §8; Tooling §2.4*

**Why.** Nothing said what a dependency cycle means, so an implementation had three
defensible readings: reject it, silently deduplicate the repeated edge and continue, or
treat the packages as one strongly-connected component compiled together. They differ in
whether a program *builds at all*, which is the FND-3 bar. Cycles are also not exotic —
a diamond over a path dependency plus one typo in a relative path produces one.

**The rule.** The graph must be **acyclic**; a chain that returns to a package already on
it is a **Config diagnostic** printing the closing chain. This follows the compilation
model rather than taste: a package is compiled against its dependencies' **finished
interfaces** (TOOL-6 keys a module by the transitive interface hashes of what it depends
on), so a cycle has no valid build order and no fixpoint to compute; and a cyclic graph has
no deterministic linearization to record in the lockfile. Modules *inside* a package are a
different question and unaffected — they are the ordinary tree of Modules §2.

**Rejected.** Silently deduplicating the repeat edge — turns a structural mistake into a
build whose contents depend on traversal order. Compiling an SCC as one unit — needs an
interface fixpoint the model does not have, makes the incremental key ill-defined, and
would let a package's public interface depend on a consumer's. Leaving it
implementation-defined — the same source builds on one toolchain and not another.

### MOD-12. A file and a directory of the same stem are one module
*provenance: refines MOD-1/MOD-8 (§1 module tree) · spec: Modules §1*

**Why.** §1 said files become modules by path and that directories form the hierarchy, but never said what
`geometry.al` **beside** `geometry/` means. Every large module eventually hits this: it grows children and
still has its own code. The three readings — one module, an error, or the file silently shadowing the
directory — differ in whether a program builds, which is the FND-3 bar. An implementation was already
relying on the permissive reading (probe: `lower.al` + `lower/place.al` link and run as one namespace,
`lower__place__…`), so leaving it unstated meant a working program rested on an unwritten rule.

**The rule.** The two halves are **one** module: the file supplies the module's own items, the directory
supplies its children, and either half may be absent. Their scope is a single scope, so MOD-8 applies across
it — declaring a name in `geometry.al` that a child file also claims is ill-formed, exactly as two
declarations of that name inside one file would be. Nothing else changes: paths, privacy (§3, down-tree),
and erasure are unaffected, and there is no new spelling to learn.

**Rejected.** Making the pair an error — it forces a module that grows children to either move its own items
into an artificial child or leave the file empty, which is churn with no gain. A conventional inner file
(`geometry/mod.al`) carrying the module's own items — invents a privileged filename and gives one concept two
spellings (OP-1); the stem already names the module. Silent shadowing in either direction — loses code
without a diagnostic, the outcome I11's spirit and FND-3 both forbid.

### MOD-13. What is emitted depends on the artifact kind — and the program can see the kind
*provenance: refines MOD-5/MOD-6 (the export gate and symbols) and TOOL-7 (the test artifact
excludes the program's entry); interacts with TOOL-11/TOOL-12 · spec: Modules §6; Tooling §2.2,
§2.7*

**Why.** Symbol emission was defined without reference to what is being built: a declaration emits a symbol
if it is `pub`-chain-reachable to the root, or if it carries `@export` (§6.3) — neither asks about
`Target.kind`. For a package that builds both an executable and a library from one source tree (two `Target`
values, TOOL-4) the consequences are concrete. The program's entry — a root-level `_start`, or an
`@export("_start")` — lands **in the `.a`**, where a consumer that defines its own `_start` gets a duplicate
symbol or a silent dependency on which archive member the linker happened to pull (members are per-module,
TOOL-6, and the entry usually shares the root module with the API). Symmetrically the whole public API surface
lands **in the executable**, which has no consumers for it. And the program could not sort this out itself:
`target.*` exposed the machine model only, so there was no `target.kind` to gate on. Meanwhile TOOL-7 had
already established the principle for tests — the artifact's contents depend on what is being built.

**The rule.** Two questions are separated (Modules §6.4). **Eligibility** to emit a symbol stays §6.3's
(the `pub` chain, or `@export`). **What an artifact holds** is a function of `kind`, in two layers: its
**exported-symbol surface** — `executable` exports `@export`s and the entry but **not** the API for merely
being built; a library exports the API (that is what it is); `object` exports everything; `source` emits
nothing — and the declarations **present** in it, which is the reachability closure from that kind's
**roots** (the entry plus every `@export` for an executable; the whole exported surface for an object or a
library). Reachability is defined once — calls, address-taken references, storage access, demanded generic
instantiations — and computed **before** the optimization minimum, so an artifact's contents do not move with
the optimization level, and a private helper reached only from an exported API function is present without
being exported.

The entry exclusion is stated over the **package**, not over one target, and it **overrides `@export`**: the
declaration named by the `entry` of any `Target` **where `entry` is applicable** — and, under
`startup = libc`, that target's start-up-called function — is excluded from every non-`executable`/`object`
artifact. A *defaulted* `entry` on a target where the field is inapplicable contributes nothing (Tooling
§2.2), so a library-only package does not accidentally exclude a root-level `_start` it never uses as an
entry. The test artifact (TOOL-7) is an `executable` whose root is the runner's entry, with the package's
entry excluded by this same rule.

This is a rule about **emission and presence**, not an optimization: `Limit.no_opt` does not reinstate the
API in an executable, and an artifact's specified contents never depend on a linker flag such as
`--gc-sections`. Two consequences are named explicitly. A `static_lib` whose exported surface is empty is
well-formed (an empty archive) — at most a **Semantic** note, since the symbol set is known only after
semantic analysis, never at configuration. Building a `kind = source` target produces no artifact, so
`build`/`run` on it is a **Config diagnostic** pointing at `check`.

For everything finer than the table, the program decides — with the mechanism it already has. `target.*`
(Tooling §2.7) therefore exposes the selected `Target`'s **build fields**, not just the machine model:
`target.kind`, `target.startup`, `target.subsystem`, `target.code_size`, `target.arm_mode`,
`target.vector_length` join `arch`/`os`/`env`/`container`/`endian`/`features`. So
`when target.kind == Kind.static_lib { … }` gates code by artifact exactly as `when target.arch == …` gates it
by machine — no new construct (OP-1). The published set is closed and deliberately excludes `name`,
`output`, `entry` and `auto_cfi` (TOOL-17): the first two name the build rather than describe the program,
gating on `entry` would let code depend on which declaration is the entry, and `auto_cfi` is non-semantic.

**Rejected.** Emitting the same symbol set for every kind and letting the linker sort it out — the `.a`
carries an entry that fights the consumer's, and the executable carries a dead API; both are observable and
neither is diagnosable. Solving it with `--gc-sections` / per-function sections — the artifact's contents would
then depend on the linker's behaviour rather than on documented rules (I1/I3), and `no_opt` would change what
is in the file. Requiring the author to gate the entry by hand (`when target.kind == Kind.executable { … }`
around every `_start`) — ceremony on the single most common shape (one package, an exe and a lib), which the
table handles for free; the gate stays available for the cases the table cannot know. Per-target source
paths (`[lib] path = …`) as the isolation mechanism — there is **one** module tree (MOD-1), and several roots
would be several trees.

### MOD-14. A dependency has one local name, not a name plus an alias
*provenance: refines MOD-7 (dependencies selected by source) and MOD-10 (identity is the source,
never the alias); interacts with TOOL-3 (no `Package.name`) · spec: Modules §8; Tooling §2.4;
Manifest appendix §3.4*

**Why.** `Dependency` carried **two** naming fields — `name`, documented as "the dependency's package
name", and an optional `alias` that overrode it — and neither description survives contact with MOD-10.
A package **has no name**: TOOL-3 removed `Package.name` deliberately, and MOD-10 rejected inventing one
because a self-declared name is not an identity (two packages may claim it) while the graph and the
lockfile key a package by its **source**. So `name` cannot be "the dependency's package name": there is
nothing on the other end to match it against, and the specification indeed never says it is checked
against anything. What it actually does is supply the namespace when `alias` is empty — MOD-10 says as
much, in the phrase "the **effective** alias (the alias, or `name` when it is empty)". Two fields, one
job, with the field that reads like an identity being the one that is purely local: that is the
duplication OP-1 forbids, and it is also a trap — a reader naturally assumes `name` must match the
dependency and that `alias` is what renames it, when in truth **both** are the consumer's own choice.

**The rule.** `Dependency` has **one** naming field: **`name`**, the dependency's **local namespace
name** in *this* manifest — the path its items live under (`‹name›::‹module›::…`, Modules §8). The
`alias` field is **removed**; there is no override because there is nothing to override. The name is
**package-local naming, never identity** (MOD-10 unchanged): the graph and the lockfile key the
dependency by its **source**, the name appears nowhere in the lockfile, and two manifests may name one
source differently or one name may cover two sources across manifests without affecting resolution.
Within **one** manifest the name is a **root-scope name**, because that is where its items hang: it MUST
be a non-empty valid identifier, and it MUST differ from another dependency's name, from the ambient
`alloc`/`std` roots (STD-1), and from any root-module declaration (the manifest handle included) or
`source_dir` child module of this package. The first three are known at configuration and are **Config**
diagnostics; the last is the ordinary duplicate-name error of MOD-8/TOOL-15 and is **Semantic**, naming
both sides. Two dependencies sharing a name would place two declarations under the same
`‹name›::‹module›::item` path, and `Dependency(name = "std")` would shadow a root the language itself
provides — both are MOD-8's rule applied to the root scope.

**Rejected.** Keeping both fields and merely rewording `name` — the redundancy stays, and so does the
reading that one of them is checked against the dependency. Keeping `alias` as the single field instead
of `name` — the term "alias" survives in the prose (an *alias* for the source, which is what it is), but
as a field it reads as optional-by-nature and states the rename twice at every use site
(`Dependency(alias = …)` for something that is simply the dependency's name here). Requiring `name` to
match the dependency's manifest handle (TOOL-15) — that would make the handle a cross-package identity,
which MOD-10 rejected on the merits; it also breaks path dependencies edited in place and adds a check
whose only effect is to forbid renaming. Making the name optional with the source's last path segment as
a default — a git URL's tail is not an identifier (`hal.git`, `libfoo-rs`), so the default would need a
sanitizing rule, and a silently derived namespace name is worse than a stated one (I3).
