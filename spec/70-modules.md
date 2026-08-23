# Alatyr Language Specification

## Chapter — Modules, Visibility, and Packages

> **Status — draft (under review).** This chapter is a draft and is not yet
> accepted; the specification as a whole is under development. See the status note
> at the top of the Overview chapter.

This chapter defines the **module tree**, **path navigation** (`::` vs `.`),
**visibility** (`pub`), **import and re-export by declaration** (there is no `use`
construct), the **symbol model** (mangling, `@export`), **FFI** (`@extern`), and
**packages and dependencies**. It builds on the
Declarations chapter (the `mod` introducer; namespaces), the Functions chapter (ABI
for `extern`), and the Lexis chapter (identifier case-sensitivity; the
symbol spelling rule).

Requirement keywords (**MUST**, **MUST NOT**, **SHOULD**, **MAY**) follow RFC 2119
(Overview §6). This chapter gives the **concrete syntactic forms** it introduces as
grammar fragments (§10); the *consolidated* EBNF is assembled in the Grammar chapter.

---

### 1. The module tree

There is **one** module tree (one language; Overview §3).

- **Files become modules by path:** `geometry/vec.al` is the module
  `geometry::vec`; directories form the hierarchy. Paths are **relative to the package's
  `‹source_dir›`** (Tooling §2.6, default `src`), whose own directory name is **not** part
  of any module path; for a manifest-less invocation it is the directory of the root file
  named on the command line (Tooling §4). A file and a directory of the **same
  stem** are **one** module (MOD-12): `geometry.al` supplies `geometry`'s own items and
  `geometry/` supplies its children, so a module that grows children keeps a home for its
  own code. Either half may be absent. The merged scope is a single scope, so MOD-8's
  no-duplicate-names rule applies across it: `geometry.al` declaring `vec` while
  `geometry/vec.al` exists is ill-formed. The **package root** is anonymous —
  by default the file `package.al`, which carries the single `Package` manifest value
  (configuration) **and** the root module's ordinary code, so a single-file package is normal
  (Tooling chapter; manifest appendix §1; TOOL-3).
- **Inline modules** extend the *same* tree: `Name := mod { … }` — `mod { … }` is a
  value-expression yielding a **namespace value** (Declarations §1.3). A file-defined
  module and an inline module are the same kind of thing.

Modules are a **compile-time** namespace; their navigation (§2) is resolved at
compile time and erased (no runtime cost).

---

### 2. Path navigation — `::` versus `.`

The two access operators are distinct (decisions MOD-3):

- **`::` — namespace navigation** (modules, types, associated items), resolved at
  **compile time**: `geometry::vec::length`; also on a module-valued binding
  (`v := geometry::vec` then `v::length`, §4.2).
- **`.` — value member access** (a field, a tuple index) **and UFCS** (a method-style
  call), at **run time**: `s.field`, `value.method(x)` ≡ `method(value, x)`; a
  **qualified** UFCS call `value.ns::f(x)` ≡ `ns::f(value, x)` reaches a module's free
  functions without importing bare names (`p.atomic::load(ord)`, `p.volatile::load()`;
  MOD-3/CT-10). `a.f(args)` is **always UFCS** — a function-valued field is called
  `(a.f)(args)` (β). The receiver (and a `.`-field / `[]`-index base) coerces one level between
  a value and a pointer to it — **auto-ref** (`a` → `ptr(a)`) or its dual **auto-deref**
  (`p` → `deref(p)`) — per the receiver/access coercion rule (Type System §4.5 / MOD-3 / OP-5).

This disambiguates a **qualified call** `geometry::length(x)` from a **UFCS call**
`value.length(x)` — they are syntactically different. (The distinction is **syntactic** —
`::` for a qualified call, `.` for UFCS — independent of identifier case; Lexis chapter.)

---

### 3. Visibility

Visibility is **Alatyr namespace visibility** — *who may name a declaration*. It is
distinct from **linker-symbol emission** (whether a symbol appears in the object
file), which is §6; `@export` (§6.3) controls the latter and is **not** governed by
the rules here.

