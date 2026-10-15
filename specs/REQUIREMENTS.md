# Requirements, EARS form

Reconstructed requirements for the BSD toolbox, 15 October 2026. Standing, conventions and status
values: see [`README.md`](README.md). Domain rules behind these requirements:
[`BUSINESS_RULES.md`](BUSINESS_RULES.md).

EARS keywords are used strictly:

| Pattern | Form |
|---|---|
| Ubiquitous | *The toolbox shall …* |
| Event driven | *When \<trigger\>, the toolbox shall …* |
| State driven | *While \<state\>, the toolbox shall …* |
| Optional feature | *Where \<feature\>, the toolbox shall …* |
| Unwanted behaviour | *If \<trigger\>, then the toolbox shall …* |

Identifiers are stable. Do not renumber. A **Defective** or **Absent** status names the issue in
`../review/ISSUES.md`, and each issue names its test in `../review/VALIDATION.md` section 16.

---

## 1. Model construction, `REQ-MOD`

| ID | Requirement | Evidence | Status |
|---|---|---|---|
| REQ-MOD-001 | The toolbox shall represent a brain state model as four independent sub-models: emission, duration, transition and initial state. | `hsmm.m:2-9`, `Manual` 1.3 | Implemented |
| REQ-MOD-002 | When the user constructs a model and omits a sub-model, the toolbox shall substitute that sub-model's default. | `hsmm.m:25-43` | Implemented |
| REQ-MOD-003 | Where the model is a hidden semi Markov model, the toolbox shall use a transition model that forbids self transitions. | `hsmm.m:35` | Implemented |
| REQ-MOD-004 | Where the model is a hidden Markov model, the toolbox shall use a transition model with a free diagonal and no duration model. | `hmm.m:26`, `hmm.m:2-8` | Implemented |
| REQ-MOD-005 | The toolbox shall accept a model whose number of states is unknown, denoted by zero. | `hsmm.m:20`, `train.m:67` | Implemented |
| REQ-MOD-006 | If a supplied sub-model does not implement the inference interface, then the toolbox shall reject it at construction with a message naming the missing method. | — | **Absent** → `H2` |
| REQ-MOD-007 | The toolbox shall offer at least one amplitude emission model and at least one autoregressive emission model. | `+emis_model/normal_normal_wishart.m`, `+emis_model/mar.m` | Partial: the MAR models are unreachable → `H2` |
| REQ-MOD-008 | The toolbox shall offer a Gaussian and a lognormal duration model. | `+dur_model/normal_normal_gamma.m`, `lognormal_normal_gamma.m` | Implemented |
| REQ-MOD-009 | Where a non parametric duration model is offered, the toolbox shall allow it to be fitted. | `+dur_model/categ_dirichlet.m` | **Defective**: no `priorfull`, so `train` cannot reach it → `H2` |
| REQ-MOD-010 | When the user copies a model for a new fit, the toolbox shall discard the previous posterior. | `hsmm.m:126-151`, `cleanpos` | **Defective** for the duration models → `M10` |

## 2. Priors, `REQ-PRI`

| ID | Requirement | Evidence | Status |
|---|---|---|---|
| REQ-PRI-001 | The toolbox shall define a conjugate prior for every model parameter. | every sub-model's `prior` struct | Implemented |
| REQ-PRI-002 | When a fit begins and a sub-model's prior is incomplete, the toolbox shall set an uninformative prior for it. | `train.m:35-51` | Implemented |
| REQ-PRI-003 | Where the user requests a data based prior, the toolbox shall scale the prior to the observed data range. | `normal_normal_wishart.m:188-193` | Implemented |
| REQ-PRI-004 | The toolbox shall allow the user to override any individual hyperparameter before fitting. | `Script_Tutorial_basic.m:37-38` | Implemented |
| REQ-PRI-005 | If a data dimension has zero range, then the toolbox shall reject the data rather than forming an infinite prior precision. | `normal_normal_wishart.m:190` | **Absent**: silent division by zero → `VALIDATION.md` tier 8 |
| REQ-PRI-006 | When a data based duration prior is requested, the toolbox shall centre it at half the maximum duration. | `+dur_model/normal_normal_gamma.m:187` | Implemented, and see `M13` for the consequence |

