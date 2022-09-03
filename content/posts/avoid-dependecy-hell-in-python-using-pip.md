---
title: Avoid dependency hell in Python using pip
date: 2022-09-03T11:13:57.554Z
author: Aymane BOUMAAZA
draft: false
tags:
  - TIL
  - python
  - pip
description: Stop having an inconsistent Python environment in you system
---



Today I learned about [pip](https://pip.pypa.io/en/stable/) config [require-virtualenv](https://docs.python-guide.org/dev/pip-virtualenv/), this option prevents installing packages unless you are in a [virtual environment](https://realpython.com/python-virtual-environments-a-primer/).

There are several ways to set it up:
- Via environment variable:
  ```bash
  export PIP_REQUIRE_VIRTUALENV=true
  set PIP_REQUIRE_VIRTUALENV=true  # on Windows
  ```

- Via pip config file:
  ```bash
  pip config set global.require-virtualenv true
  ```

- Via pip config file (`~/.config/pip/pip.conf`):
  ```toml
  [global]
  require-virtualenv = True
  ```

Once you set it up, you will get an error message if you try to install a package without being in a virtual environment:
![pip error message](/pip_require_venv.png)