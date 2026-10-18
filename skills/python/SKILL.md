---
name: python
description: Python coding style for packages, libraries and services. Covers formatting, imports, naming, typing, data classes, docstrings, errors, logging and file handling. Use whenever writing, editing or reviewing `.py` files or Python code snippets. It is a style guide only; for project setup, dependencies, tooling, testing, CI, pydantic and databases use the `python-tools` skill.
---

# Python style

How Python code should look and read. It applies to any Python code an agent
writes or edits. The consuming project's own configuration (`ruff.toml`,
`pyproject.toml`, `CONTRIBUTING.md`, `AGENTS.md`) takes precedence. When
editing existing code, match the surrounding code first and use this skill
for whatever the project leaves undefined.

Target Python 3.13 or later, and use modern syntax freely.

## Formatting

- Format with `ruff format` and lint with `ruff check`. If the project has no
  ruff configuration, use the one in the
  [`python-tools`](../python-tools/SKILL.md#ruff) skill, which enforces this
  style.
- Keep lines to 80 columns, indent with 4 spaces and use **single quotes**.
  Use double quotes only to avoid escaping (`"it's"`) and for docstrings.
- Wrap long expressions inside parentheses, never with backslashes. Break
  long strings with implicit concatenation:

  ```python
  self.logger.warning(
      f'export of {job_id} still running after {timeout}s, '
      f'{len(pending)} record(s) not written yet'
  )
  ```

- When a call or signature doesn't fit on one line, put one argument per
  line and end with a trailing comma, so the formatter keeps it exploded.

### Vertical spacing

Use blank lines to separate logical steps, and let short related statements
stay together. In particular:

- Put a blank line before `return`, `continue`, `break` and `raise` when they
  follow other statements in the same block.
- Put a blank line after an indented block (`if`, `for`, `with`, `try`)
  before the next statement.
- Use two blank lines after the imports and between top-level definitions.

```python
def load_record(path: pathlib.Path) -> dict | None:
    if not path.exists():
        logger.warning(f'{path.name}: not found, skipped')

        return None

    with path.open(encoding='utf-8') as f:
        record = json.load(f)

    record['source'] = str(path)

    return record
```

## Imports

- Put imports at the top of the module, in three groups separated by a blank
  line: standard library, third-party packages, then your own package.
- Import standard library modules whole (`import pathlib`, `import json`)
  and use them qualified (`pathlib.Path`). This keeps the origin of every
  name visible.
- Import one name per `from` line. One-name lines make diffs and merges
  cleaner. Within a group, put plain `import` lines first, then a blank line,
  then the `from` lines:

  ```python
  import json
  import pathlib

  from collections.abc import Callable
  from typing import Self

  import numpy as np

  from my_package.components import Record
  from my_package.components import Store
  ```

- Use absolute imports. Never use `from x import *`.
- Import heavy or optional dependencies lazily, inside the function that
  needs them, only when they would otherwise slow down or break importing the
  package.

## Naming and module layout

- Follow PEP 8: `snake_case` for functions, variables and modules,
  `CamelCase` for classes, `UPPER_CASE` for constants.
- Prefix internal names with an underscore: `_helper()`, `self._timeout`,
  `_DEFAULTS`. Everything without an underscore is public API.
- In packages, put implementations in private modules (`_store.py`) and
  re-export the public names from `__init__.py` with an explicit `__all__`:

  ```python
  # noqa: D104
  from my_package.storage._record import Record
  from my_package.storage._store import Store


  __all__ = [
      'Record',
      'Store',
  ]
  ```

- Define constants at module level, after the imports, and class-level
  constants as `_UPPER_CASE` class attributes. Don't scatter magic numbers or
  strings through the code.
- Use descriptive names. Single letters are fine only for short loop
  indexes and comprehensions. Name caught exceptions `error` (or `e` in a
  one-line handler).

## Typing

- Annotate the parameters and return types of every public function and
  method. Annotate attributes in `__init__` when their type isn't obvious
  from the value, in particular when they start as `None`.
- Use built-in generics and union syntax: `list[str]`, `dict[str, int]`,
  `str | None`. Never use `List`, `Dict` or `Optional`.
- Import abstract types from `collections.abc` (`Callable`, `Iterable`,
  `Mapping`) and use `typing.Self` for methods that return their instance.
- Use PEP 695 syntax for generics and aliases instead of `TypeVar`:

  ```python
  type Handler = Callable[[Record], None]


  class Repository[T]:
      def add(self, key: str, item: T) -> None: ...


  def first[T](items: list[T]) -> T | None:
      return items[0] if items else None
  ```

- Use `typing.Protocol` to describe an interface that callers can implement
  without inheriting from your classes.

## Data and classes

- Use frozen dataclasses for internal value objects, and add `slots=True`
  when you don't need inheritance tricks. For data that crosses a boundary
  (requests, messages, files, settings), use pydantic models instead (see
  [`python-tools`](../python-tools/SKILL.md#data-models-and-settings-pydantic)):

  ```python
  @dataclass(frozen=True, slots=True)
  class Record:
      name: str
      size: int = 0
      tags: tuple[str, ...] = field(default_factory=tuple)
  ```

- Use `enum.StrEnum` with `@unique` for fixed sets of states or options,
  never bare string constants.
- Write empty containers as `dict()`, `list()` and `set()`; use literals
  only when they contain values.
- Never use mutable default arguments. Use `None` or
  `field(default_factory=...)`.
- Copy before you modify data you didn't create: `dict(payload)`,
  `list(items)`, `copy.deepcopy(config)`.
- Keep `__init__` cheap: store arguments and set up state, but don't open
  files, sockets or threads there. Do that in an explicit method
  (`start()`, `open()`, `connect()`) or a classmethod constructor.
- Store constructor arguments as private attributes and expose them through
  read-only `@property` methods when callers need them.
- When subclassing, call `super().__init__()` and, for framework hooks such
  as `start()` and `stop()`, call the `super()` method too.
- Prefer functions to classes when there is no state to hold. Use
  `@staticmethod` sparingly, and `@classmethod` for alternative constructors
  (`from_dict`, `from_file`).

## Control flow

- Return early on invalid input or nothing-to-do cases instead of nesting the
  main path inside `if` blocks.
- Use `match` for dispatching on structure or several literal values, and a
  dict of callables for simple lookup tables.
- Use comprehensions for simple transformations; switch to a loop when the
  comprehension needs nested conditions or side effects.
- Use the walrus operator only when it removes a duplicated call.
- Never use `assert` for runtime checks in library code, because `python -O`
  removes it. Raise an exception instead.

## Docstrings and comments

Use NumPy-style docstrings. For a multi-line docstring, put the summary on
the line after the opening quotes and a blank line before the closing quotes.
Document constructor parameters in `__init__`, and describe what the class is
in the class docstring.

```python
def fetch(self, name: str, timeout: float = 2.0) -> Record:
    """
    Fetch a record from the store.

    Parameters
    ----------
    name : str
        Name of the record.
    timeout : float, optional
        Seconds to wait for the store. The default is 2.0.

    Returns
    -------
    Record
        The stored record.

    Raises
    ------
    RecordNotFoundError
        If no record has the given name.

    """
```

- Give every module a docstring that states its purpose. Short private
  helpers can have a one-line docstring, or none when the name says it all.
- Write comments to explain *why*, not *what*. Delete commented-out code.
- Mark known debt with `TODO:` or `HACK:` and a short reason.

## Errors

- Raise the most specific built-in exception that fits (`ValueError`,
  `TypeError`, `KeyError`, `FileNotFoundError`), or a custom one.
- Give a package its own exception hierarchy, and derive the base from the
  built-in that matches, so that existing `except` clauses keep working:

  ```python
  class StoreError(RuntimeError):
      """Base exception for store failures."""


  class RecordNotFoundError(StoreError):
      """Raised when a record does not exist."""
  ```

- Chain exceptions when translating them: `raise StoreError(...) from error`.
- Catch only what you can handle. Never write a bare `except:`. Catch
  `Exception` only at boundaries that must survive (a worker loop, a request
  handler, a plugin call), and always log what was caught there.
- Write exception and log messages in lowercase, without a trailing period,
  and include the offending value: `f'unknown format: {fmt}'`.

## Logging

- Get one logger per module or component (`logging.getLogger(__name__)`, or
  the project's logger helper). Never configure handlers in library code.
- Format messages with f-strings, and prefix them with the context they
  belong to (a request, job or record id):

  ```python
  logger.info(f'{job_id}: export completed')
  logger.error(f'{job_id}: invalid config, ignored: {error}')
  ```

- Use `logger.exception(...)` inside an `except` block when the traceback
  matters.
- Never use `print()` in library or service code. It is only for CLI output.

## Files, processes and I/O

- Use `pathlib` for every path operation, never `os.path`. Build paths with
  `pathlib.Path(base, name)`.
- Open files with `path.open()` in a context manager, with an explicit
  `encoding='utf-8'` for text. Use `Path.read_text()` and
  `Path.write_text()` for whole-file reads and writes.
- Write outputs that other processes may read to a temporary name first,
  then `rename()` them into place, so readers never see a partial file.
- Run commands with `subprocess.run([...], check=True)` and an argument list.
  Never use `shell=True` with values you didn't write.
- Put a timeout on every blocking call: network requests, `join()`, queue
  `get()` and `put()`, subprocesses.
- Read tunables (timeouts, sizes, URLs) from one constants or settings
  module, which can apply environment overrides. Never hardcode credentials.
