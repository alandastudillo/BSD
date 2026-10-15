# Architecture decision records

Fourteen decisions reconstructed from the code, and four proposed by this review. Standing and
conventions: see [`../README.md`](../README.md). None of the first fourteen was written by the
authors; they are inferences from the code, and the authors may well have decided differently from
how it reads. Treat each one as a question as much as a record.

| ADR | Decision | Status |
|---|---|---|
| [ADR-0001](0001-fit-by-variational-bayes-rather-than-em-or-mcmc.md) | Fit by variational Bayes rather than EM or MCMC | accepted, reconstructed |
| [ADR-0002](0002-model-state-duration-explicitly-as-a-hidden-semi-markov-model.md) | Model state duration explicitly, as a hidden semi Markov model | accepted, reconstructed |
| [ADR-0003](0003-use-the-residual-time-formulation-for-the-hsmm-e-step.md) | Use the residual time formulation for the HSMM E step | accepted, reconstructed |
| [ADR-0004](0004-work-in-the-log-domain-with-per-step-rescaling.md) | Work in the log domain, with per step rescaling | accepted, reconstructed |
| [ADR-0005](0005-compose-the-model-from-four-pluggable-sub-models-behind-an.md) | Compose the model from four pluggable sub-models behind an informal interface | accepted, reconstructed |
| [ADR-0006](0006-use-conjugate-exponential-family-priors-throughout.md) | Use conjugate exponential family priors throughout | accepted, reconstructed |
| [ADR-0007](0007-truncate-state-durations-at-a-user-supplied-maximum.md) | Truncate state durations at a user supplied maximum | accepted, reconstructed |
| [ADR-0008](0008-use-the-free-energy-for-convergence-restarts-and-model-order.md) | Use the free energy for convergence, restarts and model order, searched greedily | accepted, reconstructed |
| [ADR-0009](0009-initialise-from-a-hard-state-sequence-with-random-restarts.md) | Initialise from a hard state sequence, with random restarts | accepted, reconstructed |
| [ADR-0010](0010-express-group-level-parameter-tying-as-three-element-share.md) | Express group level parameter tying as three element share vectors | accepted, reconstructed |
| [ADR-0011](0011-report-the-per-sample-most-probable-state-by-default-not-the.md) | Report the per sample most probable state by default, not the Viterbi path | accepted, reconstructed |
| [ADR-0012](0012-carry-options-in-a-script-injected-struct.md) | Carry options in a script injected struct | accepted, reconstructed |
| [ADR-0013](0013-reset-a-weakly-supported-state-to-its-prior-rather-than.md) | Reset a weakly supported state to its prior rather than letting it collapse | accepted, reconstructed |
| [ADR-0014](0014-ship-a-full-graphical-interface-alongside-the-scripting-api.md) | Ship a full graphical interface alongside the scripting API, in a single class | accepted, reconstructed |
| [ADR-0015](0015-change-the-update-contract-from-per-sample-arrays-to.md) | Change the update contract from per sample arrays to sufficient statistics | proposed |
| [ADR-0016](0016-compute-gamma-with-a-checkpointed-forward-backward-pass.md) | Compute gamma with a checkpointed forward backward pass | proposed |
| [ADR-0017](0017-add-explicit-seed-control.md) | Add explicit seed control | proposed |
| [ADR-0018](0018-introduce-an-abstract-base-class-per-sub-model-family.md) | Introduce an abstract base class per sub-model family | proposed |

## How to read these

Decisions 1 to 14 explain why the system is shaped as it is, and most of them are defensible. The
consequences sections are where the review lives: each lists what the decision bought and what it
cost, with the cost tied to a numbered issue wherever one exists.

Decisions 15 to 18 are the reviewer proposals. Together they are the minimum structural change that
removes the memory problem and makes the system testable, and none of them changes the fitted
model. Costings: [`../../review/MEMORY.md`](../../review/MEMORY.md) section 5 and
[`../../review/VALIDATION.md`](../../review/VALIDATION.md) section 14.
