# Cheat-sheet

## Install one-liners

| Tool | Command | Note |
| --- | --- | --- |
| uv | `curl -LsSf https://astral.sh/uv/install.sh \| sh` then `source "$HOME/.local/bin/env"` | installs to `~/.local/bin` |
| pixi | `curl -fsSL https://pixi.sh/install.sh \| sh` then `exec bash` | installs to `~/.pixi/bin` |
| pipx | usually preinstalled; else `python3 -m pip install --user pipx && pipx ensurepath` | reload shell after ensurepath |
| condax | `pipx install condax` | prefer `pixi global` for new work |
| renv | preinstalled by the devcontainer | R itself is preinstalled too |

## Core workflow, side by side

| Task | uv | pixi | renv (in R) |
| --- | --- | --- | --- |
| Start a project | `uv init` | `pixi init -c conda-forge -c bioconda` | `renv::init()` |
| Add a package | `uv add polars` | `pixi add samtools` | `renv::install("glue")` |
| Run a command | `uv run python x.py` | `pixi run python x.py` | (auto-activated in project) |
| Interactive shell | `source .venv/bin/activate` | `pixi shell` | project `.Rprofile` |
| Write the lock | automatic on `add` | automatic on `add` | `renv::snapshot()` |
| Rebuild from lock | `uv sync` | `pixi install` | `renv::restore()` |
| Check for drift | `uv lock --check` | `pixi install --locked` | `renv::status()` |

## Conda to pixi translation

| Conda | pixi |
| --- | --- |
| `conda create -n env python=3.11` | `pixi init` then `pixi add python=3.11` |
| `conda install samtools` | `pixi add samtools` |
| `conda activate env` | `pixi shell` |
| `conda run -n env cmd` | `pixi run cmd` |
| `conda env export` | `pixi.lock` (written automatically) |
| `conda env create -f environment.yml` | `pixi init --import environment.yml` |
| global CLI tool | `pixi global install tool` |

## Application managers, which to reach for

| You want | Use |
| --- | --- |
| A PyPI CLI tool, system-wide | `pipx install tool` or `uv tool install tool` |
| A PyPI CLI tool, once | `pipx run tool` or `uvx tool` |
| A Conda or bioconda CLI tool, system-wide | `pixi global install tool` (or `condax install tool`) |

## Gotchas and fixes

For deeper failure modes and workarounds, see `limitations.md`.

- Command not found right after install: the installer edited your shell config
  but this shell has not reloaded. Fix: `exec bash` (or `source "$HOME/.local/bin/env"`
  for uv), or open a new terminal.
- pipx says a command is not on PATH: run `pipx ensurepath`, then reload the
  shell.
- condax errors on a fresh machine (cannot find conda, or solver failure): do
  not debug it, use `pixi global install` instead. It does the same job and is
  actively maintained.
- Never run `pixi global install pip`: it creates an isolated env with its own
  unreachable pip. Use `pixi add pip` inside a project instead.
- Do not `pip install` into a pixi environment by hand. Use `pixi add` for conda
  packages and the `[pypi-dependencies]` section (or `pixi add --pypi name`) for
  PyPI ones, so both stay in the lock.
- renv `init()` seems to hang or asks questions in a script: use
  `renv::init(bare = TRUE)` for a clean, non-interactive start, and
  `renv::restore(prompt = FALSE)` in scripts and CI.
- First Codespace build is slow: it is installing R and renv. This is one-time
  per Codespace. Pre-build the image before the session if you can.

## What to commit, what to ignore

| Tool | Commit | Ignore |
| --- | --- | --- |
| uv | `pyproject.toml`, `uv.lock`, `.python-version` | `.venv/` |
| pixi | `pixi.toml`, `pixi.lock` | `.pixi/` |
| renv | `renv.lock`, `.Rprofile`, `renv/activate.R`, `renv/settings.json` | `renv/library/` (auto-ignored) |

## Timing target (about 35 minutes)

Overview 5, uv 8, pixi 8, pipx 4, condax 5, renv 6, questions in whatever
remains. If you are short on time, drop the condax Part B (classic tool) and the
renv Bioconductor reference block.
