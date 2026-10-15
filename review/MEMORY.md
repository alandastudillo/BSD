# Memory behaviour, and the effort to fix it

Why a group study exhausts memory, with the arithmetic, and a costed plan to fix it. 15 October
2026, branch `rev/review`.

**Nothing here was measured.** MATLAB was unavailable, so every number below is arithmetic from
array shapes read in the source, at 8 bytes per double. Shapes are `read`; the totals are
arithmetic on those shapes; the effort figures are estimates. Section 7 says how to measure each
claim, and that should be done before any of it is quoted.

---

## 1. The reference workload

Taken from the question asked: 28 participants, one two minute EEG block each, processed the way
`Script_Tutorial_pipeline.m` processes its example.

| Quantity | Symbol | Value | Source |
|---|---|---|---|
| Participants, one block each | `nb` | 28 | the stated workload |
| Block length | | 120 s | the stated workload |
| Sampling rate after resampling | | 128 Hz | `Script_Tutorial_pipeline.m:9` |
| Samples per block | `T` | 15,360 | 120 × 128 |
| Samples, all blocks concatenated | `Tall` | 430,080 | 28 × 15,360 |
| Data dimension after PCA | `n` | 10 | `Script_Tutorial_pipeline.m:6` |
| Maximum duration | `dmax` | 200, that is 1.56 s | `Script_Tutorial_pipeline.m:5` |
| States | `m` | 6 assumed, search 2 to 10 | `program_option.m:11-12` |
| Restarts per state count | `maxitersim` | 1 by default, 5 is a common choice | `program_option.m:10` |

Two usage patterns matter, and they fail differently.

- **Pattern A, multi block.** The study struct, `X.cond(c).subj(s).block(b).data`, which is what
  the GUI builds and what the sharing feature needs. Each block is decoded separately.
- **Pattern B, concatenated.** One long matrix of 430,080 samples, which is what a researcher does
  when they just want one model for everything. Nothing in the toolbox discourages it.

---

## 2. Where the memory goes, per block

Everything in this section is in `+inference/hsmmresidual_decodelog.m`, one E step, one block.

| Array | Shape | Bytes | At `m=6`, `T=15360`, `dmax=200` |
|---|---|---|---|
| `ALPHATOTAL` | `m × dmax × T` | `8·m·dmax·T` | **147.5 MB** |
| `BETATOTAL` | `m × dmax × T` | `8·m·dmax·T` | **147.5 MB** |
| `aux = ALPHATOTAL + BETATOTAL` | `m × dmax × T` | a third copy | **147.5 MB** |
| `exp(aux)`, a temporary inside `sum(exp(aux),2)` | `m × dmax × T` | a fourth copy | **147.5 MB** |
| `eta`, only if `opt.deceta=1` | `dmax × m × T` | a fifth copy | 147.5 MB |
| `XITOTAL` | `m × m × T` | `8·m²·T` | 4.4 MB |
| `B` and `B2` from `prob` | `T × m` twice | `16·m·T` | 1.5 MB |

Peak for one E step at `m=6` is therefore about **590 MB**, and at `m=10` about **983 MB**, from
`decodelog:86-87`:

```matlab
aux=ALPHATOTAL+BETATOTAL;
gamma=squeeze(sum(exp(aux),2))';
```

Three observations, in order of how much they cost:

1. **The two `*TOTAL` arrays exist only to compute `gamma`, and `gamma` is `T × m`, which is
   0.74 MB.** The code holds 590 MB to produce 0.74 MB. Everything else in this document is
   secondary to that.
2. **The copies are avoidable with no change to the mathematics.** `aux` and `exp(aux)` are pure
   temporaries. MATLAB may elide one of them, so call it three to four copies until measured.
3. **`prob` returns the linear probabilities as well as the logs**, `normal_normal_wishart.m:88`,
   and the log decoder uses only the logs, `decodelog:18`. Small here, but it is the same habit.

The `dmax` axis is the multiplier in every one of these. The duration distribution is evaluated on
all 200 integers whether or not it has any mass there: a lognormal with a posterior mean of 30
samples and a modest variance puts essentially zero probability above, say, 90, yet all 200 slots
are carried. That is the cheapest structural saving available, section 5 item T1.2.

### 2.1 The Viterbi path is worse

`+inference/hsmmresidual_logviterbi.m`, same block:

