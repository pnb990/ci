<!--
SPDX-FileCopyrightText: 2026 Pierre-Noel Bouteville <pnb990@gmail.com>

SPDX-License-Identifier: BSD-3-Clause
-->

# Design notes

What the README does not say: how a project becomes a caller, the rules that
keep the pins valid, and what was learnt (sometimes the hard way) while the
first seven projects were converted in August-September 2026. Open work lives in
forge issues, not here.

## Why sharing, in one table

Before this repository, four projects carried fifteen hand-copied workflow
files, about 850 lines of YAML. Comments stripped, the complete list of
functional differences was:

| Difference | What it was |
|---|---|
| `actions/checkout@v4` vs `@v6` | 3 projects behind |
| image pin `ci-4999611e…` vs `ci-85fbc6f6…` | 3 projects on an older image |
| "Image provenance" step | in 2 copies out of 5 |
| bare `uv sync` instead of `--dev --frozen` | a stale `uv.lock` passed silently |
| `ruff` / `pylint` steps | in 3 copies out of 5: two `test-python.yaml` linted nothing |
| `pytest tests` / `./tests/test.py` / nothing | the one legitimate per-project variation |

One real difference; every other one was an improvement made in one project and
never propagated back. Small, unintended differences are what make sharing both
possible and needed.

For firmware, what is common is the **environment** (image, provenance,
recursive checkout, `uv sync`), not the build: make and cmake differ in the
build command and in the matrix source, and both stay in the project. Moving a
project from make to cmake changes one line in that project, nothing here.

## Becoming a caller

1. **Lint toolchain first.** `uv run` installs nothing, so `ruff`, `pylint` and
   `pyright` must already be dev dependencies of the project. Coming from
   black + pycodestyle, that is its own commit: add `ruff`, drop `black` and
   `pycodestyle`, delete `pylintrc` / the pycodestyle part of `setup.cfg`,
   regenerate `uv.lock` **in the same commit** (`--locked` fails on a lock that
   no longer matches), then fix what the tools report. Template: paramschema
   PR #8 (`[tool.ruff] line-length`, `[tool.ruff.lint] select = ["E", "W", "F",
   "I"]`, `line-too-long` disabled in pylint since E501 reports it).
   A `[tool.black] ignore = [...]` is a dead setting (black has no such key,
   pycodestyle never reads `pyproject.toml`): do not port it.
2. **pyright `exclude`.** pyright walks a directory and reads neither ruff's
   `extend-exclude` nor pylint's `ignore-paths`. Without an `exclude` it lints
   the vendored submodules (64 of domo_modbus's 69 errors were in `lib_ext/`).
   Setting `exclude` *replaces* the defaults, so repeat `**/__pycache__` and
   `**/.*`. A `pyrightconfig.json` wins over `[tool.pyright]`: keep only one.
3. **The caller**, ~30 lines, same shape as paramschema's, keeping the
   project's own trigger.
4. Fix the backlog **before** bumping the `soft-lib/ci` pin that brings a new
   gate. Bumped first, every push is red until the last fix lands.

What pyright found on the way is the argument for the no-opt-out rule: three
real bugs no linter or test had caught (`logging.handlers` used after a bare
`import logging`; a `None` guard placed after the call that raised on `None`;
`.strip()` on a value a caller can set to `None`).

## Pin rules

- **A pinned sha must stay reachable from `master`.** Forgejo resolves
  `uses: …@<sha>` against the repository, not against a branch: a pushed,
  unmerged commit already works. But a commit reachable from no branch is
  garbage. Merge with a **merge commit**; a squash or rebase merge followed by a
  branch deletion silently breaks every caller pinning the branch's sha.
- **Image tags carry no order.** To know which `ci-<sha>` is newer, read the
  registry's publication date or the history of `soft-lib/docker-images`, never
  the sha. (Two pins were once read backwards this way.)
- **`git fetch` before branching.** A stale local `master` worktree gives a
  healthy-looking branch that collides on the pin line at merge time.

## Provenance stamp: `/etc/image-info`

Every image carries `/etc/image-info`, and each workflow `cat`s it first:

```sh
IMAGE_NAME="python3"
IMAGE_VARIANT="ci"
IMAGE_COMMIT="301738e16e6da8b7d8d9892fca0569f9e84aba99"
```

Quoted and prefixed so that it is both readable and sourceable (sh, bash, zsh)
without clobbering a caller's `COMMIT`. Not `/etc/os-release`: in the image it
is a symlink to Debian's vendor file. Not `*-release` / `*-version`: the former
is the legacy distro pile `os-release` replaced, the latter names a field the
file does not hold.

It replaced `/etc/pnb-image` with **no transition period**: the pin scheme
guarantees old images stay in service, so a fallback path would hide exactly
the pin left behind. The image pin and the `soft-lib/ci` pin moved in the same
commit, per project. The rename touched 18 files in three repositories, and
zero in the projects that were already callers.

## Forgejo behaviour worth knowing

- **Workflows cannot be inherited.** A workflow file must exist in the
  project's `.forgejo/workflows/`; a submodule cannot carry one (nothing is
  checked out at parse time). A composite action in a submodule would work,
  but it costs a submodule per project to do what `uses: owner/repo@<sha>` does
  for free.
- **`.forgejo/workflows/`, not `.github/workflows/`.** Forgejo reads the
  former first and GitHub ignores it, so a GitHub mirror stops running jobs
  whose `uses:` and runner labels only resolve on this forge. Callers pinned
  before the move still name `.github/workflows/` in their `uses:`; that path
  changes with the bump that crosses the move.
- **Scheduled workflows run on the default branch only.** pnbchrono's
  `check-image-bump.yaml` canary never ran until its branch reached `master`.
- **A called workflow shows as two tasks**: a 0 s wrapper named `<job>`, which
  gets the caller's `name:`, and the real one, `<job>-1`. That is why
  `fw-build.yaml` names itself from `matrix-value`.
- **Bare `uses: actions/x@v` resolves against `DEFAULT_ACTIONS_URL`**
  (`data.forgejo.org`, a mirror of the GitHub actions), not github.com.
  `actions/upload-artifact@v4` refuses to run off github.com; Forgejo's fork at
  `https://code.forgejo.org/forgejo/upload-artifact` is used instead. That full
  URL is not GitHub syntax, but it is the smallest of the instance ties
  (`runs-on: debian` labels, `uses: soft-lib/ci/…`, `$FORGEJO_OUTPUT`): a
  GitHub mirror should carry the code, not the workflows.
