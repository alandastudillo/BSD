# Specification: contracts and algorithms

The interfaces, data formats and algorithms of the BSD toolbox, as built. 15 October 2026.
Standing: see [`README.md`](README.md). This is a description of the existing contract, including
the parts of it that are wrong, which are marked.

---

## 1. Data contracts

### 1.1 Single block input

```
data : double matrix, T × ndim
```

Rows are time, columns are channels or components. No missing value convention exists: `NaN`
propagates silently to the log likelihood, `+inference/hsmmresidual_decodelog.m:100-102`.

Verified sizes of the shipped examples, read from the MAT headers:

| File | `data` | ground truth | note |
|---|---|---|---|
| `Example1_data2dim.mat` | 5003 × 2 | `stateseq` 1 × 5003 | consistent |
| `Example2_Topography.mat` | 4000 × 128 | `stateseq` 1 × 4000 | consistent |
| `Example4_Sourcesconn.mat` | 15000 × 2 | `stateseq` 1 × 14998 | **inconsistent** → `../review/ISSUES.md L9` |
| `Example5_EEG.mat` | `dataeeg64` 23120 × 64 | none | plus `chanlocs` 1 × 64 |
| `Example3_Multiblock.mat` | study struct | struct | the multi block example |

### 1.2 Study input, multi block

```
X.cond(c).subj(s).block(b).data : double matrix, T_b × ndim
```

Exactly three nesting levels, in that order, 1 based and contiguous. `+util/addmodel.m:2-20`
discovers the shape by `size(...,2)` at each level, so a gap in the indices is not detected.

`ndim` must be the same for every block. Nothing checks it.

### 1.3 Model object

A handle object of class `hsmm` or `hmm`, carrying `ndim`, `nstates` and the four sub-models.
`nstates = 0` means "unknown, search for it".

### 1.4 Output of `train`

```
out.fe          scalar      free energy of the selected fit, last iteration  → M3
out.loglik      scalar      expected log likelihood term
out.totaldivkl  scalar      minus the sum of the four divergences
out.fedetail    1 × niter struct array, fields:
                  fe, loglik, totaldivkl,
                  divklobs, divklin, divkldur, divkltrans
out.decodevar   struct, see 1.5
out.niter       scalar      iterations actually run
out.stateseq    1 × T       integer labels, or a nested struct for a study
out.nstates     scalar      count of DISTINCT labels present, not the model's state count → H4
```

Additional returns of `train`: `sim_array`, every restart's `out`; `model`, every restart's model;
`modelstruct` and `modelstructarray` for the study path. All are retained until `train` returns.
→ `../review/MEMORY.md` section 3.

### 1.5 `decodevar`

| Field | Shape | Meaning | Note |
|---|---|---|---|
| `gamma` | `T × m` | per sample state posterior | rows do not reliably sum to one → `M5` |
| `xi` | `m × m × T` | per sample transition posterior | emptied on return in the single block path, `trainn.m:119`, retained in the study path |
| `durcount` | `dmax × m` | expected entries into each state at each duration | |
| `Emi` | `m × T` | log emission densities | the transposed `B` |
| `scale` | `1 × T` | per step scale factors, `exp(c)` | |
| `loglik` | scalar | `sum(-c)` | shifted by a constant → `M1` |
| `alpha`, `beta`, `eta` | `[]` | always empty | → `L11` |

### 1.6 Study output

The same fields, nested: `out.decodevar.cond(c).subj(s).block(b).decodevar`, and
`out.stateseq.cond(c).subj(s).block(b).stateseq`. Set at `+inference/train.m:124-133`.

---

## 2. The option set

Every option, its default, its consumer, and whether the manual documents it. Defaults are from
`program_option.m`, which is a **script** executed inside the caller's scope, not a function.

