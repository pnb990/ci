<!--
SPDX-FileCopyrightText: 2026 Pierre-Noel Bouteville <pnb990@gmail.com>

SPDX-License-Identifier: BSD-3-Clause
-->

# soft-lib/ci — shared Forgejo CI

Reusable workflows and composite actions shared by the projects of this forge,
next to `soft-lib/docker-images`. That repository shares the **images**, this
one shares the **jobs** that run in them.

It exists because the same workflows had been copied by hand into four
projects. Comments stripped, their bodies were nearly identical; the
differences that had accumulated were not deliberate choices but improvements
made in one project and never propagated back — including two projects whose
`test-python.yaml` ran no linter at all.

## Calling a workflow

A workflow file must physically exist in the calling project's
`.github/workflows/`: Forgejo parses it server-side from the tree of the pushed
commit, so there is no inheritance and a submodule cannot carry one. What each
project keeps is therefore a thin caller holding **its own trigger** — which is
right, since the trigger is exactly what is project-specific.

```yaml
# .github/workflows/python-checks.yaml, in the calling project
---
"on":
  push:

jobs:
  python-checks:
    uses: soft-lib/ci/.github/workflows/python-checks.yaml@<sha>
    with:
      image: pnb990/python3:ci-85fbc6f6edf84bcaa189cc923b6889e9d48987b2
      test: pytest tests
```

## One workflow, one job, one line in the run list

The workflows here are deliberately *not* merged into a single job, even where
they share an image and a checkout. A job is the unit the run list shows, so
folding a check into another one hides it: you can no longer see that it ran.
And steps stop at the first failure, so a merged job lets one tool's error mask
another tool's verdict.

`lint-reuse.yaml` is the case that settled it. REUSE checks licence headers,
which is not a python matter — a firmware or documentation project has headers
too, and cannot be asked to carry ruff, pylint and a `uv.lock` to get the
check. It therefore stays a workflow of its own, callable from anything, with a
`pinned: false` mode for a project with no python environment at all.

The duplication that mattered — the image, the provenance stamp, the recursive
checkout, the `uv sync` — is shared all the same, because it is shared *here*,
once, rather than by merging the jobs in the caller.

## Pinning

Everything is pinned by **commit sha**, never by branch or floating tag, and
this repository is pinned the same way the images are: `@<sha>`, bumped by a
reviewable commit in the project. A change here breaks no project until that
project bumps it.

The **image pins stay in the projects**, passed as an input. They are not
centralised here on purpose: the premise of `check-image-bump.yaml` is that a
pin is a commit in the repository it protects, so that bumping it is reviewable
there and rolling it back is a `git revert` there.

## What is here

| Workflow | Replaces | Inputs |
|---|---|---|
| `python-checks.yaml` | `test-python.yaml` | `image` (required), `test`, `submodules` |
| `lint-reuse.yaml` | `lint-reuse.yaml` | `image` (required), `submodules`, `pinned` |
| `fw-build.yaml` | `build-firmware.yaml`, `build-doc.yaml` | `image`, `build` (required), `matrix-value`, `require-matrix`, `submodules`, `artifact-name`, `artifact-path`, `artifact-retention-days` |

A python project calls the first two, and gets the two results it had before. A
project with no python calls only `lint-reuse.yaml`, with `pinned: false`.

`python-checks.yaml` runs four tools, one step each: `ruff check`, `ruff format
--check`, `pylint` and `pyright`. They are not optional and there is no input
to turn one off. A caller therefore carries `ruff`, `pylint` and `pyright` in
its dev dependencies and configures them in its `pyproject.toml` — for pyright
an `exclude` matters, as it follows a directory rather than a file list. The
reason for having no opt-out is the reason this repository exists: two of the
four original projects had a `test-python.yaml` that ran no linter at all, and
pyright was running in one project out of five. A per-project switch is how
that comes back.

`fw-build.yaml` shares the *scaffolding* of a firmware build — image,
provenance stamp, recursive checkout, `uv sync`, the optional artifact — while
the build command comes in as an input, and the matrix with the `prepare` job
that computes it stays in the project. That is where make and cmake genuinely
diverge, and it is the line this repository does not cross.

One consequence of calling rather than copying, measured rather than guessed: a
caller's own `name:` lands on the **0 s wrapper task**, not on the task that
does the work. A matrix caller would therefore get N identical rows in the run
list unless the shared job names itself. `fw-build.yaml` does, from
`matrix-value` — which is why that input earns its keep twice.

Artifacts are uploaded with
**`https://code.forgejo.org/forgejo/upload-artifact@v5`**, Forgejo's fork.
GitHub's own `actions/upload-artifact@v4` refuses to run outside github.com and
fails in four seconds without uploading anything. The fork is the same action
with that check removed; measured working here, including a 20 MB payload.

The build command carries its own environment: exporting
`TOOLCHAIN_STM32_DIR="$ARM_TOOLCHAIN_DIR"`, or whatever the project's build
system reads, is the caller's first line. The name on the left of that export
belongs to the build system, not to the image, so it is not this repository's
to know.

How a project becomes a caller, the pin rules, and what was learnt while
converting the first projects: [docs/design-notes.md](docs/design-notes.md).

## Requirements

Forgejo **15.0.6** and runner **v13.0.0** or later. Earlier versions do not
expand a reusable workflow called from a dynamic matrix
([forgejo#10647](https://codeberg.org/forgejo/forgejo/pulls/10647)).
