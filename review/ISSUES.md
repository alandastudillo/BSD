# BSD toolbox: issues

Companion to `ASSESSMENT.md`. Issues only, sorted by severity, then by difficulty to trigger with
the easiest first.

**MATLAB was not available, so nothing here is `reproduced`.** Confidence labels are:
`read` (read the code directly, the mechanism is clear), `inferred` (strong evidence, not read end
to end), `suspected` (pattern match only). Every issue below `reproduced` lists a reproduction
step as its first task, not a code change.

Severity: Critical, High, Medium, Low, Decoration (cosmetic, changes nothing), Wish (not a defect).
Categories: `CFG` `SEC` `COR` `DAT` `PRF` `EVAL` `SKEW` `OBS` `FE` `DEP` `DOC` `TST`.

Counts: High 6, Medium 15, Low 14, Decoration 7, Wish 4.

Every issue is mapped to the test that would catch it in `VALIDATION.md` section 16.

---

## High

### H1. The HSMM E step reads two arrays that the shipped defaults never fill

- **Severity** High. **Difficulty** easy. **Category** CFG. **Confidence** read.
- **Evidence**
  - `+inference/hsmmresidual_decodelog.m:37-39`: `ALPHATOTAL(:,:,t)=ALPHA;` sits inside
    `if opt.decalpha==1`.
  - `+inference/hsmmresidual_decodelog.m:48-50`: `BETATOTAL(:,:,T)=BETA;` sits inside
    `if opt.decbeta==1`, as does the per step write at `:58-60`.
  - `+inference/hsmmresidual_decodelog.m:86`: `aux=ALPHATOTAL+BETATOTAL;` is unconditional, and
    `gamma` is built from it at `:87-91`.
  - `program_option.m:16-17`: `opt.decalpha=0; opt.decbeta=0;`.
  - `grep` over the whole repository: those two fields are assigned nowhere else, and are read
    only at the three sites above.
  - `git show cafb2d5:program_option.m` has `opt.decalpha=1; opt.decbeta=1;`. The unconditional
    read has existed since `6637b2c`. So this is a regression introduced by the defaults change in
    the most recent upload of `program_option.m`.
- **Condition** Any HSMM decode or train that does not explicitly pass `decalpha` and `decbeta`.
  That is both tutorials, and the GUI, which copies the same defaults at `uiBSD.m:64-65`.
- **Impact** `hsmm.decode` and therefore `hsmm.train` stop with an undefined variable error. The
  HMM path is unaffected: `hmm_decodelog.m` does not use these arrays.
- **Reproduce** `load Examples/Example1_data2dim.mat; m=hsmm(2,4); m.train(data)`. Expect
  "Undefined function or variable 'ALPHATOTAL'". Then `m.train(data,'decalpha',1,'decbeta',1)`.
- **Smallest fix** Set the two defaults back to 1 in `program_option.m`, which restores the known
  good configuration and proves the diagnosis in one line. The larger fix is to stop storing the
  full `m` by `dmax` by `T` arrays at all and accumulate `gamma` inside the backward loop, which
  is what the commented out recursion at `decodelog:75-79` was reaching for. That is a change to
  the numerical core and needs its own verification.

### H2. Models the GUI offers do not implement the interface the inference requires

- **Severity** High. **Difficulty** easy. **Category** DEP, COR. **Confidence** read.
- **Evidence**
  - `uiBSD.m:3941-3952`, `classes_ls`, lists **every** `.m` file in a package folder and offers
    all of them in the model dropdowns, called at `uiBSD.m:1709-1739`.
  - Method inventory, from `grep -o "function [^(]*("` per file:
    - `+emis_model/mar.m` has no `priorfull` and no `expectfun`. `+inference/train.m:35` calls
      `self.emis_model.priorfull()` and `train.m:113` reaches `expectfun`.
    - `+emis_model/mar4.m` has no `trouble`. `+inference/trainn.m:58` calls it every iteration.
    - `+dur_model/categ_dirichlet.m` has no `priorfull` and no `expectfun`.
      `+inference/train.m:41` calls `self.dur_model.priorfull()`.
    - `+dur_model/uniform_aux.m` is a stub with a constructor, an empty `parsamplfun` and
      `sample` only, 27 lines total.
    - `+emis_model/Test_normal_normal_wishart2.m` and `Test_normal_normal_wishart3.m` are
      scripts, they begin `clear all` at line 2, yet they appear in the emission dropdown.
  - `uiBSD.m:1762` and `:1804` show `mar` is a deliberately supported dropdown entry, with an
    extra "Order" field.
- **Condition** Choosing any of the above in the GUI, or passing them to `hsmm(...)` from a
  script. `hsmm.m:27` has `emis_model.mar` commented as the alternative default, so it is an
  invitation.
- **Impact** Immediate "Unrecognized method" error at the start of training, after the user has
  loaded data and set every parameter. A MAR emission model and a non parametric duration model
  are both scientifically interesting options that cannot currently be reached.
- **Reproduce** `m=hsmm(2,4,emis_model.mar(2,4,2)); m.train(data)`.
- **Smallest fix** Filter the dropdowns to classes that implement the required methods, which
  also removes the `Test_*` scripts and `uniform_aux` from the menu. The real fix is `W1`, a base
  class, so that a missing method is a load time error rather than a runtime one.

### H3. The HSMM Viterbi allocates and grows gigabyte scale arrays

