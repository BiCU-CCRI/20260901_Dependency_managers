# Instructor cheat-sheet

One page to run the session from. Keep it open in a split pane while you demo.

## Install one-liners

| Tool | Command | Note |
| --- | --- | --- |
| uv | `curl -LsSf https://astral.sh/uv/install.sh \| sh` then `source "$HOME/.local/bin/env"` | installs to `~/.local/bin` |
| pixi | `curl -fsSL https://pixi.sh/install.sh \| sh` then `exec bash` | installs to `~/.pixi/bin` |
| pipx | usually preinstalled; else `python3 -m pip install --user pipx && pipx ensurepath` | reload shell after ensurepath |
| condax | `pipx install condax` | prefer `pixi global` or `conda-global` for new work |
| renv | preinstalled by the devcontainer | R itself is preinstalled too |

## Core workflow, side by side

| Task | uv | pixi | renv (in R) |
| --- | --- | --- | --- |
| Start a project | `uv init` | `pixi init -c conda-forge -c bioconda` | `renv::init()` |
| Add a package | `uv add polars` | `pixi add samtools` | `renv::install("glue")` |
| Add PyPI package | `uv add httpx` | `pixi add --pypi httpx` | n/a |
| Run a command | `uv run python x.py` | `pixi run python x.py` | `Rscript x.R` |
| Interactive shell | optional `.venv` activation | `pixi shell` | start R in project (`.Rprofile` auto-activates renv) |
| Write/update lock | automatic on changes | automatic on changes | `renv::snapshot()` |
| Strict rebuild from lock | `uv sync --locked` | `pixi install --locked` | `renv::restore()` |
| Check for drift | `uv lock --check` | `pixi lock --check` | `renv::status()` |

## Portability

| Tool | What to show |
| --- | --- |
| uv | universal project lock can encode supported platform/Python conditions; exact artifacts still vary by platform |
| pixi | `pixi workspace platform add osx-64` adds another pre-resolved target to the same `pixi.lock` |
| renv | package/source versions transfer, but R itself and native system libraries are outside renv |
| conda | modern Conda also supports exact multi-platform lock workflows; historical `environment.yml` workflows are not the same thing |

## Conda to pixi translation

| Conda | pixi |
| --- | --- |
| `conda create -n env python=3.11` | `pixi init` then `pixi add python=3.11` |
| `conda install samtools` | `pixi add samtools` |
| `pip install httpx` inside Conda env | `pixi add --pypi httpx` so it stays represented by the project |
| `conda activate env` | `pixi shell` |
| `conda run -n env cmd` | `pixi run cmd` |
| `conda env export` | project manifest + `pixi.lock` |
| `conda env create -f environment.yml` | `pixi init --import environment.yml` |
| global CLI tool | `pixi global install tool` |

## Application managers selection

| You want | Use |
| --- | --- |
| Python CLI app, user-global | `uv tool install tool` or `pipx install tool` |
| Python CLI app, one-off | `uvx tool` or `pipx run tool` |
| Conda/Bioconda CLI app, user-global | `pixi global install tool` or `conda global install tool` with conda-global |
| Research-critical CLI app | put it in the project's uv/Pixi dependencies instead |

`condax` is the older Conda application-manager implementation and is still
worth recognizing, but it is not the preferred new setup.

## Gotchas and fixes

For deeper failure modes and workarounds, see [`../limitations.md`](../limitations.md).

- Command not found right after install: the installer edited your shell config
  but this shell has not reloaded. Fix: `exec bash` (or `source "$HOME/.local/bin/env"`
  for uv), or open a new terminal.
- pipx says a command is not on PATH: run `pipx ensurepath`, then reload the
  shell.
- Do not `pip install` into a pixi environment by hand. Use `pixi add` for conda
  packages and the `[pypi-dependencies]` section (or `pixi add --pypi name`) for
  PyPI ones, so both stay in the lock.
- condax errors on a fresh machine (cannot find conda, or solver failure): do
  not debug it, use `pixi global install` instead. It does the same job and is
  actively maintained.
- Never run `pixi global install pip`: it creates an isolated env with its own
  unreachable pip. Use `pixi add pip` inside a project instead.
- `uvx` / `uv tool run` is intentionally outside the current project. If a tool
  needs the project's imports or affects research results, add it to the project
  and use `uv run`.
- A global-tool manifest is not the same as a project lockfile.
- Multi-platform locking cannot invent packages for unsupported platforms.
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
| renv | `renv.lock`, `.Rprofile`, `renv/activate.R`, `renv/settings.json` | `renv/library/` |

## Reproducibility ladder

1. Project manifest.
2. Project lockfile.
3. Pin the interpreter/runtime where needed.
4. Declare/test supported platforms.
5. Use strict locked restores in CI.
6. Add a pinned container image digest when system-level portability matters.
7. Preserve package/container artifacts for long-term archival recovery.

## Timing target (about 35 minutes)

Overview 5, uv 8, pixi 8, pipx 4, Conda application managers 5, renv 6, and
questions in whatever remains. If short on time, keep `condax`/`conda-global` as
reference material rather than running them live.
