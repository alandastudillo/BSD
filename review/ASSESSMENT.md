# BSD toolbox: first pass assessment

Reviewer notes, first read of an unfamiliar repository, 15 October 2026.
Repository: `C:\Codes\BSD\BSD`, branch `master`, HEAD `4a89dcc`.

## Scope and method

- **Read only.** Nothing was edited, installed or executed. MATLAB is not available in this
  session, so **no claim here is "reproduced"**. Every finding carries a confidence label, and
  anything below *reproduced* has a reproduction step listed as its first task in `ISSUES.md`.
- What I read in full: `hsmm.m`, `hmm.m`, `multimodel.m`, `program_option.m`, both tutorial
  scripts, all of `+inference`, all of `+in_model` and `+trans_model`, the two duration models in
  use, `+emis_model/normal_normal_wishart.m`, and the numeric helpers in `+util` that the E step
  and M step depend on.
- What I only sampled: `uiBSD.m` (181 KB of GUI code, read by targeted search), the four `mar*`
  emission models, `+util/kmeans2015.m` (882 lines, vendored), the `+util/+spm` helpers.
- Two local tools were used, on local files only, with no network: `pdftotext` on
  `Manual BSD.pdf`, and a short Python script to read the MAT file headers in `Examples/`, so that
  the array sizes quoted below are measured rather than guessed.

## 1. What the repository is

A MATLAB toolbox for segmenting continuous multivariate brain signals, EEG, MEG, LFP or fMRI,
into a small set of recurring discrete states, fitted by variational Bayes.

That sentence rests on: `Manual BSD.pdf` section 1.1, "BSD is an interactive Matlab toolbox for
dynamic Brain State Allocation, segmenting a multivariate time series (e.g. EEG/fMRI) into a set
of time intervals (Brain States)", from the Brain Dynamics Laboratory, version 1.0, November
2018; and on the code itself, `hsmm.m:1-44`, which composes a model out of four sub-models, and
`Script_Tutorial_pipeline.m`, which runs envelope, resample, PCA, fit, then topography.

Two model families are implemented:

- **HMM**, `hmm.m`, geometric state durations, full transition matrix,
  `trans_model.categ_dirichlet_matrix` (`hmm.m:26`).
- **HSMM**, `hsmm.m`, an explicit duration distribution per state and a zero diagonal transition
  matrix, `trans_model.categ_dirichlet_matrixdiag0` (`hsmm.m:35`). This is the headline feature.

## 2. Codebase map

| Unit | Role | Notes |
|---|---|---|
| `hsmm.m`, `hmm.m` | model container, composes four sub-models, delegates inference | handle classes, 177 and 123 lines |
| `+inference/train.m` | restarts, state number search, final sequence | 169 lines |
| `+inference/trainn.m` | one VB run: the M step then E step loop, free energy, stopping | 240 lines, two large branches, single block and multi block |
| `+inference/hsmmresidual_decodelog.m` | HSMM E step, residual time forward backward in logs | 123 lines, the numerical heart |
| `+inference/hsmmresidual_decode.m` | the same in linear space | kept alongside, not called by `hsmm.decode` |
| `+inference/hmm_decodelog.m`, `hmm_decode.m` | HMM E step, log and linear | 47 and 46 lines |
| `+inference/hsmmresidual_logviterbi.m` | HSMM most probable sequence | 110 lines, five nested loops |
| `+inference/hmmviterbi.m` | HMM most probable sequence | adapted from the Statistics Toolbox |
| `+emis_model/` | observation models: four Gaussian variants, four MAR variants, two test scripts | copy and paste families, no base class |
| `+dur_model/` | duration models: Gaussian, lognormal, categorical Dirichlet, a uniform stub | |
| `+trans_model/`, `+in_model/` | Dirichlet transition matrix, with and without a zero diagonal, Dirichlet initial state | |
| `+util/` | log domain arithmetic, KL divergences, initialisation, plotting, vendored third party code | 60 files |
| `multimodel.m`, `singletomulti.m` | multi condition, multi subject, multi block wrapper with parameter sharing | |
| `uiBSD.m` | the whole GUI in one class | 181 KB, about 4000 lines |
| `program_option.m` | the default option set, a script and not a function | 28 lines |
| `BSD.m`, `initBSD.m` | one line launchers, `hSmm = uiBSD;` | |

Each sub-model is expected to implement the same informal interface: `prob`, `update`, `expect`,
`priornoinf`, `priorfull`, `posteriorfull`, `cleanpos`, `copy`, `expectfun`, `divklfun`,
`trouble`, `ndatafun`, `sample`, `parsamplfun`, `graf`. No abstract base class enforces it, and
several shipped models do not implement all of it. See issue `H2`.