- **Severity** High. **Difficulty** easy. **Category** PRF. **Confidence** read, arithmetic shown.
- **Evidence**
  - `+inference/hsmmresidual_logviterbi.m:35-36`: `delta` and `deltaaux` are both
    `nstates` by `ndata` by `dmax`. `deltaaux` is then overwritten by a scalar or a small matrix
    on first use at `:55`, `:58` or `:60`, so that whole allocation is waste.
  - `+inference/hsmmresidual_logviterbi.m:32`: `B(j,:,:) = util.pdfdur(B3(:,j),dmax)` is a third
    array of the same size.
  - `+inference/hsmmresidual_logviterbi.m:84`: `sa(t,j,d,:)=[t-d posy posx]` grows an array of
    `(ndata+dmax-1)` by `nstates` by `dmax` by `3` one element at a time, with no preallocation.
  - `+inference/hsmmresidual_logviterbi.m:58`: `squeeze(delta(:,t-d,:))` copies an `nstates` by
    `dmax` matrix inside the innermost loop.
  - `+inference/hsmmresidual_logviterbi.m:42`: the loop runs to `ndata+dmax-1`, but every write
    to `delta` is gated on `t<=ndata` at `:65`, so the last `dmax-1` sweeps only write zeros into
    `sa`.
- **Condition** Any `viterbi` call on real data. The arithmetic, for the toolbox's own pipeline
  input, `Examples/Example5_EEG.mat` resampled to 128 Hz, so 5780 samples, with `dmax=200` and
  10 states: `delta` is 10 x 5780 x 200 x 8 bytes = 92.5 MB, `deltaaux` another 92.5 MB, `B`
  another 92.5 MB, and `sa` is 5979 x 10 x 200 x 3 x 8 bytes = 287 MB. About 0.56 GB for 45
  seconds of EEG, with roughly 1.2e7 passes through the innermost body. Without the resample, at
  512 Hz and 23120 samples, the same arithmetic gives about 2 GB.
- **Impact** The documented "most probable sequence" step, `Script_Tutorial_basic.m:11`, is
  impractical for recordings of research length, and the growth of `sa` makes it far slower than
  the array sizes alone suggest.
- **Reproduce** Time and watch memory for `m.viterbi(datain)` on the pipeline input with
  `dmax=200`. Run this last, after the cheaper checks.
- **Smallest fix** Two lines: delete the `deltaaux` allocation at `:36`, and preallocate `sa` with
  `zeros(ndata,nstates,dmax,3)` before the loop, and end the `t` loop at `ndata`. That is
  mechanical and keeps the result identical. Reducing the asymptotic memory needs a redesign:
  store only the backpointer and the best score per `(t,j)`, not per `(t,j,d)`.

### H4. The documented pipeline can attach topographies to the wrong state

- **Severity** High. **Difficulty** moderate. **Category** EVAL, SKEW. **Confidence** read.
- **Evidence**
  - `+inference/train.m:121-122`: `out.stateseq=pos2;` is the per sample argmax over the full set
    of `nstates` gamma columns, while `out.nstates=length(unique(pos2))` is the **count of
    distinct labels that won somewhere**.
  - `Script_Tutorial_pipeline.m:23-29`: `nstates2=outhsmm.nstates;` then
    `for j=1:nstates2, Xmat(outhsmm.stateseq==j,j)=1; end`.
  - So the design matrix is built on the assumption that the surviving labels are exactly
    `1:nstates2`. Nothing enforces that.
- **Condition** A fitted model in which some state label below the maximum never wins the argmax,
  which is exactly what happens when a state dies. See `H6`, which makes dying states likely.
  Example: a 6 state fit where labels `{1,2,4,5,6}` survive gives `nstates2=5`; column 3 of
  `Xmat` is then all zeros and state 6 is dropped from the GLM entirely.
- **Impact** A rank deficient design matrix handed to `util.spm.spm_robust_glm`, an empty
  topography plotted for one state, and a real state silently missing from the figure. The failure
  is silent: the figure is produced and looks plausible.
- **Reproduce** After a fit, compare `unique(outhsmm.stateseq)` with `1:outhsmm.nstates`.
- **Smallest fix** In the tutorial, replace the loop bound with the surviving labels:
  `lab=unique(outhsmm.stateseq); for j=1:numel(lab), Xmat(outhsmm.stateseq==lab(j),j)=1; end`.
  Better, have `train` relabel the sequence to be contiguous and record the mapping, so every
  downstream consumer is safe.

### H5. The k means initialisation can index past `dmax` and stop the first M step

- **Severity** High. **Difficulty** moderate. **Category** DAT. **Confidence** read.
- **Evidence**
  - `+util/decodevarinit.m:7`: `durcount=zeros(nstates,dmax)`.
  - `+util/decodevarinit.m:31-34`: `d` counts forward over the run of identical labels, then is
    clamped by `d=min(d,dmax)`.
  - `+util/decodevarinit.m:35-39`: when the run reaches the end of the sequence the count is
    written at `durcount(state,d+1)`, so with a clamped `d` the index is `dmax+1`. MATLAB grows
    the array rather than erroring, giving `nstates` by `dmax+1`.
  - `+util/decodevarinit.m:61`: it is returned transposed, so `dmax+1` by `nstates`.
  - `+inference/trainn.m:65`: `self.dur_model.update(opt.train,(1:opt.dmax)',decodevar.durcount)`
    then pairs a `dmax` by 1 grid with a `dmax+1` row weight vector, and
    `+dur_model/normal_normal_gamma.m:130` multiplies them with `bsxfun`.