Privacy flows **down** the module tree, exposure flows **up** only through `pub`:

- A declaration is **private by default**. A **non-`pub`** declaration of module `M`
  is visible to **`M` itself and to every module nested within `M` (its
  descendants)** — and to `M`'s own inner scopes (blocks, functions, comptime
  blocks). A **descendant module sees its ancestors' non-`pub` items**; this is what
  lets a submodule use its parent's internal helpers without exposing them. It is
  **not** visible to `M`'s ancestors, siblings, or unrelated modules.
- **`pub`** (a bare prefix keyword, Declarations §2) makes a declaration visible to
  `M`'s **enclosing (parent) module** — i.e. **upward** by one level. Visibility
  propagates further **outward only through a continuous `pub` chain**: it reaches
  *N* levels up only if every enclosing module up to that level is also `pub`; a
  break stops it. Reaching **outside the package** (external/library API) requires
  the `pub` chain all the way to the root — a library's **public API** is exactly the
  `pub`-chain-to-root-reachable surface.
- A child module (whether file-defined or inline `mod { … }`, §1) is a **descendant
  scope** for this rule: it sees its ancestors' private items, but to expose its own
  items upward it must `pub` them. There is **no** friend / sibling / package-private
  access beyond this down-tree privacy and up-tree `pub`.

---

### 4. Importing and re-export — by declaration

There is **no** `use` construct. Because a `::`-path resolves to a **value** — a
module, type, function, or constant is a compile-time entity (§1; Declarations §1) —
importing, aliasing, and re-exporting are all just the ordinary declaration form
`name := value` (Declarations §1), with `pub` for re-export. A qualified path is
**always** usable directly; binding a name is only for brevity.

#### 4.1 Import / alias — a declaration bound to a path

```
vec := math::vec               # alias the module; then vec::length(p), vec::Vec3(…)
len := math::vec::length       # bring one item under a local name
```

A path resolving to a compile-time namespace entity makes the binding a **comptime
alias**: it is **erased** and emits **no** symbol (§9). What a binding may name
follows the §3 visibility rule: any `pub` name reachable through the tree, **plus** an
ancestor's non-`pub` name when the binding site is a **descendant** (down-tree
privacy, §3); a non-`pub` name of a non-ancestor module is **not** accessible.
Renaming is automatic — the **left-hand** name is whatever you choose (`len`, `vec`).

A binding to a **runtime** place (e.g. a mutable global) is **not** an import: it is
an ordinary read of that place's value (Memory-model) — a distinct operation. Import
is specifically the comptime-entity alias.

##### 4.1.1 Listed member projection — `(a, b) := M`

A multi-name binding whose right-hand side is a **module / namespace path** projects
each listed name onto the **same-named member** of that module:

```
(fmt, vec) := std              # ≡  fmt := std::fmt ;  vec := std::vec
(sin, cos) := math::trig       # ≡  sin := math::trig::sin ; cos := math::trig::cos
```

Each name is bound to `M::<name>` — a comptime alias apiece (§4.1), erased, no symbol.
A projected member may be **any comptime entity** §4.1 admits — a submodule, a
**function**, a **`comptime` constant** (§Declarations: a constant is a `comptime`
binding — there is no `const`), or a **type** — exactly what a single `name := M::x`
aliases; projection is the symmetric *N*-name form of that one declaration:

```
(println, eprintln) := std::io     # functions
(MAX, MIN) := i64::limits          # comptime constants
(Vec3, Quat) := math::types        # types
```

It is **sugar** for the *N* single declarations and nothing more: the names are listed
**explicitly** (it is *not* a glob — §4.5), and projection binds **by name** (no
renaming; use a single `x := M::y` to rename). A listed name that is not a `pub` member
of `M` (subject to the §3 visibility rule) is a **compile error**. `pub (a, b) := M`
re-exports each (§4.3).

