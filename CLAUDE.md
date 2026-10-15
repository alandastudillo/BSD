# CLAUDE.md, BSD toolbox

Working rules for this repository. Aland's global rules in `~/.claude/CLAUDE.md` still apply and
win where they overlap. This file adds only what is specific to this codebase.

Live state of the review is in [WORKING.md](WORKING.md). Read it first.

| Want | Read |
|---|---|
| where everything is | [review/README.md](review/README.md) |
| what the code is, with diagrams | [review/SYSTEM.md](review/SYSTEM.md) |
| judgement, strengths, refutation log, open questions for the authors | [review/ASSESSMENT.md](review/ASSESSMENT.md) |
| the 46 findings, with evidence and reproduction steps | [review/ISSUES.md](review/ISSUES.md) |
| why it uses so much memory, and what fixing it costs | [review/MEMORY.md](review/MEMORY.md) |
| what a Python port would cost | [review/PORT_PYTHON.md](review/PORT_PYTHON.md) |
| how to validate it properly, and the issue to test map | [review/VALIDATION.md](review/VALIDATION.md) |
| reconstructed requirements, design and decisions | [specs/](specs/) |

Keep those documents in step with each other. The finding identifiers `H1` to `W4` are referenced
from `specs/`, from the ADRs and from the other review documents, so **do not renumber them**; the
same goes for `REQ-`, `BR-` and `ADR-` identifiers.

## What this repository is

A MATLAB toolbox, BSD, Brain State Dynamics, from the Brain Dynamics Laboratory. It segments
continuous multivariate brain signals, EEG, MEG, LFP or fMRI, into recurring discrete states, by
variational Bayes, using a hidden Markov model or a hidden semi Markov model with explicit state
durations. `Manual BSD.pdf` section 1.1 is the authority on intent.

**This is somebody else's product code.** It is not an exercise. It has context not in the
repository, and authors who are named in the manual. Missing context is a question for the team,
not a finding.

## Standing posture

- **Read only.** Do not edit, install or run anything unless Aland asks.
- MATLAB is not available in this environment. Say so rather than guessing at runtime behaviour.
  Reading is not running: label every claim `read`, `inferred` or `suspected`, never `reproduced`,
  until something has actually been executed.
- Do not propose renames, refactors, reformatting or new abstractions. The house style is theirs.
- Never send repository content anywhere. No network, no external service, no paste elsewhere.

## Getting line numbers right here

The citation rule matters more than usual in this repository, because the packages and the editor
leftovers both invite mistakes.

- Use `cat -n`, `grep -n` or the numbered Read tool. Never count lines in an unnumbered listing.
- **Package folders need the `+`.** The path is `+util/+Dirichlet/KLDirich.m`, not
  `+util/Dirichlet/KLDirich.m`. The second silently prints nothing, which looks like an empty file.
- `+util/topograph.asv` is an editor autosave, not source. `+util/initvb_old.m` and
  `+util/miplot3 - copia.m` are leftovers. Do not cite them as current behaviour.

## How the code is organised

- `hsmm.m` and `hmm.m` are thin containers. Each composes four sub-models, emission, duration,
  transition and initial state, and delegates everything to `+inference`.
- `+inference/train.m` handles restarts and the state number search. `+inference/trainn.m` is one
  VB run. The E step lives in `+inference/hsmmresidual_decodelog.m` for the HSMM and
  `+inference/hmm_decodelog.m` for the HMM.
- Every sub-model is expected to implement the same informal interface, `prob`, `update`,
  `expect`, `priornoinf`, `priorfull`, `posteriorfull`, `cleanpos`, `copy`, `expectfun`,
  `divklfun`, `trouble`, `ndatafun`, `sample`, `parsamplfun`, `graf`. **No base class enforces
  it**, and several shipped classes are missing methods, so check the inventory before assuming a
  model is usable:
  `grep -o "function [^(]*(" +emis_model/<file>.m`.
- There are four `mar*` emission variants and four Gaussian ones, plus two decoders per model
  family. Do not assume the highest number is the live one. Which is current is an open question
  for the team.

## Traps that will cost you time

- **`program_option.m` is a script, not a function.** `train`, `trainn` and all four decoders run
  it inside their own scope to obtain `opt`, for example `+inference/train.m:19`. So the default
  option set is injected by text. To find what a run actually uses, grep for every `opt.` write
  along the path, including in `uiBSD.m`.
- **The shipped defaults break the HSMM E step.** `program_option.m:16-17` set `decalpha` and
  `decbeta` to 0, while `+inference/hsmmresidual_decodelog.m:86` reads the arrays those flags
  guard. Expect an undefined variable error from any default HSMM run. See issue `H1`.
- **Every commit message is "Add files via upload".** Git history is a series of drops, so diffs
  between uploads are the only way to see intent. `git diff cafb2d5 3f1fe18 -- program_option.m`
  is what showed `H1` is a regression and not a design choice.
- **Spanish and English are mixed** in comments, diagnostics and UI strings. Author notes such as
  `%Se debe revisar`, must be checked, and `Temporalmente se modifica`, temporarily modified, mark
  places the authors were unsure. They are useful signposts, not defects in themselves.
- The GUI, `uiBSD.m`, is about 4000 lines in one class with deeply nested callbacks. Search it,
  do not read it linearly, and be careful with nested function scoping when reasoning about which
  variables are shared.

## If a change is ever asked for

Follow the global rules, and these specifics.

- State the plan in two or three lines and wait.
- One fix per commit. Show the failure first where that is possible.
- The smallest change that proves the biggest diagnosis is one line in `program_option.m`,
  restoring `decalpha` and `decbeta` to 1.
- Do not touch the index algebra at `+inference/hsmmresidual_decodelog.m:61-72`. It is correct and
  it is not obvious.
- Never widen scope. The duplicated model families, the `eval` based struct access and the GUI
  size are all known and deliberately out of scope.
