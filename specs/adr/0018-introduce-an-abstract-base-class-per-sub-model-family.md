# ADR-0018: Introduce an abstract base class per sub-model family

- **Status** **Proposed by this review**, not implemented
- **Date reconstructed** 15 October 2026
- **Confidence** proposal, grounded in `../../review/ISSUES.md H2`

## Context

`ADR-0005` established an interface that no artefact declares. Four shipped classes are missing
methods the inference calls, and the GUI offers all of them, including two test scripts, so the
failure appears as an Unrecognized method error after the user has configured a whole run.

## Decision

Declare `emis_model.base`, `dur_model.base`, `trans_model.base` and `in_model.base` as abstract
classes carrying the interface in `../SPECIFICATION.md` section 3, and have every concrete class
inherit. MATLAB then refuses to instantiate an incomplete class, so the failure moves from run time
to construction time with a message naming the missing method.

## Consequences

- `H2` becomes impossible rather than merely detected.
- The GUI can filter its menus on `isa(obj,'emis_model.base')`, which removes the test scripts and
  the stub duration class from the dropdowns at the same time.
- Shared behaviour, the repeated `priorfull`, `copy` and per state prior indexing idiom, can move to
  the base class, removing perhaps 200 duplicated lines.
- It is a breaking change for any user who has written their own sub-model, and there is no way to
  know whether anyone has.
- Doing it properly means deciding which of the four MAR variants is canonical, which is open
  question 6 in `../../review/ASSESSMENT.md` and not the reviewer decision.

## Alternatives not taken

- **A conformance test instead of a base class**, `../../review/VALIDATION.md` tier 0, which catches
  the same defect without a breaking change. Cheaper, and it is the right first step.
- **Duck typing with a clear error**, wrapping each interface call, which is noisy and partial.
