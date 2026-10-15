# WORKING.md

Durable state of the BSD toolbox review. Read this before doing anything else in this repo.

- **Branch** `rev/review`, created 15 October 2026 off `master`.
- **Baseline commit** `4a89dcc` "Update EEG_data". No toolbox file has been modified.
- **Working tree** review and specification documents only, see Session context.
- **Last updated** 15 October 2026.

## ▶ Next session starts here

1. Read `review/README.md`. It maps the six questions to the six documents and gives a reading
   order.
2. If MATLAB is now available, run `review/ASSESSMENT.md` section 8 item 1:
   `load Examples/Example1_data2dim.mat; m=hsmm(2,4); m.train(data)`. Expect the undefined
   `ALPHATOTAL` error of issue `H1`. Then rerun with `m.train(data,'decalpha',1,'decbeta',1)`.
3. Record the outcome in `review/ISSUES.md` by upgrading `H1` from `read` to `reproduced`, or by
   striking it, and paste the exact error text. Then work down the other four runs in section 8.
4. Do not edit any `.m` file until Aland asks. If a fix is agreed, the first one is one line in
   `program_option.m`, and the rule is one fix per commit with the failure shown first.
5. Nothing has been pushed. Ask before `git push -u origin rev/review`.

## Status key

| Mark | Meaning |
|---|---|
| `[ ]` **pending** | Agreed, not started |
| `[~]` **doing** | In progress right now |
| `[x]` **done** | Complete and verified where verification was possible |
| `[>]` **future** | Deferred by decision, not in the current push |
| `[?]` **question** | Blocked on an answer from the user |

## Session context

- **Goal** Answer six questions about the BSD toolbox: what the code is, what is wrong with it and
  how to test that, what its specifications are, what memory optimisation would cost, what a Python
  port would cost, and how it should properly be validated.
- **Constraint** MATLAB is unavailable, so the whole review is static. Nothing is `reproduced`.
- **Upstream** Somebody else's product code, from the Brain Dynamics Laboratory. Missing context is
  a question for the authors, not a finding.

| Document | Role |
|---|---|
| `review/README.md` | entry point: the six questions mapped to documents, reading order, what to do first |
| `review/SYSTEM.md` | what the code is: the generative model, inference, architecture, the EEG pipeline, six diagrams, the measured inventory |
| `review/ASSESSMENT.md` | judgement: purpose with evidence, one path traced end to end, what is well designed, the refutation log, eight questions for the authors, the ordered run list |
| `review/ISSUES.md` | 46 findings. High 6, Medium 15, Low 14, Decoration 7, Wish 4. Evidence, condition, reproduction and smallest fix each |
| `review/MEMORY.md` | why a 28 participant study exhausts memory, with arithmetic, and a costed three tier plan |
| `review/PORT_PYTHON.md` | Python port effort by component, library mapping, the traps specific to this codebase |
| `review/VALIDATION.md` | eleven tiers of validation, effort, acceptance checklist, and the issue to test matrix |
| `specs/README.md` | standing and conventions for the reconstructed specifications |
| `specs/BUSINESS_RULES.md` | 27 domain rules, BR-001 to BR-027, each with where it is enforced or not |
| `specs/REQUIREMENTS.md` | 87 requirements in EARS form with evidence and status. 46 implemented, 7 partial, 25 defective, 9 absent |
| `specs/SPECIFICATION.md` | data contracts, the option set, the sub-model interface, algorithm pseudocode, the extension contract |
| `specs/DESIGN.md` | layering, component responsibilities, design choices and consequences, design debt, where the design would go next |
| `specs/adr/` | 18 decision records, 14 reconstructed and 4 proposed, plus an index |
| `CLAUDE.md` | repo specific working rules |
| `Manual BSD.pdf` | the only real documentation. Version 1.0, November 2018 |

## Workstreams

| # | Name | Status | Detail |
|---|---|---|---|
| 1 | Orient and map | `[x]` | Purpose stated with evidence, package map, measured inventory |
| 2 | Trace one path end to end | `[x]` | Single subject HSMM path, data to state sequence, by reading |
| 3 | Find and refute | `[x]` | 46 findings, 7 candidates attacked, 2 dropped, 4 downgraded |
| 4 | What is the code | `[x]` | `review/SYSTEM.md`, six mermaid diagrams |
| 5 | What is wrong and how to test it | `[x]` | `review/ISSUES.md` plus the issue to test matrix |
| 6 | Reconstruct the specifications | `[x]` | `specs/`, five documents and 18 ADRs |
| 7 | Memory optimisation effort | `[x]` | `review/MEMORY.md`, costed, estimates not measurements |
| 8 | Python port effort | `[x]` | `review/PORT_PYTHON.md`, estimate only as asked |
| 9 | Validation approach | `[x]` | `review/VALIDATION.md`, eleven tiers with effort |
| 10 | Verify by running | `[ ]` | **Blocked on MATLAB.** Nothing in the review is `reproduced` |
| 11 | Fix anything | `[>]` | Deferred. No edits to the toolbox without Aland asking |