| Option | Default | Type | Consumed at | Documented |
|---|---|---|---|---|
| `train` | `'VB'` | `'VB'` or `'EM'` | every `prob`, `update`, `expect` | manual table 6.1 item 1 |
| `maxcyc` | `30` | integer | `trainn.m:50` | item 6 |
| `tol` | `0.1` | double, nats | `trainn.m:50`, `:96` | item 5, **as a percentage** → `M4` |
| `dmax` | `200` | integer samples | decoders, duration priors, initialisation | item 8 |
| `nrep` | `10` | integer | `initvb.m:5`, k means replicates | item 4 |
| `maxiter` | `50000` | integer | `initvb.m:5`, k means iterations | item 3, **but the GUI writes `matixer`** → `M6` |
| `initoption` | `'random'` | `'random'`, `'random2'`, `'kmeans'`, `'seqprior'`, `'seqprior_mar'` | `initvb.m:3-59` | item 2, only two of the five |
| `maxitersim` | `1` | integer | `train.m:53`, `:58` | item 7 |
| `minstates` | `2` | integer | `train.m:69` | item 9 |
| `maxstates` | `10` | integer | `train.m:69` | item 9 |
| `seq` | `'maxgamma'` | `'maxgamma'` or `'viterbi'` | `train.m:118-151` | section 6.2, indirectly |
| `verbose` | `'yes'` | `'yes'` or `'no'` | `trainn.m:88` | item 10 |
| `prepro` | `'nothing'` | `'nothing'` or `'meanvarnor'` | `trainn.m:18-21` | no |
| `decalpha` | `0` | 0 or 1 | `decodelog:37` | no, **and 0 breaks the decoder** → `H1` |
| `decbeta` | `0` | 0 or 1 | `decodelog:48`, `:58` | no, same |
| `deceta` | `0` | 0 or 1 | `decodelog:69`, `decodevarinit.m:18` | no, and the output is never returned → `L11` |
| `parallel` | `0` | 0 or 1 | `train.m:21`, `:53`, `:73` | tutorial comment only |
| `prior` | `'databased'` | `'default'` or `'databased'` | `train.m:37`, `:42` | section 5.2, indirectly |
| `shareemis` | `[1 0 1]` | 3 logicals, cond subj block | `trainn.m:136` | section 3.2 |
| `sharetrans` | `[1 0 1]` | as above | `trainn.m:137` | section 3.2 |
| `sharein` | `[0 1 1]` | as above | `trainn.m:138` | section 3.2 |
| `sharedur` | `[1 0 1]` | as above | `trainn.m:139` | section 3.2 |
| `graf` | `0` | integer | `trainn.m:108` | no, **and it crashes the default models** → `M8` |
| `video` | `0` | 0 or 1 | `trainn.m:43`, `:105` | no, same |
| `file` | `'kk'` | string | `trainn.m:44`, the video filename | no |
| `seqs` | absent | matrix or cell | `initvb.m:32`, `:46`, `:59` | no, required by `initoption='seqprior'` |
| `seed` | **absent** | — | — | no → `M15` |

Override mechanism: name and value pairs on `train`, applied by `setfield` at `train.m:32-34`,
**after** the defaults are loaded. Passing `'opt', struct` replaces the whole set, `train.m:12-17`.

---

## 3. The sub-model interface

This is the extension contract. A new emission or duration model must implement all of it. There
is no base class and nothing checks conformance. → `../review/ISSUES.md H2`, `ADR-0005`

| Method | Signature | Returns | Called from |
|---|---|---|---|
| constructor | `obj(ndim, nstates)`, or `(ndim, nstates, order)` for MAR, `(1, nstates)` for duration | object | `hsmm.m:25-43`, `copytrain` |
| `prob` | `[p, plog] = prob(opttrain, X)` or `(opttrain, X, state)` | `p`, `plog`: `T × m`, or `T × 1` for one state | decoders, Viterbi |
| `expect` | `[p, plog] = expect(opttrain)` | transition: `m × m`; initial: `1 × m`. `plog` is the expected log | decoders |
| `update` | `update(opttrain, X, stats)` or `(opttrain, X, stats, state)` | nothing, mutates `posterior` | `trainn.m:57-68` |
| `priornoinf` | `priornoinf(type)` or `(type, data)` or `(type, dmax)` | nothing, sets `prior` | `train.m:35-51`, `hsmm.m:105-117` |
| `priorfull` | `re = priorfull()` | logical | `train.m:35`, `:41`, `:46`, `:49` |
| `posteriorfull` | `re = posteriorfull()` | logical | `trainn.m:26` |
| `divklfun` | `divklfun()` | nothing, sets `divkl` | `hsmm.m:161-174` |
| `expectfun` | `expectfun()` | nothing, sets `expectation` | `train.m:113` |
| `cleanpos` | `cleanpos()` | nothing, empties `posterior` | `copytrain` |
| `copy` | `copy(other)` | nothing, value copies all fields | `hsmm.m:118-125` |
| `sample` | `re = sample(num, state)`, plus `dataant` for MAR | `num × ndim`, or durations | `hsmm.m:45-79` |
| `parsamplfun` | `parsamplfun(option)` | nothing, sets `parsampl` | `hsmm.m:80-85` |
| `trouble` | `flag = trouble()` | logical | `trainn.m:58` |
| `ndatafun` | `n = ndatafun(X)` | integer | decoders, Viterbi |
| `graf` | `graf(opt, decodevar)` or `(opt, decodevar, video)` | optional handle | `trainn.m:105-111` |

