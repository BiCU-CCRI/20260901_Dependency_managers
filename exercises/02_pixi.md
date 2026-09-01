# Exercise 2: pixi, project-first Conda environments

`pixi` uses the Conda package ecosystem (`conda-forge`, `bioconda`, and other
channels), so it can install compiled bioinformatics tools and system libraries,
not just Python packages. Its normal project workflow maintains a lockfile and
does not depend on a mutable `base` environment. In this exercise you will build
a project that mixes Conda and PyPI dependencies, add a second target platform,
and reproduce the environment from the lockfile.

Rough time: 8 minutes.

## 1. Install pixi

```bash
curl -fsSL https://pixi.sh/install.sh | sh
exec bash            # reload the shell so pixi is on PATH
pixi --version
```

Why `exec bash`: the installer adds `~/.pixi/bin` to your shell config. Starting
a fresh shell picks it up. Expected: a version string like `pixi 0.x.y`.

> [!NOTE]
> For a long-lived CI or archival workflow, pin the Pixi version rather than
> installing whatever happens to be latest. The floating installer is used here
> to keep the live exercise short.

## 2. Create a project and set the channels

```bash
cd /tmp
pixi init demo-pixi -c conda-forge -c bioconda
cd demo-pixi
cat pixi.toml
```

`pixi init` writes a `pixi.toml` manifest. The `-c` flags set the channels to
search: `conda-forge` for general packages and `bioconda` for bioinformatics
tools.

## 3. Add Conda and PyPI dependencies

First add packages from the Conda ecosystem:

```bash
pixi add samtools "python>=3.11" pysam
```

This installs `samtools` (a compiled program), Python, and the Conda build of
`pysam` into one project environment.

Now deliberately add a package from PyPI:

```bash
pixi add --pypi httpx
```

Pixi resolves the Conda side and coordinates the PyPI resolution with it using
uv's resolver library. Both sources are recorded in the same `pixi.lock`.

Inspect the result:

```bash
pixi list
pixi run samtools --version | head -n 2
pixi run python -c "import pysam, httpx; print(pysam.__version__, httpx.__version__)"
```

`pixi run` executes a command inside the project environment without a separate
activation step, similar to `uv run`.

## 4. Add another target platform

A major portability feature is that one `pixi.lock` can contain resolutions for
multiple declared platforms. The Codespace is Linux; add Intel macOS as a
second target:

```bash
pixi workspace platform list
pixi workspace platform add osx-64
pixi workspace platform list
```

Pixi resolves both declared platforms and stores them in the same lockfile. This
does **not** make an unavailable package magically portable: every dependency
still needs a compatible build for each declared platform. This is especially
important for Bioconda on Apple Silicon, where some packages lack native
`osx-arm64` builds.

## 5. Define a reusable task

Edit `pixi.toml` and add a task under a `[tasks]` section:

```bash
cat >> pixi.toml <<'EOF'

[tasks]
versions = "samtools --version | head -n 1 && python -c 'import pysam, httpx; print(pysam.__version__, httpx.__version__)'"
EOF

pixi run versions
```

Tasks are named commands stored in the manifest, so a collaborator can run
`pixi run versions` without needing to know the exact command. This is a small
built-in alternative to a Makefile for simple workflows.

## 6. Open an interactive shell (optional)

```bash
pixi shell
which samtools      # points inside .pixi/envs/default
exit
```

`pixi shell` is the closest analogue to `conda activate`. Use it for interactive
work; use `pixi run` for scripts and pipelines.

## 7. Reproduce from the lockfile

```bash
rm -rf .pixi
pixi install --locked
pixi run --locked samtools --version | head -n 1
```

`--locked` makes the command fail if the manifest and `pixi.lock` disagree,
rather than silently re-resolving. The lock contains the exact resolved Conda
builds and PyPI dependencies for the declared target platforms. Installed
artifacts can still differ by platform; reproducibility does not mean every
machine receives identical bytes.

## 8. Coming from Conda: import an existing environment

If you already have an `environment.yml`, you do not have to start over. This
repository ships a small one under `exercises/`, so this step is runnable:

```bash
cd "$CODESPACE_VSCODE_FOLDER"        # the tutorial repo root in Codespaces
pixi init /tmp/demo-import --import exercises/environment.yml
cd /tmp/demo-import
cat pixi.toml                        # channels and deps came from environment.yml
pixi run samtools --version | head -n 1
```

`pixi init --import` converts the Conda spec into a Pixi manifest and resolves a
`pixi.lock` when the environment is first used. An `environment.yml` with ranges
records requested constraints; the generated lock records the concrete
resolution.

Note: if you run this container outside Codespaces, `$CODESPACE_VSCODE_FOLDER`
will not be set. Run the import command from whatever directory contains your
`environment.yml` instead.

## Takeaways

- Uses the Conda package ecosystem while providing a project-first workflow.
- Can coordinate Conda and PyPI dependencies in one project lockfile.
- One lockfile can contain separate resolutions for multiple declared target
  platforms.
- `pixi run` for scripts, `pixi shell` for interactive work.
- This is a strong default for bioinformatics projects with native tools or
  mixed-language dependencies.

Commit `pixi.toml` and `pixi.lock` to git. Do not commit the `.pixi` directory.
