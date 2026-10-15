# ADR-0009: Initialise from a hard state sequence, with random restarts

- **Status** Accepted, reconstructed from the code
- **Date reconstructed** 15 October 2026
- **Confidence** read, partly documented in the manual table 6.1 items 2 to 4

## Context

Mean field variational Bayes on a latent sequence model has many local optima and needs a starting
point for either the responsibilities or the parameters.

## Decision

Start from a hard sequence, convert it to sufficient statistics with `+util/decodevarinit.m`, and run
the M step first. Two generators: random segments, the default, and k means on the samples ignoring
time. Repeat `opt.maxitersim` times and keep the best by free energy.

## Consequences

- Simple, and it reuses the same statistics contract as a real E step.
- The random generator ties segment lengths to `dmax`, so the initial duration posterior is centred
  near a fifth of `dmax` regardless of the data, `+util/unirndseq.m:3`.
- k means ignores time entirely, so its label runs can exceed `dmax`, which stops the first M step,
  `../../review/ISSUES.md H5`.
- The retry loop that guarantees every state appears can never terminate for short records,
  `../../review/ISSUES.md M13`.
- With no seed control, restarts are not reproducible, `ADR-0017`.
- `+util/decodevarinit.m` never counts the first segment duration, a small initialisation bias,
  `../../review/ISSUES.md L7`.

## Alternatives not taken

- **Sample parameters from the prior** and start with an E step, which the code supports through
  `parsamplfun(2)` and does not use.
- **A single deterministic initialisation**, cheaper and more reproducible, worse at escaping local
  optima.
