# ADR-0005: Compose the model from four pluggable sub-models behind an informal interface

- **Status** Accepted, reconstructed from the code
- **Date reconstructed** 15 October 2026
- **Confidence** read, and the four component names are documented in the manual section 1.3

## Context

The toolbox needs to support amplitude states and connectivity states, Gaussian and lognormal
durations, and both HMM and HSMM, without four copies of the inference.

## Decision

Four handle classes per model, one per component, each implementing the same informal interface,
specified in `../SPECIFICATION.md` section 3. The inference code never names a distribution family.
No abstract base class declares the interface.

## Consequences

- The best decision in the codebase. A new observation model needs no change to `+inference`, and
  the GUI picks it up by listing the package folder.
- Nothing enforces conformance, so four shipped classes are missing methods the inference calls and
  fail at run time, `../../review/ISSUES.md H2`.
- The GUI folder listing offers every file in the package, including two test scripts.
- Being handle classes, the objects alias, so three separate copy related concepts are needed,
  `copy`, `copytrain` and `cleanpos`, and one of them clears the wrong fields,
  `../../review/ISSUES.md M10`.
- The `update` signature passes per sample statistics, which is the root of the memory problem. See
  `ADR-0015`.

## Alternatives not taken

- **An abstract base class per family**, which MATLAB supports and which would turn `H2` into a load
  time error. Proposed as `ADR-0018`.
- **Function handles in a struct**, lighter, with no interface guarantee at all.