- **Reading runs through the API**: `/api/v1/repos/{o}/{r}/actions/runs`
  (per workflow, `index_in_repo` is the number in the run URL) and
  `/actions/tasks` (per job, `name` + `status`). There is no log API on 15.x,
  but on a public repository the raw log of a job is readable without a login
  at `/{o}/{r}/actions/runs/{index_in_repo}/jobs/{n}/attempt/1/logs` (`n`
  counts from 0; a called workflow's real job is `1`, after the wrapper).
- Requires Forgejo 15.0.6 / runner v13 (see README): a reusable workflow called
  from a dynamic matrix needs forgejo#10647.

## Lessons from debugging a shared job

`build-doc` stayed red through six probe rounds. The cause was a convenience
step in `fw-build.yaml`, `ls "$path" | head`: the runner runs steps under
`bash -e -o pipefail`, `head` closes the pipe after ten lines, `ls` dies of
SIGPIPE (exit 141) and the step fails, so the upload never runs. It fires only
when there is enough output: two files pass, 3786 files of doxygen do not.

- **Build probes from the difference between the probe and production**, not
  from the current hypothesis. Every probe uploaded directly or wrote two files,
  so none of them went through the broken step.
- **When the failure is in a job you cannot read, ask for the log first.** Its
  last twenty lines named the cause.
- Never truncate with a pipe under `pipefail`; count instead
  (`find … | wc -l` reads to the end).

## Debian packages: `deb-build.yaml`

- **One input, on purpose.** Build command, test command, artifact path and
  version are all derivable from `debian/` (`build-dep ./`,
  `dpkg-buildpackage -b`, `../*.deb`, `dpkg-parsechangelog`), so they are not
  inputs. A package that needs something else changes its `debian/`, as it
  would for Debian.
- **Checkout into `source/`.** dpkg-buildpackage always writes its results in
  `..` (the `.deb`s come from `dh_builddeb`, whose `--destdir` belongs to the
  package's `debian/rules`), and the `.changes` names its files without a
  path, so they must stay together. With the tree one level down, `..` is the
  job's workspace: the results land there with nothing else, as in sbuild and
  salsa-ci. A first version built in the workspace root and moved `../*.deb`
  into `out/`, i.e. globbed the runner's directory of all the owner's
  workspaces (review of #13).
- **The image carries the tools, not the package's dependencies.**
  `debian-pkg` has build-essential (which `dpkg-buildpackage` requires even
  for `Architecture: all`), debhelper, lintian and autopkgtest; Build-Depends
  are installed per job. Before it, a caller on `node:24-trixie` failed on
  `unmet build dependencies: build-essential:native`.
- **autopkgtest `null`**: the job's container is already disposable, so the
  tests run in it rather than in a nested testbed. `null` cannot revert the
  system, so a test declaring `Restrictions: breaks-testbed` is skipped, and
  all tests skipped is exit 8: a red job. Purging the package is not breaking
  the testbed; leave that restriction out.
- **Install and purge before autopkgtest** (piuparts' order). autopkgtest
  installs the package itself and a test may purge it, after which apt no
  longer knows the local `.deb` (`Unable to locate package`): nothing may
  touch the package after it. First found on yt-add-music-beet's
  `install-purge` test.
- **Secrets cross `workflow_call`** only when the caller passes them
  (`secrets: PACKAGE_TOKEN: …` or `secrets: inherit`); the workflow declares
  the secret as optional and fails the publish step if a tag arrives without
  it.
- **Tag = changelog version** (DEP-14 mangling of `~` and `:`). A registry
  version is immutable (409), so a rebuild is a new changelog entry.

## Ruled out

- **One job for lint-reuse and python-checks**: tried (`4368e1c`), reverted
  (`738d0cd`). REUSE is not a python check, and a job is the unit the run list
  shows: merged, one tool's failure masks the other's verdict. Share the
  environment, not the verdict.
- **Workflows in a git submodule**: impossible, see above.
- **Keeping the copy-from-skeleton scheme**: it is what produced the drift.
- **Calling the project Makefile from CI**: ties the CI to the file meant to
  disappear, to save one line.
- **A reusable workflow local to one project** (`uses: ./…`): other projects
  cannot call it.
- **Image pins in Forgejo `vars` or centralised here**: puts the pin outside
  the git history of the project it protects, against the premise of
  `check-image-bump`.
