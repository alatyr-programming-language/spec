## What this changes

<!-- One or two sentences. For a normative change, state the rule before and after. -->

## Kind

- [ ] Editorial (typo, wording, cross-reference, example) — no implementation can observe it
- [ ] Normative — an implementation could behave differently because of this
- [ ] Design decision recorded (a decision entry added or refined)

## Related issue

<!-- Anything beyond a typo should have been discussed first: Closes #… -->

## Checklist

- [ ] **One decision = one commit** — every affected source of truth is updated in the
      same commit: `design/invariants.md` / `spec/` / `design/decisions/` /
      `design/glossary.md` / `design/aspects-map.md`.
- [ ] No invariant `I1–I11` is violated — in particular I1 (transparent codegen),
      I2 (zero-cost), I3 (nothing hidden), I11 (no UB).
- [ ] Cross-references resolve: cited sections exist and say what the text claims, and
      cited decision IDs are the right ones — not merely real ones.
- [ ] Decision IDs are **append-only**: no existing ID reused, shifted, or repurposed; a
      superseded decision got a new entry, and `provenance:` lines are current.
- [ ] Terminology matches `design/glossary.md`; new load-bearing terms were added to it.
- [ ] Written to the **`FND-3` bar** — an implementer needs no further guess.
- [ ] English only; ASCII letters in identifiers, labels, and enumerations.
- [ ] `CHANGELOG.md` updated under **Unreleased** if the change is normative.
- [ ] No implementation code, build, or tests were added to this repository.
