# Modern dependency managers, a hands-on tour

[![Open in GitHub Codespaces](https://github.com/codespaces/badge.svg)](https://codespaces.new/BiCU-CCRI/20260901_Package_managers_demo)

A short (about 30 to 40 minute) tutorial that introduces "new" dependency and
environment managers to researchers who currently use Conda or virtualenv. It
runs entirely in a GitHub Codespace, so there is nothing to install on your own
machine.

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

- Conda's classic solver can be very slow, and large environments can take
  minutes to resolve. It also keeps a "base" environment that is easy to
  pollute and hard to reset.
- `pip` plus `virtualenv` gives you no lockfile by default, so "it works on my
  machine" is common. `requirements.txt` records what you asked for, not the
  exact resolved versions.
- Mixing `conda install` and `pip install` in the same environment is a
  frequent source of hard-to-debug conflicts.
- None of these record the exact resolved versions of every transitive
  dependency automatically, which is what reproducibility actually needs.

The newer tools are mostly written in Rust, resolve dependencies in parallel,
and write a lockfile on every change. For research code, the lockfile is the
important part: it is what lets a colleague, a reviewer, or you-in-two-years
rebuild the same environment.

## The landscape at a glance

Think of the tools in two groups: project or environment managers (you get a
reproducible environment per project) and application managers (you install a
command-line tool globally, each in its own isolated sandbox).

| Tool | Role | Ecosystem | Written in | Lockfile | Replaces |
| --- | --- | --- | --- | --- | --- |
| `uv` | Project and env manager | PyPI (Python only) | Rust | `uv.lock` | pip, virtualenv, pyenv, pip-tools, poetry, pipx |
| `pixi` | Project and env manager | Conda plus PyPI | Rust | `pixi.lock` | conda, mamba, and can call uv internally |
| `renv` | Project and env manager | CRAN, Bioconductor, GitHub | R | `renv.lock` | manual R library management |
| `pipx` | Application manager | PyPI (Python only) | Python | none | `pip install` for CLI tools |
| `condax` | Application manager | Conda | Python | none | `conda install` for CLI tools |
| `pixi global` | Application manager | Conda | Rust | manifest | condax, and pipx for Conda tools |

Conda and virtualenv are shown below only as the baseline you already know.

| Baseline tool | Role | Ecosystem | Lockfile |
| --- | --- | --- | --- |
| `conda` / `mamba` | Project and env manager | Conda plus PyPI | no (env export only) |
| `virtualenv` / `venv` + `pip` | Project and env manager | PyPI | no (requirements.txt only) |

## Pros and cons

### uv

- Pros: extremely fast, one tool for Python versions, virtual environments,
  dependencies, and CLI tools. Writes a real lockfile. `uv run` activates the
  environment for you, so there is no `activate` step to forget. Reads existing
  `requirements.txt` and `pyproject.toml`.
- Cons: Python and PyPI only. It cannot install non-Python binaries such as
  `samtools` or `bwa`. Still pre-1.0, so occasional breaking changes.

### pixi

- Pros: built on the Conda ecosystem, so it installs from `conda-forge` and
  `bioconda`, including compiled tools and R packages, not just Python. Much
  faster than Conda, always writes a lockfile, has no fragile base environment,
  and uses `uv` internally to resolve PyPI packages cleanly alongside Conda
  ones. Can import an existing `environment.yml`.
- Cons: another manifest format to learn (`pixi.toml`). Still maturing, though
  the file formats are kept backward compatible.

### renv

- Pros: the standard for reproducible R. Per-project library, automatic
  dependency discovery, a lockfile, and installs from CRAN, Bioconductor, and
  GitHub. Fits existing R and RStudio workflows with three functions.
- Cons: R only, and it manages R packages, not the R interpreter itself or
  system libraries. Restoring source packages can be slow without a binary
  package repository configured.

### pipx

- Pros: the clean way to install Python CLI tools. Each tool gets its own
  virtual environment, so nothing collides with your project libraries or with
  each other. `pipx run` runs a tool once without installing it.
- Cons: Python only, and it isolates Python dependencies but not compiled
  system dependencies. No lockfile.

### condax

- Pros: the same idea as pipx but for Conda packages, so it works for tools that
  are not on PyPI (many bioinformatics tools). Full Conda environments, so
  compiled dependencies are isolated too.
- Cons: lightly maintained. In 2026 the actively developed equivalents are
  `pixi global` and the `conda-global` project (from conda-incubator). If you
  are starting fresh, prefer `pixi global`. The exercise shows both.

## Which one should I use

A rough decision guide for research work:

- Pure Python project (a package, a script, an analysis with only PyPI
  dependencies): use `uv`.
- Project that needs compiled tools, mixed languages, or `bioconda` packages
  (most bioinformatics pipelines): use `pixi`.
- R project or analysis: use `renv`.
- A command-line tool you want available everywhere, that lives on PyPI
  (for example `ruff`, `cookiecutter`, `httpie`): use `pipx`.
- A command-line tool you want available everywhere, that lives on `conda-forge`
  or `bioconda` (for example `samtools`, `multiqc`): use `pixi global`.

You do not have to pick just one. A common setup is `pixi` per project, `uv` for
pure-Python work, `renv` for R, and `pixi global` for your everyday CLI tools.

## Running this tutorial

1. Open this repository (`BiCU-CCRI/20260901_Package_managers_demo`) in a GitHub
   Codespace: use the badge above, or the green Code button, then Codespaces,
   then create a codespace on this branch. The first build takes a few minutes
   because it installs R and renv.
2. When the container is ready, open the exercises in order. Each is a short
   Markdown file with copy-paste commands and the output you should expect.

Recommended order and rough timing:

| Step | File | Time |
| --- | --- | --- |
| 1 | `exercises/01_uv.md` | 8 min |
| 2 | `exercises/02_pixi.md` | 8 min |
| 3 | `exercises/03_pipx.md` | 4 min |
| 4 | `exercises/04_condax.md` | 5 min |
| 5 | `exercises/05_renv.md` | 6 min |

The remaining time is for the overview above and questions.

Supporting material in this repository:

- `CHEATSHEET.md`: a one-page command reference and list of common gotchas and
  fixes. Keep it open while teaching.
- `reproducibility.md`: tips and tricks focused on reproducibility and
  transferability (lockfiles, cross-platform locks, CI, HPC, containers).
- `limitations.md`: where each manager struggles or fails, with the symptom and
  a workaround. Worth reading before you commit to a tool for a project.
- `environment.yml`: a small sample Conda environment used by the pixi import
  step in exercise 2.

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
  than replace the tools above; you often use `pixi` or `uv` inside a container.

## References

- uv: <https://docs.astral.sh/uv/>
- pixi: <https://pixi.sh/latest/>
- pipx: <https://pipx.pypa.io/>
- condax: <https://mariusvniekerk.github.io/condax/> and conda-global:
  <https://conda-incubator.github.io/conda-global/>
- renv: <https://rstudio.github.io/renv/>