This **reuses the multi-out destructuring form** `(…) := expr` (Grammar §3.2): the
meaning is selected by the **right-hand side**, exactly as a single `:=` already means a
value-bind or a comptime alias by its RHS (§4.1) — a **module / namespace path** RHS is
this member projection; a **multi-output call** RHS is positional value destructuring
(Functions §3.2).

#### 4.2 Accessing a bound module

A name bound to a module is navigated with **`::`** exactly as a tree path:
`vec::length` where `vec := math::vec` (this extends `::` from tree paths to
module-valued bindings, decisions MOD-3/MOD-4). `.` remains value access / UFCS (§2).

#### 4.3 Re-export — `pub name := path`

`pub` on an import re-exports it into the module's public surface:

```
pub Vec3 := math::vec::Vec3            # re-export one item
pub geo  := math::geometry             # re-export a whole module (one line)
pub Body := physics::body::Rigid       # re-export with a new name
```

A re-export is an **alias at the name level**, **not** a new binary symbol: the symbol
is defined once at the declaration site (its name is the definition's path, §6, or its
`@export`), and `pub name := path` merely makes it reachable under another path — the
**same** symbol, no duplicate. It also widens the **export gate**: the `pub` chain that
makes a declaration an automatic linker export (§4.4) **includes** these `pub` import
bindings.

**A re-export may name only `pub` items.** A descendant's down-tree access to an
ancestor's **non-`pub`** item (§4.1) is for **local use only**; re-exporting it is a
**compile error** — otherwise a submodule could re-publish an ancestor's private helper
upward, overriding the owner's deliberate privacy. Re-export never elevates a private
item to public.

#### 4.4 The export gate

A declaration is **automatically** a linker export iff there is a continuous `pub`
chain (including the `pub`-import re-exports of §4.3) to the root (§3); the symbol name
is the definition's path (§6) or its `@export`. This gate decides **eligibility**;
whether that export actually lands in a given artifact is the artifact-`kind` rule of
**§6.4** (an executable does not export the API merely for being built). This automatic gate is the rule for the
**`pub`-API surface**; it is **not** the only way a symbol is emitted — **`@export`**
(§6.3) emits one **regardless of** the `pub` chain. `@export` is on the
**linker-emission** axis, orthogonal to namespace visibility (§3).

#### 4.5 No glob; listed member projection; collisions

There is **no** glob ("import everything unqualified", `X::*`) — invisible imports
are out. A **listed module-member projection** **is** allowed (§4.1.1): `(a, b) := M`
binds each **listed** name to `M::<name>`. The names stay **explicit** (no invisible
import, no namespace pollution); the projection only removes the per-name `M::`
repetition — sugar for the *N* single declarations. A whole library is still one
binding (`Lib := dep::lib`, accessed `Lib::…`); the full path is always an alternative
to importing at all.

A binding whose name **clashes** with an existing name in the module scope is a
**compile error** — no implicit winner (§5).

---

### 5. No duplicate names

Within a module scope, **no two** declarations, imports, or re-exports may yield the
**same final name** — a clash is a **compile error**, with no implicit winner
(decisions MOD-8; consistent with same-scope redeclaration, Declarations §6.2).

Because identifiers are **case-sensitive and underscore-significant** (Lexis chapter),
names differing only by case or underscore are **distinct** — they are **not** duplicates;
duplicates are by **exact spelling**.

**What names a scope.** A module's scope is named by its own declarations, imports and
re-exports **and by its child modules by path** (§1): `geometry/vec.al` puts `vec` into
`geometry`'s scope, which is why `geometry.al` declaring `vec` is the same clash
(MOD-12). For the **anonymous package root** the two halves are the **manifest file** and
**`‹source_dir›/`** (Tooling §2.6), so a declaration in `package.al` clashes with
`‹source_dir›/‹same-name›.al` — in particular the **manifest handle** itself
(`mylib := Package(…)` beside `‹source_dir›/mylib.al`; TOOL-15). Because one side of such
a clash is **never written as a declaration**, the diagnostic MUST name **both**: the
declaration and the file whose path became the module.

