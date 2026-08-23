# Decisions — the *why* of Alatyr

This folder answers **WHY the language is the way it is**: the design decisions, their
rationale, and the alternatives rejected. The companion layers:

- **`spec/`** answers **WHAT** the language is — the normative behavior, semantics, syntax.
- **the compiler** answers **HOW** it is implemented.
- **git** holds the **history** — these files carry the *current* rationale, not its chronology.

Above the decisions sits **`../invariants.md`** (I1–I11) — the **supreme norms**; and
**`principles.md`** — the cross-cutting **priorities** (minimalism, additivity, the
closed-package reading of *no magic*, transparency↔ergonomics, layered conventions) that
resolve design forks. Terms are defined once in **`../glossary.md`**.

## How this is organized

- **One file per theme** (mirroring the spec's chapters). An entry is a thematic unit of
  rationale, not a historical fork — its granularity is chosen for clarity.
- **Stable, theme-prefixed IDs** — `MEM-4`, `TYP-1`, … — the citation anchor.
- Each entry carries a **`provenance:`** line (what it refines), a **`spec:`** pointer
  (the normative *what*), and a `**Why.**` / `**Rejected.**` body.

### Conventions

- **IDs are append-only**, assigned in *articulation order* per theme; an existing ID is
  never reused or shifted. IDs are identity, not order — a file may present entries in a
  different (logical) order.
- The **`provenance:`** line in each entry is the **authoritative** old→new map; the index
  below is a derived reverse-lookup.

## Themes

| File | Theme | IDs |
|------|-------|-----|
| `principles.md` | Principles & priorities (cross-cutting) | PRIN-1..7 |
| `foundations.md` | Methodology, conformance, limits, niche | FND-1..12 |
| `memory.md` | Memory, lifetime, ownership, allocation | MEM-1..8 |
| `types.md` | Types, layout, branding, construction | TYP-1..13 |
| `functions-abi.md` | Functions, declarations, ABI | FN-1..12 |
| `comptime.md` | Comptime, generics, protocols, introspection | CT-1..12 |
| `operators.md` | Operators, indexing, member access, conversions | OP-1..6 |
| `control-flow.md` | Control flow, labels, errors (`?`/`!`, Tryable) | CF-1..10 |
| `modules.md` | Modules, visibility, packages, paths | MOD-1..14 |
| `codegen.md` | Codegen, lowering, optimization, backends, conventions | CG-1..14 |
| `syntax.md` | Lexis, separators, declaration/expression grammar, `@` | SYN-1..7 |
| `tooling.md` | Tooling, manifest, packages, tests | TOOL-1..20 |
| `stdlib.md` | Prelude/stdlib definitions, streams, process inputs | STD-1..3 |
| `concurrency.md` | Concurrency | CC-1..7 |
