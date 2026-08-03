# Python coding guidelines

There are a number of style guides out there. If in doubt, check what [google](http://google.github.io/styleguide/pyguide.html#doc-function-raises) proposes. We largely follow their guidelines. A few important things are reproduced here.

- [1. Linting / Auto-formatting](#1-linting--auto-formatting)
  - [Linting configuration](#linting-configuration)
  - [Formatting configuration](#formatting-configuration)
    - [Import sorting](#import-sorting)
- [2. Type hints](#2-type-hints)
- [3. Naming conventions](#3-naming-conventions)
- [4. Comments and Docstrings](#4-comments-and-docstrings)
  - [Docstrings](#docstrings)
  - [Modules](#modules)
  - [Functions and Methods](#functions-and-methods)
  - [Classes](#classes)
  - [Block and Inline Comments](#block-and-inline-comments)
  - [Punctuation, Spelling, and Grammar](#punctuation-spelling-and-grammar)
- [5. TODO Comments](#5-todo-comments)
- [6. Exceptions](#6-exceptions)
  - [Definition](#definition)
  - [Pros](#pros)
  - [Cons](#cons)
  - [Decision](#decision)
- [7. Error handling - Rules of Thumb](#7-error-handling---rules-of-thumb)
- [8. Introduce Explaining Variable](#8-introduce-explaining-variable)
- [9. Unit Testing frameworks](#9-unit-testing-frameworks)
- [10. Observablity - Logging](#10-observablity---logging)
  - [Logger](#logger)
  - [OTEL LoggerProvider](#otel-loggerprovider)
  - [Configuration](#configuration)
- [11. Observability - Tracing](#11-observability---tracing)
- [12. Observability - Metrics](#12-observability---metrics)
- [13. Observability - local stack](#13-observability---local-stack)
- [14. Dependency Management](#14-dependency-management)

**The foremost goal is that reading and understanding your python code is easy for someone else (or yourself in a few months time).**

## 1. Linting / Auto-formatting

We use [ruff](https://docs.astral.sh/ruff/) for linting (including security linting with bandit rules) and code formatting.

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

## 2. Type hints

Python code must use type hints for function arguments and return values. Type hints help code reader to understand the code and can catch type errors before runtime.

Type hints are checked and enforced by `ty`, see [ty](https://docs.astral.sh/ty/).

## 3. Naming conventions

Python code must follow these naming conventions:

- module: snake_case
- constant: UPPER_CASE
- variable: snake_case
- function/method: snake_case
- argument: snake_case
- class: PascalCase
- attribute: snake_case

These naming conventions are checked by `ruff`.

## 4. Comments and Docstrings

Be sure to use the right style for module, function, method docstrings and inline comments.

### Docstrings

Python uses docstrings to document code. A docstring is a string that is the first statement in a package, module, class or function. These strings can be extracted automatically through the __doc__ member of the object and are used by pydoc. (Try running pydoc on your module to see how it looks.) Always use the three double-quote `"""` format for docstrings (per PEP 257). A docstring should be organized as a summary line (one physical line) terminated by a period, question mark, or exclamation point, followed by a blank line, followed by the rest of the docstring starting at the same cursor position as the first quote of the first line. There are more formatting guidelines for docstrings below.

### Modules

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

## 5. TODO Comments

Use `TODO` comments for code that is temporary, a short-term solution, or good-enough but not perfect.

A `TODO` comment begins with the string `TODO` in all caps and the abbreviation (e.g. boc) or other identifier of the person or issue with the best context about the problem. This is followed by an explanation of what there is to do.

The purpose is to have a consistent `TODO` format that can be searched to find out how to get more details. A `TODO` is not a commitment that the person referenced will fix the problem. Thus when you create a `TODO`, it is almost always your name that is given.

```python
# TODO(kl@gmail.com): Use a "*" here for string repetition.
# TODO(boc) Change this to use relations.
```

If your `TODO` is of the form "At a future date do something" make sure that you either include a very specific date ("Fix by November 2009") or a very specific event ("Remove this code when all clients can handle XML responses.").

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

Python comes with a fairly mature unit testing framework [`unittest`](https://docs.python.org/3/library/unittest.html). However the standard `unittest` framework is limited in features.

That's why every project should use [`pytest`](https://docs.pytest.org/en/latest/) framework to write and run tests.

Pytest comes with the following features that should be used:

  - Fixtures: reusable setup/teardown code for tests (e.g. for mocking resources)
  - Markers: categorize tests by feature or priority
  - Parametrization: run the same test with different inputs

## 10. Observability - Logging

Python comes with a good logging framework that we should use for logging. For development all log messages should be printed to stdout in a human-readable format.

For deployment all log messages should be sent to the Opentelementry collector directly using the [`opentelemetry-python`](https://github.com/open-telemetry/opentelemetry-python) libraries.

### Logger

We should always use a logger per module named after it as follow:

```python
# my_module.py

import logging

logger = logging.getLogger(__name__)

logger.info('This is my module logger')
```

**NOTE:** don't reuse the Flask app logger.

### OTEL LoggerProvider

Together with the standard Python logging sending human readable message to stdout, we must use the 
OTEL `LoggerProvider` to send logs directly to the OTEL collector.

```python
import logging
from opentelemetry.sdk._logs import LoggerProvider, LoggingHandler
from opentelemetry.sdk._logs.export import BatchLogRecordProcessor, ConsoleLogRecordExporter # ConsoleLogExporter on versions earlier than 1.39.0
from opentelemetry._logs import set_logger_provider

provider = LoggerProvider()
processor = BatchLogRecordProcessor(ConsoleLogRecordExporter())
provider.add_log_record_processor(processor)
# Sets the global default logger provider
set_logger_provider(provider)

handler = LoggingHandler(level=logging.INFO, logger_provider=provider)
logging.basicConfig(handlers=[handler], level=logging.INFO)

logging.getLogger(__name__).info("This is an OpenTelemetry log record!")
```

See [OTEL Python Instrumentation Logs](https://opentelemetry.io/docs/languages/python/instrumentation/#logs)

### Configuration

Logging should be configured via a yaml file for local development as follow:

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

Then on the deployment it could use the same configuration to send human readable logs on stdout.

> [!IMPORTANT]
> On deployment stdout is ignored by the Observability stack, and each python service MUST use the OTEL Logger provider, see above [OTEL LoggerProvider](#otel-loggerprovider).

## 11. Observability - Tracing

Every service should implement OTEL Tracing, see [Opentelemetry Tracing](https://opentelemetry.io/docs/languages/python/instrumentation/#traces).

## 12. Observability - Metrics

OpenTelemetry instrumentation usually already implements most important metrics. However if the service
requires other metrics, they should be implemented with [OpenTelemetry Metrics](https://opentelemetry.io/docs/languages/python/instrumentation/#metrics). Any new metrics must follow the OTEL [semantic convention](https://opentelemetry.io/docs/specs/semconv/).

## 13. Observability - local stack

For local development, every service should use the following docker containers:

- otel/opentelemetry-collector-contrib => OTEL collector
- jaegertracing/all-in-one => trace analyzer
- prom/prometheus => metrics receiver and analyzer

## 14. Dependency Management

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
