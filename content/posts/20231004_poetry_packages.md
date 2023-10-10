---
title: "Managing Project Dependencies with Poetry"
date: 2023-10-04T10:43:32+08:00
tags: ["Python", "poetry"]
---

Throughout the development of Python projects, incorporating third-party packages becomes essential. The conventional approach for managing project dependencies involves using a `requirements.txt` file. However, it's easy to overlook updating this file with newly installed packages using `pip freeze > requirements.txt`. Moreover, it can be challenging to tell which dependencies were installed directly or indirectly via `requirements.txt`, making it unclear which packages are genuinely essential after removing some.

To address these issues, it's recommended to adopt a modern package manager like `poetry` for more efficient project dependency management.
<!--more-->

You can see the files in my [repository](https://github.com/slchangtw/blog_examples/tree/main/managing_project_dependencies_with_poetry).

# Installing Poetry in a Pre-existing Project

In the project folder, the first step is to set up a virtual environment. You can refer to my previous post [Use pyenv to Manage Python Versions]({{< ref "20230925_pyenv.md" >}}) for instructions on creating a virtual environment. Once you've created it with name `.venv`, activate the virtual environment and proceed to install poetry using the following command:

```bash
$ . .venv/bin/activate # For Linux and MacOS Users
.venv\Scripts\activate.bat # For Windows Users

$ pip install poetry
```

# Initializing Poetry

You can initialize a new Poetry project by executing the following command. During this process, you'll be prompted to provide some project information. Alternatively, if you wish to skip the interactive process and use default values, you can use `poetry init -n`. Let's proceed with the interactive setup

```bash
$ poetry init

Package name:  poetry-example
Version: 0.1.0
Description: An example project for poetry
Author :  <Your Name>
License: MIT
Compatible Python versions: <Use the default value>

Would you like to define your main dependencies interactively? no
Would you like to define your development dependencies interactively? no
Do you confirm generation? yes
```

You will see a file `pyproject.toml`  in the project folder. This file is the core file containing the project metadata and dependencies. 

# Installing Dependencies

You can start to install dependencies via poetry. For example, you can install `pandas` with the following command. 

```bash
$ poetry add pandas
```

In the main dependencies section of the `pyproject.toml` file, you'll see the following lines. These lines indicate that `pandas` is a required package, and there's a version constraint specified. In this example, the version should be equal to or higher than `2.1.1`. You will also see a `poetry.lock` file in the project folder. This file contains the exact versions of the packages installed. 

```toml
[tool.poetry.dependencies]
...
pandas = "^2.1.1"
```

You can also install a package with a specific version. For instance, you can install `scikit-learn` version `1.2.0` using the following command. The result of this installation will be reflected in the `pyproject.toml` file.

```bash
$ poetry add scikit-learn@1.2.0
```

```toml
[tool.poetry.dependencies]
...
pandas = "^2.1.1"
scikit-learn = "1.2.0"
```

To update a package to its latest version, you can use `poetry update <Package Name>`. 

```bash
$ poetry update scikit-learn
```

However, if you run the command as shown in this example, you won't observe any changes because we've specified the version as `1.2.0`. To enable the package to receive updates, you should modify the version constraint to `"^1.2.0"`. After making this adjustment, you can rerun the update command, and the package should then be updated

```toml
...
scikit-learn = "^1.2.0"
```

Eventually, you can use `poetry remove <Package Name>` to remove a package.

```bash
$ poetry remove scikit-learn 
```

# Organizing Dependencies in Groups

Poetry provides the flexibility to categorize dependencies into groups. By defining these groups, you can install packages for various purposes. For instance, you can add `pytest` to the `test` dependency group and `ipykernel` to the `dev` group using the following command. The dependencies will then appear in their respective groups within the `pyproject.toml` file.

```bash 
$ poetry add -G test pytest
```

```toml
[tool.poetry.group.test.dependencies]
pytest = "^7.4.2"

[tool.poetry.group.dev.dependencies]
ipykernel = "^6.25.2"
```


# Installing Dependencies from `pyproject.toml` and `poetry.lock`

Imagine you have a new machine and need to install the project dependencies. In such a scenario, you can utilize the following command to install the dependencies based on the information in the `pyproject.toml` and `poetry.lock` files. 

```bash
# Before running this command, you can remove the old virtual environment 
# and create a new one to see how it works
$ rm -rf .venv
$ python -m venv .venv
$ . .venv/bin/activate # For Linux and MacOS Users
$ pip install poetry

$ poetry install
```

By default, all **non-optional groups** will be installed by the command `poetry install`. You can designate a group as optional in the `pyproject.toml` as the following example.

```toml
[tool.poetry.group.test]
optional = true

[tool.poetry.group.test.dependencies]
pytest = "^7.4.2"
```

It's important to note that if the `poetry.lock` file is absent, Poetry will resolve and install all dependencies using the latest versions specified in the `pyproject.toml` file. Conversely, if the `poetry.lock` file is present, Poetry will install the exact versions as defined in that file.

Furthermore, you have the option to exclude one or more groups using the --without flag. In this example, only the main dependencies will be installed. This feature is particularly valuable when building a Docker image to install only the necessary dependencies, thereby optimizing build times

```bash
$ poetry install --without test,dev
```

# Learn More

- [Managing dependencies](https://python-poetry.org/docs/master/managing-dependencies/). A comprehensive guide to managing group dependencies with Poetry.
