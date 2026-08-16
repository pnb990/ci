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
| `python-checks.yaml` | `lint-reuse.yaml` + `test-python.yaml` | `image` (required), `test`, `submodules` |

Planned, not written yet: `fw-build.yaml`, sharing the *environment* of a
firmware build (image, provenance, recursive checkout, `uv sync`,
`TOOLCHAIN_STM32_DIR`) while the build command and the matrix source stay in
the project — that is where make and cmake diverge.

The design note this repository implements lives in `pnb/utils`, as
`ci-shared-workflows.md`.

## Requirements

Forgejo **15.0.6** and runner **v13.0.0** or later. Earlier versions do not
expand a reusable workflow called from a dynamic matrix
([forgejo#10647](https://codeberg.org/forgejo/forgejo/pulls/10647)).
