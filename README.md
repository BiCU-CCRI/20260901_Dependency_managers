# Modern dependency managers, a hands-on tour

[![Open in GitHub Codespaces](https://github.com/codespaces/badge.svg)](https://codespaces.new/BiCU-CCRI/20260901_Package_managers_demo)

A short (about 30 to 40 minute) tutorial that introduces modern dependency and
environment managers with a focus on **reproducibility and portability**. It
runs entirely in a GitHub Codespace, so there is nothing to install on your own
machine.

Tools covered, with hands-on exercises:

- `uv` (Python projects and environments)
- `pixi` (Conda-ecosystem projects, especially useful for bioinformatics)
- `pipx` (isolated Python command-line applications)
- `pixi global`, `condax`, and `conda-global` (isolated Conda applications)
- `renv` (reproducible R package libraries)

Other tools worth knowing are listed near the end of this file.

## Why bother, if Conda and virtualenv already work?

They do work. The difference is mostly about **workflow defaults**, not that
older tools are incapable of reproducibility.

- Older Conda versions used the classic solver, which could be very slow for
  large environments. Since Conda 23.10, `libmamba` has been the default solver,
  so solver speed is no longer the strongest reason to move away from Conda.
  Conda still centers on mutable named environments and maintains a special
  `base` environment that users can accidentally pollute.
- `pip` plus `venv` or `virtualenv` does not automatically maintain a project
  lockfile during normal installs. A hand-written `requirements.txt` often
  records direct requirements or version ranges; `pip freeze` snapshots an
  already-installed environment. Recent pip versions also have an experimental
  `pip lock` command, but locking is not the default project workflow in the
  same way it is with uv.
- Mixing `conda install` and `pip install` in the same environment can create
  difficult-to-debug state, especially if Conda is used again after pip has
  changed the environment.
- Modern Conda can export exact, multi-platform lockfiles as well. The practical
  distinction is therefore increasingly about whether the tool makes
  project-scoped manifests, locks, and locked restores the normal workflow.

Tools such as Pixi and uv are implemented in Rust and are designed around fast
resolution and automatically maintained project lockfiles. For research code,
that lockfile is particularly valuable because it records the resolved
transitive dependency graph, not only what you requested.

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
> project commands and `uv tool`, while Pixi supports both through projects and
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
  to remember. Works with `pyproject.toml` and can import requirements files.
- **Cons**: focused on Python and the Python packaging ecosystem. It cannot
  manage arbitrary non-Python tools such as `samtools` or `bwa`. uv is still
  versioned `0.x`, although Astral considers it stable software; minor releases
  may intentionally contain breaking changes, so pin uv itself in CI when that
  matters.

### `pixi`

- **Pros**: built on the Conda ecosystem, so it installs from `conda-forge`,
  `bioconda`, and other channels, including compiled tools, system libraries,
  Python, R, and other languages. Normal project operations maintain
  `pixi.lock`, and no special mutable `base` environment is required. PyPI
  dependencies are resolved with uv's resolver library and coordinated with the
  Conda environment. One lockfile can hold resolutions for multiple declared
  platforms. Existing `environment.yml` files can be imported.
- **Cons**: introduces a Pixi project model and manifest (`pixi.toml`, although
  configuration can also live in `pyproject.toml`). The project is still
  evolving. Multi-platform locking does not make packages available on platforms
  where no compatible build exists.

### `renv`

- **Pros**: widely used for reproducible R project libraries. Gives each project
  its own library, discovers R package dependencies, records exact package
  versions and sources in `renv.lock`, and supports CRAN, Bioconductor, GitHub,
  and other repositories.
- **Cons**: R-specific. It records the R version used by the project but does
  not install/manage R itself or arbitrary operating-system libraries. Restoring
  source packages can be slow when binaries are unavailable.

### `pipx`

- **Pros**: clean isolation for Python CLI applications. Each application gets
  its own virtual environment, so Python dependencies do not collide. `pipx run`
  can run an application without keeping a normal installed entry.
- **Cons**: focused on Python applications and does not manage arbitrary system
  dependencies. Ordinary `pipx install` is about isolation/convenience rather
  than project locking. Current pipx also has explicit manifest/PEP 751 lock
  workflows, but these are separate from a research project's own lockfile.

### `uv tool`

- **Pros**: the pipx-style application manager built into uv. Each installed
  Python CLI application gets its own isolated environment, while executables
  are exposed on `PATH`. `uv tool run` / `uvx` can run a tool in an isolated
  temporary/cached tool environment.
- **Cons**: Python applications only. Tool environments are outside the current
  project's `uv.lock`; if an exact tool version affects research results, make
  it a project dependency and use `uv run` instead.

### `pixi global`

- **Pros**: the Conda application-manager mode built into Pixi. It can install
  compiled and non-Python tools from `conda-forge`, `bioconda`, and other Conda
  channels into isolated environments and expose their commands on `PATH`.
  `pixi-global.toml` records the configured global environments.
- **Cons**: the global manifest is not a project `pixi.lock`, and each global
  environment is resolved for one platform. Research-critical tools should be
  project dependencies instead.

### `condax`

- **Pros**: the older "pipx for Conda" pattern: each Conda-packaged application
  is installed into its own isolated environment and exposed on `PATH`.
- **Cons**: development has slowed considerably. For new setups, prefer
  `pixi global` or the newer `conda-global` plugin.

### `conda-global`

- **Pros**: a newer Conda-native application manager from `conda-incubator`.
  Each tool lives in an isolated Conda environment. Commands such as
  `conda global install` expose small native trampoline launchers that forward
  execution into the right environment without manual activation.
  `~/.conda/global.toml` records configured tools.
- **Cons**: newer and less mature than pipx or Pixi's global tooling. It requires
  Conda. Its manifest is not a full project dependency lockfile.

## Which one should I use?

For research work where **reproducibility and portability are the priority**:

- **Pure Python project** — a package, script, or analysis whose dependencies
  all live in the Python packaging ecosystem: use `uv`.

  Commit `pyproject.toml`, `uv.lock`, and an appropriately specific Python
  version request. Use `uv sync --locked` in CI.

- **Project that needs compiled tools, system libraries, mixed languages, or
  Bioconda packages** — common in bioinformatics: use `pixi`.

  Commit `pixi.toml` and `pixi.lock`, and explicitly declare every platform you
  intend to support. Use `pixi install --locked` in CI.

- **R-only project or analysis**: use `renv` when reproducibility of the R
  package library is the main concern.

  Commit `renv.lock`. If the project also depends on an exact R interpreter,
  system libraries, or external CLI tools, use Pixi or a container for that
  outer layer as well.

- **Command-line tools that are part of the analysis or pipeline**: put them in
  the project's uv or Pixi environment rather than relying on a user-global
  installation.

- **Personal command-line utilities that do not affect results**: use `uv tool`
  / `pipx` for Python applications, or `pixi global` / `conda-global` for Conda
  applications.

The key rule is:

> **If the result of the research depends on a program, make that program a
> project dependency and put it in the project lockfile.**

## Other tools worth mentioning

- `mamba`: a Conda-compatible CLI using the libmamba ecosystem. Modern Conda
  itself already defaults to libmamba solving, so solver speed alone is no
  longer the reason to choose Mamba.
- `micromamba`: a small standalone Conda-compatible executable with no populated
  base environment, useful in containers and lightweight setups.
- `poetry`, `pdm`, `hatch`: established Python project managers. uv overlaps
  with much of their dependency/environment workflow.
- `rye`: an experimental Python manager whose work was folded into uv; new users
  should generally start with uv.
- `spack` and `easybuild`: build-from-source managers used on HPC clusters where
  compiler and hardware tuning matter.
- `Nix` and `GNU Guix`: whole-system package managers with stronger build-level
  reproducibility and a steeper learning curve.
- Containers (`Docker`, `Apptainer`/`Singularity`): an outer reproducibility
  layer that captures much more of userspace than a package lockfile. Containers
  still share the host kernel and can depend on hardware/drivers. For archival
  work, pin the base image by digest and preserve the built image as well as the
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
- `exercises/environment.yml`: a small sample Conda environment used by the Pixi
  import step in exercise 2.
