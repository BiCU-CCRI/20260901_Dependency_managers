# Exercise 2: pixi, Conda done the modern way

`pixi` uses the same Conda package ecosystem you already know (`conda-forge` and
`bioconda`), so it can install compiled bioinformatics tools, not just Python.
It is much faster than Conda, has no fragile base environment, and always writes
a lockfile. In this exercise you will build a project that mixes a compiled
tool (`samtools`) with a Python package, then reproduce it from the lockfile.

Rough time: 8 minutes.

## 1. Install pixi

```bash
curl -fsSL https://pixi.sh/install.sh | sh
exec bash            # reload the shell so pixi is on PATH
pixi --version
```

Why `exec bash`: the installer adds `~/.pixi/bin` to your shell config. Starting
a fresh shell picks it up. Expected: a version string like `pixi 0.x.y`.

## 2. Create a project and set the channels

```bash
cd /tmp
pixi init demo-pixi -c conda-forge -c bioconda
cd demo-pixi
cat pixi.toml
```

`pixi init` writes a `pixi.toml` manifest. The `-c` flags set the channels to
search, `conda-forge` for general packages and `bioconda` for bioinformatics
tools. Channel order matters: `conda-forge` first is the community convention.

## 3. Add a compiled tool and a Python package

```bash
pixi add samtools "python>=3.11" pysam
```

Note this installs `samtools` (a compiled C program), a specific Python, and
`pysam` (a Python library) into one project environment, resolved together.
`conda` can do this too, but slowly; `pixi` resolves in parallel in Rust and
writes `pixi.lock` at the same time.

Confirm the tool is really there:

```bash
pixi run samtools --version | head -n 2
```

`pixi run` executes a command inside the project environment without a separate
activate step, the same idea as `uv run`. Expected: the samtools version banner.

## 4. Define a reusable task

Edit `pixi.toml` and add a task under a `[tasks]` section:

```bash
cat >> pixi.toml <<'EOF'

[tasks]
versions = "samtools --version | head -n 1 && python -c 'import pysam; print(pysam.__version__)'"
EOF

pixi run versions
```

Tasks are named commands stored in the manifest, so a collaborator runs
`pixi run versions` without needing to know the exact command. This is a small
built-in alternative to a Makefile.

## 5. Open an interactive shell (optional)

```bash
pixi shell
which samtools      # points inside .pixi/envs/default
exit
```

`pixi shell` is the closest thing to `conda activate`. Use it for interactive
work; use `pixi run` for scripts and pipelines.

## 6. Reproduce from the lockfile

```bash
rm -rf .pixi
pixi install
pixi run samtools --version | head -n 1
```

`pixi install` rebuilds the environment to match `pixi.lock` exactly. The lock
covers both the Conda and the PyPI side, so the rebuild is bit-for-bit the same
set of versions.

## 7. Coming from Conda: import an existing environment

If you already have an `environment.yml`, you do not have to start over. This
repository ships a small one at its root, so this step is runnable:

```bash
cd "$CODESPACE_VSCODE_FOLDER"        # the tutorial repo root in Codespaces
pixi init /tmp/demo-import --import environment.yml
cd /tmp/demo-import
cat pixi.toml                        # channels and deps came from environment.yml
pixi run samtools --version | head -n 1
```

`pixi init --import` converts the Conda spec into a `pixi.toml` and, on the first
`pixi run`, resolves it and writes a `pixi.lock`. Your `environment.yml` gave
version ranges; pixi turns them into an exact, reproducible lock.

Note: if you run this container outside Codespaces, `$CODESPACE_VSCODE_FOLDER`
will not be set. Just run the `pixi init --import` command from whatever
directory holds your `environment.yml`.

## Takeaways

- Same packages as Conda (`conda-forge`, `bioconda`), far faster, with a
  lockfile every time.
- Handles compiled tools, Python, and R together in one environment.
- `pixi run` for scripts, `pixi shell` for interactive work.
- This is usually the right default for bioinformatics projects.

Commit `pixi.toml` and `pixi.lock` to git. Do not commit the `.pixi` directory.
