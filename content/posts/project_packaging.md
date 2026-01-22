---
title: "Packaging a Python Project like a Pro"
date: 2025-01-21
draft: true
description: "Stop copy-pasting scripts between projects. Learn how to structure and package your Python code to make it installable, importable, and reusable."
tags: ["python", "packaging", "uv", "best-practices"]
author: "Christophe B."
---

You've written a useful Python script. Maybe it's a data transformation, a CLI tool, or a helper module you keep copying into every project. It works, but something is wrong, not to mention when you need to make a change.

You're doing `sys.path` hacks. You're copying files here and there. You're importing things with weird relative paths that break when you run the script from a different directory. Sound familiar ?

There's a better way. And it's not complicated.

This article will walk you through packaging a Python project properly. Not "publish to PyPI" packaging (we'll get to that later) — just "I can install this locally and import it from anywhere" packaging. The kind that makes your code feel like a real project instead of a collection of loose scripts.

We'll build a small but real CLI tool: a JSON flattener that explodes nested structures into CSV rows. Along the way, you'll learn:

- How to structure a Python project using the `src` layout
- What goes into a `pyproject.toml` (and why it's the only config file you need)
- How to install your project in "editable" mode for local development
- How to expose a CLI command that works from anywhere

The complete code is available on GitHub: [json-flatten](https://github.com/christophebnx/json-flatten)

## Why bother packaging?

Let's be honest: you can get pretty far without packaging anything. Scripts work. Notebooks work. Copy-pasting works — until it doesn't.

Here's what proper packaging gives you:

**Clean imports.** No more `sys.path.append("..")` or running scripts from specific directories. Once installed, your module is importable from anywhere:

```python
from json_flatten import flatten_json
```

**Reusable code.** The same codebase works as a library (import it) and as a CLI (run it). No duplication.

**Reproducibility.** Your dependencies are declared in one place. Anyone can install your project and get the same environment.

**It's not that hard.** Seriously. The initial setup takes 10 minutes, and then you never think about it again.

## The project we're building

We'll create a CLI tool called `json-flatten`. It takes a nested JSON file and flattens it into a CSV where each leaf value becomes a row.

Given this input:

```json
{
  "user": "alice",
  "orders": [
    {"id": 1, "items": ["a", "b"]},
    {"id": 2, "items": ["c"]}
  ]
}
```

It produces this output:

```csv
user,orders.id,orders.items
alice,1,a
alice,1,b
alice,2,c
```

Three rows because there are three leaf items (`a`, `b`, `c`). The nested structure is flattened with dot-separated keys.

This is a real use case. If you've ever dealt with nested JSON exports and needed to load them into a flat table, you know the pain.

## Project structure

Here's the layout we'll use:

```
json-flatten/
├── src/
│   └── json_flatten/
│       ├── __init__.py
│       ├── cli.py
│       └── flatten.py
├── tests/
│   └── __init__.py
├── pyproject.toml
└── README.md
```

A few things to note:

**The `src/` layout.** The actual package (`json_flatten/`) lives inside a `src/` directory. This is the recommended layout by the Python packaging community. Why? It prevents a subtle bug where Python might import your local source directory instead of the installed package. With `src/`, you're forced to install the package before you can import it — which is exactly what you want.

**Underscore in the package name.** The repo and project are called `json-flatten` (with a hyphen), but the Python package is `json_flatten` (with an underscore). Hyphens aren't valid in Python identifiers, so this is the convention.

**Separation of concerns.** `flatten.py` contains the core logic — a pure function with no dependencies. `cli.py` handles the command-line interface. This means you can import `flatten_json` in your own code without pulling in CLI dependencies.

## The code (briefly)

The core logic is a recursive function that flattens dicts, explodes arrays, and collects leaf values. Here's the signature:

```python
def flatten_json(data: Any, parent_key: str = "", sep: str = ".") -> list[dict[str, Any]]:
    """
    Flatten a nested JSON structure and explode arrays.
    
    Each leaf value becomes a row. Arrays are exploded so that
    each item generates its own row(s).
    """
```

The implementation is about 40 lines of recursive logic. Nothing fancy — no pandas, no external dependencies. Just Python doing what he is good at.

The CLI uses [Typer](https://typer.tiangolo.com/), which gives us a clean interface with almost no boilerplate:

```python
@app.command()
def main(
    input_file: Path = typer.Argument(..., help="Path to the JSON file to flatten."),
    output: Optional[Path] = typer.Option(None, "--output", "-o", help="Output CSV file."),
    separator: str = typer.Option(".", "--sep", "-s", help="Separator for nested keys."),
) -> None:
    """Flatten a JSON file and output as CSV."""
```

I won't go through every line here — the article is about packaging, not the flatten algorithm nor the typer module (which is very cool). Check the [full source on GitHub](https://github.com/christophebnx/json-flatten) if you're curious.

## The pyproject.toml — the heart of it all

This is where the magic happens. The `pyproject.toml` file is the single source of truth for your project's metadata, dependencies, and build configuration.

Let's break it down section by section.

### Build system

```toml
[build-system]
requires = ["hatchling"]
build-backend = "hatchling.build"
```

This tells Python how to build your package. We're using [Hatchling](https://hatch.pypa.io/) — a modern, fast build backend. You could also use `setuptools`, but Hatchling has better defaults and less configuration.

### Project metadata

```toml
[project]
name = "json-flatten"
version = "0.1.0"
description = "Flatten nested JSON to CSV. Each leaf value becomes a row."
readme = "README.md"
requires-python = ">=3.10"
authors = [
    { name = "Christophe B.", email = "christophe@stayintheloop.dev" },
]
```

Self-explanatory. The `requires-python` field is important — it prevents installation on incompatible Python versions.

### Dependencies

```toml
dependencies = [
    "typer>=0.9.0",
]

[project.optional-dependencies]
dev = [
    "pytest>=7.0",
    "ruff>=0.1.0",
]
```

Two sections here:

- `dependencies`: required for the package to work. Anyone who installs your package gets these.
- `optional-dependencies`: grouped extras. Install with `pip install -e ".[dev]"` to get pytest and ruff for development.

### The CLI entry point

```toml
[project.scripts]
json-flatten = "json_flatten.cli:app"
```

This is the line that makes your CLI work. It says: "when someone types `json-flatten` in their terminal, run the `app` object from `json_flatten.cli`."

After installation, you can run:

```bash
json-flatten input.json -o output.csv
```

From anywhere. No need to navigate to the project directory. No `python -m` prefix. It just works.

### Build configuration

```toml
[tool.hatch.build.targets.wheel]
packages = ["src/json_flatten"]
```

This tells Hatchling where to find the actual package code. Required because we're using the `src/` layout.

### Tool configuration (bonus)

```toml
[tool.ruff]
line-length = 100
target-version = "py310"

[tool.ruff.lint]
select = ["E", "F", "I", "UP"]
```

You can configure your development tools right in `pyproject.toml`. No more separate `.ruff.toml`, `setup.cfg`, or other config files scattered around. One file to rule them all.

## Installing in editable mode

Now for the payoff. Navigate to your project directory and run:

```bash
pip install -e .
```

Or if you're using [uv](https://github.com/astral-sh/uv) (and you should — it's *fast*):

```bash
uv pip install -e .
```

The `-e` flag means "editable" — Python installs a link to your source code instead of copying it. When you modify your code, changes take effect immediately without reinstalling.

That's it. Your package is now installed.

## Verify it works

Let's make sure everything is wired up correctly.

**Test the CLI:**

```bash
json-flatten --help
```

You should see a nicely formatted help message courtesy of Typer.

**Test with real data:**

```bash
echo '{"user": "alice", "tags": ["a", "b", "c"]}' > test.json
json-flatten test.json
```

Output:

```csv
user,tags
alice,a
alice,b
alice,c
```

**Test the import:**

Open a Python shell from any directory:

```python
>>> from json_flatten import flatten_json
>>> flatten_json({"x": [1, 2]})
[{'x': 1}, {'x': 2}]
```

No `sys.path` hacks. No running from a specific folder. It just works — like any other installed package.

## The "aha" moment

Here's what we've achieved:

1. **Structure**: a clean `src` layout that separates source from config
2. **Metadata**: everything in one `pyproject.toml` file
3. **Dual use**: works as a CLI tool *and* as an importable library
4. **Editable install**: change your code, see results immediately

The total setup time? About 10 minutes. And now your code is a proper Python package.

Next time you write something useful, start with this structure. Future you will be grateful.

## What's next?

This article covered local development. But what if you want to share your package with the world — or at least with your team?

In a future post, we'll cover publishing to PyPI: building distributions, versioning, and making your package `pip install`-able by anyone.

For now, go package something. Your `sys.path.append` days are over.

---

*The complete source code is available at [github.com/christophebnx/json-flatten](https://github.com/christophebnx/json-flatten).*