## 3. One path traced end to end

The documented single subject HSMM path, `Script_Tutorial_basic.m:3-7`:

1. `hsmm(ndim,nstates)` builds four sub-models, defaulting to Normal Wishart emissions, a zero
   diagonal Dirichlet transition matrix, a Dirichlet initial state and a Gaussian duration model
   (`hsmm.m:25-43`).
2. `train` loads defaults by running the `program_option` script inside the function scope
   (`train.m:19`), fills any missing priors (`train.m:35-51`), then either loops over restarts at
   a known state count (`train.m:53-62`), or searches the state count upward from `minstates` and
   stops at the first drop in free energy (`train.m:68-99`).
3. `trainn` initialises a hard state sequence, by k means or by random segments (`trainn.m:29`,
   `+util/initvb.m`), converts it to sufficient statistics (`+util/decodevarinit.m`), then
   iterates: update emissions, duration, transition and initial state (`trainn.m:57-68`), run the
   E step (`trainn.m:70`), compute the free energy (`trainn.m:86`), test convergence
   (`trainn.m:95-104`).
4. The E step is `inference.hsmmresidual_decodelog`. Forward pass `decodelog:30-40`, backward pass
   `decodelog:52-80`, then `gamma`, `xi` and `durcount` are assembled and exponentiated
   (`decodelog:86-108`).
5. Back in `train`, the best restart is chosen by free energy (`train.m:63`), expectations are
   computed (`train.m:113`), and the reported state sequence is the per sample argmax of `gamma`
   (`train.m:118-122`).

**This path does not complete with the shipped defaults.** Step 4 reads `ALPHATOTAL` and
`BETATOTAL` at `decodelog:86`, but those two arrays are written only inside `if opt.decalpha==1`
and `if opt.decbeta==1` (`decodelog:37-39`, `decodelog:48-50`), and `program_option.m:16-17` set
both flags to 0. That is issue `H1`, and it is the first thing to check when MATLAB is available.

## 4. What is genuinely well designed

Said first, and meant. These are the parts I would leave alone.

- **The four way decomposition is the right abstraction.** Emission, duration, transition and
  initial state are independent objects with a uniform update and expectation interface, so the
  inference code in `trainn.m:57-68` never names a distribution family. Adding a new observation
  model genuinely does not touch the inference. This is better separated than most research code.
- **The HSMM duration statistic is correct, and it is not obvious.** The index algebra at
  `decodelog:61-72` accumulates, in the log domain, the expected number of entries into each state
  at each duration. I worked through the `repelem`, `reshape` and `permute` chain by hand: the
  result is indexed by duration and entered state, summed over the previous state, which is
  exactly what the duration M step consumes at `trainn.m:65`. I would not touch it.
- **The zero diagonal transition model is handled consistently.** Forbidding self transitions is
  the correct choice once duration is explicit, and the matching KL divergence carefully drops the
  diagonal element from each row rather than feeding a zero concentration into `KLDirich`
  (`categ_dirichlet_matrixdiag0.m:105-111`). That is the kind of detail that is usually wrong.
- **Free energy with restarts is the right model comparison machinery.** Several random restarts
  per state count, best by free energy (`train.m:53-66`), with the per iteration free energy kept
  for inspection (`trainn.m:118`) and plotted in the tutorial. The intent is sound even where the
  arithmetic needs checking.
- **The sharing design for multi subject studies is elegant.** Four share vectors over condition,
  subject and block let any subset of the four sub-models be tied at any level
  (`trainn.m:136-139`, `+util/groupindex.m`). That is a real feature, not a bolt on.
- **Priors are explicit, data based and user editable.** `priornoinf('databased',...)` scales the
  prior to the observed data range (`normal_normal_wishart.m:188-193`), and the tutorial shows how
  to override a single hyperparameter (`Script_Tutorial_basic.m:37-38`). Good practice for a
  Bayesian toolbox.
- **The move to the log domain was the right call.** A linear and a log version of each decoder
  are both present, and `hsmm.decode` points at the log version (`hsmm.m:87`). Keeping the linear
  one for reference is reasonable.

## 5. Findings

Full detail, with evidence, conditions and the smallest fix, is in `ISSUES.md`. Counts by
severity: High 6, Medium 15, Low 14, Decoration 7, Wish 4.

There is no Critical, and that is deliberate. Nothing I can defend from reading alone produces a
silently wrong published number without a stated condition. The two that come closest are called
out as `H4` and `H6`.

