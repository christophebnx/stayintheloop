---
title: "Stop Using Dicts for Config — Use Pydantic Instead"
date: 2026-01-08
summary: "Load your YAML/TOML config files into validated, typed Python objects. No more silent typos, no more runtime surprises."
tags: ["python", "pydantic", "config", "data-engineering"]
---

You've seen this code. Maybe you've written it:

```python
config = yaml.safe_load(open("config.yaml"))
db_host = config["database"]["hsot"]  # typo → KeyError material
```

Or worse, the typo doesn't crash — it just returns `None` and propagates silently through your pipeline.

Let's fix this properly.

## The goal

```
config.yaml  →  yaml.safe_load()  →  Pydantic model  →  typed, validated object
```

One loading function, full validation, autocomplete in your IDE, explicit errors when something's wrong.

---

## Step 1: Basic structure with nested models

**config.yaml**
```yaml
database:
  host: localhost
  port: 5432
  name: myapp

api:
  timeout: 30
  retries: 3
```

**config.py**
```python
from pathlib import Path
from pydantic import BaseModel
import yaml


class DatabaseConfig(BaseModel):
    host: str
    port: int
    name: str


class ApiConfig(BaseModel):
    timeout: int
    retries: int


class AppConfig(BaseModel):
    database: DatabaseConfig
    api: ApiConfig


def load_config(path: Path) -> AppConfig:
    with open(path) as f:
        raw = yaml.safe_load(f)
    return AppConfig.model_validate(raw)
```

**Usage:**
```python
config = load_config(Path("config.yaml"))
print(config.database.port)  # 5432 — typed as int, validated
print(config.database.hsot)  # AttributeError immediately, not KeyError in prod
```

Already better. But we can go further.

---

## Step 2: Field constraints

Pydantic's `Field()` lets you add validation rules:

```python
from pydantic import BaseModel, Field


class DatabaseConfig(BaseModel):
    host: str
    port: int = Field(ge=1, le=65535)  # valid port range
    name: str = Field(min_length=1)    # no empty strings
    pool_size: int = Field(default=5, ge=1, le=100)


class ApiConfig(BaseModel):
    timeout: int = Field(ge=1, le=300)
    retries: int = Field(ge=0, le=10)
    base_url: str = Field(pattern=r"^https?://")  # must be a URL
```

Now if someone puts `port: 99999` in the YAML, it fails at load time with a clear message — not somewhere deep in your connection logic.

### Common Field constraints

| Constraint | Works on | Example |
|------------|----------|---------|
| `ge`, `le`, `gt`, `lt` | numbers | `Field(ge=0, le=100)` |
| `min_length`, `max_length` | strings, lists | `Field(min_length=1)` |
| `pattern` | strings | `Field(pattern=r"^\w+$")` |
| `default` | any | `Field(default=30)` |
| `default_factory` | mutable types | `Field(default_factory=list)` |

---

## Step 3: Model-level configuration with `model_config`

You can control how Pydantic behaves during validation:

```python
from pydantic import BaseModel, ConfigDict


class AppConfig(BaseModel):
    model_config = ConfigDict(
        strict=True,
        frozen=True,
        extra="forbid",
    )

    database: DatabaseConfig
    api: ApiConfig
```

### The options that matter for config files

**`strict=True`**

No type coercion. If your YAML has `port: "5432"` (string), it fails instead of silently converting to int.

```python
# strict=False (default): "5432" → 5432 ✓
# strict=True: "5432" → ValidationError ✗
```

I go back and forth on this one. Strict catches mistakes in your YAML, but YAML itself is loose with types. Your call.

**`frozen=True`**

Makes the config immutable after creation:

```python
config = load_config(Path("config.yaml"))
config.database.port = 9999  # ❌ ValidationError: instance is frozen
```

For config objects, this is almost always what you want. Config loads once at startup, nobody should mutate it after.

**`extra="forbid"`**

Unknown fields in the YAML raise an error:

```yaml
database:
  host: localhost
  port: 5432
  naem: myapp  # typo!
```

```python
# extra="ignore" (default): typo silently ignored
# extra="forbid": ValidationError: Extra inputs are not permitted
```

