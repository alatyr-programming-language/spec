# Aspects map

The **design** order followed dependencies (from semantic cores to syntax); the order of presentation
in the book will be fixed when writing. Status: `[ ]` not started · `[~]` in progress · `[x]` designed.

---

## Status (high level)
**DESIGN PASS COMPLETE — all aspects 0–14 covered + cross-cutting (concurrency/overflow/float).**

The full decision rationale lives by theme in **`decisions/`** — see
**`decisions/README.md`** for the index. This map tracks only **aspect/chapter status**
and **deferred (additive) growth**.

---

## Aspects

- `[x]` **0. Composition / limits.** FND-10/FND-9: single language, unified grammar/AST; **no tier ladder** —
  restrictions via **orthogonal limits** (`no_abstractions`/`no_alloc`/`freestanding`/…); single `.al`;
  manifest ceiling + in-file stricter. `no_abstractions`'s precise 1:1 allow-list is **CG-12** (Assembly §1.1).
  (FND-9, FND-12, FND-11, FND-10, CG-12)
- `[x]` **1. Language manifesto.** FND-4, FND-5: thesis, niche, pillars (I1–I11), non-goals (1–10).
- `[x]` **2. Invariants and contracts.** I1–I11 (see invariants.md).
- `[x]` **3. Execution and memory model.** Subnodes: vocabulary (MEM-6), storage classes (MEM-7), mutability
  (MEM-7), pointers (MEM-8), keystone "storage+lifetime" — **one language reference (scoped) + a library
  allocator-provider menu over first-class reference tokens** via `@alloc(value)` (MEM-1/STD-1, reframed by MEM-3; `get` ordinary via a
  `scoped` second-class return), closures (FN-6), copy/move+linearity (MEM-2). Joint detailing is complete.
- `[x]` **4. Type system.** Foundations TYP-1–CT-2; detailing TYP-2–TYP-12
  (`bitsN` core + interpretations u/i/f,
  construction machinery, aggregates, layout control, value construction, `uint(N)` multiword unsigned
  integers at multiples of 64 — `u128 ≡ uint(128)`, TYP-10; raw-union members have one
  zero/single/anonymous-tuple payload with fixed construction, layout, copy, comptime/static,
  and definitely-active access rules, TYP-11).
  `@require` is the explicit non-owning-underlying constructor gate with an ordinary ABI
  predicate call (TYP-12).
- `[x]` **5. Declarations and bindings.** Unified form `name:type=value` (FN-1), scope/shadowing/order,
  no `const` (FN-2).
- `[x]` **6. Functions / parameters / ABI.** Result via one output-place model: anonymous projected `-> T` or named `out`, with `in`/`in out` directions + projection (FN-4),
  function values (FN-6) + the function-value **type** `fn(T…)->R` (FN-10) + the type-erased **`dyn fn(T…)->R`**
  closure surface (`dyn` keyword + `dyn_over` over explicit storage, FN-11 — resolves the §6.3 dyn spec gap),
  overloading by signature (FN-7), ABI=comptime value + passing modes + defaults/named + variadics (FN-5).
- `[x]` **7. Control flow.** Raw asm (labels/jumps) + structured (if/match/loop/while/for/
  break/return) in a single language (CF-1/FND-9); match exhaustive; safety via MEM-2/TYP-8.
- `[x]` **8. Modules / visibility / packages.** Single tree (files-by-path + `mod{}`), private+`pub`-chain
  (down-tree privacy), import/reexport = declaration `name := path` (`use` removed, MOD-4), dep local namespace names (MOD-14),
  no duplicates (MOD-8/MOD-4). FFI/linking manifest model specified: per-library **link mode**
  (`static`/`dynamic`, static default) → hermetic-static default binary, non-hermetic status
  manifest-determinable + toolchain-surfaced (MOD-9).
- `[x]` **9. Comptime and generics.** Boundary (CT-7), generics (CT-3), constraint-predicates (CT-4),
  comptime execution (CT-8), introspection (CT-6) + comptime field projection `a.(f)` (TYP-9) + comptime
  variant matching `T.(v)` (CT-9 — together making the v1 structural derives ordinary library code over
  `typeinfo` for **every** type, struct/array/enum, no built-in comparison in the compiler),
  `when`-guard (CT-5), operators (OP-2; bit shift/rotate `shl`/`shr`/`rotl`/`rotr` = named call/UFCS ops, OP-6).
- `[x]` **10. Assembly correspondence.** Instruction intrinsics/1:1/checked-guards (CG-2), operands/registers/
  `at()` (arch surface)/UFCS-SIMD (CG-2), data (`[u8;N]`/embed) + data-driven arches (CG-2). Content — additive data.
- `[x]` **11. Codegen / lowering / GAS.** Pipeline structured→backend-specific assembly-correspondence→backend emission
  (GAS→as→ld for register-ISA backends); lowering rules = contract I1;
  determinism; centralized emission; containers by target (CG-1).
- `[x]` **12. Standard library / built-ins.** Prelude tiering (base/alloc/std) + auto-DCE;
  intrinsics vs builtins; panic/exit/assert; allocators `@alloc(value)` (STD-1).
