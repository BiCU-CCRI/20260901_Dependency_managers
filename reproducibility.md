# Reproducibility and portability tips

The goal is not merely to isolate dependencies, but to make it practical to
rebuild a research environment on another machine and to notice when that
rebuild would require a new resolution.

## The core idea: manifest versus lockfile

A modern project usually separates two concepts:

- **Manifest**: what you requested, often including version ranges. Examples:
  `pyproject.toml`, `pixi.toml`, and `environment.yml`.
- **Lockfile**: the concrete dependency resolution, including transitive
  dependencies and often hashes/build/source information. Examples: `uv.lock`,
  `pixi.lock`, `renv.lock`, Conda lockfiles, and PEP 751 `pylock.toml`.

A lockfile is a major part of reproducibility, but it is not the whole system.
The interpreter, target platform, native libraries, package availability, GPU
stack, and container/OS layer can all matter too.

## Commit the right files, ignore the environment itself

| Tool | Commit to git | Git-ignore |
| --- | --- | --- |
| uv | `pyproject.toml`, `uv.lock`, `.python-version` | `.venv/` |
| pixi | `pixi.toml` (or Pixi config in `pyproject.toml`), `pixi.lock` | `.pixi/` |
| renv | `renv.lock`, `.Rprofile`, `renv/activate.R`, `renv/settings.json` | `renv/library/` |

Do not commit the materialized environment directory. Rebuild it from the
committed specification instead.

## Pin the interpreter deliberately

Package versions alone are not enough if the interpreter drifts.

- uv: `uv python pin 3.11` pins the 3.11 series, while an exact request such as
  `uv python pin 3.11.13` is stronger when you need the same patch release.
- Pixi: add Python or R as normal dependencies, for example
  `pixi add "python=3.11.13"`, so the interpreter is part of the locked Conda
  resolution.
- renv: `renv.lock` records the R version used, but renv does not install R.
  Pair it with Pixi, a container, or another runtime-management layer when the
  exact R interpreter matters.

## Make the lock portable across target platforms

Portability does **not** mean one binary artifact runs everywhere. It means the
project has a pre-resolved, tested dependency solution for each platform you
claim to support.

### Pixi

Pixi resolves every declared target platform and stores those resolutions in one
`pixi.lock`:

```bash
pixi workspace platform add linux-64 osx-64
pixi install
```

Add only platforms you intend to support and test. If a Bioconda package has no
build for `osx-arm64`, declaring that platform cannot make it available.

### uv

uv produces a universal project lock that can encode conditional resolutions
across supported Python versions and platforms. The selected wheel/artifact can
still differ by operating system or architecture.

### renv

renv records package names, versions, and sources. A restore on another platform
may download a different binary or compile from source, so native system
requirements can still differ.

### Conda

Modern Conda can also create and consume exact multi-platform lockfiles. This is
important context when comparing current Conda with Pixi: the main difference is
in workflow and defaults, not that Conda is incapable of locking.

## Fail loudly when the lock is stale

In CI, do not silently rewrite a lockfile because a manifest changed.

- uv: `uv sync --locked`; `uv lock --check` verifies the lock without installing.
  `--frozen` means use the lock as-is without checking freshness.
- Pixi: `pixi install --locked`; `--frozen` uses the existing lock without
  updating it.
- renv: `renv::restore()` installs from the lock; `renv::status()` reports drift.

A strict restore answers a valuable question: **does this exact commit already
contain everything needed to rebuild its environment?**

## Reproduce on a new machine

| Tool | Typical command | Notes |
| --- | --- | --- |
| uv | `uv sync --locked` | use the committed `uv.lock` |
| pixi | `pixi install --locked` | uses the current platform's entry from `pixi.lock` |
| renv | `renv::restore(prompt = FALSE)` | run from the project directory |
| conda | create/install from an exact Conda lockfile | current Conda supports native lock workflows |

For application managers such as `uv tool`, `pipx`, `pixi global`, and
`conda-global`, remember that isolation is not automatically the same as project
reproducibility. If a research result depends on the application, prefer to make
it a project dependency.

Current pipx also has explicit manifest and PEP 751 lock workflows, which can be
useful for reproducing a personal set of Python CLI applications. That still does
not make those tools part of a research project's own dependency graph unless
the project records them.

## HPC and offline systems

- Pixi and uv install in user space; they do not require a shared cluster-wide
  environment to be writable.
- Resolve/download on a login node when compute nodes have no network access.
- uv supports offline installs from a primed cache (`uv sync --offline`).
- For Pixi/Conda, keep package caches on suitable shared or scratch storage when
  home-directory quotas are small.
- For renv, binary repositories can avoid expensive source builds; otherwise
  ensure compilers and system libraries are available.
- On clusters where containers are supported, Apptainer is often the practical
  outer portability layer.

## Re-resolution is a separate reproducibility problem

A committed lockfile preserves the resolution you already have. If you delete
or regenerate it years later, the set of available packages may have changed.

uv can constrain fresh resolution by release date, for example:

```bash
uv lock --exclude-newer 2026-01-01
```

For long-term archival work, also preserve critical package artifacts or a built
container image. A lockfile cannot download an upstream artifact that no longer
exists.

## Containers are an outer layer, not a replacement for locks

A container captures substantially more userspace state than a package lockfile,
but it is not literally the whole computer. Containers normally share the host
kernel and may depend on host CPU/GPU hardware and drivers.

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

Also pin the base image by digest for archival work. A Dockerfile using
`FROM ubuntu:latest` and unpinned network installs is not a deterministic build.

## The tutorial itself intentionally leaves one layer floating

The live Codespace uses floating image/feature/tool installer versions to keep a
30–40 minute workshop simple. That is deliberate teaching material: after you
lock the project dependencies, ask what is still unpinned.

Typical answers include:

- the devcontainer base image and features;
- the uv/Pixi executable versions installed by the workshop;
- the R version installed by the devcontainer feature;
- external repositories and upstream artifact availability;
- the host kernel and hardware.

For a paper, regulated workflow, or archival pipeline, pin those layers too.

## Where package lockfiles stop

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
