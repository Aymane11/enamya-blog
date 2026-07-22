---
title: "Partial Unit Test Mock"
date: 2025-10-21T08:55:58+01:00
author: "Aymane BOUMAAZA"
draft: false
description: "partial-unit-test-mock"
tags:
   - TIL
   - python
   - unit-test
   - mock
---

Today I learned how to check for some arguments when using `unittest.mock`.

The problem:
I have the following function:

```python
def my_function(a):
    return function_to_mock(a, object())
```

When testing `my_function` (with a mock for `function_to_mock`), I can check if function_to_mock was called with the right arguments, but I cannot recreate the same `object()` instance passed to it.

```python
from unittest.mock import patch
def test_my_function():
    with patch("function_to_mock") as mocker:
        actual = my_function(1)
        expected = "something"
        assert actual == expected
        # How to check for second argument ??
        # mocker.assert_called_once_with(10, ???)


`unittest.mock.ANY` comes to rescue, and the test can become

```python
from unittest.mock import patch, ANY
def test_my_function():
    with patch("function_to_mock") as mocker:
        actual = my_function(1)
        expected = "something"
        assert actual == expected
        # How to check for second argument ??
        mocker.assert_called_once_with(10, ANY)
````