### 1 to 3, the review proper `[x]`

- [x] Purpose stated in one sentence with its evidence named
- [x] Example data dimensions measured by parsing the MAT headers locally, so no size is a guess
- [x] `Manual BSD.pdf` extracted to text locally and checked against the code
- [x] Full path traced: `Script_Tutorial_basic.m` through `train`, `trainn`,
      `hsmmresidual_decodelog`, the four M steps, free energy, state sequence
- [x] Verified by hand the one piece of non obvious tensor algebra, the duration statistic at
      `hsmmresidual_decodelog.m:61-72`. It is correct
- [x] Every candidate finding attacked at its claimed severity; refutation log written
- [x] Confidence labels on the 35 severity ranked issues: 33 `read`, 2 `inferred`, 0 `reproduced`.
      The 7 Decoration and 4 Wish items are not claims and carry no label

### 4 to 9, the six questions `[x]`

- [x] `review/SYSTEM.md`, `review/README.md`
- [x] `review/ISSUES.md` extended with `M15`, no seed control, found while writing the validation
      plan
- [x] `specs/` with business rules, EARS requirements, specification, design, 18 ADRs
- [x] `review/MEMORY.md`, with the 28 participant workload costed from measured array shapes
- [x] `review/PORT_PYTHON.md`, by component, from measured line counts
- [x] `review/VALIDATION.md`, eleven tiers, with every issue mapped to the test that catches it

### 10, verify by running `[ ]`

In the order of `review/ASSESSMENT.md` section 8.

- [ ] `H1`, the undefined `ALPHATOTAL`. One line, highest value
- [ ] `M3` and `M9`, print `[out.fedetail.fe]` and check monotonicity
- [ ] `M5`, check whether the rows of `decodevar.gamma` sum to 1
- [ ] `H6` and `H4`, count states with `Nk<15`, compare `unique(stateseq)` with `1:nstates`
- [ ] `H3`, time and measure the Viterbi memory. Run this last
- [ ] `MEMORY.md` section 7, the six memory measurements, before quoting any figure from it
- [ ] `M12`, one `disp` in the GUI settles whether `select_eeg_coords` returns the search path
- [ ] `M11`, `numel(obj.classes_ls('emis_model'))` on a non Windows MATLAB

## Open questions

Blocked on Aland. None asked more than once.

- [?] Push `rev/review` to `origin`, or keep it local? Asked 1 time.
- [?] Do the eight questions in `review/ASSESSMENT.md` section 7 go to the authors as written? They
      are deliberately not findings, because the answers live in the authors' heads. Asked 1 time.
- [?] Which of memory, port or validation is the actual next piece of work? They share a
      prerequisite, `VALIDATION.md` tiers 0 to 3, so that is the answer regardless, but the order
      after it depends on intent. Asked 1 time.

**Resolved since the last update:** whether `review/` should be committed. Yes, on `rev/review`,
since the user asked for a branch for exactly this work.

## Decisions taken

- **15 October 2026** No Critical severity assigned, even though `H1` stops the headline feature.
  Why: in Aland's rubric Critical means a wrong or unsafe result, and High covers "a crash on
  normal input". `H1` is a crash. Calling it Critical would be easy to disprove.
- **15 October 2026** `H4` and `H6` kept at High rather than Critical. Why: both can produce a
  silently wrong published figure, which is Critical shaped, but both need a stated condition and
  the condition cannot be demonstrated without running.
- **15 October 2026** The GUI "Iterations and Repetitions are swapped" claim was dropped, not
  downgraded. Why: `uiBSD.m:2567-2568` swap the names and `uiBSD.m:985-986` swap them back.
  Recorded in the refutation log so it is not rediscovered.
- **15 October 2026** Style findings are kept out of Low, in a Decoration section. Why: Aland's
  rubric separates them so Low never carries both a real defect and a naming preference.
- **15 October 2026** Every line number quoted comes from numbered output, `cat -n` or `grep -n`,
  never from counting an unnumbered listing. Why: an off by two citation read aloud costs more than
  no citation.
- **15 October 2026** The specifications are written as a reconstruction with an explicit standing
  note, and every requirement carries a status. Why: a specification that claims authority it does
  not have would be worse than none, and the status column turns the document into a gap analysis
  rather than a wish list.
- **15 October 2026** Memory and port figures are given as ranges with an explicit statement that
  they are arithmetic and estimates, each with a "how to measure" or "what would move this"
  section. Why: the most likely way these documents fail is someone quoting an estimate as a
  measurement.
- **15 October 2026** The recommended first move for all three forward looking documents is the
  same: `VALIDATION.md` tiers 0 to 3 and 7. Why: the memory work and the port both change numerics,
  and neither can be shown safe without a golden master and a brute force equivalence test.
- **15 October 2026** Recommended against porting the GUI to Python. Why: 3,548 of the 10,047
  lines, no science in it, and two of its defects are already known. A CLI plus notebooks is a
  tenth of the cost.