## 3. Initialisation, `REQ-INI`

| ID | Requirement | Evidence | Status |
|---|---|---|---|
| REQ-INI-001 | The toolbox shall initialise the state responsibilities from a hard state sequence. | `trainn.m:29-32` | Implemented |
| REQ-INI-002 | Where the user selects random initialisation, the toolbox shall generate random segments whose lengths are bounded by the maximum duration. | `+util/unirndseq.m:3` | Implemented |
| REQ-INI-003 | Where the user selects k means initialisation, the toolbox shall cluster the samples and use the cluster labels as the initial sequence. | `+util/initvb.m:3-6` | Implemented |
| REQ-INI-004 | The toolbox shall ensure every state appears in the initial sequence. | `+util/initvb.m:19-21` | Implemented for the single block path; **Absent** for the k means path, where nothing checks |
| REQ-INI-005 | If an initial sequence containing every state cannot be generated, then the toolbox shall fail with a message naming the data length, the maximum duration and the number of states. | `+util/initvb.m:19-21` | **Defective**: infinite loop → `M13` |
| REQ-INI-006 | Where the user supplies an initial sequence, the toolbox shall use it unchanged. | `+util/initvb.m:31-52`, `initoption='seqprior'` | Implemented, undocumented in the manual |
| REQ-INI-007 | When converting a hard sequence to duration counts, the toolbox shall assign every segment a duration in the range one to `dmax`. | `+util/decodevarinit.m:31-44` | **Defective**: can index `dmax+1` → `H5` |

## 4. Inference, `REQ-INF`

| ID | Requirement | Evidence | Status |
|---|---|---|---|
| REQ-INF-001 | The toolbox shall fit models by variational Bayes, alternating a parameter update and a state update. | `trainn.m:50-112` | Implemented |
| REQ-INF-002 | While fitting, the toolbox shall compute a free energy after each iteration. | `trainn.m:86` | Implemented |
| REQ-INF-003 | The free energy shall equal the expected log likelihood minus the sum of the four parameter divergences. | `hsmm.m:161-174` | **Defective**: the likelihood term carries a wrong Gaussian constant → `M1` |
| REQ-INF-004 | When the free energy improves by less than the tolerance, the toolbox shall stop iterating. | `trainn.m:50`, `:96` | Implemented |
| REQ-INF-005 | When the iteration count reaches the configured maximum, the toolbox shall stop iterating. | `trainn.m:50` | Implemented |
| REQ-INF-006 | If the free energy decreases between iterations, then the toolbox shall record that it happened and report it to the user. | `trainn.m:97-101` | **Defective**: it is silently absorbed by setting the difference to infinity → `M3` |
| REQ-INF-007 | The toolbox shall return, for the chosen fit, the best free energy achieved rather than the last. | `trainn.m:115` | **Defective**: returns the last → `M3` |
| REQ-INF-008 | The free energy of a fit shall not decrease across iterations. | the variational Bayes guarantee | **Defective, expected**: the duration E step and M step use different densities → `M9` |
| REQ-INF-009 | Where the user requests several restarts, the toolbox shall fit each independently and keep the one with the highest free energy. | `train.m:53-66` | Implemented |
| REQ-INF-010 | Where parallel execution is enabled, the toolbox shall distribute restarts across workers. | `train.m:53`, `:73` | Implemented, with a memory consequence → `MEMORY.md` 3.1 |
| REQ-INF-011 | While fitting, the toolbox shall report progress to the user when verbose output is enabled. | `trainn.m:88-93` | Implemented |
| REQ-INF-012 | If an emission model reports a numerical failure, then the toolbox shall abandon that restart and record it. | `trainn.m:58-61` | **Defective**: abandons it but then reads fields that were never written → `L14` |