`parsamplfun(option)` codes, consistent across classes:

| `option` | Meaning |
|---|---|
| 1 | sample parameters from the posterior |
| 2 | sample parameters from the prior |
| 3 | set parameters to the posterior expectation |
| 4 | set parameters to the prior expectation |

### 3.1 Which statistic each `update` expects

| Sub-model | Third argument | Shape |
|---|---|---|
| emission | `decodevar.gamma` | `T × m` |
| duration | `decodevar.durcount`, with `X = (1:dmax)'` | `dmax × m` |
| transition | `decodevar.xi` | `m × m × T` |
| initial state | `decodevar.gamma(1,:)` | `1 × m` |

This is the contract that makes the memory problem structural: it demands per sample arrays where
the mathematics needs only aggregates. → `../review/MEMORY.md` section 4,
`adr/0015-change-the-update-contract-from-per-sample-arrays-to.md`

---

## 4. Algorithm specifications

Notation: `m` states, `T` samples, `dmax` maximum duration, `A` the expected log transition matrix,
`π` the expected log initial distribution, `B(i,t)` the expected log emission density of sample `t`
under state `i`, `P(i,d)` the expected log probability that a visit to `i` lasts `d`.

### 4.1 HSMM E step, residual time formulation

State variable: the pair (current state, samples remaining). Forward array `ALPHA(i,d)`.

```
ALPHA ← π broadcast across all d                      % decodelog:28, and see the note below
for t = 1 … T
    enter(i,d)    ← logsumexp_j( A(j,i) + ALPHA(j,1) ) + B(i,t) + P(i,d)
    continue(i,d) ← ALPHA(i,d+1) + B(i,t)             % d = dmax has no successor: −inf
    ALPHA(i,d)    ← logaddexp( enter(i,d), continue(i,d) )
    c(t)          ← − logsumexp over all (i,d) of ALPHA
    ALPHA         ← ALPHA + c(t)                      % rescale
    ALPHAx(:,t)   ← ALPHA(:,1)                        % the "about to leave" column
end
loglik ← Σ_t −c(t)
```

```
BETA ← [0, −inf × (dmax−1)]
for t = T−1 … 1
    y(i)        ← B(i,t+1) + c(t+1)
    z(i)        ← y(i) + logsumexp_d( P(i,d) + BETA(i,d) )
    zdur(i,d)   ← y(i) + P(i,d) + BETA(i,d)
    BETA(i,d)   ← y(i) + BETA(i,d−1)        for d ≥ 2
    BETA(i,1)   ← logsumexp_j( A(i,j) + z(j) )
    XI(i,j,t)   ← ALPHAx(i,t) + z(j) + A(i,j)
    durcount(d,j) ← logaddexp( durcount(d,j),
                      logsumexp_i( ALPHAx(i,t) + A(i,j) + zdur(j,d) ) )
end
gamma(t,i) ← Σ_d exp( ALPHA_t(i,d) + BETA_t(i,d) ),  normalised
```

Source: `+inference/hsmmresidual_decodelog.m:30-99`. The `durcount` recurrence is implemented with
a `repmat`, `repelem`, `reshape`, `permute` chain at `:61-72`; I verified by hand that it computes
the expression above.

Two specified behaviours that are **not** what the code does:

- `gamma` is normalised **per sample** in the specification above. The code divides the whole
  matrix by its column sum at `t = 10`. → `M5`
- The forward initialisation above matches the code, and it is not a standard convention: every
  residual duration cell starts at `π(i)` rather than at `π(i) + P(i,d)`, so the first segment's
  duration is not drawn from the duration model and the initial mass sums to `dmax`. The code has
  the conventional alternative commented out one line above. This needs an author's decision, not a
  reviewer's. → `../review/ASSESSMENT.md` question 2, `../review/VALIDATION.md` tier 2

