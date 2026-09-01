# Reproducibility and portability tips

The whole reason to move past plain Conda and virtualenv is that the newer tools
make an environment more reproducible (the same versions every time) and portable
(the same environment on your laptop, a colleague's machine, CI, and an HPC
cluster).

## The core idea: manifest versus lockfile

A modern project usually separates two concepts:

- **Manifest**: what you requested, often including version ranges. Examples:
  `pyproject.toml` (`uv`), `pixi.toml` (`pixi`), the packages you install in R
  (`renv`). It is short and **human-edited**.
- **Lockfile**: the concrete dependency resolution, including transitive
  dependencies and often hashes/build/source information. Examples: `uv.lock` (`uv`),
  `pixi.lock` (`pixi`), `renv.lock` (`renv`), Conda lockfiles, and PEP 751 `pylock.toml`. It is **machine-written**.

Reproducibility comes from the **lockfile**, not the manifest. A `requirements.txt`
or an `environment.yml` with ranges (~manifest) **records intent**, **not** the **resolved result**, so
two people can install "the same" environment and get different versions.

Neither of these solves everything — the interpreter, target platform, native libraries, package availability, GPU
stack, and container/OS layer can all matter too.

**Rule of thumb**: commit both files, and let the tool rebuild from the lockfile.

## Commit the right files (and ignore the rest)

The environment directory itself is large, machine-specific, and never belongs
in git. Commit the manifest and the lockfile instead.

| Tool | Commit to git | Git-ignore |
| --- | --- | --- |
| uv | `pyproject.toml`, `uv.lock`, `.python-version` | `.venv/` |
| pixi | `pixi.toml` (or pixi config in `pyproject.toml`), `pixi.lock` | `.pixi/` |
| renv | `renv.lock`, `.Rprofile`, `renv/activate.R`, `renv/settings.json` | `renv/library/` |

`renv` writes its own `renv/.gitignore` so the library is excluded automatically.

## Pin the interpreter, not just the packages

An environment is not reproducible if the interpreter (Python or R) version drifts. Pin it:

- uv: `uv python pin 3.11.13` writes `.python-version`; `uv sync` then uses exactly
  that interpreter, downloading it if needed.
- pixi: add the interpreter as a normal dependency, `pixi add "python=3.11.13"`, so
  it is captured in `pixi.lock` like everything else.
- renv: `renv.lock` records the R version used, but does not install it. Pair it
  with a container, or pixi to pin the interpreter too.

## Make the lock portable across target platforms

This is where the newer tools clearly beat Conda for research - when you need to move between devices (including CI, codespaces, etc.):

### Pixi

Pixi resolves for every platform you declare and stores all of them in one `pixi.lock`. Declare them once:

```bash
pixi workspace platform add linux-64 osx-arm64 win-64
pixi install
```

Add only platforms you intend to support and test. If a Bioconda package has no
build for `osx-arm64`, declaring that platform cannot make it available.

### uv

uv produces a universal lockfile by default: a single `uv.lock` that captures
resolution across platforms and Python versions, so `uv sync` works on
whatever machine checks it out.

### renv

renv records package name, version, and source, which is platform-independent. On
`restore()`, renv fetches the right binary for the current OS (or builds
from source), so the lockfile transfers even though the installed artifacts
differ per platform.

### Conda

Modern Conda can also create and consume exact multi-platform lockfiles. This is
important context when comparing current Conda with Pixi: the main difference is
in workflow and defaults, not that Conda is incapable of locking.

## Fail loudly when the lock is stale

In CI, do not silently rewrite a lockfile because a manifest changed. Each tool has a strict mode:

- uv: `uv sync --locked` (error if `uv.lock` is out of date) or `uv sync --frozen`
  (install from the lock as-is without checking the manifest, aka *freshness*). `uv lock --check`
  verifies without installing.
- pixi: `pixi install --locked` (error if `pixi.lock` is out of sync with the
  manifest) or `pixi install --frozen` (install from the lock as-is without checking the manifest). These are
  also settable via `PIXI_LOCKED=true` / `PIXI_FROZEN=true`.
- renv: `renv::restore()` installs strictly from the lockfile;
  `renv::status()` reports drift between the lockfile, the library, and the code.

## Reproduce on a new machine

