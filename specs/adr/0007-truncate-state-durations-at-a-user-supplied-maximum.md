# ADR-0007: Truncate state durations at a user supplied maximum

- **Status** Accepted, reconstructed from the code
- **Date reconstructed** 15 October 2026
- **Confidence** read, and `dmax` is documented in the manual table 6.1 item 8

## Context

The residual time lattice of `ADR-0003` must be finite, so some maximum duration is
unavoidable.

## Decision

A single option, `opt.dmax`, in samples, default 200. It bounds the lattice, and the duration
distribution is evaluated and renormalised on the grid 1 to `dmax`.

## Consequences

- The lattice is finite and the recursion is simple.
- `dmax` acquires two further roles that a user reading "maximum state duration" will not expect: it
  centres the data based duration prior at half of `dmax`,
  `+dur_model/normal_normal_gamma.m:187-188`, and it sets the segment lengths of the random
  initialisation, `+util/unirndseq.m:3`. Results therefore move with it through three channels,
  `../../review/ISSUES.md M13`.
- Every `dmax` slot is carried whether or not the duration posterior has mass there, a three to ten
  times memory multiplier, `../../review/MEMORY.md` T1.2.
- Any true duration longer than `dmax` is silently represented as `dmax`.

## Alternatives not taken

- **Separate the three roles** into a lattice bound, a prior mean and an initialisation scale, with
  the current single value as the default for all three. Cheap, and it removes the surprise.
- **Adaptive truncation** from the current posterior, which is what `MEMORY.md` T1.2 proposes.
