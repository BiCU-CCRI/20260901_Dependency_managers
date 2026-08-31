# Exercise 5: renv, reproducible R environments

`renv` gives each R project its own private package library and records the
exact version of every package in a lockfile (`renv.lock`). It is the R
counterpart to `uv.lock` or `pixi.lock`. The whole workflow is three functions:
`init`, `snapshot`, `restore`.

`renv` is pre-installed in this Codespace (see `.devcontainer/devcontainer.json`),
and R here is configured to install fast binary packages.

Rough time: 6 minutes.

## 1. Create a project and initialise renv

```bash
cd /tmp
mkdir demo-renv && cd demo-renv
```

Now start R and initialise. Running R non-interactively keeps the tutorial
scriptable:

```bash
R -q -e 'renv::init(bare = TRUE)'
```

`renv::init()` creates a project-local library, an `.Rprofile` that activates it
automatically whenever R starts in this folder, and an initial `renv.lock`. The
`bare = TRUE` option skips the automatic dependency scan so this first step is
fast and deterministic in a live session; in real projects you can drop it and
let renv discover your packages.

## 2. Add a package to the project

Create a tiny script so renv has a dependency to discover, then install it:

```bash
echo 'library(glue); print(glue("renv works: {1 + 1}"))' > analysis.R
R -q -e 'renv::install("glue")'
```

`renv::install()` installs into the project library only, not your global R
library. Another project can use a different version of `glue` without conflict.

Check the script runs:

```bash
Rscript analysis.R
```

Expected: `renv works: 2`.

## 3. Snapshot the exact versions

```bash
R -q -e 'renv::snapshot()'
```

`renv::snapshot()` writes the exact version of every package the project uses
into `renv.lock`. Confirm `glue` is recorded:

```bash
grep -A2 '"glue"' renv.lock
```

This lockfile is the reproducible record. Commit it to git.

## 4. Reproduce the environment from the lockfile

This is what a collaborator does after cloning:

```bash
rm -rf renv/library
R -q -e 'renv::restore(prompt = FALSE)'
Rscript analysis.R
```

`renv::restore()` reads `renv.lock` and reinstalls the exact recorded versions,
rebuilding the project library. The script runs again with the same versions.

## 5. Installing from Bioconductor or GitHub (reference)

renv is not limited to CRAN, which matters for bioinformatics:

```bash
# Examples only, these take longer so skip in the live session:
# R -q -e 'renv::install("bioc::DESeq2")'      # from Bioconductor
# R -q -e 'renv::install("tidyverse/dplyr")'   # from GitHub
```

Whatever the source, `renv::snapshot()` pins it in `renv.lock`.

## Takeaways

- Three functions cover the workflow: `init`, `snapshot`, `restore`.
- Per-project library plus `renv.lock` gives you reproducible R.
- Installs from CRAN, Bioconductor, and GitHub, all pinned in one lockfile.
- Limitation: it manages R packages, not the R interpreter or system libraries.
  Pair it with `pixi` or a container when you also need to pin those.

Commit `renv.lock` and the `.Rprofile` and `renv/activate.R` files. The
`renv/library` directory is git-ignored automatically.