- **Condition** `initoption='kmeans'` and a final run of identical cluster labels longer than
  `dmax`. With `dmax=200` at 128 Hz that is 1.6 s, which is ordinary for k means labels on an EEG
  envelope. The random initialisation is safe, because `+util/unirndseq.m:3` caps segments at
  about `0.33*dmax`.
- **Impact** A dimension mismatch error on the very first M step, with a message that points at
  the duration model rather than at the initialisation.
- **Reproduce** `m.train(data,'initoption','kmeans','dmax',50)` on `Example1`, which makes a long
  final run likely, and inspect `size(decodevar.durcount)`.
- **Smallest fix** Clamp after the increment: use `min(d+1,dmax)` at `+util/decodevarinit.m:39`.
  One line, and it keeps the existing truncation convention.

### H6. A hard coded occupancy threshold removes the complexity cost of a dying state

- **Severity** High. **Difficulty** moderate. **Category** EVAL. **Confidence** read, magnitude
  not measured.
- **Evidence**
  - `+emis_model/normal_normal_wishart.m:143`: `if Nk < 15 % extinguished`, where `Nk` is
    `sum(gamma(:,state))`, an absolute sample count. The branch then copies the prior into the
    posterior, `:145-153`.
  - Earlier thresholds are commented out immediately above, `:140-142`, including a percentage
    version, so the constant has been tuned by hand.
  - The sibling duration model uses a different constant for the same idea,
    `+dur_model/normal_normal_gamma.m:153`, `if Nk < 0.001`.
  - When the posterior equals the prior, the KL term for that state is zero by construction, and
    `hsmm.m:161-174` sums exactly those KL terms into the free energy.
  - `+inference/train.m:96-99` chooses the number of states by comparing those free energies and
    stops at the first decrease.
- **Condition** Any fit in which a state falls below 15 expected samples, which is the normal fate
  of surplus states during a state number search.
- **Impact** Two effects, both on the headline output, the number of brain states. First, a dead
  state contributes no complexity penalty, so the free energy loses the mechanism that is supposed
  to stop the search. Second, the threshold is absolute, so the same data at twice the sampling
  rate behaves differently. I can show the mechanism from reading, but not its size without
  running, so this needs measurement before it is quoted.
- **Reproduce** Fit `Example1` with `minstates=2, maxstates=10`, record `ma` from
  `+inference/train.m:96` for each state count, and count states with `Nk<15` at each. If the free
  energy curve is flat or rising at the top of the range, the penalty has indeed gone.
- **Smallest fix** None that is safe without a measurement. Start by logging `Nk` per state per
  iteration, which is also `OBS`: nothing currently records that a state died.

---

## Medium

### M1. The reported free energy is not the free energy

- **Severity** Medium. **Difficulty** easy. **Category** EVAL. **Confidence** read.
- **Evidence** `+emis_model/normal_normal_wishart.m:87` ends with `- n*log(2*pi)`. The Gaussian
  log density constant is `-(n/2)*log(2*pi)`. All four MAR models get it right, for example
  `+emis_model/mar.m:116`, `-d/2*log(2*pi)`, which is good internal evidence that this is a typo
  rather than a convention. The same form appears in `normal_normal_wishart2.m:90`,
  `normal_normal_wishart3.m:90` and `+emis_model/normal_normal_gamma.m:85`.
- **Condition** Always, for the Gaussian emission models, which are the defaults.
- **Impact** The offset is the same for every state and every sample, so it cancels in `gamma` and
  `xi` and does not change which state wins. It does not cancel in `decodevar.loglik`, so the
  number printed each iteration by `trainn.m:92`, stored in `out.fe`, and plotted by the tutorial
  at `Script_Tutorial_basic.m:59`, is shifted by `T*(n/2)*log(2*pi)`. For the pipeline case, 5780
  samples and 10 components, that is about 53,000 nats. Comparisons at fixed dimension and fixed
  length are unaffected, so the chosen number of states is not. Comparisons across datasets or
  dimensions are meaningless.
- **Reproduce** Fit two models that differ only in `ncomp` and compare the free energies, or
  simply evaluate `prob` against `mvnpdf` for a known mean and covariance.
- **Smallest fix** `- n/2*log(2*pi)` in all four Gaussian emission models. Note in the release
  notes that previously published free energy values shift by a constant.

### M2. Documented defaults disagree with the shipped defaults

- **Severity** Medium. **Difficulty** easy. **Category** DOC. **Confidence** read.
- **Evidence** `Script_Tutorial_basic.m:69-75` documents `dmax=100`, `maxcyc=100`, `tol=0.01`,
  `maxstates=15`. `program_option.m:2-4` and `:12` set `maxcyc=30`, `tol=0.1`, `dmax=200`,
  `maxstates=10`.
- **Impact** A user who trusts the comment gets a third of the iterations and a tolerance ten
  times looser than documented. `maxcyc=30` with `tol=0.1` is a plausible cause of "it did not
  converge" reports.
- **Smallest fix** Make the tutorial read the values from `program_option` instead of restating
  them, or delete the list.

### M3. A decreasing free energy is masked, and the reported value is the last, not the best