---

### 6. Symbols — export and mangling

#### 6.1 The export symbol

An exported declaration's **linker symbol** is its **declaration spelling** — **not**
normalized: case and underscores exactly **as written** (Lexis chapter) — qualified
by its **module path**, with each `::` rendered as **`__`** (a valid C identifier
separator):

```
geometry::vec::length   →  geometry__vec__length
```

The result is a valid C identifier — v1 identifiers are ASCII (Grammar §2.2) — so an
Alatyr export is **linkable and callable from C directly**. A root-level declaration is **unprefixed** (`_start` → `_start`).

#### 6.2 Signature suffix for overloads

For a non-overloaded declaration, the symbol is the **path-derived spelling** of §6.1:
the declaration spelling qualified by `::` → `__`.

For a **function overload set** (Functions §1.4 / FN-7), each overload emits a distinct
symbol whose path-derived spelling is suffixed by a deterministic encoding of its
parameter signature. The suffix is transparent and compile-time-only (I1/I3), with no
runtime dispatch (I2). This is the only signature-based addition to the symbol scheme.
An exact `@export("name")` or `@extern("name")` uses the exact supplied foreign/linker
name (§6.3, §7.2).

#### 6.3 `@export`

A declaration becomes a **linker symbol** in either of two **independent** ways:

- **automatically**, if it is `pub`-chain-reachable to the root (§3, §4.4) — the
  pub-API surface is the library's symbol surface (this automatic emission is what the
  artifact-`kind` table of §6.4 qualifies: an executable does not carry the API merely
  for being one);
- **explicitly**, if it carries **`@export`** — a **linker-level directive** that
  emits the symbol **regardless of** the `pub` chain. This is for declarations that
  must be linker-visible but are **not** part of the Alatyr namespace public API — an
  interrupt handler, a C-callable internal routine, a symbol a linker script names.
  `@export("exact")` gives an **exact** symbol name (C interop, a fixed entry name). An
  `executable`'s entry needs no `@export`: it is emitted because `Target.entry` names it
  (§6.4).

The two axes do not conflict: `pub` is **namespace visibility** (who may *name* it in
Alatyr), `@export` is **symbol emission** (what the *linker* sees). A non-`pub`
declaration with `@export` is a linker symbol yet not nameable as public API; a
`pub` declaration is auto-emitted without needing `@export`. (`@symbol` is not used.)

#### 6.4 What is emitted depends on the artifact kind

Two questions must not be conflated. **Which declarations may emit a symbol** is §6.3 (the
`pub`-chain gate, or `@export`). **What an artifact contains** is fixed here by the built
target's **`kind`** (Tooling §2.2; MOD-13), in two layers: the artifact's **exported
symbols**, and the **declarations present** in it — the closure of what its **roots** reach.

**The exported-symbol surface:**

| `kind` | `pub`-chain-to-root API | `@export` | the entry declaration |
|---|---|---|---|
| `executable` | **not** exported | exported | exported |
| `static_lib` / `shared_lib` | exported (it **is** the API) | exported | never present — see the exclusion |
| `object` | exported | exported | exported |
| `source` | nothing is emitted (the sources merge into the consumer) | — | — |

**The roots** — what the artifact must contain:

- **`executable`** — the **entry**: under `startup = raw` the declaration `Target.entry`
  names, under `startup = libc` the start-up-called function (ABI appendix §3.3); plus
  every `@export` declaration.
- **`object`** — its whole exported surface (someone else links it, so every exported
  symbol is a root). `entry` selects nothing at link time here — an object has no link
  step — it only makes that declaration a root and an exported symbol.
- **`static_lib` / `shared_lib`** — every exported symbol of the table above.
- **`source`** — none; the consumer's roots apply after merging.

