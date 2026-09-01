# Exercise 3: pipx, isolated Python command-line tools

`pipx` installs Python command-line applications, each in its own virtual
environment, and puts just the command on your `PATH`. This avoids the classic
problem where installing one user-level Python tool changes dependencies used
by another.

Rough time: 4 minutes.

## 1. Make sure pipx is available

The Codespaces image usually already has `pipx`. Check, and install it only if
needed:

```bash
pipx --version || python3 -m pip install --user pipx
pipx ensurepath
exec bash            # reload PATH if ensurepath changed it
pipx --version
```

`pipx ensurepath` makes sure the directory pipx links tools into is on your
`PATH`. Reloading the shell applies it.

## 2. Install a tool and run it from anywhere

```bash
pipx install cowsay
cd /tmp
cowsay -t "isolated tool on PATH"
```

`cowsay` now works from any directory for this user, but its dependencies live
only in its own environment. Nothing was added to your project or to a shared
site-packages directory.

## 3. See what isolation actually means

```bash
pipx list
```

Each tool is listed with its own environment. If tool A needs `click 7` and
tool B needs `click 8`, both can coexist because they do not share one Python
environment.

## 4. Run a tool once, without installing it

```bash
pipx run pycowsay "run once, no install"
```

`pipx run` creates or reuses an isolated cached environment for the application.
Use this for something you need occasionally without keeping a normal installed
tool entry.

## 5. Upgrade and remove

```bash
pipx upgrade cowsay
pipx uninstall cowsay
```

Removal is clean: the tool's environment and exposed command are removed without
having to reason about packages shared with unrelated applications.

## Reproducibility note

The ordinary `pipx install` workflow is designed primarily for isolation and
convenience, not for recording a research project's dependencies. Current pipx
also has manifest/lock workflows that can use PEP 751 `pylock.toml`, but those
locks are explicit rather than something created by every normal install.

Even when a global tool is pinned, if a research result depends on it, prefer to
make it a project dependency so the project itself records the requirement.

## Takeaways

- Use `pipx` for Python CLI tools you want available user-globally.
- One isolated environment per tool prevents Python dependency collisions.
- `pipx run` is the one-off equivalent.
- `uv tool install` and `uvx` (from exercise 1) cover the same general use case;
  if you already use `uv`, you may not need pipx separately.
- Application-manager isolation is not the same as project reproducibility.
- Limitation: Python applications only. For Conda/Bioconda tools, use the next
  exercise.