- **Severity** Medium. **Difficulty** easy. **Category** EVAL, OBS. **Confidence** read.
- **Evidence** `+inference/trainn.m:95-103`: on `dif<0` the code sets `dif=inf` and continues,
  with the diagnostic `disp('ERROR')` commented out. `trainn.m:115-118`: `out.fe` and
  `out.loglik` are taken from `freearray(end)`. `+inference/train.m:63` and `:96` then select
  restarts and state counts by comparing those final values.
- **Condition** Only bites if the free energy is not monotone. The presence of this guard, of the
  `real()` calls at `decodelog:105-108`, and of `M9`, says the authors have seen decreases.
- **Impact** Three things at once. Convergence can never be declared on a decreasing step, so the
  run always exhausts `maxcyc`. The reported free energy may be worse than the best one seen. And
  the clearest available signal that the bound is wrong is discarded instead of logged.
- **Reproduce** Print `[out.fedetail.fe]` and check monotonicity. This is item 2 of the run list
  in `ASSESSMENT.md`.
- **Smallest fix** Keep the guard, but record it: store a `decreased` flag and the best iteration
  on `out`, and report `max` rather than `end`. Three lines, no change to the mathematics.

### M4. `Tolerance` is documented as a percentage and implemented as nats

- **Severity** Medium. **Difficulty** easy. **Category** DOC. **Confidence** read.
- **Evidence** `Manual BSD.pdf` table 6.1 item 5: "Convergence criterion. Minimum percentage
  difference between iteration that define convergence", selectable range "0% to 100%".
  `+inference/trainn.m:96`: `dif=(freearray(cycle).fe-freearray(cycle-1).fe)`, an absolute
  difference, compared against `opt.tol` at `:50`. `uiBSD.m:984` passes the box value straight
  through with no conversion. `Script_Tutorial_basic.m:73` repeats the percentage claim.
- **Impact** A user entering 5, meaning five percent, gets a stopping threshold of 5 nats, which
  on a long recording stops almost immediately. The scale of a sensible value also depends on `T`
  and on `M1`.
- **Smallest fix** Change the manual and the comment, or implement the documented relative test.
  Decide which, since the second changes results.

### M5. The HSMM state posterior is normalised using a single hard coded time point

- **Severity** Medium. **Difficulty** easy. **Category** COR, DAT. **Confidence** read.
- **Evidence** `+inference/hsmmresidual_decodelog.m:86-91`:
  `gamma=squeeze(sum(exp(aux),2))'; nor=sum(gamma(10,:)); gamma=gamma/nor;`. The linear version
  does the same at `hsmmresidual_decode.m:69-71`. For comparison, the HMM decoder normalises every
  row, `hmm_decodelog.m:34`.
- **Condition** Always. Two separate consequences.
  - If `T<10`, `gamma(10,:)` is out of range and the decode stops. A 10 sample block is unusual,
    but the multi block path decodes each block separately, `multimodel.m:22-33`, so a short block
    in a study is enough.
  - Otherwise the rows of `gamma` are only as normalised as the arithmetic at `t=10` happens to
    be. In exact arithmetic the sum is the same at every `t`, which is presumably the reasoning,
    but after `exp` of scaled log quantities it is not, and `gamma` is used directly as the
    responsibility weight in every M step, `trainn.m:57-68`.
- **Impact** A silent, data dependent scale error on all sufficient statistics, and an avoidable
  crash on short blocks.
- **Reproduce** `sum(outhsmm.decodevar.gamma,2)` at several time points. Expect values that differ
  from 1 and from each other.
- **Smallest fix** Normalise each row: `gamma=gamma./sum(gamma,2)`. One line, and it also removes
  the `T<10` trap.

### M6. The GUI's k means iteration box writes a field nothing reads

- **Severity** Medium. **Difficulty** easy. **Category** CFG. **Confidence** read.
- **Evidence** `uiBSD.m:982`: `setfield(obj.study.opt, 'matixer', data.maxiter)`. The consumer is
  `+util/initvb.m:5`, which passes `opt.maxiter` to k means. `program_option.m:7` sets
  `opt.maxiter=50000`. `uiBSD.m:2517` reads the box into `max_iter` and `:2564` stores it, so the
  value travels correctly right up to the misspelling.
- **Impact** The control documented in `Manual BSD.pdf` table 6.1 item 3, "Initialization:
  Iterations", has no effect. Every k means run uses 50000 iterations regardless.
- **Smallest fix** Correct the field name to `maxiter`. One character class of change, and it
  makes a documented control work.

### M7. `States Min = 1` is accepted and then indexes element zero

- **Severity** Medium. **Difficulty** easy. **Category** CFG. **Confidence** read.
- **Evidence** `uiBSD.m:2547-2551` accepts any `min_states >= 1`. `+inference/train.m:69` starts
  the search at `opt.minstates` and `:97` compares `ma(kstate)<ma(kstate-1)`, so with
  `kstate=1` the index is `ma(0)`.
- **Impact** An immediate index error after the user has configured and started a run. A single
  state model is also a reasonable thing for a user to ask for as a baseline.
- **Smallest fix** Either validate `min_states>=2` in the GUI, or guard the comparison with
  `kstate>opt.minstates` in `train.m:97`. The guard is better, since it also protects script users.

### M8. Enabling the plotting options crashes the default emission models

- **Severity** Medium. **Difficulty** easy. **Category** COR. **Confidence** read.
- **Evidence** `+inference/trainn.m:105-111` calls `self.emis_model.graf(opt,decodevar,video)` or
  `graf(opt,decodevar)`. The method inventory shows `graf` exists only on the four `mar*` classes;
  none of the four `normal_normal_*` emission models has it. `program_option.m:25-26` advertises
  `opt.graf` and `opt.video`.