## 5. Decoding, `REQ-DEC`

| ID | Requirement | Evidence | Status |
|---|---|---|---|
| REQ-DEC-001 | The toolbox shall compute, for each sample, the posterior probability of each state. | `decodevar.gamma` | **Defective**: normalised using a single hard coded time index → `M5` |
| REQ-DEC-002 | The per sample posterior probabilities shall sum to one at every sample. | arithmetic requirement | **Defective** → `M5` |
| REQ-DEC-003 | The toolbox shall compute the expected number of transitions between each pair of states. | `decodevar.xi` | Implemented |
| REQ-DEC-004 | Where the model has explicit durations, the toolbox shall compute the expected number of entries into each state at each duration. | `decodelog:61-72` | Implemented, and verified by hand |
| REQ-DEC-005 | The toolbox shall compute the log likelihood of the data under the current parameters. | `decodelog:99` | Partial: shifted by the constant in `M1` |
| REQ-DEC-006 | The toolbox shall perform decoding in the log domain with per step rescaling, so that long recordings do not underflow. | `decodelog:30-40`, `ADR-0004` | Implemented |
| REQ-DEC-007 | When decoding with the shipped default options, the toolbox shall complete without error. | `program_option.m:16-17` versus `decodelog:86` | **Defective**: undefined variable → `H1` |
| REQ-DEC-008 | If the data has fewer samples than the decoder's internal assumptions require, then the toolbox shall reject it with a clear message. | `decodelog:88` | **Absent**: indexes sample 10 unconditionally → `M5` |
| REQ-DEC-009 | Where the user requests it, the toolbox shall return the full forward and backward arrays. | `decodelog:112-120` | **Defective**: the returns are commented out → `L11` |
| REQ-DEC-010 | The toolbox shall compute the single most probable state sequence on request. | `hsmmresidual_logviterbi.m`, `hmmviterbi.m` | Partial: correctness caveats in `L12`, `L13`, and impractical memory in `H3` |
| REQ-DEC-011 | Where the HMM most probable sequence is computed, the toolbox shall use the fitted initial state distribution. | `hmmviterbi.m:32-33` | **Defective**: assumes state one at step zero |

## 6. Number of states, `REQ-SEL`

| ID | Requirement | Evidence | Status |
|---|---|---|---|
| REQ-SEL-001 | When the number of states is unknown, the toolbox shall fit candidate models from `minstates` to `maxstates`. | `train.m:69` | Implemented |
| REQ-SEL-002 | The toolbox shall select the candidate number of states with the highest free energy. | `train.m:96-109` | Partial: the search stops at the first decrease, so later candidates are never fitted |
| REQ-SEL-003 | When a candidate's free energy is lower than its predecessor's, the toolbox shall stop the search and keep the predecessor. | `train.m:97-99` | Implemented, by design → `ADR-0008` |
| REQ-SEL-004 | The free energy shall penalise additional states that the data does not support. | the purpose of the bound | **Defective**: a state pinned to its prior contributes no divergence → `H6` |
| REQ-SEL-005 | If the search range begins below two states, then the toolbox shall reject the configuration. | `train.m:97` | **Defective**: indexes element zero → `M7` |
| REQ-SEL-006 | The toolbox shall report the number of states in the selected model. | `train.m:122` | **Defective**: reports the count of labels used, which can differ from the model's state count → `H4` |

## 7. Group studies, `REQ-GRP`

