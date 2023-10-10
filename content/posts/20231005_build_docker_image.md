---
title: "Using Docker to Run Your Python Tests"
date: 2023-10-05T09:27:00+08:00
tags: ["Python", "Docker"]
---

Have you ever come across the infamous 'It works on my machine' issue? I'm sure you have—it's a common challenge in software development. To tackle this problem, Docker containers offer a solution by allowing you to encapsulate your code and execute it in a consistent environment. As a result, Docker is widely adopted in various domains, including the automation of CI/CD pipelines. In this article, I will demonstrate how to leverage Docker to create and execute a Python test environment.
<!--more-->

You can check out the source code from this [Github repository](https://github.com/slchangtw/blog_examples/tree/main/using_docker_to_run_your_python_tests).

# Task Description

In this task, we will test a function that utilizes `pendulum` to compute the difference between two dates. To proceed with this task, you should:

1. have [Docker](https://docs.docker.com/engine/install/) installed on your machine,
2. understand how to use `poetry` for Python package management; refer to this [post]({{< ref "20231004_poetry_packages.md" >}}) for detailed instructions,
3. know how to use `pytest` to test code; refer to this [post]({{< ref "20231001_pytest.md" >}}) for detailed instructions,

# Writing the Function and Tests

The function we are going to test is `calculate_date_difference`, and it should be located in the module `src/calculate_date_difference.py`. Here's the definition of the function:

```python
import pendulum

def calculate_date_difference(
    start_date: pendulum.Date, end_date: pendulum.Date
) -> int:
    difference = end_date - start_date
    return difference.days
```

The test cases are defined within the module `tests/test_calculate_date_difference.py`. You can find the test case definitions below. Additionally, remember to include an empty `__init__.py` file in the tests directory to enable running `pytest tests/` from the current directory.

```python
import pendulum
import pytest

from src.calculate_date_difference import calculate_date_difference


@pytest.mark.parametrize(
    "date_1, date_2, expected",
    [
        ((2023, 1, 1), (2023, 1, 10), 9),
        ((2023, 1, 1), (2023, 1, 1), 0),
        ((2023, 1, 10), (2023, 1, 1), -9),
    ],
)
def test_calculate_date_difference(date_1, date_2, expected):
    start_date = pendulum.datetime(*date_1)
    end_date = pendulum.datetime(*date_2)
    assert calculate_date_difference(start_date, end_date) == expected
```

And your `pyproject.toml` file should appear as follows:

```toml
[tool.poetry]
name = "pendulum-example"
version = "0.1.0"
description = ""
authors = ["Your Name <you@example.com>"]
readme = "README.md"

[tool.poetry.dependencies]
python = "^3.10"
pendulum = "^2.1.2"

[tool.poetry.group.test.dependencies]
pytest = "^7.4.2"

[tool.poetry.group.dev.dependencies]
ipykernel = "^6.25.2"

[build-system]
requires = ["poetry-core"]
build-backend = "poetry.core.masonry.api"
```

# Builing the Docker Image via Dockerfile

In this section, we will generate a Docker image that includes all the essential dependencies for running the tests. To achieve this, we will create a file named `Dockerfile` within the project's root directory. This file should consist of the following instructions:

```Dockerfile
# A
FROM python:3.10.13-slim

# B
ENV PYTHONUNBUFFERED=1 \ 
    PYTHONDONTWRITEBYTECODE=1 \
    PIP_NO_CACHE_DIR=off \
    POETRY_VIRTUALENVS_CREATE=false

# C
COPY src/ src/
COPY tests/ tests/
COPY poetry.lock pyproject.toml ./

# D
RUN pip install -U pip && \
    pip install poetry==1.5.1 && \
    poetry install --no-interaction --no-cache --without dev

# E
ENTRYPOINT ["pytest", "-s", "tests"]
```

Let's go through the instructions one by one:

- A: The `FROM` instruction sets the base image to the official Python `slim` image, which is a lightweight version suitable for production use. Ensure the Python version matches that specified in `pyproject.toml`.
  
- B: The `ENV` instruction defines environment variables in the image:
  - `PYTHONUNBUFFERED=1` disables output buffering, enabling real-time application output.
  - `POETRY_VIRTUALENVS_CREATE=false` prevents Poetry from creating a virtual environment.
  - `PYTHONDONTWRITEBYTECODE=1` stops Python from writing .pyc files to disk.
  - `PIP_NO_CACHE_DIR=off` deactivates the pip cache to reduce image size.

- C: The COPY instruction transfers the source code, tests, `poetry.lock`, and `pyproject.toml` files to the image, similar to the `cp` command in Linux.

- D: The `RUN` instruction executes commands within the image. Here, we install dependencies using Poetry with specific flags:
  - `--no-interaction` prevents user input prompts.
  - `--no-cache` avoids using the cache.
  - `--without dev` excludes dev group dependencies, unnecessary for running tests.

- E: The `ENTRYPOINT` instruction defines the command to execute when the container launches. Here, we run the tests using pytest with the `-s` flag to display the test output.

# Building and Running the Docker Image

To build the Docker image, execute the following command in the project's current directory. The `-t` flag lets you assign a name to the image, and the `.` denotes that we're using the Dockerfile in the current directory. Once the image is built, you can use `docker images` to check if the image has been successfully created.

```bash
$ docker build -t pytest_example .
$ docker images 
```

You can execute the tests using the following command. The `--rm` flag ensures the container is removed after it finishes, and the test results will be displayed in the output.

```bash
$ docker run --rm pytest_example
```

# Learn More

- [Containerize an application](https://docs.docker.com/get-started/02_our_app/).
- [Overview of best practices for writing Dockerfiles](https://docs.docker.com/develop/develop-images/dockerfile_best-practices/)
