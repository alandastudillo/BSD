# ADR-0017: Add explicit seed control

- **Status** **Proposed by this review**, not implemented
- **Date reconstructed** 15 October 2026
- **Confidence** proposal, grounded in `../../review/ISSUES.md M15`

## Context

Nothing in the toolbox calls `rng`. The default initialisation is random, k means adds ten random
replicates, and several sub-models sample parameters. Two identical calls therefore give different
answers, no published figure can be regenerated, and three whole tiers of the validation plan,
synthetic recovery, model order calibration and golden masters, cannot be written at all.

## Decision

Add `opt.seed`, empty by default, and `if ~isempty(opt.seed), rng(opt.seed); end` at the top of
`+inference/trainn.m`. Record the seed in `out`. Keep `initoption='seqprior'` as the stronger
guarantee for golden masters, since it fixes the starting sequence itself rather than the stream
that generates it.

## Consequences

- Two lines, and it unblocks the test harness.
- Results become reproducible, which is a publication requirement rather than a convenience.
- Seeding inside `parfor` needs care: each worker has its own stream, so the seed must be combined
  with the restart index to keep restarts both distinct and reproducible.
- An empty default preserves current behaviour exactly, so nothing breaks.

## Alternatives not taken

- **Seed at the top level only**, which does not survive `parfor`.
- **Always seed with a fixed value**, which would silently remove the benefit of multiple
  restarts.
