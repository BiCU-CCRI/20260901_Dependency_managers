# When these managers struggle: limitations and failure modes

No tool covers every case. Pick the tool that matches the ecosystem your
software actually lives in, and move to a container or whole-system tool when
the problem is outside the package manager's scope.

## uv

- **Non-Python software.** uv manages the Python packaging ecosystem, not
  arbitrary Conda/system packages such as `samtools`, `bwa`, or R. Workaround:
  use Pixi/Conda for those dependencies, or put the Python layer under Pixi as
  PyPI dependencies.
- **Source-only Python packages.** If no compatible wheel exists, uv builds from
  source and needs a compiler plus development headers/libraries. Workaround:
  install the native build requirements or use a Conda build when available.
- **Heavy native/GPU stacks.** PyTorch/CUDA and geospatial stacks are possible,
  but index/configuration details can be more involved than using a matching
  Conda package set.
- **Tool-version drift.** uv is still versioned `0.x`. Astral considers it stable,
  but minor releases can intentionally contain breaking changes. Pin the uv
  executable in long-lived CI rather than always installing latest.
- **A universal lock is not one universal binary.** The lock can encode choices
  for multiple platforms/Python versions, but the selected wheels and native
  artifacts still vary by platform.

## Pixi

- **Platform availability.** A multi-platform lock cannot solve a target for
  which one of the packages has no build. Bioconda on Apple Silicon is a common
  example: some tools still lack native `osx-arm64` builds. Workaround: choose a
  supported platform, use Rosetta where appropriate, or run Linux in a
  container/Codespace.
- **Conda + PyPI interactions.** Pixi coordinates the Conda resolution with a
  PyPI resolution using uv. Usually this works well, but package-name mappings,
  native ABI assumptions, or a source-built PyPI package can still cause
  problems. Prefer the Conda build when a suitable one exists and native
  compatibility matters.
- **Large multi-platform locks.** `pixi.lock` can be large and can conflict in
  Git. Resolve the human-edited manifest conflict and regenerate the lock rather
  than hand-merging dependency entries.
- **Upstream availability.** A locked Conda build can disappear or a channel can
  change. For archival work, preserve critical artifacts or a built container
  image as well as the lock.
- **Windows/other unsupported targets.** Many bioinformatics packages simply do
  not publish builds for every platform.

## pipx

- **Applications, not importable project libraries.** A package installed with
  pipx lives in the tool's private environment; your project cannot normally
  import it. Use uv/Pixi/a venv for libraries.
- **Plugins must share the tool environment.** Plugin-based applications may
  need `pipx inject` rather than a separate pip install.
- **System dependencies are not managed.** pipx isolates Python package
  dependencies, not arbitrary native binaries or system libraries.
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

- **Lightly maintained and environment-sensitive.** It can be more sensitive to
  the surrounding Conda installation and solver setup. Prefer `pixi global` or
  `conda-global` for new installations.
- **Requires Conda and channel configuration.** A Bioconda tool will not appear
  if the required channels are not configured.

## conda-global

- **Newer project.** It is less mature than long-established pipx and newer than
  Pixi's global workflow.
- **Requires Conda.** Unlike Pixi, it is a Conda plugin rather than a standalone
  environment manager.
- **Manifest, not project lock.** `~/.conda/global.toml` records configured
  tools, but it should not replace a project lockfile when research results
  depend on those tools.

## renv

- **System libraries for compiled R packages.** Packages such as `sf`, `terra`,
  `xml2`, `units`, and `rJava` need native libraries or runtimes outside renv.
  Install those separately, use Pixi/Conda, or use a container/base image that
  already supplies them.
- **Old package versions may require source builds.** Archived binaries are not
  always available. Use a suitable binary snapshot repository where possible or
  preserve the built environment for long-term archival work.
- **R/Bioconductor coupling.** Bioconductor releases are tied to particular R
  versions. renv records the R version but cannot install/change R for you.
- **Sources can disappear.** CRAN archives, GitHub repositories, or private
  sources can become unavailable. Preserve/mirror critical artifacts for
  long-lived projects.
- **Scope.** renv manages R packages, not the R interpreter, Python environments,
  Java, arbitrary system packages, or host configuration.

## Modern Conda

- Current Conda is a stronger baseline than older tutorials often imply: it uses
  libmamba solving by default and supports exact multi-platform lock workflows.
- Its historical workflow still centers on named mutable environments rather
  than automatically maintaining a per-project lock on every dependency change.
- `environment.yml` by itself is usually a manifest/constraint file, not
  equivalent to a concrete multi-platform lock unless it contains fully exact
  package/build information.

## Cross-cutting limits

- **Lockfiles do not pin the complete machine.** Kernel, CPU, GPU driver,
  firmware, and external services can still differ.
- **Package availability is part of reproducibility.** A perfect lock cannot
  fetch an artifact that upstream removed. Mirror or preserve critical artifacts
  for archival work.
- **Private indexes/proxies/authentication.** A collaborator must also have the
  network path and credentials needed to reach private dependencies.
- **Offline/HPC nodes.** Prime caches on a connected login node, use offline
  restore modes where supported, or deploy a container.
- **Disk quotas.** Environment and package caches can be large. Move caches to
  project/scratch storage when needed.
- **Containers are not the entire computer.** They capture much more userspace
  state but normally share the host kernel and can still depend on host GPU
  drivers/hardware. Pin base images by digest for archival builds.

## The short version

- Pure Python project: uv is a strong project-first default.
- Native/bioinformatics/mixed-language project: Pixi is a strong default.
- R package library: renv; add Pixi/container layers when R itself or native
  libraries also need pinning.
- User-global applications: `uv tool`/pipx for Python, `pixi global` or
  `conda-global` for Conda.
- Research-critical executable: keep it in the project, not only in a global
  application manager.
- Need reproducibility beyond packages: add strict CI restores, a pinned
  container image, and artifact preservation as required.
