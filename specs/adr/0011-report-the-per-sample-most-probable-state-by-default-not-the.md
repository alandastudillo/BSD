# ADR-0011: Report the per sample most probable state by default, not the Viterbi path

- **Status** Accepted, reconstructed from the code
- **Date reconstructed** 15 October 2026
- **Confidence** read

## Context

Users want one label per sample. Two different quantities can supply it: the marginal most probable
state at each sample, or the single most probable whole sequence.

## Decision

`opt.seq='maxgamma'` is the default, the per sample argmax of `gamma`,
`+inference/train.m:118-122`. Viterbi is available as `opt.seq='viterbi'`.

## Consequences

- Cheap: the quantity is already computed by the E step, so the default path costs nothing extra.
- The result is not necessarily a sequence the model can generate. A one sample visit can appear
  even when the duration model gives it no mass, so the reported segmentation can contradict the
  fitted durations. Not documented anywhere.
- It also drives `out.nstates`, defined as the count of distinct winning labels, which is the cause
  of `../../review/ISSUES.md H4`.
- The Viterbi alternative is correct in principle and currently impractical on real recordings,
  `../../review/ISSUES.md H3`, so users are effectively steered to the default.

## Alternatives not taken

- **Viterbi by default**, principled and, as implemented, unaffordable.
- **A duration aware smoothing of the marginals**, which would remove the contradiction cheaply.