This one is non-negotiable for me. Typos in config files should explode loudly.

---

## Step 4: Custom validation with `@field_validator`

For logic that `Field()` can't express:

```python
from pydantic import BaseModel, field_validator


class DatabaseConfig(BaseModel):
    host: str
    port: int
    name: str

    @field_validator("host")
    @classmethod
    def host_not_empty(cls, v: str) -> str:
        if not v.strip():
            raise ValueError("host cannot be empty or whitespace")
        return v.strip()

    @field_validator("name")
    @classmethod
    def name_lowercase(cls, v: str) -> str:
        return v.lower()  # normalize to lowercase
```

Key points:
- Decorator is `@field_validator`, not `@validator` (v1 syntax)
- Must be a `@classmethod`
- Return the (possibly transformed) value
- Raise `ValueError` to fail validation

---

## Step 5: Cross-field validation with `@model_validator`

Single-field validators are great, but sometimes validity depends on *combinations* of fields. That's where `@model_validator` comes in.

### Basic pattern

```python
from pydantic import BaseModel, model_validator


class ApiConfig(BaseModel):
    timeout: int
    retries: int
    retry_delay: int

    @model_validator(mode="after")
    def check_retry_timing(self) -> "ApiConfig":
        worst_case = self.timeout + (self.retry_delay * self.retries)
        if worst_case > 300:
            raise ValueError(
                f"worst-case duration is {worst_case}s, max allowed is 300s — "
                f"reduce retries or delay"
            )
        return self
```

`mode="after"` means the validator runs after individual field validation — all fields are already populated and typed.

### Mutually exclusive fields

A common pattern: multiple ways to configure something, but only one should be used.

```python
class AuthConfig(BaseModel):
    api_key: str | None = None
    oauth_token: str | None = None
    username: str | None = None
    password: str | None = None

    @model_validator(mode="after")
    def exactly_one_auth_method(self) -> "AuthConfig":
        methods = []
        if self.api_key:
            methods.append("api_key")
        if self.oauth_token:
            methods.append("oauth")
        if self.username or self.password:
            methods.append("basic_auth")

        if len(methods) == 0:
            raise ValueError("no auth method configured — need api_key, oauth, or basic auth")
        if len(methods) > 1:
            raise ValueError(f"multiple auth methods configured: {methods} — pick one")

        if "basic_auth" in methods and not (self.username and self.password):
            raise ValueError("basic auth requires both username and password")

        return self
```

No more ambiguous configs where someone sets both `api_key` and `oauth_token` and wonders which one gets used.

### Conditional validation based on environment

Different rules for dev vs prod:

```python
class AppConfig(BaseModel):
    env: str  # "dev", "staging", "prod"
    database: DatabaseConfig
    debug: bool = False

    @model_validator(mode="after")
    def prod_safety_checks(self) -> "AppConfig":
        if self.env == "prod":
            if self.debug:
                raise ValueError("debug=true not allowed in prod")
            if "localhost" in self.database.host:
                raise ValueError("localhost database not allowed in prod")
        return self
```

Catch the mistake at config load time, not when your pipeline hits production.

### Logical dependencies

When one field being set implies constraints on another:

```python
class RetryConfig(BaseModel):
    enabled: bool
    max_retries: int
    delay_seconds: int

    @model_validator(mode="after")
    def check_retry_consistency(self) -> "RetryConfig":
        if self.enabled and self.max_retries == 0:
            raise ValueError("retries enabled but max_retries is 0 — pick one")
        if not self.enabled and self.max_retries > 0:
            raise ValueError("max_retries set but retries disabled — pick one")
        return self
```

### `mode="before"` vs `mode="after"`

| Mode | Runs when | Input type | Use case |
|------|-----------|------------|----------|
| `after` | After field validation | Model instance | Cross-field validation, business logic |
| `before` | Before field validation | Raw `dict` | Normalize/transform raw input |

Example with `mode="before"` — normalizing keys before validation:

```python
@model_validator(mode="before")
@classmethod
def normalize_keys(cls, data: dict) -> dict:
    # Accept "DB_HOST" or "db_host" or "db-host"
    return {k.lower().replace("-", "_"): v for k, v in data.items()}
```

