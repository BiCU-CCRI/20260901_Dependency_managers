# Exercise 4: condax and pixi global, isolated Conda command-line tools

`pipx` and `uv tool install` are designed for command-line applications packaged in the Python ecosystem.

Conda packages can include much more than Python: compiled C/C++ programs, R software, system libraries, and bioinformatics tools such as `samtools`, `bwa`, and `bcftools`.

For Conda-packaged command-line tools, `pixi global` provides a similar workflow:

- install the tool into an isolated Conda environment;
- expose its command on your `PATH`;
- use it without activating that environment;
- avoid accumulating unrelated application dependencies in a shared Conda environment.

In 2026, `condax` is only lightly maintained. The actively developed equivalent
is `pixi global` or `conda-global` from conda-incubator. This
exercise shows `pixi global` first (recommended) and `condax` second (so you
recognise it if a colleague uses it).

## Part A: pixi global (recommended)

You already installed `pixi` in exercise 2, so this needs no extra setup.
Bioconda packages are distributed through the `bioconda` channel. Because Pixi uses `conda-forge` by default, specify both channels when installing a Bioconda tool:

```bash
pixi global install -c conda-forge -c bioconda samtools
```

Check that the command is available:

```bash
samtools --version | head -n 1
```

`pixi global install` installs the tool into an isolated Conda environment and exposes its executable on your `PATH`, so you can run it without activating the environment.

Because this is a full Conda environment, compiled libraries and other non-Python dependencies are isolated as well.

Inspect your globally available Pixi tools:

```bash
pixi global list
```

Remove `samtools` when you are finished:

```bash
pixi global uninstall samtools
```

### Why this is preferable to installing tools into Conda `base`

Installing many unrelated applications into one shared Conda environment can lead to dependency conflicts and difficult upgrades.

With `pixi global`, command-line applications can live in separate isolated environments while still behaving like normal commands on your `PATH`.

You also do not need administrator or root access to install them for your own user account.

## Part B: `condax` (the classic tool)

`condax` introduced the same general idea before `pixi global`: install a Conda application into its own environment and expose its command globally for your user account.

Install `condax` with `pipx` so that `condax` itself is isolated:

```bash
pipx install condax
condax --version
```

Install a small command-line tool:

```bash
condax install tabulate
```

The `tabulate` command is now available on your `PATH` from its own Conda environment.

List applications managed by `condax`:

```bash
condax list
```

Remove the application:

```bash
condax remove tabulate
```

### Why `condax` is secondary here

`condax` pioneered this workflow, but development and releases have slowed compared with Pixi.

You may still encounter `condax` in existing setups, so it is useful to recognise the command and understand what it does. For new installations, prefer `pixi global`.

If you encounter compatibility or solver problems with `condax`, use `pixi global` rather than spending course time debugging an older tool.

## What about `conda-global`?

`conda-global` is a newer project from `conda-incubator` that provides a similar per-tool environment model for Conda applications.

It is actively developed, but as of 2026 it is still an emerging option. For this course, use `pixi global` as the default.

## Takeaways

- **Python-packaged CLI tool:** use `uv tool install` or `pipx`.
- **Conda-packaged CLI tool:** use `pixi global install`.
- **Older tool you may encounter:** `condax`.
- **Emerging Conda-native alternative:** `conda-global`.

The key mental model is:

> **Install command-line applications globally for convenience, but isolate their dependencies underneath.**

That gives you commands that behave like normal system tools without turning one shared package-manager environment into a dependency dumping ground.
