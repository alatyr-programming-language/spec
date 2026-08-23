# Changelog

This specification is **living and versioned**: it does not stop once a compiler exists.
Implementations cite a version, so every change that an implementer could notice is
recorded here.

## Versioning policy

The version string is `MAJOR.MINOR.PATCH`, but the three digits do **not** all describe
the document. They describe two different things, and keeping them apart is the whole
point of the scheme:

- **MAJOR is the version of the language**, not of the text. It is `1`: this document
  specifies Alatyr v1. It moves only if a program valid under the previous language
  version stops being valid — which `I10` and `PRIN-2` forbid, since growth within v1 is
  additive by construction. It is therefore expected to stay `1`.
- **MINOR is a normative revision of the document** — a gap closed, a collision resolved,
  a rule made explicit, an entry added to an enumeration. An implementer must re-read.
  Such a change can alter what an implementation must *do* without invalidating any
  existing program; that is exactly the additive growth `I10` permits.
- **PATCH is editorial only** — typos, wording, cross-references, added examples. No
  conforming implementation can observe the difference.

This is deliberately not plain library SemVer. A specification has no "0.x, anything may
change" phase in the SemVer sense: the language is specified in full, and what is in
motion is the precision of the text, not the shape of the language. Pinning MAJOR to the
language version is what keeps the number meaningful instead of making every substantive
correction look like a breaking release.

**Pre-release.** Until the document is *accepted* it carries an ordered `-draft.N`
counter: `1.0.0-draft.1` < `1.0.0-draft.2` < `1.0.0`. While the suffix is present the
document is under review — the MINOR/PATCH split above is not yet promised between
draft revisions, and any rule may still change. Every chapter carries
`Status — draft (under review)`; the suffix and that line are the same fact, stated in
the two places a reader looks.

Each entry names the affected sources of truth, because a normative change touches
several at once (see `CONTRIBUTING.md`).

## Unreleased

Nothing yet.

## 1.0.0-draft.1

First public release of the complete first draft. The whole document is **under review**
and not yet accepted.

**Specification (`spec/`)** — all 14 narrative chapters, `00-overview` through
`130-grammar`, plus 4 appendices: `140-appendix-manifest` (manifest schema),
`150-appendix-abi` (ABI per target), `160-appendix-stdlib` (prelude and standard
library), `170-appendix-arch` (per-architecture tables for all six v1 architectures and
the closed supported-target set).

**Design record (`design/`)** — invariants `I1–I11`; the decisions organized by theme
under stable theme-prefixed IDs (`MEM-`, `TYP-`, `FN-`, …), indexed by
`design/decisions/README.md`; the glossary; the aspect map. Every open question from the
design phase is closed.

**Implementation** — none of this text has been validated against a toolchain. The
reference compiler and the standard library are in development in the sibling
`compiler` and `stdlib` repositories; their feedback will arrive as refinements here.
