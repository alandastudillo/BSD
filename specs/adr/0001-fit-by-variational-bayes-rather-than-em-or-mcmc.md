# ADR-0001: Fit by variational Bayes rather than EM or MCMC

- **Status** Accepted, reconstructed from the code
- **Date reconstructed** 15 October 2026
- **Confidence** read. The manual, section 1.1, cites Beal 2003, which is the variational Bayes reference, so this choice is also documented

## Context

The toolbox must fit a latent state model to brain signals and report how certain the fit is. The
three realistic choices in 2018 were maximum likelihood by EM, full Bayesian inference by MCMC, or
variational Bayes.

## Decision

Mean field variational Bayes with conjugate exponential family priors. `opt.train='VB'` is the
default and the only fully implemented path, `program_option.m:1`. Every sub-model carries an `EM`
branch, and in most classes it is commented out or stubbed, for example
`+emis_model/normal_normal_wishart.m:101-106`.

## Consequences

- Posteriors rather than point estimates, which is what the manual promises in figure 1.2.
- A single scalar objective, the free energy, serves convergence, restart selection and model order
  selection. That economy is attractive and it concentrates risk: see `ADR-0008`.
- Conjugacy constrains which observation and duration models are easy to add.
- The `EM` branches are dead weight: present in every class, never exercised, and one of them has a
  loop bug, `../../review/ISSUES.md L3`.

## Alternatives not taken

- **MCMC** would give exact posteriors and no free energy to get wrong, at a run time cost a GUI
  driven toolbox could not absorb in 2018.
- **Plain EM** would be simpler and would lose the uncertainty, which is the toolbox selling
  point.