| Tool | Typical command | Notes |
| --- | --- | --- |
| uv | `uv sync --locked` | use the committed `uv.lock` |
| pixi | `pixi install --locked` | uses the current platform's entry from `pixi.lock` |
| renv | `renv::restore(clean = TRUE, prompt = FALSE)` | run from the project directory |
| conda | create/install from an exact Conda lockfile | current Conda supports native lock workflows |

 > [!NOTE]
 > For application managers such as `uv tool`, `pipx`, `pixi global`, and
 > `conda-global`, remember that isolation is not automatically the same as project
 > reproducibility. If a research result depends on the application, prefer to make
 > it a project dependency.

>[!NOTE]
>Current pipx also has explicit manifest and PEP 751 lock workflows, which can be
>useful for reproducing a personal set of Python CLI applications. That still does
>not make those tools part of a research project's own dependency graph unless
>the project records them.
>
>| Tool | Typical command | Notes |
>| --- | --- | --- |
>| pipx | `pipx manifest sync pipx.toml` | use `pipx manifest lock pipx.toml` first for PEP 751-locked tools |

## HPC and offline systems

- pixi and uv themselves install in user space; project dependencies are placed in project-local `.pixi` or `.venv` directories.
- Use `pixi install --frozen` on the cluster so a login node with a different
  network or clock cannot quietly change your versions.
- Resolve/download on a login node when compute nodes have no network access.
- uv supports offline installs from a primed cache (`uv sync --offline`).
- For pixi/Conda, keep package caches on suitable shared or scratch storage when
  home-directory quotas are small.
- For renv on a cluster, configure a binary package repository (for example, [Posit Public
  Package Manager](https://packagemanager.posit.co/)) so `restore()` installs binaries instead of compiling, which
  is far faster and avoids missing compiler issues.
- On clusters where containers are supported, Podman (or rootless Docker) or Apptainer are often the practical
  outer portability layer.

## Date-pin for extra determinism

A committed lockfile preserves the resolution you already have. If you delete
or regenerate it years later, the set of available packages may have changed.

uv can constrain fresh resolution by release date, for example:

```bash
uv lock --exclude-newer 2026-01-01
```

Refuses any package released after that date. Handy for "resolve this the way it would have resolved back then".

For long-term archival work, also preserve critical package artifacts or a built
container image. A lockfile cannot download an upstream artifact that no longer
exists.

Combined with a committed lockfile, this makes both the current build and any
future re-resolution reproducible.

## Containers are the outermost layer

A lockfile pins packages, but not the operating system, system libraries, or
compilers. When you need that level of reproducibility (papers, pipelines,
regulated work), wrap the environment in a container:

A strong pattern is:

```text
project manifest + lockfile
        -> strict environment restore
        -> container image built from pinned inputs
        -> preserve the built image/digest
```

Inside a container build, use the lock rather than performing an unconstrained
fresh install:

```bash
uv sync --locked
# or
pixi install --locked
```

Also pin the base image by **digest** for archival work. A Dockerfile using
`FROM ubuntu:latest` and unpinned network installs is not a deterministic build.

This layering is complementary: lockfile for package versions, container for the
system around them.

## Migrating between tools

Examples of how to migrate between tools:

- Conda to pixi: `pixi init --import environment.yml` (see exercise 2).
- pip to uv: `uv` reads an existing `requirements.txt` (`uv add -r requirements.txt`)
  and an existing `pyproject.toml`.
- Capturing pipx tools for a new machine: `pipx list --json > snap.json`, commit
  it, then `pipx install-all snap.json` elsewhere.

## Lockfiles are not everything

Be honest about the limits so people trust the tooling - Lockfiles pin package
versions and hashes, not the compiler, kernel, glibc, or GPU driver. Two machines
can install the identical locked packages and still behave differently if the system
layer differs.

Use increasingly strong layers only when your project needs them:

1. project manifest + lockfile;
2. exact interpreter/runtime where relevant;
3. explicit supported platforms and CI rebuilds;
4. container image for the userspace/system layer;
5. package mirrors/artifact preservation for long-term archival recovery;
6. Nix/Guix or reproducible-build systems when build inputs themselves must be
   controlled more deeply.

The practical research rule remains simple: **if the result depends on a
program, record that program in the project, lock it, and test that the lock can
be restored from a clean environment.**
