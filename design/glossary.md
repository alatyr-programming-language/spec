# Glossary

Canonical terms, so that terminology doesn't drift between chapters. Expanded as work progresses.

- **Limits** — orthogonal opt-in restrictions of a unit (FND-10; there is no tier ladder). One language: raw
  assembly constructs + structured + comptime coexist; limits *forbid*: `no_abstractions`
  (only 1:1 GAS), `no_alloc`, `freestanding`, `no_comptime`, `no_opt` (un-optimized output, CG-5), … Combined freely. Declared:
  manifest `limits:` (package ceiling) + in-file `@limits(...)` (stricter). One extension `.al`.

- **`no_abstractions`** — the limit restricting a translation unit to **only the 1:1 assembly
  surface** (Assembly §1.1; CG-12): every emitted runtime instruction is one the programmer
  wrote, 1-to-1 with GAS. **Admits** instruction intrinsics + `asm`, operands (registers /
  immediates / `at(…)` memory / labels / decorations), raw transfer (native — no grant here),
  data definitions, the ABI-framed-or-naked function frame over local operands with a returned
  destination, and **comptime** (erased, so exempt). **Forbids** (compile error) the *structured
  abstractions*: the library value-operators (`+`/`-`/`*`/…), structured control flow
  (`if`/`while`/`for`/`match`/…), `?`, calls to non-instruction functions, structured data
  access (`s.f`, enum `match`, `a[i]`, slices), and `ptr`/`deref` + pointer arithmetic — each
  re-expressed via an intrinsic, labels + raw transfer, or an `at(…)` operand. Binds the unit's
  own source (FND-12), orthogonal to the `checked`/`unchecked` axis (CG-6).

- **Capability** (*capability*) — a tracked ability forbidden by a limit: allocation, IO, panic,
  writing unchecked operations, OS-dependence. (Future per-function effects — the same vocabulary;
  thread-safety `Send`/`Sync` are **library-level** type-capabilities — comptime predicates over
  `typeinfo` with a nominal brand opt-out, not a language mechanism — additive, CC-5.)

