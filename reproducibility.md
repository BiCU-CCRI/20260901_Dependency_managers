# Reproducibility and portability tips

The whole reason to move past plain Conda and virtualenv is that the newer tools
make an environment reproducible (the same versions every time) and portable
(the same environment on your laptop, a colleague's machine, CI, and an HPC
cluster). This page collects the practices that matter, with the exact commands.

## The one idea: manifest versus lockfile

Every modern tool separates two files:

- The **manifest** is what you asked for, usually version ranges. Examples:
  `pyproject.toml` (`uv`), `pixi.toml` (`pixi`), the packages you install in R
  (`renv`). It is short and **human-edited**.
- The **lockfile** is what was actually resolved: the exact version and often a
  hash of every package, including transitive dependencies. Examples: `uv.lock` (`uv`),
  `pixi.lock` (`pixi.lock`), `renv.lock` (`renv`). It is **machine-written**.

Reproducibility comes from the **lockfile**, not the manifest. A `requirements.txt`
or an `environment.yml` with ranges **records intent**, **not** the **resolved result**, so
two people can install "the same" environment and get different versions.

**Rule of thumb**: commit both files, and let the tool rebuild from the lockfile.

## Commit the right files (and ignore the rest)

The environment directory itself is large, machine-specific, and never belongs
in git. Commit the manifest and the lockfile instead.

| Tool | Commit to git | Git-ignore |
| --- | --- | --- |
| uv | `pyproject.toml`, `uv.lock`, `.python-version` | `.venv/` |
| pixi | `pixi.toml`, `pixi.lock` | `.pixi/` |
| renv | `renv.lock`, `.Rprofile`, `renv/activate.R`, `renv/settings.json` | `renv/library/` |

`renv` writes its own `renv/.gitignore` so the library is excluded automatically.

## Pin the interpreter, not just the packages

An environment is not reproducible if the Python or R version can drift. Pin it:

- uv: `uv python pin 3.10.4` writes `.python-version`; `uv sync` then uses exactly
  that interpreter, downloading it if needed.
- pixi: add the interpreter as a normal dependency, `pixi add "python=3.10.4"`, so
  it is captured in `pixi.lock` like everything else.
- renv: `renv.lock` records the R version used. renv does not install R itself,
  so pair it with the system, a container, or pixi to pin the interpreter too.

## Make lockfiles portable across platforms

This is where the newer tools clearly beat Conda for research - when you need to move between devices (including CI, codespaces, etc.):

- pixi solves for every platform you declare and stores all of them in one
  `pixi.lock`. Declare them once:

  ```bash
  pixi workspace platform add linux-64 osx-arm64 win-64
  pixi install
  ```

  Now the same lockfile reproduces on all four, so a Mac user and a cluster
  running Linux get matching, pre-resolved environments from the same commit.

- uv produces a universal lockfile by default: a single `uv.lock` that captures
  resolution across platforms and Python versions, so `uv sync` works on
  whatever machine checks it out.

- renv records package name, version, and source, which is platform-independent.
  On `restore()`, renv fetches the right binary for the current OS (or builds
  from source), so the lockfile transfers even though the installed artifacts
  differ per platform.

## Fail loudly when the lock is stale

In automation you want the build to error if someone changed the manifest but
forgot to update the lockfile, rather than silently re-resolving to new
versions. Each tool has a strict mode:

- uv: `uv sync --locked` (error if `uv.lock` is out of date) or `uv sync --frozen`
  (install from the lock as-is without checking the manifest). `uv lock --check`
  verifies without installing.
- pixi: `pixi install --locked` (error if `pixi.lock` is out of sync with the
  manifest) or `pixi install --frozen` (install from the lock as-is without checking the manifest). These are
  also settable via `PIXI_LOCKED=true` / `PIXI_FROZEN=true`.
- renv: `renv::restore()` already installs strictly from the lockfile;
  `renv::status()` reports drift between the lockfile, the library, and the code.

## Reproduce on a new machine

| Tool | Command | Notes |
| --- | --- | --- |
| uv | `uv sync` | add `--locked` in CI |
| pixi | `pixi install` | add `--locked` in CI |
| renv | `renv::restore(prompt = FALSE)` | run in the project directory |
| pipx | `pipx install-all snap.json` | after `pipx list --json > snap.json` |
| conda (baseline) | `conda env create -f environment.yml` | no exact lock, versions may drift |

## Transferring to an HPC cluster

- pixi needs no admin rights and no cluster-wide Conda install. It installs into
  your home directory and drops the tools into a project-local `.pixi`.
- Use `pixi install --frozen` on the cluster so a login node with a different
  network or clock cannot quietly change your versions.
- uv can run fully offline once packages are cached (`uv sync --offline`), which
  is useful on compute nodes without internet access. Create the cache on a login
  node first.
- For renv on a cluster, configure a binary package repository (for example, [Posit Public
  Package Manager](https://packagemanager.posit.co/)) so `restore()` installs binaries instead of compiling, which
  is far faster and avoids missing compiler issues.

## Date-pin for extra determinism

Even a lockfile can be regenerated with newer packages later. To make a fresh
resolution deterministic, cap it to a date:

- uv: `uv lock --exclude-newer 2026-01-01` refuses any package released after
  that date. Handy for "resolve this the way it would have resolved back then".

Combined with a committed lockfile, this makes both the current build and any
future re-resolution reproducible.

## Containers are the outermost layer

A lockfile pins packages, but not the operating system, system libraries, or
compilers. When you need that level of reproducibility (papers, pipelines,
regulated work), wrap the environment in a container:

- Put `pixi` or `uv` inside the image and run `pixi install --frozen` or
  `uv sync --frozen` during the build, so the container is built from your
  committed lockfile. The lockfile inside the container is what makes it
  reproducible; a bare `pip install` in a Dockerfile is not.
- On HPC, prefer Podman (or rootless Docker), or Apptainer (formerly Singularity). It runs without root and is
  the usual container runtime on shared clusters.
- This layering is complementary: lockfile for package versions, container for
  the system around them.

## Migrating between tools

Examples of how to migrate between tools:

- Conda to pixi: `pixi init --import environment.yml` (see exercise 2).
- pip to uv: `uv` reads an existing `requirements.txt` (`uv add -r requirements.txt`)
  and an existing `pyproject.toml`.
- Capturing pipx tools for a new machine: `pipx list --json > snap.json`, commit
  it, then `pipx install-all snap.json` elsewhere.

## Lockfiles are not everything

Be honest about the limits so people trust the tooling:

- Lockfiles pin package versions and hashes, not the compiler, kernel, glibc, or
  GPU driver. Two machines can install the identical locked packages and still
  behave differently if the system layer differs.
- For bit-for-bit builds independent of the host, the stronger tools are
  containers (Apptainer, Docker) and whole-system managers (Nix, GNU Guix).
- The practical ladder for research: lockfile first (cheap, covers most needs),
  then a container when results must be portable and archival, then Nix or Guix
  only if you truly need reproducible builds down to the system libraries.
