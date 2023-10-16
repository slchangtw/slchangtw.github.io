---
title: "Using Github Actions for Automated Testing"
date: 2023-10-13T14:24:50+08:00
tags: ["GitHub Actions", "Testing"]
---


Testing plays a crucial role in software development. When you've completed a code module and wish to merge it into the main branch, it's essential to ensure that the code adheres to the style guidelines and functions correctly. This is where testing becomes crucial. Since testing is a repetitive task, automating the testing process can greatly reduce the time and effort required to validate the code. Therefore, in this post, I'll demonstrate how to utilize GitHub Actions for automating the testing process, which encompasses checking code formatting, linting, and running unit tests
<!--more-->

The source code for this post is available [here](https://github.com/slchangtw/blog_examples/tree/main/.github/workflows).

# Task Description

In this task, we will set up a GitHub Actions workflow to verify code formatting using black, perform linting with ruff, and run unit tests that interact with cloud services. To follow along with this task, you should be familiar with the topics covered in the following posts:

1. [Testing Functions with Localstack for Cloud Service Interactions]({{< ref "20231008_localstack_for_tests.md" >}}).
2. [Ensuring Code Quality with Pre-commit Hooks]({{< ref "20231011_precommit.md">}})

# Setting up GitHub Actions

To set up GitHub Actions, you need to create a `.github/workflows` folder in the root directory of your repository. Then, create a YAML file in the folder to define the workflow. In this post, we'll create a `run_tests.yml` file. Here's the file's content:

```yaml
# A
name: Run tests

on:
  pull_request:
    branches: [ main ]

# B
defaults:
  run:
    working-directory: ci_tests

# C
jobs:
  run_test:
    name: Run tests
    runs-on: ubuntu-latest
    env:
      COMPOSE_FILE: docker-compose.yml
    
    # D
    steps:
      # E
      - name: Checkout code
        uses: actions/checkout@v3

      - name: Set up Python 3.10
        uses: actions/setup-python@v4
        with:
          python-version: 3.10.13

      - name: Upgrade pip
        run: |
          pip install -U pip
      
      # F
      - name: Run black check
        run: |
          pip install black==23.3.0
          black --check .

      - name: Run ruff check
        run: |
          pip install ruff==0.0.278
          ruff check .
      
      # G
      - name: Docker compose build
        run: |
          docker-compose -f $COMPOSE_FILE build

      - name: Run tests
        run: |
          docker-compose -f $COMPOSE_FILE run --rm tests

      - name: Clean up
        run: |
          docker-compose -f $COMPOSE_FILE down
```

Let's review the contents of the file using the section headings

## A. Workflow Trigger

The `name` section specifies the name that will be displayed on GitHub. The `on` section defines the events that trigger the workflow. In this instance, the workflow will be initiated when a pull request is submitted to the `main` branch.


## B. Default Settings

The `defaults` section establishes a set of default settings that will be applicable to all jobs within the workflow. For this project's structure, we specify the `working-directory` as `ci_tests` to run the workflow within the `ci_tests` folder. In practice, you typically wouldn't need to set this, as the default working directory is the root directory of the repository.

## C. Jobs

A workflow consists of one or more `jobs`, which can run in parallel or sequentially. Each job runs within a runner environment specified by `runs-on`. In our case, we've designated `ubuntu-latest` as the runner environment. The `env` section defines the environment variables accessible to the job. In this instance, we set the `COMPOSE_FILE` environment variable to `docker-compose.yml`.

## D. Steps

The `steps` section outlines the actions to be performed within the job. Each step represents a task that can be executed within the runner environment.

## E. Checking out Code and Setting up Python

To enable the workflow to access the repository, we start by checking out the code using `actions/checkout`. Next, we configure Python with `actions/setup-python`. This action installs the specified Python version. Additionally, we update the `pip` package manager with `pip install -U pip`.

## F. Running Black and Ruff

These two steps install black and ruff and perform the respective checks. The `black` step installs the black package and runs the `black --check .` command to assess code formatting. The `ruff` step installs the ruff package and executes the `ruff check .` command to inspect code linting.

## G. Building Docker Compose and Running Tests

As explained in this post, we utilized docker-compose and Localstack for testing functions that interact with cloud services. In this context, the `docker-compose` step manages various tasks. It initiates the image build using `docker-compose -f $COMPOSE_FILE build`, where `$COMPOSE_FILE` corresponds to the environment variable specified in the env section. Subsequently, the `docker-compose` step executes the tests with `docker-compose -f $COMPOSE_FILE run --rm tests`. Finally, it performs resource cleanup via `docker-compose -f $COMPOSE_FILE down`.

# Running the Workflow

To initiate the workflow, you should generate a pull request to the `main` branch. Once the pull request is established, the workflow will automatically activate. To monitor the workflow's status, you can access the `Actions` tab located on the repository's main page. The workflow will be visible within the `All workflows` section, and clicking on it will provide details regarding its progress.
![alt text](https://i.imgur.com/J6wGvXK.png)

# Learn More

- [Understanding GitHub Actions](https://docs.github.com/en/actions/learn-github-actions/understanding-github-actions). This page offers a summary of the fundamental components of GitHub Actions.
- [Using actions/checkout in various scenarios](https://github.com/actions/checkout). This page offers examples of how to include additional arguments for `actions/checkout`.
  