- **Condition** `opt.graf>0` or `opt.video==1` with a Gaussian emission model, that is, with the
  default model.
- **Impact** An "Unrecognized method" error part way through training, after work has been done.
  With `opt.video==1` a half written AVI file is also left open, `trainn.m:44-46`.
- **Smallest fix** Guard the calls with `ismethod(self.emis_model,'graf')`, or add a no op `graf`
  to the Gaussian models.

### M9. The duration E step and M step do not use the same density

- **Severity** Medium. **Difficulty** easy to observe. **Category** COR, EVAL. **Confidence** read.
- **Evidence**
  - `+dur_model/normal_normal_gamma.m:90-92`: the duration log probabilities are renormalised
    over the supplied grid, but only when the grid is all integers,
    `if sum(abs(X-floor(X)))==0`. The decoder always supplies `(1:dmax)'`,
    `decodelog:21`, so in practice the E step always uses a truncated, renormalised distribution
    over 1 to `dmax`.
  - `+dur_model/normal_normal_gamma.m:158-165`: the M step is the standard Normal Gamma conjugate
    update for an **untruncated** Gaussian. Nothing accounts for the truncation.
  - `hsmm.m:161-174`: the free energy adds the KL of that untruncated posterior against its prior.
  - The same pattern is in `+dur_model/lognormal_normal_gamma.m:77-79` and `:128-130`, where the
    author has also left `%Se debe revisar` on the KL line, `:212`.
- **Impact** The variational bound is assembled from two different models of duration, so it is
  not guaranteed to increase, which is the most likely explanation for the masking in `M3`. The
  free energy is then unsafe as a model selection criterion, which is the one job it has here.
- **Reproduce** Item 2 of the run list. If the free energy decreases on some iterations with the
  Gaussian duration model, this is the first suspect.
- **Smallest fix** None is small. The honest short term step is to document the inconsistency and
  stop using the free energy for anything but monitoring. A proper fix means either a truncation
  aware update, or a duration family that is discrete by construction, which
  `+dur_model/categ_dirichlet.m` already is, see question 5 in `ASSESSMENT.md`.

### M10. The duration model's `cleanpos` clears the wrong fields, so training is not idempotent

- **Severity** Medium. **Difficulty** easy. **Category** DAT. **Confidence** read.
- **Evidence** `+dur_model/normal_normal_gamma.m:203-207` sets `self.posterior.mean=[]` and
  `self.posterior.prec=[]`, but the real fields are `mean_normal` and `prec_gamma`, declared at
  `:5`. The same two lines appear in `+dur_model/lognormal_normal_gamma.m:171-176`. The emission
  model gets it right, `+emis_model/normal_normal_wishart.m:195-199`. The effect is to add two
  unused fields and leave the posterior intact.
- **Condition** A second `train` on the same model object, or any use of `copytrain(1,...)`, which
  is how the state number search is meant to reset candidates, `+inference/train.m:74` and `:85`.
- **Impact** The stale duration posterior is then read back as the starting point of the M step,
  `+dur_model/normal_normal_gamma.m:141-143` and `:160`, where `mk` and `Rk` feed the precision
  update directly. Training twice does not give the same answer as training once, and nothing
  reports it. Note `+inference/trainn.m:26` decides whether to initialise by looking only at the
  **emission** posterior, so a stale duration posterior is never noticed.
- **Reproduce** `m.train(data); p1=m.dur_model.posterior; m.train(data); isequal(p1,m.dur_model.posterior)`.
  Also check `fieldnames(m.dur_model.posterior)` for the stray `mean` and `prec`.
- **Smallest fix** Clear the correct field names in both duration models. Two lines each.

### M11. The GUI's model menus depend on Windows `ls` behaviour

- **Severity** Medium. **Difficulty** easy. **Category** DEP. **Confidence** inferred.
- **Evidence** `uiBSD.m:3943-3950`: `elements = ls(complete_path)` then
  `for e=3:size(elements,1)` and `elements(e,:)`. This assumes `ls` returns an m by n character
  matrix with `.` and `..` first, which is its documented Windows behaviour. On UNIX, MATLAB's
  `ls` returns a single character row of names, so `size(elements,1)` is 1 and the loop body never
  runs.
- **Impact** On macOS and Linux every model dropdown is empty, so the GUI cannot create a model.
  The manual, section 2.1, promises "Matlab version 2015 and above" with no platform restriction.
- **Reproduce** `numel(obj.classes_ls('emis_model'))` on a non Windows MATLAB. Labelled inferred
  because I cannot run it here.
- **Smallest fix** Use `dir([ '+' path '/*.m'])` and take `{d.name}`. Platform independent and
  shorter.

### M12. The electrode file chooser probably returns the MATLAB search path

- **Severity** Medium. **Difficulty** easy. **Category** FE. **Confidence** inferred.
- **Evidence** `uiBSD.m:3761` declares `function g = select_eeg_coords(obj)`. `path` is assigned
  only inside two sibling nested functions, `:3781` in `get_eeg_file` and `:3790` in
  `browse_action`, and is read at `:3825` in a third, `ok_action`, as `g = path;`. MATLAB shares a
  variable between nested functions only when it also exists in the parent's scope. `g` does,
  because it is the output argument. `path` does not appear in the parent body. If that reading is
  right, `path` in `ok_action` resolves to the built in function and `g` becomes the whole MATLAB
  search path.