The shape of the problem, one line each:

- The HSMM E step cannot run with the shipped option defaults. `H1`
- Models offered in the GUI do not implement the interface the inference requires. `H2`
- The HSMM Viterbi allocates and grows gigabyte scale arrays for an ordinary recording. `H3`
- The documented EEG pipeline can attach topographies to the wrong state labels. `H4`
- The k means initialisation can crash the first M step on a long final segment. `H5`
- A hard coded occupancy threshold of 15 samples pins dying states to their prior, which removes
  their complexity cost from the free energy used to choose the number of states. `H6`

Two cross cutting observations that are not single issues:

- **The authors did not trust the free energy either.** `trainn.m:97-101` detects a decrease and
  then sets `dif=inf` to keep going, `decodelog:105-108` wraps four outputs in `real()`,
  `decodelog:78-79` has the gamma recursion commented out with "Temporalmente se modifica", and
  `+dur_model/lognormal_normal_gamma.m:212` carries the comment "Se debe revisar" on the KL term.
  These are all on the same quantity. Treat the free energy curve as unverified until item 2 of
  section 8 is run.
- **Dead and duplicated code outnumbers live code in places.** Four `mar` variants, four Gaussian
  emission variants, two decoders per model family, `initvb_old.m`, `miplot3 - copia.m`,
  `topograph.asv`. This is the main reason a newcomer cannot tell what is load bearing.

## 6. Refutation log

Every candidate was taken back to the code and attacked at its claimed severity. What changed:

| Candidate | Claimed | Became | Why |
|---|---|---|---|
| The GUI swaps "Iterations" and "Repetitions" | High | **Dropped** | `uiBSD.m:2567-2568` do look swapped, but `uiBSD.m:985-986` swap them back. The intermediate names are misleading, the mapping onto `opt.maxcyc` and `opt.maxitersim` is correct. Nothing to fix but the names. |
| `groupindex` writes the flag at the wrong index | Medium | **Decoration, `D5`** | `groupindex.m:9` reads `flag(mcond,msubj,mblock)` but `groupindex.m:12` writes `flag(kcond,ksubj,kblock)`. The two coincide for the first member of every group, because the loops start at 1, so grouping is correct today. A real latent bug with no current effect. |
| `in_model.priorfull` tests the posterior | Medium | **Low, `L1`** | `in_model/categ_dirichlet.m:83` tests `posterior` where every sibling class tests `prior`. Wrong, but on the current call path the posterior is empty exactly when the prior needs filling, so the answer is right by accident. |
| The duration pdf is not normalised over 1 to dmax | High | **Medium, `M9`** | It is normalised, but only inside `if sum(abs(X-floor(X)))==0`, `+dur_model/normal_normal_gamma.m:90-92`. The decoder always passes an integer grid, so the branch always fires in practice. The defensible claim is narrower: the E step uses a renormalised density while the M step uses the untruncated conjugate update, so the free energy is not guaranteed to increase. |
| The emission `2*pi` constant corrupts model selection | High | **Medium, `M1`** | `normal_normal_wishart.m:87` uses `-n*log(2*pi)` where the Gaussian constant is `-n/2*log(2*pi)`, and all four MAR models use the correct `-d/2*log(2*pi)`. The offset is identical for every state and every sample, so it cancels in `gamma` and `xi`, and in any comparison at fixed dimension and length. It corrupts the reported free energy, not the chosen number of states. |
| `eval` on GUI strings is an injection risk | SEC | **Dropped** | `uiBSD.m:1810` and `+util/inmatr.m:7` do build code from strings, but the inputs are dropdown entries and loop indices in a desktop application. Slow and hard to debug, not a security finding. Recorded as `D3`. |
| `select_eeg_coords` returns the MATLAB search path | High | **Medium, `M12`, inferred** | `uiBSD.m:3825` reads `path`, which is assigned only inside sibling nested functions at `3781` and `3790`. If MATLAB does not share the variable, `path` resolves to the built in function. This turns on a nested function scoping rule I cannot execute here, so it stays inferred with a one line check. |

Where refutation changed nothing, the reason is stated per issue in `ISSUES.md`. For `H1` in
particular I searched the whole repository: `decalpha` and `decbeta` are assigned in exactly one
place, `program_option.m:16-17`, and assigned 0, and the only other occurrences anywhere are the
three reads inside the decoder.

## 7. Questions for the team, not findings

These are places where the answer lives in someone's head or in a paper, not in the repository.
Asking is the correct move, not writing them up as defects.

