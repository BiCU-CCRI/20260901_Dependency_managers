# Exercise 4: condax and pixi global, isolated Conda command-line tools

`pipx` only reaches PyPI. Many bioinformatics tools (`samtools`, `bwa`,
`multiqc`, and thousands more) live on `bioconda`, not PyPI. `condax` was the
original "pipx for Conda": install a Conda tool globally, isolated in its own
environment, with just the command on your `PATH`.

In 2026, `condax` is only lightly maintained. The actively developed equivalent
is `pixi global`, and there is also `conda-global` from conda-incubator. This
exercise shows `pixi global` first (recommended) and `condax` second (so you
recognise it if a colleague uses it).

Rough time: 5 minutes.

## Part A: pixi global (recommended)

You already installed `pixi` in exercise 2, so this needs no extra setup.

```bash
pixi global install samtools
samtools --version | head -n 1
```

`pixi global install` creates a dedicated isolated environment for the tool and
links its command onto your `PATH`. Because it is a full Conda environment,
compiled dependencies are isolated too, not just Python ones.

Inspect and manage global tools:

```bash
pixi global list
pixi global uninstall samtools
```

Why this replaces the fragile Conda base environment: each tool is separate, so
there is no shared base to corrupt, and you never need root to add a tool.

## Part B: condax (the classic tool)

`condax` is a Python package that drives Conda underneath. Install it with pipx
so it stays isolated:

```bash
pipx install condax
condax --version
```

Install a tool:

```bash
condax install tabulate
# The 'tabulate' command is now on your PATH, in its own Conda environment.
condax list
```

Remove it:

```bash
condax remove tabulate
```

Warning and fix: because `condax` is lightly maintained, on a fresh system it
can fail to find `conda`, or hit solver errors with newer Conda versions. If a
`condax` command errors out, the fix is not to debug it, use `pixi global`
instead, which does the same job and is actively maintained. That is exactly why
Part A comes first.

## Takeaways

- For Conda or bioinformatics CLI tools you want system-wide, use
  `pixi global install`.
- `condax` pioneered this pattern and you may still see it in existing setups,
  but prefer `pixi global` (or `conda-global`) for new work.
- Mental model:
  - PyPI CLI tool, isolated and global: `pipx` or `uv tool install`.
  - Conda CLI tool, isolated and global: `pixi global` (modern) or `condax`
    (classic).
