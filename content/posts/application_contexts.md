---
title: "Tired of Passing 10 Parameters to Your Functions: Go For The ApplicationContext Pattern for Data Pipelines"
date: 2026-01-12
draft: false
tags: ["python", "clean-code", "data-engineering", "patterns"]
summary: "Your pipeline functions don't need tons of parameters. Here's a simple pattern to clean up your Databricks and Snowflake code."
---

We've all written this code. A pipeline function that starts with 3 parameters, then grows to 5, then 8, then... you stop counting.

```python
def process_daily_sales(
    df: DataFrame,
    spark: SparkSession,
    logger: Logger,
    config: dict,
    output_path: str,
    env: str
):
    ...
```

Every function in the pipeline drags along the same luggage. Need logging? Pass the logger. Need config? Pass the config. It spreads like a virus through your codebase.

There's a cleaner way.

## The ApplicationContext Pattern

Instead of passing individual dependencies everywhere, bundle them into a single context object.

```python
from pydantic_settings import BaseSettings
from functools import cached_property
import structlog

class PipelineConfig(BaseSettings):
    """Configuration loaded from environment variables."""
    env: str = "dev"
    output_path: str = "/mnt/delta/output"
    log_level: str = "INFO"
    
    model_config = {"env_prefix": "PIPELINE_"}


class PipelineContext:
    """Runtime context for pipeline execution."""
    
    def __init__(self, config: PipelineConfig, spark):
        self.config = config
        self.spark = spark
    
    @cached_property
    def logger(self):
        return structlog.get_logger().bind(env=self.config.env)
```

The key idea: **one object carries everything your pipeline needs**. Config, logger, Spark session — all accessible from a single `ctx` parameter.

## Using the Context

Your pipeline code becomes dramatically cleaner:

```python
def run_pipeline(ctx: PipelineContext):
    ctx.logger.info("Starting pipeline")
    
    df = ctx.spark.read.format("delta").load("/mnt/delta/sales")
    df_filtered = df.filter(df.amount > 100)
    
    df_filtered.write.format("delta").mode("overwrite").save(ctx.config.output_path)
    
    ctx.logger.info("Pipeline complete", rows=df_filtered.count())
```

Every function takes the data it transforms plus the context. That's it.

## The Databricks Entry Point

In your Databricks notebook:

```python
config = PipelineConfig()  # Loads from environment
ctx = PipelineContext(config, spark)  # spark is provided by Databricks

run_pipeline(ctx)
```

Clean, readable, and all dependencies are explicit.

## Testing Becomes Trivial

This is where the pattern really pays off. Create a test context with mocked dependencies:

```python
# conftest.py
import pytest

@pytest.fixture
def test_ctx(spark_session):
    config = PipelineConfig(env="test", output_path="/tmp/test_output")
    return PipelineContext(config, spark_session)
```

Now your tests are clean:

```python
def test_pipeline_filters_correctly(test_ctx):
    # Create test data
    df = test_ctx.spark.createDataFrame([
        {"id": 1, "amount": 150},
        {"id": 2, "amount": 50}
    ])
    
    result = df.filter(df.amount > 100)
    
    assert result.count() == 1
```

No complex setup. No dependency injection framework. Just pass the test context.

## "Isn't This a God Object?"

Fair question. The answer: it depends on what you put in it.

The context should contain **infrastructure concerns** — connections, loggers, configuration. Things that are plumbing, not business logic.

It should **not** contain business state, data caches, or anything that changes during pipeline execution. Keep it boring and predictable.

If your context starts growing methods like `calculate_revenue()` or `validate_order()`, you've crossed the line. Those belong in your domain code, not the context.

## Why Not Use a DI Framework?

You could. Libraries like `dependency-injector` exist and work fine.

But for data pipelines, the ceremony usually isn't worth it. You typically have one execution path per job, not the complex dependency graphs that benefit from a full DI container.

The explicit context object is readable, debuggable, and doesn't require your team to learn a framework. When a new engineer reads your code, they see exactly what's happening.

## The Takeaway

Next time you're about to add a sixth parameter to a function, stop. Bundle your infrastructure dependencies into a context object.

Your functions become easier to read. Your tests become easier to write. And your future self won't curse you during the next refactoring session.

The pattern isn't revolutionary — it's just discipline. But sometimes discipline is exactly what messy pipeline code needs.
