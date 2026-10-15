# ADR-0016: Compute gamma with a checkpointed forward backward pass

- **Status** **Proposed by this review**, not implemented
- **Date reconstructed** 15 October 2026
- **Confidence** proposal, grounded in `../../review/MEMORY.md` T1.3

## Context

`gamma` needs the forward and the backward array at the same time step, and they are produced in
opposite directions. The code resolves this by storing both in full and then adding and
exponentiating them, `+inference/hsmmresidual_decodelog.m:86-87`. That is three to four copies of
147 MB per block at the reference workload, to produce a 0.74 MB result. It is also why the two
`dec` flags exist, and therefore why `../../review/ISSUES.md H1` is possible at all.

## Decision

Store the forward array only every `K` steps. During the backward pass, recompute each `K` long
block forward from its checkpoint and consume it immediately. With `K` of order the square root of
`T`, storage falls from `m` times `dmax` times `T` to about `m` times `dmax` times the square root
of `T`, which is 147 MB down to about 1.2 MB, at the cost of roughly twice the forward arithmetic.
Remove `opt.decalpha` and `opt.decbeta`, since nothing then depends on retaining the arrays.

## Consequences

- The largest single memory reduction available without changing the model, and it deletes the
  configuration trap behind `H1`.
- About twice the forward arithmetic. For a memory bound job that is the right trade, but measure it
  before accepting, `MEMORY.md` section 7 item 5.
- The implementation should normalise `gamma` per sample while it is in there, which fixes
  `../../review/ISSUES.md M5` in the same edit.
- Requires `../../review/VALIDATION.md` tier 2 to exist first. This is the change most likely to
  introduce a subtle error, and the brute force test is what makes it safe.

## Alternatives not taken

- **Single precision for the stored arrays**: a factor of two, for a loss of accuracy in the bound.
- **A segment based recursion**, the alternative in `ADR-0003`, which needs no checkpointing at all
  but is a rewrite of the E step rather than a change to it.
