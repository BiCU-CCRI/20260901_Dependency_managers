# Modern dependency managers, a hands-on tour

[![Open in GitHub Codespaces](https://github.com/codespaces/badge.svg)](https://codespaces.new/BiCU-CCRI/20260901_Package_managers_demo)

A short tutorial that introduces modern dependency and environment managers with a focus on **reproducibility and portability**. It runs entirely in a GitHub Codespace, so there is nothing to install on your own machine.

Tools covered, with hands-on exercises:

- `uv` (Python projects and environments)
- `pixi` (Conda-ecosystem projects, especially useful for bioinformatics)
- `pipx` (isolated Python command-line tools)
- `pixi global`, `condax`, and `conda-global` (isolated Conda tools/applications)
- `renv` (reproducible R environments)

Other tools worth knowing are listed near the end of this file.

## Why bother, if Conda and virtualenv already work?

They do work, but they have well-known pain points that the newer tools were
built to fix:

- Older Conda versions used the classic solver, which could be very slow for
  large environments. Since Conda [23.10](https://docs.conda.io/projects/conda/en/23.10.x/release-notes.html),
  `libmamba` solver has been the default, so speed is no longer a reason. Conda still centers and maintains
  a special base environment, which users can accidentally pollute by installing
  project dependencies into it.
- `pip` plus `venv` or `virtualenv` does not automatically maintain a project lockfile for normal installs.
  A hand-written requirements.txt often contains only direct requirements or
  broad version ranges. `pip freeze` snapshots an already-installed environment. Recent pip versions also have an experimental
  `pip lock` command, but locking is not the default project workflow in the
  same way it is with uv.
- Mixing `conda install` and `pip install` in the same environment can lead to
  difficult dependency conflicts, especially if Conda is used again after pip
  has modified the environment.
- Traditional Conda and pip workflows do not automatically keep a
  fully resolved, version-controlled description of the environment in sync
  with every dependency change. Both can export detailed environment
  specifications, and recent versions also have additional lockfile support,
  but this is not the default workflow.

Tools such as pixi and uv are implemented in Rust and are designed around fast
dependency resolution and automatically maintained project lockfiles. For research
code, that lockfile is particularly valuable: it records the resolved
dependency graph rather than just the dependencies you requested, making it
much easier for a colleague, a reviewer, or you-in-two-years to reconstruct the
software environment.

> [!NOTE]
> A lockfile substantially improves reproducibility, but it is not the whole
> story. Interpreter versions, target platforms, package availability, system
> libraries, GPU drivers, and the container/OS layer can still matter.

## The landscape of dependency managers

There are two main use cases:

- **Project/environment managers** — create a reproducible environment for a
  particular project.
- **Application managers** — install command-line applications into isolated
  environments and expose their executables user-globally.

> [!NOTE]
> These categories are not mutually exclusive: `uv` supports both through its
> project commands and `uv tool`, while pixi supports both through projects and
> `pixi global`.

| Tool | Role | Ecosystem | Written in | Lockfile / reproducibility | Roughly replaces |
| --- | --- | --- | --- | --- | --- |
| `uv` | Project/env manager | PyPI / Python | Rust | `uv.lock` | pip, virtualenv, pip-tools, poetry, pyenv |
| `uv tool` | Application manager | PyPI / Python | Rust | no project lockfile | `pipx`; isolated Python CLI applications |
| `pixi` | Project/env manager | Conda + PyPI | Rust | `pixi.lock` | conda/mamba for project environments |
| `pixi global` | Application manager | Conda | Rust | `pixi-global.toml` manifest, not `pixi.lock` | `condax`; Conda analogue of `pipx` |
| `renv` | Project/env manager | CRAN, Bioconductor, GitHub and other R sources | R | `renv.lock` | manual per-project R library management |
| `pipx` | Application manager | PyPI / Python | Python | ordinary installs are not automatically locked; current versions also support explicit manifest/PEP 751 lock workflows | user-level `pip install` for Python CLI tools |
| `condax` | Application manager | Conda | Python | none | pipx-style isolated Conda applications |
| `conda-global` | Application manager / Conda plugin | Conda | Python + Rust launchers | `~/.conda/global.toml` manifest, not a project lockfile | Conda-native alternative to `condax` / `pixi global` |

For comparison, the traditional tools are:

| Baseline tool | Role | Ecosystem | Reproducibility mechanism |
| --- | --- | --- | --- |
| `conda` / `mamba` | Environment/package manager | Conda, with pip interoperability | `environment.yml`, explicit specs; current Conda also supports native multi-platform lockfiles and `pixi.lock` consumption |
| `venv` / `virtualenv` + `pip` | Python environment + package installer | PyPI / Python | `requirements.txt`, `pip freeze`; recent pip also has experimental PEP 751 locking |

> [!NOTE]
> **Manifest != lockfile.** `pyproject.toml`, `pixi.toml`, `environment.yml`,
> and global-tool manifests primarily describe what you want. `uv.lock`,
> `pixi.lock`, `renv.lock`, Conda lockfiles, and PEP 751 `pylock.toml` describe
> concrete resolved dependencies.

## Pros and cons

### `uv`

- **Pros**: extremely fast; one tool for Python versions, virtual environments,
  dependencies, and CLI tools. Maintains `uv.lock`. `uv run` executes directly
  inside the project's managed environment, so there is no shell-activation step
  to remember. Works with `pyproject.toml` and can import dependencies.
- **Cons**: focused on Python and the Python packaging ecosystem. It cannot
  manage arbitrary non-Python tools such as `samtools` or `bwa`. uv is still
  versioned `0.x`, although Astral considers it stable software; minor releases
  may intentionally contain breaking changes (pin uv itself in CI when that
  matters).

### `pixi`

- **Pros**: built on the Conda ecosystem, so it installs from `conda-forge`,
  `bioconda`, and other channels, including compiled tools, system libraries,
  Python, R, and other languages. Normal project operations maintain
  `pixi.lock`, and no special `base` environment is required. PyPI
  dependencies are resolved with uv's resolver and coordinated with the
  Conda environment. One lockfile can hold resolutions for multiple declared
  platforms. Existing `environment.yml` files can be imported.
- **Cons**: introduces a pixi project model and manifest (`pixi.toml`, although
  configuration can also live in `pyproject.toml`). The project is still
  evolving. Multi-platform locking does not make packages available on platforms
  where no compatible build exists.

### `renv`

- **Pros**: widely used for reproducible R project libraries. Gives each project
  its own library, discovers R package dependencies, records exact package
  versions and sources in `renv.lock`, and supports packages from CRAN, Bioconductor, GitHub,
  and other repositories.
- **Cons**: R-specific. It records the R version used by the project but does
  not install/manage R itself or arbitrary operating-system libraries. Restoring
  source packages can be slow when binaries are unavailable.

### `pipx`

- **Pros**: clean isolation for Python CLI applications. Each application gets
  its own virtual environment, so Python dependencies do not collide. `pipx run`
  can run an application without permanently installing it.
- **Cons**: focused on Python applications and does not manage arbitrary system
  dependencies. Ordinary `pipx install` is about isolation/convenience rather
  than project locking. Current pipx also has explicit manifest/PEP 751 lock
  workflows, but these are separate from a research project's own lockfile.

### `uv tool`

- **Pros**: the pipx-style application manager built into uv. Each installed
  Python CLI application gets its own isolated environment, while executables
  are exposed on `PATH`. `uv tool run` / `uvx` can run a tool in an isolated
  temporary/cached tool environment. It is extremely fast and avoids needing a
  separate `pipx` installation if you already use `uv`.
- **Cons**: Python and the Python packaging ecosystem only.
  Globally installed tools are not recorded in the project's `uv.lock`, so
  tools required to reproduce an analysis should normally be project
  dependencies and run with `uv run` instead.

### `pixi global`

- **Pros**: the `conda/pipx`-style application manager built into pixi. It can install
  command-line applications from the Conda ecosystem, including compiled and
  non-Python tools from `conda-forge`, `bioconda`, and other Conda channels into isolated environments and expose their commands on `PATH`. `pixi-global.toml` records the configured global environments.
- **Cons**: the global manifest is not a project `pixi.lock`, and each global
  environment is resolved for one platform. Research-critical tools should be
  project dependencies instead rather than installed with `pixi global`.

### `condax`

- **Pros**: the older "pipx for Conda" pattern: each Conda-packaged application
  is installed into its own isolated environment and exposed on `PATH`.
- **Cons**: development has slowed considerably (last release is from 2024). For new setups, prefer
  `pixi global` or the newer `conda-global` plugin.

### `conda-global`

- **Pros**: a newer Conda-native application manager from `conda-incubator`.
  Like `pipx`, `condax`, and `pixi global`, it installs each command-line tool
  into its own isolated environment and exposes the executable globally. It
  works across the Conda package ecosystem, so it can install compiled and
  non-Python applications as well as Python tools. Commands such as
  `conda global install` expose small native trampoline launchers that forward
  execution into the right environment without manual activation. `~/.conda/global.toml` manifest records configured tools.
- **Cons**: it is a newer project and less mature than `pixi global` or `pipx`.
  It requires an existing Conda installation. Its `global.toml` is a manifest rather than a full dependency
  lockfile, so globally installed applications should not be relied upon for
  software that must be reproduced exactly as part of a research workflow.

## Which one should I use?

For research work where **reproducibility and portability are the priority**:

- **Pure Python project** — a package, script, or analysis whose dependencies
  all come from the Python packaging ecosystem: use `uv`.

  Commit `pyproject.toml`, `uv.lock`, and `.python-version` with an appropriately specific Python version.

- **Project that needs compiled tools, system libraries, mixed languages, or
  Bioconda packages** — common in bioinformatics: use `pixi`.

  Commit both `pixi.toml`, `pixi.lock`, and explicitly list every platform
  you intend to support.

- **R-only project or analysis**: use `renv` when reproducibility of the R
  package library is the main concern.

  Commit `renv.lock`. However, `renv` records the required R version
  but does not install it. If the
  project also depends on a particular R interpreter, compiled libraries, or
  external command-line tools, use a container or `pixi` to manage those parts of the
  environment as well.

- **Command-line tools that are part of the analysis or pipeline**: put them in
  the project's `uv` or `pixi` environment rather than installing them
  globally.

- **Personal command-line utilities that do not affect the project's results**:
  use `uv tool` / `pipx` for Python applications, or `pixi global` /
  `conda-global` for Conda applications.

  These are convenient for tools you want available everywhere, but they
  should not be relied upon for software that is required to reproduce an
  analysis because their state is not captured by the project's dependency
  lockfile.

You do not have to pick just one. A reproducibility-focused setup might be:

- `uv` for pure-Python projects;
- `pixi` for bioinformatics, mixed-language, and system-dependency-heavy
  projects;
- `renv` for R package libraries, optionally combined with `pixi` when the
  R interpreter or external libraries also need to be pinned;
- `uv tool` or `pixi global` only for personal utilities that are not part of
  the reproducible workflow.

The key rule is:

> **If the result of the research depends on a program, make that program a
> project dependency and put it in the project lockfile.**

## Other tools worth mentioning

- `mamba`: fast C++ drop-in replacements for `conda`. Modern Conda
  itself already defaults to libmamba solving, so solver speed alone is no
  longer the reason to choose Mamba.
- `micromamba`: a small standalone Conda-compatible executable with no populated
  base environment, useful in containers and lightweight setups.
- `poetry`, `pdm`, `hatch`: earlier-generation Python project managers. Still
  common, but `uv` is faster and covers most of what they do.
- `rye`: an experimental Python manager that has been folded into `uv`, so new
  users should start with `uv` directly.
- `spack` and `easybuild`: build-from-source managers used on HPC clusters,
  where compiler and hardware tuning matter.
- `Nix` and `GNU Guix`: whole-system reproducible package managers. Very
  powerful and very reproducible, with a steeper learning curve.
- Containers (`Docker`, `Apptainer`/`Singularity`): an outer reproducibility
  layer that captures much more of userspace than a package lockfile. Containers
  still share the host kernel and can depend on hardware/drivers. For archival
  work, pin the base image by **digest** and preserve the built image as well as the
  lockfile.

## References

- `uv`: <https://docs.astral.sh/uv/>
- `uv tool` / `uvx`: <https://docs.astral.sh/uv/concepts/tools/>
- `pixi`: <https://pixi.sh/latest/>
- `pixi global`: <https://pixi.prefix.dev/latest/global_tools/introduction/>
- `renv`: <https://rstudio.github.io/renv/>
- `pipx`: <https://pipx.pypa.io/>
- `condax`: <https://mariusvniekerk.github.io/condax/>
- `conda-global`: <https://conda-incubator.github.io/conda-global/>
- `conda`: <https://docs.conda.io/projects/conda/en/stable/>
- `mamba` / `libmamba` / `micromamba`: <https://mamba.readthedocs.io/en/stable/>
- `pip`: <https://pip.pypa.io/en/stable/>
- `venv`: <https://docs.python.org/3/library/venv.html>
- `virtualenv`: <https://virtualenv.pypa.io/>

## Running this tutorial

1. Open this repository in a GitHub Codespace using the badge above, or choose
   **Code -> Codespaces -> Create codespace on this branch**. The first build
   takes a few minutes because it installs R and renv.
2. Open the exercises in order under `exercises/`. Each is a short Markdown file
   with copy-paste commands and expected behavior.

> [!IMPORTANT]
> The tutorial container and installer commands intentionally use some floating
> tool/image versions to keep the live session simple. That is a useful boundary
> of the demo: a project lockfile does not automatically pin the devcontainer,
> package-manager executable, OS kernel, or external repositories. See
> `reproducibility.md` for the stronger archival setup.

## Supporting material

- `exercises/CHEATSHEET.md`: a one-page command reference and common gotchas.
- `reproducibility.md`: practices focused on reproducibility and portability
  (locks, target platforms, strict CI restores, HPC, and containers).
- `limitations.md`: failure modes, symptoms, and workarounds.
- `exercises/environment.yml`: a small sample Conda environment used by the pixi
  import step in exercise 2.
