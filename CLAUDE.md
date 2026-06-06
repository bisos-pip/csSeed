# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this is

`bisos.csSeed` is a Python package, part of BISOS (ByStar Internet Services
Operating System), built on the [PyCS-Framework](https://github.com/bisos-pip/pycs)
(`bisos.b.cs`). It provides reusable **seeds** for building Target-Oriented
Command-Services (TOCS) modules — command-line tools that operate on collections
of Managed Objects across one or more targets/destinations.

The Python project lives under `py3/`; the package source is `py3/bisos/csSeed/`.

## Commands

All commands assume you are in `py3/`.

- **Build / publish to PyPI:** `./pypiProc.sh` — a thin wrapper that re-execs
  `/bisos/core/bsip/bin/seedPypiProc.sh`. Run `./pypiProc.sh` with no args to
  see its menu of subcommands (build, upload, version bump, etc.). The package
  version is set in `setup.py` (`pkgVersion()`), and the version string in
  `setup.py` is auto-managed by this script — don't hand-edit it expecting it to stick.
- **Install editable for development:** `pip install -e .`
- **Lint / type-check** (paths from the in-file "Workbench" headers): the venv
  interpreter is `/bisos/venv/py3/bisos3/bin/python`; linters live in
  `/bisos/pipx/bin/` (`pyflakes`, `flake8`, `pylint`, `pycodestyle`). `mypy` is
  used (a `.mypy_cache/` for py3.11 is present).
- **Smoke test** (no test suite exists): run a seed's example menu, e.g.
  `bin/addCmnds-helloWorld.pcs` (prints the example menu) or
  `bin/addCmnds-helloWorld.pcs -i helloWorld` (runs the `helloWorld` command).

There is no unit-test framework in this repo. Verification is done by running
the `.cs`/`.pcs` executables and inspecting their output — the expected output
is captured inline in each command via `self.captureRunStr(...)`.

## The seed / plant model — the core concept

This is the single most important thing to understand before editing.

A **seed** is a generic, packaged CS executable (`bin/*-seed.cs`) that supplies
the framework `main`, the common parameters, and the player/examples machinery —
but no domain commands. A **plant** is a `.pcs` script that imports a seed module
and supplies the actual domain commands. At runtime they combine into one tool.

The wiring (see `bisos/csSeed/seedsLib.py`):

1. You run a plant, e.g. `bin/addCmnds-helloWorld.pcs`. It does
   `from bisos.csSeed import addCmnds_seed`.
2. `addCmnds_seed.py` registers an `atexit` hook (`atexit_plantWithWhich`) that
   calls `seedsLib.plantWithWhich('addCmnds-seed.cs')`.
3. On interpreter exit, `plantWithWhich` locates the seed `.cs` on `$PATH`
   (`shutil.which`), records the plant↔seed pair in the `seededCsxuInfo`
   singleton, and re-execs the seed via `b.importFile.execWithWhich`.
4. The seed `.cs` runs `g_csMain`. Because `seededCsxuInfo.plantOfThisSeed` is
   now set, the seed imports the original plant back as a module named
   `plantedCsu` (`b.importFileAs('plantedCsu', ...)`), merges its commands into
   the CSU list, and runs the plant's `examples_csu` / `examples_pcs`.

So: **the plant file is the entry point you invoke, but the seed's `g_csMain` is
what actually drives execution, with the plant injected as `plantedCsu`.**

Two seed families exist, each with a `_seed` (atexit/wiring), `_seedInfo`
(singleton + `setup()` for plants to register data), and a paired `bin/*-seed.cs`:

- **addCmnds** (`addCmnds_seed.py`, `addCmnds_seedInfo.py`, `bin/addCmnds-seed.cs`):
  the general-purpose seed. Plants register `seedType`, `kwSeedInfo`,
  `commonParamsFuncs`, and `examplesFuncsList` via `addCmnds_seedInfo.setup()`.
  Example plants: `bin/addCmnds-helloWorld.pcs`, `bin/exmpl-addCmnds.pcs`.
- **csCmndsList** (`csCmndsList_seed.py`, `csCmndsList_seedInfo.py`,
  `csCmndsList_csu.py`, `bin/csCmndsList-seed.cs`): for declaring a list of CS
  commands (`CsCmnd` dataclass: `verb`, `pars`, `args`, `doContinue`) to be run,
  potentially against multiple targets/clusters. This seed is also classified
  `cs-mu` (uploadable — see `--upload`). Example plant: `bin/exmpl-csCmndsList.pcs`.

When adding a new seed, follow this same triple (`_seed` + `_seedInfo` + `bin/*-seed.cs`)
and the singleton-`setup()` registration pattern.

## Writing CS commands

Commands are classes subclassing `cs.Cmnd` (from `bisos.b import cs`). Each
declares `cmndParamsMandatory`, `cmndParamsOptional`, `cmndArgsLen`, and a
`cmnd(self, rtInv, cmndOutcome, ...)` method that returns a `b.op.Outcome`
(set via `cmndOutcome.set(opError=..., opResults=...)`). A module-level
`examples_csu()` / `examples_pcs()` function builds the interactive example menu
using `cs.examples.*`. `bin/exmpl-addCmnds.pcs` is the most complete worked
example (params + args + stdin + py-invocation).

## COMEEGA / org-babel dynamic blocks — do not hand-edit

Every source file is dense with `####+BEGIN: ... ####+END:` blocks and
`#+begin_org ... #+end_org` docstrings. These are **machine-generated** by
Emacs/Blee org-mode dynamic blocks (the BISOS "COMEEGA" authoring system). The
content between a `####+BEGIN:` and its matching `####+END:` is regenerated from
the parameters on the `BEGIN` line — class headers, function signatures,
import blocks, the `main` block, and file boilerplate are all produced this way.

When editing:
- Edit only the body **after** a `####+END:` marker (the actual implementation),
  not the generated header/signature between the markers — hand edits inside a
  dblock are liable to be overwritten on regeneration.
- The `csInfo` dict near the top of each file (version, status, panel) is also
  managed; treat its `version` as generated.
- Files end with `### no-byte-compile: t` and `#+STARTUP: showall` — leave intact.

## Documentation

Primary docs are Blee-Panels under `py3/panels/` (org-mode), plus the project
overview in `README.org` / `README.rst` / `_description.org`. The `.rst` and the
title in `_description.org` are consumed by `setup.py` for the PyPI long
description, so keep them in sync with `README.org`.