| ID | Requirement | Evidence | Status |
|---|---|---|---|
| REQ-GRP-001 | The toolbox shall accept a study organised as condition, subject and block. | `+util/addmodel.m:2-20` | Implemented |
| REQ-GRP-002 | The toolbox shall allow each of the four sub-models to be tied independently across conditions, subjects or blocks. | `program_option.m:21-24`, `+util/groupindex.m` | Implemented |
| REQ-GRP-003 | While fitting a study, the toolbox shall update each tied sub-model once from the pooled statistics of its group. | `trainn.m:168-198` | Implemented |
| REQ-GRP-004 | While fitting a study, the toolbox shall decode each block separately. | `multimodel.m:22-33` | Implemented |
| REQ-GRP-005 | The free energy of a study shall be the sum of the block log likelihoods minus the divergences of the distinct sub-models. | `multimodel.m:70-119` | Implemented |
| REQ-GRP-006 | When initialising a study, the toolbox shall ensure every state appears in every block. | `trainn.m:146-154` | Implemented |
| REQ-GRP-007 | The toolbox shall not retain per sample decoding results for blocks it has finished updating. | — | **Absent**: everything is retained, for every restart → `MEMORY.md` 3 |

## 8. Synthetic data, `REQ-GEN`

| ID | Requirement | Evidence | Status |
|---|---|---|---|
| REQ-GEN-001 | The toolbox shall generate data from a model, returning the data, the state sequence and the durations. | `hsmm.m:45-79` | Implemented |
| REQ-GEN-002 | When generating, the toolbox shall draw each segment's duration from the duration model of its state. | `hsmm.m:57` | Implemented |
| REQ-GEN-003 | Where a fixed length is requested, the toolbox shall truncate the generated series to that length. | `hsmm.m:73-78` | Implemented |
| REQ-GEN-004 | When generating with an autoregressive emission model, the toolbox shall condition each segment on the preceding samples. | `hsmm.m:65-72` | Implemented |

## 9. Outputs and persistence, `REQ-OUT`

| ID | Requirement | Evidence | Status |
|---|---|---|---|
| REQ-OUT-001 | The toolbox shall return the fitted posterior for every parameter. | the `posterior` struct on each sub-model | Implemented |
| REQ-OUT-002 | The toolbox shall return point expectations for every parameter on request. | `train.m:113`, `expectfun` | Implemented, except where `expectfun` is missing → `H2` |
| REQ-OUT-003 | The toolbox shall return the per iteration free energy history. | `trainn.m:118` | Implemented |
| REQ-OUT-004 | The toolbox shall return a single state sequence per block. | `train.m:118-151` | Implemented |
| REQ-OUT-005 | Where the per sample most probable state is requested, the toolbox shall return the per sample argmax of the state posterior. | `train.m:120-121` | Implemented |
| REQ-OUT-006 | The returned state labels shall be the contiguous integers one to the returned number of states. | the downstream assumption at `Script_Tutorial_pipeline.m:27-29` | **Defective** → `H4` |
| REQ-OUT-007 | Where the user saves a study, the toolbox shall persist the data, the model, the options and the results together. | `uiBSD.m:652` | Implemented |

## 10. User interface, `REQ-UI`

| ID | Requirement | Evidence | Status |
|---|---|---|---|
| REQ-UI-001 | The toolbox shall provide a graphical interface covering loading, model creation, training, decoding and viewing. | `uiBSD.m`, `Manual` sections 3 to 7 | Implemented |
| REQ-UI-002 | The toolbox shall provide a scripting interface equivalent to the graphical one. | both tutorial scripts | Implemented |
| REQ-UI-003 | When the user sets a training parameter in the interface, the toolbox shall use that value. | `uiBSD.m:970-1002` | **Defective** for the k means iteration limit → `M6` |
| REQ-UI-004 | The interface shall offer only sub-models that can be fitted. | `uiBSD.m:3941-3952` | **Defective**: it offers every file in the package folder, including test scripts → `H2` |
| REQ-UI-005 | If the user enters a parameter value the inference cannot accept, then the interface shall refuse it with a message. | `uiBSD.m:2547-2551` | Partial: range checks exist but admit one state → `M7` |
| REQ-UI-006 | When the user selects an electrode coordinate file, the toolbox shall use that file for topographic displays. | `uiBSD.m:3761-3825` | **Defective, inferred**: the chosen path is probably not returned → `M12` |
| REQ-UI-007 | The interface shall list electrode coordinate files that are present in the installation. | `uiBSD.m:34` | **Defective**: offers `ele_bio128.xyz`, which does not exist → `M12` |

