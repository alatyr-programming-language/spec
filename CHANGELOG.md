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
  existing program; that is exactly the additive growth `I10` permits. During review this
  is the digit that moves.
- **PATCH is editorial only** — typos, wording, cross-references, added examples. No
  conforming implementation can observe the difference.

This is deliberately not plain library SemVer. A specification has no "0.x, anything may
change" phase in the SemVer sense: the language is specified in full, and what is in
motion is the precision of the text, not the shape of the language. Pinning MAJOR to the
language version is what keeps the number meaningful instead of making every substantive
correction look like a breaking release.

**Maturity is a separate axis, and it is not in the version string.** Whether the
document is *accepted* is stated where a reader actually meets it: every chapter carries
`Status — draft (under review)`, and the Status block in `README.md` and
`spec/00-overview.md` says the same. Acceptance is recorded by flipping those lines and
by an entry here — not by a version bump, and not by a `-draft` suffix. A suffix would
say a third time what the Status line already says, and would freeze MINOR/PATCH during
exactly the phase in which the most revisions happen.

So `1.0.0` does **not** claim the text is settled. It says: revision 1.0.0 of the
specification of Alatyr v1. Read the Status line for how far along it is.

**Revisions, not commits.** The number moves when a revision is *published*, not when a
commit lands. Entries accumulate under *Unreleased* and are released together; each
release is tagged `v<version>` in git, which is what makes a citation resolve to exact
text. The procedure is in `CONTRIBUTING.md`.

Each entry names the affected sources of truth, because a normative change touches
several at once (see `CONTRIBUTING.md`).

## Unreleased

**Normative — the CLI gains `help`, `version`, and a verbosity contract.** Tooling §4 specified
the command set, discovery, target and profile selection, and artifact placement, but said
nothing about how a toolchain reports on *itself*. Two conforming implementations could
therefore disagree on whether `alatyr --version` works at all, and nothing stopped a
`--verbose` flag from changing the artifact it was supposed to be merely describing.

**TOOL-21** makes `help` and `version` introspection commands: available as both a command and
a flag, forbidden to read a manifest, discover a package, resolve dependencies or produce an
artifact, hence valid from any directory; written to stdout with a successful exit; `version`
must report the **revision of this specification** the implementation conforms to, since an
implementation's own version and the revision are separate axes (Overview §6). A bare invocation with no
arguments reports the same material on **stderr** with a **failure** exit — help asked for is an
answer, the same text volunteered after a mistake is a diagnostic — so a script that lost its
arguments still fails. What either prints is quality-of-implementation.

**TOOL-22** makes output verbosity an observation rather than a build parameter: such a flag
must not change the artifact, the diagnostics, the accept/reject outcome or the exit status,
and the same input must build byte-identically with and without it (§6.2). This is stricter
than the debug-only budget/limit overrides, whose effect a package's success merely must not
depend on.

**TOOL-14** is clarified: an unrecognised command or flag is an invocation-level Config
diagnostic naming the offending argument, and an implementation must not silently reinterpret
an argument it does not recognise as a source path — which reports a fault in a file the user
never named and hides a mistyped flag behind a file-system error.

§7 conformance item 4 now carries all three.

**Normative — conformance cites a revision, not the language version.** Overview §6 now
states that `v1` (the language) and `1.0.0`, `1.1.0`, … (revisions of this document) are
separate axes: program validity holds across the whole major (I10), conformance does not,
because a MINOR revision may pin behavior an earlier one left unstated. "Conforms to `v1`"
is therefore not a well-formed claim.

`v1` was load-bearing throughout the text — including inside the conformance rules — and
defined nowhere. It now has a glossary entry, alongside **revision**, and a decision entry
(FND-13).

## 1.0.0

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
reference compiler, which carries the standard library with it, is in development in the
sibling `compiler` repository; its feedback will arrive as refinements here.