- **Impact** `uiBSD.m:1289` and `:1563` then test `isempty(ruta)`, which passes, and hand the
  value to `topoplot(topo, ruta)` at `:1321` and `:1593`. The topography view fails, or draws with
  the wrong montage, after a dialog that appeared to work.
- **Reproduce** One line: `dbstop at 3825`, or `disp(class(g)); disp(numel(g))` after the dialog.
- **Smallest fix** Initialise `path=''` in the parent function body, immediately after line 3761,
  so the variable is genuinely shared. One line, and it settles the question at the same time.
- **Related** `uiBSD.m:34` offers `ele_bio128.xyz`, which is not in the repository. The two files
  present are `ele_bio64.xyz` and `bnet_ele128.xyz`.

### M13. The random initialisation can loop forever

- **Severity** Medium. **Difficulty** moderate. **Category** PRF, DAT. **Confidence** read.
- **Evidence** `+util/initvb.m:19-21`: `while(length(unique(stateseq))~=self.nstates)` retries
  `util.unirndseq` until every state appears. `+util/unirndseq.m:3` draws segment lengths from
  roughly `0.075*dmax` to `0.33*dmax`, so a sequence of `ndata` samples holds at most about
  `ndata/(0.075*dmax)` segments.
- **Condition** `nstates` greater than the number of segments the generator can fit, that is,
  short records with a large `dmax` and a wide state search. With `dmax=200` and 1000 samples the
  ceiling is about 66 segments, which is safe, but `dmax=200` on a 300 sample block with 10 states
  is not.
- **Impact** The run hangs with no message and no progress output, inside a `parfor` in the
  parallel case, `+inference/train.m:73`, where it is harder still to diagnose.
- **Smallest fix** Cap the retries and raise a clear error naming `ndata`, `dmax` and `nstates`.
- **Note, not a defect** The same function ties the initial segment length to `dmax`, as do the
  duration priors, `+dur_model/normal_normal_gamma.m:187-188`. So `dmax` influences the result
  through three separate channels. Worth documenting for users who sweep it.

### M14. Third party code is vendored without a licence or a dependency note

- **Severity** Medium. **Difficulty** easy. **Category** DEP, DOC. **Confidence** read.
- **Evidence**
  - `+util/kmeans2015.m:152`: "Copyright 1993-2014 The MathWorks, Inc.", 882 lines, the
    Statistics Toolbox `kmeans` with its help text intact. Line 1 declares
    `function [...] = kmeans2(...)`, so the file name and function name also disagree.
  - `+util/+spm/*.m` and `+util/+Normal/spm_kl_normal.m:12`: "Copyright (C) 2008 Wellcome Trust
    Centre for Neuroimaging", with SPM revision ids still in place.
  - `+util/logsumexp.m:8-9`: "Written by Tom Minka, (c) Microsoft Corporation".
  - `+util/+Gamma/klgamma.m:21`: "Copyright (C) 2001 Matthew J. Beal".
  - Binary artefacts with no provenance: `+util/+Normal/KL.zip`, `+util/sparseinv.mltbx`.
  - There is no `LICENSE` and no `README.md`. `README.md` was deleted in commit `9129627`.
  - EEGLAB is required at `Script_Tutorial_pipeline.m:34` and `uiBSD.m:1321`, `topoplot`. That
    dependency **is** documented, in `Manual BSD.pdf` section 2.1, but only there.
- **Impact** Redistribution risk that is not the reviewer's call to resolve, and a new user who
  clones the repository has no way to learn what else must be on the path. See question 7 in
  `ASSESSMENT.md`.
- **Smallest fix** A `README.md` with dependencies and a licence statement, and either remove the
  vendored MathWorks file in favour of a call to `kmeans`, or document the permission to ship it.

### M15. No seed control, so no run can be reproduced exactly

- **Severity** Medium. **Difficulty** easy. **Category** OBS, EVAL. **Confidence** read.
- **Evidence** `grep -rn "rng(" --include=*.m .` finds nothing in the toolbox's own code; the only
  hit anywhere is `+util/kmeans2015.m:339`, which reads the global stream rather than setting it.
  `program_option.m` has no seed field. The default initialisation is random,
  `program_option.m:8`, and draws through `+util/unirndseq.m:4,7,11`. The k means path is also
  random, with `opt.nrep=10` replicates, `program_option.m:6`.
- **Condition** Always.
- **Impact** Two identical calls give different answers, and a figure in a paper cannot be
  regenerated. It also makes three whole tiers of testing impossible to write: synthetic recovery,
  model order calibration and golden masters all need a pinned starting point. See
  `VALIDATION.md` section 13.
- **Reproduce** `m=hsmm(2,4); a=m.train(data); m2=hsmm(2,4); b=m2.train(data); a.fe==b.fe`.
  Expect false.
- **Smallest fix** Add `opt.seed=[]` to `program_option.m` and
  `if ~isempty(opt.seed), rng(opt.seed); end` at the top of `+inference/trainn.m`. Two lines, and
  it unblocks the test harness. `+util/initvb.m:31-52` already accepts a stored sequence through
  `initoption='seqprior'`, which covers the golden master case even before a seed exists.

---

## Low

### L1. `in_model.priorfull` tests the posterior