- `[x]` **13. Tooling.** Manifest schema (configuration→build-plan), CLI, diagnostics (stage+span),
  cross-toolchain + reproducible builds (TOOL-1), the `@test` construct + `alatyr test` runner (TOOL-5),
  per-module compilation + the output-neutral content-addressable cache / incremental build (TOOL-6).
  Build defaults: the artifact name — the manifest binding plus the `kind`×`container` affix (TOOL-11);
  the entry point as an Alatyr path + the `startup` mode `raw`/`libc` (TOOL-12) with the per-target
  entry prologue `@abi(entry)` (FN-12); fixed project paths + the `target_dir` layout + CLI-only
  relocation (TOOL-13); upward manifest discovery + the manifest-less invocation (TOOL-14); and symbol
  emission per artifact `kind` with `target.kind` visible to source (MOD-13), and the manifest handle's
  status — package-wide by down-tree privacy, never `pub`, clashing with a same-named module (TOOL-15).
  A dependency carries **one** local namespace name, `Dependency.name`, with `alias` removed (MOD-14);
  vendoring — and with it `vendor_dir`/`--vendor-dir` — is deferred post-v1 (TOOL-16); the configuration
  prelude splits into manifest-only structures and source-visible enums, `target.*` is a closed set, and
  `check` is defined (TOOL-17). The **platform is a `Machine` variant**, not a bag of sometimes-applicable
  `Target` fields — `os`/`container`/`endian`/`subsystem`/`env` become projections, the supported set is
  `(variant, arch)` pairs, and a non-ISA backend is a new variant (TOOL-18). **Android** is such a variant,
  carrying the API level (TOOL-19). The toolchain's output contract is the **build plan** (`--plan` /
  `alatyr plan`); distribution packaging (`.deb`, APK, …) is deliberately **not** a `Kind` and not a
  command — an external tool, or a future `publish`, consumes the plan (TOOL-20).
- `[x]` **14. Lexis and grammar.** Lexis (SYN-1), separators (SYN-4), declarations/expressions (SYN-5),
  attributes `@` — flat syntax + a **family taxonomy** (storage/layout/contract/abi/codegen/linkage/build/
  control/test) for docs & diagnostics, no `@family.name` (SYN-7/3B; isolation via the diagnostic, which names
  the family), control/blocks (CF-1), memory word-functions `ptr`/`deref`/`size`/`align` (MEM-7), `snippet`-removed/`@label`-labels (CF-6/CF-4).
  14.7 EBNF = assembly in the spec while writing.

## Cross-cutting refinements

Cross-cutting decisions — concurrency (`decisions/concurrency.md`), overflow / float / the
verification mode (`decisions/codegen.md`) — are articulated in `decisions/`.

## Deferred (additive future, FND-6) — NOT design gaps, but content/growth
- Instruction sets/arches (CG-2), additional stdlib types/functions beyond the closed v1 surfaces (STD-1), refcount mechanism
  (MEM-1), **`uint(N)` at non-multiple-of-64 widths** (a partial top word + its mask/carry/overflow rules),
  **sub-native** widths (byte-plus-mask), and **signed** multiword `int(N)` (v1 pins `uint(N)` at multiples of
  64 only, TYP-10), per-function effects/effect polymorphism (FND-11), sugar keywords `fn`/`struct` (FN-1), bindgen
  (FN-8), dependency **version constraints + a registry/resolver** (v1 selects by path / git ref only, MOD-7),
  **multi-package workspaces** (no `members` field in v1 — several packages in one repository are wired by
  ordinary path dependencies, TOOL-9),
  `Send`/`Sync` library predicates (CC-5), a **volatile-qualified pointer** `ptr(volatile T)` (CC-2),
  **non-GAS backends** (WASM/WAT, …; the abstraction is fixed by CG-4, the backend itself is additive),
  **prelude convenience catalogs** over the v1 protocol shapes (STD-1): iterator adapters
  (`map`/`filter`/…), `format`-template (gated on comptime-variadics) / `Display`+derive (gated on `typeinfo`) / `format->String` (gated on alloc) / `write_all` over `Writer` — *the minimal alloc-free scalar rendering is v1*, broader structural derives
  (`clone`/three-way `compare`/streaming `Hasher`/serialization), field-level `typeinfo` annotations.
  Implementation details: the `gen` provider surface (its generational-ref representation) + the OS-backed region provider (contracts set by MEM-1; the
  region/handle v1 surface is now specified by **MEM-3** — `Handle(T)`, `@alloc`→handle/trap, access via `get`).
  **Allocator providers:** v1 has the **`arena`** provider surface only (region mechanism, §5.2.1/MEM-3);
  **`gen`/`manual`/`system` are additive** (surfaces specified when first built). There is **no package-wide
  `allocator` manifest field in v1** (deferred/additive — no zero-config default provider; `arena_over` needs a
  caller buffer); allocation is **per-site `@alloc(value)`** on every target.
  Cache fingerprints (TOOL-4). **Vendoring** — a vendored `DepSource` surface, its lockfile check, its place
  in MOD-10's source identity, and the `vendor_dir`/`--vendor-dir` pair withdrawn from v1 with it (TOOL-16).
  All addable without breakage (FND-6).