| Array | Shape | At `m=6`, `T=15360`, `dmax=200` |
|---|---|---|
| `delta` | `m × T × dmax` | 147.5 MB |
| `deltaaux`, allocated then immediately overwritten by a scalar | `m × T × dmax` | 147.5 MB, pure waste |
| `B`, from `util.pdfdur` per state | `m × T × dmax` | 147.5 MB |
| `sa`, **grown one element at a time** | `(T+dmax−1) × m × dmax × 3` | 448 MB |

About **890 MB per block**, and the `sa` growth means repeated reallocation of an array that ends
up at 448 MB. → `ISSUES.md H3`

### 2.2 The initialisation has its own version of the problem

`+util/decodevarinit.m:17`: `xi = zeros(nstates,nstates,ndataefec)`, a dense `m × m × T` array to
hold at most `T` ones, since a hard sequence has at most one transition per sample. At `m=6` and
`T=15360` that is 4.4 MB for about 15 KB of information. In pattern B, at `Tall`, it is 124 MB.
With `opt.deceta=1` the same function allocates `dmax × m × T`, which is 147 MB per block and
**4.1 GB** on the concatenated series.

---

## 3. The dominant term: nothing is ever released

`+inference/train.m` keeps one `out` struct **per restart per candidate state count**, and each
`out` carries a full `decodevar`:

```
model(j,kstate)              = self.copytrain(1,kstate)        train.m:74, 85
[sim_array(j,kstate) modelstructarray(j,kstate)] = ... trainn(...)   train.m:81, 92
```

All of them stay alive until `train` returns, and exactly one is used, `train.m:101-109`.

Retained bytes per `out`:

| Field | Shape | Pattern A, 28 blocks | Pattern B, concatenated |
|---|---|---|---|
| `decodevar.gamma` | `T × m` per block | 0.74 MB × 28 = 20.6 MB | 20.6 MB |
| `decodevar.Emi` | `m × T` per block | 20.6 MB | 20.6 MB |
| `decodevar.xi` | `m² × T` per block | **124 MB**, not cleared in the multi block branch | cleared, `trainn.m:119` |
| `decodevar.durcount`, `scale`, `fedetail` | small | about 1 MB | about 1 MB |
| **per `out`** | | **about 166 MB** | **about 42 MB** |

Multiply by the number of `out` structs. States 2 to 10 is nine candidates; with five restarts that
is 45:

| Scenario | Per `out` | Count | **Accumulated** |
|---|---|---|---|
| Pattern A, 1 restart, states 2 to 10 | 166 MB | 9 | **1.5 GB** |
| Pattern A, 5 restarts, states 2 to 10 | 166 MB | 45 | **7.4 GB** |
| Pattern B, 5 restarts, states 2 to 10 | 42 MB | 45 | **1.9 GB** |

Add the per block E step transient of 0.6 to 1.0 GB, the data itself, 34 MB in PCA space or 220 MB
at 64 channels, and 1,260 model objects in pattern A, since `+util/addmodel.m:13` copies the model
once per block for each of the 45 candidates.

**So the honest answer to "why does it use a lot of memory" is: about 90 percent of it is retention
of results nobody reads, and almost all of the rest is four copies of an array that exists to
produce something 200 times smaller.** Neither is an algorithmic limitation. Both are engineering.

### 3.1 The parallel switch multiplies the transient

`+inference/train.m:21-25`:

```matlab
if opt.parallel
    numcores=feature('numcores');
    c=parcluster;
    c.NumWorkers=numcores*3;
end
```

`parfor` then runs over restarts, `train.m:53` and `:73`, and each worker holds its own copy of the
working set. Three workers per core is a deliberate oversubscription, presumably to hide I/O, but
for a memory bound job it multiplies the peak by three times the core count. On an eight core
machine: 24 × 0.6 GB, about **14 GB of transient alone**, before the retained results. Oversubscribing
a memory bound `parfor` is the one line in this file I would change first.

---

## 4. What the algorithm actually needs to keep

This is the key to the whole optimisation, and it is worth stating separately because it is a fact
about the mathematics, not a preference.

Each M step consumes **aggregates**, not per sample arrays, `trainn.m:57-68`:

| Sub-model | What it needs | Size, independent of `T` |
|---|---|---|
| Emission, Gaussian | `Nk = Σ_t γ(t,k)`, `Σ_t γ(t,k)·y_t`, `Σ_t γ(t,k)·y_t y_tᵀ` | `m·(1 + n + n²)` |
| Transition | `Σ_t ξ(i,j,t)` | `m²` |
| Duration | `durcount(d,k)` | `dmax·m` |
| Initial state | `γ(1,:)` | `m` |

