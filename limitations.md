# When these managers struggle: limitations and failure modes

No tool covers every case. Pick the tool that matches the ecosystem your software
actually lives in, and drop to a container when the problem is the system layer
rather than the packages.

## uv

- **Non-Python software.** uv only reaches PyPI, so it cannot install compiled tools
  such as `samtools`, `bwa`, or `bedtools`, nor R. Symptom: `uv add samtools`
  finds nothing usable, and the command is not on your `PATH` afterwards.
  Workaround: use pixi or Conda for those tools; keep uv for the pure-Python
  layer, or let pixi manage PyPI packages for you via its `pypi-dependencies`.
- **Source-only packages Python packages.** If a package has no prebuilt wheel
  for your Python version or platform, uv builds it from source and needs build
  tools and development headers. Symptom: a build error mentioning `gcc`,
  `Python.h`, or a missing system library. Workaround: install the compiler and
  `-dev` headers, wait for a wheel, or use a Conda build where the binary already
  exists.
- **Heavy native/GPU stacks.** These are solvable
  with uv but need the correct extra index configured, but are trickier than the
  Conda equivalent. Workaround: follow uv's PyTorch/GPU index guide, or use pixi,
  where CUDA builds come from the channel directly.
- **Tool-version drift.** uv is still below version 1.0 and occasionally ships breaking
  changes between minor versions. Workaround: pin the uv version in CI rather
  than always installing latest.
- **A universal lock is not one universal binary.** The lock can encode choices
  for multiple platforms/Python versions, but the selected wheels and native
  artifacts still vary by platform.

## pixi

- **Platform availability.** Many bioinformatics tools still ship only
  `linux-64` and `osx-64` builds, not native `osx-arm64` (Apple M-series).
  Symptom: on an M-series Mac, `pixi add minimap2` (or `bowtie2`, `blast`, and
  others) fails to solve for `osx-arm64` with a "no candidates were found"
  style error. Workaround: build an `osx-64` environment and run it under
  Rosetta, restrict the project to `linux-64` and develop in a Linux container
  or Codespace, or check whether a native build has since appeared. This is the
  most common pixi *issue*.
- **PyPI + Conda interaction.** Packages not on any Conda channel go
  under `pypi-dependencies` (resolved by uv internally). Usually fine, but the
  Conda-to-PyPI name mapping can occasionally pick the wrong package, or a PyPI
  package can need a system library that the Conda side does not provide.
  Workaround: prefer the Conda build when a package exists on both, and move a
  problematic PyPI dependency into the Conda `dependencies` if a build exists.
- **Large lockfiles and git merge conflicts.** A multi-platform `pixi.lock` is big,
  so two people editing dependencies can hit a lockfile conflict. Workaround: do
  not hand-merge the lock. Resolve the conflict in `pixi.toml`, then run
  `pixi install` to regenerate `pixi.lock`.
- **Upstream availability**. Conda channels occasionally remove or repackage a
  build, so a hash pinned in an old lock can disappear. Workaround: for archival
  reproducibility, capture the environment in a container image, or mirror the
  critical packages.
- **Windows/other unsupported targets.** Many bioinformatics packages simply do
  not publish builds for every platform.

## pipx

- **Applications, not portable libraries.** `pipx install` a library you intend to
  `import` does not work, because the package lives in the tool's private
  environment, not your project. Symptom: the install succeeds but your code
  cannot import it. Workaround: use uv, pixi, or a venv for importable
  dependencies; use pipx only for command-line tools.
- **Plugins must share the tool environment.** Tools with plugin ecosystems (for example MkDocs,
  Jupyter, pytest plugins) will not see a plugin you install separately.
  Symptom: "plugin not found" even though you pip-installed it. Workaround:
  `pipx inject <tool> <plugin>` so the plugin lands in the tool's own
  environment.
- **System dependencies are not managed.** pipx isolates Python dependencies only.
  A CLI that shells out to a system binary or links a C library still needs that
  present on the machine. Workaround: install the system dependency separately,
  or use a Conda-based tool manager (`pixi global`, `condax`) that isolates those
  too.
- **Ordinary installs are not automatically project-locked.** Current pipx also
  supports explicit manifest/PEP 751 lock workflows, which can reproduce a set
  of user applications. That is useful for workstation setup, but a
  research-critical tool is still better represented in the project's own
  dependency graph.

## uv tool

- Same Python-only ecosystem boundary as uv.
- Tool environments are intentionally isolated from the current project. This is
  excellent for personal CLI applications but wrong for a tool that needs to
  import the project's dependencies or whose exact version is part of the
  analysis. Use a project dependency plus `uv run` in that case.

## pixi global

- Global environments are resolved per platform and are described by a global
  manifest rather than the project's `pixi.lock`.
- This makes `pixi global` appropriate for personal user-global applications,
  not as the reproducibility mechanism for research-critical executables.

## condax

