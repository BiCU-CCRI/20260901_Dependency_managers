# Exercise 4: isolated Conda command-line tools

`pipx` and `uv tool` target Python applications. Many bioinformatics tools
(`samtools`, `bwa`, and thousands more) are distributed through Conda/Bioconda,
so a Conda-aware application manager is useful when you want a command on your
`PATH` without putting it in a shared `base` environment.

This exercise shows `pixi global` first. It also introduces `condax`, the older
"pipx for Conda" implementation, and `conda-global`, a newer Conda-native plugin.

Rough time: 5 minutes.

## Part A: pixi global

You already installed `pixi` in exercise 2, so this needs no extra setup.

```bash
pixi global install samtools
samtools --version | head -n 1
```

`pixi global install` creates an isolated Conda environment for the application
and exposes its executable on your user `PATH`. Compiled Conda dependencies are
isolated along with the application, not just Python packages.

Inspect and manage global tools:

```bash
pixi global list
pixi global uninstall samtools
```

Pixi maintains a global manifest describing these environments, but that
manifest is not the same as a project `pixi.lock`. If the exact `samtools`
version affects an analysis, put it in the project's `pixi.toml` instead.

## Part B: condax (the older tool)

`condax` is a Python package that drives Conda underneath. Install it with pipx
so condax itself stays isolated:

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

`condax` development has slowed and it can be more sensitive to the surrounding
Conda setup. Prefer `pixi global` or `conda-global` for a new installation; the
example remains useful because existing systems may still use condax.

## Part C: conda-global (reference)

`conda-global` is a newer `conda-incubator` plugin that brings the same pattern
directly into the Conda CLI. After installing the plugin according to its
project documentation, the mental model is:

```bash
conda global install <tool>
```

Each application gets an isolated Conda environment. Small native trampoline
launchers expose the commands on `PATH` and forward execution into the correct
environment without requiring `conda activate` first. Its
`~/.conda/global.toml` is a manifest of configured tools, not a project lockfile.

We keep this part as reference rather than installing another manager during the
short live session.

## Takeaways

- For a Conda/Bioconda CLI application you want available user-globally,
  `pixi global` is a simple default if you already use Pixi.
- `conda-global` offers the same application-manager model as a Conda-native
  plugin; `condax` is the older implementation you may encounter in existing
  setups.
- PyPI CLI application, isolated and user-global: `pipx` or `uv tool install`.
- Conda CLI application, isolated and user-global: `pixi global` or
  `conda-global`.
- If a program affects research results, make it a **project dependency** rather
  than relying on a user-global installation.
