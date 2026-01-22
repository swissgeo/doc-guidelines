# Python coding guidelines

There are a number of style guides out there. If in doubt, check what [google](http://google.github.io/styleguide/pyguide.html#doc-function-raises) proposes. We largely follow their guidelines. A few important things are reproduced here.

- [1. Linting / Auto-formatting](#1-linting--auto-formatting)
  - [Linting configuration](#linting-configuration)
  - [Formatting configuration](#formatting-configuration)
    - [Import sorting](#import-sorting)
  - [Formatting with Yapf - DEPRECATED](#formatting-with-yapf---deprecated)
  - [Linting with pylint - DEPRECATED](#linting-with-pylint---deprecated)
    - [Ignore linting warning/refactoring/convention](#ignore-linting-warningrefactoringconvention)
  - [Yapf and Pylint IDE Integration - DEPRECATED](#yapf-and-pylint-ide-integration---deprecated)
    - [Visual Studio Code](#visual-studio-code)
    - [PyCharm](#pycharm)
- [2. Naming conventions](#2-naming-conventions)
- [3. Comments and Docstrings](#3-comments-and-docstrings)
  - [Docstrings](#docstrings)
  - [Modules](#modules)
  - [Functions and Methods](#functions-and-methods)
  - [Classes](#classes)
  - [Block and Inline Comments](#block-and-inline-comments)
  - [Punctuation, Spelling, and Grammar](#punctuation-spelling-and-grammar)
- [4. TODO Comments](#4-todo-comments)
- [5. Imports formatting - DEPRECATED](#5-imports-formatting---deprecated)
  - [.isort.cfg](#isortcfg)
- [6. Exceptions](#6-exceptions)
  - [Definition](#definition)
  - [Pros](#pros)
  - [Cons](#cons)
  - [Decision](#decision)
- [7. Error handling - Rules of Thumb](#7-error-handling---rules-of-thumb)
- [8. Introduce Explaining Variable](#8-introduce-explaining-variable)
- [9. Unit Testing frameworks](#9-unit-testing-frameworks)
- [10. Logging](#10-logging)
  - [Logger](#logger)
  - [Configuration](#configuration)
    - [Flask logging configuration](#flask-logging-configuration)
    - [Django logging configuration](#django-logging-configuration)
    - [Gunicorn logging configuration](#gunicorn-logging-configuration)
- [11. Dependency Management](#11-dependency-management)

**The foremost goal is that reading and understanding your python code is easy for someone else (or yourself in a few months time).**

## 1. Linting / Auto-formatting

We use [ruff](https://docs.astral.sh/ruff/) for linting (including security linting with bandit rules) and code formatting.

> [!NOTE]
> We used to use pylint, yapf and isort for linting and formating and some project might still use those tools instead of ruff.
> Migration to ruff decision is done on a per project basis depending on the effort/benefit, which means that we still have
> project using those older tools.
> However all new project should use ruff instead.

### Linting configuration

When starting new project we should use the most restrictive rules and deactivate rules during the project development
based on the project needs.

Here below is a minimal `ruff` linting `pyproject.toml` configuration to use when starting a new project.

```toml
#------------------------------------------------------------------------------
# Linting configuration
[tool.ruff.lint]
select = ["ALL"]
ignore = [
    "CPY",    # ignore copyright
    "D100",   # Ignore missing docstring in module
    "D213",   # Ignore docstring conflicting rule
    "D203",   # Ignore docstring conflicting rule
    "FBT001", # Ignore boolean-type-hint-positional-argument
    "FBT002", # Ignore boolean-default-value-positional-argument
    "PTH123", # Ignore builtin-open
    "TD",     # Ignore flake8-todos
    "FIX",    # ignore fixme comments
    "COM812", # flak8-commas missing-trailing-comma is ignored to avoid conflict with formatter which enforces trailing commas
]

# Allow fix for all enabled rules (when `--fix`) is provided.
unfixable = []

fixable = ["ALL"]
# Allow unused variables when underscore-prefixed.
dummy-variable-rgx = "^(_+|(_+[a-zA-Z0-9_]*[a-zA-Z0-9]+?))$"

#------------------------------------------------------------------------------
# ruff ignore per files
[tool.ruff.lint.per-file-ignores]

"tests/**" = [
    "S101",   # Ignore usage of assert statements in tests
    "INP001", # Ignore implicit namespace packages in tests
]

```

To run linting do

```bash
ruff check
```

Or with auto fix

```bash
ruff check --fix
```

### Formatting configuration

Here below is our formatting ruff configuration in `pyproject.toml` that we use

```toml
#------------------------------------------------------------------------------
# Formatting configuration
[tool.ruff.format]
# Like Black, use double quotes for strings.
quote-style = "double"

# Like Black, indent with spaces, rather than tabs.
indent-style = "space"

# Like Black, respect magic trailing commas.
skip-magic-trailing-comma = false

# Like Black, automatically detect the appropriate line ending.
line-ending = "auto"
```

To run the formatting do

```bash
ruff format
```

#### Import sorting

Import sorting is done by ruff, but currently `ruff format` don't support it and we have to do it
via `ruff check` linter as follow:

```bash
ruff check --select I --fix
```

### Formatting with Yapf - DEPRECATED

> [!warning] DEPRECATED - use ruff instead

Most of the current formatters for Python --- e.g., autopep8, and pep8ify --- are made to remove lint errors from code. This has some obvious limitations. For instance, code that conforms to the PEP 8 guidelines may not be reformatted. But it doesn't mean that the code looks good.

[YAPF](https://github.com/google/yapf/) takes a different approach. In essence, the algorithm takes the code and reformats it to the best formatting that conforms to the style guide, even if the original code didn't violate the style guide: end all holy wars about formatting - if the whole codebase of a project is simply piped through YAPF whenever modifications are made, the style remains consistent throughout the project and there's no point arguing about style in every code review.

The ultimate goal is that the code YAPF produces is as good as the code that a programmer would write if they were following the style guide. It takes away some of the drudgery of maintaining your code.

We use the `google` style with a few small modifications. Simply copy-paste this to the root of every new project.

```ini
[style]
based_on_style=google
# Put closing brackets on a separate line, dedent, if the bracketed
# expression can't fit in a single line. Applies to all kinds of brackets,
# including function definitions and calls. For example:
#
#   config = {
#       'key1': 'value1',
#       'key2': 'value2',
#   }        # <--- this bracket is dedent and on a separate line
#
#   time_series = self.remote_client.query_entity_counters(
#       entity='dev3246.region1',
#       key='dns.query_latency_tcp',
#       transform=Transformation.AVERAGE(window=timedelta(seconds=60)),
#       start_ts=now()-timedelta(days=3),
#       end_ts=now(),
#   )        # <--- this bracket is dedent and on a separate line
dedent_closing_brackets=True
coalesce_brackets=True

# This avoid issues with complex dictionary
# see https://github.com/google/yapf/issues/392#issuecomment-407958737
indent_dictionary_value=True
allow_split_before_dict_value=False

# Split before arguments, but do not split all sub expressions recursively
# (unless needed).
split_all_top_level_comma_separated_values=True

# Split lines longer than 100 characters (this only applies to code not to
# comment and docstring)
column_limit=100
```

### Linting with pylint - DEPRECATED

> [!warning] DEPRECATED - use ruff instead

Although formatting is good, it doesn't check for syntax errors nor for non pythonic idioms or bad code practice.
Therefore we also use a linter. There are several linter on the market (pylint, flake8, bandit, ...), we use
[pylint](http://pylint.pycqa.org/en/stable/) because it cover most of the errors and also has the advantage to be able
to disable rules by alias instead of by code (e.g. `# pylint: disable=unused-import` instead of `# noqa: F401` for flake8).

We use the following `pylint` configuration: [.pylintrc](assets/.pylintrc)

#### Ignore linting warning/refactoring/convention

There might be good reason to disable locally some linting messages. When doing this, the reason why we disable a rule
should be documented next to the disable pragma. Disable pragma can be entered as following:

- single line

```python
a, b = ... # pylint: disable=unbalanced-tuple-unpacking
```

- single scope

```python
def test():
    # Disable all the no-member violations in this function
    # pylint: disable=no-member
    ...
```

- block

```python
def test(self):
    ...
    if self.bla:
        # Disable all line-too-long in this if block
        # pylint: disable=line-too-long
        print('This block contain very very long lines that we explicitly don\'t want to split into several lines to match the 100 characters max line length rule')
    ...
```

For more detail in ignoring pylint issues see
[Pylint Messages Control](http://pylint.pycqa.org/en/stable/user_guide/message-control.html)

### Yapf and Pylint IDE Integration - DEPRECATED

> [!warning] DEPRECATED - use ruff instead

#### Visual Studio Code

To integrate `yapf` and `pylint` into Visual Studio Code simply add the following settings into the user or workspace
settings:

```json
{
  "editor.formatOnSave": true,
  "files.trimTrailingWhitespace": true,
  "files.autoSave": "onFocusChange",
  "python.pythonPath": ".venv/bin/python",
  "python.formatting.provider": "yapf",
  "python.linting.enabled": true,
  "python.linting.pylintEnabled": true,
  "python.linting.ignorePatterns": [
    ".vscode/*.py",
    "**/site-packages/**/*.py",
    ".venv",
    "build"
  ]
}
```

#### PyCharm

[How to setup](https://www.jetbrains.com/help/pycharm/configuring-third-party-tools.html)

## 2. Naming conventions

Python code must follow these naming conventions:

- module: snake_case
- constant: UPPER_CASE
- variable: snake_case
- function/method: snake_case
- argument: snake_case
- class: PascalCase
- attribute: snake_case

These naming conventions are checked by `pylint` (see `*-naming-style` keys in [pylintrc](assets/.pylintrc#L222))

## 3. Comments and Docstrings

Be sure to use the right style for module, function, method docstrings and inline comments.

### Docstrings

Python uses docstrings to document code. A docstring is a string that is the first statement in a package, module, class or function. These strings can be extracted automatically through the __doc__ member of the object and are used by pydoc. (Try running pydoc on your module to see how it looks.) Always use the three double-quote `"""` format for docstrings (per PEP 257). A docstring should be organized as a summary line (one physical line) terminated by a period, question mark, or exclamation point, followed by a blank line, followed by the rest of the docstring starting at the same cursor position as the first quote of the first line. There are more formatting guidelines for docstrings below.

### Modules

Every file should contain license boilerplate. Choose the appropriate boilerplate for the license used by the project (for example, Apache 2.0, BSD, LGPL, GPL)

Files should start with a docstring describing the contents and usage of the module.

```python
"""A one line summary of the module or program, terminated by a period.

Leave one blank line.  The rest of this docstring should contain an
overall description of the module or program.  Optionally, it may also
contain a brief description of exported classes and functions and/or usage
examples.

  Typical usage example:

  foo = ClassFoo()
  bar = foo.FunctionBar()
"""
```

### Functions and Methods

In this section, "function" means a method, function, or generator.

A function must have a docstring, unless it meets all of the following criteria:

- not externally visible
- very short
- obvious

A docstring should give enough information to write a call to the function without reading the function’s code. The docstring should be descriptive-style (`"""Fetches rows from a Bigtable."""`) rather than imperative-style (`"""Fetch rows from a Bigtable."""`), except for @property data descriptors, which should use the same style as attributes. A docstring should describe the function’s calling syntax and its semantics, not its implementation. For tricky code, comments alongside the code are more appropriate than using docstrings.

A method that overrides a method from a base class may have a simple docstring sending the reader to its overridden method’s docstring, such as `"""See base class."""`. The rationale is that there is no need to repeat in many places documentation that is already present in the base method’s docstring. However, if the overriding method’s behavior is substantially different from the overridden method, or details need to be provided (e.g., documenting additional side effects), a docstring with at least those differences is required on the overriding method.

Certain aspects of a function should be documented in special sections, listed below. Each section begins with a heading line, which ends with a colon. All sections other than the heading should maintain a hanging indent of two or four spaces (be consistent within a file). These sections can be omitted in cases where the function’s name and signature are informative enough that it can be aptly described using a one-line docstring.

**Args:**
    List each parameter by name. A description should follow the name, and be separated by a colon and a newline. Optionally can the type
    of parameter be specified next to the name after the colon separator. If the description is too long to fit on a single 80-character line, use a hanging indent of 4 spaces. The description should include required type(s) if the code does not contain a corresponding type annotation. If a function accepts `*foo` (variable length argument lists) and/or `**bar` (arbitrary keyword arguments), they should be listed as `*foo` and `**bar`.

**Returns:** (or Yields: for generators)
    Describe the type and semantics of the return value. If the function only returns None, this section is not required. It may also be omitted if the docstring starts with Returns or Yields (e.g. """Returns row from Bigtable as a tuple of strings.""") and the opening sentence is sufficient to describe return value.

**Raises:**
    List all exceptions that are relevant to the interface. You should not document exceptions that get raised if the API specified in the docstring is violated (because this would paradoxically make behavior under violation of the API part of the API).

```python
def fetch_bigtable_rows(big_table, keys, other_silly_variable=None):
    """Fetches rows from a Bigtable.

    Retrieves rows pertaining to the given keys from the Table instance
    represented by big_table.  Silly things may happen if
    other_silly_variable is not None.

    Args:
        big_table:
            An open Bigtable Table instance.
        keys: list
            A sequence of strings representing the key of each table row
            to fetch.
        other_silly_variable: string
            Another optional variable, that has a much
            longer name than the other args, and which does nothing.

    Returns:
        A dict mapping keys to the corresponding table row data
        fetched. Each row is represented as a tuple of strings. For
        example:

        {'Serak': ('Rigel VII', 'Preparer'),
         'Zim': ('Irk', 'Invader'),
         'Lrrr': ('Omicron Persei 8', 'Emperor')}

        If a key from the keys argument is missing from the dictionary,
        then that row was not found in the table.

    Raises:
        IOError: An error occurred accessing the bigtable.Table object.
    """
```

### Classes

Classes should have a docstring below the class definition describing the class. If your class has public attributes, they should be documented here in an Attributes section and follow the same formatting as a function’s Args section.

```python
class SampleClass(object):
    """Summary of class here.

    Longer class information....
    Longer class information....

    Attributes:
        likes_spam:
            A boolean indicating if we like SPAM or not.
        eggs:
            An integer count of the eggs we have laid.
    """

    def __init__(self, likes_spam=False):
        """Inits SampleClass with blah."""
        self.likes_spam = likes_spam
        self.eggs = 0

    def public_method(self):
        """Performs operation blah."""
```

### Block and Inline Comments

The final place to have comments is in tricky parts of the code. Comments should not document the what, but the why: What the code does, the code itself should describe in a readable and comprehensible way, but why the code was written in a certain way and what trade-offs were made should be written in a comment. If you’re going to have to explain it at the next code review, you should comment it now. Complicated operations get a few lines of comments before the operations commence. Non-obvious ones get comments at the end of the line.

```python
# We use a weighted dictionary search to find out where i is in
# the array.  We extrapolate position based on the largest num
# in the array and the array size and then do binary search to
# get the exact number.

if i & (i-1) == 0:  # True if i is 0 or a power of 2.
```

To improve legibility, these comments should start at least 2 spaces away from the code with the comment character `#`, followed by at least one space before the text of the comment itself.

On the other hand, never describe the code. Assume the person reading the code knows Python (though not what you’re trying to do) better than you do.

```python
# BAD COMMENT: Now go through the b array and make sure whenever i occurs
# the next element is i+1
```

### Punctuation, Spelling, and Grammar

Pay attention to punctuation, spelling, and grammar; it is easier to read well-written comments than badly written ones.

Comments should be as readable as narrative text, with proper capitalization and punctuation. In many cases, complete sentences are more readable than sentence fragments. Shorter comments, such as comments at the end of a line of code, can sometimes be less formal, but you should be consistent with your style.

Although it can be frustrating to have a code reviewer point out that you are using a comma when you should be using a semicolon, it is very important that source code maintain a high level of clarity and readability. Proper punctuation, spelling, and grammar help with that goal.

## 4. TODO Comments

Use `TODO` comments for code that is temporary, a short-term solution, or good-enough but not perfect.

A `TODO` comment begins with the string `TODO` in all caps and the abbreviation (e.g. boc) or other identifier of the person or issue with the best context about the problem. This is followed by an explanation of what there is to do.

The purpose is to have a consistent `TODO` format that can be searched to find out how to get more details. A `TODO` is not a commitment that the person referenced will fix the problem. Thus when you create a `TODO`, it is almost always your name that is given.

```python
# TODO(kl@gmail.com): Use a "*" here for string repetition.
# TODO(boc) Change this to use relations.
```

If your `TODO` is of the form "At a future date do something" make sure that you either include a very specific date ("Fix by November 2009") or a very specific event ("Remove this code when all clients can handle XML responses.").

## 5. Imports formatting - DEPRECATED

Imports should be on separate lines.

E.g.:

```python
Yes: import os
     import sys

No:  import os, sys
```

Imports are always put at the top of the file, just after any module comments and docstrings and before module globals and constants. Imports should be grouped from most generic to least generic:

1. Python standard library imports. For example:

    ```python
    import sys
    ```

1. third-party module or package imports. For example:

    ```python
    import tensorflow as tf
    ```

1. Code repository sub-package imports. For example:

    ```python
    from otherproject.ai import mind
    ```

1. application-specific imports that are part of the same top level sub-package as this file. For example:

    ```python
    from myproject.backend.hgwells import time_machine
    ```

Within each grouping, imports should be sorted lexicographically, ignoring case, according to each module’s full package path. Code may optionally place a blank line between import sections.

```python
import collections
import queue
import sys

from absl import app
from absl import flags
import bs4
import cryptography
import tensorflow as tf

from book.genres import scifi
from otherproject.ai import body
from otherproject.ai import mind
from otherproject.ai import soul

from myproject.backend.hgwells import time_machine
from myproject.backend.state_machine import main_loop
```

[`isort`](https://timothycrosley.github.io/isort/) does the job of grouping and sorting your imports alphabetically and according to your configuration. If you have packages that you want to have treated as thirdparty, add them to `known_third_party`. If you wanna specially group imports from e.g. a framework, you can create a custom `known_acme` and add `ACME` to the `sections`

### .isort.cfg

```ini
[settings]
known_third_party=pytest
known_django=django
known_flask=flask
force_single_line=True
sections=FUTURE,STDLIB,THIRDPARTY,DJANGO,FLASK,FIRSTPARTY,LOCALFOLDER

# other possible options
# line_length=80
# force_to_top=file1.py,file2.py
# skip=file3.py,file4.py
# known_future_library=future,pies
# known_standard_library=std,std2
# known_third_party=randomthirdparty
# known_first_party=mylib1,mylib2
# indent='    '
# multi_line_output=0  # 0 is default
# length_sort=1
# forced_separate=django.contrib,django.utils
# default_section=FIRSTPARTY
# no_lines_before=LOCALFOLDER

```

## 6. Exceptions

Exceptions are allowed but must be used carefully.

### Definition

Exceptions are a means of breaking out of the normal flow of control of a code block to handle errors or other exceptional conditions.

### Pros

The control flow of normal operation code is not cluttered by error-handling code. It also allows the control flow to skip multiple frames when a certain condition occurs, e.g., returning from N nested functions in one step instead of having to carry-through error codes.

### Cons

May cause the control flow to be confusing. Easy to miss error cases when making library calls.

### Decision

Exceptions must follow certain conditions:

- Raise exceptions like this: `raise MyError('Error message')` or `raise MyError()`. Do not use the two-argument form (`raise MyError, 'Error message'`).

- Make use of built-in exception classes when it makes sense. For example, raise a `ValueError` to indicate a programming mistake like a violated precondition (such as if you were passed a negative number but required a positive one). Do not use assert statements for validating argument values of a public API. assert is used to ensure internal correctness, not to enforce correct usage nor to indicate that some unexpected event occurred. If an exception is desired in the latter cases, use a raise statement. For example:

  ```python
  # Yes:
    def connect_to_next_port(self, minimum):
      """Connects to the next available port.

      Args:
        minimum: A port value greater or equal to 1024.

      Returns:
        The new minimum port.

      Raises:
        ConnectionError: If no available port is found.
      """
      if minimum < 1024:
        # Note that this raising of ValueError is not mentioned in the doc
        # string's "Raises:" section because it is not appropriate to
        # guarantee this specific behavioral reaction to API misuse.
        raise ValueError('Minimum port must be at least 1024, not %d.' % (minimum,))
      port = self._find_next_open_port(minimum)
      if not port:
        raise ConnectionError('Could not connect to service on %d or higher.' % (minimum,))
      assert port >= minimum, 'Unexpected port %d when minimum was %d.' % (port, minimum)
      return port
  ```

  ```python
  # No:
    def connect_to_next_port(self, minimum):
      """Connects to the next available port.

      Args:
        minimum: A port value greater or equal to 1024.

      Returns:
        The new minimum port.
      """
      assert minimum >= 1024, 'Minimum port must be at least 1024.'
      port = self._find_next_open_port(minimum)
      assert port is not None
      return port
  ```

- Libraries or packages may define their own exceptions. When doing so they must inherit from an existing exception class. Exception names should end in `Error` and should not introduce stutter (foo.FooError).
- Never use catch-all except: statements, or catch `Exception` or `StandardError` (see also [7. Error handling - Rules of Thumb](#7-error-handling---rules-of-thumb)), unless you are
  - re-raising the exception, or
  - creating an isolation point in the program where exceptions are not propagated but are recorded and suppressed instead, such as protecting a thread from crashing by guarding its outermost block
  
  Catch-all exception is check by `pylint` and is configured via `overgeneral-exceptions` in [.pylintrc](assets/.pylintrc#L526).
  Python is very tolerant in this regard and except: will really catch everything including misspelled names, `sys.exit()` calls, `Ctrl+C` interrupts, unittest failures and all kinds of other exceptions that you simply don’t want to catch.
- Minimize the amount of code in a `try/except` block. The larger the body of the try, the more likely that an exception will be raised by a line of code that you didn’t expect to raise an exception. In those cases, the `try/except` block hides a real error.
- Use the `finally` clause to execute code whether or not an exception is raised in the try block. This is often useful for cleanup, i.e., closing a file.
- When capturing an exception, use `as` rather than a comma. For example:

  ```python
  try:
    raise Error()
  except Error as error:
    pass
  ```

- Every exception must also be logged with the correct severity level; `CRITICAL`.
- When re-raising exception, in order to have a comprehensible backtrace always use `raise ... from ...` form. There is two use cases with the `from`:
  1. The new exception contains all useful information and/or the exception is meant to be anyway handle later on (e.g. Django `ValidationError` exception). In this case in order to have consice backtrace uses `from None`

  ```python
  def get_value(my_dict, key):
    try:
      return my_dict[key]
    except KeyError as error:
      raise KeyError(f'Key {key} missing from my_dict') from None

  # This would result to such backtrace
  >>> get_value({}, 'my_key')
  Traceback (most recent call last):
    File "<stdin>", line 1, in <module>
    File "<stdin>", line 5, in get_value
  KeyError: 'Key my_key missing from my_dict'

  # instead of 
  >>> get_value({}, 'my_key')
  Traceback (most recent call last):
    File "<stdin>", line 3, in get_value
  KeyError: 'my_key'

  During handling of the above exception, another exception occurred:

  Traceback (most recent call last):
    File "<stdin>", line 1, in <module>
    File "<stdin>", line 5, in get_value
  KeyError: 'Key my_key missing from my_dict'
  ```

  1. The original exception still contains useful information, therefore use `from error` to have the original and new backtrace

  ```python
  def do_something():
    try:
      raise ValueError('Original error')
    except ValueError as error:
      raise RuntimeError('This should not happen') from error

  # This generate the following backtrace
  >>> do_something()
  Traceback (most recent call last):
    File "<stdin>", line 3, in do_something
  ValueError: Original error

  The above exception was the direct cause of the following exception:

  Traceback (most recent call last):
    File "<stdin>", line 1, in <module>
    File "<stdin>", line 5, in do_something
  RuntimeError: This should not happen
  
  # Instead of
  >>> do_something()
  Traceback (most recent call last):
    File "<stdin>", line 3, in do_something
  ValueError: Original error

  During handling of the above exception, another exception occurred:

  Traceback (most recent call last):
    File "<stdin>", line 1, in <module>
    File "<stdin>", line 5, in do_something
  RuntimeError: This should not happen
  # As you noticed the message is slightly different..
  ```

  For more information about `raise ... from ...` form see [Raise … from … in Python](https://stefan.sofa-rockers.org/2020/10/28/raise-from/)

## 7. Error handling - Rules of Thumb

- Only handle known _Exceptions_ -> **NO BROAD EXCEPTION !**

  ```python
  # NEVER DO THIS !
  try:
    return something()
  except:
    return ''

  # NOR THIS !
  try:
    return something()
  except Exception:
    return ''
  ```

- Only handle _Exceptions_ if you know how to fix it

  ```python
  # DON'T DO THIS !
  try:
    return something(param)
  except ValueError:
    return ''

  # But you can do this
  try:
    return True, something(param)
  except ValueError as err:
    logger.error('invalid param: %s', err)
    return False, 'invalid param: %s' % (err)
  ```

- Always log backtrace for unexpected _Exceptions_

  ```python
  try:
    return something()
  except Exception as err:
    logger.exception(err)
    raise
  ```

- Always use `from` when re-raising new exception

  ```python
  # When the re-raise exception contains all information use `from None`
  try:
    return something()
  except KeyError as err:
    raise KeyError(f'Key {err} is missing') from None

  # or when original exception contains useful information
  try:
    return something(param)
  except ValueError as err:
    raise MyException('Invalid parameter') from err
  ```

- Let higher level application handle unexpected _Exceptions_ whenever possible
  - Flask and Django handles all unexpected _Exceptions_ with logging backtrace and returns a `500, Internal Server Error`

- Let crash the application with unexpected _Exceptions_ rather sooner than later

## 8. Introduce Explaining Variable

This will help to explain the meaning of each variable when expressions are hard to read.

```python
# maybe not the most illustrative example, but you get the idea
# change
if ( "MAC" in platform.upper() and \
    "IE" in browser.upper() and \
    was_initialized() and \
    resize > 0 ):
    # do something

# to
is_mac_os = "MAC" in platform.upper()
is_IEBrowser = "IE" in browser.upper()
was_resized = resize > 0
if (is_mac_os and is_IEBrowser and was_initialized() and was_resized):
    # do something
```

## 9. Unit Testing frameworks

Python comes with a fairly mature unit testing framework [`unittest`](https://docs.python.org/3/library/unittest.html) that should be used. There are a number of different test runners available:

- nose2 (successor of `nose` which isn't developed anymore)
- pytest
- various test runners included in frameworks

In any case, the tests should be based on `unittest`.

## 10. Logging

Python comes with a good logging framework that we should use for logging. Unfortunately this framework lack for good JSON formatting which is the best logging format when working with ELK (Elasticsearch-Logstash-Kibana). Therefore we use the [`logging-utilities`](https://pypi.org/project/logging-utilities/) library for JSON support, Flask Request extension and ISO time.

### Logger

We should always use a logger per module named after it as follow:

```python
# my_module.py

import logging

logger = logging.getLogger(__name__)

logger.info('This is my module logger')
```

**NOTE:** don't reuse the Flask app logger.

### Configuration

Logging should be configured via a yaml file as follow:

```python
import logging
import logging.config
import os

import yaml


logger = logging.getLogger(__name__)

def get_logging_cfg():
    cfg_file = os.getenv('LOGGING_CFG', 'logging-cfg-local.yml')

    config = {}
    with open(cfg_file, 'rt') as fd:
        config = yaml.safe_load(fd.read())

    logger.debug('Load logging configuration from file %s', cfg_file)
    return config


def init_logging():
    config = get_logging_cfg()
    logging.config.dictConfig(config)
```

Each application should use the following configurations depending on the environment:

- [logging-cfg-local.yml](assets/logging-cfg-local.yaml) (local development)
- [logging-cfg-flask-dev.yml](assets/logging-cfg-flask-dev.yaml)
- [logging-cfg-flask-prod.yml](assets/logging-cfg-flask-prod.yaml)
- [logging-cfg-django-dev.yml](assets/logging-cfg-django-dev.yaml)
- [logging-cfg-django-prod.yml](assets/logging-cfg-django-prod.yaml)

#### Flask logging configuration

When serving flask directly (e.g. `make serve`) we need to configure logging by calling the `init_logging()` method insid the `service_launcher.py` file (see [geoadmin/template-service-flask/service_launcher.py](https://github.com/geoadmin/template-service-flask/blob/master/service_launcher.py))

#### Django logging configuration

When serving Django directly (e.g. `make serve` without WSGI server), we need to set the logging configuration in `project/settings.py` as follow:

```python
# Logging
# https://docs.djangoproject.com/en/3.1/topics/logging/


# Read configuration from file
def get_logging_config():
    '''Read logging configuration

    Read and parse the yaml logging configuration file passed in the environment variable
    LOGGING_CFG and return it as dictionary
    '''
    log_config = {}
    with open(os.getenv('LOGGING_CFG', 'logging-cfg-local.yml'), 'rt') as fd:
        log_config = yaml.safe_load(fd.read())
    return log_config


LOGGING = get_logging_config()
```

#### Gunicorn logging configuration

To configure gunicorn logging you need to set the `logconfig_dict` config with the output of `get_logging_cfg()`

```python
# We use the port 5000 as default, otherwise we set the HTTP_PORT env variable within the container.
if __name__ == '__main__':
    HTTP_PORT = str(os.environ.get('HTTP_PORT', "5000"))
    # Bind to 0.0.0.0 to let your app listen to all network interfaces.
    options = {
        'bind': '%s:%s' % ('0.0.0.0', HTTP_PORT),
        'worker_class': 'gevent',
        'workers': 2,  # scaling horizontally is left to Kubernetes
        'timeout': 60,
        'logconfig_dict': get_logging_config()
    }
    StandaloneApplication(application, options).run()
```

## 11. Dependency Management

All packages used in production should be pinned to a major version. Automatically updating these packages will install the latest minor or patch version available within that major release. Development packages, on the other hand, should not be pinned unless they need to match a specific version of a production package (for example, boto3-stubs for boto3). We use [uv](https://docs.astral.sh/uv/) to manage packages.

To add a package pinned to a major release, or to update a package to a new major release, run:

```bash
uv add logging-utilities~=5.0
```

Or directly modify the `pyproject.toml`:

```toml
dependencies = [
  "logging-utilities~=5.0"
]
```

Note the [uv version specifier](https://docs.astral.sh/uv/concepts/projects/dependencies/#dependency-specifiers) `~=` used here, which, in this case, pins the package version to  `>=5,<6`.

To update the packages to the latest minor/compatible versions, run:

```bash
uv lock --upgrade
```

See [uv upgrading locked packages](https://docs.astral.sh/uv/concepts/projects/sync/#upgrading-locked-package-versions)

To see what new major/incompatible releases are available, run:

```bash
uv pip list --outdated
```
