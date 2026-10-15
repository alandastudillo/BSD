# ADR-0002: Model state duration explicitly, as a hidden semi Markov model

- **Status** Accepted, reconstructed from the code
- **Date reconstructed** 15 October 2026
- **Confidence** read, and documented in the manual section 1.3

## Context

In a hidden Markov model the time spent in a state is geometrically distributed, with its mode at
one sample. Brain states are believed to persist for tens to hundreds of milliseconds, so the
geometric assumption both misfits the data and biases the segmentation toward rapid switching.

## Decision

Give every state its own explicit duration distribution, and forbid self transitions so that
duration is counted once. `hsmm.m` carries a fourth sub-model, `dur_model`, and defaults the
transition model to the zero diagonal variant, `hsmm.m:35`. The plain HMM remains available as
`hmm.m`.

## Consequences

- The distinctive capability of the toolbox, and the reason it exists.
- The hidden state becomes the pair of state and remaining duration, which forces `ADR-0003` and
  `ADR-0007`, and through them the memory profile in `../../review/MEMORY.md`.
- Durations become interpretable and directly reportable, which is what the domain wants.
- The zero diagonal has to be maintained consistently in the update, the expectation and the KL. It
  is: `+trans_model/categ_dirichlet_matrixdiag0.m:64`, `:72`, `:105-111`.

## Alternatives not taken

- **A hierarchical HMM** or a sticky HDP-HMM would lengthen dwell times without an explicit duration
  distribution, at a cost in interpretability.
- **Keeping the HMM and smoothing** the state sequence afterwards would be cheaper and would not be
  a model of duration at all.
