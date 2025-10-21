---
title: "Check Multiple Types in Python"
date: 2025-10-21T23:25:32+01:00
author: "Aymane BOUMAAZA"
draft: false
description: "How to check for multiple types in Python"
tags:
 - TIL
 - python
 - type-checking
---

Today I learned how to check for multiple types in Python -in a single line-.

## TL;DR:

You can pass all types to check in a tuple to the `isinstance` function (or use `typing.Union` in Python 3.10+).

---

I discovered this while reading parts of the implementation of [mcp2py](https://github.com/MaximeRivest/mcp2py/blob/31ebab220f94499257b82404f4a83390fd52ace2/src/mcp2py/roots.py#L33C1-L33C39). I came across a call to `isinstance` where the `classinfo` parameter is a tuple (`if isinstance(roots, (str, Path)):`), which was new to me.


I checked the Python docs, took a **very quick** look at the [`isinstance(object, classinfo, /)`](https://docs.python.org/3/library/functions.html#isinstance) documentation, only to find zero examples of its usage _(the absence of examples in the Python doc frustrates me everytime,and I prefer reading examples first)_, then I asked ChatGPT, who explained that the line checks if `roots` is a `str` or a `pathlib.Path` object. I came back to double check in Python docs, which confirmed ChatGPT claims:

> If `classinfo` is a tuple of type objects (or recursively, other such tuples) or a `Union` Type of multiple types, return `True` if `object` is an instance of any of the types

So I learned that instead of calling `isinstance` multiple times for each type, I can simply do:
```python
from typing import Union
from pathlib import Path

str_path = "path/to/file.txt"
path_obj = Path("path/to/file.txt")

isinstance(str_path, (Path, str))  # True
isinstance(path_obj, (Path, str))  # True, too
```