### 4.2 M step

```
emission:    Nk(k) = Σ_t gamma(t,k)
             if Nk(k) < 15   → posterior ← prior            % the extinction rule, H6
             else            → conjugate Normal-Wishart update from
                               Σ_t gamma(t,k), Σ_t gamma(t,k) y_t, Σ_t gamma(t,k) y_t y_tᵀ
duration:    conjugate Normal-Gamma update from durcount, over the grid 1 … dmax
transition:  posterior concentration ← prior + Σ_t xi(:,:,t), then zero the diagonal
initial:     posterior concentration ← prior + gamma(1,:)
```

Source: `+emis_model/normal_normal_wishart.m:107-167`,
`+dur_model/normal_normal_gamma.m:120-168`, `+trans_model/categ_dirichlet_matrixdiag0.m:58-66`,
`+in_model/categ_dirichlet.m:55-65`.

### 4.3 Free energy

```
fe = loglik − KL(emission) − KL(duration) − KL(transition) − KL(initial)
```

`hsmm.m:161-174`. For a study, the log likelihoods of the blocks are summed and the divergences of
the **distinct** sub-models are summed, `multimodel.m:70-119`, which is the correct accounting for
tied parameters.

### 4.4 Training loop

```
decodevar ← from the initial hard sequence, or from decode() if a posterior already exists
dif ← ∞; cycle ← 0
while dif > tol and cycle < maxcyc
    cycle ← cycle + 1
    M step for all four sub-models
    if emission reports trouble → record −inf and break
    decodevar ← E step
    fe(cycle) ← free energy
    if cycle > 1
        dif ← fe(cycle) − fe(cycle−1)
        if dif < 0 → dif ← ∞           % masks the failure, M3
```

`+inference/trainn.m:50-112`.

### 4.5 Number of states

```
if nstates > 0
    for j = 1 … maxitersim: fit, record fe
    keep argmax fe
else
    for K = minstates … maxstates
        for j = 1 … maxitersim: fit with K states, record fe
        ma(K) ← best fe at K
        if ma(K) < ma(K−1) → break and keep K−1
```

`+inference/train.m:45-110`. Note `ma(K−1)` is evaluated with no lower guard, so `minstates = 1`
indexes element zero. → `M7`

### 4.6 HSMM most probable sequence

```
u(i,t,d) ← Σ_{s=t−d+1}^{t} B(i,s)          % cumulative emission log density over a window
delta(j,t,d) ← max over (i,h) of [ delta(i,t−d,h) + A(i,j) + P(j,d) + u(j,t,d) ]
backtrace from argmax_{j,d} delta(j,T,d)
```

`+inference/hsmmresidual_logviterbi.m`, with `u` built by `+util/pdfdur.m`. Three deviations from
the specification above: the window sum pads with log probability zero before the start of the
record, `L6`; the first segment is charged a transition probability, `L13`; and the backtrace guard
does not stop on a negative index, `L12`.

---

## 5. Adding a new emission model, worked contract

The toolbox's main extension point. To add one:

1. Create `+emis_model/<name>.m` as `classdef <name> < handle`.
2. Declare the properties `prior`, `posterior`, `parsampl`, `eelog`, `expectation`, `divkl`,
   `ndim`, `nstates`. The containers index priors and posteriors as cell arrays over states, and
   the code tests `size(self.prior.mean_normal,2) > 1` to decide whether priors are per state or
   shared, for example `normal_normal_wishart.m:118-122`.
3. Implement every method in section 3, the constructor included. Missing any of them fails at runtime, not at
   construction.
4. The constructor must tolerate `nstates = 0` and empty `ndim`, because `hsmm` constructs
   sub-models before the state count is known, `hsmm.m:25-43`.
5. `prob` must return the **log** density as its second output. The log decoders use only that
   one, `decodelog:18`.
6. `update` must tolerate an all zero responsibility column. The convention in the existing code is
   to add `1e-10` to the occupancy, `normal_normal_wishart.m:116`.

Known traps, from the existing implementations: the Gaussian constant is wrong in all four
Gaussian models and right in all four MAR models, `M1`; `cleanpos` must clear the real field
names, `M10`; and `trouble` must exist even if it always returns false, `L14`.
