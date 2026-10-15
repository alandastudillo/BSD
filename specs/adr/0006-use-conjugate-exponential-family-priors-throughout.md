# ADR-0006: Use conjugate exponential family priors throughout

- **Status** Accepted, reconstructed from the code
- **Date reconstructed** 15 October 2026
- **Confidence** read, and documented in the manual figure 1.2

## Context

Variational Bayes is tractable in closed form when each prior is conjugate to its likelihood. The
alternative is numerical optimisation inside every update.

## Decision

Normal Wishart for full covariance Gaussian emissions, Normal Gamma for diagonal ones and for the
MAR coefficients, Normal Gamma for the Gaussian and lognormal duration parameters, and Dirichlet
for the transition rows and the initial distribution.

## Consequences

- Every M step is a closed form parameter update, which is fast and stable.
- Every KL term has a closed form, supplied by SPM and Beal derived helpers in `+util`.
- The duration model is the awkward case: a Gaussian is conjugate, but it is a continuous
  distribution used for a positive integer quantity, which forces the truncation renormalisation at
  `+dur_model/normal_normal_gamma.m:90-92` and breaks the very conjugacy that motivated the choice.
  That mismatch is `../../review/ISSUES.md M9`.
- The lognormal variant mitigates the shape problem and inherits the same truncation mismatch.

## Alternatives not taken

- **A categorical Dirichlet duration model**, which is genuinely conjugate for a bounded integer
  duration. It exists, `+dur_model/categ_dirichlet.m`, and is unreachable from `train`,
  `../../review/ISSUES.md H2`. This looks like the right answer, never finished.
- **A negative binomial or Poisson duration model**: a better shape, a harder update.
