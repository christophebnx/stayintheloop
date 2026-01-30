---
title: "The `with` Statement Is More Powerful Than You Think"
date: 2026-01-29
summary: "You use `with open()` every day. Here's how to build your own context managers — and why they'll save you from leaked connections and forgotten cleanup."
tags: ["python", "patterns", "data-engineering"]
---

You've written this pattern a hundred times:

```python
conn = get_connection()
try:
    result = conn.execute(query)
finally:
    conn.close()
```

It works. But scatter it across 50 functions and eventually someone forgets the `finally`. Or adds an early `return` before the cleanup. Or catches an exception and forgets to re-raise.

Then you're debugging connection leaks at 2am wondering why your pool is exhausted.

There's a better way. And you already know half of it.

## The pattern you already use

```python
with open("data.csv") as f:
    content = f.read()
# file is closed here, guaranteed
```

The `with` statement guarantees cleanup happens — even if an exception fires, even if you return early. No discipline required.

The good news: you can build your own.

## Creating a context manager in 5 lines

```python
from contextlib import contextmanager

@contextmanager
def get_db_connection(host: str, port: int):
    conn = psycopg2.connect(host=host, port=port)
    try:
        yield conn
    finally:
        conn.close()
```

That's it. The `yield` separates setup from cleanup:

1. Everything before `yield` runs when entering the `with` block
2. The yielded value becomes the `as` variable
3. Everything after `yield` (in `finally`) runs when exiting — always

Usage:

```python
with get_db_connection("localhost", 5432) as conn:
    result = conn.execute("SELECT * FROM users")
# conn.close() called automatically
```

## Patterns that save time in data engineering

### Timer for any operation

```python
import time
from contextlib import contextmanager

@contextmanager
def timed(operation: str):
    start = time.perf_counter()
    yield
    elapsed = time.perf_counter() - start
    print(f"{operation} took {elapsed:.2f}s")
```

```python
with timed("load_parquet"):
    df = pd.read_parquet("huge_file.parquet")
# prints: load_parquet took 4.23s
```

No return value needed — sometimes you just want setup/teardown around a block.

### Temporary file that cleans itself

```python
from pathlib import Path
from uuid import uuid4
from contextlib import contextmanager

@contextmanager
def temp_parquet(df: pd.DataFrame):
    path = Path(f"/tmp/{uuid4()}.parquet")
    df.to_parquet(path)
    try:
        yield path
    finally:
        path.unlink(missing_ok=True)
```

```python
with temp_parquet(my_dataframe) as path:
    # path exists and contains the data
    upload_to_s3(path)
# file is deleted, even if upload_to_s3 crashes
```

### Database transaction

```python
@contextmanager
def transaction(conn):
    try:
        yield conn
        conn.commit()
    except Exception:
        conn.rollback()
        raise
```

```python
with get_db_connection("localhost", 5432) as conn:
    with transaction(conn):
        conn.execute("INSERT INTO users ...")
        conn.execute("INSERT INTO audit_log ...")
        # both commit, or both rollback
```

### Temporary working directory

```python
import os
from pathlib import Path
from contextlib import contextmanager

@contextmanager
def working_directory(path: Path):
    original = Path.cwd()
    os.chdir(path)
    try:
        yield
    finally:
        os.chdir(original)
```

```python
with working_directory(Path("/data/project")):
    # all relative paths resolve from /data/project
    process_files()
# back to original directory
```

### Spark session lifecycle

```python
@contextmanager
def spark_session(app_name: str):
    spark = SparkSession.builder.appName(app_name).getOrCreate()
    try:
        yield spark
    finally:
        spark.stop()
```

```python
with spark_session("daily_etl") as spark:
    df = spark.read.parquet("s3://bucket/data/")
    # ... process ...
# spark.stop() called automatically
```

## Returning something vs returning nothing

Two patterns:

```python
# Returns a resource — use "as"
with get_db_connection(...) as conn:
    conn.execute(...)

# Just wraps a block — no "as" needed
with timed("operation"):
    do_stuff()
```

Both are valid. The `yield` can yield a value or yield nothing.

## Stacking multiple context managers

Python 3.10+ lets you stack them cleanly:

```python
with (
    get_db_connection(config.database) as conn,
    timed("full_pipeline"),
    temp_directory() as tmpdir,
):
    # all three are set up
    # all three will clean up in reverse order
    ...
```

Before 3.10, use `contextlib.ExitStack` for dynamic stacking, or just nest them:

```python
with get_db_connection(config.database) as conn:
    with timed("full_pipeline"):
        ...
```

## When to use `@contextmanager` vs a class

You can also write context managers as classes with `__enter__` and `__exit__`:

```python
class DBConnection:
    def __init__(self, host: str, port: int):
        self.host = host
        self.port = port
        self.conn = None

    def __enter__(self):
        self.conn = psycopg2.connect(host=self.host, port=self.port)
        return self.conn

    def __exit__(self, exc_type, exc_val, exc_tb):
        self.conn.close()
        return False  # don't suppress exceptions
```

When to use which:

| Approach | When |
|----------|------|
| `@contextmanager` | 90% of cases — simple, readable, less boilerplate |
| Class with `__enter__`/`__exit__` | Need to store state, reuse the manager, or customize exception handling |

The class approach gives you access to exception info in `__exit__`, which matters if you want to handle errors differently. For most cleanup scenarios, `@contextmanager` wins.

## Context managers from the stdlib worth knowing

| Module | Context manager | What it does |
|--------|-----------------|--------------|
| `contextlib` | `suppress(ValueError)` | Silently ignore specific exceptions |
| `contextlib` | `redirect_stdout(f)` | Capture stdout to a file |
| `contextlib` | `nullcontext(value)` | No-op context manager (useful for optional wrapping) |
| `tempfile` | `TemporaryDirectory()` | Directory that deletes itself |
| `tempfile` | `NamedTemporaryFile()` | File that deletes itself |
| `threading` | `Lock()` | Mutex for thread safety |

Example with `suppress`:

```python
from contextlib import suppress

with suppress(FileNotFoundError):
    os.remove("maybe_exists.txt")
# no try/except needed, no crash if file doesn't exist
```

## The rule of thumb

If you're writing `try/finally` to clean up a resource, stop. Write a context manager instead.

Your future self — debugging at 3am — will thank you.

---

## Quick reference

```python
from contextlib import contextmanager

@contextmanager
def my_context():
    # setup
    resource = acquire_something()
    try:
        yield resource  # or just `yield` if nothing to return
    finally:
        # cleanup — always runs
        resource.release()
```

```python
with my_context() as resource:
    # use resource
# cleanup done
```