**Reachability.** A declaration is **present** in an artifact iff it is a root or is
**reachable** from one, where reachable is the static closure over: calls; address-taken
references (a function value or a `dyn` capture, Functions §10/§11); reads or writes of a
declaration's storage; and the instantiations a generic root demands (Comptime §3.4). The
closure is computed **before** the optimization minimum (Codegen §3; CG-5), so an
artifact's contents do not vary with optimization level. Presence and export are
**independent**: a present but non-exported declaration emits no `.global` (Codegen §5.1),
which is how a private helper reached only from an exported API function is in the archive
without being part of the API.

**The entry exclusion is package-wide and overrides `@export`.** The declaration named by
the `entry` of **any** `Target` of the package **where `entry` is applicable** (Tooling
§2.2) — and, under `startup = libc`, that target's start-up-called function — is excluded
from every artifact that is not `executable` / `object`, **even when it carries
`@export`**. A **defaulted** `entry` on a target where the field is inapplicable
contributes nothing to the exclusion set (Tooling §2.2). A library target declares no
`entry` of its own, yet must still leave the program's entry out of the archive.

**The test artifact** (Tooling §4.1; TOOL-7) is an `executable` whose root is the
**runner's** entry; the package's entry is excluded from it by the same rule, and the
`@test` items are additional roots.

This is a rule about **emission and presence**, not an optimization: `Limit.no_opt` does
not reinstate the public API in an executable, and the **specified** contents of an
artifact — its exported surface and its present declarations — are fixed by these rules
alone, never by a linker flag (I1/I3). A linker may additionally strip what these rules
already leave unreferenced (`ld --gc-sections`; Stdlib §2), which is why that flag can
never *add* to an artifact or decide an exported symbol's fate: everything it could remove
is by construction absent from the specified set. A `static_lib` whose exported surface is
**empty** is well-formed — an empty archive, at most a **Semantic** note (the symbol set is
known only after semantic analysis, never at configuration).

For distinctions the table cannot know, source gates on the artifact itself:
`when target.kind == Kind.static_lib { … }` (Tooling §2.7).

#### 6.5 The symbol is the declaration spelling

Identifiers are **case-sensitive** and underscore-significant (Lexis); the export symbol
is the **declaration spelling** (§6.1), so distinct spellings are distinct symbols. Names
differing in case or underscore are **distinct** identifiers (not duplicates), each with
its own symbol — there is no case-folding collision.

#### 6.6 Edge — `__` in a declared name

A `__` *within* a declared name is a **lint** (it could be misread as a path
separator). Demangling is by the **module tree**, not a blind split on `__`, so it is
unambiguous; and for C, `__` is part of a valid identifier.

#### 6.7 Linker-symbol uniqueness

The no-duplicate rule of §5 governs **names within a module scope**; it does **not**
by itself prevent two declarations from emitting the **same linker symbol**, now that
`@export` (§6.3) can emit symbols independently of `pub`. Therefore: **every emitted
linker symbol MUST be unique across the whole package** — a clash is a **compile
error**. In particular, two `@export("foo")`, or an `@export("foo")` colliding with
an automatically-derived symbol `foo`, or two path-derived symbols that mangle to the
same string, are all ill-formed. (Path-derived symbols are *usually* distinct — module
paths are distinct and names are unique per scope (§5) — but a declared name containing
`__` (a lint, not an error, §6.6) can collide after mangling, e.g. `a::b`'s `c` and
`a`'s `b__c` both → `a__b__c`; this rule catches those **and** `@export("exact")` names,
which the path scheme does not police.) Collisions with
**external** (`@extern`) symbols are resolved by the linker per the usual rules; the
compiler guarantees uniqueness only among the symbols **it** emits.

---

### 7. FFI — importing external symbols

#### 7.1 `@extern` declaration

An external symbol is declared with the **`@extern`** attribute (a linkage-cluster
attribute, not a keyword): a **signature + ABI, with no body**:

```
printf := @extern @abi(c) fn(in fmt : str, args : ...) -> i32
```

This tells the compiler: the symbol is **external**, call it with this **ABI and
signature** (the result via `-> T` or named `out`; Functions §3), and **resolve it at
link time**.

