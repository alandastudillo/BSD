# Business rules

The domain rules the BSD toolbox embodies. These are statements about **brain state segmentation**,
not about software: they would survive a rewrite in any language. Reconstructed from the code and
from `Manual BSD.pdf`, 15 October 2026. Standing and conventions: see [`README.md`](README.md).

Each rule names where it is enforced, or notes that it is not.

---

## Domain model

**BR-001. A recording is a sequence of states.**
A continuous multivariate brain signal is modelled as a sequence of discrete, recurring states.
Every sample belongs to exactly one state.
*Enforced by*: the model structure itself. `Manual BSD.pdf` section 1.1.

**BR-002. A state is a characteristic pattern, not a time.**
A state is defined by its emission model, a spatial pattern of amplitude or a pattern of directed
lag relationships, and the same state can recur any number of times.
*Enforced by*: one emission model per state, shared across all visits.
`+emis_model/*`, indexed by state throughout.

**BR-003. A visit to a state has a duration drawn from that state's own duration distribution.**
This is the defining rule of the toolbox and the reason it is a semi Markov model rather than a
Markov model.
*Enforced by*: `+dur_model/*`, consumed at `+inference/trainn.m:65`.
`Manual BSD.pdf` section 1.3.

**BR-004. A state cannot follow itself.**
Because duration is explicit, a self transition would double count the duration. Consecutive
segments are therefore always in different states.
*Enforced by*: `+trans_model/categ_dirichlet_matrixdiag0.m:64`, which zeroes the diagonal after
every update, and `:86`, which excludes it from the prior.

**BR-005. The duration of a visit is at least one sample and at most `dmax` samples.**
*Enforced by*: the duration grid `1:dmax` passed to every duration evaluation,
`+inference/hsmmresidual_decodelog.m:21`, and `max(1,·)` when generating,
`+dur_model/normal_normal_gamma.m:66`.
*Note*: `dmax` is a user parameter, so this rule is a modelling choice and not a property of the
brain. See BR-013.

**BR-006. Where durations are not modelled explicitly, they are implicitly geometric.**
That is the HMM case, offered as an alternative family.
*Enforced by*: `hmm.m`, which has no duration model and a free transition diagonal, `hmm.m:26`.

**BR-007. The first sample belongs to a state drawn from a separate initial state distribution.**
*Enforced by*: `+in_model/categ_dirichlet.m`, updated from the first row of the responsibilities,
`+inference/trainn.m:68`.

---

## Inference rules

**BR-008. Parameters are random variables with priors, not point values.**
Every parameter has a conjugate prior and the output is a posterior distribution.
*Enforced by*: the `prior` and `posterior` structures on every sub-model class.
`Manual BSD.pdf` section 1.3, figure 1.2.

**BR-009. A prior must be set before fitting, and may be uninformative or data based.**
*Enforced by*: `+inference/train.m:35-51`, which calls `priornoinf` for any sub-model whose prior
is incomplete; `opt.prior` selects `'default'` or `'databased'`, `program_option.m:20`.

**BR-010. Fitting maximises a lower bound on the model evidence, the free energy.**
*Enforced by*: `hsmm.m:161-174`, and the convergence test at `+inference/trainn.m:95-104`.

**BR-011. The free energy also decides between fits.**
It selects between random restarts and between candidate numbers of states.
*Enforced by*: `+inference/train.m:63` and `:96`.
*Defective*: the quantity it compares is not a correct free energy and may not be the best value
seen. → `../review/ISSUES.md M1, M3, M9`

**BR-012. The number of states may be unknown and is then chosen by the data.**
A search runs from `minstates` upward and stops when the free energy stops improving.
*Enforced by*: `+inference/train.m:68-99`, triggered by `nstates = 0`.
*Defective*: the stop is greedy and the complexity penalty can vanish.
→ `../review/ISSUES.md H6`

**BR-013. The maximum duration `dmax` is a modelling decision with three consequences, not one.**
It truncates the duration distribution, it sets the duration prior mean to `dmax/2`, and it sets
the segment lengths of the random initialisation.
*Enforced by*: `+dur_model/normal_normal_gamma.m:187-188` and `+util/unirndseq.m:3`.
*Consequence*: results move with `dmax` through three channels at once.
→ `../review/ISSUES.md M13`