- **Lightly maintained and version-sensitive.** On a fresh machine or a newer Conda,
  condax can fail to locate Conda or error during solving. Symptom: a traceback
  about a missing `conda`, or a solver failure on install. Workaround: do not
  debug it; use `pixi global install` instead, which does the same job and is
  actively maintained. This is why exercise 4 leads with `pixi global`.
- **Requires Conda present and channels configured.** Symptom: bioconda tools are not
  found until you add the channel. Workaround: configure `conda-forge` and
  `bioconda`, or again prefer `pixi global`, where channels are explicit.

## conda-global

- **Newer project.** It is less mature than long-established pipx and newer than
  pixi's global workflow.
- **Requires Conda.** Unlike pixi, it is a Conda plugin rather than a standalone
  environment manager.
- **Manifest, not project lock.** `~/.conda/global.toml` records configured
  tools, but it should not replace a project lockfile when research results
  depend on those tools.

## renv

- **System libraries for compiled R packages.** Packages such as `sf`, `terra`,
  `xml2`, `curl`, `units`, and `rJava` need system libraries or runtimes outside renv.
  Symptom: `renv::restore()` fails while
  compiling, with "configuration failed" or a "library not found" message. renv
  manages R packages, not the system libraries they link. Workaround: install
  the system dependencies first (pixi/Conda, or apt), use a base image that has
  them (for example the rocker images), or let pak install system requirements.
- **Old versions may be source-only.** Restoring an exact older version often means a
  source build, because package repositories keep binaries only for current
  versions and serve archived versions as source. Symptom: a slow compile, or a
  failure without a toolchain. Workaround: configure a dated binary repository
  (Posit Public Package Manager snapshots) so `restore()` fetches binaries.
- **R and Bioconductor version coupling.** Bioconductor packages are tied to a
  specific R and Bioconductor release. Restoring a lockfile under a different R
  version can mismatch or fail, and renv cannot change your R for you. Symptom:
  Bioconductor packages refuse to install or resolve to unexpected versions.
  Workaround: match the R version the lockfile expects (via a container or pixi),
  then restore.
- **Sources might vanish or rate-limit.** A package archived from CRAN, or a GitHub
  repository that was deleted or hit an API rate limit, cannot be fetched.
  Symptom: `restore()` errors fetching a specific package. Workaround: set a
  `GITHUB_PAT` (personal access token) to avoid GitHub rate limits, and mirror critical
  packages for the long term.
- **Scope.** renv does not manage R itself, Python bridged through reticulate, Java,
  or system configuration. Pair it with pixi or a container when those matter.

## Modern Conda

- Current Conda is a stronger baseline than older tutorials often imply: it uses
  `libmamba` solving by default and supports exact multi-platform lock workflows.
- Its historical workflow still centers on named mutable environments rather
  than automatically maintaining a per-project lock on every dependency change.
- `environment.yml` by itself is usually a manifest/constraint file, not
  equivalent to a concrete multi-platform lock unless it contains fully exact
  package/build information.

## Limits that apply to all of them

- **Lockfiles do not pin the system layer.** They record package versions and
  hashes, not the operating system, glibc, compiler, or GPU driver. Two machines
  can install the identical locked packages and still behave differently.
  Workaround: when that matters, wrap the environment in a container (Podman or Apptainer
  on HPC, Docker elsewhere) or use a whole-system manager such as Nix or Guix.
- **"Reproducible" depends on upstream staying available.** Removed, or
  re-uploaded packages break a restore even with a perfect lockfile. Workaround:
  for archival work, keep a built container image or a local package mirror, not
  just the lockfile.
- **Private indexes, proxies, and authentication.** Corporate mirrors and private
  registries need credentials and network access that the default installers do
  not assume. Symptom: resolution or download failures behind a proxy.
  Workaround: configure the internal index and tokens explicitly per tool.
- **Offline compute nodes.** Many HPC compute nodes have no internet,
  so resolving or downloading on the node fails. Symptom: network timeouts during
  install on a compute node. Workaround: prime the cache on a login node, then
  install offline (for example `uv sync --offline`), or ship a container.
- **Disk quotas.** Package caches and project libraries can be large and can exhaust
  a home-directory quota on a cluster. Symptom: "disk quota exceeded" during
  install. Workaround: relocate the caches to scratch or project storage with the
  relevant environment variables (for example `UV_CACHE_DIR`, `CONDA_PKGS_DIRS`,
  `RENV_PATHS_CACHE`, and pixi's cache directory).
- **Containers are not the entire computer.** They capture much more userspace
  state but normally share the host kernel and can still depend on host GPU
  drivers/hardware. Pin base images by digest for archival builds.

## The short version

- Need a non-Python or non-R binary tool: uv and renv cannot help; use pixi or
  Conda.
- On Apple Silicon with bioinformatics tools: expect missing `osx-arm64` builds;
  plan for a Linux container.
- Installing a library to import, not a command to run: pipx is the wrong tool.
- Compiling R or Python packages from source: you also need the system
  dependencies and a compiler, which these tools do not provide.
- Reproducing the whole system, not just the packages: reach for a container or
  Nix/Guix.
