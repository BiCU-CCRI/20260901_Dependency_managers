# Modern dependency managers, a hands-on tour

[![Open in GitHub Codespaces](https://github.com/codespaces/badge.svg)](https://codespaces.new/BiCU-CCRI/20260901_Package_managers_demo)

A short (about 30 to 40 minute) tutorial that introduces "new" dependency and
environment managers. It runs entirely in a GitHub Codespace, so there is nothing
to install on your own machine.

Tools covered, with hands-on exercises:

- `uv` (Python projects and environments)
- `pixi` (Conda-ecosystem projects, the natural upgrade path for bioinformatics)
- `pipx` (isolated Python command-line tools)
- `condax` (isolated Conda command-line tools) and its modern successors
- `renv` (reproducible R environments)

Other tools worth knowing are listed near the end of this file.

## Why bother, if Conda and virtualenv already work

They do work, but they have well-known pain points that the newer tools were
built to fix:

- Older Conda versions used the classic solver, which could be very slow for
  large environments. Since Conda [23.10](https://docs.conda.io/projects/conda/en/23.10.x/release-notes.html),
  however, the much faster `libmamba` solver has been the default. Conda also maintains
  a special base environment, which users can accidentally pollute by installing
  project dependencies into it.
- `pip` plus `venv` or `virtualenv` does not automatically maintain a lockfile.
  A hand-written requirements.txt often contains only direct requirements or
  broad version constraints. pip freeze can snapshot the exact versions of
  installed Python packages, including transitive dependencies, but this is a
  manual snapshot rather than a lockfile that stays synchronized with the
  project.
- Mixing `conda install` and `pip install` in the same environment can lead to
  difficult dependency conflicts, especially if Conda is used again after pip
  has modified the environment.
- Traditional Conda and pip workflows therefore do not automatically keep a
  fully resolved, version-controlled description of the environment in sync
  with every dependency change. Both can export detailed environment
  specifications, and recent versions also have additional lockfile support,
  but this is not the default workflow.

Tools such as Pixi and uv are implemented in Rust and are designed around fast
dependency resolution and automatically maintained lockfiles. For research
code, that lockfile is particularly valuable: it records the resolved
dependency graph rather than just the dependencies you requested, making it
much easier for a colleague, a reviewer, or you-in-two-years to reconstruct the
software environment.

>[!NOTE]
>A lockfile records the complete resolved dependency graph and enough artifact/platform
>information to make environment reconstruction substantially more reproducible.

## The landscape of depenency managers

There are two main use cases:

- **Project/environment managers** — create a reproducible environment for a
  particular project.
- **Application managers** — install command-line applications into isolated
  environments and expose their executables globally.

>[!NOTE]
>These categories are not mutually exclusive: `uv` supports both through its
>project commands and `uv tool`, while Pixi supports both through workspaces and
>`pixi global`.

| Tool          | Role                | Ecosystem                                              | Written in | Project lockfile                         | Roughly replaces                                             |
| ------------- | ------------------- | ------------------------------------------------------ | ---------- | ---------------------------------------- | ------------------------------------------------------------ |
| `uv`          | Project/env manager | PyPI / Python                                          | Rust       | `uv.lock`                                | pip, virtualenv, pip-tools, poetry, pyenv                    |
| `uv tool`     | Application manager | PyPI / Python                                          | Rust       | no lockfile                              | `pipx`; isolated installation of Python CLI tools            |
| `pixi`        | Project/env manager | Conda + PyPI                                           | Rust       | `pixi.lock`                              | conda/mamba for project environments                         |
| `pixi global` | Application manager | Conda                                                  | Rust       | no lockfile; `pixi-global.toml` manifest | `condax`; roughly the Conda analogue of `pipx`               |
| `renv`        | Project/env manager | CRAN, Bioconductor, GitHub and other R package sources | R          | `renv.lock`                              | manual per-project R library management                      |
| `pipx`        | Application manager | PyPI / Python                                          | Python     | none                                     | global/user `pip install` for Python CLI tools               |
| `condax`      | Application manager | Conda                                                  | Python     | none                                     | pipx-style isolated installation of Conda-packaged CLI tools |
| `conda-global` | Application manager | Conda                                                 | Python + Rust launcher | no lockfile; `~/.conda/global.toml` manifest | `condax`; Conda-native analogue of `pipx` and alternative to `pixi global` |

For comparison, the traditional tools are:

| Baseline tool                 | Role                                   | Ecosystem                        | Reproducibility mechanism                                                                                          |
| ----------------------------- | -------------------------------------- | -------------------------------- | ------------------------------------------------------------------------------------------------------------------ |
| `conda` / `mamba`             | Environment/package manager            | Conda, with pip interoperability | `environment.yml`, explicit specs; modern Conda also supports `conda-lock.yaml` and `pixi.lock`                    |
| `venv` / `virtualenv` + `pip` | Python environment + package installer | PyPI / Python                    | no automatically maintained project lockfile; `requirements.txt`, `pip freeze`, or PEP 751 `pylock.toml` workflows |

>[!NOTE]
>**manifest ≠ lockfile**. `pyproject.toml`, `pixi.toml`, `environment.yml`, and `pixi-global.toml`
>primarily describe what you want; `uv.lock`, `pixi.lock`, `renv.lock`, and Conda's newer
>lockfiles describe what was actually resolved, including transitive dependencies.
>Pixi explicitly makes that distinction in its lockfile documentation.

## Pros and cons

### uv

- **Pros**: extremely fast; one tool for Python versions, virtual environments,
  dependencies, and CLI tools. Maintains a real lockfile. `uv run` runs commands
  directly inside the project's managed environment, so there is no `activate`
  step to remember. Works with existing `pyproject.toml` projects and can import
  dependencies from `requirements.txt`.
- **Cons**: Python and PyPI only. It cannot install non-Python binaries such as
  `samtools` or `bwa`. Still pre-1.0, so occasional breaking changes.

### pixi

- **Pros**: built on the Conda ecosystem, so it can install packages from
  `conda-forge`, `bioconda`, and other Conda channels, including compiled tools,
  system libraries, Python, R, and other languages. It automatically maintains
  `pixi.lock` and does not require a special `base` environment. For PyPI dependencies, Pixi uses the `uv` resolver
  and coordinates those dependencies with the Conda-resolved
  environment. It can also import an existing `environment.yml`.
- **Cons**: introduces another project manifest format (`pixi.toml`), although
  Pixi can also store its configuration in `pyproject.toml`. The project is
  still evolving.

### `renv`

- **Pros**: the de facto standard for reproducible R project environments.
  Gives each project its own library, discovers R package dependencies,
  records exact package versions and sources in `renv.lock`, and supports
  packages from CRAN, Bioconductor, GitHub, and other repositories. The core
  workflow fits naturally into existing R and RStudio projects with just three commands.
- **Cons**: R-specific. It records the R version used by the project but does
  not install or manage the R interpreter itself, and it does not itself manage
  arbitrary operating-system libraries. Restoring packages from source can be
  slow when suitable binary packages are unavailable.

### `pipx`

- **Pros**: a clean way to install Python CLI applications. Each application
  gets its own virtual environment, so its Python dependencies do not collide
  with project libraries or with other installed tools. `pipx run` can run an
  application in a temporary environment without permanently installing it.
- **Cons**: focused on Python applications. Its virtual environments isolate
  Python package dependencies but do not manage arbitrary operating-system
  dependencies. It does not maintain a dependency lockfile for installed
  applications.

### `condax`

- **Pros**: essentially the Conda analogue of `pipx`: installs applications
  packaged for Conda into separate Conda environments and exposes their
  commands globally. This is useful for tools that are not available on PyPI,
  including many bioinformatics applications. Because these are full Conda
  environments, Conda-packaged compiled libraries and other dependencies are
  isolated along with the application.
- **Cons**: development has slowed considerably; the most recent PyPI release
  is from 2024. In 2026, more actively developed alternatives include
  `pixi global` and the `conda-global` project from [`conda-incubator`](https://github.com/conda-incubator). For a new
  setup, `pixi global` is generally the more mature choice; `conda-global` is
  newer and closely aligned with the Conda ecosystem.

### `uv tool`

- **Pros**: the `pipx`-style application manager built directly into `uv`. Each
  installed Python CLI application gets its own isolated virtual environment,
  while its executables are exposed on your `PATH`. `uv tool run` (or the
  shorter `uvx`) can run a tool in an isolated temporary environment without
  permanently installing it. It is extremely fast and avoids needing a
  separate `pipx` installation if you already use `uv`.
- **Cons**: Python and the Python packaging ecosystem only, so it cannot install
  arbitrary Conda packages or non-Python tools such as `samtools` or `bwa`.
  Globally installed tools are not recorded in the project's `uv.lock`, so
  tools required to reproduce an analysis should normally be project
  dependencies and run with `uv run` instead.

### `pixi global`

- **Pros**: the `pipx`-style application manager built into Pixi. It can install
  command-line applications from the Conda ecosystem, including compiled and
  non-Python tools from `conda-forge`, `bioconda`, and other Conda channels. By default, each tool is
  placed in its own isolated environment and its executables are exposed on
  your `PATH`. Pixi maintains a `pixi-global.toml` manifest describing the
  global environments, requested dependencies, channels, and exposed
  executables; this manifest can be version-controlled and shared.
- **Cons**: `pixi-global.toml` is a manifest, not a dependency lockfile like
  `pixi.lock`, so it does not provide the same exact reproducibility as a
  project-scoped Pixi environment. Each global environment is resolved for one
  platform. Tools that affect the results of an analysis should therefore
  normally be declared in the project's `pixi.toml` and captured in
  `pixi.lock`, rather than installed with `pixi global`.

### `conda-global`

- **Pros**: a newer Conda-native application manager from `conda-incubator`.
  Like `pipx`, `condax`, and `pixi global`, it installs each command-line tool
  into its own isolated environment and exposes the executable globally. It
  works across the Conda package ecosystem, so it can install compiled and
  non-Python applications as well as Python tools. It integrates directly into
  Conda as commands such as `conda global install` can run applications without activating their Conda
  environments (via Rust trampolines). A `~/.conda/global.toml` manifest records the configured global
  tools and environments.
- **Cons**: it is a newer project and less mature than `pixi global` or `pipx`.
  It requires an existing Conda installation, whereas Pixi is a standalone
  executable. Its `global.toml` is a manifest rather than a full dependency
  lockfile, so globally installed applications should not be relied upon for
  software that must be reproduced exactly as part of a research workflow.

## Which one should I use?

For research work where **reproducibility and portability are the priority**:

- **Pure Python project*- — a package, script, or analysis whose dependencies
  all come from the Python packaging ecosystem: use `uv`.

  Commit both `pyproject.toml` and `uv.lock`. The lockfile is universal and can
  describe the resolved dependencies needed across supported operating systems,
  architectures, and Python versions.

- **Project that needs compiled tools, system libraries, mixed languages, or
  Bioconda packages*- — as is common in bioinformatics: use `pixi`.

  Commit both `pixi.toml` and `pixi.lock`, and explicitly list every platform
  you intend to support. Pixi resolves each supported platform separately and
  stores all of those resolutions in the same lockfile.

- **R-only project or analysis**: use `renv` when reproducibility of the R
  package library is the main concern.

  Commit `renv.lock`. Note, however, that `renv` records the required R version
  but does not install R itself or manage operating-system libraries. If the
  project also depends on a particular R interpreter, compiled libraries, or
  external command-line tools, use `pixi` to manage those parts of the
  environment as well.

- **Command-line tools that are part of the analysis or pipeline**: put them in
  the project's `uv` or `pixi` environment rather than installing them
  globally.

  For example, if a pipeline depends on a particular version of `samtools`,
  declare `samtools` in `pixi.toml` so its exact resolved build is recorded in
  `pixi.lock`. Likewise, Python development tools such as `ruff` or `pytest`
  can be project dependencies in `uv`.

- **Personal command-line utilities that do not affect the project's results**:
  use `uv tool` / `pipx` for PyPI applications, or `pixi global` /
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

The key rule is: **if the result of the research depends on a program, make that
program a project dependency and put it in the lockfile.**

## Other tools worth mentioning

- `mamba` and `micromamba`: fast C++ drop-in replacements for `conda`. If you
  want to keep the Conda workflow but drop the slow solver, these are the
  smallest change. `pixi` largely supersedes them for new projects.
- `poetry`, `pdm`, `hatch`: earlier-generation Python project managers. Still
  common, but `uv` is faster and covers most of what they do.
- `rye`: an experimental Python manager that has been folded into `uv`, so new
  users should start with `uv` directly.
- `spack` and `easybuild`: build-from-source managers used on HPC clusters,
  where compiler and hardware tuning matter.
- `Nix` and `GNU Guix`: whole-system reproducible package managers. Very
  powerful and very reproducible, with a steeper learning curve.
- Containers (`Docker`, `Apptainer`/`Singularity`): the strongest
  reproducibility, capturing the whole operating system. They complement rather
  than replace the tools above; you often use `pixi`, `uv`, or `micromamba` inside a container.

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

1. Open this repository (`BiCU-CCRI/20260901_Package_managers_demo`) in a GitHub
   Codespace: use the badge above, or the green Code button, then Codespaces,
   then `create a codespace on this branch`. The first build takes a few minutes
   because it installs R and renv.
2. When the container is ready, open the exercises in order. Each is a short
   Markdown file with copy-paste commands and the output you should expect.

## Supporting material

- `CHEATSHEET.md`: a one-page command reference and list of common gotchas and
  fixes. Keep it open while teaching.
- `reproducibility.md`: tips and tricks focused on reproducibility and
  portability (lockfiles, cross-platform locks, CI, HPC, containers).
- `limitations.md`: where each manager struggles or fails, with the symptom and
  a workaround. Worth reading before you commit to a tool for a project.
- `environment.yml`: a small sample Conda environment used by the `pixi import`
  step in exercise 2.