**BR-014. A state that almost no data supports is returned to its prior.**
Operationally: a state whose expected occupancy falls below a threshold is reset rather than fitted.
*Enforced by*: `+emis_model/normal_normal_wishart.m:143`, with the threshold hard coded at 15
samples, and `+dur_model/normal_normal_gamma.m:153`, with a different threshold of 0.001.
*Questionable*: the thresholds are absolute sample counts, inconsistent between sub-models, and
they remove the state's complexity cost from the free energy.
→ `../review/ISSUES.md H6`, and `../review/ASSESSMENT.md` question 4.

---

## Study and group rules

**BR-015. A study is organised as condition, then subject, then block.**
Exactly three levels, in that order.
*Enforced by*: the data contract `X.cond(c).subj(s).block(b).data`, `+util/addmodel.m:2-20`.

**BR-016. Each of the four sub-models may be tied, independently, at any combination of the three
levels.**
So emissions can be subject specific while transitions are shared across everyone, and so on.
*Enforced by*: the four share vectors `opt.shareemis`, `sharetrans`, `sharein`, `sharedur`,
`program_option.m:21-24`, resolved by `+util/groupindex.m`.

**BR-017. By default, states are subject specific and shared across conditions and blocks; the
starting distribution is condition specific.**
`shareemis = sharetrans = sharedur = [1 0 1]` and `sharein = [0 1 1]`, where 1 means tied.
*Enforced by*: `program_option.m:21-24`.
*Inferred*: this is a scientific default with consequences for any group comparison, and the manual
does not justify it. A question for the authors.

**BR-018. Blocks are conditionally independent given the parameters.**
Each block is decoded separately and contributes additively to the free energy.
*Enforced by*: `multimodel.m:22-33` for decoding and `multimodel.m:105-111` for the likelihood sum.
*Consequence*: there is no carry over of state across a block boundary, which is correct for
separate recordings and wrong if a user splits one continuous recording into blocks. Not documented.

---

## Output rules

**BR-019. The per sample output is a probability over states, not a label.**
*Enforced by*: `decodevar.gamma`, `T × nstates`.

**BR-020. A single label sequence, when required, is either the per sample most probable state or
the most probable whole sequence, and the two are different quantities.**
*Enforced by*: `opt.seq`, `'maxgamma'` by default, `'viterbi'` otherwise,
`+inference/train.m:118-151`.
*Consequence*: the default can return a sequence the model assigns zero probability to, for example
a visit shorter than the duration model permits. Not documented.

**BR-021. State labels are arbitrary and carry no meaning across fits.**
Two fits of the same data may label the same state differently, so any comparison needs matching.
*Enforced by*: nothing. `+util/renameseq.m` attempts matching for the tutorial and is defective.
→ `../review/ISSUES.md L10`

**BR-022. In sensor space, a state's topography is recovered by regression, not read off the
emission mean.**
Because the model is fitted in a reduced space, the emission means live in that space.
*Enforced by*: `Script_Tutorial_pipeline.m:25-30`, a robust GLM of the sensor data on the state
indicator matrix.
*Defective*: the indicator matrix assumes contiguous state labels.
→ `../review/ISSUES.md H4`

---

## Rules that the domain requires and the code does not enforce

Listed because their absence is itself a finding.

| Rule | Status |
|---|---|
| **BR-023.** A result must be reproducible from a recorded configuration. | **Absent.** No seed control anywhere. → `../review/ISSUES.md M15` |
| **BR-024.** The data and any ground truth labels must be the same length. | **Absent.** No validation; `Examples/Example4_Sourcesconn.mat` violates it. → `../review/ISSUES.md L9` |
| **BR-025.** A fitted model must be comparable against a held out sample. | **Absent.** No predictive scoring exists, only the internal free energy. → `../review/VALIDATION.md` tier 6 |
| **BR-026.** The number of returned states must equal the number of distinct labels in the returned sequence. | **Defective.** `nstates` is a count and `stateseq` holds labels; they can disagree. → `../review/ISSUES.md H4` |
| **BR-027.** Input must be rejected clearly when the model cannot be fitted to it. | **Absent.** No input validation in the model or inference layer; `grep` finds no `assert` and no `error` in either. → `../review/VALIDATION.md` tier 8 |
