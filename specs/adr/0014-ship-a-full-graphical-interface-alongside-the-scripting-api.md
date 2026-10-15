# ADR-0014: Ship a full graphical interface alongside the scripting API, in a single class

- **Status** Accepted, reconstructed from the code
- **Date reconstructed** 15 October 2026
- **Confidence** read, and the manual sections 3 to 7 are entirely about the GUI

## Context

The audience is neuroscientists, not programmers. The manual is written as a GUI walkthrough, so the
interface is a primary deliverable rather than a convenience.

## Decision

`uiBSD.m`, one `classdef` of 3,548 lines with 122 nested functions, building every dialog
programmatically with `uicontrol`. `BSD.m` and `initBSD.m` are one line launchers.

## Consequences

- Non programmers can use the toolbox, which is the point, and the manual walkthrough matches it.
- The GUI holds no science, so it can be excluded from a port at a tenth of the cost,
  `../../review/PORT_PYTHON.md` section 1.
- 122 nested callbacks in one scope make variable sharing subtle, which is the mechanism behind
  `../../review/ISSUES.md M12`.
- The model menus are built by listing package folders with `ls`, which couples the interface to
  Windows, `../../review/ISSUES.md M11`, and surfaces unusable classes, `H2`.
- Four `my_warning('TODO help')` stubs remain, `uiBSD.m:1782`, `2251`, `2510` and `3019`.

## Alternatives not taken

- **MATLAB App Designer**, which did not exist in a usable form when this was begun.
- **Scripting plus notebooks**, cheaper, and a poor fit for the stated audience.