`@extern <fn-sig>` is a **body-less function declaration form**, not an ordinary
function value: it is its own grammar production (`extern-decl`, Grammar §3.11), valid
only as the RHS of a binding, and it has **no body to evaluate or pass**. It names an
external callable for calls and linking — it cannot itself be used as a first-class
function value (there is nothing to project or store beyond its address, which is taken
as for any symbol).

#### 7.2 External name

The external symbol's name is the **declared** name (§6.1) by default, or an exact
name via **`@extern("exact_external_name")`** — which accepts **any foreign scheme**
(a flat C name, a C++-mangled name, a Rust symbol, a `__`-separated name). Import is
deliberately flexible; export is C-compatible (§6.1).

#### 7.3 Symmetry

`@export` / `@export("…")` face **outward**; `@extern` / `@extern("…")` face
**inward**. One pair per direction.

#### 7.4 The C ABI is the common denominator

`@abi(c)` is the FFI common denominator — other languages are reached through the C
ABI. C-variadics exist for FFI only (Functions §7.3; `unchecked`-grant only). Foreign
relocations/TLS are asm-level operands (Assembly chapter).

#### 7.5 Linking and link mode

External libraries, their paths, and linker flags are declared in the **manifest** and
passed to `ld` (Tooling chapter); dynamic imports go through the container's import
mechanism (Codegen chapter).

Each external library carries a **link mode** — a first-class, **per-library** property
(MOD-9), not an opaque linker flag:

```alatyr
LinkMode := enum { static, dynamic }
Lib      := struct { name : str, link : LinkMode = LinkMode.static }
```

- **`static`** (the default) — the library's `.a` archive is **absorbed** into the
  binary. A bare name (`Lib(name = "m")`) links statically.
- **`dynamic`** — the binary **references** the `.so`/`.dll`, resolved by the OS loader
  at run time (`Lib(name = "ssl", link = LinkMode.dynamic)`).

**Binary link-mode rule.** The produced binary is **fully static** — self-contained, with
**no** runtime interpreter and **no** `.so` dependency — **unless at least one linked
library (directly or transitively) is `dynamic`**, in which case the binary becomes
**dynamic** and gains a runtime interpreter/loader. A `static` library inside a dynamic
binary is still absorbed as an archive. The hermetic-static build is the **default**.

**Hermeticity is manifest-determinable (I3).** A build linking **only `static`** libraries
is a **hermetic build** — self-contained and byte-for-byte reproducible with the archive
pinned by the lockfile (the reproducibility guarantee, Tooling §6.2 / TOOL-1, holds end to
end). A build with **any `dynamic`** library is **non-hermetic** — its runtime behavior
depends on the host's installed library. This status is **determinable from the manifest**,
and the toolchain **surfaces it** (a Config note/diagnostic, Tooling §2.5), so the
reproducibility guarantee's *scope* stays explicit rather than silently eroded. The
source→object mapping is reproducible regardless of what is linked; link mode governs only
the runtime dependency. (The C boundary itself stays `unchecked`, §7.4 / FN-8 — a linked
external is not the language's correctness concern; pin its version via the lockfile, or the
host owns it.)

---

### 8. Packages and dependencies

- A **package** is the unit of build and distribution — defined by its **manifest**
  (configuration; Tooling chapter).
- **Dependencies** are declared in the manifest (a path or git source, pinned by a
  hashed lockfile; Tooling chapter).
- A dependency's items live under its **local namespace name** — the single naming field
  `Dependency.name` (MOD-14), a non-empty identifier that shares the **root scope** with
  this package's own root declarations, its `source_dir` child modules and the ambient
  `alloc`/`std` roots, so a clash is a diagnostic (§5; Manifest appendix §3.4):
  `<name>::<module>::…` — they do
  **not** flatly pollute the importing namespace. They are reached qualified, or bound
  to a local name (`m := <name>::<module>`, §4.1). That name is **package-local naming**,
  never identity: the dependency **graph** and the lockfile key a package by its
  **source** — the git URL as written, or a path dependency's lexically-normalized
  absolute path (MOD-10, Tooling §2.4).