Note: `mode="before"` requires `@classmethod` because there's no instance yet.

---

## Step 6: Computed fields

Need a value derived from other fields?

```python
from pydantic import BaseModel, computed_field


class DatabaseConfig(BaseModel):
    host: str
    port: int
    name: str
    user: str
    password: str

    @computed_field
    @property
    def connection_string(self) -> str:
        return f"postgresql://{self.user}:{self.password}@{self.host}:{self.port}/{self.name}"
```

`connection_string` isn't in the YAML — it's computed on access. Appears in `.model_dump()` output too.

---

## What validation errors look like

When something fails, Pydantic gives you useful errors:

```python
# config.yaml with port: 99999
try:
    config = load_config(Path("config.yaml"))
except ValidationError as e:
    print(e)
```

```
1 validation error for AppConfig
database.port
  Input should be less than or equal to 65535 [type=less_than_equal, input_value=99999, input=99999]
```

You get:
- The path to the field (`database.port`)
- What went wrong
- The actual value that failed

Much better than debugging a `ConnectionRefusedError` three functions deep.

---

## YAML vs TOML

Everything above works with TOML too — just swap the loader:

```python
import tomllib  # stdlib since 3.11

def load_config(path: Path) -> AppConfig:
    with open(path, "rb") as f:  # note: binary mode for tomllib
        raw = tomllib.load(f)
    return AppConfig.model_validate(raw)
```

TOML has stricter typing (integers stay integers), which pairs well with `strict=True`. YAML is more common in data/devops ecosystems. Pick based on your team.

---

## The complete pattern

```python
from pathlib import Path
from pydantic import BaseModel, ConfigDict, Field, field_validator, model_validator
import yaml


class DatabaseConfig(BaseModel):
    host: str
    port: int = Field(ge=1, le=65535)
    name: str = Field(min_length=1)
    pool_size: int = Field(default=5, ge=1, le=100)

    @field_validator("host")
    @classmethod
    def strip_host(cls, v: str) -> str:
        stripped = v.strip()
        if not stripped:
            raise ValueError("host cannot be empty")
        return stripped


class ApiConfig(BaseModel):
    timeout: int = Field(ge=1, le=300)
    retries: int = Field(ge=0, le=10)
    retry_delay: int = Field(default=5, ge=1, le=60)

    @model_validator(mode="after")
    def check_retry_timing(self) -> "ApiConfig":
        if self.retries > 0:
            total_retry_time = self.retries * self.retry_delay
            if total_retry_time > self.timeout:
                raise ValueError(
                    f"retry strategy exceeds timeout: "
                    f"{self.retries} retries × {self.retry_delay}s = {total_retry_time}s > {self.timeout}s timeout"
                )
        return self


class AppConfig(BaseModel):
    model_config = ConfigDict(
        frozen=True,
        extra="forbid",
    )

    env: str = Field(pattern=r"^(dev|staging|prod)$")
    database: DatabaseConfig
    api: ApiConfig

    @model_validator(mode="after")
    def prod_safety_checks(self) -> "AppConfig":
        if self.env == "prod":
            if "localhost" in self.database.host:
                raise ValueError("localhost database not allowed in prod")
        return self


def load_config(path: Path) -> AppConfig:
    with open(path) as f:
        raw = yaml.safe_load(f)
    return AppConfig.model_validate(raw)
```

**config.yaml**
```yaml
env: prod

database:
  host: db.example.com
  port: 5432
  name: myapp
  pool_size: 10

api:
  timeout: 60
  retries: 3
  retry_delay: 10
```

Validated. Typed. Immutable. Cross-checked. No more silent config bugs.

---

## Quick reference

| Feature | Syntax |
|---------|--------|
| Nested config | Create separate `BaseModel` classes |
| Field constraints | `Field(ge=1, le=100, min_length=1)` |
| Model behavior | `model_config = ConfigDict(...)` |
| Single field validation | `@field_validator("field_name")` |
| Multi-field validation | `@model_validator(mode="after")` |
| Computed values | `@computed_field` + `@property` |
| Load from dict | `AppConfig.model_validate(raw)` |
| Export to dict | `config.model_dump()` |