## Future, deferred by decision

- `[>]` Fixing anything in the toolbox. The brief was a review; the house rule is read only until
  asked.
- `[>]` Reading the four `mar*` emission models line by line. None is the default, and the
  duplication question goes to the authors first, `ASSESSMENT.md` question 6.
- `[>]` Reading `uiBSD.m` in full, 3,548 lines. The model implementation was the question. The GUI
  was searched for specific behaviour only, and its findings are labelled accordingly.
- `[>]` Auditing `+util/kmeans2015.m`, 538 vendored MathWorks lines. The question there is
  licensing, `M14`, not correctness.
- `[>]` A prototype of any `MEMORY.md` or `VALIDATION.md` item. Costed, not built.

## Environment

- **Platform** Windows 11, `C:\Codes\BSD\BSD`, PowerShell and Git Bash both available.
- **MATLAB** Not installed or not on `PATH` in this session. Nothing has been executed. The manual,
  section 2.1, requires R2015 or later, and `repelem` at
  `+inference/hsmmresidual_decodelog.m:62` confirms R2015a as the true floor.
- **EEGLAB** Required for `topoplot`, `Script_Tutorial_pipeline.m:34` and `uiBSD.m:1321`. Not in
  the repository; documented only in the PDF manual.
- Tools used for the review, both local, no network:
  - `pdftotext -layout "Manual BSD.pdf" <scratchpad>/manual.txt`, from `/mingw64/bin`.
  - Python 3.14 at `/c/Python314/python`, with **no** numpy or scipy. MAT headers parsed with
    `struct` and `zlib` only. The ADR generator is at `<scratchpad>/gen_adr.py` if the records ever
    need regenerating.

## Gotchas, hard won, do not rediscover

- **The Bash tool breaks on a literal single quote inside a heredoc.** Two long heredocs failed with
  `unexpected EOF while looking for matching '`. Write long documents with the Write tool, and write
  generator scripts to a file first, then run them. This cost two attempts.
- **MAT version 5 compressed elements are not 8 byte padded.** Advancing the pointer by the padded
  length desynchronises the parser after the first variable. Advance by `8+nbytes` for element type
  15 and `8+pad(nbytes)` otherwise. That is what made `dataeeg64`, 23120 by 64, readable.
- **MAT tags use a small element format.** When the top 16 bits of the tag word are non zero, the
  byte count is in those bits and the payload is in the remaining 4 bytes of the tag.
- **`program_option.m` is a script, not a function.** Six files run it in their own scope to obtain
  defaults. Grepping for `opt.` assignments is the only way to know what a run actually uses, and
  this pattern is the direct cause of `H1`.
- **Package folder paths need the `+`.** `cat +util/Dirichlet/KLDirich.m` silently produces nothing;
  the path is `+util/+Dirichlet/KLDirich.m`. I briefly thought those files were empty.
- **`git log` alone understates the history.** Every commit is "Add files via upload", so diffs
  between uploads are the only way to see intent.
  `git diff cafb2d5 3f1fe18 -- program_option.m` is what proved `H1` is a regression.
- **`+util/topograph.asv` is an editor autosave**, not source. Do not read it as current.

## Key file map

| Location | Why it matters |
|---|---|
| `+inference/hsmmresidual_decodelog.m:86` | the line that stops the HSMM with shipped defaults, `H1` |
| `program_option.m:16-17` | the two flags that cause it; the previous upload had them at 1 |
| `+inference/hsmmresidual_decodelog.m:61-72` | the duration statistic. Correct, non obvious, do not touch |
| `+inference/train.m:53-109` | restarts and the state search, and the dominant memory term |
| `+inference/train.m:118-122` | where `stateseq` and `nstates` are produced, the root of `H4` |
| `+inference/trainn.m:50-112` | the VB loop. `:95-103` masks a decreasing free energy |
| `+inference/trainn.m:57-68` | the four M steps, and the only place the sub-model interface is exercised |
| `+emis_model/normal_normal_wishart.m:87` | the `2*pi` constant, `M1` |
| `+emis_model/normal_normal_wishart.m:143` | `Nk < 15`, the extinction threshold, `H6` |
| `+dur_model/normal_normal_gamma.m:90-92` | duration renormalisation the M step does not match, `M9` |
| `+dur_model/normal_normal_gamma.m:203-207` | `cleanpos` clearing fields that do not exist, `M10` |
| `+inference/hsmmresidual_logviterbi.m:35-36` | the gigabyte allocations, `H3` |
| `+util/decodevarinit.m:35-39` | the `dmax+1` index, `H5` |
| `+util/outmatr.m:5-7` | `eval` built field access, and why study memory scales with participants |
| `uiBSD.m:3941-3952` | `classes_ls`, which puts unusable classes in the model menus, `H2` and `M11` |
| `uiBSD.m:982` | `matixer`, the misspelled option field, `M6` |
