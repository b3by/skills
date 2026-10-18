# Pydantic

Pydantic v2 models for data crossing a boundary, and `pydantic-settings` for
configuration.

```bash
uv add 'pydantic>=2.14,<3'
uv add 'pydantic-settings>=2.16,<3'   # only if the project reads settings
```

## Models

```python
from typing import Annotated

from pydantic import BaseModel
from pydantic import ConfigDict
from pydantic import Field
from pydantic import field_validator


class RecordIn(BaseModel):
    """
    A record as received from clients.

    Attributes
    ----------
    name : str
        Unique record name, stored lowercase.
    size : int
        Size in bytes.
    tags : tuple[str, ...]
        Free-form labels.

    """

    model_config = ConfigDict(frozen=True, extra='forbid')

    name: Annotated[str, Field(min_length=1, max_length=64)]
    size: Annotated[int, Field(ge=0)] = 0
    tags: tuple[str, ...] = tuple()

    @field_validator('name')
    @classmethod
    def _normalise_name(cls, value: str) -> str:
        return value.strip().lower()
```

- Document fields in an `Attributes` section of the class docstring. For
  models that are part of a public API schema, also set
  `Field(description=...)`, which ends up in the generated JSON schema.
- Use `frozen=True` and `extra='forbid'` by default. Relax `extra` only for
  input you deliberately accept partially, such as third-party webhooks
  (`extra='ignore'`).
- Prefer precise types to validators: `PositiveInt`, `HttpUrl`, `EmailStr`
  (needs `pydantic[email]`), `Literal['a', 'b']`, enums, `datetime` with a
  timezone (`AwareDatetime`).
- Use `@field_validator` for one field and `@model_validator(mode='after')`
  for rules across fields. Validators raise `ValueError` with a lowercase
  message.
- Keep separate models for input, output and storage (`RecordIn`,
  `RecordOut`, the database row) instead of one model with optional fields.
  Build an output model from an ORM object with
  `ConfigDict(from_attributes=True)` and `RecordOut.model_validate(row)`.

## Parsing and serialising

```python
record = RecordIn.model_validate(data)      # from a dict
record = RecordIn.model_validate_json(raw)  # from JSON text or bytes
payload = record.model_dump(mode='json')    # to a JSON-safe dict
text = record.model_dump_json()             # to JSON text
```

- To change a frozen model, validate a new one:
  `RecordIn.model_validate(record.model_dump() | {'size': 4})`.
  `model_copy(update=...)` skips validation, so use it only with values you
  control.

- Validate at the boundary, once, and pass model instances inward. Inner code
  shouldn't re-validate or handle raw dicts.
- Catch `pydantic.ValidationError` at the boundary and turn it into the
  boundary's error (an HTTP 422, a CLI message, a rejected message), with
  `error.errors()` for the details.
- Never use `model_construct()` on untrusted data: it skips validation.
- Use `TypeAdapter` to validate types that aren't models, such as
  `TypeAdapter(list[RecordIn]).validate_json(raw)`.

## Settings

```python
import functools

from pydantic import SecretStr
from pydantic_settings import BaseSettings
from pydantic_settings import SettingsConfigDict


class Settings(BaseSettings):
    """
    Service settings.

    Every field is read from an environment variable with the ``MY_APP_``
    prefix, for example ``MY_APP_DATABASE_URL``.

    """

    model_config = SettingsConfigDict(
        env_prefix='MY_APP_',
        env_file='.env',
        extra='ignore',
    )

    database_url: SecretStr
    api_token: SecretStr
    request_timeout: float = 10.0


@functools.cache
def get_settings() -> Settings:
    """Return the process-wide settings, loaded once."""
    return Settings()
```

- Load settings once, at startup, through a cached function, and pass the
  values (or the settings object) to the code that needs them. Don't read
  `os.environ` across the code base.
- Fields without a default are required, so a missing variable fails at
  startup with a clear error.
- Store credentials, tokens and URLs with passwords as `SecretStr`, and call
  `get_secret_value()` only where the value is used. Never log settings.
- Keep `.env` out of version control, and commit a `.env.example` with
  placeholder values.