At `m=6`, `n=10`, `dmax=200`, all four sets of statistics together are **under 100 KB**. The
toolbox currently materialises and retains hundreds of megabytes to compute them, because
`update` is written to take the full `gamma` matrix and the full `xi` array as arguments, and
because `util.outmatr` concatenates every block's `gamma` into one matrix before calling it,
`trainn.m:170-171`.

Changing the `update` contract from "here is every responsibility" to "here are the accumulated
statistics" removes the retention problem at the root, makes the multi block path streaming, and
makes the memory cost independent of the number of participants. That is the single highest value
change in this document.

---

## 5. A costed plan

Estimates are engineer days for one competent MATLAB engineer who did not write this code, and they
include writing the test that proves the change. They assume the validation harness in
`VALIDATION.md` tiers 1 and 2 exists first, which is costed separately there, because without it no
numerical change to this code can be shown to be safe.

### Tier 0, quick wins, no change to the mathematics

| # | Change | Where | Effort | Expected effect |
|---|---|---|---|---|
| T0.1 | Keep only the running best `out`, `model` and `modelstruct` instead of arrays indexed by restart and state count | `train.m:53-109` | 1.5 d | removes the ×9 to ×45 factor: **−1.4 to −7.3 GB** |
| T0.2 | Clear `decodevar.xi` in the multi block branch as the single block branch already does | `trainn.m:226` versus `:119` | 0.25 d | **−124 MB** per retained `out` |
| T0.3 | Accumulate `gamma` inside the backward loop instead of building `aux` and `exp(aux)` | `decodelog:86-91` | 1 d | removes two of four copies: **−295 MB** per E step |
| T0.4 | Stop oversubscribing workers; make it an option, default one per core | `train.m:24` | 0.25 d | peak ÷3 when `parallel=1` |
| T0.5 | Return the linear probabilities from `prob` only when asked | all 4 Gaussian emission classes | 0.5 d | small, and it is the same habit everywhere |
| T0.6 | Viterbi: delete the `deltaaux` allocation, preallocate `sa`, end the loop at `ndata` | `logviterbi:35-36, 42, 84` | 0.5 d | **−150 MB** and a large constant factor in time |
| T0.7 | Build the initialisation `xi` sparse, or skip it and pass the hard counts directly | `decodevarinit.m:17` | 1 d | −4.4 MB per block, −124 MB in pattern B |
| | **Tier 0 total** | | **5 d** | pattern A, five restarts: about **7.4 GB → 0.9 GB** |

Tier 0 changes no equation. Each one is independently testable by asserting that `gamma`, `xi`,
`durcount` and `fe` are bit comparable, or equal to within `1e-12`, before and after.

### Tier 1, structural, still the same model

| # | Change | Effort | Expected effect |
|---|---|---|---|
| T1.1 | **Sufficient statistics contract.** Change `update` on the live sub-models to accept accumulated statistics, and have the decoders emit them. The multi block loop then streams: decode a block, accumulate, discard. | 8 to 12 d | memory becomes **independent of the number of participants**; pattern A peak drops to one block's working set |
| T1.2 | **Duration support truncation.** Carry only the duration slots where the posterior duration probability exceeds a tolerance, with `dmax` as the ceiling rather than the working size. Typically a 3 to 10 times saving on every `dmax` scaled array, including the Viterbi. | 5 to 8 d | **−60 to −90 percent** of the E step and Viterbi working set |
| T1.3 | **Checkpointed forward backward.** Store the forward array every `K` steps only and recompute each `K` block during the backward pass. With `K = √T` the forward storage falls from `m·dmax·T` to about `m·dmax·√T`: 147 MB becomes **1.2 MB**, at the cost of roughly double the forward arithmetic. | 5 to 8 d | the single biggest reduction available without changing the model |
| T1.4 | Single precision option for the big working arrays, with the free energy kept in double | 2 to 3 d | a further ÷2 where accepted |
| T1.5 | Memory map the input data per block rather than loading the whole study | 2 to 3 d | −220 MB at 64 channels, more for longer studies |
| | **Tier 1 total** | **22 to 34 d** | pattern A peak of order **50 to 150 MB**, flat in participants |

### Tier 2, algorithmic

