# ADR-0008: Use the free energy for convergence, restarts and model order, searched greedily

- **Status** Accepted, reconstructed from the code
- **Date reconstructed** 15 October 2026
- **Confidence** read. The search range is documented in the manual table 6.1 item 9; the stopping rule is not

## Context

Variational Bayes provides a lower bound on the model evidence. A bound that is comparable across
models is, in principle, exactly what is needed to choose the number of states, with no separate
information criterion and no cross validation.

## Decision

One quantity, three jobs: stop iterating when it improves by less than `opt.tol`; keep the restart
with the highest value; and search the state count upward from `minstates`, stopping at the first
candidate whose best value is lower than its predecessor, `+inference/train.m:96-99`.

## Consequences

- Economical and coherent, with no extra machinery to tune.
- All three jobs fail together if the bound is wrong, and there are four reasons to think it is: a
  wrong Gaussian constant, `M1`; a decrease masked rather than reported, `M3`; the duration
  truncation mismatch, `M9`; and the loss of the complexity penalty when a state is pinned to its
  prior, `H6`.
- The greedy stop cannot see past a single dip, so a free energy curve that dips at four states
  returns three.
- `out.fe` is the last iteration value rather than the best, so the comparison across restarts is
  not quite a comparison of best fits, `M3`.
- No external criterion exists, so there is currently nothing to check the bound against.
  `../../review/VALIDATION.md` tier 6 proposes held out predictive likelihood for exactly this.

## Alternatives not taken

- **Fit the whole range and take the maximum**, which costs more compute and removes the greedy
  failure mode.
- **Held out predictive likelihood**, independent of whether the bound is correct.
- **An information criterion**, cheap and poorly justified for a latent variable model of this
  kind.