- **Selection is by source only, exact-pinned (v1).** A dependency is a **path** or a
  **git ref**; there is **no** per-dependency semver version constraint and **no** central
  registry in v1 — resolution is `path` / `git-ref` + the hashed **lockfile**, which is what
  makes builds byte-for-byte **reproducible** (Tooling / TOOL-1; MOD-7). A **version-
  constraint solver + a package registry** are **additive** (post-v1, FND-6): a `version`
  field and a registry source would be new optional inputs, added without breaking a v1
  manifest.
- **The dependency graph is acyclic** (MOD-11). A package that depends, directly or
  through any chain, on itself is a **Config diagnostic** that prints the cycle as the
  chain of sources that closes it. There is no package-level mutual recursion: a package
  is compiled against its dependencies' finished interfaces, so a cycle has no valid
  build order to begin with. (Modules *within* one package are unaffected — that is the
  ordinary tree of §2.)
- The **standard library tiers are not dependencies**: `alloc` and `std` are
  **compiler-provided ambient root modules** of those names (Stdlib §1), sharing this
  same tree — reached qualified (`std::io::stdout`) or bound (`out := std::io::stdout`,
  §4.1), gated by the `no_alloc` / `freestanding` limits. The base prelude is injected
  unqualified. No manifest entry brings them in.

---

### 9. Lowering (summary)

- The module tree and `::` navigation are **compile-time** and **erased** — no
  runtime representation, no runtime cost.
- An exported declaration becomes a **linker symbol** by §6; a non-exported name
  need not.
- A **re-export adds no symbol** (§4.3); it only widens name reachability.
- An **`@extern`** declaration emits no definition; it is **resolved at link** (§7).
- `pub` and import/re-export bindings (`name := path` / `pub name := path`) affect
  names and reachability, not runtime behavior (I3); an import binding to a comptime
  entity is a comptime alias, erased.

---

### 10. Syntax (grammar fragments)

The forms this chapter introduces are given here as **grammar fragments**; the
**consolidated EBNF** is assembled in the Grammar chapter. Nonterminals not defined
here (`ident`, `module-body`, `string`, `fn-sig`, `attribute`) come from the
Declarations, Functions, and Grammar chapters.

```ebnf
(* inline module — a value introducer (Grammar §3.4), used as the RHS of an ordinary  *)
(* binding; its body is a `module-body` (declarations only), NOT an executable block. *)
mod-value    ::= "mod" "{" module-body "}"              (* Geometry := mod { … } *)
(* there is no separate `module-decl`: `Geometry := mod { … }` is `ident ":=" mod-value` *)

(* path navigation (§2) *)
path         ::= ident { "::" ident }                   (* :: = namespace; `.` = value/UFCS (Type System §4.5) *)

(* import / alias / re-export (§4) — NO `use` construct; it is an ordinary declaration *)
(* whose value is a `::`-path (`path` above); `pub` re-exports. e.g.:                   *)
(*   vec := math::vec                  (import/alias a module)                          *)
(*   len := math::vec::length          (import one item)                                *)
(*   pub Body := physics::body::Rigid  (re-export, renamed)                             *)
(* (see `declaration` in Declarations §9.)                                              *)

(* visibility (§3) — pub is a prefix keyword on any declaration *)
pub-decl     ::= "pub" declaration

(* symbols (§6) *)
export-attr  ::= "@export" [ "(" string ")" ]            (* force symbol / exact name *)

(* FFI (§7) — @extern: a body-less signature (fn-sig from Functions §9) + ABI, NO body *)
extern-decl  ::= ident ":=" "@extern" [ "(" string ")" ] { attribute } fn-sig
               (* e.g. printf := @extern @abi(c) fn(in fmt : str, args : ...) -> i32 *)
```

Normative notes:

- `::` is **namespace** navigation (compile-time); `.` is **value** access + UFCS
  (run time) — they are distinct operators (§2).
