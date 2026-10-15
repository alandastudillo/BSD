# ADR-0010: Express group level parameter tying as three element share vectors

- **Status** Accepted, reconstructed from the code
- **Date reconstructed** 15 October 2026
- **Confidence** read, and documented in the manual section 3.2

## Context

A group study needs to decide, per model component, whether parameters are shared across conditions,
subjects and blocks. That is eight combinations per component across four components, so some
configuration language is needed.

## Decision

One three element logical vector per component, over condition, subject and block:
`opt.shareemis`, `sharetrans`, `sharein`, `sharedur`. `+util/groupindex.m` turns them into groups of
blocks, and the M step updates one model per group from the pooled statistics.

## Consequences

- Very expressive for very little code, and the toolbox most distinctive capability.
- The defaults encode a scientific position the manual does not justify: states are subject specific
  and shared across conditions, while the initial distribution is condition specific,
  `program_option.m:21-24`. See `../BUSINESS_RULES.md` BR-017.
- The implementation reaches into the nested study struct with `eval` built field access,
  `+util/inmatr.m` and `outmatr.m`, which is slow and undebuggable, `../../review/ISSUES.md D3`.
- Group statistics are formed by concatenating every member block per sample arrays, which is what
  makes study memory scale with the number of participants, `../../review/MEMORY.md` section 4.
- `+util/groupindex.m:9` versus `:12` contains a latent index bug that is currently masked,
  `../../review/ISSUES.md D5`.

## Alternatives not taken

- **A hierarchical prior** over subjects, the statistically principled version of partial sharing,
  and considerably more work.
- **Fit each subject separately and match states afterwards**, which is what many groups do, and
  which needs a matcher the toolbox does not have, `../../review/ISSUES.md L10`.
