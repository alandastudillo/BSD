# ADR-0004: Work in the log domain, with per step rescaling

- **Status** Accepted, reconstructed from the code, superseding an earlier linear implementation
- **Date reconstructed** 15 October 2026
- **Confidence** read

## Context

Forward quantities on a recording of tens of thousands of samples underflow double precision within
a few hundred steps. Both a linear implementation with scaling and a log domain implementation
exist in the repository, and the linear one is the older.

## Decision

Decode in logs, and additionally rescale at each step by the total mass,
`+inference/hsmmresidual_decodelog.m:34-35`. `hsmm.decode` points at the log version, `hsmm.m:87`,
with the linear version retained but unused, `hsmmresidual_decode.m`.

## Consequences

- No underflow, and the log likelihood falls out as the sum of the log scale factors.
- Belt and braces: rescaling inside the log domain is redundant for range, harmless, and convenient
  for accumulating the likelihood.
- The final `gamma` assembly leaves the log domain and exponentiates a full `m` by `dmax` by `T`
  array, `decodelog:86-87`, which gives back part of the benefit and is the memory hot spot.
- Four `real()` calls were added at `decodelog:105-108` when complex values began appearing, which
  masks rather than fixes whatever produced them. The abandoned `logsubexp` based `gamma` recursion
  at `:75-79` is the likely source.
- Two implementations of the same algorithm now exist and only one is live.

## Alternatives not taken

- **Linear with scaling only**, which is what `hsmmresidual_decode.m` does and which is adequate in
  practice. It was presumably abandoned for a reason the repository does not record.
- **Extended exponent arithmetic**, more intrusive and unnecessary here.
