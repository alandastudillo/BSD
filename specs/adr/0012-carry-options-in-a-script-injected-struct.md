# ADR-0012: Carry options in a script injected struct

- **Status** Accepted, reconstructed from the code. **This review recommends superseding it**
- **Date reconstructed** 15 October 2026
- **Confidence** read

## Context

Twenty five options must reach four decoders, two Viterbi routines and two orchestration functions,
with user overrides at the call site.

## Decision

`program_option.m` is a script, not a function. Each consumer runs it in its own scope to obtain
`opt`, then applies name and value overrides with `setfield`. Six files do this: `train.m:19`,
`trainn.m:6`, and the four decoders and Viterbi routines.

## Consequences

- Minimal plumbing, and overriding anything is a one liner.
- There is no single place where the effective configuration of a run exists, and no validation of
  any option value.
- A default changed in one file silently broke a consumer two layers away, which is exactly
  `../../review/ISSUES.md H1`.
- The GUI writes one option under a misspelled name and nothing notices,
  `../../review/ISSUES.md M6`.
- Undocumented options exist that only one branch reads, such as `opt.seqs`.
- A port to any other language cannot reproduce this pattern, so it has to be redesigned anyway,
  `../../review/PORT_PYTHON.md` section 4.1.

## Alternatives not taken

- **A validated options function** returning a struct, with an assertion per field. A small change
  that removes a class of defect.
- **An options class** with properties and set methods, which MATLAB supports and which would give
  the GUI type checking for free.
