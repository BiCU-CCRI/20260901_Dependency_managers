# Exercise 3: pipx, isolated Python command-line tools

`pipx` installs Python command-line applications, each in its own virtual
environment, and puts just the command on your `PATH`. This is the fix for the
classic problem where `pip install` one tool quietly breaks another by changing
a shared dependency.

## 1. Make sure pipx is available

The Codespaces image usually already has `pipx`. Check:

```bash
pipx --version
```

Install it **only if needed**:

```bash
python3 -m pip install --user pipx # install pipx if it isn't installed already
pipx ensurepath      # run ensurepath if you had to install pipx
exec bash            # reload PATH if ensurepath changed it
pipx --version       # check the installation
```

`pipx ensurepath` makes sure the directory pipx links tools into is on your
`PATH`. Reloading the shell applies it.

## 2. Install a tool and run it from anywhere

```bash
pipx install cowsay==6.0          # On purpose not the latest one
cd /tmp
cowsay -t "isolated tool on PATH"
```

`cowsay` now works from any directory for this user, but its dependencies live only in its own
environment. Nothing was added to your project or to a shared site-packages.

## 3. See what isolation actually means

```bash
pipx list
```

Each tool is listed with its own environment. If tool A needs `click 7` and
tool B needs `click 8`, both can coexist, because they do not share the same Python
environment.

## 4. Run a tool once, without installing it

```bash
pipx run pycowsay "run once, no install"
```

Running `pycowsay` (run with `pipx run`) will fail while `cowsay` (installed via `pipx install`) will work:

```bash
pycowsay "I will not work"
cowsay -t "Moo, I work just fine"
```

`pipx run` builds or reuses a cached environment for the application. The cache
 is retained for later runs and expires after a period of inactivity. Use this for
 something you need occasionally.

## 5. Upgrade and remove

```bash
pipx list | grep cowsay
pipx upgrade cowsay
pipx list | grep cowsay
pipx uninstall cowsay
cowsay "Now I don't work either"
```

Removal is clean: the tool's whole environment and its command go away, with no
guessing about shared packages.

## Reproducibility note

The ordinary `pipx install` workflow is designed primarily for isolation and
convenience, not for recording a research project's dependencies. Current pipx
also has manifest/lock workflows that can use PEP 751 `pylock.toml`, but those
locks are explicit rather than something created by every normal install.

Even when a global tool is pinned, if a research result depends on it, prefer to
make it a project dependency so the project itself records the requirement.

## Takeaways

- Use `pipx` for Python CLI tools you want available system-wide.
- One isolated environment per tool prevents Python dependency collisions.
- `pipx run` is the install-free, one-off equivalent.
- `uv tool install` and `uvx` (from exercise 1) cover the same general use case;
  if you already use `uv`, you may not need pipx separately.
- Application-manager isolation is not the same as project reproducibility.
- Limitation: Python applications only. For Conda/Bioconda tools, use the next
  exercise.