`+in_model/categ_dirichlet.m:81-86` returns 0 when `posterior.categ_dirichlet.conc` is empty,
where every sibling class tests the prior, for example
`+trans_model/categ_dirichlet_matrixdiag0.m:79-84`. Called at `+inference/train.m:49`. Currently
masked: on the live path the posterior is empty exactly when the prior needs filling, so the
answer is right by accident. It becomes wrong for any object that has a posterior but no prior,
and `update` at `:62` would then add to an empty prior. `COR`, easy, read. One word fix.

### L2. `in_model.prob` uses an undefined variable

`+in_model/categ_dirichlet.m:42`: `p=exp(p)` where `p` has not been assigned, presumably
`exp(plog)`. The method is not called anywhere in the repository, so it is dead today. `COR`,
easy, read.

### L3. The lognormal duration model's EM branch iterates one state

`+dur_model/lognormal_normal_gamma.m:66`: `for state=k1:k1` where every sibling uses `k1:k2`. Only
the `VB` branch is reachable from `train`, since `program_option.m:1` sets `opt.train='VB'`, so
this is dead today and would silently return one column if `EM` were ever enabled. `COR`, easy,
read.

### L4. The lognormal initialisation takes a square root before inverting

`+dur_model/lognormal_normal_gamma.m:124`: `Sk = sqrt(bsxfun(@times,XC,gamma)'*XC / Nk)` then
`Rk = inv(Sk)`, so the initial precision is one over the standard deviation, not one over the
variance. The sibling Gaussian model has no `sqrt`,
`+dur_model/normal_normal_gamma.m:148-150`. First M step only, and it feeds the precision update
at `:129`. `COR`, easy, read.

### L5. A dead store assigns a mean where a precision is read

`+emis_model/normal_normal_wishart.m:131`: `Rk=self.posterior.mean_normal{state}.mean;` where the
duration model's equivalent line correctly reads `.prec`,
`+dur_model/normal_normal_gamma.m:143`. `mk` and `Rk` are not used afterwards in this class, since
both branches below overwrite the posterior, so it is latent. It is worth fixing because the
pattern is copied across four classes and in `wishart2` the same slot **is** live. `COR`, easy,
read.

### L6. `pdfdur` pads missing emissions with a log probability of zero

`+util/pdfdur.m:3-5`: the accumulator starts as `zeros(T,durmax)` and the recursion at `:10` reads
`re(t-1,d-1)`. For `d>t` the window reaches before the start of the record and the missing
emissions contribute log probability 0, that is, probability 1, rather than minus infinity. The
effect is to inflate the score of long durations in the first `dmax` samples of the Viterbi path.
`re(2,:)=0` at `:5` is also overwritten immediately by the loop. `COR`, easy, read.

### L7. The initialisation never counts the first segment's duration

`+util/decodevarinit.m:21-22`: duration counts are accumulated only at a label change, and the
count that is written is for the **following** segment, `:29`, `:39`, `:44`. The first segment of
the record therefore contributes no duration count. Initialisation only, so the effect decays, but
it biases the first duration M step for short records. `COR`, easy, read.

### L8. `trainn` calls a package that does not exist

`+inference/trainn.m:11`: `aux_func.defaultopt;` and there is no `+aux_func` folder in the
repository. The branch is reached when `trainn` is called with name and value pairs rather than an
options struct, which is what the public signature at `hsmm.m:96` invites. `train` always passes a
struct, and `+util/initvb.m:48` also passes a struct, so nothing on the documented path reaches
it. `COR`, easy, read.

### L9. `Example4`'s ground truth is two samples shorter than its data

Measured from the MAT headers: `Examples/Example4_Sourcesconn.mat` holds `data` 15000 by 2 and
`stateseq` 1 by 14998. The tutorial scoring line, `Script_Tutorial_basic.m:14`, compares the two
elementwise and divides by `length(data)`, which cannot work for this file. The deleted
`Script_Tutorial4.m`, commit `de6d456`, presumably handled it. `TST`, easy, read.

### L10. The accuracy helper can collide labels and silently mis-score

`+util/renameseq.m` is what produces the error percentage in the tutorial,
`Script_Tutorial_basic.m:13-14`. Three problems. `:13` histograms the true labels over
`0:1:10`, hard coded, so any label above 10 is folded into the last bin. `:20-26` assigns
unmatched estimated states the labels 10, 9, 8 and so on, which can collide with genuine labels,
and prints `Revisar` rather than warning properly. `:6` sizes `seq` by the number of unique true
labels and then indexes it by time, relying on MATLAB growing the array. It is a greedy matcher,
not an optimal assignment, so the score also depends on the order states happen to be processed.
`EVAL`, easy, read.

### L11. `eta` is never returned, even when it is requested

`+inference/hsmmresidual_decodelog.m:111` sets `decodevar.eta=[]` and the assignment at
`:118-120` that would fill it is commented out, while `:69-71` still computes and accumulates it
when `opt.deceta==1`. So asking for `eta` costs the memory and returns nothing.
`+util/decodevarinit.m` still has four `deceta` branches that fill it. `OBS`, easy, read.

### L12. The HSMM Viterbi backtrace guard does not do what it reads as

`+inference/hsmmresidual_logviterbi.m:101`: `while (count & dm) > 0` evaluates a logical AND and
then compares it with zero, so it is true whenever both are non zero, and it does not stop on a
negative `count`. Meanwhile `:76-79` deliberately stores a zero sentinel for a dead path, and the
next pass indexes `sa(count+dm,jm,dm,3)` with `jm=0`, which is an invalid index rather than a
clean failure. `COR`, moderate, read.

