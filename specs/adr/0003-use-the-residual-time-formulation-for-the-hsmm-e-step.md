# ADR-0003: Use the residual time formulation for the HSMM E step

- **Status** Accepted, reconstructed from the code
- **Date reconstructed** 15 October 2026
- **Confidence** read

## Context

An explicit duration model breaks the Markov property over states alone, so the standard forward
backward recursion does not apply. The two usual remedies are to expand the state space with a
duration counter, or to write a segment based recursion that sums over segment start points.

## Decision

Expand the state space to the pair of state and samples remaining, giving an `m` by `dmax` lattice
with an ordinary Markov recursion over it, `+inference/hsmmresidual_decodelog.m:30-80`. Two moves
per step: shift the remaining duration down, or leave and draw a fresh duration.

## Consequences

- One recursion, close in shape to the familiar HMM one, and easy to check by eye.
- Cost is order `T` times `m` times `dmax` in time and, as written, in memory. Every working array
  gains a `dmax` axis. That is the structural cause of the memory profile.
- The segment based alternative has the same time complexity but needs only order `T` times `m`
  storage for the forward quantities, at the price of a less obvious recursion.
- The treatment of the two ends of the series becomes implicit rather than explicit, and the choice
  at `decodelog:28` matches neither standard convention. That is open question 2 in
  `../../review/ASSESSMENT.md`.

## Alternatives not taken

- **Segment based forward backward**, summing over start points, which is the formulation in most of
  the HSMM literature and is cheaper in memory.
- **An approximate HMM with duration dependent self transitions**, cheaper and not the same
  model.
