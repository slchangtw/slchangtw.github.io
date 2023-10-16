---
title: "Ensuring Code Quality with pre-commit Hooks"
date: 2023-10-11T20:42:38+08:00
tags: ["Python", "pre-commit", "black", "ruff"]
---

A hallmark of clean code is consistent formatting, which becomes important when collaborating in a team. While manually using formatters or linters to check code is possible, it's easy to forget. A more effective approach is to use tools like pre-commit hooks for automated code inspection before committing
<!--more-->

The code in this post is available [here](https://github.com/slchangtw/blog_examples/tree/main/ci_tests).

# Formatting and Linting

Before delving into pre-commit hooks, let's first examine the tools we'll be utilizing. `black` is a widely employed Python code formatter that essentially adheres to [PEP 8](https://peps.python.org/pep-0008/) and can swiftly reformat your code. One potential drawback of `black` is that it may convert the entire codebase to its own style. However, in most cases, it proves to be a valuable tool for maintaining consistent formatting.

After installing `black` by `pip` or by a package manger, you can easily run it on a Python file or an entire folder using the following command. The file(s) will be reformatted if no syntax errors are detected.

```bash
$ black <file_or_folder>
```

The Python linter introduced in this post is `ruff`, which is a relatively new tool compared to other linters like `pylint` or `flake8`, but it distinguishes itself with its speed. To execute `ruff` on the specified files or directories (after you installed it), use the following command:

```bash
$ ruff check <file_or_folder>
```

If a linting error can be automatically fixed by `ruff`, you can use the `--fix` flag to correct it:

```bash
$ ruff check --fix <file_or_folder>
```

To customize the linting rules, ruff can be configured using a `pyproject.toml`, `ruff.toml`, or `.ruff.toml` file. For example, if you have a function that returns a very long string, which violates the line length limit, you can add the following configuration to the `pyproject.toml` file to disable the error.

```python
def get_long_string() -> str:
    # This is just an example; in real code, you can
    # use / or parentheses to break a long string.
    return "1231231231231231231231231231231231231231231231231231231231231231231231231222222222"
```

```toml
ignore = ["E501"]
```

# Installing pre-commit Hooks

`pre-commit` can be also installed using `pip` or a package manager. After installation, you can create a `.pre-commit-config.yaml` configuration file. Here's an example that utilizes `black` and `ruff` to check Python files.

```yaml
repos:
  - repo: https://github.com/psf/black
    # Black version.
    rev: 23.3.0
    hooks:
      - id: black
        language_version: python3

  - repo: https://github.com/astral-sh/ruff-pre-commit
    # Ruff version.
    rev: v0.0.278
    hooks:
      - id: ruff
```

To install the `pre-commit` hooks, run the following command:

```bash
$ pre-commit install
```

Once installed, `pre-commit` hooks will execute prior to each commit. If any of the hooks encounter an issue, the commit will be canceled, and an error message will appear.

# Learn More

- [The Black code style](https://black.readthedocs.io/en/stable/the_black_code_style/current_style.html)
- [Configuring Ruff](https://docs.astral.sh/ruff/configuration/). A detailed explanation of the configuration options.