- `pub` is a bare prefix keyword (Declarations §2); export to the root needs a
  continuous `pub` chain (§3).
- A `@extern` declaration has **no body**; an `@export`/`@extern` string gives an
  exact symbol name (§6.3, §7.2).
- The export symbol is the declaration spelling with `::`→`__`; overloads add the
  deterministic parameter-signature suffix of §6.2.

---

### 11. Conformance (normative summary)

A conforming implementation MUST:

1. build **one** module tree from files-by-path and inline `mod { … }` modules, with
   an anonymous manifest-defined package root, resolved at compile time and erased
   (§1, §9);
2. treat `::` as compile-time namespace navigation and `.` as value member
   access / UFCS, keeping them distinct (§2);
3. make declarations **private by default** (Alatyr namespace visibility, distinct
   from linker emission) with **down-tree privacy**: a non-`pub` item is visible to
   its module and all **descendant** modules but not to ancestors/siblings/unrelated
   modules; make `pub` visible to the parent and require a **continuous `pub` chain**
   for further outward / external visibility; no friend/sibling/package-private access
   beyond this (§3);
4. provide **no** `use` construct: import / alias / re-export are the ordinary
   declaration `name := path` (a comptime alias of a namespace entity, erased, no
   symbol) and `pub name := path` (re-export); navigate a module-valued binding with
   `::`; provide **no** glob and **no** selective-bulk import (N imports = N
   declarations); allow a binding to name any `pub` tree-reachable item plus an
   ancestor's non-`pub` item from a descendant site (§3); treat a re-export as a
   **name alias, not a new symbol**, restrict re-export to **`pub`** items, and count
   `pub` re-exports in the **export gate** (§4);
5. reject duplicate final names in a module scope, by **exact spelling**
   (case-sensitive, underscore-significant; §5) — counting its own declarations /
   imports / re-exports **and** the names its **child modules by path** introduce
   (for the root: the manifest file's declarations against `‹source_dir›/`'s children),
   naming both sides when one of them is a file rather than a declaration;
6. distinguish an artifact's **exported symbols** from the declarations **present** in
   it, fixing both per artifact `kind` (§6.4) — the roots per kind, presence as the
   reachability closure from a root computed before the optimization minimum, the
   package-wide entry exclusion (which overrides `@export`), and the test artifact's
   runner entry — and derive an export symbol as the **declaration spelling** (case-sensitive, underscores
   as written) qualified by `::`→`__`, root unprefixed, and add a deterministic
   parameter-signature suffix for overloads (§6.2; Functions §1.4);
   emit a linker symbol either automatically (pub-chain-reachable) **or** explicitly
   via **`@export`** / `@export("exact")` — a linker-emission directive **orthogonal**
   to `pub` namespace visibility (so a non-`pub` `@export` symbol is valid); place those
   symbols in an artifact per the **`kind` table of §6.4** (no automatic API in an
   executable; the package's entry declaration excluded from a library; nothing emitted
   for `kind = source`) as a rule of emission, not an optimization; and reject any clash
   among the linker symbols it emits — every emitted symbol unique package-wide,
   including `@export("exact")` names (§6);
7. resolve an **`@extern`** declaration (signature + ABI, no body) at link time, take
   its external name as declared or `@extern("exact")`, and use `@abi(c)` as the
   FFI denominator (§7); honor each external library's **link mode** (`static`
   default / `dynamic`, per-library; §7.5), produce a **fully static (hermetic)** binary
   unless some library is `dynamic` (directly or transitively) — then a dynamic binary —
   and **surface** the non-hermetic status when any `dynamic` library is linked (§7.5;
   Tooling §2.5);
8. treat a package as the manifest-defined build/ship unit, with manifest-declared
   dependencies placed under their local namespace name — one naming field, unique
   within a manifest, never an identity (no flat pollution) (§8; MOD-14);
9. introduce **no** runtime cost or representation for modules, visibility, or
   import/re-export bindings (§9; I3).