| # | Change | Effort | When it is the right answer |
|---|---|---|---|
| T2.1 | **Stochastic variational inference.** Treat blocks or subjects as a mini batch, take natural gradient steps on the global parameters, Hoffman and colleagues 2013. Memory becomes one mini batch, and convergence is usually faster in wall clock on large studies. | 15 to 25 d, plus validation | many blocks, which is exactly the 28 participant case |
| T2.2 | **Out of core sufficient statistics**, written and read per block via HDF5 or `matfile` | 5 to 8 d | when even one block's working set does not fit, that is very long recordings |
| T2.3 | GPU for the forward backward recursions | 10 to 15 d | only after T1, and only if time rather than memory is the binding constraint |
| | | | |

T2.1 changes the fitted result. It must be shown to agree with the batch answer within a stated
tolerance on synthetic data before it is used on real data, which is why `VALIDATION.md` tiers 3
and 4 are a prerequisite rather than a nicety.

### Recommended sequence and total

| Phase | Content | Effort | Peak memory, pattern A, five restarts |
|---|---|---|---|
| 0 | `VALIDATION.md` tiers 0 to 2, the safety net | 8 to 12 d | — |
| 1 | Tier 0, all seven items | 5 d | 7.4 GB → **0.9 GB** |
| 2 | T1.1 and T1.3, the two structural wins | 13 to 20 d | → **0.1 GB**, flat in participants |
| 3 | T1.2, T1.4, T1.5 if still needed | 9 to 14 d | → **0.05 GB** |
| 4 | T2.1 only if the study grows past a few hundred blocks | 15 to 25 d | one mini batch |
| | **To get a 28 participant study comfortable on a laptop** | **26 to 37 d** | about **0.1 GB** |

In calendar terms, with one engineer at a realistic 60 percent on this, phases 0 to 2 are about six
to eight weeks. The first week of it, tier 0, removes roughly nine tenths of the problem, and that
asymmetry is the main thing to take from this document.

### 5.1 What I would not do

- **Do not start with the stochastic approach.** It is the most interesting option and the worst
  first move: it changes the answer, so it needs the validation harness, and it would be solving a
  problem that tier 0 and T1.1 largely remove. Reach for it when the number of blocks is the
  binding constraint, not when retention is.
- **Do not reduce `dmax` to save memory.** It changes the model, the duration prior and the
  initialisation all at once, `ISSUES.md M13`. T1.2 gets the same saving without touching the
  science.
- **Do not rewrite the GUI to fix memory.** None of the memory is in the GUI.

---

## 6. Honest uncertainty

- Every figure is arithmetic on array shapes, not a measurement. MATLAB's copy on write and its
  in place optimisations could remove one of the four copies in section 2, which would change 590
  MB to about 440 MB per E step. It does not change any conclusion.
- The `×45` factor in section 3 assumes five restarts across nine candidate state counts. With the
  shipped default of one restart it is `×9`, which is still 1.5 GB in pattern A.
- `ISSUES.md H1` means the HSMM E step currently stops before any of this runs. The memory figures
  describe the code as written and as it behaved in the previous upload, where the two `dec*` flags
  were on. Fixing `H1` by turning the flags back on restores exactly the behaviour costed here.
- Effort figures assume no requirement to preserve bit level compatibility with published results.
  If that is required, add the golden master work from `VALIDATION.md` tier 7 and roughly 30
  percent to every tier 1 item.

## 7. How to measure each claim

Run these before quoting any number above.

1. **Per E step peak.** Wrap one `decode` call: `p0=whos; ...; profile('-memory','on')`, or simply
   `feature('memstats')` on Windows before and after, and read `whos` inside the decoder for the
   four big arrays. Confirms section 2.
2. **Retention.** After `train` returns, `s=whos('sim_array'); s.bytes`, and
   `whos('modelstructarray')`. Confirms section 3, and will be the largest single number.
3. **The ×3 worker multiplier.** `parpool` with the default versus `numcores` workers, watching
   the Windows working set. Confirms 3.1.
4. **Duration support.** After a fit, `P=m.dur_model.prob('VB',(1:dmax)'); nnz(P>1e-6)/numel(P)`.
   That ratio is the saving available from T1.2, measured rather than assumed.
5. **Checkpointing payoff.** Instrument the forward pass to count flops, then compare against a
   `K = √T` prototype on one block. Confirms the two times arithmetic cost in T1.3.
6. **A memory budget test.** Once tier 0 lands, add the assertion from `VALIDATION.md` tier 9 so a
   regression shows up as a failing test rather than as a crash six months later.
