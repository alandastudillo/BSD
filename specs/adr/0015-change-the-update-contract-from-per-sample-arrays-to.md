# ADR-0015: Change the update contract from per sample arrays to sufficient statistics

- **Status** **Proposed by this review**, not implemented
- **Date reconstructed** 15 October 2026
- **Confidence** proposal, grounded in `../../review/MEMORY.md` section 4

## Context

Every M step consumes aggregates: occupancy, weighted first and second moments, summed transition
counts, duration counts. At six states, ten dimensions and `dmax` of 200, all four sets of
statistics together are under 100 KB. The current `update` signature instead takes the full
responsibility matrix and the full transition array, and `+util/outmatr.m` concatenates every block
copy before calling it. That contract is why memory scales with the record length and with the
number of participants, and why nothing can be released until `train` returns.

## Decision

Change `update` on the live sub-models to accept an accumulated statistics struct, and have the
decoders emit it. The study loop then streams: decode a block, accumulate into the group statistics,
discard the per sample arrays. Keep the full arrays available behind a flag, for plotting and
inspection.

## Consequences

- Study memory becomes independent of the number of participants, and the retention in
  `+inference/train.m:53-109` becomes a non problem.
- **The model does not change.** The same posterior and the same free energy, so the change can be
  gated on golden masters, `../../review/VALIDATION.md` tier 7, rather than on a new scientific
  argument.
- It touches the public interface of every sub-model, so every class, including the MAR variants,
  must be migrated or deliberately left behind. That is the main cost: 8 to 12 days in
  `../../review/MEMORY.md` T1.1.
- The GUI per iteration plotting reads `decodevar.gamma`, so the flag is required, not optional.

## Alternatives not taken

- **Leave the contract and prune the retention only**, which is `MEMORY.md` tier 0 and gets most of
  the win for a fifth of the cost. Do that first regardless.
- **Out of core statistics**, which solves a different problem: a single block too large for
  memory.
