# Exercise 3: pipx, isolated Python command-line tools

`pipx` installs Python command-line applications, each in its own virtual
environment, and puts just the command on your `PATH`. This is the fix for the
classic problem where `pip install` one tool quietly breaks another by changing
a shared dependency.

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

`cowsay` now works from any directory, but its dependencies live only in its own
environment. Nothing was added to your project or to a shared site-packages.

## 3. See what isolation actually means

```bash
pipx list
```

Each tool is listed with its own environment. If tool A needs `click 7` and
tool B needs `click 8`, both are fine here, because they never share an
environment.

## 4. Run a tool once, without installing it

```bash
pipx run pycowsay "run once, no install"
```

`pipx run` builds a cached environment on first use and reuses it briefly, then
cleans it up. Use this for something you need occasionally.

## 5. Upgrade and remove

```bash
pipx upgrade cowsay
pipx uninstall cowsay
```

Removal is clean: the tool's whole environment and its command go away, with no
guessing about shared packages.

## Takeaways

- Use `pipx` for Python CLI tools you want available system-wide.
- One isolated environment per tool means no dependency collisions.
- `pipx run` is the install-free, one-off equivalent.
- Note: `uv tool install` and `uvx` (from exercise 1) do the same job. If you
  already use `uv`, you may not need pipx separately.
- Limitation: PyPI and Python only. For Conda or bioinformatics tools, use the
  next exercise.