### L13. The HSMM Viterbi charges a transition into the first state

`+inference/hsmmresidual_logviterbi.m:55`: at `t==d` the score is
`ini(1,j)+A(:,j)+pdur(d,j)+B(j,t,d)`, so entering state `j` at the start is charged a transition
probability as well as the initial probability, and `:72-75` then maximises over the previous
state. The commented line above shows the same intent. `delta(:,1,1)=ini` at `:38` is overwritten
by the loop. Affects the first segment only. `COR`, moderate, read.

### L14. A model that reports trouble loses the free energy fields

`+inference/trainn.m:58-61`: on `self.emis_model.trouble()` the code writes
`freearray(cycle).fe=-inf` and breaks. If that happens on the first cycle, `freearray` has only
the field `fe`, and `:116-117` then read `freearray(end).loglik` and `.totaldivkl`, which do not
exist. If it happens later those fields are left empty. Currently latent, because
`+emis_model/normal_normal_wishart.m:231-233` returns 0 unconditionally, so only the `mar` models
can trigger it. `COR`, moderate, read.

---

## Decoration

Cosmetic or dead. These change nothing. Listed so that `Low` does not have to carry them.

### D1. Dead computation inside the backward loop

`+inference/hsmmresidual_decodelog.m:75-76` computes `XIaux1` and `XIaux2`, which are never read,
because the two lines that consumed them are commented out at `:78-79` with the note
"Temporalmente se modifica". `GAMMATOTAL` is written once at `:47` and never read. The cost is one
`logsumexpmat` and one `logsumexp` per time step, inside the E step, which runs every iteration.

### D2. Dead stores with a wrong index in the transition sampler

`+trans_model/categ_dirichlet_matrixdiag0.m:24-25`, `:36-37`, `:42-44`: `aux` is a row vector and
`aux(j,j)=0` writes at a two dimensional position, which grows `aux` into a `j` by `j` matrix for
`j>1`. The variable is then never used in those three branches, so nothing breaks. Option 2 does
it correctly, `aux(1,j)=0` at `:31`.

### D3. Struct access built with `eval`

`+util/outmatr.m:5-7` and `+util/inmatr.m:7`, `:15` build field access expressions as strings and
`eval` them, inside loops that run on every VB iteration of the multi block path,
`+inference/trainn.m:170-196`. Dynamic field names would do the same job and would be visible to
the debugger. Not a security issue, the inputs are loop indices.

### D4. Leftovers that make it hard to tell what is live

`+util/initvb_old.m`, `+util/miplot3 - copia.m`, `+util/topograph.asv`, an editor autosave,
`+util/+Normal/KL.zip`, `+util/sparseinv.mltbx`, and `EEG_data`, a 2 byte empty file added in the
two most recent commits. Four `mar` variants and four Gaussian emission variants with no note on
which is current. Comments are mixed Spanish and English, including UI strings,
`uiBSD.m:3804`, `'<Seleccionar archivo>'`, in an otherwise English interface.

### D5. `groupindex` writes its flag at the wrong index

`+util/groupindex.m:9` reads `flag(mcond,msubj,mblock)`, the mapped index, while `:12` writes
`flag(kcond,ksubj,kblock)`, the original. They coincide for the first member of every group
because the loops start at 1, so grouping is correct today. A latent bug with no current effect.

### D6. Diagnostics by side effect

`+inference/train.m:70` is a bare `kstate` with no semicolon, used as progress output.
`+inference/hsmmresidual_decodelog.m:100-102` and `hsmmresidual_decode.m:86-88` use a bare string
`'loglik indefinida'` as a warning. `uiBSD.m:1782`, `2251`, `2510` and `3019` are `obj.my_warning('TODO help')`. Use
`fprintf` or `warning` with an identifier, so output can be suppressed and found.

### D7. Names that disagree with their files, and scripts inside packages

`+util/kmeans2015.m:1` declares `function kmeans2`. `+emis_model/Test_normal_normal_wishart2.m`
and `Test_normal_normal_wishart3.m` are scripts, they start with `clear all`, inside a package
folder, where MATLAB will not let them be called with the package prefix. They are also picked up
by the GUI's class listing, see `H2`.

---

## Wish

Not defects. Improvements someone might want.

### W1. One abstract base class per sub-model family

`emis_model.base`, `dur_model.base` and so on, declaring the fifteen method interface the
inference actually uses. A model missing a method would then fail at construction with a clear
message instead of mid training, which is `H2`, and new contributors would have a checklist.

### W2. A Viterbi that fits in memory

Either the mechanical fix in `H3`, or a version that keeps only the best score and backpointer per
`(t,state)` and reconstructs durations on the way back, with an option to process in overlapping
windows for long recordings.

### W3. A minimal regression test

One script that fits `Examples/Example1_data2dim.mat` with a fixed seed and asserts three things:
the free energy is non decreasing, the labelling error is below a recorded threshold, and the
recovered state count equals four. That single test would have caught `H1`, `M3` and `M9`.
There is currently no test harness, and the two `Test_*` files are unreachable, see `D7`.

### W4. A README

Dependencies, including EEGLAB, the MATLAB release, the licence position on the vendored files,
and five lines of quick start. The manual is good but it is a PDF, and `README.md` was deleted in
commit `9129627`.