- **Verification: `checked` / `unchecked` / `no_unchecked`** (CG-6) — the axis "is there a built-in check", the name
  is about the *mechanism*, not about "safety" (our default is weaker than Rust's `safe`: races are hardware-defined,
  there is no borrow-checker). **`checked`** — the fixed unnamed default (not overridable by the manifest):
  the operation is provably correct or deterministically traps (I11). **`unchecked`** — a bare keyword,
  a **scoped verification mode** (CG-7, parallel to `comptime`): within the scope — `unchecked expr` (one
  expression) or `unchecked { … }` (a block) — the **checked-guard family is dropped** (arithmetic wraps,
  indexing skips the bounds check, narrowing truncates, float→int and div0/shift/alignment faults are
  hardware-defined; never UB, I11) **and** the genuinely-raw operations (pointer arithmetic,
  int→ptr, a raw-union member read not proven definitely active / an `uninit` read, raw
  transfer, `asm`) become writeable. Granularity is the author's choice (one line or a
  block); the scope is always lexically visible. **No re-entry / no `checked` keyword**
  (symmetry with phase: no
  `runtime` from inside `comptime`). **`no_unchecked`** — a limit
  transitively forbidding the word `unchecked` from appearing (an audit promise). Don't confuse: `unchecked` = local
  narrowing (keyword), `no_unchecked` = a ban on narrowing (limit). Symmetry with phase: unnamed default · exception marker
  · banning limit (phase: runtime / `comptime` / `no_comptime`). The mode is **non-transitive by default** (it does not
  reach into a called function's body) but is an **observable comptime fact** — `verify.checked` (CT-11) — readable through
  `when` / `comptime if`, see below.
- **`verify.checked` / mode-polymorphic / mode-opaque function** (CT-11) — the verification mode is a **comptime-known
  fact** exposed as the comptime boolean **`verify.checked`** (true in a checked context, false inside an `unchecked`
  scope; the scope *sets* the mode, `verify.checked` only *reads* it — not re-entry). A function whose body reads it
  (`comptime if verify.checked { <guard> }`) is **mode-polymorphic**: instantiated per the verification mode of the call site
  (≤ 2 variants, the unused one DCE'd, zero-cost). A function that does not read it is **mode-opaque** — its verification
  is fixed at definition and a caller's `unchecked` does not alter it. This is how a **library-defined** operator/conversion
  (TYP-2/TYP-5) drops its guard inside `unchecked` (the guard is comptime-absent in the unchecked instantiation) without the
  compiler recognizing the guard.

- **Raw union / member payload / definitely active** (TYP-11) — a **raw union** is an
  untagged overlap of named members, all at offset `0`, with no hidden discriminant or
  run-time active-member metadata. A member has exactly one **payload**: `()` for zero
  components, the component type for one, and an anonymous tuple `(T0, T1, ...)` for two
  or more, using ordinary tuple layout/projection. A **trackable union place** is a binding
  root or fixed non-pointer subobject path; pointer-derived/dynamic places are active-unknown.
  Such a place, or a union value expression carrying the same analysis facts, is
  **definitely active as `m`** when the fixed compile-time dataflow analysis proves it
  definitely initialized and proves every reaching path activated `m`; checked `u.m`
  requires those facts, while `unchecked u.m` permits raw reinterpretation.
  Construction is type-qualified (`U.m(args)`); whole-member assignment is `u.m = payload`;
  `u.m(args)` is UFCS, not a union write.

- **`@inline`** (CG-10) — a function attribute declaring the function **substituted at its direct call sites**:
  the body is spliced in (each argument bound to a fresh local as needed), so **no `call`** is emitted and the
  function has **no standalone symbol** unless its address is taken or it is `@export`ed (an out-of-line copy is then
  emitted too). Inlinability is thus **declared, not inferred from body shape** (replaces the implicit
  one-instruction-body heuristic) — the native operators (TYP-2) carry it. Ill-formed on a recursive function. Distinct
  from `@abi(value)` (which sets the calling convention of a *still-called* function); the two compose
  (`@inline @abi(c)` — inlined at direct calls, C-ABI for the out-of-line copy).

- **`@require(pred) U` / required type / constructor gate** (TYP-12) — the type-function
  `require(U, pred)`, producing a **distinct, layout-identical** type `R`. `pred` is a
  comptime-known callable specialized to `fn(in U) -> bool`; v1 permits only non-owning
  `U` and only comptime captures (erased by specialization). In checked `R(v)`, `v` is
  evaluated once, `pred` is invoked once through the ordinary ABI on a distinct logical
  by-value argument while the result is preserved, false emits the target's direct inline
  trap, and true delivers `R`. `unchecked R(v)` omits the argument copy, call, branch, and
  trap. The gate is explicit `U`→`R`, never an implicit conversion; a required aggregate
  is built as `R(U(...))`, cannot be partially initialized or component-mutated in checked
  code, and may be replaced whole by another `R` (`unchecked` bypasses only the contract,
  not ordinary field immutability). Within `R(v)` dispatch, a source not assignable to
  `U` may use only an ordinary **direct** `@convert` to `R`; dispatch never composes a
  conversion to `U` with the gate. Repeated decorators nest nearest-first.

- **`?` (try / propagation)** (CF-8) — a postfix control operator: `x?` yields the successful value, or
  exits the function early with a failure (`None`/`Err`), running the `defer`s (MEM-1). Sugar over `match`+return, without
  exceptions/unwinding; zero-cost, visible. Dispatched via the **comptime "tryable" protocol** (the form, not the names
  `Option`/`Result`); those merely implement the protocol in the prelude. Contract: the function's result (`-> T` or named `out`) is a compatible tryable type;
  failure conversion is via a **declared** conversion `OutErr(err)` applied by `?` (the `T(v)` lattice;
  declared-and-visible, not a hidden `From`, I8; auto-application off under `no_abstractions`). The converted
  failure is then **wrapped** into the enclosing tryable — `Option`/`Result` by their built-in `None`/`Err`, a
  **user** tryable by its **`from_failure`** constructor (the fourth `Tryable` operation, CF-10), the same-type
  case returning the operand verbatim.

- **Protocol (comptime form)** — a structural contract by which a language construct works with a type, **not**
  depending on its name: a condition requires *truthy* (`bool` and types providing `is_true`; numeric scalars are **not** truthy — write `n != 0`), `?` requires *tryable* (`Option`/`Result` and
  user-defined), `for`/iteration requires an *iterator* (a `next` yielding an *optional* shape — present/absent, no error payload; ranges/slices/arrays and user-defined),
  the comparison operators and `HashMap` keys require *eq*/*ord*/*hash* (derived structurally over `typeinfo`, overridable), and output targets a *writer* (a byte sink, whose `write` returns the writer's **own** tier-appropriate error — `IoError` for a std stream, `AllocError` for an in-memory buffer; never a protocol-fixed error, STD-2) — STD-1.
  Implemented via predicates/`resolves` (CT-4/CT-6); the prelude supplies the canonical types (CF-8).

- **Overload set / overloading by signature** (FN-7) — several **function values** bound to the **same name** in one
  scope, distinguished by their **parameter signature** (the ordered value-parameter types). A call resolves to the
  unique overload whose parameters accept the argument types (zero → unresolved, more than one → ambiguous; both are
  errors); each overload lowers to a **distinct signature-mangled symbol** with no runtime dispatch (I1/I2/I3). The one
  exception to one-binding-per-name (Declarations §6.2); lets a free-function protocol (the *iterator*'s `iter`/`next`,
  a type's `eq`/`hash`) supply per-type implementations under a shared name. A name MAY combine **one generic
  definition** with concrete overloads — the derive `generic-default + concrete-override` pattern — with resolution
  preferring an exactly-matching **concrete** overload and falling back to the **generic** (monomorphized) only on
  concrete-miss; otherwise a generic function is monomorphized, not overloaded. UFCS method dispatch (MOD-3) is the same
  resolution keyed on the receiver.

- **Receiver/access coercion — auto-ref / auto-deref** (MOD-3 / OP-5) — the one-level adjustment, between a value
  and a pointer to it, applied to the operand of a **value-access position**: a UFCS method receiver, a field
  `a.field`, an index `a[i]`. **Auto-ref** lifts a value to a scoped reference when the position wants
  `ptr([mut] R)` and the operand is `R` (`a` → `ptr(a)` — how a method borrows its receiver without
  consuming it). **Auto-deref**, its dual, lowers a pointer to its pointee when the position wants `R` and the
  operand is `ptr([mut] R)` (`p` → `deref(p)` — `p.field` ≡ `deref(p).field`, likewise for `[i]` and a
  method receiver). Resolved **exact match → auto-ref → auto-deref**, applied **exactly one level** (a
  pointer-to-pointer derefs once; further needs explicit `deref`, so deref depth is never hidden, I3). Both are
  **borrow/read** paths — neither moves or consumes an owning value. A type-directed postfix coercion (Type System
  §4.5): it borrows no operator glyph and adds no grammar.

- **Translation unit** — a separate source file. The carrier of the local limits contract (FND-11/FND-10).

- **Machine model** — facts about the target, fixed at configuration: bit widths, the storage model
  (registers, or an operand stack + locals on a structured/VM backend), endianness, available
  ISA/backend features. Input for the type system (CG-4).

- **`Machine` variant** (TOOL-18) — how a manifest states the platform: `Target.machine` is a variant of
  the `Machine` enum — `Linux(arch, env, startup)` / `Freebsd` / `Windows(arch, subsystem, startup)` /
  `Macos(arch, startup)` / `Android(arch, api, startup)` / `Bare(arch, env)` / `Com()` — each carrying **only** the fields that platform
  has, so an inapplicable field (a PE `subsystem` off PE, a `startup` on a freestanding target) cannot be
  written rather than being diagnosed. There is no `os`/`container`/`endian` field: those are **`target.*`
  projections** of the variant (`target.os`, `target.container`, `target.endian`), which is why the
  familiar gates (`when target.os == Os.none`) keep their spelling. The **supported targets** are
  `(variant, arch)` pairs (per-arch appendix §1); an additive non-ISA backend (WASM) arrives as a new
  variant with no container projection (CG-14). `code_size` and `arm_mode` stay on `Target` — they are
  arch-specific codegen knobs, not platform facts.

- **Backend** — the lowering target. **Register-ISA** backends emit GAS (`as`+`ld`); a **structured/VM**
  backend (e.g. WASM) emits its own form (WAT → `wasm`). The **portable core** (types, functions/ABI,
  structured control flow, data, comptime) lowers to every backend; raw assembly-correspondence
  (registers, raw transfer, `asm`, code-point labels) is a **target-capability**, present only where the
  backend supports it. Register-ISA + GAS are v1; other backends are additive (FND-6/CG-4).

- **Data block** (*data block*) — a layout of bytes/words of the target's native bit width; the final form
  into which any type is laid out.

- **`bitsN`** — a core primitive: a raw block of N bits without interpretation (native widths `bits8/16/32/64`).
  Operations only interpretation-independent (bitwise, shift/rotate, bit-equality, move, `bitcast`).
  The numeric types `uN`/`iN`/`fN` — interpretations of this block, defined in the prelude (TYP-2).

- **`uint(N)` / multiword value** (TYP-10) — `uint(N)` is the **prelude type-function** for fixed-width
  unsigned integers **wider** than native: for a comptime `N` a **positive multiple of 64**, an unsigned
  integer stored as a **multiword** value — `N/64` machine words, **little-endian** (word 0 least-significant).
  `u128 ≡ uint(128)`. A **library recipe**, not a kernel/builtin type (TYP-2): its `+`/`-`/`*`/`/`/`%` and
  comparison are library operator-functions over the words (ripple-carry / schoolbook / long-division /
  hi-to-lo unsigned compare), with **visible** cost (`N/64` words). This is the library value that makes a
  non-native width (`u64` on a 32-bit target, a `usize` decomposition) an ordinary value, never a backend
  register pair. Non-multiple-of-64 (partial-top-word) widths, sub-native widths, and signed `int(N)` are
  **additive** (deferred, TYP-10).

- **Type** — a recipe for laying out a value into data blocks. At the upper levels — a comptime value,
  computed by a function over the machine model.

- **Type identity** — the rule "when two types are one and the same". Preliminary decision: nominal
  by declaration; structural only for anonymous/derived layouts.

- **Branding** (*newtype*) — a comptime primitive minting a fresh nominal identity on top of
  a shared layout. wrap/unwrap explicit, zero-cost.

- **Pointer** — the type `ptr(T)` (immutable pointee) / `ptr(mut T)` (mutable); MEM-8. The pointer
  operations are flat prelude word-functions: `ptr(x)` (address of a place), `deref(p)` (dereference) — there is no `mem` module (MEM-7).

- **bitcast** (reinterpretation, reinterpret) — treating the same bits as another type. ~0
  instructions, changes the meaning; always explicit and "loud". Builtin (OP-1): `bitcast(T, x)` ≡ `x.bitcast(T)`.

- **Conversion** — the transition of a value between types with possible emission of instructions. Classes:
  *widen* (widening, lossless), *narrow* (truncation, lossy), *numeric* (domain change, e.g. int↔float
  by value). The syntax of an explicit conversion — calling the type as a function: `u64(x)`. A user type joins
  this form with a **conversion-constructor**: a function marked `@convert`, single source parameter, target
  return type, dispatched at the `T(v)` site by ordinary name resolution (Type System §4.6, TYP-6).

- **Lowering** (*lowering*) — transformation of an upper-level construct into a lower-level equivalent
  by a documented rule.

- **Observable behavior** — what a conforming implementation MUST preserve: results, traps, and the order
  of observable effects (`volatile`/`atomic`/IO). Optimization and placement may differ between
  implementations only where observable behavior does not (CG-5).

- **Semantically-preserving optimization** (CG-5) — a compiler transform that lowers **cost** without
  changing observable behavior; licensed by the absence of UB (I11), so the documented lowering rule is a
  cost **ceiling** (I1), not an exact prediction outside `no_abstractions`. Behavior-changing transforms
  (float fast-math, …) are **opt-in** and visible (CG-9), never default. Knobs: `no_abstractions` (byte-exact
  floor), `no_opt` (un-optimized), and the profile's non-semantic optimization level.

- **escape to assembly** — raw assembly is available **everywhere inline** (FND-9); there is no separate
  `snippet` construct (removed in CF-6). A function/named raw region without a frame/ABI/return = `@abi(naked) fn`
  (FN-5/CF-6; the entry form wherever no platform entry contract exists — `Os.none`,
  `Container.com`, and under `no_abstractions` — see **Entry point**).

- **Label** (*label*) — an **attribute `@label(name)`** naming a place in code (CF-4; not an introducer), **two kinds**:
  **code-point** (`@label(n) <instruction>`) — a `jmp` target; the name in value position = a **raw code address** =
  a pointer-width raw block (`bits64` on the target; `bitsN` — a meta-family, in source a concrete width;
  there is no separate type; jump tables `[bits64; N]`; a `jmp` target, NOT callable; not opaque — ordinary bitsN-operations;
  intra-activation); **structured** (`@label(n) loop/while/for/block`) — a target **only** for `break n`/`continue n`
  (`continue` — loop only), **not a value, without an address**. Code has no address-of operation (the name of a code entity
  is the address — code has no "value to load", unlike a data place → `ptr(T)`). **Taking the address and
  storing it — ordinary bits** (not unchecked); a **raw control-transfer** (`jmp`/branch, direct OR indirect)
  in **structured code** requires `unchecked` (it bypasses §9, lexically CG-6), under `no_abstractions` — native.
  **Scope — function-scope
  (C-style):** the name is unique and visible throughout the function (forward+backward), there is no block-shadowing of labels; a shared
  namespace with bindings (a label name ≠ a binding name in the function). The value of a labeled loop → into an ordinary binding.

- **Comptime value** — a value known and computed at compile time; `type` — one of
  such values.

- **Value introducer** (*introducer*) — surface syntax for writing a value-literal on the
  RHS of a declaration: `struct/enum/union{}` (type-literals), `fn(){}` (a function), `mod{}`
  (a module) (FN-1/SYN-6). An **ABI** value is **not** an introducer — it is an ordinary `Abi`
  struct value, built `Abi(...)` (FN-5/SYN-6). **Admission criterion (SYN-6):** an
  introducer is admitted **iff** it denotes a genuinely distinct *kind of value-literal*
  not expressible as an ordinary call/expression — never merely to shorten or annotate a
  function of a particular result type. Consequences: a **generic type** is an ordinary
  type-returning function `fn(…) -> type`, **not** an introducer (the removed `type(…)`
  introducer was sugar for it); a **predicate** `fn(K) bool` and a **lever/attribute**
  `fn(K) K` get their role from the **consumption slot** (`when`/`@`), not an introducer
  (CT-5, CT-10). `type` is a keyword for the **kind of types** (`T : type`), not an introducer.

— *Memory model vocabulary (MEM-6):*

- **Value** (*value*) — a typed datum; without identity, location, scope.

- **Place** (*place*, storage location) — the storage location of a value; has a type, storage class,
  mutability, lifetime. Addressable only if resident in memory.

- **Field projection** (*field projection*) — `a.(f)`: access the field of aggregate `a` named by a **comptime**
  value `f` (a `Field` from `typeinfo`, or its `str` name), resolved at comptime to the named access `a.<f.name>`
  — same place/type/permission, identical emitted code (TYP-9; Comptime §5.4). The form that lets a structural derive
  read the field it is iterating. Distinct from UFCS `a.f(args)` (which begins with an ident/path, not `(`).

- **Comptime variant pattern** (*comptime variant pattern*) — `T.(v)`: the **enum** dual of field projection, in
  **pattern** position. Inside a `comptime for v in typeinfo(T).variants` it resolves at comptime to the concrete
  variant pattern `T.<v.name>` (`v` a `Variant`, or its `str` name); `T.(v)(p)` binds the matched variant's **whole
  payload** as one value `p` (a tuple when multi-component). With `comptime for` admitted in **match-arm** position
  (one arm per variant ⇒ exhaustive), it lets a structural derive compare an enum value of generic `T` — the form that
  makes `eq`/`lt`/`hash` library code over `typeinfo` for **every** type (CT-9; Comptime §5.5).

- **Compound assignment** (*compound assignment*) — `place ⊕= expr`: sugar for `place = place ⊕ expr` with the
  place evaluated **once** (OP-2; Memory §1). `⊕` ranges over the binary glyph operators `+ - * / % & | ^`; it
  is a statement, not an expression, and inherits its overload from the operator-function `⊕` (no separate hook).

- **Binding** (*binding*) — the association of a name with a place; the carrier of scope.

- **Address** (*address*, pointer) — a pointer-width value designating a place-in-memory.

- **Storage class** — where a place lives: register / stack / static / allocator-managed
  (colloquially "heap"; the spec uses *allocator-managed*, Memory §2.1).

- **Extent** (*extent*) — how long a place is valid. Distinct from scope (of a name).

- **Scope** (*scope*) — where in the source a binding name is accessible.

- **place-expression / value-expression** — classification: a place yields a place (variable, field, index,
  dereference), a value yields a value (literal, arithmetic, by-value call, taking an address).

- **Mutability** — permission to write into a place, bound to the path to it (binding/field/pointer),
  not to the value. Immutable by default; `mut` — opt-in. Effective permission = AND over the whole path.

— *Lifetime (MEM-1):*

- **Lifetime mechanism** — the way of managing a place's validity. The **language** has **one**
  reference — the **scoped** (second-class) reference; the longer-lived mechanisms
  (region(arena) / generational / manual) are **library allocator providers** over one
  **allocator-reference** protocol (MEM-1), selected via `@alloc(value)` (STD-1; except
  scoped — does not allocate); the provider comes from the allocator value's type and hands
  out a first-class reference token (for a region, `Handle(T)`, a number), not a language
  reference family.

- **Region** (*region*, arena) — a named unit of allocation-and-lifetime; references into it
  do not outlive it; freed in bulk.

- **Ambient allocator** (*ambient*, `alloc::with`) — the allocator in effect for elided allocator
  arguments in a region (MEM-5). Established/overridden by the prelude form `alloc::with(ar) { … }`
  (a layer-1 convention; the outermost one, in `main`, is the program default — there is **no**
  manifest or process-static global). An elided allocator resolves to: nearest `alloc::with` → the
  enclosing function's own allocator parameter → a `§5.1` default → else a compile error. Always a
  **named binding in scope** (no reflective `current()` accessor).

- **Allocator parameter** — an `in` parameter of the library protocol **`alloc`** (a pointer to a
  mutable allocator, `in a : ptr(mut alloc)`; never `in out`, Functions §5.1/§2.2). Distinguished
  by type alone; within the body it **is** the ambient allocator, and at a call site it is **elidable**
  from the ambient (Functions §5.5). Shapes: **A** none / **B** required (escaping producers) / **C**
  B + a §5.1 default (scratch-only, escape-checked). MEM-4.

- **Handle / index** — a key value (a number) into an arena; not a pointer, hence without the lifetime problem;
  validity is checked on access against the owning arena. For long-lived links (graphs and the like). v1
  surface (MEM-3): the ordinary library type `Handle(T)` (a number, no region name; distinct per `T` because a
  type-function result is nominal by (function, args), Type System §4.1); `@alloc(a) x := init`
  binds it; the value is read by `get(a, h)` — an **ordinary function** with a `scoped` (second-class)
  return (MEM-3), bounds-checked → a scoped pointer — never by dereferencing the handle itself.

- **Second-class reference** (*scoped*) — a reference that cannot be stored beyond, or returned upward
  past, its scope/region; zero-cost, checked statically (the escape rule, Memory §5.3.1). May appear as a
  **parameter** (a scoped borrow the callee cannot leak) and, via the **`scoped` result qualifier**
  (MEM-3), as a **second-class return** — a result the *caller* may use at the call site but not store or
  re-escape (how `get` is an ordinary function, not a privileged intrinsic). The marker is local and
  unparameterized (it never binds the result's scope to a particular argument), so there are no lifetime
  variables (MEM-1).

- **`defer`** — explicit deferred cleanup: visible in the source, executed on exit from the scope (not hidden
  control flow, per I3). A replacement for implicit RAII-Drop.

- **Closure** — a function + captured environment. *Static* — a concrete type (an aggregate of captures +
  function), zero-cost, monomorphized. *Type-erased (dyn)* — a fat value (a code pointer +
  a pointer-to-environment), the environment in explicit storage (MEM-1), an indirect call. Capture by value by default.

- **Function-value type** — `fn(T0, T1, …) -> R` (FN-10): the **type** of a **non-capturing** function value
  (a top-level function or non-capturing lambda), written in **type position** as the **bodyless** signature
  (parameter **types** with optional directions `in`/`out`/`in out`, **no names**, **no** `{…}` block; `-> R`
  optional — its absence types a procedure). **Representation: one machine word — a code address**; a value of
  this type is bound / passed / returned / **stored** (struct/array/tuple field) and **called** `f(args)` (one
  indirect call, no environment, no allocation). Distinct from the `fn(…){…}` value-expression (which has a
  block) and from the generic *callable* constraint (a comptime predicate that also admits static closures).

- **`dyn` closure surface / `dyn_over`** — `dyn fn(T0, T1, …) -> R` (FN-11): the **type-erased** closure type
  — the `dyn` **keyword** prefixing a function-value type — a **two-word `{code, env}` fat pair** (a code
  pointer + an environment pointer) that lets **different capturing closures share one runtime type**. `dyn`
  is a **type-constructor keyword** (it switches the representation), not an `@`-attribute. The environment
  lives in **explicit user-provided storage** the `dyn` value **borrows** — never a hidden heap box (I3) — and
  its validity is bounded by that storage's extent, checked by the **escape rule** (Memory §5.3.1).
  **Construction:** the prelude word-function **`dyn_over(ptr(mut store))`** over a named place `store` holding
  a static closure (Memory §6.2). **Call:** `d(args)` — indirect through `code`, passing `env`. Visible cost:
  two words + one indirect call. Deliberately unlike a `Box<dyn>` (no hidden allocation).

- **Owning value** — a value-"ticket" behind a real resource (file/region/piece of heap).
  A type is owning iff it carries the **`@owning`** marker or is an aggregate transitively
  containing an owning field (contagion); a bare pointer/handle is **not** owning (MEM-2).
  Non-copyable; passed (move, the source is invalidated); a borrow via `ptr` does not consume.

- **Move / consume** (*move*) — transfer of ownership; the source becomes invalid. Only for
  owning values (for copyable ones — a copy). An owning value is consumed by being passed to
  an **`in`** parameter, returned, or used to fill an `out`; `in out` / `ptr` borrow,
  not consume (MEM-2).

- **Linearity (strict)** — an owning value must be consumed/passed exactly once on each
  path of normal exit (catches double-free, use-after-free and leaks). `defer` consumes on all
  exits. `forget(x)` — an explicit valve for a deliberate leak.

— *Configuration (CT-10):*

- **Lever** — a primitive of the **fixed core set** the codegen honors for **layout/representation**, each with a
  documented lowering rule (CT-10): `repr`/`align`/`packed`/`offset`/`endian`/`niche`. Closed (the programmer
  adds no new honored codegen behavior, I10); composed **openly** by comptime functions.
  `require` (the explicit checked constructor gate, TYP-12) is a **comptime-function
  attribute**, not a machine lever; `brand` (TYP-4) is an ordinary comptime
  type-constructor call, not an attribute or a lever.

- **Effector** — a comptime function that composes levers, applied to a construct as `@name(args)`
  (≡ `construct.name(args)`); prelude and library effectors are indistinguishable (CT-10, OP-1). `@` is
  compile-time and **never wraps an expression** (CG-6); runtime operation effects are ordinary builtin
  calls (`atomic::load`, CC-2), not `@`. A `fn(type) bool` effector-shape is a **predicate** (`when`/`comptime if`).

— *Stdlib types & analyses (terms used normatively across the spec):*

- **`Never`** (*bottom type*) — the named divergence type (CF-7): it has **no value**, so an
  expression of type `Never` only **diverges**. The result type of `panic`/`exit` and of any
  non-returning function (incl. an `@extern` one) — so no separate `@noreturn` is needed (OP-1).
  A `Never` result is a **flow terminator** (code after it is unreachable; Control Flow §9). It is
  a named type, not a keyword/symbol (CF-7/STD-1 — clarity over terseness).

- **`str` / `String`** — `str` is a **slice of UTF-8 bytes** (`[u8]`, Types §7 / Stdlib §3.6): a
  `{ptr, byte-length}` view; its byte length is **not** its code-point count, and indexing is by
  byte (an in-bounds byte read, traps out of range). `String` is the **alloc-tier growable owned**
  UTF-8 buffer (Stdlib §6) — the counterpart that *grows*; `str` is a borrowed view.

- **Range** — the value `a..b` (**half-open**, `a ≤ x < b`) or `a..=b` (**inclusive**); used in
  slicing (`s[lo..hi]`, open-ended `s[lo..]`/`s[..hi]`/`s[..]`) and `match` patterns (Control
  Flow §5.4). `..=` is what reaches a type's maximum (`0..=255`).

- **`embed`** — the comptime builtin `embed(comptime path : str) -> [u8; N]`: a **reproducible**
  compile-time file embed into a byte array (Comptime §2.4).

- **Link mode** (MOD-9) — the per-library property choosing how an external (C/system) library is
  linked: **`static`** (the default — the `.a` archive is **absorbed** into the binary) or
  **`dynamic`** (the `.so`/`.dll` is **referenced**, resolved by the OS loader at run time). A
  first-class manifest field (`Lib := struct { name : str, link : LinkMode = LinkMode.static }`,
  manifest appendix §3.5), **not** an opaque `linker_flags` string. The produced binary is **fully
  static** unless some linked library (directly or transitively) is `dynamic`, which makes the
  binary dynamic (Modules §7.5). Distinct from the output **`Kind`** (executable/static_lib/
  shared_lib), which is what the package *itself* produces, not how it *consumes* an external.

- **Startup mode** (TOOL-12) — the per-target choice of whether the platform's **start-up files**
  and **libc** are linked: **`raw`** (the default — none of them; the process entry is the symbol
  `Target.entry` names, and the library reaches the OS through `@abi(syscall)`) or **`libc`** (they
  are linked and **own** the process entry, so `entry` is not applicable and the program supplies the
  function start-up calls — `main` on ELF/Mach-O). Distinct from the triple's **`env`** (`gnu`/`musl`/
  `msvc`/…), which is an **ABI** parameter and *not* an instruction to link libc; and distinct from
  **hosted vs freestanding** (a property of `os`). `Os.none` admits only `raw`.

- **Entry point** (TOOL-12; FN-12) — the declaration a process starts in, named by `Target.entry` as
  an **Alatyr path** (default `"_start"`, the root module's), never as a raw linker symbol: the
  toolchain derives the symbol that declaration emits (Modules §6.1 / `@export("exact")`) and states
  it to the linker explicitly. Its frame is written either by hand — **`@abi(naked)`**, the applicable form
  on `Os.none`/`Container.com` and under `no_abstractions` — or by the compiler's **per-target entry
  prologue**, `@abi(entry)` (ABI appendix §3.3), which captures the platform's entry state into the
  documented `alatyr_entry_state` static (so `args`/`env` work), calls the body, and terminates the
  process with its result. Under `startup = libc` neither applies: the platform's start-up calls a
  function the program supplies by exact name (`main`, or the `subsystem` form on PE).

- **Manifest handle** (TOOL-3; TOOL-15) — the **binding name** of the package's single `Package`
  value (`mylib := Package(…)`), under which source reads the manifest's own declared fields as
  comptime data (`mylib.version`, `mylib.targets`; Tooling §2.7). It is an **ordinary root-module
  declaration**: non-`pub`, therefore visible to **every module of the package** by down-tree privacy
  and to nothing outside it — unlike `target.*`/`build.*`, which configuration publishes
  unconditionally. `pub` on it is a Config diagnostic (its type lives in the configuration prelude), so
  metadata is published by re-exporting fields. It emits **no symbol**, and it shares the root scope
  with `source_dir`'s child modules, so a same-named module is a duplicate-name error (Modules §5). It
  is also the **base of the artifact name** (see below) — there is no `Package.name`.

- **Artifact name** (TOOL-11) — the produced file's name: a **base** plus a **`kind`×`container`
  affix** (`app` → `app` / `app.exe`, `libapp.a` / `app.lib`, …). The base is the **manifest
  binding's name** (`app := Package(…)`) — there is no `Package.name` — or, for a manifest-less
  invocation, the root file's stem. An explicit `Target.output` is taken **verbatim** (no affix
  added) and is a file **name**, not a path; the location is `target_dir`, overridden per artifact by
  `-o`.

- **Hermetic build** (MOD-9) — a build that links **only `static`** libraries: **self-contained**
  (no runtime interpreter, no `.so` dependency) and **byte-for-byte reproducible** with the archive
  pinned by the lockfile (the reproducibility guarantee, Tooling §6.2 / TOOL-1, holds end to end).
  The **default** binary mode. A build with **any `dynamic`** library is **non-hermetic** — its
  runtime behavior depends on the host's installed library — a status **determinable from the
  manifest** that the toolchain **surfaces** (a Config note, Tooling §2.5), keeping the guarantee's
  scope explicit (I3).
- **Per-module compilation** (TOOL-6) — the build model where each **module** compiles independently to
  its own `.s`/`.o` in a per-module arena, then a deterministic link produces the artifact. Bounds compile
  memory to one module (not the whole tree), enables parallelism, and is the substrate for incrementality.
- **Module interface** (TOOL-6) — everything a *dependent* module can observe about a module: exported
  signatures, struct/enum **layout** (offsets/size/align — part of the ABI under `@packed`/`@offset`/
  `@align`/`@repr`), generic signatures + demanded instantiations, `@inline` bodies, and comptime-observable
  facts. Hashed as ONE **interface hash** — the unit of incremental invalidation.
- **Interface hash** (TOOL-6) — the single content hash of a module's interface. A module `M`'s cache key
  is `hash(M's source + the transitive interface hashes of M's dependencies)`; when it is unchanged, `M`'s
  cached `.o` is reused.
- **Output-neutral cache** (TOOL-6) — the `.cache/` content-addressable `.s`/`.o` store is a **local,
  disposable** speed artifact (gitignored, never shipped) whose hard invariant is that a **cached** build is
  **byte-identical** to a clean (`rm -rf .cache/`) build — the cache may only make a build *faster*, never
  change its *output*. Reproducibility comes from the lockfile + source (TOOL-1/MOD-7), orthogonally.

- **`uninit`** — the explicit opt-out for **uninitialized storage** (Types §9): in checked code a
  read of `uninit` storage is forbidden (definite-assignment); writing then reading is required.

- **`forget`** — the prelude builtin `forget(in v : T)`: **discharge a linearity obligation
  without releasing** the resource (a deliberate leak valve; Memory §5.9 / MEM-2). Consumes the
  value for the linearity checker without running its release.

- **Definite assignment** — the safety analysis that **every read of a binding is provably
  preceded by an assignment on all paths** (Control Flow §9; Types §9): reading uninitialized
  storage is ill-formed (unless via `uninit`). A `Never`-typed terminator makes following code
  unreachable for this analysis.

- **Escape rule (second-class flow discipline)** — the static analysis enforcing that a **scoped**
  reference does not outlive its scope (Memory §5.3.1; MEM-1/MEM-3): a `scoped ptr` (e.g. a `get`
  result, or a scoped parameter) may be dereferenced and passed further down, but **not** stored
  where it outlives the call, returned (unless the result is itself `scoped`), assigned to an
  `out`, or captured in an outliving closure. Local and unparameterized — no lifetime variable, no
  cross-function inference.

- **Data race** — two unsynchronized accesses to the same location with at least one a write
  (Concurrency §5). A data race has **hardware-defined** behavior, **never** UB (I11) — a torn or
  stale value, not "whatever suits the compiler"; race-freedom is obtained **by construction**
  (single-threaded by default; sharing only via the synchronized surface).

- **Memory ordering** — the `Ordering` comptime prelude enum (`relaxed`/`acquire`/`release`/…) that
  parameterizes an atomic operation (Concurrency §3): atomicity is a property of the **operation**,
  not the type. (The atomic surface beyond the v1 primitives is additive, CC-2.)

- **`v1` (the language version)** — the **first version of the language**: the feature set this
  document defines — the six architectures, the closed supported-target set, the enumerated
  prelude/stdlib and per-target ABIs. It is an **epoch of the language**, not a range of document
  revisions and not a SemVer constraint. Every `1.*.*` revision of this document specifies `v1`
  (the MAJOR digit *is* the language version); a later language version would be `v2`, which
  `I10`/`PRIN-2` make unlikely, since growth within `v1` is additive. Program validity is
  guaranteed across the whole of `1.*.*`; **conformance is not** — a conformance claim cites a
  revision, never `v1` (Overview §6, FND-13).

- **Revision** — a published version of *this document*, `MAJOR.MINOR.PATCH` (e.g. `1.1.0`). MAJOR
  is the language version above; MINOR marks a normative revision of the text (a gap closed, a
  collision resolved); PATCH is editorial. Distinct from **maturity** (`draft (under review)` /
  `accepted`), which is carried by each chapter's Status line and never by the number.
