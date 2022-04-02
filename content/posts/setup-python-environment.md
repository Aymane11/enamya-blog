---
title: Setup Python Environment
date: 2022-04-02T12:59:18Z
author: Aymane BOUMAAZA
draft: false
description: A quick overview on how I setup my Python environment
tags:
  - python
  - environment
  - setup
  - tools
---


## Introduction

Whenever I want to start a new Python project, I used to go for the standard `venv` included with Python. But, I easily got rid of it, since the folder it creates makes my editor too clumsy.

Afterwards, I discovered [`Pipenv`](https://pipenv.pypa.io/) and I was very happy with it, until I figured out that it doesn't work well with some packages (e.g. `colorama`), since it specifies the platform type in the `Pipfile.lock` file, that I always add to my [`.gitignore` file](https://gist.github.com/Aymane11/9c0244405da5e091a9b76e8cbedb7e95). Recently I looked for a solution to this [cross-platform issue](https://github.com/pypa/pipenv/issues/3902), that's when I found out that people are suggesting using [`Poetry`](https://python-poetry.org/), a tool which I already heard about, but never took time to give it a chance.

I finally gave `Poetry` a chance that day, and from that point, `Poetry` became my best friend.

In my first blog post, I'll take you through my Python Setup.

## Poetry

### Installing poetry

> Poetry is a Python packaging and dependency management tool that assists you during the entire development process, from installing dependencies, code packaging, to publishing your code.

#### Powershell

```powershell
(Invoke-WebRequest -Uri https://raw.githubusercontent.com/python-poetry/poetry/master/get-poetry.py -UseBasicParsing).Content | python -
```

#### Bash

```bash
curl -sSL https://raw.githubusercontent.com/python-poetry/poetry/master/get-poetry.py | python -
```

> Verify that the installation was successful by running `poetry --version`.

### Poetry lifecycle

#### 1. Project setup

```console
poetry new blueprint
```

The above command will ask you for meta infos related to the project (version, authors, license...), then a new directory will be created with the following content:

```console
blueprint
├── pyproject.toml        # contains dependencies, project metadata and more
├── README.rst
├── blueprint
│   └── __init__.py
└── tests
    ├── __init__.py
    └── test_blueprint.py
```

#### 2. Add dependencies

In order to install a dependency, you need to run the following command:

```console
poetry add <package>
```

If you'd like to add the dependency as a dev dependency, you can use the `--dev (-D)` flag:

```console
poetry add -D <dev-package>
```

#### 3. Export `requirements.txt` file

This command generates a `requirements.txt` file containing all the dependencies you've added to your project.

```console
poetry export -f requirements.txt --output requirements.txt
```

If you want to export the dev dependencies too, you can use the `--dev` flag:

```console
poetry export -f requirements.txt --dev --output requirements_dev.txt
```

### Poe the Poet

[Poe](https://github.com/nat-n/poethepoet) is a task runner plugin that runs with Poetry, allowing you to run tasks in Poetry's virtual environment. It can be added to projects using different ways, but I prefer adding it a dev dependency.

```console
poetry add --dev poethepoet
```

Tasks are defined withing the `pyproject.toml` file:

```toml
# pyproject.toml
[tool.poe.tasks]
    [tool.poe.tasks.format]
    help = "Run black on the code base"
    cmd  = "black ."
```

And can be run using the following command:

```console
poe format      # or `poetry run poe format` if outside poetry shell
```



## Code Editor

My editor of choice is [Visual Studio Code](https://code.visualstudio.com/) and I use it to develop all my projects.

This is a list of the Python related extensions I use in my editor:

- [Python's official extension](https://marketplace.visualstudio.com/items?itemName=ms-python.python): Includes IntelliSense using Pylance, Linting, Jupyter Notebooks, code formatting, refactoring, unit tests ...
- [Python Test Explorer](https://marketplace.visualstudio.com/items?itemName=LittleFoxTeam.vscode-python-test-adapter): I've been struggling with unit tests discovery on Windows, and this extension helps me to run them.
- [Python Environment Manager](https://marketplace.visualstudio.com/items?itemName=donjayamanne.python-environment-manager): This extension gives a full overview on Python environments.
- [autoDocstring](https://marketplace.visualstudio.com/items?itemName=njpwerner.autodocstring): This extension helps me to generate docstrings for my code, all I have to do is type `"""` and the extension will generate the docstring for me.
- [Sourcery](https://marketplace.visualstudio.com/items?itemName=sourcery.sourcery): An awesome Python refactoring tool.
- [Tabnine](https://marketplace.visualstudio.com/items?itemName=TabNine.tabnine-vscode) and [GitHub Copilot](https://marketplace.visualstudio.com/items?itemName=GitHub.copilot): Cause who doesn't like AI writing code for them?


## Code formatting and linting

For code formatting, I use [`black`](https://github.com/psf/black) and for linting, I use [`flake8`](https://flake8.pycqa.org/en/latest/) or [`pylint`](https://pylint.pycqa.org/en/latest/).


## Unit testing

I admit it, I rarely write tests for my code (cause my code never fails 😎), but on serious projects I use [`pytest`](https://docs.pytest.org/) with [`pytest-cov`](https://pytest-cov.readthedocs.io/en/latest/) for code coverage, plus [`pytest-mock`](https://github.com/pytest-dev/pytest-mock/) mocking fixtures.

With poe, I can run tests with coverage using the following command:

```console
poe test
```

```toml
# pyproject.toml
[tool.poe.tasks]
  [tool.poe.tasks.test]
    help = "Run pytest on the code base"
    cmd = "pytest -v --cov=src --cov-report=term"
```

## Pre-commit hooks

Before publishing my code to GitHub, I always have to re-check my code for linting, formatting, security issues... and I use [`pre-commit`](https://pre-commit.com/) to do that.

It can be installed using the following command:

```
pip install pre-commit
```

Here's a sample of the `.pre-commit-config.yaml` configuration file:

```yaml
# .pre-commit-config.yaml
repos:
-   repo: https://github.com/pre-commit/pre-commit-hooks
    rev: v2.3.0
    hooks:
    -   id: check-yaml
    -   id: end-of-file-fixer
    -   id: trailing-whitespace
    -   id: check-added-large-files
    -   id: detect-private-key
-   repo: https://github.com/psf/black
    rev: 22.1.0
    hooks:
      - id: black
        language_version: python3.8
-   repo: https://gitlab.com/pycqa/flake8
    rev: 3.7.9
    hooks:
    - id: flake8
```

## GitHub actions

The entire pipeline of my projects is managed by `GitHub Actions`, here's the <abbr title="Continuous Integration">CI</abbr> configuration for my projects:

```yaml
name: Check Code

on:
  push:
    branches: [ main ] # on push to main
    paths-ignore:
      - '*.md'         # ignore .md files changes
  pull_request:
    branches: [ main ] # on pull requests to main
    paths-ignore:
      - '*.md'         # ignore .md files changes

jobs:
  ci:
    strategy:
      max-parallel: 2
      matrix:
        python-version: [3.8, 3.9]
        poetry-version: [1.1.13]
        os: [ubuntu-latest, macos-latest, windows-latest]
    runs-on: ${{ matrix.os }}
    timeout-minutes: 10


    steps:
    - name: Check out repository code
      uses: actions/checkout@v2

      # Setup Python (faster than using Python container)
    - name: Setup Python ${{ matrix.python-version }} on ${{ matrix.os }}
      uses: actions/setup-python@v2
      with:
        python-version: ${{ matrix.python-version }}

    - name: Install wheel
      run: python -m pip install wheel

    - name: Install poetry ${{ matrix.poetry-version }}
      uses: abatilo/actions-poetry@v2.0.0
      with:
        poetry-version: ${{ matrix.poetry-version }}

    - name: Cache Poetry virtualenv
      uses: actions/cache@v2
      id: cache
      with:
        path: ~/.virtualenvs
        key: poetry-${{ hashFiles('**/poetry.lock') }}-py${{ matrix.python-version }}-os${{ matrix.os }}
        restore-keys: |
          poetry-${{ hashFiles('**/poetry.lock') }}-py${{ matrix.python-version }}-os${{ matrix.os }}

    - name: Config Poetry
      run: |
        poetry config virtualenvs.in-project false
        poetry config virtualenvs.path ~/.virtualenvs

    - name: Install Dependencies using Poetry
      run: poetry install
      if: steps.cache.outputs.cache-hit != 'true'

    - name: Check for security issues
      run: poetry run poe bandit

    - name: Run test suite
      run: poetry run poe test
```

## Conclusion

There are many libraries and tools that can be added to the list such as [`bandit`](https://bandit.readthedocs.io/en/latest/), [`autoflake`](https://github.com/PyCQA/autoflake), `make`, [`Docker`](https://www.docker.com/), etc. But I'll let those for another day.

I made a [GitHub repo](https://github.com/Aymane11/blueprint) that I use as a template for my Python projects, give it a try and let me know what you think!

Finally, I'd like to wish you all **Ramadan Mubarak**! Hope you liked my first blog post!