## 11. Non functional, `REQ-NFR`

| ID | Requirement | Evidence | Status |
|---|---|---|---|
| REQ-NFR-001 | The toolbox shall fit a study of 28 two minute blocks within the memory of a 16 GB workstation. | the stated workload | **Defective**: about 8 GB of retained results alone, and more with parallel enabled → `MEMORY.md` |
| REQ-NFR-002 | The memory used by a study fit shall not grow with the number of restarts or candidate state counts. | — | **Absent** → `MEMORY.md` 3 |
| REQ-NFR-003 | When the user supplies a seed, the toolbox shall produce identical results for identical inputs. | — | **Absent** → `M15` |
| REQ-NFR-004 | The toolbox shall run on every platform that the supported MATLAB release supports. | `Manual` 2.1 | **Defective**: the interface's model menus depend on Windows `ls` behaviour → `M11` |
| REQ-NFR-005 | The toolbox shall require MATLAB R2015a or later. | `Manual` 2.1, and `repelem` at `decodelog:62`, introduced in R2015a | Implemented, and consistent |
| REQ-NFR-006 | The toolbox shall declare its external dependencies in the repository. | `Manual` 2.1 only | Partial: EEGLAB is in the manual, nothing is in the repository → `M14` |
| REQ-NFR-007 | The toolbox shall state the licence under which it and its vendored third party files are distributed. | — | **Absent** → `M14` |
| REQ-NFR-008 | The toolbox shall have an automated test suite covering the inference. | — | **Absent**: no harness exists → `VALIDATION.md` |
| REQ-NFR-009 | If an input cannot be fitted, then the toolbox shall reject it with a message that names the problem. | — | **Absent**: no validation in the model or inference layer → `VALIDATION.md` tier 8 |
| REQ-NFR-010 | The toolbox shall not silently discard numerical anomalies. | `decodelog:105-108`, four `real()` calls; `trainn.m:97-101` | **Defective** → `M3` |

---

## 12. Coverage summary

| Area | Requirements | Implemented | Partial | Defective | Absent |
|---|---|---|---|---|---|
| `REQ-MOD` model construction | 10 | 6 | 1 | 2 | 1 |
| `REQ-PRI` priors | 6 | 5 | 0 | 0 | 1 |
| `REQ-INI` initialisation | 7 | 4 | 1 | 2 | 0 |
| `REQ-INF` inference | 12 | 7 | 0 | 5 | 0 |
| `REQ-DEC` decoding | 11 | 3 | 2 | 5 | 1 |
| `REQ-SEL` number of states | 6 | 2 | 1 | 3 | 0 |
| `REQ-GRP` group studies | 7 | 6 | 0 | 0 | 1 |
| `REQ-GEN` synthetic data | 4 | 4 | 0 | 0 | 0 |
| `REQ-OUT` outputs | 7 | 6 | 0 | 1 | 0 |
| `REQ-UI` interface | 7 | 2 | 1 | 4 | 0 |
| `REQ-NFR` non functional | 10 | 1 | 1 | 3 | 5 |
| **total** | **87** | **46** | **7** | **25** | **9** |

So **a little over half the reconstructed requirements are met**, and the defects cluster hard:
inference, decoding and state number selection hold 13 of the 25, and those three areas are the
science. The interface holds 4 more. Non functional is the weakest area by proportion, which is
what you would expect of research code: 1 of 10.

Read that table with care. "Implemented" means the code appears to do it, from reading only. With
no test suite, **none of the 46 is confirmed**, and that is the single most important line in this
document.