1. **Was the switch of `opt.decalpha` and `opt.decbeta` to 0 deliberate?**
   `git show cafb2d5:program_option.m` has both at 1. The decoder's unconditional read of
   `ALPHATOTAL` has been there since `6637b2c`. This looks like a late memory saving change that
   missed one line.
2. **Is the forward initialisation `ALPHA=repmat(in',1,dmax)` intended?** `decodelog:28`, with the
   alternative `[in', zeros(m,dmax-1)]` commented out immediately above. As written, every
   residual duration cell starts at the initial state probability, so the first segment's duration
   is not drawn from the duration model and the initial mass sums to `dmax` rather than to 1.
   There may well be a reason in the source paper. The manual does not give the equations.
3. **What is `Tolerance` meant to be?** The manual, table 6.1 item 5, says "Minimum percentage
   difference between iteration", range 0% to 100%. The code compares an absolute free energy
   difference in nats, `trainn.m:96`. Which is intended?
4. **Where did `Nk < 15` come from?** `normal_normal_wishart.m:143`, with `0.01` and a percentage
   version commented out just above, and `0.001` in the sibling duration model
   (`+dur_model/normal_normal_gamma.m:153`). If it encodes "a state needs at least 15 samples to
   be real", it probably has to scale with record length or sampling rate.
5. **Is `dur_model.categ_dirichlet` meant to be usable?** A non parametric duration distribution
   is a scientifically interesting option, but the class lacks `priorfull` and `expectfun`, so
   `train` cannot reach it. Abandoned, or is a call path missing?
6. **Are `mar2`, `mar3`, `mar4` and `normal_normal_wishart2`, `3` live, or working copies?** They
   differ from their base class in small, deliberate looking ways. If one of them is current, the
   others are a trap for the next reader.
7. **Which third party files may be redistributed?** `+util/kmeans2015.m` carries
   "Copyright 1993-2014 The MathWorks, Inc." at line 152. `+util/+spm/*` and
   `+util/+Normal/spm_kl_normal.m` carry Wellcome Trust Centre for Neuroimaging notices.
   `+util/logsumexp.m` is Microsoft, by Tom Minka. There is no licence file and no README in the
   repository. That is a decision for whoever owns distribution, not a code change.
8. **Is `EEG_data` a placeholder?** It is a 2 byte empty file, added in the two most recent
   commits, `5b07ecb` and `4a89dcc`.

## 8. First five things to run, in this order, when MATLAB is available

Each of these is a measurement, not a change. The first four should take minutes.

1. `hsmmmodel=hsmm(2,4); outhsmm=hsmmmodel.train(data);` on `Examples/Example1_data2dim.mat`.
   Expect the undefined `ALPHATOTAL` error of `H1`. Then repeat with
   `train(data,'decalpha',1,'decbeta',1)` to confirm that the rest of the path runs.
2. With the flags on, print `[outhsmm.fedetail.fe]` and check whether it is monotone. That sizes
   `M3`, `M9` and the masking at `trainn.m:97-101` in one shot.
3. `sum(outhsmm.decodevar.gamma,2)` at a few time points. If the rows do not sum to 1, `M5` is
   live, because the whole matrix is normalised by its value at `t=10` alone
   (`decodelog:86-91`).
4. Count how many states have `sum(gamma(:,k)) < 15` after a fit, and compare
   `unique(outhsmm.stateseq)` against `1:outhsmm.nstates`. That is `H6` and `H4` together.
5. Only then `hsmmmodel.viterbi(data)` on the 5780 sample pipeline input with `dmax=200`, while
   watching memory. That is `H3`. Do not start with this one.

## 9. Measured facts used above

So that no number here is taken on trust.

- `Examples/Example5_EEG.mat` holds `dataeeg64`, 23120 by 64, and `chanlocs`, 1 by 64. At the 512
  Hz stated in `Script_Tutorial_pipeline.m:8` that is 45.2 s, and 5780 samples after the resample
  to 128 Hz at line 11.
- `Examples/Example1_data2dim.mat` holds `data`, 5003 by 2, and `stateseq`, 1 by 5003. Lengths
  agree, so the tutorial's error line is sound for this file.
- `Examples/Example4_Sourcesconn.mat` holds `data`, 15000 by 2, but `stateseq`, 1 by 14998. Those
  lengths do not agree. See `L9`.
- `Examples/Example2_Topography.mat` holds `data`, 4000 by 128, and `stateseq`, 1 by 4000.
- Method: the MAT version 5 headers were parsed locally with Python, reading element tags and
  dimensions only.
