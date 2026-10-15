# ADR-0013: Reset a weakly supported state to its prior rather than letting it collapse

- **Status** Accepted, reconstructed from the code
- **Date reconstructed** 15 October 2026
- **Confidence** read

## Context

A state that attracts almost no responsibility produces a singular scatter matrix, and inverting it
either errors or yields an absurd precision. The degenerate case has to be handled somehow.

## Decision

If the expected occupancy falls below a threshold, copy the prior into the posterior for that state.
The threshold is hard coded at 15 samples in the emission models,
`+emis_model/normal_normal_wishart.m:143`, and at 0.001 in the duration model,
`+dur_model/normal_normal_gamma.m:153`. Earlier values, including a percentage version, are
commented out immediately above.

## Consequences

- Numerically robust: no singular inverses, and the fit continues.
- A state pinned to its prior has a KL divergence of exactly zero, so it contributes **no complexity
  cost** to the free energy. That removes the mechanism by which the free energy is supposed to
  penalise surplus states, which is `../../review/ISSUES.md H6` and which undermines `ADR-0008`.
- The threshold is an absolute sample count, so identical data at twice the sampling rate behaves
  differently.
- The two thresholds are inconsistent between sub-models by four orders of magnitude.
- Nothing records that a state died: no warning, and no field in `out`.

## Alternatives not taken

- **A relative threshold**, a fraction of the record length, which at least scales.
- **Let the prior do the regularising** and drop the special case, which is the usual variational
  Bayes answer: a strong enough prior makes the update well posed at any occupancy.
- **Prune the state and refit** with fewer states, which is explicit and reportable.
