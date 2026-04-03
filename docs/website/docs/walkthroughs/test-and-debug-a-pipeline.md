---
title: Test and debug a pipeline
description: How to test dlt pipelines locally and debug common issues
keywords: [how to, test, debug, pipeline, troubleshooting, duckdb, load info, trace]
---

# Test and debug a pipeline

This guide shows you how to test dlt pipelines during development and debug issues when they arise. You will learn to use DuckDB as a local test destination, inspect load results, read pipeline traces, and handle common errors.

## Prerequisites

- A working [dlt installation](../reference/installation.md)
- A pipeline script (see [create a pipeline](create-a-pipeline.md) if you need one)

## 1. Test locally with DuckDB

DuckDB runs in-process and requires no credentials, making it the fastest way to test a pipeline. Set `dev_mode=True` so dlt creates a fresh pipeline state on every run, preventing leftover data from interfering with your tests.

```py
import dlt

pipeline = dlt.pipeline(
    pipeline_name="test_pipeline",
    destination="duckdb",
    dataset_name="test_data",
    dev_mode=True,
)

load_info = pipeline.run([{"id": 1, "name": "Alice"}, {"id": 2, "name": "Bob"}])
print(load_info)
```

The output shows which tables dlt created, how many rows it loaded, and whether any jobs failed.

:::tip
Use `dev_mode=True` only during development. It wipes pipeline state on every run, which means incremental loading state is lost between runs.
:::

## 2. Inspect load results

The `load_info` object returned by `pipeline.run()` tells you exactly what happened during loading. Check it after every run.

```py
import dlt

pipeline = dlt.pipeline(
    pipeline_name="test_pipeline",
    destination="duckdb",
    dataset_name="test_data",
    dev_mode=True,
)

load_info = pipeline.run(
    [{"id": 1, "name": "Alice"}, {"id": 2, "name": "Bob"}],
    table_name="people",
)

# check for failed jobs
if load_info.has_failed_jobs:
    for package in load_info.load_packages:
        for job in package.jobs.get("failed_jobs", []):
            print(f"Failed job: {job.job_file_info}")
            print(f"Error: {job.failed_message}")

# raise an exception if any jobs failed
load_info.raise_on_failed_jobs()

# print timing
print(f"Started: {load_info.started_at}")
print(f"Finished: {load_info.finished_at}")
```

## 3. Verify loaded data

Query the destination to confirm your data arrived correctly. The `pipeline.dataset()` method gives you direct access to loaded tables.

```py
import dlt

pipeline = dlt.pipeline(
    pipeline_name="test_pipeline",
    destination="duckdb",
    dataset_name="test_data",
    dev_mode=True,
)

load_info = pipeline.run(
    [{"id": 1, "name": "Alice"}, {"id": 2, "name": "Bob"}],
    table_name="people",
)

# get row counts for all tables
print(pipeline.dataset().row_counts())

# read all rows from a specific table
rows = pipeline.dataset().people.fetchall()
print(rows)
```

For more ways to access loaded data, see [access loaded data](../general-usage/dataset-access/dataset.md).

## 4. Check schema changes

When dlt loads data, it may create or modify tables and columns. Inspect `schema_update` in each load package to see what changed.

```py
import dlt

pipeline = dlt.pipeline(
    pipeline_name="test_pipeline",
    destination="duckdb",
    dataset_name="test_data",
    dev_mode=True,
)

load_info = pipeline.run(
    [{"id": 1, "name": "Alice", "email": "alice@example.com"}],
    table_name="people",
)

for package in load_info.load_packages:
    for table_name, table in package.schema_update.items():
        print(f"Table {table_name}:")
        for column_name, column in table["columns"].items():
            print(f"  {column_name}: {column['data_type']}")
```

You can also export the full schema as YAML for review:

```py
print(pipeline.default_schema.to_pretty_yaml())
```

## 5. Read the pipeline trace

The trace records timing and configuration details for each pipeline step (extract, normalize, load). Access it through `pipeline.last_trace`.

```py
import dlt

pipeline = dlt.pipeline(
    pipeline_name="test_pipeline",
    destination="duckdb",
    dataset_name="test_data",
    dev_mode=True,
)

load_info = pipeline.run(
    [{"id": 1, "name": "Alice"}, {"id": 2, "name": "Bob"}],
    table_name="people",
)

# print the full trace
print(pipeline.last_trace)

# access individual step results
print(pipeline.last_trace.last_extract_info)
print(pipeline.last_trace.last_normalize_info)

# get row counts from normalization
print(pipeline.last_trace.last_normalize_info.row_counts)
```

## 6. Debug with the CLI

The dlt CLI provides commands for inspecting pipeline state without writing code.

```sh
# show pipeline state, schemas, and load packages
dlt pipeline test_pipeline info

# show the last run trace with timing
dlt pipeline test_pipeline trace

# display the schema in YAML format
dlt pipeline test_pipeline schema --format yaml

# inspect failed jobs and their error messages
dlt pipeline test_pipeline failed-jobs
```

For full stack traces on exceptions, add the `--debug` flag:

```sh
dlt --debug pipeline test_pipeline info
```

## 7. Enable verbose logging

By default, dlt logs at the `WARNING` level. Set the log level to `INFO` or `DEBUG` to see detailed output during development.

Add to your `.dlt/config.toml`:

```toml
[runtime]
log_level="INFO"
```

Or set the environment variable:

```sh
export RUNTIME__LOG_LEVEL=INFO
```

:::note
Avoid `DEBUG` level in production. It generates a large volume of output and may include sensitive information.
:::

## 8. Handle common errors

When a pipeline step fails, dlt raises `PipelineStepFailed`. This exception wraps the actual cause and tells you which step failed.

```py
import dlt
from dlt.pipeline.exceptions import PipelineStepFailed

pipeline = dlt.pipeline(
    pipeline_name="test_pipeline",
    destination="duckdb",
    dataset_name="test_data",
    dev_mode=True,
)

try:
    load_info = pipeline.run(my_source())
except PipelineStepFailed as e:
    print(f"Step failed: {e.step}")
    print(f"Cause: {e.exception}")
    if e.has_pending_data:
        print("Pipeline has pending data from a previous step.")
```

Common error types and what they indicate:

| Error | Cause | Resolution |
|-------|-------|------------|
| `PipelineStepFailed` | A pipeline step (extract, normalize, or load) raised an exception | Check `e.exception` for the underlying cause |
| `DestinationHasFailedJobs` | One or more load jobs failed at the destination | Run `dlt pipeline <name> failed-jobs` to see error messages |
| `PipelineConfigMissing` | A required configuration value or secret is missing | Add the value to `.dlt/secrets.toml` or `.dlt/config.toml` |
| `TerminalException` | An unrecoverable error (bad credentials, missing permissions) | Fix the root cause; retrying will not help |
| `TransientException` | A temporary error (network timeout, rate limit) | Retry the pipeline run |

## 9. Drop pending data

If a pipeline fails partway through, it may leave pending data that blocks the next run. Clear it with:

```sh
dlt pipeline test_pipeline drop-pending-packages
```

Or programmatically:

```py
pipeline.drop_pending_packages(with_partial_loads=True)
```

## Next steps

- [Run a pipeline](run-a-pipeline.md) for execution options.
- [Run in production](../running-in-production/running.md) for monitoring, alerting, and retry strategies.
- [View the schema](../general-usage/dataset-access/view-dlt-schema.md) for schema export and visualization.
- [Access loaded data](../general-usage/dataset-access/dataset.md) for querying destination tables.
